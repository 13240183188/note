https://mp.weixin.qq.com/s?src=11&timestamp=1600066648&ver=2583&signature=fdxQuP7b5jAlVKIOiyU3V7pWIFFue-6ShSJsJ5V79yvY0JcnCUSeOaxaIl3*j4jxNCrwXjOJdcv51SaWhwF5vVe5Ty9jINZH0iHbNbT-IqIxFxSrfFmOVNZnP*2j2Grz&new=1

面试官会经常而且非常严肃的问出：请你解释下Runnable或Thread的区别？新手非常容易上当，不知如何回答，就胡乱吹一通。鄙人今天告诉你们这二者本身实现的效果没有本质区别，就是接口和类的区别。问出这个问题的面试官本身就是个二流子！如果非要说区别，请看如下：



Thread的实现方式是继承其Thread类

Runnable的实现方式是实现其Runnable接口

Runnable接口支持多继承，但基本上用不到

Thread是实现了Runnable接口并进行了扩展，而Thread和Runnable的实质是实现与被实现的关系，不是同类型东西。

网络上流传的最大的一个错误结论：Thread不可以可以实现多个线程间的资源共享，而Runnable更容易实现！这是错的！网络得出此结论的例子如下：

public class Test {
public static void main(String[] args) {
new MyThreadTest().start();
new MyThreadTest().start();

    }

     static class MyThread extends Thread{
        private int goods= 5;
        public void run(){
            while(true){
                System.out.println("goods= " + goods--);
                if(ticket < 0){
                    break;
                }
            }
        }
    }
}
运行结果如下：
goods= 5
goods= 5
goods= 4
goods= 3
goods= 2
goods= 1
goods= 0
goods= 4
goods= 3
goods= 2
goods= 1
goods= 0

Process finished with exit code 0

很显然，总共5个商品但卖了10。这就像两个商人再卖同一商品，原因稍后分析。现在看看使用runnable的结果：
public class Test2 {
public static void main(String[] args) {
// TODO Auto-generated method stub
MyThreadTest2 mt=new MyThreadTest2();
new Thread(mt).start();
new Thread(mt).start();


    }
    static class MyThreadTest2 implements Runnable{
        private int ticket = 5;
        public void run(){
            while(true){
                System.out.println("goods= " + ticket--);
                if(ticket < 0){
                    break;
                }
            }
        }
    }
}
运行结果如下：
goods= 5
goods= 4
goods= 3
goods= 1
goods= 0
goods= 2
Runnable更容易可以实现多个线程间的资源共享，而Thread不可以?大错特错！

这个例子结果多卖一倍的原因根本不是因为Runnable和Thread的区别，我们先来看如下两行代码：

new MyThreadTest().start();
new MyThreadTest().start();
例子中，创建了两个MyThreadTest对象，每个对象都有自己独立的goods成员变量，当然会多卖1倍。如果把goods定义为静态static类型，就对了第一步了（因为是多线程同时访问一个变量会有同步问题，加上锁才是最终正确的代码）。
现在看另一个例子中，如下代码：
MyThreadTest2 mt=new MyThreadTest2();
new Thread(mt).start();
new Thread(mt).start();
只创建了一个Runnable对象，里面使用的都是同一个对象mt(使用同一个属性),那肯定只卖一倍商品（但也会有多线程同步问题，同样需要加锁），根本不是Runnable和Thread的区别造成的。再来看一个使用Thread方式的正确例子：
public class Test3  extends Thread {
private int goods= 10;
public void run(){
for(int i =0;i<10;i++){
synchronized (this){
if(this.ticket>0){
try {
Thread.sleep(100);
System.out.println(Thread.currentThread().getName()+"卖商品---->"+(this.goods--));
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}
}
public static void main(String[] arg){
Test3 t1 = new Test3();
new Thread(t1,"线程1").start();
new Thread(t1,"线程2").start();
}
}
结果如下：
线程1卖商品---->10
线程1卖商品---->9
线程1卖商品---->8
线程1卖商品---->7
线程1卖商品---->6
线程1卖商品---->5
线程1卖商品---->4
线程1卖商品---->3
线程1卖商品---->2
线程1卖商品---->1

上例中只创建了一个Thread对象（子类Test3）,效果和Runnable一样。
上面讨论下来，Thread和Runnable实现的效果没有根本的没区别，只是写法不同罢了，这才是正确的结论，网上的大神所说的Runnable更容易实现资源共享！



我们可以先看下Thread得源码,就是实现了runnable接口：

public
class Thread implements Runnable {
/* Make sure registerNatives is the first thing <clinit> does. */
private static native void registerNatives();
static {
registerNatives();
}
Thread实现了Runnable接口，提供了更多的可用方法和成员而已。



结论，Thread和Runnable的实质是继承与被继承的关系，是一个实现类和接口的关系,只是他们都实现了线程。无论使用Runnable还是Thread，都会new Thread，然后执行run方法。



            用法上区别，如果有复杂的线程操作需求，那就选择继承Thread(有其他拓展)，如果只是简单的执行一个任务，那就实现runnable。

再遇到二笔面试官问Thread和Runnable的区别，你可以bs一下了