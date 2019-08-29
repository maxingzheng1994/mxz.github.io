# ArrayList 和 LinkedList
1. 为什么实现 RandomAccess接口
  只是一个接口标志
  ``` java
      @SuppressWarnings("unchecked")
    public static <T> int binarySearch(List<? extends T> list, T key, Comparator<? super T> c) {
        if (c==null)
            return binarySearch((List<? extends Comparable<? super T>>) list, key);

        if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
            return Collections.indexedBinarySearch(list, key, c);
        else
            return Collections.iteratorBinarySearch(list, key, c);
    }
  ```
一个使用迭代器遍历一个使用 普通遍历
分别遍历 ArrayList(查找快，增删慢) 和LinkedList(增删快，查找慢)
Arraylist 正常遍历快   LinkedList  迭代器遍历快
LinkedList  
正常遍历  for（）{get(i)}  二分法查找需要好久
```java
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
迭代器遍历
```java
        public E next() {
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }
```
所以遍历的时候 instanceof 判断是否实现RandomAccess 接口

