---
layout: post
category: "program"
title:  "线程池概念及实现"
tags: [编程,java,thread]
---

1. 理解误区，一直认为线程池就像缓存区一样存放着N个线程对象Thread,当需要处理一些东西时就向池中投递Runnable对象，然后线程池能通过一个线程对象来启动一个线程，然后让线程执行Runnable，直到有一次使用线程时发现一个线程对象只能调用一次start()方法时，才感觉到上面描述的线程池的概念是错误的。

1. 其实线程池并不仅缓存N个线程对象Thread（这不是重点），重要的是在线程池初始化时会通过这N个线程对象启动N个线程，然后这些线程循环执行一个（各自）的队列，若队列为空时线程就处于等待状态，一旦有新的任务加入到队列的时候线程会开始执行，当然执行完一个Runnable后要把它从队列中移除。

1. 线程是程序中一个单一的顺序控制流程，这个概念太抽象。线程是能执行方法且有栈结构的东西，方法一定是被某个线程（而不只一个）执行，这样程序员更好理解一点。线程间平等且又相互制约，平等是线程间没有父子关系（并不是说main里创建了新的线程,就非得等新的线程执行完成后才结束），制约是它们共同抢占cpu资源。

```
public class ThreadPool {
  /*
  * 线程池并不仅仅像对象缓存那样,把线程对象thread缓存起来.
  * 且是把所有线程启动,让它们循环执行一个消息队列,这里每个线程池* 里的线程都有与之对应的队列,
  * 以便线程池里的线程互不影响,当想让线程池完成新的任务时,就向消* 息队列中投递Runnable.
  * 当消息队列中没有任务时,线程处于等待状态,直到有新的任务加入.
  */


 /*
  * Thread.sleep()与obj.wait(),都会让当前线程停止执行
  * Thread.sleep(),线程休眠指定的时间,但线程并不释放它已经拥有* 对象的监视权.
  * obj.wait(),此方法调用前当前线程必须已拥有obj的监视权(即在*synchronized代码片中)
  * 此方法调用后当前线程会暂时失去obj对象的监视权.
  * 所以Thread.sleep()导致想拥有当前线程已拥有对象监视权的线程* T继续等待,而obj.wait()
  * 可能导致,想拥有obj监视权的线程得到监视权,进而运行.
  */

  public static void main(String[] args) {
    ThreadPool pool = ThreadPool.getInstance();

    for (int i = 0; i < 600; i++) {
      System.out.println(Thread.currentThread().getName() + " create task" + i);
      pool.addTask(new Task(i));
    }
    try {
      Thread.sleep(10000);
    } catch (Exception e) {
      System.out.println("an exception:" + e);
    }
    pool.destroy();
    System.out.println(Thread.currentThread().getName() + " is over");
  }
  /* 单例 */
  private static final ThreadPool instance = ThreadPool.getInstance();
  /* 默认池中线程数,最大个数 */
  private static final int THREAD_MAX_COUNT = 5;
  /* 默认池中线程数 */
  private int mThreadCount;
  /* 池中的所有线程 */
  private WorkerThread[] mWorkerThread;
  private ThreadPool() {
    this(THREAD_MAX_COUNT);
  }
  private ThreadPool(int poolCount) {
    if (0 >= poolCount || poolCount > THREAD_MAX_COUNT) {
      poolCount = THREAD_MAX_COUNT;
    }
    mThreadCount = poolCount;
    mWorkerThread = new WorkerThread[mThreadCount];
    for (int i = 0; i < mWorkerThread.length; i++) {
      mWorkerThread[i] = new WorkerThread("thread" + i);
      mWorkerThread[i].start();
    }
  }
  public static synchronized ThreadPool getInstance() {
    if (instance == null)
      return new ThreadPool();
    return instance;
  }
  /**
  * 向线程池中增加新的任务
  * @param newTask 新的任务
  */
  public void addTask(Task newTask) {
    /*
    *  找出线程池中剩余任务最少的那个线程对象
    */
    WorkerThread minTasktThread = null;
    for (WorkerThread thread : mWorkerThread) {
      if (null == minTasktThread || minTasktThread.getTaskCount() > thread.getTaskCount()) {
      minTasktThread = thread;
      }
    }
    if (null != minTasktThread) {
      synchronized (minTasktThread.mTaskQueue) {
        minTasktThread.addTask(newTask);
        minTasktThread.mTaskQueue.notifyAll();
      }
    }
  }
  /**
  * 批量向线程池中增加新的任务
  * 注：这些任务未必都被同一个线程来执行
  * @param taskes 任务列表
  */
  public void batchAddTask(Task[] taskes) {
    if (taskes == null || taskes.length == 0) {
    return;
    }
    synchronized (taskes) {
      for (Task task : taskes) {
        addTask(task);
      }
    }
  }
  @Override
  public String toString() {
    StringBuffer sb = new StringBuffer();
    sb.append(super.toString());
    sb.append(";thread count:" + mThreadCount + ";");
    return sb.toString();
  }
  /**
  * 销毁线程池
  */
  public synchronized void destroy() {
    for (WorkerThread thread : mWorkerThread) {
      thread.stopWork();
    }
  }
  /**
  * 池中工作的线程
  */
  private class WorkerThread extends Thread {
    /* 该工作线程是否有效 */
    private boolean isRunning;
    /* 该工作线程是否可以执行新任务 */
    private boolean isWaiting = true;
    /* 任务列表 */
    private final List<Runnable> mTaskQueue = Collections
    .synchronizedList(new LinkedList<Runnable>());
    public WorkerThread(String name) {
      super(name);
    }
    public void stopWork() {
      isRunning = false;
      synchronized (mTaskQueue) {
        mTaskQueue.notifyAll();
      }
    }
    /**
    * 是否可以执行新的任务
    *
    * @return true, 可以投递新的任务到任务队列中
    */
    @Deprecated
    public boolean isWaiting() {
      return isRunning && isWaiting && isAlive() && !isInterrupted();
    }

    int getTaskCount() {
      return mTaskQueue.size();
    }

    /**
    * 向线程队列中加入新的任务
    * @param newtTask 新任务
    * @return true, 添加成功
    */
    boolean addTask(Task newTask) {
      System.out.println(getName() + " add task" + newTask.taskId);
      return mTaskQueue.add(newTask);
    }
    public void run() {
      isRunning = true;
      while (isRunning) {
        // 锁定要放在循环的内部
        synchronized (mTaskQueue) {
          if (!mTaskQueue.isEmpty()) {
            // 只执行一个任务,而不要一下执行全部
            Runnable task = mTaskQueue.remove(0);
            if (null != task) {
              // TODO 最好能对异常进行捕获
              task.run();
            }
          } else {
            try {
              isWaiting = true;
              mTaskQueue.wait();
              isWaiting = false;
            } catch (InterruptedException e) {
              e.printStackTrace();
              // 此处不处理异常,因其不会抛出
              System.err.println("an exception:" + e);
            }
          }
        }
      }
      System.out.println(getName() + " is over!");
    }
  }

  /* just for debug */
  class Task implements Runnable {
    int taskId;
    public Task(int taskId) {
      this.taskId = taskId;
    }
    @Override
    public void run()
    {
      System.out.println(Thread.currentThread().getName() + " run task" + taskId);
    }
  }
}
```
