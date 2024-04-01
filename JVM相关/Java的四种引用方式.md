# Java的四种引用方式

java对象的引用包括强引用，软引用，弱引用，虚引用。

Java中提供这四种引用类型主要有两个目的：

​	一是可以让程序员通过代码的方式决定某些对象的生命周期；

​	二是有利于JVM进行垃圾回收。



### 强引用

强引用是指创建一个对象并把这个对象赋给一个引用变量。例如：

```java
Object object =new Object();
String str ="hello";
```

强引用有引用变量指向时永远不会被垃圾回收，JVM宁愿抛出OutOfMemory错误也不会回收这种对象。可以显示地将引用赋值为null，JVM就会在合适的时间回收该对象。





### 软引用

如果一个对象具有软引用，内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，垃圾收集线程会在虚拟机抛出OutOfMemoryError之前回收软可及对象，而且**虚拟机会尽可能优先回收长时间闲置不用的软可及对象，对那些刚刚构建的或刚刚使用过的“新”软可反对象会被虚拟机尽可能保留**（软引用有两个特殊的变量：clock——记录上一次GC的时间；timestamp——记录该变量上一次get的时间。这两个变量之差表示**这个软引用对象距离上次GC时一直没被使用的时间**，当这个差值大于阈值_max_interval（该阈值由上一次GC后剩余可用空间计算得出，空间越大，该阈值越大）时，则该对象标记为可废弃的，将在下一次GC时回收）。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存,比如网页缓存、图片缓存等。使用软引用能防止内存泄露，增强程序的健壮性。

```java
MyObject aRef = new MyObject();  
SoftReference aSoftRef = new SoftReference(aRef); 
aRef = null;
```

将MyObject对象变成软引用对象，并结束对这个对象的强引用。

```java
MyObject anotherRef = (MyObject) aSoftRef.get();
```

SoftReference保存一个Java对象的软引用后，在垃圾线程对Java对象回收前，可以通过SoftReference类提供的get方法获得该Java对象的强引用。但是一旦垃圾线程回收了该Java对象，get方法返回null。

作为一个Java对象，SoftReference对象除了具有保存软引用的特殊性之外，也具有Java对象的一般性。所以当软可及对象被回收之后，这个SoftReference对象已经不再具有存在的价值，需要一个适当的清除机制，避免大量SoftReference对象带来的内存泄漏。在java.lang.ref包里还提供了ReferenceQueue。如果在创建SoftReference对象的时候，使用了一个ReferenceQueue对象作为参数提供给SoftReference的构造方法，如:

```java
ReferenceQueue queue = new ReferenceQueue();  
SoftReference ref = new SoftReference(aMyObject, queue);
```

当这个SoftReference所软引用的aMyOhject被垃圾收集器回收的同时，ref所强引用的SoftReference对象被列入ReferenceQueue。也就是说，ReferenceQueue中保存的对象是Reference对象，而且是已经失去了它所软引用的对象的Reference对象。另外从ReferenceQueue这个名字也可以看出，它是一个队列，当我们调用它的poll()方法的时候，如果这个队列中不是空队列，那么将返回队列前面的那个Reference对象。

在任何时候，我们都可以调用ReferenceQueue的poll()方法来检查是否有它所关心的非强可及对象被回收。如果队列为空，将返回一个null,否则该方法返回队列中前面的一个Reference对象。利用这个方法，我们可以检查哪个SoftReference所软引用的对象已经被回收。于是我们可以把这些失去所软引用的对象的SoftReference对象清除掉。





### 弱引用

弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。

```java
public static void main(String[] args) {  
    WeakReference<People> reference = new WeakReference<People>(new People("zhouqian",20));  
    System.out.println(reference.get());  
    System.gc(); 
    System.out.println(reference.get());  
}
```

第二个输出结果是null，这说明只要JVM进行垃圾回收，被弱引用关联的对象必定会被回收掉。不过要注意的是，这里所说的被弱引用关联的对象是指只有弱引用与之关联，如果存在强引用同时与之关联，则进行垃圾回收时也不会回收该对象（软引用也是如此）。

```java
public static void main(String[] args) {  
    People people = new People("zhouqian",20);  
    WeakReference<People> reference = new WeakReference<People>(people);
    System.out.println(reference.get());  
    System.gc();  
    System.out.println(reference.get());  
}
```

两个输出结果相同。





### 虚引用

虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。 **为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。** 虚引用必须和引用队列关联使用，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之关联的引用队列中。

```java
public static void main(String[] args) {  
    ReferenceQueue<String> queue = new ReferenceQueue<String>();  
    PhantomReference<String> pr = new PhantomReference<String>(new String("hello"), queue);  
    System.out.println(pr.get());  
}
```