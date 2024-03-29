## **一、java内存模型**

```
public class HelloWorld {
    private int data = 0;
    public increment() {
        data++;    
    }
}
HelloWorld helloWorld = new HelloWorld(); //对象其实是在堆内存里，包含实例变量
//线程1
new Thread() {
    public void run() {
        helloWorld.increment();    
    }
}.start();
//线程2
new Thread() {
    public void run() {
        helloWorld.increment();    
    }
}.start();
```

以上代码实际执行如下：

![java内存模型](1-java内存模型与volatile关键字.assets/java内存模型.png)

1. Java内存模型中，所有的变量都存储在主内存中，每条线程还有自己的工作内存，线程对变量的所有操作都必须在工作内存中进行，不能直接读写主内存中的变量；不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需通过主内存完成。

2. 如果将Java内存模型和Java堆、栈比较，主内存对应Java堆中的对象实例部分，工作内存对应虚拟机栈中的部分。

3. 主内存和工作内存之间需要交互，Java内存模型中有8种原子操作：

   1）lock：作用于主内存变量，将其标识为线程独占。

   2）unlock：作用于主内存变量，将其从锁定状态释放，释放后才可被其他线程锁定。

   3）read：作用于主内存的变量，将一个变量从主内存中传输到工作内存中，以便随后的load动作使用。

   4）load：作用于工作内存中的变量，把read操作从主内存中得到的变量值放入工作内存的变量副本中。

   5）use：作用于工作内存的变量，把工作内存中的一个变量的值传递给执行引擎，当虚拟机需要使用到变量的值的字节码指令时会执行这个操作。

   6）assign：作用于工作内存的变量，把一个从执行引擎接受到的值赋给工作内存中的变量。当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。

   7）store：作用于工作内存中的变量，把工作内存中的一个变量的值传送到主内存中，以便之后write操作使用。

   8）write：作用于主内存中的变量，把store操作从工作内存中得到的变量值放入主内存的变量中。

## **二、内存模型特征**

Java内存模型就是围绕着在并发过程中如何处理原子性、可见性和有序性这三个特性来建立的。

**1、原子性（Atomicity）**

- 由Java内存模型类直接保证原子性变量的操作包括read、load、use、assign、store和write。可以认为基本数据类型的访问读写是具备原子性的。
- 另外需要注意的是非原子性协定，即：Java内存模型中定了一条相对宽松的规定，允许虚拟机将没有被volatile修饰的64位数据读写操作分为两次32位操作进行，也就是说，对long、double这种64位的数据类型的read、load、store和write操作可以不保证原子性操作。这种情况在64的虚拟机下是不会发生的，64位虚拟机下是原子性的，但是在32位的虚拟机下，这组操作不是原子性的。如果要更大范围的原子性保证，Java内存模型提供了lock和unlock操作，字节码指令对应的是monitorenter和monitorexit，反映到Java代码就是synchronized了。

**2、可见性（Visibility）**

```
data = 0;
//线程1
new Thread() {
    public void run() {
        data++;
    }
}.start();
//线程2
new Thread() {
    public void run() {
        while(data == 0) {
            Thread.sleep(100);        
        }  
    }
}.start();
```

- 没有可见性：线程1执行data++之后，主内存中的data值已经变了，一段时间内，线程2中仍然在使用自己工作内存中的data=0，导致线程2没有正常结束。这个叫没有可见性。
- 有可见性：线程1执行data++之后，主内存中的data值已经变了，线程2立即能感知到这个修改，线程2中的data值会置为失效状态，线程2再次使用data值的时候会重新从主内存中加载data=1到自己的工作内存中去，正常结束线程2。这个叫有可见性。
- 可见性是指一个线程修改了共享变量的值，其他线程能立即得知这个修改。Java内存模型是通过在变量修改后立即将最新的值同步到主内存、在变量读取前从主内存再次读取最新值来保证可见性的。volatile变量与普通变量的区别是，volatile变量的特殊规则保证了最新值能立刻同步到主内存。而普通变量不能保证这一点。除了volatile外，还有synchronized和final关键字能保证可见性。synchronized的可见性对应Java内存模型中lock和unlock，lock会清空工作内存中变量的副本，unlock操作前会将变量同步到主内存。

**3、有序性（ordering）**

对于代码，同时还有一个问题是指令重排序，编译器和指令器，有的时候为了提高代码执行效率，会将指令重排序，比如下面的代码：

```
flag = false;
//线程1
new Thread() {
    public void run() {
        prepare();  //准备资源
        flag = true;
    }
}.start();
//线程2
new Thread() {
    public void run() {
        while(!flag) {
            Thread.sleep(100);        
        }
        execute();  //基于线程1->prepare()准备好的资源,执行后续操作
    }
}.start();
```

- 没有有序性：指令重排序后，可能flag=true 在 prepare()之前执行，就会导致线程2的execute()缺少资源，从而导致程序异常。
- 有有序性：严格按照程序顺序执行，不会发生指令重排序。

**java中有一个happens-before原则：**

编译器、指令器可能对代码重排序，乱排，要守一定的规则，happens-before原则，只要符合happens-before的原则，那么就不能胡乱重排，如果不符合这些规则的话，那就可以自己排序。

- **程序次序规则**：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作
- **锁定规则**：一个unLock操作先行发生于后面对同一个锁的lock操作，比如说在代码里有先对一个lock.lock()，lock.unlock()，lock.lock()
- **volatile****变量规则**：对一个volatile变量的写操作先行发生于后面对这个volatile变量的读操作，volatile变量写，再是读，必须保证是先写，再读
- **传递规则**：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C
- **线程启动规则**：Thread对象的start()方法先行发生于此线程的每个一个动作，thread.start()，thread.interrupt()
- **线程中断规则**：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生
- **线程终结规则**：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行
- **对象终结规则**：一个对象的初始化完成先行发生于他的finalize()方法的开始

## **三、volatile关键字**

volatile关键字是用来解决可见性和有序性，在有些罕见的条件之下，可以有限的保证原子性，他主要不是用来保证原子性的

有序性：volatile要求的是，volatile变量操作前面的代码一定不能指令重排到volatile变量操作后面，volatile后面的代码也不能指令重排到volatile前面。

**volatile底层原理，如何实现保证可见性的呢？如何实现保证有序性的呢？**

（1）lock指令：volatile保证可见性 ：lock前缀指令 + MESI缓存一致性协议

对volatile修饰的变量，执行写操作的话，JVM会发送一条lock前缀指令（此lock与锁操作lock()不同）给CPU，CPU在计算完之后会立即将这个值写回主内存，同时因为有MESI缓存一致性协议，所以各个CPU都会对总线进行嗅探，自己本地缓存中的数据是否被别人修改 ，如果发现别人修改了某个缓存的数据，那么CPU就会将自己本地缓存的数据过期掉，然后这个CPU上执行的线程在读取那个变量的时候，就会从主内存重新加载最新的数据了 

（2）内存屏障：volatile禁止指令重排序 

![内存屏障](1-java内存模型与volatile关键字.assets/内存屏障.png)