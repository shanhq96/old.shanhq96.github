
# 哲学家进餐问题（Java实现）

[TOC]

----------

## 问题描述

> *哲学家就餐问题可以这样表述，假设有五位哲学家围坐在一张圆形餐桌旁，做以下两件事情之一：吃饭，或者思考。吃东西的时候，他们就停止思考，思考的时候也停止吃东西。餐桌中间有一大碗意大利面，每两个哲学家之间有一只餐叉。因为用一只餐叉很难吃到意大利面，所以假设哲学家必须用两只餐叉吃东西。他们只能使用自己左右手边的那两只餐叉。哲学家就餐问题有时也用米饭和筷子而不是意大利面和餐叉来描述，因为很明显，吃米饭必须用两根筷子。* *（引用维基百科）*

如图所示

![哲学家问题图示](https://lh3.googleusercontent.com/-9jlg1_MKjkg/WaZpLJzVW2I/AAAAAAAAAAM/tRma1fe3ieY_lOfzUFJaYhP_kcjE47BfgCLcBGAs/s500/An_illustration_of_the_dining_philosophers_problem.png "An_illustration_of_the_dining_philosophers_problem.png")


----------


## 问题分析

哲学家从来不交谈，这就很危险，可能产生死锁，每个哲学家都拿着左手的餐叉，永远都在等右边的餐叉（或者相反）。

即使没有死锁，也有可能发生资源耗尽。例如，假设规定当哲学家等待另一只餐叉超过五分钟后就放下自己手里的那一只餐叉，并且再等五分钟后进行下一次尝试。这个策略消除了死锁（系统总会进入到下一个状态），但仍然有可能发生“活锁”。如果五位哲学家在完全相同的时刻进入餐厅，并同时拿起左边的餐叉，那么这些哲学家就会等待五分钟，同时放下手中的餐叉，再等五分钟，又同时拿起这些餐叉。

在实际的计算机问题中，缺乏餐叉可以类比为缺乏共享资源。一种常用的计算机技术是资源加锁，用来保证在某个时刻，资源只能被一个程序或一段代码访问。当一个程序想要使用的资源已经被另一个程序锁定，它就等待资源解锁。当多个程序涉及到加锁的资源时，在某些情况下就有可能发生死锁。例如，某个程序需要访问两个文件，当两个这样的程序各锁了一个文件，那它们都在等待对方解锁另一个文件，而这永远不会发生。


----------


## 约定

本文将叉子替换为筷子，更便于理解。


----------


## 哲学家类

> 首先创建一个类，用以描述哲学家的思考（think）和吃面（eat）的行为。

```java

package cn.rknight.study.diningphiosophers;

import java.text.MessageFormat;

import static java.lang.Thread.sleep;

/**
 * Author: shanhq96@gmail.com
 * DateTime: 2017/8/30 11:00
 * Description:Philosopher has eat() and think()
 */
public class Philosopher {
    private final int id;//哲学家编号

    public Philosopher(int id) {
        this.id = id;
    }

    public int getId() {
        return id;
    }

    /**
     * 哲学家进餐行为
     */
    public void eat() {
        try {
            System.out.println(MessageFormat.format("Philosopher {0} start eating", this.id));
            sleep(500);//假设哲学家进餐需要0.5s的时间
            System.out.println(MessageFormat.format("Philosopher {0} has ate", this.id));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    /**
     * 哲学家思考行为
     */
    public void think() {
        System.out.println(MessageFormat.format("Philosopher {0} start thinking", this.id));
    }
}

```

----------


## 解决思路

> 1.保证当哲学家能同时获取到两根叉子的时候才拿叉子，否则就不拿。
> 2.奇数编号哲学家先拿左手边叉子，偶数号哲学家先拿右手边叉子。

1.保证当哲学家能同时获取到两根叉子的时候才拿叉子，否则就不拿。

```java

package cn.rknight.study.diningphiosophers;

import org.junit.Test;

import java.util.concurrent.Semaphore;

/**
 * Author: shanhq96@gmail.com
 * DateTime: 2017/8/30 11:09
 * Description:保证一个哲学家只有同时拿起两个叉子（筷子）才去拿.
 */
public class PhilosopherMain {
    private static final int EAT_TIMES = 100;
    private Semaphore mutex = new Semaphore(1);
    private Semaphore[] chopsticks = {new Semaphore(1), new Semaphore(1), new Semaphore(1), new Semaphore(1), new Semaphore(1)};
    private Semaphore treadMutex = new Semaphore(0);//通过threadMutex可以保证5位哲学家"公平"的同时开始进行思考或吃饭的活动

    @Test
    public void test() throws InterruptedException {
        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(new PhilosopherThread(i));
            threads[i].start();
        }
        while (true) {
            //当threadMutex信号量中的等待队列的长度达到5的时候，意味着所有的哲学家已经就位，可以开始思考或吃面了。
            if (treadMutex.getQueueLength() == 5) {
                treadMutex.release(5);//释放5个threadMutex，以便让5个哲学家线程可以继续运行。
                break;//退出循环
            }
        }
        for (int i = 0; i < 5; i++) {
            threads[i].join();//使主线程等待所有子线程结束再退出。
        }
    }

    class PhilosopherThread implements Runnable {

        final Philosopher philosopher;

        public PhilosopherThread(int philosopherId) {
            this.philosopher = new Philosopher(philosopherId);//为哲学家赋编号
        }

        public void run() {
            try {
                treadMutex.acquire();//可以让当前线程因为获取不到treadMutex的资源而进入等待状态。
                final int id = this.philosopher.getId();// 当前专家的编号
                for (int i = 0; i < EAT_TIMES; i++) {
                    this.philosopher.think();
                    mutex.acquire();//通过mutex信号量可以让哲学家拿左右筷子的行为变为原子操作。
                    //哲学家获取左右的筷子
                    chopsticks[(id + 1) % 5].acquire();
                    chopsticks[id].acquire();
                    mutex.release();//拿到筷子之后释放mutex的锁，允许其他哲学家去拿筷子。
                    this.philosopher.eat();
                    //哲学家放下左右的筷子
                    chopsticks[(id + 1) % 5].release();
                    chopsticks[id].release();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}

```

> **运行结果**
> 
> Philosopher 1 start thinking
> Philosopher 1 start eating
> Philosopher 4 start thinking
> Philosopher 4 start eating
> Philosopher 0 start thinking
> Philosopher 3 start thinking
> Philosopher 2 start thinking
> Philosopher 1 has ate
> Philosopher 1 start thinking
> Philosopher 4 has ate
> Philosopher 0 start eating
> Philosopher 3 start eating
> ……
> <small>（程序正常运行结束，并未发生死锁，且没有哲学家饿死）</small>

----------

2.奇数编号哲学家先拿左手边叉子，偶数号哲学家先拿右手边叉子。

```java

package cn.rknight.study.diningphiosophers;

import org.junit.Test;

import java.text.MessageFormat;
import java.util.concurrent.Semaphore;

/**
 * Author: shanhq96@gmail.com
 * DateTime: 2017/8/30 14:52
 * Description:奇数哲学家先拿左边的筷子，偶数哲学家先拿右边的筷子.
 */
public class PhilosopherMain2 {
    private static final int EAT_TIMES = 100;
    private Semaphore[] chopsticks = {new Semaphore(1), new Semaphore(1), new Semaphore(1), new Semaphore(1), new Semaphore(1)};
    private Semaphore treadMutex = new Semaphore(0);//通过threadMutex可以保证5位哲学家"公平"的同时开始进行思考或吃饭的活动

    @Test
    public void test() throws InterruptedException {
        Thread[] threads = new Thread[5];
        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(new PhilosopherThread(i));
            threads[i].start();
        }
        while (true) {
            //当threadMutex信号量中的等待队列的长度达到5的时候，意味着所有的哲学家已经就位，可以开始思考或吃面了。
            if (treadMutex.getQueueLength() == 5) {
                treadMutex.release(5);//释放5个threadMutex，以便让5个哲学家线程可以继续运行。
                break;//退出循环
            }
        }
        for (int i = 0; i < 5; i++) {
            threads[i].join();//使主线程等待所有子线程结束再退出。
        }
    }

    class PhilosopherThread implements Runnable {

        final Philosopher philosopher;

        public PhilosopherThread(int philosopherId) {
            this.philosopher = new Philosopher(philosopherId);
        }

        public void run() {
            try {
                treadMutex.acquire();
                final int id = this.philosopher.getId();// 当前专家的编号
                for (int i = 0; i < EAT_TIMES; i++) {
                    this.philosopher.think();
                    if (id % 2 != 0) {
                        //奇数专家先拿左手边筷子，再拿右手边筷子
                        chopsticks[id].acquire();
                        chopsticks[(id + 1) % 5].acquire();
                    } else {
                        //偶数专家先拿右手边筷子，再拿左手边筷子
                        chopsticks[(id + 1) % 5].acquire();
                        chopsticks[id].acquire();
                    }
                    this.philosopher.eat();
                    chopsticks[id].release();
                    chopsticks[(id + 1) % 5].release();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

}

```

> **运行结果**
> 
> Philosopher 0 start thinking
> Philosopher 0 start eating
> Philosopher 4 start thinking
> Philosopher 1 start thinking
> Philosopher 2 start thinking
> Philosopher 2 start eating
> Philosopher 3 start thinking
> Philosopher 0 has ate
> Philosopher 2 has ate
> Philosopher 2 start thinking
> Philosopher 1 start eating
> Philosopher 4 start eating
> Philosopher 0 start thinking
> Philosopher 4 has ate
> ……
> <small>（程序正常运行结束，并未发生死锁，且没有哲学家饿死）</small>

----------