https://blog.csdn.net/zxd8080666/article/details/83214089

Synchronized与ReentrantLock区别总结（简单粗暴，一目了然） 原创
2018-10-22
14点赞

奋力奔跑的蜗牛

码龄5年

关注
这篇文章是关于这两个同步锁的简单总结比较，关于底层源码实现原理没有过多涉及，后面会有关于这两个同步锁的底层原理篇幅去介绍。

相似点：
这两种同步方式有很多相似之处，它们都是加锁方式同步，而且都是阻塞式的同步，也就是说当如果一个线程获得了对象锁，进入了同步块，其他访问该同步块的线程都必须阻塞在同步块外面等待，而进行线程阻塞和唤醒的代价是比较高的（操作系统需要在用户态与内核态之间来回切换，代价很高，不过可以通过对锁优化进行改善）。

功能区别：
这两种方式最大区别就是对于Synchronized来说，它是java语言的关键字，是原生语法层面的互斥，需要jvm实现。而ReentrantLock它是JDK 1.5之后提供的API层面的互斥锁，需要lock()和unlock()方法配合try/finally语句块来完成

便利性：很明显Synchronized的使用比较方便简洁，并且由编译器去保证锁的加锁和释放，而ReenTrantLock需要手工声明来加锁和释放锁，为了避免忘记手工释放锁造成死锁，所以最好在finally中声明释放锁。

锁的细粒度和灵活度：很明显ReenTrantLock优于Synchronized

性能的区别：
在Synchronized优化以前，synchronized的性能是比ReenTrantLock差很多的，但是自从Synchronized引入了偏向锁，轻量级锁（自旋锁）后，两者的性能就差不多了，在两种方法都可用的情况下，官方甚至建议使用synchronized，其实synchronized的优化我感觉就借鉴了ReenTrantLock中的CAS技术。都是试图在用户态就把加锁问题解决，避免进入内核态的线程阻塞。

1.Synchronized
Synchronized进过编译，会在同步块的前后分别形成monitorenter和monitorexit这个两个字节码指令。在执行monitorenter指令时，首先要尝试获取对象锁。如果这个对象没被锁定，或者当前线程已经拥有了那个对象锁，把锁的计算器加1，相应的，在执行monitorexit指令时会将锁计算器就减1，当计算器为0时，锁就被释放了。如果获取对象锁失败，那当前线程就要阻塞，直到对象锁被另一个线程释放为止。

public class SynDemo{

	public static void main(String[] arg){
		Runnable t1=new MyThread();
		new Thread(t1,"t1").start();
		new Thread(t1,"t2").start();
	}

}
class MyThread implements Runnable {

	@Override
	public void run() {
		synchronized (this) {
			for(int i=0;i<10;i++)
				System.out.println(Thread.currentThread().getName()+":"+i);
		}
		
	}

}

2.ReentrantLock
由于ReentrantLock是java.util.concurrent包下提供的一套互斥锁，相比Synchronized，ReentrantLock类提供了一些高级功能，主要有以下3项：

        1.等待可中断，持有锁的线程长期不释放的时候，正在等待的线程可以选择放弃等待，这相当于Synchronized来说可以避免出现死锁的情况。通过lock.lockInterruptibly()来实现这个机制。

        2.公平锁，多个线程等待同一个锁时，必须按照申请锁的时间顺序获得锁，Synchronized锁非公平锁，ReentrantLock默认的构造函数是创建的非公平锁，可以通过参数true设为公平锁，但公平锁表现的性能不是很好。

公平锁、非公平锁的创建方式：

//创建一个非公平锁，默认是非公平锁
Lock lock = new ReentrantLock();
Lock lock = new ReentrantLock(false);

//创建一个公平锁，构造传参true
Lock lock = new ReentrantLock(true);
3.锁绑定多个条件，一个ReentrantLock对象可以同时绑定对个对象。ReenTrantLock提供了一个Condition（条件）类，用来实现分组唤醒需要唤醒的线程们，而不是像synchronized要么随机唤醒一个线程要么唤醒全部线程。

ReenTrantLock实现的原理：
之后还会总结一篇ReenTrantLock相关的原理底层原理分析，简单来说，ReenTrantLock的实现是一种自旋锁，通过循环调用CAS操作来实现加锁。它的性能比较好也是因为避免了使线程进入内核态的阻塞状态。想尽办法避免线程进入内核的阻塞状态是我们去分析和理解锁设计的关键钥匙。

什么情况下使用ReenTrantLock：
答案是，如果你需要实现ReenTrantLock的三个独有功能时。

ReentrantLock的用法如下：

public class SynDemo{

	public static void main(String[] arg){
		Runnable t1=new MyThread();
		new Thread(t1,"t1").start();
		new Thread(t1,"t2").start();
	}

}
class MyThread implements Runnable {

	private Lock lock=new ReentrantLock();
	public void run() {
			lock.lock();
			try{
				for(int i=0;i<5;i++)
					System.out.println(Thread.currentThread().getName()+":"+i);
			}finally{
				lock.unlock();
			}
	}

}
对ReentrantLock的源码分析这有一篇很好的文章

http://www.blogjava.net/zhanglongsr/articles/356782.html

后续在补充的问题：

Synchronized的原理

ReentrantLock的原理。

ReentrantLock为什么是可重入的。

公平锁和非公平锁是什么？有什么区别。

