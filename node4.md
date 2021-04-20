# 4同步操作

## 4.1等待一个时间或其他条件

当一个线程A需要等待另一个线程B完成一个任务：

1. 设置一个标志flag，当B完成时将标志设为true。线程A不断的检测flag。两个问题：a.线程A不断检查标志浪费处理时间；b.就是这个flag需要mutex保护，A不断检测锁定，B没有办法改写flag。
2. 设置标志flag，让等待线程A在检查间隙解锁flag给等待线程B执行的时间。用std::this_thread::sleep_for()。但是这个方法休眠时间短还是浪费时间检查，休眠时间长可能B已经处理完了A还在休眠。也就是下面第一份代码。
3. 用条件变量等待，注意写法。在B完成后触发条件唤醒A

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

这里用flag作为等待的标志，flag=false时进行循环，在循环内先解锁100ms然后再上锁，在此期间将mutex让给其他线程，其他线程就有机会获取锁并设置标志位。

`<condition_variable>`头文件中的`std::condition_variable`和`std::condition_variable_any`。前者简单，仅能和mutex一起用，优先使用；后者灵活，可以和合适的互斥量一起工作。

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

最重要的一点5：这里的条件变量data_cond.wait会判断等待条件(这里是个lambda表达式，任何可调用对象都行，lambda很适合用在这里)，如果这个条件不满足会直接解锁mutex lk并将该线程阻塞直到notify_one()通知条件变量时再苏醒，重新获取mutex再次进行条件判断——这也是为什么用unique_lock，因为等待线程必须在等待期间解锁，如果用lock_guard一直锁住，那么被等待线程B就没有办法获取。wait就是busy-wait的优化。

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

## future的使用

要想从线程中返回异步任务结果，一般使用全局变量，不安全。std::future提供了异步操作结果的机制，轻松解决异步任务返回结果。

两种future: unique_future std::future<>和shared_future        std::shared_future<>。这两个的区别与智能指针的unique_ptr和shared_ptr类似。

future和std::packaged_task<>和std::promise好像是绑定的，可以通过这两个来获取。

std::future只能与指定事件关联；std::shared_future可以关联多个事件。`这叫相似？shared_ptr也不能指向多个对象啊。——准确的说法我觉得应该是允不允许有拷贝的意思，unique_prt只能move不能拷贝，也就是只有这个指针能指向这个地址；而shared_ptr可以通过拷贝使得多个指针指向同一个地址。类推到这里，std::future就不适用与多个线程等待同一结果的情况，用shared_future给每个线程都配一个` 可以初始化为空。

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

`std::thread 和 std::async的区别`：二者不是一个概念，对于std::async并不一定创建一个新的线程，它有两个参数std::launch::async和std::launch::defered选择，指定下前者表示创建新线程执行，后者表示推迟到get或者wait时再执行。另外就是可以通过对future用get()得到async的结果，也可以std::luanch::async | std::luanch::defered自动选择。

传递参数上和std::thread一样。

`async的工作流程
std::async先将异步操作用std::packaged_task包装起来，然后把异步操作的结果放到std::promise中，这个过程就是创建future的过程；外面通过future.get()或wait()来获取结果。在调用get()或wait()时线程就会阻塞，直到future的状态为ready，返回该值。`

`future promise和packaged_task的关系
future是一个底层的对象，std::promise和std::packaged_task的结果最终都是通过内部的future返回出来的。std::future提供了一个访问异步操作结果的机制，和线程一样是一个级别属于低层次的对象。std::promise和std::packaged_task都是较高层的对象，内部都有future以便获得异步操作结果。而std::promise和std::packaged_task没什么关系，std::packaged_task可以把这个异步操作存在std::promise中，仅此而已。`

异步操作不可能马上获取结果只能在未来获取。可以查询future的状态来判断是否可能获取到结果了。

future的三种状态

+ defered 异步操作还没开始
+ ready 异步操作已经完成
+ timeout 异步操作超时
  
获取结果的三种方式：

+ get 等待异步操作结束并返回结果
+ wait 等待异步操作完成，没有返回值
+ wait for 超时等待返回结果

通过std::packaged_task在线程间传递任务。

std::packaged_task<>把future和可调用对象进行绑定。调用std::packageed_task<>的时候就会调用关联的可调用对象，然后设置future状态为ready。

`std::packaged_task有什么用？一个大型操作可以划分自包含的子任务，把每个子任务封装在std::packaged_task<>实例中，然后把实例传递给任务调度器或线程池，这样就抽象出了任务的细节，调度器只需要处理std::packaged_task<>而不是单独的函数。`

std::package_task<>是可调用对象，可以封装在std::function中，作为线程函数传入到std::thread对象中或作为可调用对象传递给另一个函数或直接调用。当std::package_task<>作为函数调用时，实参将由函数操作符传递给底层函数，并将返回值作为异步操作的结果存储在std::future中，并可以通过get_future()获取。

线程间传递任务：

```C++
#include <deque>
#include <mutex>
#include <future>
#include <thread>
#include <utility>

std::mutex m;
std::deque<std::packaged_task<void()> > tasks;

bool gui_shutdown_message_received();
void get_and_process_gui_message();

void gui_thread()  // 1
{
  while(!gui_shutdown_message_received())  // 2
  {
    get_and_process_gui_message();  // 3
    std::packaged_task<void()> task;
    {
      std::lock_guard<std::mutex> lk(m);
      if(tasks.empty())  // 4
        continue;
      task=std::move(tasks.front());  // 5
      tasks.pop_front();
    }
    task();  // 6
  }
}

std::thread gui_bg_thread(gui_thread);

template<typename Func>
std::future<void> post_task_for_gui_thread(Func f)
{
  std::packaged_task<void()> task(f);  // 7
  std::future<void> res=task.get_future();  // 8
  std::lock_guard<std::mutex> lk(m);
  tasks.push_back(std::move(task));  // 9
  return res; // 10
}
```

7创建一个std::packaged_task，因为std::packaged_task总是与future对应，可以8通过get_future得到相应的std::future对象，然后加锁，将这个task加入到队列中。在gui_thread()中弹出一个task并执行。可以通过res得到这个task的结果。

任务的结果还可以通过std::promises得到。通过少数线程处理网络连接，每个线程同时处理多个连接。线程处理多个连接事件，来自不同的端口连接的数据包基本以乱序方式进行处理。

std::promise\<T\>提供设置值的方式(类型T)，这个类型T与std::future<T>对象相关联，通过future来读取。future阻塞等待线程，提供数据的线程可以使用promise对相关值进行设置，并将future的状态设置为ready。可以对std::promise用get_future得到对应的std：：future对象。

```C++
#include <future>

void process_connections(connection_set& connections)
{
  while(!done(connections))  // 1
  {
    for(connection_iterator  // 2
            connection=connections.begin(),end=connections.end();
          connection!=end;
          ++connection)
    {
      if(connection->has_incoming_data())  // 3
      {
        data_packet data=connection->incoming();
        std::promise<payload_type>& p=
            connection->get_promise(data.id);  // 4
        p.set_value(data.payload);
      }
      if(connection->has_outgoing_data())  // 5
      {
        outgoing_packet data=
            connection->top_of_outgoing_queue();
        connection->send(data.payload);
        data.promise.set_value(true);  // 6
      }
    }
  }
}
```

2依次检查每个连接，检查是否有数据或正在发送已入队的传出数据。

还可以把异常存到future中

多个线程的等待：让多个线程等待同一事件。std::future独享同步结果，get()是一次性的。这里要用std::shared_future，因为std::future虽然能移动，但是一次只能有一个实例访问一个特定的异步操作结果。不能拷贝，而std::shared_future是可拷贝的，所以可以多个对象引用同一关联future的结果。为了避免数据竞争，需要用锁保护。优先使用的方法：为了替代只有一个拷贝对象的情况，可以让每个线程都有自己的拷贝对象，这样每个线程可以安全访问本地的shared_future对象。

![a](https://wx1.sinaimg.cn/mw690/006xxTXxly1gppdpbz0joj31ga0u0n00.jpg)
![b](https://wx1.sinaimg.cn/mw690/006xxTXxly1gppdpbzasdj31hu0u042j.jpg)

`什么时候使用条件变量condition_variable什么时候用future？——我觉得可能是取决于是不是要传出参数（猜的）`

## 限时等待

这个章节很大篇幅在讲如何使用C++的时钟与时长。

线程阻塞会把线程挂起一段(不确定的)时间直到响应的事件发生。但有时需要限定等待事件或终止等待。

两种指定超时方式 时间段和时间点。

condition_variable.wait_until(mutex, std::chrono::steady_clock::xx)等待直到某一时间，解锁。

超时可以用在线程的sleep(挂起)，条件变量的等待，timed_mutex的尝试加锁、解锁，unique_lock，future和shared_future的等待。

这些等待直到就可以了。
