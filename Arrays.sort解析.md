# Arrays.sort()解读

<font face="微软雅黑">

在学习了排序算法之后, 再来看看 Java 源码中的, Arrays.sort() 方法对于排序的实现.

都是对基本数据类型的排序实现, 下面来看看这段代码:

1. Arrays.sort(int[] a)

        public static void sort(int[] a) {
            DualPivotQuicksort.sort(a, 0, a.length - 1, null, 0, 0);
        }

1. DualPivotQuicksort.sort

    在这里我将代码一步步拆开来看, 一点点解读, 并尝试在自己调用, 复制解读代码:

        static void sort(int[] a, int left, int right, int[] work,
                         int workBase, int workLen)

    参数部分暂时跳过, 在使用时自然能看到对应的作用;

    但在这里需要注意一个问题, 则是 left right 分别为 0, 和 array.length - 1, 传入的是数组的始末索引;

        if (right - left < QUICKSORT_THRESHOLD) {
            sort(a, left, right, true);
            return;
        }

1. 第一部分:

    在数组长度, 小于 QUICKSORT_THRESHOLD(286) 的时候, 采取的是双轴快速排序.

    至于这里的常量取值, 不是很明白, 个人觉得应该是经验数值.

        private static void sort(int[] a, int left, int right, 
                                boolean leftmost)

        int length = right - left + 1;

    这里表示当前需要排序的长度;

        if (length < INSERTION_SORT_THRESHOLD) {

    这里就比较好理解, 在使用快速排序的时候, 如果数组被切分到一定长度, 则选择 插入排序来替换, 速度更快, 需要的空间也更小. 这里的长度取的是 47.

        if (leftmost) {

    对于leftmost, 文档说明为 指示当前数组为数组的左侧部分, 在三向快速排序中, 我们将数组分为三部分, < mid, == mid, >mid, 虽然在双轴快速排序中, 与三向快速排序切分细节略有不同, 但这里的 leftmost指的是数组 &lt; mid, 部分的数组.

        for (int i = left, j = i; i < right; j = ++i) {
            int ai = a[i + 1];
            while (ai < a[j]) {
                //优化
                a[j + 1] = a[j];
                if (j-- == left) {
                    break;
                }
            }
            a[j + 1] = ai;
        }

    传统插入排序, 在这里就采取了插入排序的优化手段之一: 减少交换次数, 使得较大的部分直接右移, 而非通过交换的方式.

        else {
            do {
                if (left >= right) {
                    return;
                }
            } while (a[++left] >= a[left - 1]);

    这段代码还是比较有趣的, 在数组的快速排序拆分过程中, 将右侧, 也就是 >mid 的部分, 因为右侧都是大于等于 mid值的, 直接跳过其中已经排序好的部分, 将插入排序的起始位置选定一个更合适的位置, 优化排序;

        for (int k = left; ++left <= right; k = ++left) {
            ...
            ...
        a[right + 1] = last;

    这段是'双插入排序'的核心代码, 等快速插入排序看完之后, 再来看这段代码;

        int seventh = (length >> 3) + (length >> 6) + 1;
        int e3 = (left + right) >>> 1; // The midpoint
        int e2 = e3 - seventh;
        int e1 = e2 - seventh;
        int e4 = e3 + seventh;
        int e5 = e4 + seventh;
        if (a[e2] < a[e1]) { int t = a[e2]; a[e2] = a[e1]; a[e1] = t; }

        if (a[e3] < a[e2]) { int t = a[e3]; a[e3] = a[e2]; a[e2] = t;
            if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
        }
        if (a[e4] < a[e3]) { int t = a[e4]; a[e4] = a[e3]; a[e3] = t;
            if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
                if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
            }
        }
        if (a[e5] < a[e4]) { int t = a[e5]; a[e5] = a[e4]; a[e4] = t;
            if (t < a[e3]) { a[e4] = a[e3]; a[e3] = t;
                if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
                    if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
                }
            }
        }

    这段代码是这个 sort方法的真正起始部分, 前面的部分, 是迭代的结束代码;

    不同于在快速排序中我们所采取的 打乱数组, 从第一位开始选取的方式, 而是先将数组进行 6 分, 取出中间的5个节点, 将节点通过插入排序的方式排序;

    至于说为什么采取这样的方式进行6分, 嗯, 据经验所得.

    同样的在这里有一个点 容易被忽略, 由于数组的最大长度为 286, 则, seventh的最大值同样为41, 所以只进行一次快速排序, 在数组被切分后就执行插入排序.

        int less  = left;  // The index of the first element of center part
        int great = right; // The index before the first element of right part

    因为初始只有一个分区, center part 和 right part 为一个, 但 这里的 less表示, 中间分区的第一个元素, great + 1 表示 右侧分区的第一个元素.

        if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
            int pivot1 = a[e2];
            int pivot2 = a[e4];

    在5个节点两两不等时, 采取 a[e2], a[e4], 作为双枢轴快速排序的两个 枢纽.

        a[e2] = a[left];
        a[e4] = a[right];

    在这段代码中, 能够看出来的仅仅是 将 pivot1, pivot2, 所对应的元素不参与排序, 同时将左右两端的元素移位, 此时, 在循环中, 可以跳过左右两个元素.(所以还是不明白有什么更深层的含义)

        while (a[++less] < pivot1);
        while (a[--great] > pivot2);

        outer:
        for (int k = less - 1; ++k <= great; ) {
            int ak = a[k];
            if (ak < pivot1) { // Move a[k] to left part
                a[k] = a[less];
                /*
                * Here and below we use "a[i] = b; i++;" instead
                * of "a[i++] = b;" due to performance issue.
                */
                a[less] = ak;
                ++less;
            } else if (ak > pivot2) { // Move a[k] to right part
                //将 great指针 不断左移,直到, <= pivot2
                while (a[great] > pivot2) {
                    if (great-- == k) {
                        break outer;
                    }
                }
                if (a[great] < pivot1) { // a[great] <= pivot2
                    a[k] = a[less];
                    a[less] = a[great];
                    ++less;
                } else { // pivot1 <= a[great] <= pivot2
                    a[k] = a[great];
                }
                /*
                * Here and below we use "a[i] = b; i--;" instead
                * of "a[i--] = b;" due to performance issue.
                */
                a[great] = ak;
                --great;
            }
        }

        *left part           center part                right part
        * +--------------------------------------------------------------+
        * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
        * +--------------------------------------------------------------+
        *               ^                          ^       ^
        *               |                          |       |
        *              less                        k     great

    与快速排序类似:

    (left, less) : &lt; pivot1 的值

    [less, k): pivot1 <= && <= pivot2

    [k, great]\: 表示未排序的值.

    (great, right): > pivot2

        a[left]  = a[less - 1];
        a[less - 1] = pivot1;
        a[right] = a[great + 1];
        a[great + 1] = pivot2;

    在这里将之前被替换的值交换回来.

        sort(a, left, less - 2, leftmost);
        sort(a, great + 2, right, false);

    在这里需要注意的一点是: sort(a, great + 2, right, false);

    a[great + 1]表示的是右侧数组最小的值, 因此在插入排序中可以被当做哨兵.

    但上面这一步, 仅仅是将 左右排序, 中间部分还没有进行排序;

        if (less < e1 && e5 < great) {

    如果中间部分数组过大的话

        while (a[less] == pivot1) {
            ++less;
        }

        while (a[great] == pivot2) {
            --great;
        }

        /*   left part         center part                  right part
        * +----------------------------------------------------------+
        * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
        * +----------------------------------------------------------+
        *              ^                        ^       ^
        *              |                        |       |
        *             less                      k     great
        ******************************************************/

        outer:
        for (int k = less - 1; ++k <= great; ) {
            int ak = a[k];
            if (ak == pivot1) { // Move a[k] to left part
                a[k] = a[less];
                a[less] = ak;
                ++less;
            } else if (ak == pivot2) { // Move a[k] to right part
                while (a[great] == pivot2) {
                    if (great-- == k) {
                        break outer;
                    }
                }
                if (a[great] == pivot1) { // a[great] < pivot2
                    a[k] = a[less];
                    a[less] = pivot1;
                    ++less;
                } else { // pivot1 < a[great] < pivot2
                    a[k] = a[great];
                }
                a[great] = ak;
                --great;
            }
        }
        }
        sort(a, less, great, false);

    这段代码就不再多做解释, 同样是 切分, 只是方式有所区别;

        } else { // Partitioning with one pivot

            int pivot = a[e3];

    这里也不再赘述, 如果不满足 , a[e1] 到 a[e5] 均不相等的话, 则取中间元素, 作为切分元素, 标准的 三向快速排序.

    那么回过头来, 看看 插入排序的一段代码: 即 leftmost为 false 时:

        for (int k = left; ++left <= right; k = ++left) {
            int a1 = a[k], a2 = a[left];

            if (a1 < a2) {
                a2 = a1; a1 = a[left];
            }
            while (a1 < a[--k]) {
                a[k + 2] = a[k];
            }
            a[++k + 1] = a1;

            while (a2 < a[--k]) {
                a[k + 1] = a[k];
            }
            a[k + 1] = a2;
        }
        int last = a[right];

        while (last < a[--right]) {
            a[right + 1] = a[right];
        }
        a[right + 1] = last;

    在这里, 当 leftmost为 false时, 被排序的部分 a[left - 1]为元素中最小的部分, 充当哨兵的角色. 于是采取 双插入排序, 理解起来也比较简单, 每次选取两个元素排序, 首先将两个元素中较大的 做插入排序, 而后在 第一个元素的插入位置作为起点将第二个元素插入排序;

    同时由于有哨兵存在, 所以少一个边界判定.

1. 主体部分二

        int[] run = new int[MAX_RUN_COUNT + 1];
        int count = 0; run[0] = left;

    结合下面代码一起来看这段话:

        // Check if the array is nearly sorted
        for (int k = left; k < right; run[count] = k) {
            if (a[k] < a[k + 1]) { // ascending
                while (++k <= right && a[k - 1] <= a[k]);
            } else if (a[k] > a[k + 1]) { // descending
                while (++k <= right && a[k - 1] >= a[k]);
                for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                    int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
                }
            } else { // equal
                for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                    if (--m == 0) {
                        sort(a, left, right, true);
                        return;
                    }
                }
            }

            if (++count == MAX_RUN_COUNT) {
                sort(a, left, right, true);
                return;
            }
        }

    在这里需要把握一下 count 的值, 以更好地估计, 每执行一次, k + 1, count的值等于子数组的数量. run[count]记录的是每一个子数组的起点元素的索引, 每个子数组的实际范围应该是:

    a[run[count]] ~ a[run[count + 1] - 1];

    其中第1个子数组 count 为0;

    如果最后的子数组不是单元素数组, 则 run[count] = k, 此时 k = right + 1, 否则的话,run[count] = k = right.

    从这里解释, 判断 数组的无序性, 用了一个比较有趣的理念, 任何一个数组都可以被拆分成 若干个有序数组, 无论是升序, 降序 或是等值数组, 在这里就是将数组拆分成若干个升序子数组.

    如果为升序, 则将子数组倒序即可.

    run存储的则是 子数组的 每一个开始和结束的下标, 在这里需要注意的两个地方是:

    在 划分子数组时 的判等条件, 在之前的 排序算法的学习时, 归并排序对于含有大量重复元素的排序效率并不高, 而三向快速排序无疑是极为适合的,所以当重复元素连续超过33个的时候, 即认为数组中重复元素过多, 需要选择三向快速排序.

    当 ++count 达到最大值 67, 也就是子数组数超过当前限度, 表明数组的无序性较强, 则切换成快速排序.

        if (run[count] == right++) { // The last run contains one element
            run[++count] = right;
        } else if (count == 1) { // The array is already sorted
            return;
        }

    在这里又有所不同: 如果最后一个子数组为单元素子数组, 则此时的 run[count] = right; 否则为 right + 1; 为了保证末尾始终指向 right + 1, 所以需要用这种方式来满足, 至于原因, 下文解答.

    如果划分后，原数组中的最后一个元素并不是独自为一个子序列,此时run数组的最后一个元素已经是哨兵，就不需要再添加了。

        byte odd = 0;
        for (int n = 1; (n <<= 1) < count; odd ^= 1);

    这段判断子数组是否是 2的幂次, 如果是2的幂次, 在归并排序中, 数组恰好被切分成对等的子数组, 在之前的 排序学习中, 我们在 归并排序中, 有一种优化的方式是从目标数组拷贝到源数组, 通过这种将目标数组与源数组来回转换的方式 , 实现了 不需要在过程中每次都对 数组进行复制.

    但当时是在迭代中使用, 而这里则是通过递归的方式.

    所以需要通过 odd决定 第一次被拷贝的源数组究竟是 a 还是 b;

        // Use or create temporary array b for merging
        int[] b;                 // temp array; alternates with a
        int ao, bo;              // array offsets from 'left'
        int blen = right - left; // space needed for b
        if (work == null || workLen < blen || workBase + blen > work.length) {
            work = new int[blen];
            workBase = 0;
        }
        if (odd == 0) {
            System.arraycopy(a, left, work, workBase, blen);
            b = a;
            bo = 0;
            a = work;
            ao = workBase - left;
        } else {
            b = work;
            ao = 0;
            bo = workBase - left;
        }

    这段代码正是确定了第一次的目标数组和源数组.

        // Merging
        for (int last; count > 1; count = last) {
            for (int k = (last = 0) + 2; k <= count; k += 2) {
                int hi = run[k], mi = run[k - 1];
                for (int i = run[k - 2], p = i, q = mi; i < hi; ++i) {
                    if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) {
                        b[i + bo] = a[p++ + ao];
                    } else {
                        b[i + bo] = a[q++ + ao];
                    }
                }
                //生成第二轮归并的索引
                run[++last] = hi;
            }
            //如果子数组个数为奇数, 直接填充至最末端
            if ((count & 1) != 0) {
                for (int i = right, lo = run[count - 1]; --i >= lo;
                    b[i + bo] = a[i + ao]
                );
                run[++last] = right;
            }
            int[] t = a; a = b; b = t;
            int o = ao; ao = bo; bo = o;
        }

    观察不难发现, 在merge归并排序中, 这里并没有采取我们所更容易操作理解的 递归 的方式来进行排序,也不难理解, 节省空间. 但这段代码写的还是很精巧, 将原本需要四个判断的合并成两行代码;

    一直以来对 数组个数为奇数时 直接合并进已排序数组表示非常困惑, 却一直忽略了 run[++last] = right; 这句代码, 它表示, 不仅仅是合并进已排序数组, 同时也保留这个子数组, 作为一个单独的子数组参与下次的排序;

    至于之前提到过 对于最后一个子数组所含元素个数不同, 有不同的处理方式, 从而保证了末尾始终未 right + 1, 因为在归并排序中, 内循环的判断代码:

        for (int i = run[k - 2], p = i, q = mi; i < hi; ++i)

    为 i < hi, 在代码中的 right值为 array.length - 1; 为了不对末尾元素再多做区别, 简化代码, 所以使得 run[count] 始终等于 right + 1, 也就是数组长度.

1. 总结

    代码分析到这里已经结束了, 也没有遗留的问题点, 心得也就总结一下:

    1. 排序算法的选择:

        不难发现, 在数组不大的情况下, 优先选择快速排序, 如果更小, 则用插入排序解决问题. 甚至于快速排序也做了优化.

        在数组元素较多的时候, 要分几种情况:

        数组无序性较强, 或者重复元素较多, 这两种情况都采取快速排序.

        数组有序性还不错, 则采用归并排序.

    1. 算法的优化

        1. 快速排序的处理上, 采取了双轴快速排序, 省去了打乱数组选择 切分点的方式, 做了一步简化. 但我们又知道, 正是为了防止极端情况的出现, 因此才要在原来的方式中打乱数组. 所以 快速排序中将5个轴心进行排序, 将数组均分, 等等思路都是相当好的.

        1. 不仅如此, 如果第一次切分中间元素仍然过多, 又用了三向快速排序的方式进行处理. 比起打乱数组 是一种更好的选择, 虽然, 代码变长了.

        1. 快速排序, 也仅仅只进行了一次递归(包含大量重复元素的大容量数组除外), 同时巧妙利用了 双轴快速排序中出现的边界元素, 充当哨兵的角色, 选择了 双插入排序 用更快地速度处理数组.

        1. 双插入排序 对插入排序的进一步优化. 选择了两个元素中较大的先进行插入, 其次插入较小的, 思路相当精巧有趣.

        1. 数组有序行的校验, 这块代码比较有趣, 在有序性校验的同时, 用了一个容量不大的数组run[] 将原数组进行切分, 却仅仅只保存索引, 快捷方便, 同时在校验的时候, 将升序 等值做了进一步处理优化, 方便了下一步的处理.

        1. 归并排序的核心代码优化, 同样的, 借用了辅助数组, 使得处理数组更为方便, if (q >= hi || p < mi && a[p + ao] <= a[q + ao]) 简洁快速的比较, 进一步优化了原先的比较方式.

        1. 在迭代的同时, 不需要对数组进行复制排序, 之前的算法中有实现这点, 但是用的是递归的方式进行处理. 进一步优化数组, 节省空间.

    1. 读代码的心得

        1. 代码优化往往就是在那一两行读不懂的代码中, 还是得打破砂锅问到底的态度.

        1. 需要细致, 争取弄明白每一个变量究竟是为了什么, 起到怎样的作用.
        当然这点我觉得是在研究算法类代码时, 因为优化速度, 内存空间比较重要, 另一类巧取思路, 就可大而化之, 不必读懂每一句代码.把握核心点即可.

        1. 还是要对算法有一定了解, 甚至于其他基础有所了解, 对艺术不懂的人, 即使原样复现, 又怎么能理解 巧夺天工的技艺, 天马行空的想象?

参考:

[http://blog.csdn.net/octopusflying/article/details/52388012](http://blog.csdn.net/octopusflying/article/details/52388012)