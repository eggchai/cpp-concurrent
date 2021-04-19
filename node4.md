# 4同步操作

>不仅想要保护数据，还想对单独的线程进行同步。例如，在第一个线程完成前，等待另一个线程执行完成。通常，线程会等待特定事件发生，或者等待某一条件达成。这可能需要定期检查“任务完成”标识，或将类似的东西放到共享数据中。像这种情况就需要在线程中进行同步，C++标准库提供了一些工具可用于同步，形式上表现为条件变量(condition variables)和future。

## 4.1等待时间的条件

>夜间的火车，如何在正确的站点下车呢？一种方法是整晚不睡；另一种方法是看到站时间，定一个稍早的闹钟，但火车晚点就会过早的醒，或者闹钟没电。理想方式是：无论早晚，只要火车到站就有人或其他方式把你叫醒。

如何设置一个标志，少消耗资源的唤醒等待线程。

>当一个线程等待另一个线程完成时，可以持续的检查共享数据标志(用于做保护工作的互斥量)，直到另一线程完成工作时对这个标识进行重置。不过，这种方式会消耗线程的执行时间检查标识，并且当互斥量上锁后，其他线程就没有办法获取锁，就会持续等待。因为对等待线程资源的限制，并且在任务完成时阻碍对标识的设置。类似于保持清醒状态和列车驾驶员聊了一晚上：驾驶员不得不缓慢驾驶，因为你分散了他的注意力，所以火车需要更长的时间，才能到站。同样，等待的线程会等待更长的时间，也会消耗更多的系统资源。

```C++
bool flag;
std::mutex m;

void wait_for_flag()
{
  std::unique_lock<std::mutex> lk(m);
  while(!flag)
  {
    lk.unlock();  // 1 解锁互斥量
    std::this_thread::sleep_for(std::chrono::milliseconds(100));  // 2 休眠100ms
    lk.lock();   // 3 再锁互斥量
  }
}
```

这里用flag作为等待的标志，flag=false时进行循环，在循环内先解锁100s然后再上锁，将mutex让给其他线程，其他线程就有机会获取锁并设置标志位。

这个实现线程休眠时没有浪费执行时间，但很难确定正确的休眠时间。休眠太久可能会让任务等待时间过久。更好的解决方式是用C++标准库提供的工具等待事件的发生。通过另一线程触发等待事件的机制是最基本的唤醒方式——条件变量。当某些线程被终止时为了唤醒等待线程，终止线程会像等待着的线程广播条件达成的信息。

`<condition_variable>`头文件中的`std::condition_variable`和`std::condition_variable_any`。前者简单，仅能和mutex一起用；后者灵活，可以和合适的互斥量一起工作。

**注意用法：**

```C++
std::mutex mut;
std::queue<data_chunk> data_queue;  // 1
std::condition_variable data_cond;

void data_preparation_thread()
{
  while(more_data_to_prepare())
  {
    data_chunk const data=prepare_data();
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);  // 2
    data_cond.notify_one();  // 3
  }
}

void data_processing_thread()
{
  while(true)
  {
    std::unique_lock<std::mutex> lk(mut);  // 4
    data_cond.wait(
         lk,[]{return !data_queue.empty();});  // 5
    data_chunk data=data_queue.front();
    data_queue.pop();
    lk.unlock();  // 6
    process(data);
    if(is_last_chunk(data))
      break;
  }
}
```

最重要的一点5：这里的条件变量data_cond.wait会判断等待条件(这里是个lambda表达式，任何可调用对象都行)，如果这个条件不满足会直接解锁mutex lk并将该线程阻塞等待notify_one()通知条件变量时再苏醒，重新获取mutex再次进行条件判断——这也是为什么用unique_lock。wait就是busy-wait的优化。

线程安全的队列deque：新增两个函数try_pop()和wait_and_pop()。

```C++
#include <queue>
#include <memory>
#include <mutex>
#include <condition_variable>

template<typename T>
class threadsafe_queue
{
private:
  mutable std::mutex mut;  // 1 互斥量必须是可变的 
  std::queue<T> data_queue;
  std::condition_variable data_cond;
public:
  threadsafe_queue()
  {}
  threadsafe_queue(threadsafe_queue const& other)
  {
    std::lock_guard<std::mutex> lk(other.mut);
    data_queue=other.data_queue;
  }

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(new_value);
    data_cond.notify_one();
  }

  void wait_and_pop(T& value)
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    value=data_queue.front();
    data_queue.pop();
  }

  std::shared_ptr<T> wait_and_pop()
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
    data_queue.pop();
    return res;
  }

  bool try_pop(T& value)
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return false;
    value=data_queue.front();
    data_queue.pop();
    return true;
  }

  std::shared_ptr<T> try_pop()
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return std::shared_ptr<T>();
    std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
    data_queue.pop();
    return res;
  }

  bool empty() const
  {
    std::lock_guard<std::mutex> lk(mut);
    return data_queue.empty();
  }
};
```

`在push中是锁住整个队列的，怎么只锁住头或尾呢？`

`还有一种可能是很多线程等待同一事件。对于通知都需要作出回应，用notify_all()。 如果是等待一组可用数据块时，一个条件变量就不是同步操作最好的选择了，下面介绍future，对条件变量的补足。`

## future

乘飞机办完登记手续后，等待广播通知登机的期间可以做其他事，但等待的只有广播通知。这种事成为future，线程等待特定事件时，某种程度上就需要直到期望的结果。之后线程会周期性的等待或检查(信息板)是否触发。触发后future状态变为就绪状态。future可能与数据相关(如登机口编号)也可能不是。

两种future: unique_future `std::future<>`和shared_future `std::shared_future<>`。这两个的区别与智能指针的unique_ptr和shared_ptr类似。

std::future只能与指定时间关联；std::shared_future可以关联多个事件。`这叫相似？shared_ptr也不能指向多个对象啊。` 可以初始化为空。

```C++
#include <future>
#include <iostream>

int find_the_answer_to_ltuae();
void do_other_stuff();
int main()
{
  std::future<int> the_answer=std::async(find_the_answer_to_ltuae);
  do_other_stuff();
  std::cout<<"The answer is "<<the_answer.get()<<std::endl;
}
```

`std::thread 和 std::async的区别`：二者不是一个概念，对于std::async并不一定创建一个新的线程，它有两个参数std::launch::async和std::launch::defered自动选择，指定下前者表示创建新线程执行，后者表示推迟到get或者wait时再执行。另外就是可以通过对future用get()得到async的结果，也可以std::luanch::async|std::luanch::defered；而如果用线程实现的话需要写返回结果。

传递参数上和std::thread一样。

std::async先将异步操作用std::packaged_task包装起来，然后把异步操作的结果放到std::promise中，这个过程就是创建future的过程；外面通过future.get()或wait()来获取结果。

异步操作不可能马上获取结果只能在未来获取。可以查询future的状态来判断是否可能获取到结果了

+ defered 异步操作还没开始
+ ready 异步操作已经完成
+ timeout 异步操作超时
  
获取结果的三种方式：

+ get 等待异步操作结束并返回结果
+ wait 等待异步操作完成，没有返回值
+ wait for 超时等待返回结果

通过std::packaged_task在线程间传递任务。

`什么时候使用条件变量condition_variable什么时候用future？`