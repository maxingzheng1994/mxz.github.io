---
title: base
date: 2019-02-13 19:41:58
tags:
categories: java
---
1、static方法，因为被static修饰的方法是属于类的，而不是属于实例的

2、final方法，因为被final修饰的方法无法被子类重写

3、private方法和protected方法，前者是因为被private修饰的方法对子类不可见，后者是因为尽管被protected修饰的方法可以被子类见到，也可以被子类重写，但是它是无法被外部所引用的，一个不能被外部引用的方法，怎么能谈多态呢

final 
    1、被final修饰的类不可以被继承

2、被final修饰的方法不可以被重写 (JVM会尝试为之寻求内联，这对于提升Java的效率是非常重要的。因此，假如能确定方法不会被继承，那么尽量将方法定义为final的)

3、被final修饰的变量不可以被改变 不可变的是变量的引用，而不是引用指向的内容 数组内容也可改变

static 
    静态资源的加载顺序是严格按照静态资源的定义顺序来加载的
   静态代码块对于定义在它之后的静态变量，可以赋值，但是不能访问。
    静态代码块是严格按照父类静态代码块->子类静态代码块的顺序加载的，且只加载一次
    静态内部类  单例使用静态内部类
    import static

序列化和反序列化
    Java为用户定义了默认的序列化、反序列化方法，其实就是ObjectOutputStream的defaultWriteObject方法和ObjectInputStream的defaultReadObject方法
     File file = new File("D:" + File.separator + "s.txt");
 4     OutputStream os = new FileOutputStream(file);  
 5     ObjectOutputStream oos = new ObjectOutputStream(os);
 6     oos.writeObject(new SerializableObject("str0", "str1"));
 7     oos.close();
 8         
 9     InputStream is = new FileInputStream(file);
10     ObjectInputStream ois = new ObjectInputStream(is);
11     SerializableObject so = (SerializableObject)ois.readObject();
12     System.out.println("str0 = " + so.getStr0());
13     System.out.println("str1 = " + so.getStr1());
14     ois.close();

    被声明为transient的属性不会被序列化，这就是transient关键字的作用    被声明为static的属性不会被序列化，这个问题可以这么理解，序列化保存的是对象的状态，但是static修饰的变量是属于类的而不是属于对象的，因此序列化的时候不会序列化它
  
String 常量池  
StringBuffer和StringBuilder原理一样，无非是在底层维护了一个char数组，每次append的时候就往char数组里面放字符而已，在最终sb.toString()的时候，用一个new String()方法把char数组里面的内容都转成String，这样，整个过程中只产生了一个StringBuilder对象与一个String对象，非常节省空间。StringBuilder唯一的性能损耗点在于char数组不够的时候需要进行扩容，扩容需要进行数组拷贝，一定程度上降低了效率。

StringBuffer和StringBuilder用法一模一样，唯一的区别只是StringBuffer是线程安全的，它对所有方法都做了同步，StringBuilder是线程非安全的，所以在不涉及线程安全的场景，比如方法内部，尽量使用StringBuilder，避免同步带来的消耗。


Comparable(内比较器)和Comparator(外比较器 耦合性低 )的区别
实现Comparable接口的(compareTo(Ａ)方法)　　　创建一个比较器实现Comparator的compare(A,B) 对比两个

语法糖 (就是Java编译器在编译期间做的手脚)
    forEach 在编译的时候编译器会自动将对for这个关键字的使用转化为对目标的迭代器的使用                 
    Java将对于数组的foreach循环转换为对于这个数组每一个的循环引用。
    自动装箱  Integer a = 100; Integer b= 100  a== b (true)  Integer c = 200; Integer d = 200 c==d(false) 以128位分界线做了缓存的
    泛型: 静态资源不认识泛型。 所以我们要加上一个<T>，告诉static方法，后面的T是一个泛型 private static <T> T ifThenElse(boolean b, T first, T second)
    内部类: 内部类访问外部成员变量必须为final(final修饰的直接放入常量池，局部类就可以随便用了)  静态内部类只能访问其外部类的静态成员与静态方法  
            好处：(java不支持多继承，可实现内部类继承，内部类又可以访问外部类所有东西， 而且隐藏了自己身藏公与名)
            
Thread.sleep()  wait  sleep方法不会释放掉监视器(monitor)的所有权，而wait方法会释放掉监视器的所有权 所谓监视器，就是假如sleep方法和wait方法处于同步方法/同步方法块中，它们所持有的对象锁
协同式线程调度与抢占式线程调度。Thread.sleep(0) 作用避免假死  抢占了5mS 执行2ms后碰到sleep(0) 剩的3ms扔掉重新抢占
Reactor模式即反应器模式，是并发系统常用的多线程处理方式，用以节省系统的资源，提高系统的吞吐量

CAS ，Compare and Swap即比较并交换
有三个操作数：内存值V、旧的预期值A、要修改的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做并返回false。


多线程  利用多个cpu，防阻塞
Runnable接口和Callable接口的区别 Runnable接口中的run()方法的返回值是void，它做的事情只是纯粹地去执行run()方法中的代码而已；Callable接口中的call()方法是有返回值的，是一个泛型，和Future、FutureTask配合可以用来获取异步执行的结果。
重入锁  加锁后调用的方法加通向的锁， 锁计数+2