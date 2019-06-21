# 2 Mutual Exclusion

相互排斥可能是多处理器编程中最普遍的协调形式。 本章介绍通过读写共享内存工作的经典互斥算法。 尽管这些算法并未在实践中使用，但我们研究它们是因为它们为每个同步领域中出现的各种算法和正确性问题提供了理想的介绍。 本章还提供了不可能性证明。 这个证明告诉我们通过读写共享内存工作的互斥解决方案的局限性，并将有助于激发后面章节中出现的真实互斥算法。 本章是少数包含算法证明的章节之一。 虽然读者可以随意跳过这些证明，但理解它们所呈现的推理类型是非常有帮助的，因为我们可以使用相同的方法来推理后面章节中考虑的实用算法。

## 2.1 Time
关于并发计算的推理主要是关于时间的推理。有时我们希望事情同时发生，有时我们希望它们在不同的时间发生。 我们需要推理涉及多个时间间隔如何重叠的复杂条件，或者有时候，它们如何避免重叠。 我们需要一种简单但毫不含糊的语言来及时讨论事件和持续时间。 日常英语太模糊和不精确。 相反，我们引入了一个简单的词汇表和符号来描述并发线程如何及时表现。


1689年，艾萨克·牛顿（Isaac Newton）表达了“绝对的，真实的和数学的时间本身，从它自身的本质来说，它与外在的东西无关地流动。”我们赞同他的时间概念，即使不是他的散文风格。 线程共享一个公共时间（虽然不一定是公共时钟）。 线程是状态机(state machine)，其状态转换称为事件(events)。

事件是即时的：它们发生在一个瞬间。 要求事件永远不会同时发生是很方便的：不同的事件在不同的时间发生。 （实际上，如果我们不确定两个事件发生在非常接近的时间顺序，那么任何顺序都可以。）线程A产生一系列事件a0，a1，...线程通常包含循环， 所以单个程序语句可以产生许多事件。 我们用ai^j表示事件ai的第j次出现。 一个事件a在另一个事件b之前，如果a发生在较早的时间，则写入a→b。 优先关系“→”是事件的总顺序。

设a0和a1为a0→a1的事件。 间隔（a0，a1）是a0和a1之间的持续时间。 区间IA =（a0，a1）在IB =（b0，b1）之前，写成IA→IB， 如果a1→b0（即，如果IA的最终事件在IB的起始事件之前）。 更简洁地说，→关系是间隔的部分顺序。 与→无关的间隔被认为是并发的。 通过类比事件，我们用IA^J表示区间IA的第j次执行。

## 2.2 Critical Sections

在前面的章节中，我们讨论了图2.1中所示的Counter类实现。我们观察到这种实现在单线程系统中是正确的，但在由两个或多个线程使用时行为不正确。 如果两个线程都在标记为“危险区域开始”的行读取值字段，然后在标记为“危险区域结束”的行更新该字段，则会出现此问题。


```java
class Counter {
  private  in value;
  public Count(int c) {    // Constructor
    value = c;
  }

  // increment and return prior value
  public int getAndIncrement() {
    int temp = value;    // start of danger zone
    value = temp + 1;  // end of danger zone
    return temp;
  }
}
```
**Figure 2.1 The Counter class.**


如果我们将这两行放入到临界区，我们可以避免这个问题：一次只能由一个线程执行的代码块。 我们将这个属性称为互斥。 处理互斥的标准方法是通过实现图2.2所示接口的Lock对象。

```java
public interface Lock {
  public void lock();    // before entering critical section
  public void unlock();  // before leaving critical section
}
```
**Figure 2.2 The Lock interface**


为简洁起见，我们说线程在执行 lock() 方法调用时获取（交替锁定 alternately locks）锁，并在执行 unlock() 方法调用时释放（交替解锁 alternately lokcs）锁。 图2.3显示了如何使用Lock字段向共享计数器实现添加互斥。 使用 lock() 和 unlock() 方法的线程必须遵循特定的格式。 在以下情况下，线程形成良好：

1. 每个关键部分都与一个唯一的Lock对象相关联，
2. 线程在尝试进入临界区时调用该对象的 lock() 方法，并且
3. 线程在离开临界区时调用 unlock() 方法。

```java
public class Counter {
  private long value;
  private Lock lock;    // to protect critical section

  public long getAndIncrement() {
    lock.lock();    // enter critical section
    try {
      long temp = value;   // in critical section
      value = temp + 1;    // in critical section
    } finally {
      lock.unlock();       // leave critical section
    }
    return temp;
  }
}
```
**Figure 2.3 Using a lock object.**

>Parama 2.2.1 在Java中，这些方法需要以下面的结构形式使用。

```java
mutex.lock();
try {
 ... // body
  } finally {
    mutex.unlock();
  }
```

>这个习惯用法确保在进入try块之前获取锁，并且当控制离开块时释放锁，即使块中的某些语句引发意外异常也是如此。


我们现在正式化了一个好的Lock算法应该满足的属性。 设CSjA(CS A下标，j上标)为A在第j次执行临界区间的间隔。 让我们假设，为简单起见，每个线程经常无限地获取和释放锁定，同时进行其他工作。

**Mutual Exclusion** 不同线程的关键部分不重叠。 对于线程A和B，以及整数j和k，CSkA（CS [A下][k上]）→CSjB(CS [B下][K上]或CSjB(CS [B下][j上]→CS kA(CS [A下][k上]。

**Freedom from Deadlock** 如果某个线程尝试获取锁，那么某个线程将成功获取锁。 如果线程A调用 lock() 但从未获取锁，那么其他线程必须在临界区间完成无限次操作。

**Freedom from Starvation** 尝试获取锁的每个线程最终都会成功。 每次调用lock()最终都会返回。 此属性有时称为锁定自由。 

请注意，饥饿自由意味着死锁自由。
