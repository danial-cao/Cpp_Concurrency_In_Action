#第9章 高级线程管理

**本章主要内容**

- 线程池<br>
- 处理线程池中任务的依赖关系<br>
- 池中线程如何获取任务<br>
- 中断线程<br>

在之前的章节中，我们通过创建`std::thread`对象来对每一个线程进行管理。在一些地方，你已经看到这种方式是不可取的了，因为需要在线程的整个生命周期中对其进行管理，根据当前使用的硬件来决定恰当的线程数量，等等。理想情况是将代码划分为最小块，以便能并发执行，然后将他们传递给处理器和标准库，进行性能的并行优化。

另一种情况是，当你使用多线程来解决某个问题时，当某个条件达成的时候，就可以提前结束。这可能是因为结果已经确定好了，或者因为发生了错误，亦或者是用户显式的终止操作。无论是哪种原因，线程都需要发送“请停止”的请求，放弃给定的任务，清理，然后尽快停止运行。

在本章，我们将了解一下管理线程和任务的机制，从自动管理线程数量和自自动管理划分任务开始。

##9.1 线程池

在很多公司里面，雇员通常会在办公室度过他们的办公时光(偶尔也会外出访问客户或供应商)，或是参加贸易展会。虽然这些路程可能很有必要，并且可能需要很多人一起去，不过对于一些比较特别的雇员来说，这一去就是几个月，甚至是几年。公司要给每个雇员都配一辆车，这基本上是不可能的，不过公司可以提供一些共用车辆；这样就会有一定数量车，来让所有雇员使用。当一个员工要去异地旅游时，那么他就可以从共用车辆中预定一辆，并在返回公司的时候将车交还。如果有一天共用车辆没有闲置的，雇员就不得不将其旅程后延了。

线程池就是类似的一种方式。在大多数系统中，将每个任务制定给一个线程是不切实际的，不过可以利用现有的并发可能，进行并发。线程池就提供了这样的功能；提交到线程池中的任务将会被并发执行，提交的任务将会挂在任务队列上。队列中的每一个任务都会被池中的一个工作线程所获取，当任务执行完成后，再回到线程池中获取下一个任务。

创建一个线程池的时候，会遇到几个关键的设计问题，比如，可使用的线程数量，高效的任务分配方式，以及是否需要等待一个任务完成。在本节，我们将看到很多线程池的实现是如何解决这些问题的，让我们从最简单的线程池开始吧！

###9.1.1 最简单的线程池

作为最简单的线程池，其拥有固定数量的工作线程(通常工作线程的数量是与`std::thread::hardware_concurrency()`相同的)。当有工作需要完成，可以调用函数将任务挂在任务队列中。每个工作线程都会从任务队列上获取任务，然后执行这个任务，执行完成后再回来获取新的任务。在最简单的情况下，线程就不需要等待其他线程完成对应任务了。如果需要等待，那么就需要对同步进行管理。

下面清单中的代码就展示了一个最简单的线程池实现。

清单9.1 简单的线程池
```c++
class thread_pool
{
  std::atomic_bool done;
  thread_safe_queue<std::function<void()> > work_queue;  // 1
  std::vector<std::thread> threads;  // 2
  join_threads joiner;  // 3

  void worker_thread()
  {
    while(!done)  // 4
    {
      std::function<void()> task;
      if(work_queue.try_pop(task))  // 5
      {
        task();  // 6
      }
      else
      {
        std::this_thread::yield();  // 7
      }
    }
  }

public:
  thread_pool():
    done(false),joiner(threads)
  {
    unsigned const thread_count=std::thread::hardware_concurrency();  // 8

    try
    {
      for(unsigned i=0;i<thread_count;++i)
      {
        threads.push_back( 
          std::thread(&thread_pool::worker_thread,this));  // 9
      }
    }
    catch(...)
    {
      done=true;  // 10
      throw;
    }
  }

  ~thread_pool()
  {
    done=true;  // 11
  }

  template<typename FunctionType>
  void submit(FunctionType f)
  {
    work_queue.push(std::function<void()>(f));  // 12
  }
};
```

这个实现中有一组工作线程②，并且使用了一个线程安全队列(从第6章)①来管理任务队列。这种情况下，用户不用等待任务，并且这些函数不需要返回任何值，所以这里可以使用`std::function<void()>`来封装任务。submit()函数会对函数或可调用对象包装成一个`std::function<void()>`实例，并将其推入队列中⑫。

线程始于构造函数：使用`std::thread::hardware_concurrency()`来获取硬件支持多少个并发线程⑧，折现线程会在worker_thread()成员函数中执行⑨。

当有异常抛出，线程启动就会失败，所以需要保证任何已启动的线程都会停止，并且能在这种情况下清理的十分干净。当有异常抛出时，通过使用*try-catch*来设置done标识⑩，还有join_threads类的实例(来自于第8章)③用来汇聚所有线程。这里当然也需要析构函数：仅设置done标识⑪，并且join_threads可以确保所有线程在线程池销毁前，全部执行完成。注意成员声明的顺序很重要：done标识和worker_queue必须在threads数组之前声明，而这个数据必须在joiner前声明。这就能确保成员能以正确的顺序销毁；比如，所有线程都停止运行时，队列就可以安全的销毁了。

worker_thread函数本身就很简单：就是执行一个循环直到done标识被设置④，从任务队列上获取任务⑤，以及同时执行这些任务⑥。如果任务队列上没有任务，函数会调用`std::this_thread::yield()`进行休息⑦，并且给予其他线程向任务队列上推送任务的机会。

对于一些简单的情况，这样线程池就足以满足要求，特别是任务完全独立没有任何返回值，或需要执行一些阻塞操作的。不过在很多情况下，这样简单的线程池是完全不够用的，还有其他情况使用这样简单的线程池可能会出现问题，比如：死锁。同样，在简单例子中，使用`std::async`能提供更好的功能，如第8章中的那些例子一样。在本章中，我们将了解一下更加复杂的线程池实现，通过添加性能满足用户需求，或减少问题的发生几率。首先，从已经提交的任务开始说起。

###9.1.2 等待提交到线程池中的任务

在第8章中的例子中，在线程间的任务划分完成后，代码会显式生成新线程，而主线程通常就是等待新生成的线程结束，确保在返回调用之前，所有任务都完成了。使用线程池，就需要等待任务提交到线程池中，而非直接提交给单个线程。这与基于`std::async`的方法(第8章等待future的例子)类似，使用清单9.1中的简单线程池，这里必须使用第4章中提到的技术：条件变量和future。这就会增加代码的复杂度；不过，这要比直接对任务进行等待的方式好的多。

通过增加线程池的复杂度，你可以直接等待任务完成。可以使用submit()函数返回一个对任务描述的句柄，并用来等待任务的完成。任务句柄会用条件变量或future进行包装，这样使用线程池来简化代码。

一种特殊的情况就是，执行任务的线程需要返回一个结果到主线程上进行处理。你已经在本书中看到多这样的例子，比如parallel_accumulate()函数(第2章)。在这种情况下，需要使用future来对最终的结果进行转移。清单9.2就展示了对简单线程池的一些修改，这样就能等待任务完成，以及在在工作线程完成后，返回一个结果到等待线程中去不过`std::packaged_task<>`实例是不可拷贝的，仅是可移动的，所以这里不能再使用`std::function<>`来实现任务队列，因为`std::function<>`需要存储可复制构造的函数对象。或者，包装一个自定义函数，用来处理只可移动的类型。这就是一个带有函数操作符的类型擦除类。只需要处理那些没有函数和无返回的函数，所以这是一个简单的虚函数调用。

清单9.2 可等待任务的线程池
```c++
class function_wrapper
{
  struct impl_base {
    virtual void call()=0;
    virtual ~impl_base() {}
  };

  std::unique_ptr<impl_base> impl;
  template<typename F>
  struct impl_type: impl_base
  {
    F f;
    impl_type(F&& f_): f(std::move(f_)) {}
    void call() { f(); }
  };
public:
  template<typename F>
  function_wrapper(F&& f):
    impl(new impl_type<F>(std::move(f)))
  {}

  void operator()() { impl->call(); }

  function_wrapper() = default;

  function_wrapper(function_wrapper&& other):
    impl(std::move(other.impl))
  {}
 
  function_wrapper& operator=(function_wrapper&& other)
  {
    impl=std::move(other.impl);
    return *this;
  }

  function_wrapper(const function_wrapper&)=delete;
  function_wrapper(function_wrapper&)=delete;
  function_wrapper& operator=(const function_wrapper&)=delete;
};

class thread_pool
{
  thread_safe_queue<function_wrapper> work_queue;  // 使用function_wrapper，而非使用std::function

  void worker_thread()
  {
    while(!done)
    {
      function_wrapper task;
      if(work_queue.try_pop(task))
      {
        task();
      }
      else
      {
        std::this_thread::yield();
      }
    }
  }
public:
  template<typename FunctionType>
  std::future<typename std::result_of<FunctionType()>::type>  // 1
    submit(FunctionType f)
  {
    typedef typename std::result_of<FunctionType()>::type
      result_type;  // 2
    
    std::packaged_task<result_type()> task(std::move(f));  // 3
    std::future<result_type> res(task.get_future());  // 4
    work_queue.push(std::move(task));  // 5
    return res;  // 6
  }
  // 休息一下
};
```

首先，修改的是submit()函数①返回一个`std::future<>`保存任务的返回值，并且允许调用者等待任务完全结束。这就需要你知道提供函数f的返回类型，这就是使用`std::result_of<>`的原因：`std::result_of<FunctionType()>::type`是FunctionType类型的引用实例(如f)，并且没有参数。同样，在函数中可以对result_type typedef②使用`std::result_of<>`。

###9.1.3 需要等待的任务

###9.1.4 避免任务队列的竞争

###9.1.5 获取任务

##9.2 中断线程

###9.2.1 启动和中断其他线程

###9.2.2 检查线程是否中断

###9.2.3 中断条件变量等待

###9.2.4 中断`std::condition_variable_any`的等待

###9.2.5 中断其他阻塞调用

###9.2.6 处理中断

###9.2.7 应用退出时中断后台任务

##9.3 小结