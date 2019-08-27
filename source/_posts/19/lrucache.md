# lrucache
LruCache采用的缓存算法为LRU(Least Recently Used)，即最近最少使用算法。这一算法的核心思想是当缓存数据达到预设上限后，会优先淘汰近期最少使用的缓存对象。

```java
class LRUCache {
    HashMap<Object, Node> map;
    Node head;
    Node tail;
    long capacity;
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<Object, Node>();
    }
    
    public Object get(int key) {
        Node node = map.get(key);
        if (node != null) {
            remove(node, false);
            setHead(node);
            return node.value;
        }
        return -1;
    }

    public void put(int key, int value) {
        Node node = map.get(key);
        if (node != null) {
            node.value = value;
            remove(node, false);
        } else{
            node = new Node<>( key, value);
            if (map.size() >= capacity) {
                // 超过容量， 移除最后一个
                remove(tail, true);
            }
            map.put(key , node);
        }
        // 设置为head
        setHead(node);
    }

    private void setHead(Node node) {
        if (head != null) {
            node.next = head;
            head.pre = node;
        }
        head = node;
        if (tail == null) {
            tail = node;
        }
    }

    // 从链表上移除
    /**
     * @param node 被移除的节点
     * @param keyR  是否从map中也移除
     */
    private void remove(Node node, boolean keyR) {
        // 判断当前是否是头节点
        if (node != head) {
            node.pre.next = node.next;
        } else {
            head = node.next;
        }

        if (node != tail) {
            node.next.pre = node.pre;
        } else {
            tail = node.pre;
        }

        // 清空当前节点的关联
        node.clean();

        if (keyR) {
            map.remove(node.key);
        }
    }

    class Node<T> {
        private Node next;
        public Node pre;
        public T value;
        public int key;

        public Node(int key, T value) {
            this.value = value;
            this.key = key;
        }
        public void clean() {
            this.pre = null;
            this.next = null;
        }
    }
    public static void main(String[] args) {
         LRUCache cache = new LRUCache( 2 /* 缓存容量 */ );
        cache.put(1, 1);
        cache.put(2, 2);
        cache.get(1);       // 返回  1
        cache.put(3, 3);    // 该操作会使得密钥 2 作废
        cache.get(2);       // 返回 -1 (未找到)
        cache.put(4, 4);    // 该操作会使得密钥 1 作废
        cache.get(1);       // 返回 -1 (未找到)
        cache.get(3);        //返回  3
        cache.get(4);        //返回  4
    }
}
```
![](https://raw.githubusercontent.com/mxz1994/note/master/lrucache.png)


## 二 LruCache增加时效清退机制
在业务场景中，Redis中的广告数据有可能做修改。服务本身作为数据的使用方，无法感知到数据源的变化。当缓存的命中率较高或者部分数据在较长时间内多次命中，可能出现数据失效的情况。即数据源发生了变化，但服务无法及时更新数据。针对这一业务场景，增加了时效清退机制。

缓存数据单元将数据进入LruCache的时间戳与数据一起缓存下来。缓存过期时间表示缓存单元缓存的时间上限。查询中的时效性判断表示查询时的时间戳与缓存时间戳的差值超过缓存过期时间，则强制将此数据清空，重新请求Redis获取数据做缓存。

优点：在查询中做时效性判断可以最低程度的减少时效判断对服务的中断。当LruCache预设上限较低时，定期做全量数据清理对于服务本身影响较小。但如果LruCache的预设上限非常高，则一次全量数据清理耗时可能达到秒级甚至分钟级，将严重阻断服务本身的运行。所以将时效性判断加入到查询中，只对单一的缓存单元做时效性判断，在服务性能和数据有效性之间做了折中，满足业务需求。

## 三：高QPS下HashLruCache的应用

高频率更新会导致第一个
HashLruCache  根据Hash取模 进行分段更新 ，将缓存数据分散到N个LruCache上 参考ConcurrentHashMap


## 四: 零拷贝
理想的情况是LruCache对外仅仅提供数据地址，即数据指针。使用方在业务需要使用的地方通过数据指针获取数据。这样可以将复杂的数据拷贝操作变为简单的地址拷贝，大量减少拷贝操作的性能消耗，即数据的零拷贝机制。直接的零拷贝机制存在安全隐患，即由于LruCache中的时效清退机制，可能会出现某一数据已经过期被删除，但是使用方仍然通过持有失效的数据指针来获取该数据
通过拷贝指针替换拷贝数据，大量降低了获取复杂业务数据的耗时，同时将临界区减小到最小