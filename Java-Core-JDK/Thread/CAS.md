## CAS

### CAS在锁中对应的关系

在`JDK 5`之前Java语言是靠`synchronized`关键字保证同步的，这会导致有锁

锁机制存在以下问题：
- 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。
- 一个线程持有锁会导致其它所有需要此锁的线程挂起。
- 如果一个优先级高的线程等待一个优先级低的线程释放锁会导致优先级倒置，引起性能风险。

`volatile`是不错的机制，但是`volatile`不能保证原子性。因此对于同步最终还是要回到锁机制上来。

独占锁是一种悲观锁，`synchronized`就是一种独占锁，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。而另一个更加有效的锁就是乐观锁。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。乐观锁用到的机制就是`CAS`，`Compare and Swap`。

## CAS的原理

### CAS的基本概念

在`Java`发展初期，java语言是不能够利用硬件提供的这些便利来提升系统的性能的。而随着java不断的发展,`Java`本地方法(`JNI`)的出现，使得`java`程序越过`JVM`直接调用本地方法提供了一种便捷的方式，因而`java`在并发的手段上也多了起来。而在`Doug Lea`提供的`cucurenct`包中，`CAS`理论是它实现整个`java`包的基石。

`CAS `操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。 如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值 。否则，处理器不做任何操作。无论哪种情况，它都会在`CAS`指令之前返回该 位置的值。（在`CAS`的一些特殊情况下将仅返回`CAS`是否成功，而不提取当前 值。）`CAS`有效地说明了“我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。”

通常将`CAS`用于同步的方式是从地址 `V` 读取值 `A`，执行多步计算来获得新 值 `B`，然后使用 `CAS` 将 `V` 的值从 `A` 改为 `B`。如果 `V` 处的值尚未同时更改，则 `CAS` 操作成功。

类似于 `CAS` 的指令允许算法执行读-修改-写操作，而无需害怕其他线程同时 修改变量，因为如果其他线程修改变量，那么 `CAS` 会检测它（并失败），算法 可以对该操作重新计算。

### CAS原理

下面是`sun.misc.Unsafe`类的`compareAndSwapInt()`方法的源代码：
```java
public final native boolean compareAndSwapInt(Object o, long offset,
                                              int expected,
                                              int x);
```

可以看到这是个本地方法调用。这个本地方法在`openjdk`中依次调用的`c++`代码为：`unsafe.cpp`，`atomic.cpp`和`atomicwindowsx86.inline.hpp`。这个本地方法的最终实现在`openjdk`的如下位置：`openjdk-7-fcs-src-b147-27jun2011\\openjdk\\hotspot\\src\\oscpu\\windowsx86\\vm\\ atomicwindowsx86.inline.hpp`（对应于`windows`操作系统，`X86`处理器）。下面是对应于`intel x86`处理器的源代码的片段：

```assembly

// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}

```

如上面源代码所示，程序会根据当前处理器的类型来决定是否为`cmpxchg`指令添加`lock`前缀。如果程序是在多处理器上运行，就为`cmpxchg`指令加上`lock`前缀（`lock cmpxchg`）。反之，如果程序是在单处理器上运行，就省略`lock`前缀（单处理器自身会维护单处理器内的顺序一致性，不需要`lock`前缀提供的内存屏障效果）。



确保对内存的读-改-写操作原子执行。在`Pentium`及`Pentium`之前的处理器中，带有`lock`前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从`Pentium 4`，`Intel Xeon`及`P6`处理器开始，`intel`在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（`area of memory`）在`lock`前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（`cache line`）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（`cache locking`），缓存锁定将大大降低`lock`前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。

同时，他会禁止该指令与之前和之后的读和写指令重排序，把写缓冲区中的所有数据刷新到内存中。

#### CPU的锁类型
##### 1. 处理器自动保证基本内存操作的原子性
首先处理器会自动保证基本的内存操作的原子性。处理器保证从系统内存当中读取或者写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。奔腾6和最新的处理器能自动保证单处理器对同一个缓存行里进行16/32/64位的操作是原子的，但是复杂的内存操作，处理器不能自动保证其原子性，比如跨总线宽度，跨多个缓存行，跨页表的访问。但是处理器提供总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性。

##### 2. 使用总线锁保证原子性
第一个机制是通过总线锁保证原子性。如果多个处理器同时对共享变量进行读改写（`i++`就是经典的读改写操作）操作，那么共享变量就会被多个处理器同时进行操作，这样读改写操作就不是原子的，操作完之后共享变量的值会和期望的不一致，举个例子：如果`i=1`,我们进行两次`i++`操作，我们期望的结果是3，但是有可能结果是2。原因是有可能多个处理器同时从各自的缓存中读取变量i，分别进行加一操作，然后分别写入系统内存当中。那么想要保证读改写共享变量的操作是原子的，就必须保证`CPU1`读改写共享变量的时候，`CPU2`不能操作缓存了该共享变量内存地址的缓存。

处理器使用总线锁就是来解决这个问题的。所谓总线锁就是使用处理器提供的一个`LOCK＃`信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住,那么该处理器可以独占使用共享内存。

##### 3. 使用缓存锁保证原子性
第二个机制是通过缓存锁定保证原子性。在同一时刻我们只需保证对某个内存地址的操作是原子性即可，但总线锁定把CPU和内存之间通信锁住了，这使得锁定期间，其他处理器不能操作其他内存地址的数据，所以总线锁定的开销比较大，最近的处理器在某些场合下使用缓存锁定代替总线锁定来进行优化。

频繁使用的内存会缓存在处理器的`L1`，`L2`和`L3`高速缓存里，那么原子操作就可以直接在处理器内部缓存中进行，并不需要声明总线锁，在奔腾6和最近的处理器中可以使用“缓存锁定”的方式来实现复杂的原子性。所谓“缓存锁定”就是如果缓存在处理器缓存行中内存区域在`LOCK`操作期间被锁定，当它执行锁操作回写内存时，处理器不在总线上声言`LOCK＃`信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改被两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时会起缓存行无效，在例1中，当`CPU1`修改缓存行中的`i`时使用缓存锁定，那么`CPU2`就不能同时缓存了i的缓存行。

但是有两种情况下处理器不会使用缓存锁定。第一种情况是：当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行（`cache line`），则处理器会调用总线锁定。第二种情况是：有些处理器不支持缓存锁定。对于Inter486和奔腾处理器,就算锁定的内存区域在处理器的缓存行中也会调用总线锁定。

以上两个机制我们可以通过`Inter`处理器提供了很多LOCK前缀的指令来实现。比如位测试和修改指令`BTS`，`BTR`，`BTC`，交换指令`XADD`，`CMPXCHG`和其他一些操作数和逻辑指令，比如`ADD`（加），`OR`（或）等，被这些指令操作的内存区域就会加锁，导致其他处理器不能同时访问它。



### CAS解决的问题
利用`CPU`的`CAS`指令，同时借助`JNI`来完成`Java`的非阻塞算法。其它原子操作都是利用类似的特性完成的。而整个`J.U.C`都是建立在`CAS`之上的，因此对于`synchronized`阻塞算法，`J.U.C`在性能上有了很大的提升。

### CAS存在的问题

`CAS`虽然很高效的解决原子操作，但是`CAS`仍然存在三大问题。`ABA`问题，循环时间长开销大和只能保证一个共享变量的原子操作

1. `ABA`问题。因为`CAS`需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是`A`，变成了`B`，又变成了`A`，那么使用`CAS`进行检查时会发现它的值没有发生变化，但是实际上却变化了。`ABA`问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么`A－B－A` 就会变成`1A-2B－3A`。</br>
从`Java1.5`开始`JDK`的`atomic`包里提供了一个类`AtomicStampedReference`来解决`ABA`问题。这个类的`compareAndSet`方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。</br>
关于`ABA`问题参考: [这里]:http://blog.hesey.net/2011/09/resolve-aba-by-atomicstampedreference.html

2. 循环时间长开销大。自旋`CAS`如果长时间不成功，会给`CPU`带来非常大的执行开销。如果`JVM`能支持处理器提供的`pause`指令那么效率会有一定的提升，`pause`指令有两个作用，第一它可以延迟流水线执行指令（`de-pipeline`）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（`memory order violation`）而引起`CPU`流水线被清空（`CPU pipeline flush`），从而提高`CPU`的执行效率。



3. 只能保证一个共享变量的原子操作。当对一个共享变量执行操作时，我们可以使用循环`CAS`的方式来保证原子操作，但是对多个共享变量操作时，循环`CAS`就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量`i＝2`,`j=a`，合并一下`ij=2a`，然后用`CAS`来操作`ij`。从`Java1.5`开始`JDK`提供了`AtomicReference`类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行`CAS`操作。


### 在concurrent包下的实现

由于`java`的`CAS`同时具有 `volatile` 读和`volatile`写的内存语义，因此`Java`线程之间的通信现在有了下面四种方式：

`A`线程写`volatile`变量，随后`B`线程读这个`volatile`变量。
`A`线程写`volatile`变量，随后`B`线程用`CAS`更新这个`volatile`变量。
`A`线程用`CAS`更新一个`volatile`变量，随后`B`线程用`CAS`更新这个`volatile`变量。
`A`线程用`CAS`更新一个`volatile`变量，随后`B`线程读这个`volatile`变量。
`Java`的`CAS`会使用现代处理器上提供的高效机器级别原子指令，这些原子指令以原子方式对内存执行读-改-写操作，这是在多处理器中实现同步的关键（从本质上来说，能够支持原子性读-改-写指令的计算机器，是顺序计算图灵机的异步等价机器，因此任何现代的多处理器都会去支持某种能对内存执行原子性读-改-写操作的原子指令）。同时，`volatile`变量的读/写和`CAS`可以实现线程之间的通信。把这些特性整合在一起，就形成了整个`concurrent`包得以实现的基石。如果我们仔细分析`concurrent`包的源代码实现，会发现一个通用化的实现模式：

首先，声明共享变量为v`olatile`；
然后，使用`CAS`的原子条件更新来实现线程之间的同步；
同时，配合以`volatile`的读/写和`CAS`所具有的`volatile`读和写的内存语义来实现线程之间的通信。


## JUC并发框架下的原子类(atomic)
> 注意，本文章引用自[Java并发_CAS原理分析][1],如果想查看更详细的部分，请点击前往。

调用`JUC`并发框架下原子类的方法时，不需要考虑多线程问题。那么我们分析它是怎么解决多线程问题的。以`AtomicInteger`类为例


### 成员变量

```Java
// 通过它来实现CAS操作的。因为是int类型，所以调用它的compareAndSwapInt方法
private static final Unsafe unsafe = Unsafe.getUnsafe();

// value这个共享变量在AtomicInteger对象上内存偏移量，
// 通过它直接在内存中修改value的值，compareAndSwapInt方法中需要这个参数
private static final long valueOffset;

// 通过静态代码块，在AtomicInteger类加载时就会调用
static {
    try {
        // 通过unsafe类，获取value变量在AtomicInteger对象上内存偏移量
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

// 共享变量，AtomicInteger就保证了对它多线程操作的安全性。
// 使用volatile修饰，解决了可见性和有序性问题。
private volatile int value;
```
有三个重要的属性：
- `unsafe`: 通过它实现`CAS`操作，因为共享变量是`int`类型，所以调用`compareAndSwapInt`方法。
- `valueOffset`: 共享变量`value`在`AtomicInteger`对象上内存偏移量
- `value`: 共享变量，使用`volatile`修饰，解决了可见性和有序性问题。

### 重要方法

#### 1. get与set方法
```Java
// 直接读取。因为是volatile关键子修饰的，总是能看到(任意线程)对这个volatile变量最新的写入
public final int get() {
    return value;
}

// 直接写入。因为是volatile关键子修饰的，所以它修改value变量也会立即被别的线程读取到。
public final void set(int newValue) {
    value = newValue;
}

```

因为`value`变量是`volatile`关键字修饰的，它总是能读取(任意线程)对这个`volatile`变量最新的写入。它修改`value`变量也会立即被别的线程读取到。

#### 2. compareAndSet方法
```Java
// 如果value变量的当前值(内存值)等于期望值(expect)，那么就把update赋值给value变量，返回true。
// 如果value变量的当前值(内存值)不等于期望值(expect)，就什么都不做，返回false。
// 这个就是CAS操作，使用unsafe.compareAndSwapInt方法，保证整个操作过程的原子性
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

通过调用`unsafe`的`compareAndSwapInt`方法实现`CAS`函数的。但是`CAS`函数只能保证比较并交换操作的原子性，但是更新操作并不一定会执行。比如我们想让共享变量`value`自增。
共享变量`value`自增是三个操作，1.读取`value`值，2.计算`value+1`的值，3.将`value+1`的值赋值给`value`。分析这三个操作：

读取`value`值,因为`value`变量是`volatile`关键字修饰的，能够读取到任意线程对它最后一次修改的值，所以没问题。
计算`value+1`的值：这个时候就有问题了，可能在计算这个值的时候，其他线程更改了`value`值，因为没有加同步锁，所以其他线程可以更改`value`值。

将`value+1`的值赋值给`value`: 使用`CAS`函数，如果返回`false`，说明在当前线程读取`value`值到调用`CAS`函数方法前，共享变量被其他线程修改了，那么`value+1`的结果值就不是我们想要的了，因为要重新计算。

#### 3. getAndAddInt方法
```Java
 public final int getAndAddInt(Object obj, long valueOffset, int var) {
    int expect;
    // 利用循环，直到更新成功才跳出循环。
    do {
        // 获取value的最新值
        expect = this.getIntVolatile(obj, valueOffset);
        // expect + var表示需要更新的值，如果compareAndSwapInt返回false，说明value值被其他线程更改了。
        // 那么就循环重试，再次获取value最新值expect，然后再计算需要更新的值expect + var。直到更新成功
    } while(!this.compareAndSwapInt(obj, valueOffset, expect, expect + var));

    // 返回当前线程在更改value成功后的，value变量原先值。并不是更改后的值
    return expect;
}
```

这个方法在`Unsafe`类中，利用`do_while`循环，先利用当前值，计算更新值，然后通过`compareAndSwapInt`方法设置`value`变量，如果`compareAndSwapInt`方法返回失败，表示`value`变量的值被别的线程更改了，所以循环获取`value`变量最新值，再通过`compareAndSwapInt`方法设置`value`变量。直到设置成功。跳出循环，返回更新前的值。

```Java
// 将value的值当前值的基础上加1，并返回当前值
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

// 将value的值当前值的基础上加-1，并返回当前值
public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}


// 将value的值当前值的基础上加delta，并返回当前值
public final int getAndAdd(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta);
}


// 将value的值当前值的基础上加1，并返回更新后的值(即当前值加1)
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// 将value的值当前值的基础上加-1，并返回更新后的值(即当前值加-1)
public final int decrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}

// 将value的值当前值的基础上加delta，并返回更新后的值(即当前值加delta)
public final int addAndGet(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}
```
都是利用`unsafe.getAndAddInt`方法实现的。

#### 实例：

```Java
class Data {
    AtomicInteger num;

    public Data(int num) {
        this.num = new AtomicInteger(num);
    }

    public int getAndDecrement() {
        return num.getAndDecrement();
    }
}

class MyRun implements Runnable {

    private Data data;
    // 用来记录所有卖出票的编号
    private List<Integer> list;
    private CountDownLatch latch;

    public MyRun(Data data, List<Integer> list, CountDownLatch latch) {
        this.data = data;
        this.list = list;
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            action();
        }  finally {
            // 释放latch共享锁
            latch.countDown();
        }
    }

    // 进行买票操作，注意这里没有使用data.num>0作为判断条件，直到卖完线程退出。
    // 那么做会导致这两处使用了共享变量data.num，那么做多线程同步时，就要考虑更多条件。
    // 这里只for循环了5次，表示每个线程只卖5张票，并将所有卖出去编号存入list集合中。
    public void action() {
        for (int i = 0; i < 5; i++) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            int newNum = data.getAndDecrement();

            System.out.println("线程"+Thread.currentThread().getName()+"  num=="+newNum);
            list.add(newNum);
        }
    }
}

public class ThreadTest {


    public static void startThread(Data data, String name, List<Integer> list,CountDownLatch latch) {
        Thread t = new Thread(new MyRun(data, list, latch), name);
        t.start();
    }

    public static void main(String[] args) {
        // 使用CountDownLatch来让主线程等待子线程都执行完毕时，才结束
        CountDownLatch latch = new CountDownLatch(6);

        long start = System.currentTimeMillis();
        // 这里用并发list集合
        List<Integer> list = new CopyOnWriteArrayList();
        Data data = new Data(30);
        startThread(data, "t1", list, latch);
        startThread(data, "t2", list, latch);
        startThread(data, "t3", list, latch);
        startThread(data, "t4", list, latch);
        startThread(data, "t5", list, latch);
        startThread(data, "t6", list, latch);


        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        // 处理一下list集合，进行排序和翻转
        Collections.sort(list);
        Collections.reverse(list);
        System.out.println(list);

        long time = System.currentTimeMillis() - start;
        // 输出一共花费的时间
        System.out.println("\n主线程结束 time=="+time);
    }
}

```




[1]:https://www.jianshu.com/p/a142350e9b7a
