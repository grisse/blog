# Java线程基础知识

本文不讲解线程的基础概念，默认读者了解线程概念、Java Thread类和Runnable接口等基础知识，基础概念可以查阅 [Java Api Doc中关于Thread类的描述](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html)。

本文包含以下内容：

- Java线程的状态
- 线程的正确中止方式
- 线程间通信
- 线程封闭之ThreadLocal和栈封闭

## Java线程的状态

在 [`java.lang.Thread.State`](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.State.html) 枚举类中定义了 6 种线程状态：

- NEW：尚未启动的线程处于此状态。即通过构造方法新创建的 `Thread` 类处于此状态。
- RUNNABLE：可运行线程的状态。此状态的线程在JVM中正在执行，但是可能正在等待来自操作系统的其他资源：如CPU时间片。
- BLOCKED：线程阻塞等待获取监视器锁(monitor lock)的状态。处于synchronized同步代码块或方法中被阻塞等待一个资源锁的线程处于此状态。
- WAITING：等待线程的线程状态。此状态的线程依赖另一个线程的通知调度，线程中调用以下 **没有超时(timeout)** 的方法之一进入此状态:
    - Object.wait (with no timeout)
    - Thread.join (with no timeout)
    - LockSupport.park

- TIMED_WAITING：具有指定等待时间的等待线程的线程状态。和 `WAITING` 状态有些相似，但是不同的是 `WAITING` 状态的线程会 **一直** 处于等待状态，直到收到被唤醒继续执行的通知（如调用 `Object.notify()`方法 或 `Object.notifyAll()`方法）。而 `TIMED_WAITING` 状态是带超时时间的等待状态，如果在超时时间内没有收到唤醒通知，便抛出超时异常或继续执行。调用以下指定等待超时时间的方法会使线程进入此状态：
    - Thread.sleep
    - Object.wait (with timeout)
    - Thread.join (with timeout)
    - LockSupport.parkNanos
    - LockSupport.parkUntil

- TERMINATED：已终止线程的线程状态。线程已经执行完毕。

### Java 线程状态切换图示

### Java 线程状态切换Demo

以下几个代码片段展示了 Java 线程状态之间相互切换：

#### NEW -> RUNNABLE -> TERMINATED

````java
public class Main() {
    public static void main(String[] args) {
        // NEW -> RUNNABLE -> TERMINATED
        Thread thread1 = new Thread(() ->
                System.out.println("thread1执行了，当前状态：" + Thread.currentThread().getState()));
        System.out.println("调用start方法前，thread1当前状态：" + thread1.getState());
        thread1.start();
        try {
            Thread.sleep(2000L);
            System.out.println("等待2秒后，thread1当前状态：" + thread1.getState());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
````
执行结果：

````text
调用start方法前，thread1当前状态：NEW
thread1执行了，当前状态：RUNNABLE
等待两秒后，thread1当前状态：TERMINATED
````

#### NEW -> RUNNABLE -> WAITING/TIMED_WAITING -> RUNNABLE -> TERMINATED

````java
public class Main() {
    public static void main(String[] args) {
        // NEW -> RUNNABLE -> WAITING -> RUNNABLE -> TERMINATED
        Thread thread2 = new Thread(() -> {
            try {
                Thread.sleep(1500L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("thread2执行了，当前状态：" + Thread.currentThread().getState());
        });
        System.out.println("调用start方法前，thread2当前状态：" + thread2.getState());
        thread2.start();
        System.out.println("调用start方法，thread2当前状态：" + thread2.getState());
        try {
            Thread.sleep(200L);
            System.out.println("等待200毫秒后，thread2当前状态：" + thread2.getState());
            Thread.sleep(2000L);
            System.out.println("等待2秒后，thread2当前状态：" + thread2.getState());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
````
执行结果：

````text
调用start方法前，thread2当前状态：NEW
调用start方法，thread2当前状态：RUNNABLE
等待200毫秒后，thread2当前状态：TIMED_WAITING
thread2执行了，当前状态：RUNNABLE
等待2秒后，thread2当前状态：TERMINATED
````

#### NEW -> RUNNABLE -> BLOCKED -> RUNNABLE -> TERMINATED

````java
public class Main {
    public static void main(String[] args) {
        // NEW -> RUNNABLE -> BLOCKED -> RUNNABLE -> TERMINATED
        Thread thread3 = new Thread(() -> {
            synchronized (Main.class) {
                System.out.println("thread3执行了，当前状态：" + Thread.currentThread().getState());
            }
        });

        synchronized (Main.class) {
            System.out.println("调用start方法前，thread3当前状态：" + thread3.getState());
            thread3.start();
            System.out.println("调用start方法，thread3当前状态：" + thread3.getState());
            try {
                Thread.sleep(200L);
                System.out.println("等待200毫秒后，thread3当前状态：" + thread3.getState());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        try {
            Thread.sleep(2000L);
            System.out.println("等待2秒后，让thread3抢到锁，thread3当前状态：" + thread3.getState());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

}

````

执行结果：

````text
调用start方法前，thread3当前状态：NEW
调用start方法，thread3当前状态：RUNNABLE
等待200毫秒后，thread3当前状态：BLOCKED
thread3执行了，当前状态：RUNNABLE
等待2秒后，让thread3抢到锁，thread3当前状态：TERMINATED
````