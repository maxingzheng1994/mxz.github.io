# Spliterator
Spliterator是什么？
Spliterator就是为了并行遍历元素而设计的一个迭代器
![](_v_images/20190708174505114_26691.png)

ArrayList 中的 ArrayListSpliterator
```java
 static final class ArrayListSpliterator<E> implements Spliterator<E> {
        
        private final ArrayList<E> list; 
        private int index; // current index, modified on advance/split
        private int fence; // -1 until used; then one past last index
        private int expectedModCount; // initialized when fence set

        /** Create new spliterator covering the given  range */
        ArrayListSpliterator(ArrayList<E> list, int origin, int fence,
                             int expectedModCount) {
            this.list = list; 
            this.index = origin;
            this.fence = fence;  //没用到的时候初始值是-1
            this.expectedModCount = expectedModCount;
        }
        //获取结束位置 (fence)
        private int getFence() { 
            int hi; 
            ArrayList<E> lst;
            if ((hi = fence) < 0) {  // fence = -1 最早的初始化
                if ((lst = list) == null)
                    hi = fence = 0;
                else {
                    expectedModCount = lst.modCount;
                    hi = fence = lst.size;
                }
            }
            return hi;
        }
        //分割list 返回一个新的spliterator, list 没变化
        public ArrayListSpliterator<E> trySplit() {
            int hi = getFence(), lo = index, mid = (lo + hi) >>> 1;
            return (lo >= mid) ? null : // divide range in half unless too small
                new ArrayListSpliterator<E>(list, lo, index = mid,
                                            expectedModCount); // 创建 前半段
        }

        // 返回true， 仍有元素尚未处理完毕
        public boolean tryAdvance(Consumer<? super E> action) {
            if (action == null)
                throw new NullPointerException();
            int hi = getFence(), i = index;
            if (i < hi) {
                index = i + 1;
                @SuppressWarnings("unchecked") E e = (E)list.elementData[i];
                action.accept(e);
                if (list.modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                return true;
            }
            return false;
        }
        
        // 顺序遍历所有剩下的元素
        public void forEachRemaining(Consumer<? super E> action) {
            int i, hi, mc; // hoist accesses and checks from loop
            ArrayList<E> lst; Object[] a;
            if (action == null)
                throw new NullPointerException();
            if ((lst = list) != null && (a = lst.elementData) != null) {
                if ((hi = fence) < 0) {
                    mc = lst.modCount;
                    hi = lst.size;
                }
                else
                    mc = expectedModCount;
                if ((i = index) >= 0 && (index = hi) <= a.length) {
                    for (; i < hi; ++i) {
                        @SuppressWarnings("unchecked") E e = (E) a[i];
                        action.accept(e);
                    }
                    if (lst.modCount == mc)
                        return;
                }
            }
            throw new ConcurrentModificationException();
        }
        // 剩余元素
        public long estimateSize() {
            return (long) (getFence() - index);
        }
        // 特征值
        public int characteristics() {
            return Spliterator.ORDERED | Spliterator.SIZED | Spliterator.SUBSIZED;
        }
    }
```

[github](https://github.com/mxz1994/factory/tree/master/sb2019/src/main/java/com/mxz/spliterator)
