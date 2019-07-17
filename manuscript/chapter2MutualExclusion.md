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




![paragraph 2-1](https://github.com/mysonhushu/the-art-of-multiprocessor-programming-zh/blob/master/manuscript/images/pragraph2-1.png "pragraph 2.1")


我们现在正式化了一个好的Lock算法应该满足的属性。 设CSjA(CS A下标，j上标)为A在第j次执行临界区间的间隔。 让我们假设，为简单起见，每个线程经常无限地获取和释放锁定，同时进行其他工作。

**Mutual Exclusion** 不同线程的关键部分不重叠。 对于线程A和B，以及整数j和k，CSkA（CS [A下][k上]）→CSjB(CS [B下][K上]或CSjB(CS [B下][j上]→CS kA(CS [A下][k上]。

**Freedom from Deadlock** 如果某个线程尝试获取锁，那么某个线程将成功获取锁。 如果线程A调用 lock() 但从未获取锁，那么其他线程必须在临界区间完成无限次操作。

**Freedom from Starvation** 尝试获取锁的每个线程最终都会成功。 每次调用lock()最终都会返回。 此属性有时称为锁定自由。 

请注意，饥饿自由意味着死锁自由。

互斥属性显然是必不可少的。如果没有此属性，我们无法保证计算结果是正确的。在第1章的术语中，互斥是一种安全属性。死锁自由(deadlock-freedom)属性很重要。这意味着系统永远不会“冻结”(freezes)。个别线程可能永远陷入困境（称为饥饿 starvation），但某些线程会取得进展。在第1章的术语中，死锁自由(deadlock-freedom)是一种活跃的属性。请注意，即使它使用的每个锁都满足deadlock-freedom属性，程序仍然可以死锁。例如，考虑共享锁0和1的线程A和B.首先，A获取0，B获取1.接下来，A尝试获取1，B尝试获取0.线程死锁，因为每个线程等待另一个释放它的锁。

饥饿自由属性(starvation-freedom property),虽然明显可取，是这三个当中最强迫的。稍后，我们将看到实际的互斥算法无法做到饥饿自由。这些算法通常是基于理想的饥饿自由情况下发展的，但实际上是不太可能发生的。然而，理解饥饿的能力对于理解它是否是一个现实的威胁至关重要。

饥饿自由的属性同样很弱。从某种意义上说，它无法确保一个线程在等待一段时间后就一定能进入临界区。稍后，我们将研究限制线程可以等待多长时间的算法。

## 2.3 2-Thread Solutions

我们从两个不充分但有趣的Lock算法开始。

### 2.3.1 The LockOne Class

图2.4显示了LockOne算法。我们的双线程锁定算法遵循以下约定：线程有0和1，调用线程有i，另一个 j = 1  -  i。每个线程通过调用 ThreadID.get() 获取其索引

```java
class LockOne implements Lock {
  private boolean[] flag = new boolean[2];
  // thread-local index, 0 or 1
  public void lock() {
    int i = ThreadID.get();
    int j = 1 - i;
    flag[i] = true;
    while (flag[j]) {} // wait
  }

  public void unlock() {
    int i = ThreadID.get();
    flag[i] = false;
  }

}
```
**Figure 2.4 The LockOne algorithm.**

>Pragma 2.3.1 实际上，图2.4 中的布尔标志变量以及后面算法中的 victim 和 label 变量，都必须声明为 **volatile** 才能正常工作。我们将在第３章和附录A中解释原因。

我们使用 write_A(x=v)来表示A将值v赋值给字段x的事件。然后用read_A(v == x) 来表示A从字段x读取值v的事件。当v这个值不重要的时候，有时会被忽略。例如，在图2.4中，事件 write_A(flag[i] = true) 被 lock() 方法的第7行调用时。

>引理 2.3.1. LockOne 算法满足互相排斥。

证明: 假设不满足，那么必然存在整数 j 和 k, 满足 CS_A^J 不触发 CS_B^k 并且 CS_B^k 不触发 CS_A^J。　考虑每一个线程在它的第 k次(j次) 进入临界区前最后一次执行　lock() 方法。

观察代码，我们将发现：

 
![formula 2-3](https://github.com/mysonhushu/the-art-of-multiprocessor-programming-zh/blob/master/manuscript/images/formula2-3.png "formula 2.3")


![formula 2-3-4](https://github.com/mysonhushu/the-art-of-multiprocessor-programming-zh/blob/master/manuscript/images/formula2-3-4.png "formula 2.3.4")

请注意，一旦 flag[B] 被设置为true, 它的值就会变为true. 根据公式 Eg.2.3.3, 否则，线程A不能将读取标志[B]视为false。

接下来，write_A（flag [A] = true）→r read_B（flag [A] == false）没有插入对flag []数组的插入，这是一个矛盾。

LockOne算法是不合适的，因为它在线程执行时会死锁是交错的。如果writeA（flag [A] = true）和writeB（flag [B] = true）事件发生在readA（flag [B]）和readB（flag [A]）事件之前，则两个线程都会永远等待。然而，LockOne有一个有趣的属性：如果一个线程在另一个线程之前运行，则不会发生死锁，一切都很好。


### 2.3.2 The LockTwo Class

图2.5 显示了另外一种lock算法，LockTwo.

```java
class LockTwo implements Lock {
  private volatile int victim;
  public void lock() {
    int i = ThreadID.get();
    victim = i;        // let the other go first
    while ( victim == 1) {} //waite
  }

  public void unlock() {}
}
```


![Lemma 2.3.2](https://github.com/mysonhushu/the-art-of-multiprocessor-programming-zh/blob/master/manuscript/images/Lemma2-3-2.png "Lemma 2.3.2")

LockTwo类是不合适的，因为如果一个线程完全在另一个线程之前运行它会死锁。然而，LockTwo有一个有趣的属性：如果线程并发运行，则 lock() 方法成功。 LockOne和LockTwo类相互补充：每个类都会在导致另一个死锁的条件下成功。

### 2.3.3 The Peterson Lock

我们现在将LockOne和LockTwo算法结合起来构建一个starvation-free Lock算法，如图2.6所示。该算法可以说是最紧凑，优雅的双线互斥算法。它的发明者之后被称为“彼得森的算法”(Peterson's Algorithm)。

```java
class Peterson implements Lock {
  // thread-local index, 0 or 1 
  private volatile boolean[] flag = new boolean[2];
  private volatile int victim;
  public void lock() {
    int i = ThreadID.get();
    int j = 1 - i;
    flag[i] = true; // I'm interested
    victim = i;     // you go first
    while(flag[j] && victim == i) {}; //wait
  }
  public void unlock() {
    int i = ThreadID.get();
    flag[i] = flase; // I'm not interested
  }
}

```
**Figure 2.6 The Peterson lock algorithm.**


>Lemma2.3.3 The Peterson lock algorithm satisfies mutual exclusion.

证明：假设不成立，像上面那样，考虑线程A和线程B最后一次执行方法lock(). 检查代码，我们会发现。

```java
write_A(flag[A] = true)->      (2.3.8)
write_A(victim =A) -> read_A(flag[B]) -> read_A(victim) -> CS_A
write_B(flag[B]=true) -> write_B(victim=B) -> write_B(victim=B) -> read_B(flag[A])->read_B(victim)->CS_B      (2.3.9)
```

假设，在不失一般性的情况下，A是写入的最后一个线程 victimfield 领域。

```java
write_B(victim = B) -> write_A(victime=A)   (2.3.10)
```

方程式2.3.10 暗示A观察到的 victim 是方程式2.3.8中A的victim。由于A仍然进入其临界区，因此必须观察到标志[B]为假， 所以我们有

```java
write_A(victim=A)->read_A(flag[B] == false)    (2.3.11)
```

方程式2.3.9和方程式2.3.11,将链接符号放在一起，实现如方程式2.3.12.
```java
write_B(flag[B]=true)->write_B(victim=B)->write_A(victim=A) -> read_A(flag[B] == false)   (2.3.12)
```

接下来是 write_B(flag [B] = true)→ read_A(flag [B] == false)。这种观察产生了矛盾，因为在关键部分执行之前没有执行对标志[B]的其他写入。

>Lemma 2.3.4 The Peterson lock algorithm is starvation-free.

证明: 假设没有。假设（不失一般性）A在 lock() 方法中永远运行。它必须执行while语句，等待 flag[B] 变为 false 或者 victim 被设置为B.

当A未能取得进展时，B在做什么？也许B反复进入和离开其关键部分。但是，如果是这样，那么B很快就会将 victim 设置为 B.因为它重新进入临界区域。一旦 victim 被设置为B，它就不会改变，并且A必须最终从 lock() 方法返回，这是一个矛盾。

所以一定是B也卡在它的 lock() 方法调用中，等待标志[A]变为假或者受害者被设置为A.但受害者不能同时是A和B，这是一个矛盾。

>推论 2.3.1 Peterson 锁定算法没有死锁(The Peterson lock algorithm is dealock-free.)

## 2.4 The Filter Lock

我们现在考虑将基于两个现场的互斥协议应用于 N 个线程，这里的N大于2。第一种解决方法，Filter lock, 是Peterson锁定到多个线程的直接概括。第二个解决方案，即Bakery锁，可能是最简单和最知名的n-thread解决方案。

Filter lock, 如图 2.7 所示，创建 n-1　个"waiting rooms", 称之为等级(levels),一个线程，在获取锁之前，必须先遍历这些等级，这些等级在图2.8中有描述，每个等级满足两个重要的属性：

- 至少有一个线程尝试进入该等级并成功。 
- 如果超过了一个线程尝试进入等级,那么至少有一个线程是阻塞的(例如：继续等待那个等级).



![Figure 2.7 The Filter lock algorithm](https://github.com/mysonhushu/the-art-of-multiprocessor-programming-zh/blob/master/manuscript/images/Figure2-7.png "Figure 2.7 The Filter lock algorithm")



![Figure 2.8](https://github.com/mysonhushu/the-art-of-multiprocessor-programming-zh/blob/master/manuscript/images/Figure2-8.png "Figure 2.8")

Peterson锁使用包含两布尔标志元素的数组来指示线程是否正在尝试进入临界区。Filter锁定使用n元素 integer level []数组来概括此概念，其中level [A] 的值表示线程A尝试进入的最高级别。每个线程必须通过 n-1 级别的“排除”才能进入其关键部分。每个级别s 都有一个明显的 victim[s] 字段，用于“过滤掉”一个线程，将其从下一级别排除。


最初，线程A处于0级。我们说A在 j>0 时处于j级，当它最后在第17行用 level[A]>=j 完成等待循环时。 （所以j级的线程也是在 j-1 级，依此类推。）

>Lemma 2.4.1 For j between 0 and n - 1, there are at most n - j threads at level j.

证明: 通过对j的归纳。基本情况，其中j = 0，是微不足道的。对于归纳步骤，归纳假设意味着在j-1级最多有n-j + 1个线程。为了表明至少有一个线程不能进入j级，我们通过矛盾来论证：假设有n-j+1个线程在level j 。


![Lemma 2.4.1](https://github.com/mysonhushu/the-art-of-multiprocessor-programming-zh/blob/master/manuscript/images/Figure2-8.png "Lemma 2.4.1")


Perterson锁使用双元素bool标志数组来指示线程是否正在尝试进入临界区。Filter锁定使用n元素integer level []数组来概括此概念，其中level [A]的值表示线程A尝试输入的最高级别。每个线程必须通过n  -  1级别的“排除”才能进入其关键部分。 每个级别 e 都有一个明显的victim [e] 字段，用于“过滤掉”一个线程，将其从下一级别排除。
最初，线程A处于0级。我们说A在j> 0时处于j级，当它最后在第17行用等级[A] j完成等待循环时。 （所以级别j的线程也在级别j  -  1，依此类推。）

引理 2.4.1. 位于 0 和 n - 1 之间的 j 来说，Level j 等级最多有 n-1 个线程。
































