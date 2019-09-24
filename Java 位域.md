# Java位域

<font face="微软雅黑">
这个概念是在 Effective Java中了解到的, 可以通过EnumSet来代替位域这种方式表达.

并不是很常见的概念, 因此记录下.

如果在这之前恰好了解过 bitmap这种数据结构就更好了。

不了解也没有关系。

bitmap 就是用bit的每一位来代表一个特殊的状态值， 或者说标签属性等等.

举例来说, 8位的数值, 用 0000 0001 代表 北, 0000 0010 代表南, 0000 0100 代表西 依次类推.

那么当我们拿到一串bit， 如：

0100 0000 自然可以去对应的映射关系表中查找到 究竟是属于哪一种类型， 如果我们想同时传递两个数值呢？

只需要 0000 0011 这样就可以表示 北 南 两个方向了， 当然 至多可以表示 8个方向.

我们来试试这种表示方式:

    public class Direction {

        public static final short NORTH = 1;

        public static final short SOUTH = 1 << 2;

        public static final short WEST = 1 << 3;

        public static final short EAST = 1 << 4;

        public static final short SOUTH_EAST = 1 << 8;
    }

在这里我只是简略的定义了其中5中.

那么可能会有一个问题， 既然使用 short来表示， 为什么不用 1 2 3 ... 8 来表示数据呢？ 这样我们甚至都不需要2 的 8次方, 只需要 3位就能够表示所有数据了.

但是不妨让我们再来想一想, 在使用 1 ~ 8 的方式中如何同时传入多种状态呢？

在这里是不是必须使用 一个 short[] 去接收数据？

那么用位有什么好处呢？

    void array(NORTH | SOUTH | SOUTH_EAST)

在方法的调用上 可以采用这种直观易懂且计算速度快的方式， 而在传入值 不难发现 最终只有一个值:

    1000 0011

这一个数值即表示了包含了相应的三种状态.

而这就是 java中 位域的使用方式.

那么进一步来看, 当我们不再满足 8位 甚至需要更多种状态值的时候 可以切换到 int long 甚至于 bitmap. 接收无限位。

但仅仅是位域这种表示 我们仅仅支持 64种以下的状态类型， 因为 java种最长的基本类型 也就只有64位了。

那么继续来看看这种位域有什么缺陷呢？

使用int 类型 或 long类型， 没有办法加入一些自定义的东西， 通常情况下 在这种地方使用枚举是更好地选择。

否则的话所有的地方依然要使用 switch判断的方式， 另外由于 int定义为 static final 时 本身就是编译时常量，

如果有人依赖他， 将来即使这里的数值更新了， 比如删掉两三个， 即使不重新编译， 对方的class文件依然不会出错。 但事实上， 出错是一种必然。

就上面的例子来说， 我们想要返回所有的String 该怎么办？

必然是 switch case return "南" 类似的方式.

那么切换成枚举类型呢?

    public enum EnumDirection {

        NORTH("north"), EAST("east"), SOUTH("south"), WEST("south");

        private final String name;

        private EnumDirection(String name) {
            this.name = name;
        }

        public String getName() {
            return this.name;
        }
    }

枚举类型的好处，不再赘述。

那与今天的主题, 位域有什么关系呢？

我们知道，位域的优点 占用内存小， 表示方便， 传递值方便， 性能高.

EnumSet， 让我抄一段描述:

> 这个类实现Set接口,提供了丰富的功能，类型安全性，以及可以从任何其他Set实现中得到的互用性。但是在内部具体的实现上，每个EnumSet内容都表示为位矢量。如果底层的枚举类型有64个或者更少的元素——大多数如此。整个EnumSet就用单个long来表示，因此它的性能比的上位域的性能。批处理，如removeAll和retainAll，都是利用位算法来实现的。就像手工替代位域实现得那样。

是的， 是位运算.

就看一段代码：

    public boolean contains(Object e) {
        if (e == null)
            return false;
        Class<?> eClass = e.getClass();
        if (eClass != elementType && eClass.getSuperclass() != elementType)
            return false;

        return (elements & (1L << ((Enum<?>)e).ordinal())) != 0;
    }

我们关注到最后一行， 如上述EnumDirection， WEST 的 ordinal() 即是4, 也就意味着 它值在这里被理解为 1 << 4

而通过 elements 传入enumSet 的集合， 如：

    EnumSet<EnumDirection> enumSet = EnumSet.of(EnumDirection.EAST, EnumDirection.NORTH);

不难获知 enumSet 的 elements值为 0000 0011 当然 这里是 long类型, 我只写了最后8位, 而

    0000 0011 & 0000 1000 必然是等于 0的 因此 contains 返回false.

这是极其高效的方式. 而目的也正在于解决 int型 位域的种种弊端.
</font>