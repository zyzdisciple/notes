# Java内存模型

<font face="微软雅黑">

这是《深入理解Java虚拟机》的第十二章, 在之前内存区域篇章, 已经略微提到过这个概念. 因为往往有人对 内存区域 内存模型, 概念理解略有偏差.

在[java 工作内存](https://www.cnblogs.com/yujian-bcq/p/4583451.html)这篇文章里, 对Java的内存区域划分, 和 Java内存模型这两个概念解释的比较清楚, 这是从两个角度去看待Java中变量的存储方式, 不需要强行拿来比较. 不太合适.

JVM的静态内存储模型(内存区域)只是一种对内 存的物理划分而已，它只局限在内存, 而计算机不仅仅只有内存.

## 序言

在这之前先了解一点课外知识:

[CPU与内存的那些事](https://www.cnblogs.com/zhangj95/p/5647051.html)

CPU的执行速度很快, 那么到底有多快呢? 在Core 2 3.0GHz上，大部分简单指令的执行只需要一个时钟周期，也就是1/3纳秒。即使是真空中传播的光，在这段时间内也只能走10厘米(约4英寸), 所以在有关程序优化方面, 最重要的一点是: 从各个方面来确定是否需要进行优化, 而几个指令的优化在初始设计上, 是没有必要考虑的.

当CPU运转起来之后, 它便会通过L1 cache和L2 cache对系统中的主存进行读写访问。

而访问速度呢? 我们把CPU的一个时钟周期看作一秒。那么，从L1 cache读取信息就好像是拿起桌上的一张草稿纸（3秒）；从L2 cache读取信息则是从身边的书架上取出一本书（14秒）；而从主存中读取信息则相当于走到办公楼下去买个零食（4分钟）。而硬盘的寻道操作( 也就是在磁盘表面移动读写磁头到正确的磁道上，然后再等待磁盘旋转到正确的位置上，以便读取指定扇区内的信息。)需要等待的时间则是相当于离开办公大楼并开始长达一年零三个月的环球旅行.

而L1 和 L2cache是什么呢?

参见链接:
[CPU的缓存L1,L2,L3](https://blog.csdn.net/myxmu/article/details/17021975)

也就是高速缓冲存储器, 从内存中读取数据的速度, 与 CPU的执行效率相较而言, 实在是差距太大, 因此插入了高速缓存这个设计. 将内存中读取到的数据存储在 高速缓存中, 需要时直接从高速缓存中读取, 如果依次在 L1, L2中都读取不到, 则从内存中对数据进行读取, 加载至高速缓存中. 至于缓存命中率等等问题, 暂时就不在考虑范围内了.

而当将数据存储在高速缓存中之后呢? 将运算需要使用到的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存之中没这样处理器就无需等待缓慢的内存读写了。

但是引入了一个新的问题：缓存一致性（Cache Coherence）。在多处理器系统中，每个处理器都有自己的高速缓存，而他们又共享同一主存, 当回写入主存时, 究竟以谁的数据为准? 这就需要协议来进行约束. 缓存一致性协议.

而什么是内存模型呢?

在特定的操作协议下， 对特定的内存或高速缓存进行读写的过程抽象。换句话说， 就是对变量的读写规则的定义，对于常用的计算机而言， 就是变量在CPU， 高速缓存， 内存中的一个流转过程。

## Java的内存模型

而与之类似的， Java的内存模型主要目的是定义Java程序中各个变量的访问规则, 它包括了实例字段, 静态字段, 构成数组对象的元素, 而不包含 局部变量 方法参数, 后者为线程私有, 不会存在竞争问题.

在Java虚拟机中, 内存分为主内存, 及工作内存:
主内存是所有的线程所共享的, 对应的是物理硬件的内存, 而工作内存则是cpu的寄存器和高速缓存的抽象描述。

而规范 JSR-133:Java内存模型与线程规范 中有更为详细的, 明确的规范, 我个人看得云里雾里.

PDF版链接:[JSR133中文版](http://ifeve.com/wp-content/uploads/2014/03/JSR133%E4%B8%AD%E6%96%87%E7%89%881.pdf)

而事实上, 目前仅仅需要了解, 该如何判断 程序 是否是线程安全的, 如果不安全, 又是因为什么原因导致的?

### 线程不安全

原子性: 一个操作是不可中断的，要么全部执行成功要么全部执行失败，有着“同生共死”的感觉。

可见性: 可见性是指当一个线程修改了共享变量后，其他线程能够立即得知这个修改。但这里的立即得知又有不同, 并非是主动通知, 而是当需要读的时候, 能够拿到这个变量的最新值, 所以才会存在 volatile 并不能够保证线程安全.

示例代码:

    public class Main {

        public int i = 0;

        public static final Object obj = new Object();

        public void increment() {
            synchronized (obj) {
                this.i++;
            }
        }

        public static void main(String[] args) {
            Main main = new Main();
            for (int i = 0; i < 200; i++) {
                new Thread(() ->{
                    for (int j = 0; j < 1000; j++) {
                        main.increment();

                    }
                }).start();
            }
            //如果注释掉下面这段循环, 毫无疑问可以保证结果永远为200000
            for (int i = 0; i < 20; i++) {
                new Thread(() ->{
                    for (int j = 0; j < 1000; j++) {
                        System.out.println(Thread.activeCount());
                        main.i++;
                    }
                }).start();
            }
            //我是通过Idea直接执行, 活动线程最低为2.
            while (Thread.activeCount() > 2) {
                Thread.yield();
            }
            System.out.println(main.i);
        }

    }

所以在多个地方都可以对共享变量更改的时候, 最好调用对应Class内部的更改方法, 同时在内部方法上加锁即可, 否则即使在当前方法体内加锁, 保证多线程执行当前方法时, 不存在安全问题, 但其他地方同样拥有对变量的修改权限时, 依然会导致问题.

同样的, 延伸开来来看, 当需要从数据库取数据, 判断, 然后更新这种操作时, 在我目前看来最优的方式依然是将操作封装, 在其他地方不能够直接对 相应数据进行更新, 即使在数据库本身 repeatable read 事务模式下, 依然不能够保证数据的正确性. 粗暴的加锁, 仅能解决当前问题.

有序性: 指的是指令重排序所导致的问题, 代码并不一定会按照其本身的顺序被执行.

指令重排序:

大多数现代微处理器都会采用将指令乱序执行（out-of-order execution，简称OoOE或OOE）的方法.

在条件允许的情况下，直接运行当前有能力立即执行的后续指令，避开获取下一条指令所需数据时造成的等待。

通过乱序执行的技术，处理器可以大大提高执行效率。

除了处理器，常见的Java运行时环境的JIT编译器也会做指令重排序操作，即生成的机器指令与字节码指令顺序不一致。

    boolean initialized = false;
    //以下在 线程A中执行
    threadA.doSth();
    initialized = true;

    //以下则在线程B中执行
    while(!initialized) {
        sleep();
    }
    threadB.doSthAfterA();

在这样的代码中, 有可能就会出现 threadA.doSth() 尚未执行, 而threadB.doSthAfterA() 已经执行, 为什么呢?(在JIT编译之后, 生成本地代码, 有可能会被对应平台的处理器进行指令重排序) 就是因为指令重排序的存在. 在单线程环境中, threadA.doSth() 并不依赖 initialized; 所以将 initialized = true 放在 threadA.doSth() 在计算机看来也是完全可行的. 但这样就会导致对应的问题.

三大特性就是原子性, 可见性, 有序性.

### volatile的特殊性

这个关键字用于保证当前属性 的 可见性, 以及能够起到禁止指令重排序的作用.

可见性无需多言, 在上面已经提到过了. 而同样的, 仅保证可见性, 而不保证原子性, 错误的操作依然会导致线程不安全.

volatile关键字, 仅保证, 当数据被更改时 会被立即更新到内存中去, 而并不能保证上次获取的数据是最新数据.

    public volatile int a;

当有threadA 读取A 并进行 ++ 操作时, 分为几步, 读取A, 将值放入栈中, 取出栈顶数 ++ 后 放回栈顶;

volatile仅保证在第一步, 读取的时候读取到的数据一定是最新值, 而在其后的几步操作则无法保证, 同时也保证, 在更新值时 会立即将数据更新至内存.

所以 volatile的适用范围:

1. 运算结果并不依赖当前值, 或确保只有单一的线程能够对值进行修改.

1. 变量不需要与其他变量一起参与不变约束.(这句话在我理解来是这样的:)

        public volatile boolean a = false;

        if (a && conditionA)
            doSth();

    这里的 a && conditionA 当在判定条件中, 在这种条件中就类似于进行了如下操作

        boolean result = a && conditionA;

    最终result的值, 依赖了 a的当前值.

那么第二点, 有序性的保证:

参考: [指令重排序，内存模型排序规则，内存屏障](https://www.cnblogs.com/straybirds/p/8612952.html)

指令重排序的原因, 原理等等可以参考: [指令重排序](https://www.jianshu.com/p/c6f190018db1)

内存屏障:

内存屏障（Memory Barrier，或有时叫做内存栅栏，Memory Fence）是一种CPU指令，是CPU或编译器在对内存随机访问的操作中的一个同步点，使得此点之前的所有读写操作都执行后才可以开始执行此点之后的操作。

常见的x86/x64，通常使用lock指令前缀加上一个空操作来实现内存屏障，注意当然不能真的是nop指令，但是可以用来实现空操作的指令其实是很多的，比如Linux中采用的
1
addl $0, 0 (%esp)

Java编译器也会根据内存屏障的规则禁止重排序。

内存屏障可以被分为以下几种类型：

LoadLoad  屏障：对于这样的语句Load1; LoadLoad; Load2，在Load2及后续读取操作要读取的数据被访问前，保证Load1要读取的数据被读取完毕。

StoreStore屏障：对于这样的语句Store1; StoreStore; Store2，在Store2及后续写入操作执行前，保证Store1的写入操作对其它处理器可见。

LoadStore 屏障：对于这样的语句Load1; LoadStore; Store2，在Store2及后续写入操作被刷出前，保证Load1要读取的数据被读取完毕。

StoreLoad 屏障：对于这样的语句Store1; StoreLoad; Load2，在Load2及后续所有读取操作执行前，保证Store1的写入对所有处理器可见。

它的开销是四种屏障中最大的。在大多数处理器的实现中，这个屏障是个万能屏障，兼具其它三种内存屏障的功能。

通过这种方式, 就达到了禁止指令重排序.

    //通过这两个参数, 输出JIT编译后的汇编代码
    -XX:+UnlockDiagnosticVMOptions
    -XX:+PrintAssembly
    public volatile int a = 0;

    public void doSth() {
        a++;
    }

    public static void main(String[] args) {
        Main main = new Main();
        //循环2000次,触发JIT
        for (int i = 0; i < 2000; i++) {
            main.doSth();
        }
    }

    //截取部分汇编代码, 在Idea中直接启动即可. 在输出中搜索doSth();

    0x0000000003483cf8: je      3483d17h          ;*aload_0
                                                    ; - controller.Main::doSth@0 (line 44)
                                                    -- 对应a++;这行代码;

    0x0000000003483cfe: mov     esi,dword ptr [rdx+0ch]  ;*getfield a
                                                    ; - controller.Main::doSth@2 (line 44)

    0x0000000003483d01: inc     esi
    0x0000000003483d03: mov     dword ptr [rdx+0ch],esi
    //内存屏障
    0x0000000003483d06: lock add dword ptr [rsp],0h  ;

### final关键字

参考:[深入理解 Java 内存模型（六）——final](https://www.infoq.cn/article/java-memory-model-6)

我觉得相关文章中,解释的已经相当详细明了,  核心如下:

对 final 域的读和写更像是普通的变量访问。对于 final 域，编译器和处理器要遵守两个重排序规则：

1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

    a. JMM 禁止编译器把 final 域的写重排序到构造函数之外。

    b. 编译器会在 final 域的写之后，构造函数 return 之前，插入一个 StoreStore 屏障。这个屏障禁止处理器把 final 域的写重排序到构造函数之外。

    言外之意是什么呢? 这意味着对于普通域而言, 就没有这样的要求了对于以下代码:

        public class FinalExample {
            int i;                            // 普通变量 
            final int j;                      //final 变量 
            static FinalExample obj;

            public void FinalExample () {     // 构造函数 
                i = 1;                        // 写普通域 
                j = 2;                        // 写 final 域 
            }

            public static void writer () {    // 写线程 A 执行 
                obj = new FinalExample ();
            }

            public static void reader () {       // 读线程 B 执行 
                FinalExample object = obj;       // 读对象引用 
                int a = object.i;                // 读普通域 
                int b = object.j;                // 读 final 域 
            }
        }
    
    先执行线程A, 再执行线程B, 就有可能会出现这样一种情况, B线程已经拿到obj的真实引用, 但obj对象的普通域 i 因为 重排序, 在构造器return之后,才进行赋值操作. 这样就会导致前后读取到的值并不一致.

    而对final的重排序规则则解决了这个问题.
   
2. 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

    a. 在一个线程中，初次读对象引用与初次读该对象包含的 final 域，JMM 禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）。编译器会在读 final 域操作的前面插入一个 LoadLoad 屏障。

3. 引用类型

    a. 在构造函数内对一个 final 引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

而除了以上规则以外, 还需要一个保证：在构造函数内部，不能让这个被构造对象的引用为其他线程可见，也就是对象引用不能在构造函数中“逸出”。

    public class FinalReferenceEscapeExample {
        final int i;
        static FinalReferenceEscapeExample obj;

        public FinalReferenceEscapeExample () {
            i = 1;               //1 写 final 域 
            obj = this;          //2 this 引用在此“逸出”
        }

        public static void writer() {
            new FinalReferenceEscapeExample ();
        }

        public static void reader {
            if (obj != null) {           //3
                int temp = obj.i;        //4
            }
        }
    }
final语义增强的目的是什么呢?

在旧的 Java 内存模型中 ，最严重的一个缺陷就是线程可能看到 final 域的值会改变。比如，一个线程当前看到一个整形 final 域的值为 0（还未初始化之前的默认值），过一段时间之后这个线程再去读这个 final 域的值时，却发现值变为了 1（被某个线程初始化之后的值）。

为了修补这个漏洞，JSR-133 专家组增强了 final 的语义。通过为 final 域增加写和读重排序规则，可以为 java 程序员提供初始化安全保证：只要对象是正确构造的（被构造对象的引用在构造函数中没有“逸出”），那么不需要使用同步（指 lock 和 volatile 的使用），就可以保证任意线程都能看到这个 final 域在构造函数中被初始化之后的值。

### 如何判断

那么更重要的一个问题是, 对于我们开发者而言, 又该怎样判断一段代码是不是线程安全的呢?

Happens-before 先行发生 原则

程序次序法则:线程中的每个动作A都happens-before于该线程中的每一个动作B，其中，在程序中，所有的动作B都能出现在A之后。(单线程中)

监视器锁法则:对一个监视器锁的解锁 happens-before于每一个后续对同一监视器锁的加锁。

volatile变量法则:对volatile域的写入操作happens-before于每一个后续对同一个域的读写操作。

线程启动法则:在一个线程里，对Thread.start的调用会happens-before于每个启动线程的动作。

线程终结法则:线程中的任何动作都happens-before于其他线程检测到这个线程已经终结、或者从Thread.join调用中成功返回，或Thread.isAlive返回false。

中断法则: 一个线程调用另一个线程的interrupt happens-before于被中断的线程发现中断。

终结法则:一个对象的构造函数的结束happens-before于这个对象finalizer的开始。

传递性:如果A happens-before于B，且B happens-before于C，则A happens-before于C

Happens-before关系只是对Java内存模型的一种近似性的描述，它并不够严谨，但便于日常程序开发参考使用.

关于更严谨的Java内存模型的定义和描述，请阅读JSR-133原文; JSR-133链接上面已经给出来了.

如果不满足上面的条件, 那么就必须考虑是否是线程安全的了.

</font>