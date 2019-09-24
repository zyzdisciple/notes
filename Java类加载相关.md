# Java Class文件及类加载

<font face="微软雅黑">

在Java内存区域介绍， 及垃圾收集中都有提到过， 方法区这个概念， 存储的是Java的类信息， 当Java类被加载之后， 就会被存储到方法区中。

那么Java类是如何被加载的呢？Jvm又是如何解读 class 文件， 全限定名等等相关的东西又是怎样融入Java的体系中呢？

## class文件

这里的class文件并非仅仅是指 .class文件， 而是符合Java虚拟机规范的 class文件或相应的字节流。甚至于 class文件并非与Java强相关， 只要能够被相应的编译器解读并生成相应的 class文件，在这里并不在乎它的语言来源究竟是什么。

并非仅仅是class文件有这种特性，对于计算机而言，本身也是如此，只要能够转换为相应的寄存器指令，并不在乎，也不需要管指令的生成来源是什么，而Jvm虚拟机正是解读字节码指令的平台。

class文件是一组以8位字节为基础单位的二进制流，所以class文件的数据存储是按照严格的规则进行存储，按照顺序解读，每个字节，每个位置代表什么都是被严格限定的，它没有相关的描述符， 只有这样才能够被jvm解读。

class文件中只有两种数据结构：

1. 无符号数
   
   无符号数属于基本数据类型，以u1,u2,u4,u8分别来代表：1，2，4，8个字节的无符号数。

   无符号数可以用来描述：数字，索引引用，数量值或按照UTF-8编码构成的字符串值。

   换句话说，就是：数值，可以被用来做相关运算的数值；索引；表示表大小，属性长度等等的数量值；以及存储的字面量。 包括 变量名， 字符串常量等其他值。

2. 表

    表， 是有层次关系的复合数据结构。

### 文件的解读

class文件的解读真的是没有什么奥秘可言，均是人为规定了 在某个特定的位置表示什么意义，该如何被解读， 当拿到一个class文件只需要按照Java虚拟机规范中一点点解读，就能够将整个class文件翻译成可读懂的文件， 甚至于翻译成Java代码也不是问题。

eg. 每个class文件的头四个字节都是0xCAFFBABA，这个魔数值用来表示当前文件是可以被JVM解读的class文件，也是文件类型的真正描述，毕竟我们都知道后缀名可以被随意更改而不影响文件使用特定的方式进行解读。

eg. 紧接着的魔数值得四个字节存储的class文件的版本号， 前两个是次版本号，后两个是主版本号。 主版本号时从45开始，而每向上一个主版本则加一。

JVM虚拟机可以解读更低版本的class文件，而对高版本的class文件拒绝解读。

而剩余的class文件解读方式都是诸如此类的方式。

当然， 我们不需要自己费劲的去将class文件转换为16进制，然后一个个去解读它。（windows中以16进制的方式显示 可以采用 WinHex软件）

在Java环境下，我们仅仅需要 

    javap -verbose <classpath>

即可，但仅能够输出public 方法，若是想知道更全面的信息

    javap -h

有相关的提示信息，其使用语法则是：

    javap <options> <classes>

### 常量池

class文件中的常量池，之所以要提到这个概念，与之后要提到的一个概念：动态连接，息息相关， 因此需要略做介绍。

直接看代码：

    public class TestClass{
	
        public int a = 1;
        
        public static final String b = "zyzdisciple";
        
        public String myName = "zyzdisciple2";
        
        public static int re(int a, final int b) {
            return a + b;
        }

    }

经过编译后， javap -verbose TestClass.class，截取其常量池片段：

    Constant pool:
    #1 = Methodref          #6.#22         // java/lang/Object."<init>":(
    #2 = Fieldref           #5.#23         // TestClass.a:I
    #3 = String             #24            // zyzdisciple2
    #4 = Fieldref           #5.#25         // TestClass.myName:Ljava/lang
    #5 = Class              #26            // TestClass
    #6 = Class              #27            // java/lang/Object
    #7 = Utf8               a
    #8 = Utf8               I
    #9 = Utf8               b
    #10 = Utf8               Ljava/lang/String;
    #11 = Utf8               ConstantValue
    #12 = String             #28            // zyzdisciple
    #13 = Utf8               myName
    #14 = Utf8               <init>
    #15 = Utf8               ()V
    #16 = Utf8               Code
    #17 = Utf8               LineNumberTable
    #18 = Utf8               re
    #19 = Utf8               (II)I
    #20 = Utf8               SourceFile
    #21 = Utf8               TestClass.java
    #22 = NameAndType        #14:#15        // "<init>":()V
    #23 = NameAndType        #7:#8          // a:I
    #24 = Utf8               zyzdisciple2
    #25 = NameAndType        #13:#10        // myName:Ljava/lang/String;
    #26 = Utf8               TestClass
    #27 = Utf8               java/lang/Object
    #28 = Utf8               zyzdisciple

在常量池中主要存储两大类型常量： 字面量 和 符号引用：

1. 字面量： 文本字符串， 声明为final的常量值。

    如上例子中的 #28 存储的就是常量， 同时也作为字面量 其中的 String， #12 #3 都是字面量。

2. 符号引用：

    符号引用包括下面三类常量：

    1. 类和接口的全限定名； #5 #6（TestClass继承自Object， 父类的全限定名也存储为符号引用。） 在常量池中存储的类全限定名有以下几种：

        1. 直接继承的父类
        2. 直接实现的接口
        3. 在方法中调用的各种类
        4. 内部类

        目前我试过且能想到的只有这几种。

    2. 字段的名称及描述符
    3. 方法的名称及描述符

        字段的名称不多解释， 而这里的描述符则是指用来描述字段的数据类型，方法的参数列表（包括数量，类型，及顺序）及返回值。

        其描述符与类型的关系对应如下：

        B byte； C char； D double； F float； I int； J long；

        S short； Z boolean； V void； L 对象类型 eg. Ljava/langg/Object

正是通过符号引用， 来指向对应的常量，从上述也可以看出来， 对于class而言，方法的返回值类型不同，其实应该表示的是不同的方法，因为其描述符不同。

但在编译器中却不允许这种情况出现，不仅不能重载， 也不能进行重写。都会提示错误。

### Code属性

既然Java文件编译之后生成的class文件存储有所有相关信息，那么我写的那么一堆代码呢？ 那一堆if else for 循环都到哪里去了？

答案就是存储在code属性中。

通过 javap命令就可以看到相关的数据。

java程序方法体中的的代码经过编译器处理后，最终变为字节码指令存储在code属性内， code_length用来代表字节码长度， 而code属性则用来存储一系列的字节码流。 code_length虽然是一个 u4长度的值，理论上最大可以到达 2^32 - 1的长度， 但事实上，在java虚拟机规范中明确要求不能够超过65535条字节码指令，也即U2的长度， 超过即会导致编译失败。

而这种问题最可能出现在 jsp页面的编写中。

同样，有代码如下：

    public class TestA extends TestB {

        public int add (int a, int b) {
            return a + b;
        }
    }

    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: ireturn

### 字节码指令

code属性就是上面那样的一条条字节码指令， 字节码指令正是由u1类型的， 代表着某特种特定操作含义的数字（称为操作码）以及跟随其后的零至多个代表此操作所需的参数构成的。

每个字节码指令都是一个u1类型的单字节， 也就意味着在 java中最多可以表达256种指令。

而java中的大部分指令都是不含操作数的。这点就与基于寄存器的指令集有所不同。

之前在java内存中提到过，java的操作是基于栈的，最小的操作元素是栈帧，也即一个 slot。

而在上面的代码中也明确看到了： 对所有指令都没有明确的指明操作数是谁，因为所做的操作无非只有两种，出栈，入栈。

而相应的当确定了指令是谁， 其代表的含义即是对当前栈顶元素进行的操作，栈顶元素也就自然而然的成为了相应的操作数。

而字节码指令本身大都也已经指定了相应的操作数类型究竟是什么了。

字节码指令按照功能主要有以下几类：

1. 加载和存储指令

    即将数据在栈帧中的局部变量表和操作数栈之间来回传输。

1. 运算指令

    用于对两个操作数栈上的值进行某种特定运算，并把结果重新存储在操作数栈顶。

1. 类型转换指令
   
   可以将两种不同的数值类型进行转换。

1. 对象的创建与访问指令

1. 操作数栈管理指令

1. 控制转移指令

    即各种 判断 跳转指令

1. 方法调用和返回指令

1. 异常处理指令

1. 同步指令

    即synchronizedx相关。

## Java类加载
 
 终于， 我们在Class文件中存储了种种信息，那么虚拟机又如何，在何时，通过怎样的方式将class文件加载，将信息存入内存中呢？

 类从被加载到虚拟机内存中开始，到被卸载为止，共经历这样几个阶段：

 加载， 验证， 准备， 解析， 初始化， 使用， 卸载7个阶段。

 其中前五个阶段是按序开始， 但仅仅是指开始，并没有说按序进行，可以交叉进行。

 对于何时加载一个类，虚拟机中没有明确的规范， 但对何时必须初始化一个类， 却是有着非常明确且严格的约定， “有且只有”以下五种情况：

 1. 遇到new、putstatic、getstatic、invokestatic、 这四条字节码指令时，如果类没有进行初始化， 需要先触发其初始化。 这是指， new 一个实例对象时， 读取或设置一个类的静态字段时，以及调用一个类的静态方法时。

1. 使用java.lang.reflect包的方法对类进行反射调用时。

1. 当初始化一个类时， 如果发现其父类还没有进行初始化， 则需触发其父类的初始化。（但接口并不会进行相应的初始化）

1. 当虚拟机启动时，用户需要指定一个要执行的主类（main（）入口类），虚拟机会先初始化这个类。

1. 当使用JDK1.5支持时，如果一个java.langl.incoke.MethodHandle实例最后的解析结果REF_getStatic、REF_putStatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

而除了上述五种以外， 所有的引用类方法都不会触发初始化， 称为被动引用。

需要注意的是：

    MyObject obj = new MyObject[]

这种代码并不会触发MyObject的类的初始化，数组的创建指令是 newarray， 并不在上述的五种情况中。

而接口也同样不会进行初始化， 只有当使用到其中常量时才会进行初始化。

## 类加载过程

### 加载

加载是类加载的第一个阶段， 在加载阶段：虚拟机需要完成：

1. 通过一个类的全限定名来获取此类的二进制字节流。

1. 将这个字节流所代表的静态存储结构转化为方法区的运行时内存结构。

1. 在内存中生成一个代表这个类的java.lang.Class对象，并作为方法区中该类的访问入口。

在加载过程 JVM虚拟机给予了最大的灵活性。

可以从ZIP包中读取， 如 jar包， war包

可以从网络中读取

可以运行时计算生成

可以从其他文件生成转换而来 等其他方式。

对于运行时自动生成这点而言，运用的最多的地方是动态代理技术，在java.lang.Proxy 中， 就是用了 ProxyGenerator.generatoerProxyClass 来生成特定的“*$Proxy”的代理类的二进制字节流。

至于什么是动态代理, 参考链接：

[java动态代理实现与原理详细分析](https://www.cnblogs.com/gonjan-blog/p/6685611.html)

### 验证

1. 文件格式验证
   
    由于第一步加载中的 class的来源多种多样， 因此class文件的安全性就无法保证， 如果从网络上加载了一个非法的 class文件，却没有进行校验，可能会导致虚拟的崩溃，因此， 校验有一定的必要性。

    如同大多数校验一样， 当我们拿到一个.class文件时， 首先校验的是， 这是否是一个class文件，能否正确识别这是一个class文件， 又由于class文件的格式规则有着严格要求， 它是否满足？

    凡此种种，其目的只是为了能够将class文件正确解析，并可以按照规则存入方法区， 而在这之后，字节流就被加载进入方法区， 其后的校验均是对方法区的存储结构校验。

2. 元数据验证

    在文件格式满足要求且可以正确存入方法区之后，第二步所需要做的事情就是校验是否符合java语言的基础规范本身。

    在class文件中存储有相应的字段描述符， class继承关系等等， 但仅仅是存储， 并不会校验是否合理， 如在一个 class中引用了另一个class的private 字段也是依然可以的。

3. 字节码验证

    保证了数据符合java的语法规范之后， 下一步要做的是验证语义是否表达完整， 符合规范。元数据验证更多的是验证是否可进行相应的操作，根据java的关键字， 等基础规则验证是否可行， 而不进行实际上的语义判断。

4. 符号引用验证

    这个阶段的校验发生在【解析】阶段，是在将虚拟机的符号引用转换为直接引用的时候，符号引用验证可以看做是对类自身以外的（常量池中的各种符号引用）信息进行匹配性校验，通常需校验：

    根据全限定名能否找到对应的类

    在指定类中是否存在符合需要的方法，字段

    符号引用中的类、字段、方法的访问性是否能够被当前类访问。

### 准备阶段

准备阶段是为类变量（static）分配内存并设置初始值的阶段。 

这里的分配内存并非是指分配堆内存， 而是将变量的索引存入方法区， 如果是基础变量的话， 也会存入相应的值， 对于：

    private static Object obj = new Object();

而言，new Object（）所占用的内存依然是在堆中， 不难理解，当再度为static变量赋值为其他对象时， 原有的Object的内存就可以被回收， 因此充分满足 一个对象的相关回收条件。

但如果类字段为 final 的， 在准备阶段还会对类变量进行赋值。

### 解析阶段

是将虚拟机常量池中的符号引用 替换为 直接引用的过程。

在之前提到过符号引用有三种， 类的接口及全限定名， 字段名称及描述符， 方法名称及描述符。

而直接引用则是直接指向目标的指针、相对偏移量、或是一个能够间接定位到目标的句柄。

虚拟机规范中并未规定解析阶段发生的具体时间，只要求在执行了newarray，checkcast，getfield，getstatic，instanceof，invokedynamic，invokeinterface，invokespecial，invokestatic，invokevritual，ldc，ldc_w, multianewarray，new，putfield和putstatic这16个用于操作符号引用的字节码指令之前，先对其符号引用进行解析。

解析主要针对以下几种：

类或接口、字段、类方法、接口方法、方法类型、方法句柄、点限定符

1. 类或接口的解析

    假设当前所处的类为D， 如果要把一个从未解析过的符号引用N解析为一个类或接口C的直接引用， 需要经历以下步骤：

    a. 如果C并非数组类型，那么虚拟机会将代表N的全限定名传递给D的类加载器去加载这个类C，在加载过程中如果触发了任何异常都会宣告解析失败。

    b.如果C是一个数组类型，并且数组的元素类型为对象，也就是N的描述符会是类似”[Ljava.lang.Integer"的形式，那么会按照第a点的规则加载数组元素类型。如果N的描述符如前面所假设的形式，需要加载的元素类型就是“java.lang.Integer”，接着由虚拟机生成一个代表此数组和元素的数组对象。

    c.如果上面的步骤没有出现任何异常，那么C在虚拟机中实际上已经成为一个有效的类或接口了，但在解析完成之前还要进行符号引用验证，确认C是否具备对D的访问权限，如果发现不具体访问权限，将抛出java.lang.IllegalAccessError错误。

2. 字段解析

    要解析一个未被解析过的字段符号引用，首先将会对字段表内class_index项中索引的CONSTANT_Class_info符号引用进行解析，也就是字段所属性的类或接口的符号引用。如果在解析这个类或接口符号引用过程中出现了任何异常，都会导致字段符号引用解析失败。如果解析成功完成，那么这个字段所属性的类或接口用C表示，虚拟机规范要求如下步骤对C进行后续字段的搜索：

    a.如果C本身就包含了简单名称和字段描述符都与目标相匹配的字段，则返回这个字段的直接引用，查找结束。

    b.否则，如果C中实现了接口，将会按照继承关系从下往上递归搜索各个接口和它的父接口，如果接口中包含了简单名称答字段描述符都与目标相匹配的字段，则返回该字段的直接引用，查找结束。

    c.否则，如果C不是java.lang.Object的话，将会按照继承关系从下往上递归搜索其父类，如果在父类中包含了简单名称和字段描述符都与目标匹配的字段，则返回这个字段的直接引用，查找结束。

    d.否则，查找失败，招抛出java.lang.NoSuchFieldError错误。

    需要注意的是， 虽然在解释中 b.c是拥有先后顺序的， 但是在实际应用中， 如果一个类 其自身实现了接口（非父类实现接口）， 同时在继承树，或接口列表中存在相同字段，则是不被允许的。无论接口或是父类都是其直接上级，无法确定字段的究竟归属是谁。

        public class TestB extends TestD implements TestInterface {
            //public TestC testC = new TestC("objectB");
        }

        public class TestD {
            public TestC testC = new TestC("objectD");
        }

        public interface TestInterface {
            TestC testC = new TestC("interface");
        }

        public static void main(String[] args) {
            TestB testB = new TestB();
            //编译报错
            TestC testC = testB.testC;
            System.out.println("~~~~~~~~~~~~~~~");
            System.out.println(testC.name);
        }

    此时因为TestB中不存在 testC属性，因此需要向上级查找，无法获知究竟是谁， 唯有当将注释放开， 即可通过。

3. 类方法解析

    类方法解析的第一个步骤与字段解析一样，也是需要先解析出方法表的class_index项中索引的方法所属性类或接口的符号引用，如果解析成功，依然用C表示这个类，接下来虚拟机将会按照如下步骤进行后续的类方法搜索：

    a.类方法和接口方法符号引用的常量类型定义是分开的，如果在类方法表中发现了class_index中索引的C是个接口，那么直接就抛出java.lang.IncompatibleClassChangeError错误。

    b.如果通了第a步，在类C中查找是否有简单名称和描述符与目标相匹配的方法，如果有则返回这个方法的直接引用，查找结束。

    c.否则，在类C的父类中递归查找是否有简单名称和字段描述符都与目标匹配的方法，则返回这个方法的直接引用，查找结束。

    d.否则，在类C实现的接口列表及它们的父接口中递归查找否有简单名称和字段描述符都与目标匹配的方法，说明类C是一个抽象类，这时候查找结束，抛出java.lang.AbstractMethodError错误。

    e.否则，宣告查找失败，抛出java.lang.NoSuchMethodError错误。

    最后，如果查找过程成功返回了直接引用，将会对晕个方法进行权限验证；如果发现不具务对此方法的访问权限，将抛出java.lang.IllegalAccessError错误。

    在这里困惑了不少时间， 这里需要注意的一个地方是： 类方法，仅仅是指 static 方法，并不包括相应的实例方法，与前面提到的字段解析有所不同，字段解析并没有对 是否为 静态进行区分。

    这点与java的动态语言调用有关， 下一篇再提。

### 初始化

类的初始化是类加载过程的最后一步，前面的类加载动作，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正执行类中定义的Java程序代码(或者说是字节码)。

   在准备阶段，变量已经赋过一次系统要求的初始值，而在初始化阶段是执行类构造器\<clinit\>()方法的过程。

1. \<clinit\>()方法是由编译器自动收集类中所有类变量的赋值动作和静态语句块(static{})中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后变量，在前面的静态语句块中可以赋值，但是不能访问。
 
2. \<clinit\>()方法与类的构造器\<init\>()不同，它不需要显示地调用父类类构造器，虚拟机会保证在子类的\<clinit\>()方法执行之前，父类的\<clinit\>()方法已经执行完毕。因此在虚拟机中第一个被执行\<clinit\>()方法的类肯定是java.lang.Object。

3. \<clinit\>()方法对于类或接口来说并不是必须的，如果一个类中没有静态语句块，也没有对类变量的赋值操作，那么编译器可以不为这类生成\<clinit\>()方法。

4. 接口中不能使用静态语句块，但仍然可以有变量初始化的同仁操作，因此接口与类一样都会生成\<clinit\>()方法，但接口与类不同的是，执行接口的\<clinit\>()方法不需要先执行父接口的\<clinit\>()方法。只有当父接口中定义的变量被使用时，父接口才会初始化。

5. 虚拟机会保证一个类的\<clinit\>()方法在多线程环境中被正确地加锁同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的\<clinit\>()方法，其它线程都需要阻塞等待，直到活动线程执行\<clinit\>()方法完毕。如果在一个类的\<clinit\>()方法中有很耗时的操作，那就可能造成多个线程阻塞，在实际应用中这种阻塞往往是很隐蔽的。

## Java类加载器

在类加载阶段， “通过一个类的全限定名来获取描述此类的二进制字节流”， 这个动作放在java虚拟机外部去实现， 而实现这个动作的代码模块被称为“类加载器”。

对于任意一个类， 都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性. 比较两个类(并非是指实例对象)是否相等,只有在同一种类加载器下才有意义.

相等,指的是, 类的Class对象的equals方法, isInstance()方法的返回值, 还有 instance of 关键字的判定结果.

而大家都有所耳闻的是, Java的类加载机制采取的是双亲委派模型:

1. 从Java虚拟机的角度来讲, 只存在两种不同的类加载器:一是启动类加载器(Bootstrap ClassLoader), 这个类加载器采用 C++语言实现, 是虚拟机自身的一部分,另一种则是其他所有的类加载器. 这些类都继承自抽象类java.lang.ClassLoader.

2. 从Java开发者的角度来看, 可以分为三种类加载器.

    a. 启动类加载器(BootstrapClassLoader) Bootstrap类加载器负责加载rt.jar中的JDK类文件，它是所有类加载器的父加载器.

    b. 扩展类加载器(Extension ClassLoader)，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录.

    c. 应用程序类加载器(Application ClassLoader),  也即 ClassLoader.getSystemClassLoader()方法的返回值.一般也称作系统类加载器,它负责加载用户类路径上所指定的类库, 如果应用程序中未曾自定义过类加载器, 那么一般这个就是程序中默认的类加载器.

    <font color="orange">同样的, 执行相同的代码, 我发现与网上得到的结论都是不同的, 我在这里用的是jdk11的最后一个版本.

        public static void main(String[] args) {
            System.out.println(Thread.currentThread().getContextClassLoader());
            Class<?> cls = Main.class;
            // 取得Class类对象的类加载器信息
            System.out.println(cls.getClassLoader());
            // 取得Class类对象的类加载器父加载器信息
            System.out.println(cls.getClassLoader().getParent());
            // 取得Class类对象的类加载器父加载器的父加载器信息
            System.out.println(cls.getClassLoader().getParent().getParent());
        }
    
    输出的是:

        jdk.internal.loader.ClassLoaders$AppClassLoader@2437c6dc
        jdk.internal.loader.ClassLoaders$AppClassLoader@2437c6dc
        jdk.internal.loader.ClassLoaders$PlatformClassLoader@1e643faf
        null

    是的, 所以在Java11以后的关系应该是, AppClassLoader -> PlatFormClassLoader -> BootStapClassLoader
    </font>

3. loadClass

    参考:
    [深入理解ClassLoader工作机制（jdk1.8）](https://blog.csdn.net/u014634338/article/details/81434327)

    下面还是以 jdk1.8为基准

        protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
        {
            synchronized (getClassLoadingLock(name)) {
                // First, check if the class has already been loaded
                Class<?> c = findLoadedClass(name);
                if (c == null) {
                    long t0 = System.nanoTime();
                    try {
                        //a:
                        if (parent != null) {
                            c = parent.loadClass(name, false);
                        } else {
                            c = findBootstrapClassOrNull(name);
                        }
                    } catch (ClassNotFoundException e) {
                        // ClassNotFoundException thrown if class not found
                        // from the non-null parent class loader
                    }

                    if (c == null) {
                        // If still not found, then invoke findClass in order
                        // to find the class.
                        long t1 = System.nanoTime();
                        //b:
                        c = findClass(name);

                        // this is the defining class loader; record the stats
                        PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                        PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                        PerfCounter.getFindClasses().increment();
                    }
                }
                //c:
                if (resolve) {
                    resolveClass(c);
                }
                return c;
            }
        }

    源码很简单, 查找class是否已经被加载, 如果没有加载:

    a: 对应注释中的a处, 调用parent的classLoader, 如果代码都遵循双亲委托模型, 则会一层层向上调用, 直到BootstrapClassLoader.

    而parent并非是继承关系, 是通过组合的形式关联起来.

        private final ClassLoader parent;

    只有当父类无法加载当前类调用子类的findClass()方法去加载当前类, 层层向下.

    b: 注释b处

        protected Class<?> findClass(String name) throws ClassNotFoundException {
            throw new ClassNotFoundException(name);
        }

    在ClassLoader类中findClass源码如上, 不难发现, 依循双亲委托模型, 只需要实现自己的findClass方法即可. 这才是我们在自己的实现类中, 唯一要做的事情.

    而当我们拿到对应的Class文件, 或字节流形式的文件, 究竟该以怎样的方式实现自己的findClass, 进而加载这个类呢?

    I: Class<?> defineClass(String name, java.nio.ByteBuffer b,ProtectionDomain protectionDomain)

    指定保护域（protectionDomain），把ByteBuffer的内容转换成 Java 类。

    II: Class<?> defineClass(String name, byte[] b, int off, int len)

    把字节数组 b中的内容转换成 Java 类

    正是通过这个方法, 我们可以把本地文件或网络流传入, 进而生成相应的class, 加载进jvm虚拟机.

    而另一个方法也就是注释c处:

        protected final void resolveClass(Class<?> c) {
            //native方法
            resolveClass0(c);
        }

    当类被加载进来之后是否需要进行相关的链接操作也正是由这个参数所指定.

    链接: 也就是 验证, 准备, 解析相关操作.

    至于具体的重写 findCLass() , 案例中就有或者网上百度一大堆, 不再多说.

1. 双亲委派模型

    提到过了, 双亲委派模型是层层向上, 直到最顶层, 如果顶层无法加载类, 再层层向下, 直到抛出ClassNotFoundException; 其中父级与子类之间并不用继承关系进行捆绑, 而是用组合的形式进行绑定.

    那么为什么要使用双亲委派模型?

    如果我们自己定义了一个 java.lang.String 放在classpath下, 那么当使用时, 系统中就会出现多个 String类, String的种种行为就无法保证, 而如果有人上传的框架或插件中, 偷偷的放入了自己的 java.lang.String, 并用来做一些坏事, 使用插件的人又该怎么预防呢?

    因此才需要从上到下一级级进行加载, 如果当前类已经被加载过了, 那么就不再会被加载. 这就保证了即使定义了自己的 java.lang.String 也会被BootstrapClassLoader加载到正确的 String类, 自己定义的永远不会执行.

1. 破坏双亲委托模型

    前面提到的类加载器的代理模式并不能解决 Java 应用开发中会遇到的类加载器的全部问题。Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JCE、JNDI、JAXP 和 JBI 等。这些 SPI 的接口由 Java 核心库来提供.

    基础类之所以被称为"基础"，是因为它们总是作为被调用代码调用的API。

    但是，如果基础类又要调用用户的代码，那该怎么办呢。这并非是不可能的事情，一个典型的例子便是JNDI服务, 它的代码由启动类加载器去加载(在JDK1.3时放进rt.jar)

    但JNDI的目的就是对资源进行集中管理和查找，它需要调用独立厂商实现部部署在应用程序的classpath下的JNDI接口提供者(SPI, Service Provider Interface)的代码，但启动类加载器不可能"认识"之些代码，该怎么办？

    线程上下文类加载器正好解决了这个问题。如果不做任何的设置，Java 应用的线程的上下文类加载器默认就是系统上下文类加载器。在 SPI 接口的代码中使用线程上下文类加载器，就可以成功的加载到 SPI 实现的类。线程上下文类加载器在很多 SPI 的实现中都会用到。

        Thread.currentThread().getContextClassLoader();

    至于想了解更多:

    [深入理解Java类加载器(2)：线程上下文类加载器](https://blog.csdn.net/zhoudaxia/article/details/35897057)

    对我目前而言, 还不曾涉及到需要用到类加载相关的东西, 不进行相关框架设计, 谈论更多也只是纸上谈兵, 算不得数, 也与当前目的违背.