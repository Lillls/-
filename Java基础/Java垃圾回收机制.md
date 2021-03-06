# Java垃圾回收机制



### Java的四种引用

1. 强引用 （Strong Reference）:

   ```java
   Object obj = new Object()
   ```

   这类引用是Java程序中最普遍的。只要强引用还存在，垃圾收集器就永远不会回收掉被引用的对象。

2. 软引用 （Soft Reference）

   ```java
   static String s = "";
   SoftReference<String> softReference = new SoftReference<>(s);
   ```

   它用来描述一些可能还有用，但并非必须的对象。在系统内存不够用时，这类引用关联的对象将被垃圾收集器回收。JDK1.2之后提供了`SoftReference类来实现软引用。

   举个例子：

   图片加载框架中，通过软引用实现内存缓存。伪代码

   ```java
   //如果缓存过了
   if(mImageCache.containsKey(imageUrl)){
     SoftReference<Drawable> softReference = mImageCache.get(imageUrl);
     if（softReference.get()!=null）{
       return softReference.get(); 
     }
   }
   //如果没有缓存过或者缓存被清理了
   //下载图片资源
   ImageRes res = download(imageUrl)
   //将得到的图片存放到缓存中
   mImageCache.put(imageUrl, new SoftReference<Drawable>(drawable));
   return res;
   ```

   

3. 弱引用（Weak Reference）

   ```java
   static String s = "";
   WeakReference<String> reference = new WeakReference<>(s);
   ```

   它也是用来描述非须对象的，但它的强度比软引用更弱些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在JDK1.2之后，提供了`WeakReference`类来实现弱引用。

   例子见`ThreadLocal,ThreadMap`

   ThreadLocalMap类 该类时ThreadLocal的静态内部类，该Map使用开放地址法处理hash冲突的Map类，key为ThreadLocal对象，value为TheadLocal对象所对应的值value；

   其中Entry对象当中的Key值对TheadLocal的引用就是WeakReference

   ```java
   static class Entry extends WeakReference<ThreadLocal>
   {
        /* The value assocaiated with this ThreadLocal.*/
       Object value;
       Entry(ThreadLocal k,Object v)
       {
           super(k);
           value = v;
       }
   }
   ```

   这样当ThreadLocal对象除了Entry对象外没有其他引用的时候，在下一次垃圾回收发生时，该对象将被回收；

   这也就导致在使用Entry对象获取key值得时候，需要判断是否为空，如果为空，则说明已经被回收了，此时将value值手动清除即可；

   其实因为ThreadLocal牵涉到线程本地变量的操作，对于对象何时被清除，程序逻辑一般不太好实现，所以JDK设计者将其设置为了自动清除，

   其实大部分TheadLocal对象我们都将其设置为private static 对象，一般不会被弱引用清除掉；

4. 虚引用（Phantom Reference）

   最弱的一种引用关系，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的是希望能在这个对象被收集器回收时收到一个系统通知。JDK1.2之后提供了`PhantomReference` 类来实现虚引用。

## 可回收对象的判定

1. **引用计数法(Reference Counting Collector)** 

   给一个对象添加一个引用计数器，每当有一个地方引用它时，计数器的值就加1，当引用失效时计数器就减1，当计数器的值为0时，代表对象不再被使用，可以进行回收。

   引用计数法实现简单，判断效率也很高，但是它很难解决对象之间互相引用的问题。

   ```java
   	public static void main(String[] args) {
           Object object1 = new Object();
           Object object2 = new Object();
            
           object1.object = object2;
           object2.object = object1;
            
           object1 = null;
           object2 = null;
       }
   ```

   最后面两句将object1和object2赋值为null，也就是说object1和object2指向的对象已经不可能再被访问，但是由于它们互相引用对方，导致它们的引用计数器都不为0，那么垃圾收集器就永远不会回收它们。

   

2. **根搜索算法（Tracing Collector**）

   首先明白一个概念：GC Root

   GC Root包括四种：

   1. 虚拟机栈中引用的对象（栈桢中的本地变量表）

   2. 方法区类静态属性的变量

   3. 方法区常量引用的对象

   4. 本地方法栈中（即一般说的Native方法）引用的变量

       

   通过一系列叫做“GC Root”的节点，从这些节点开始向下搜索，搜索到的路径叫做引用链，当一个对象到GC Root没有任何引用链相连时，表明该对象不可用。可以被回收

# 垃圾回收算法

1. **标记清除算法（Mark-Sweep）**

   标记-清除（Mark-Sweep）算法是现代垃圾回收算法的思想基础。标记-清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。一种可行的实现是，在标记阶段，首先通过根节点，标记所有从根节点开始的可达对象。因此，未被标记的对象就是未被引用的垃圾对象。然后，在清除阶段，清除所有未被标记的对象。
   如图：

   ![](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-07-082011.jpg)

   优点是简单，容易实现。
   缺点是容易产生内存碎片，碎片太多可能会导致后续过程中需要为大对象分配空间时无法找到足够的空间而提前触发新的一次垃圾收集动作。

2. **标记整理算法**

   标记整理算法类似与标记清除算法，不过它标记完对象后，不仅对可回收的对象进行回收,还会将存活对象向一段移动,并更新指针.

   标记-整理算法是在标记-清除算法的基础上，又进行了对象的移动，因此成本更高，但是却解决了内存碎片的问题.

   ![](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-07-083247.jpg)

3. **复制算法**

   复制算法将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用的内存空间一次清理掉，这样一来就不容易出现内存碎片的问题。
   优缺点就是，实现简单，运行高效且不容易产生内存碎片，但是却对内存空间的使用做出了高昂的代价，因为能够使用的内存缩减到原来的一半。
   从算法原理我们可以看出，Copying算法的效率跟存活对象的数目多少有很大的关系，如果存活对象很多，那么Copying算法的效率将会大大降低。

   ![](http://xiaoyu-ipic.oss-cn-beijing.aliyuncs.com/2019-11-07-082338.jpg)

   

4. **分代回收算法**

   分代的垃圾回收策略，是基于这样一个事实：**不同的对象的生命周期是不一样的**。因此，不同生命周期的对象可以采取不同的回收算法，以便提高回收效率。

   - **年轻代（Young Generation）**

     1. **所有新生成的对象首先都是放在年轻代的**。年轻代的目标就是尽可能快速的收集掉那些生命周期短的对象。

     2. 新生代内存按照8:1:1的比例分为一个eden区和两个survivor(survivor0,survivor1)区。一个Eden区，两个 Survivor区(一般而言)。大部分对象在Eden区中生成。回收时先将eden区存活对象复制到一个survivor0区，然后清空eden区，当这个survivor0区也存放满了时，则将eden区和survivor0区存活对象复制到另一个survivor1区，然后清空eden和这个survivor0区，此时survivor0区是空的，然后将survivor0区和survivor1区交换，即保持survivor1区为空， 如此往复。
     3. 当survivor1区不足以存放 eden和survivor0的存活对象时，就将存活对象直接存放到老年代。若是老年代也满了就会触发一次Full GC，也就是新生代、老年代都进行回收
     4. 新生代发生的GC也叫做Minor GC，MinorGC发生频率比较高(不一定等Eden区满了才触发)

     采用的算法是复制算法，新创建的对象都是放在Eden空间，这是很频繁的，尤其是大量的局部变量产生的临时对象，这些对象绝大部分都应该马上被回收，能**存活下来被转移到survivor空间的往往不多**。所以，设置较大的Eden空间和较小的Survivor空间是合理的，大大提高了内存的使用率，缓解了Copying算法的缺点。

   - **年老代（Old Generation）**

     1. 在年轻代中经历了N次垃圾回收后仍然存活的对象，就会被放到年老代中。因此，可以认为年老代中存放的都是一些生命周期较长的对象
     2. 内存比新生代也大很多(大概比例是1:2)，当老年代内存满时触发Major GC即Full GC，Full GC发生频率比较低，老年代对象存活时间比较长，存活率标记高。

     老年代因为对象存活率高，没有额外空间对它进行分配担保，就必须使用“标记清理”或者“标记整理”算法来进行回收。

# 触发GC的类型

- GC_FOR_MALLOC: 表示是在堆上分配对象时内存不足触发的GC。
- GC_CONCURRENT: 当我们应用程序的堆内存达到一定量，或者可以理解为快要满的时候，系统会自动触发GC操作来释放内存。
- GC_EXPLICIT: 表示是应用程序调用System.gc、VMRuntime.gc接口或者收到SIGUSR1信号时触发的GC。
- GC_BEFORE_OOM: 表示是在准备抛OOM异常之前进行的最后努力而触发的GC。

虚拟机会把触发GC的信息打印出来,可以查看虚拟机这些信息,来进行分析优化.