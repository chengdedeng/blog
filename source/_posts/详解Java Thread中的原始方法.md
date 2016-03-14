title: 详解Java Thread中的原始方法
date: 2011-3-9 22:56:29
categories: 技术
tags: [java,线程]

-----

>Thread中大量的方法已经被抛弃，很多人可能都知道它们线程不安全，本篇文章就是要详细的介绍这些不被推荐的方法背后隐藏的诸多陷阱。

### Thread.stop
当一个正在运行的线程，被执行stop时，该线程会抛出一个**ThreadDeath**实例。如果此时该线程已经进入[管程](https://zh.wikipedia.org/zh-cn/%E7%9B%A3%E8%A6%96%E5%99%A8_(%E7%A8%8B%E5%BA%8F%E5%90%8C%E6%AD%A5%E5%8C%96\))中，就可能使受管程保护的对象出现不确定的状态，从而导致使用该对象的别的线程出现问题。这样的对象，我们称之为**损坏的对象**。当线程操作包含损坏对象，任意的行为都会影响结果，而这些行为又是琢磨不定的。**ThreadDeath**不像别的未检查异常，它会悄悄的杀死线程，不会有任何警告，从而导致程序腐化。代码腐化可能会在任何时候导致实际损害发生，甚至在未来几小时或者几天。

<!--more-->


为了形象的说明代码腐化所带来的影响，我们可以详解下面这段代码

``` java
/**
 * Created by IntelliJ IDEA
 * User:杨果
 * Date:11/3/9
 * Time:下午10:05
 * <p/>
 * Description:
 */
public class ThreadDeathTest {
    private static int resource = 0;

    public static synchronized void monitor(String threadName) {
        System.out.println("*****************");
        try {
            System.out.println(threadName + "进入管程");
            System.out.println(threadName + "开始操作互斥资源");
            resource += 1;
            Thread.sleep(5000);
            resource -= 1;
            System.out.println(threadName + "互斥资源操作完成");
            System.out.println(threadName + "退出管程");
        } catch (Throwable ex) {
            System.out.println("捕获非检查异常:" + ex);
            //ex.printStackTrace();
        }
    }


    public static void main(String[] args) {
        try {
            Thread thread1 = new Thread() {
                public void run() {
                    monitor("Thread1");
                }
            };
            thread1.start();
            Thread thread2 = new Thread() {
                public void run() {
                    monitor("Thread2");
                }
            };
            thread2.start();
            //给充足的时间，让线程启动
            Thread.sleep(1000);
            thread1.stop();
            thread2.join();
            System.out.println("------------------");
            System.out.println("互斥资源期待值为：" + 0);
            System.out.println("互斥资源实际值为：" + resource);
        } catch (Throwable t) {
            System.out.println("Caught in main: " + t);
        }
    }
}
```

这段程序的输出的结果为

```
*****************
Thread1进入管程
Thread1开始操作互斥资源
捕获非检查异常:java.lang.ThreadDeath
*****************
Thread2进入管程
Thread2开始操作互斥资源
Thread2互斥资源操作完成
Thread2退出管程
------------------
互斥资源期待值为：0
互斥资源实际值为：1
```

代码中我们用一个同步方法**monitor**（管程）来保护**resource**，确保它会被互斥的使用。**Thread1**和**Thread2**都需要调用同步方法，来操作**resource**。**Thread1**先进入**monitor**，**Thread2**在等待互斥锁，当**Thread1**被调用了**stop**时还处于同步块，还没有来的急将**resource**处理完，便由于抛出**TheadDeath**释放了互斥锁，**Thread2**获得锁进入管程，便得到上面的结果。那这个代码就腐化了，这时产生的结果就会存在问题的。


那既然**ThreadDeath**会导致代码腐化，那我们是否可以捕获它，然后修复已经**损坏的对象**呢？

当然，理论上这是可行的，但是这会给编写正确的多线程代码增加极大的难度。该任务基本上是不可能成功的，原因有如下两点：

1. 一个线程可以在任何地方抛出一个ThreadDeath异常。基于这点，所有的同步方法和块必须详细的设计。
2. 一个线程在**catch**或者**finally**中清理**ThreadDeath**时，可能会抛出第二个**ThreadDeath**。清理工作要一直重复，直到清理成功。这个确保代码是非常复杂的。

总之，该方法是不适用的。


除了上面提供的所有问题，**Thread.stop(Throwable)**方法可能会被用来生成目标线程未准备处理的异常（如果不是因为该方法，线程不太可能抛出包括已检查异常在内的异常）。下面例子中方法的行为等同于**throw**操作，但是它绕开了编译器保证每个方法调用者都能知道方法会抛出的已检查异常。

``` java
 static void sneakyThrow(Throwable t) {
        Thread.currentThread().stop(t);
    }
    
```



在绝大多数情况下，我们可以通过修改变量表示目标线程应该停止运行，来代替stop方法。当然线程要周期性的检查该变量，如果该变量表示线程要终止，那么线程需要一种有序的方式从**run**方法退出。为了确保停止请求能够及时响应，该变量必须用**volatile**修饰或者同步访问该变量。

下面假设你的applet中包含这样的一段代码：

``` java
private Thread blinker;

    public void start() {
        blinker = new Thread(this);
        blinker.start();
    }

    public void stop() {
        blinker.stop();  // UNSAFE!
    }

    public void run() {
        while (true) {
            try {
                Thread.sleep(interval);
            } catch (InterruptedException e){
            }
            repaint();
        }
    }
```
采用变量的方式，让线程的停止变的安全的代码如下：

``` java
private volatile Thread blinker;

    public void stop() {
        blinker = null;
    }

    public void run() {
        Thread thisThread = Thread.currentThread();
        while (blinker == thisThread) {
            try {
                Thread.sleep(interval);
            } catch (InterruptedException e){
            }
            repaint();
        }
    }

```
---------
### Thread.interrupt

如果我们要停止一个等待周期非常长的线程，如正在等待输入的线程，就要使用**Thread.interrupt**方法。我们只需要在我们上面的例子的例子中改变状态的地方，加上**Thread.interrupt**来打断等待。

``` java
public void stop() {
        Thread moribund = waiter;
        waiter = null;
        moribund.interrupt();
    }
```
这种方式中，任何捕获了**interrupt exception**的方法，如果不准备立即处理该异常，那么重新申明（reasserts）该异常是非常重要的。此处用reasserts而不用rethrows，因为这里不总是重新抛出异常。因为存在一个方法捕获了**InterruptedException**而不抛出这个已检查的异常，而是重新终端自己，如下面的代码。

``` java
Thread.currentThread().interrupt();
```
这样就可以尽可能的保证线程能够再增加一个**InterruptedException**。

---------

在实际中，不是所有的线程都会响应**Thread.interrupt**，这个时候我们可以使用特定于应用程序之上的技巧。例如，一个线程正在一个已知的套接字上等待，我们可以关闭该套接字，使程序立即返回。但不幸的是，没有一个通用的技巧。故意拒绝服务攻击，**Thread.stop**和**Thread.interrupt**不能正常工作的IO操作等导致处于等待中的线程，都会使**Thread.interrupt**和**Thread.stop**不会被响应。

测试代码如下：

``` java
public class Test {
    public static class T1 implements Runnable {
        @Override
        public void run() {
            BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
            System.out.println("Enter your value:");
            String str = null;
            try {
                str = br.readLine();
            } catch (IOException e) {
                e.printStackTrace();
            }
            System.out.println("your value is :" + str);
        }
    }


    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new T1());
        t1.start();
        //等待线程启动成功
        Thread.sleep(1000);
        t1.stop();
//        or
//        t1.interrupt();
    }
}

```
--------

### Thread.suspend
**Thread.suspend**天生具有死锁倾向。如果目标线程挂起时在保护关键系统资源的管程上持有锁，则在目标线程重新开始以前任何线程都不能访问该资源。如果重新开始目标线程的线程想在调用
### Thread.resume
**Thread.resume**之前需要锁定管程，则会发生死锁。这类死锁通常表现为自我“冻结”的进程。

同**Thread.stop**一样，谨慎的做法是“目标线程”有一个变量来表示其状态(active或者suspended)。理想的状态是暂停时，使用**Object.wait**让线程等待。当线程恢复时，目标线程使用**Object.notify**来发出通知通知。

加入你的applet程序中包含这样一段处理鼠标点击的处理逻辑，我们用**blinker**来表示线程状态的切换者。

``` java
   private boolean threadSuspended;

    Public void mousePressed(MouseEvent e) {
        e.consume();

        if (threadSuspended)
            blinker.resume();
        else
            blinker.suspend();  // DEADLOCK-PRONE!

        threadSuspended = !threadSuspended;
    }
```
上面这段代码是不安全的，你可以在上面的事件处理中避免使用**Thread.suspend**和**Thread.resume**，如下所示

``` java
 public synchronized void mousePressed(MouseEvent e) {
        e.consume();

        threadSuspended = !threadSuspended;

        if (!threadSuspended)
            notify();
    }
```
但是你需要将下面的代码加入到线程的**run**方法中

``` java
synchronized(this) {
                    while (threadSuspended)
                        wait();
                }
```

由于**wait**方法会抛出**InterruptedException**，所以我们需要用try ... catch 来处理。

``` java
public void run() {
        while (true) {
            try {
                Thread.sleep(interval);

                synchronized(this) {
                    while (threadSuspended)
                        wait();
                }
            } catch (InterruptedException e){
            }
            repaint();
        }
    }
```
需要注意的是**notify**在**mousePressed**方法中，而**wait**在**run**方法的同步块中。

由于Java的同步操作代价较大，我们可以尽量减小同步块的体积，然锁占用的减少的最少。我们可以将代码再次优化为如下所示：

``` java
    private volatile boolean threadSuspended;

    public void run() {
        while (true) {
            try {
                Thread.sleep(interval);

                if (threadSuspended) {
                    synchronized(this) {
                        while (threadSuspended)
                            wait();
                    }
                }
            } catch (InterruptedException e){
            }
            repaint();
        }
    }
```

**volatile**的使用此处就不展开了，具体的可以查阅相关文档。

----
### Thread.destroy
**Thread.destroy**从未被实现并且已经弃用。假如它被实现了，也会像**Thread.suspend**一样容易导致死锁。






















