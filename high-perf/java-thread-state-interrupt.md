# Java线程基础知识之线程状态和线程中止

本文不讲解线程的基础概念，默认读者了解线程概念、Java Thread类和Runnable接口等基础知识。

## Java线程的状态

在 [`Java.lang.Thread.State`](https://docs.oracle.com/Javase/7/docs/api/Java/lang/Thread.State.html) 枚举类中定义了 6 种线程状态：

- NEW：尚未启动的线程处于此状态。即通过 `Thread` 类构造方法创建的线程对象处于此状态。
- RUNNABLE：可运行线程的状态。此状态的线程已经可以执行，但是需要获取到CPU时间片后才会执行。
- BLOCKED：线程阻塞等待获取监视器锁(monitor lock)的状态。处于 `synchronized` 同步代码块或方法中被阻塞等待资源锁的线程处于此状态。
- WAITING：等待线程的线程状态。此状态的线程依赖另一个线程的唤醒通知，线程中调用以下 **没有超时(timeout)** 的方法之一进入此状态:
    - Object.wait (with no timeout)
    - Thread.join (with no timeout)
    - LockSupport.park

- TIMED_WAITING：具有指定等待时间的等待线程的线程状态。和 `WAITING` 状态有些相似，但是不同的是 `WAITING` 状态的线程会 **一直** 处于等待状态，直到收到被唤醒继续执行的通知（如调用 `Object.notify()`方法 或 `Object.notifyAll()`方法）。而 `TIMED_WAITING` 状态是带超时时间的等待状态，如果在超时时间内没有收到唤醒通知，则抛出超时异常或继续执行。调用以下指定超时时间的方法会使线程进入此状态：
    - Thread.sleep
    - Object.wait (with timeout)
    - Thread.join (with timeout)
    - LockSupport.parkNanos
    - LockSupport.parkUntil

- TERMINATED：已终止线程的线程状态。线程已经执行完毕。

### Java 线程状态切换图示

![Java 线程状态切换图示](https://cdn.nlark.com/yuque/0/2020/jpeg/373109/1604848443256-29e8591e-8a42-4922-a1bc-7c53cf568a9f.jpeg)

### Java 线程状态切换Demo

以下几个代码片段展示了 Java 线程状态之间相互切换：

#### NEW -> RUNNABLE -> TERMINATED

````Java
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

````Plain Text
调用start方法前，thread1当前状态：NEW
thread1执行了，当前状态：RUNNABLE
等待两秒后，thread1当前状态：TERMINATED
````

#### NEW -> RUNNABLE -> WAITING/TIMED_WAITING -> RUNNABLE -> TERMINATED

````Java
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

````Plain Text
调用start方法前，thread2当前状态：NEW
调用start方法，thread2当前状态：RUNNABLE
等待200毫秒后，thread2当前状态：TIMED_WAITING
thread2执行了，当前状态：RUNNABLE
等待2秒后，thread2当前状态：TERMINATED
````

#### NEW -> RUNNABLE -> BLOCKED -> RUNNABLE -> TERMINATED

````Java
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

````Plain Text
调用start方法前，thread3当前状态：NEW
调用start方法，thread3当前状态：RUNNABLE
等待200毫秒后，thread3当前状态：BLOCKED
thread3执行了，当前状态：RUNNABLE
等待2秒后，让thread3抢到锁，thread3当前状态：TERMINATED
````

## 线程中止

使正在执行的线程中止有三种方式： ~~`Thread.stop()`~~、 `Thread.interrupt()` 和判断标志位方式。

### 不正确的线程中止方式 —— Thread.stop()

这种方式已经被JDK弃用，被弃用的原因是这种方式可能会导致线程安全问题，下面的代码段因为调用了 `Thread.stop()` 而导致没有保证同步代码块里面数据的一致性，破坏了线程安全。

````Java
public class Main {

    public static void main(String[] args) throws InterruptedException {

        AtomicInteger i = new AtomicInteger();
        AtomicInteger j = new AtomicInteger();
        Thread thread = new Thread(() -> {
            synchronized (Main.class) {
                i.incrementAndGet();
                try {
                    // 模拟耗时操作
                    Thread.sleep(10000);
                } catch (InterruptedException e) {
                    System.out.println("捕获异常, do something...");
                    e.printStackTrace();
                }
                j.incrementAndGet();
            }
        });

        thread.start();
        // 休眠1秒，确保i自增成功
        Thread.sleep(1000);

        //中止线程
        thread.stop();
        
        // 确保线程已经中止
        while (thread.isAlive()) {
        }

        //输出结果
        System.out.println("i=" + i.get() + " j=" + j.get());
    }
}
````

上述代码段中，处于同步代码块中的对变量 `i` 和 `j` 的操作，我们期望保证数据的一致性，即理想输出结果为"i=1 j=1"，而程序的执行结果是"i=1, j=0"，违背了 `synchronized` 同步代码块原子性操作的用意，破坏了线程安全。

### 正确的线程中止方式 —— Thread.interrupt()

JDK 所推荐的线程中止方式是 `Thread.interrupt()`。

如果目标线程在调用 `Object` 类的 `wait()`、`wait(long)`、`wait(long,int)` 方法或 `Thread` 类的 `join()`、`join(long,int)`、`sleep(long,int)`方法被阻塞，则该线程的中断状态将被清除，收到 `InterruptedException`异常。

如果目标线程在 `interruptible channel` 的 I/O 操作中被阻塞，则该 channel 将被关闭。将设置目标线程的中断状态，收到 `ClosedByInterruptException` 异常。

如果目标线程在 `Selector` 中被阻塞，则设置该线程的中断状态，并且线程将立即返回一个特殊异常值（非零值）。

如果以上条件都不满足，会设置此线程的中断状态。对于非 alive 的线程，调用 `interrupt` 不会有任何效果。

将上面的代码段中 `thread.stop()` 改为 `thread.interrupt()`，得到如下输出：

````Plain Text
捕获异常, do something...
Java.lang.InterruptedException: sleep interrupted
	at Java.lang.Thread.sleep(Native Method)
	at Main.lambda$main$0(Main.Java:39)
	at Java.lang.Thread.run(Thread.Java:748)
i=1 j=1
````

程序执行结果"i=1 j=1"达到了预期效果，保证了数据一致性。通过 `Thread.interrupt()` 方式中止线程，该线程会收到由 `Thread.sleep()` 方法抛出的 `InterruptedException` 异常，由开发者控制捕获到该异常后程序的执行逻辑，而不是像 `Thread.sleep()` 那样直接强制中止线程。

### 正确的线程中止方式 —— 标志位

除了 `Thread.interrupt()` 方式外，还有一种正确的线程中止方式是通过判断标志位，来控制线程执行的中止。

````Java
public class Main {

    public volatile static boolean flag = true;

    public static void main(String[] args) throws InterruptedException {

        Thread thread = new Thread(() -> {
            while (flag) {
                System.out.println("running...");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        thread.start();
        // 等待3秒
        Thread.sleep(3000);

        //设置标志位为false，代表线程中止
        flag = false;
        System.out.println("线程运行结束");
    }
}

````
输出结果：

````Plain Text
running...
running...
running...
线程运行结束
````
上面代码通过共享变量 `flag` 作为标志位，在主线程中改变它的值达到了中止线程的目的。但是这种方式具有局限性，即要求代码业务逻辑中有一个可以用来判断控制线程执行中止的标志位。