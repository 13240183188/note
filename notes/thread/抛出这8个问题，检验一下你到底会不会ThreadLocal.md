https://mp.weixin.qq.com/s/2139B5EfyoUkbkSdz8QgYA


精彩推荐
一百期Java面试题汇总
SpringBoot内容聚合
IntelliJ IDEA内容聚合
Mybatis内容聚合
0、问题
和Synchronized的区别
存储在jvm的哪个区域
真的只是当前线程可见吗
会导致内存泄漏么
为什么用Entry数组而不是Entry对象
你学习的开源框架哪些用到了ThreadLocal
ThreadLocal里的对象一定是线程安全的吗
笔试题
一、概述
1、官方术语
ThreadLocal类是用来提供线程内部的局部变量。让这些变量在多线程环境下访问（get/set）时能保证各个线程里的变量相对独立于其他线程内的变量。
2、大白话
ThreadLocal是一个关于创建线程局部变量的类。
通常情况下，我们创建的成员变量都是线程不安全的。因为他可能被多个线程同时修改，此变量对于多个线程之间彼此并不独立，是共享变量。而使用ThreadLocal创建的变量只能被当前线程访问，其他线程无法访问和修改。也就是说：将线程公有化变成线程私有化。
二、应用场景
每个线程都需要一个独享的对象（比如工具类，典型的就是SimpleDateFormat，每次使用都new一个多浪费性能呀，直接放到成员变量里又是线程不安全，所以把他用ThreadLocal管理起来就完美了。）
比如：
/**
* Description: SimpleDateFormat就一份，不浪费资源。
*
* @author TongWei.Chen 2020-07-10 14:00:29
  */
  public class ThreadLocalTest05 {

  public static String dateToStr(int millisSeconds) {
  Date date = new Date(millisSeconds);
  SimpleDateFormat simpleDateFormat = ThreadSafeFormatter.dateFormatThreadLocal.get();
  return simpleDateFormat.format(date);
  }

  private static final ExecutorService executorService = Executors.newFixedThreadPool(100);

  public static void main(String[] args) {
  for (int i = 0; i < 3000; i++) {
  int j = i;
  executorService.execute(() -> {
  String date = dateToStr(j * 1000);
  // 从结果中可以看出是线程安全的，时间没有重复的。
  System.out.println(date);
  });
  }
  executorService.shutdown();
  }
  }

class ThreadSafeFormatter {
public static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal = new ThreadLocal() {
@Override
protected SimpleDateFormat initialValue() {
return new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
}
};

    // java8的写法，装逼神器
//    public static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal =
//            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd hh:mm:ss"));
}
细心的朋友已经发现了，这TM也是每个线程都创建一个SimpleDateFormat啊，跟直接在方法内部new没区别，错了，大错特错！1个请求进来是一个线程，他可能贯穿了N个方法，你这N个方法假设有3个都在使用dateToStr()，你直接new的话会产生三个SimpleDateFormat对象，而用ThreadLocal的话只会产生一个对象，一个线程一个。
每个线程内需要保存全局变量（比如在登录成功后将用户信息存到ThreadLocal里，然后当前线程操作的业务逻辑直接get取就完事了，有效的避免的参数来回传递的麻烦之处），一定层级上减少代码耦合度。
再细化一点就是：
比如存储 交易id等信息。每个线程私有。
比如aop里记录日志需要before记录请求id，end拿出请求id，这也可以。
比如jdbc连接池（很典型的一个ThreadLocal用法）
....等等....
三、核心知识
1、类关系
每个Thread对象中都持有一个ThreadLocalMap的成员变量。每个ThreadLocalMap内部又维护了N个Entry节点，也就是Entry数组，每个Entry代表一个完整的对象，key是ThreadLocal本身，value是ThreadLocal的泛型值。
核心源码如下
// java.lang.Thread类里持有ThreadLocalMap的引用
public class Thread implements Runnable {
ThreadLocal.ThreadLocalMap threadLocals = null;
}

// java.lang.ThreadLocal有内部静态类ThreadLocalMap
public class ThreadLocal<T> {
static class ThreadLocalMap {
private Entry[] table;

        // ThreadLocalMap内部有Entry类，Entry的key是ThreadLocal本身，value是泛型值
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    }
}
2、类关系图
ThreadLocal内存结构图。


3、主要方法
initialValue：初始化。在get方法里懒加载的。
get：得到这个线程对应的value。如果调用get之前没set过，则get内部会执行initialValue方法进行初始化。
set：为这个线程设置一个新值。
remove：删除这个线程对应的值，防止内存泄露的最佳手段。
3.1、initialValue
3.1.1、什么意思
见名知意，初始化一些value（泛型值）。懒加载的。
3.1.2、触发时机
调用get方法之前没有调用set方法，则get方法内部会触发initialValue，也就是说get的时候如果没拿到东西，则会触发initialValue。
3.1.3、补充说明
通常，每个线程最多调用一次此方法。但是如果已经调用了remove()，然后再次调用get()的话，则可以再次触发initialValue。
如果要重写的话一般建议采取匿名内部类的方式重写此方法，否则默认返回的是null。
比如：
public static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal = new ThreadLocal() {
@Override
protected SimpleDateFormat initialValue() {
return new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
}
};
// Java8的高逼格写法
public static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal =
ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd hh:mm:ss"));
3.1.4、源码
// 由子类提供实现。
// protected的含义就是交给子类干的。
protected T initialValue() {
return null;
}
3.2、get
3.2.1、什么意思
获取当前线程下的ThreadLocal中的值。
3.2.2、源码
/**
* 获取当前线程下的entry里的value值。
* 先获取当前线程下的ThreadLocalMap，
* 然后以当前ThreadLocal为key取出map中的value
  */
  public T get() {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取当前线程对应的ThreadLocalMap对象。
  ThreadLocalMap map = getMap(t);
  // 若获取到了。则获取此ThreadLocalMap下的entry对象，若entry也获取到了，那么直接获取entry对应的value返回即可。
  if (map != null) {
  // 获取此ThreadLocalMap下的entry对象
  ThreadLocalMap.Entry e = map.getEntry(this);
  // 若entry也获取到了
  if (e != null) {
  @SuppressWarnings("unchecked")
  // 直接获取entry对应的value返回。
  T result = (T)e.value;
  return result;
  }
  }
  // 若没获取到ThreadLocalMap或没获取到Entry，则设置初始值。
  // 知识点：我早就说了，初始值方法是延迟加载，只有在get才会用到，这下看到了吧，只有在这获取没获取到才会初始化，下次就肯定有值了，所以只会执行一次！！！
  return setInitialValue();
  }
  3.3、set
  3.3.1、什么意思
  其实干的事和initialValue是一样的，都是set值，只是调用时机不同。set是想用就用，api摆在这里，你想用就调一下set方法。很自由。
  3.3.2、源码
  /**
* 设置当前线程的线程局部变量的值
* 实际上ThreadLocal的值是放入了当前线程的一个ThreadLocalMap实例中，所以只能在本线程中访问。
  */
  public void set(T value) {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取当前线程对应的ThreadLocalMap实例，注意这里是将t传进去了，t是当前线程，就是说ThreadLocalMap是在线程里持有的引用。
  ThreadLocalMap map = getMap(t);
  // 若当前线程有对应的ThreadLocalMap实例，则将当前ThreadLocal对象作为key，value做为值存到ThreadLocalMap的entry里。
  if (map != null)
  map.set(this, value);
  else
  // 若当前线程没有对应的ThreadLocalMap实例，则创建ThreadLocalMap，并将此线程与之绑定
  createMap(t, value);
  }
  3.4、remove
  3.4.1、什么意思
  将当前线程下的ThreadLocal的值删除，目的是为了减少内存占用。主要目的是防止内存泄漏。内存泄漏问题下面会说。
  3.4.2、源码
  /**
* 将当前线程局部变量的值删除，目的是为了减少内存占用。主要目的是防止内存泄漏。内存泄漏问题下面会说。
  */
  public void remove() {
  // 获取当前线程的ThreadLocalMap对象，并将其移除。
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
  // 直接移除以当前ThreadLocal为key的value
  m.remove(this);
  }
  4、ThreadLocalMap
  为啥单独拿出来说下，我就是想强调一点：这个东西是归Thread类所有的。它的引用在Thread类里，这也证实了一个问题：ThreadLocalMap类内部为什么有Entry数组，而不是Entry对象？
  因为你业务代码能new好多个ThreadLocal对象，各司其职。但是在一次请求里，也就是一个线程里，ThreadLocalMap是同一个，而不是多个，不管你new几次ThreadLocal，ThreadLocalMap在一个线程里就一个，因为再说一次，ThreadLocalMap的引用是在Thread里的，所以它里面的Entry数组存放的是一个线程里你new出来的多个ThreadLocal对象。
  核心源码如下：
  // 在你调用ThreadLocal.get()方法的时候就会调用这个方法，它的返回是当前线程里的threadLocals的引用。
  // 这个引用指向的是ThreadLocal里的ThreadLocalMap对象
  ThreadLocalMap getMap(Thread t) {
  return t.threadLocals;
  }

public class Thread implements Runnable {
// ThreadLocal.ThreadLocalMap
ThreadLocal.ThreadLocalMap threadLocals = null;
}
四、完整源码
1、核心源码
// 本地线程。Thread：线程。Local：本地
public class ThreadLocal<T> {

    // 构造器
public ThreadLocal() {}

    // 初始值，用来初始化值用的，比如：ThreadLocal<Integer> count = new ThreadLocal<>();
    // 你想Integer value = count.get(); value++;这样是报错的，因为count现在还没值，取出来的是个null,所以你需要先重写此方法为value赋上初始值，本身方法是protected也代表就是为了子类重写的。
    // 此方法是一个延迟调用方法，在线程第一次调用get的时候才执行，下面具体分析源码就知道了。
protected T initialValue() {}

    // 创建ThreadLocalMap，ThreadLocal底层其实就是一个map来维护的。
void createMap(Thread t, T firstValue) {}

    // 返回该当前线程对应的线程局部变量值。
public T get() {}

    // 获取ThreadLocalMap
ThreadLocalMap getMap(Thread t) {}

    // 设置当前线程的线程局部变量的值
public void set(T value) {}

    // 将当前线程局部变量的值删除，目的是为了减少内存占用。其实当线程结束后对应该线程的局部变量将自动被垃圾回收，所以无需我们调用remove，我们调用remove无非也就是加快内存回收速度。
public void remove() {}

    // 设置初始值，调用initialValue
private T setInitialValue() {}

    // 静态内部类，一个map来维护的！！！
static class ThreadLocalMap {

        // ThreadLocalMap的静态内部类，继承了弱引用，这正是不会造成内存泄漏根本原因
        // Entry的key为ThreadLocal并且是弱引用。value是值
static class Entry extends WeakReference<ThreadLocal<?>> {}
}

}
2、set()
/**
* 设置当前线程的线程局部变量的值
* 实际上ThreadLocal的值是放入了当前线程的一个ThreadLocalMap实例中，所以只能在本线程中访问。
  */
  public void set(T value) {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取当前线程对应的ThreadLocalMap实例
  ThreadLocalMap map = getMap(t);
  // 若当前线程有对应的ThreadLocalMap实例，则将当前ThreadLocal对象作为key，value做为值存到ThreadLocalMap的entry里。
  if (map != null)
  map.set(this, value);
  else
  // 若当前线程没有对应的ThreadLocalMap实例，则创建ThreadLocalMap，并将此线程与之绑定
  createMap(t, value);
  }
  3、getMap()
  // 在你调用ThreadLocal.get()方法的时候就会调用这个方法，它的返回是当前线程里的threadLocals的引用。
  // 这个引用指向的是ThreadLocal里的ThreadLocalMap对象
  ThreadLocalMap getMap(Thread t) {
  return t.threadLocals;
  }

public class Thread implements Runnable {
// ThreadLocal.ThreadLocalMap
ThreadLocal.ThreadLocalMap threadLocals = null;
}
4、map.set()
// 不多BB，就和HashMap的set一个道理，只是赋值key,value。
// 需要注意的是这里key是ThreadLocal对象，value是值
private void set(ThreadLocal<?> key, Object value) {}
5、createMap()
/**
* 创建ThreadLocalMap对象。
* t.threadLocals在上面的getMap中详细介绍了。此处不BB。
* 实例化ThreadLocalMap并且传入两个值，一个是当前ThreadLocal对象一个是value。
  */
  void createMap(Thread t, T firstValue) {
  t.threadLocals = new ThreadLocalMap(this, firstValue);
  }

// ThreadLocalMap构造器。
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
table = new Entry[INITIAL_CAPACITY];
int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
// 重点看这里！！！！！！
// new了一个ThreadLocalMap的内部类Entry，且将key和value传入。
// key是ThreadLocal对象。
table[i] = new Entry(firstKey, firstValue);
size = 1;
setThreshold(INITIAL_CAPACITY);
}

/**
* 到这里朋友们应该真相大白了，其实ThreadLocal就是内部维护一个ThreadLocalMap，
* 而ThreadLocalMap内部又维护了一个Entry对象。Entry对象是key-value形式，
* key是ThreadLocal对象，value是传入的value
* 所以我们对ThreadLocal的操作其实都是对内部的ThreadLocalMap.Entry的操作
* 所以保证了线程之前互不干扰。
  */
  6、get()
  /**
* 获取当前线程下的entry里的value值。
* 先获取当前线程下的ThreadLocalMap，
* 然后以当前ThreadLocal为key取出map中的value
  */
  public T get() {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取当前线程对应的ThreadLocalMap对象。
  ThreadLocalMap map = getMap(t);
  // 若获取到了。则获取此ThreadLocalMap下的entry对象，若entry也获取到了，那么直接获取entry对应的value返回即可。
  if (map != null) {
  // 获取此ThreadLocalMap下的entry对象
  ThreadLocalMap.Entry e = map.getEntry(this);
  // 若entry也获取到了
  if (e != null) {
  @SuppressWarnings("unchecked")
  // 直接获取entry对应的value返回。
  T result = (T)e.value;
  return result;
  }
  }
  // 若没获取到ThreadLocalMap或没获取到Entry，则设置初始值。
  // 知识点：我早就说了，初始值方法是延迟加载，只有在get才会用到，这下看到了吧，只有在这获取没获取到才会初始化，下次就肯定有值了，所以只会执行一次！！！
  return setInitialValue();
  }
  7、setInitialValue()
  // 设置初始值
  private T setInitialValue() {
  // 调用初始值方法，由子类提供。
  T value = initialValue();
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取map
  ThreadLocalMap map = getMap(t);
  // 获取到了
  if (map != null)
  // set
  map.set(this, value);
  else
  // 没获取到。创建map并赋值
  createMap(t, value);
  // 返回初始值。
  return value;
  }
  8、initialValue()
  // 由子类提供实现。
  // protected
  protected T initialValue() {
  return null;
  }
  9、remove()
  /**
* 将当前线程局部变量的值删除，目的是为了减少内存占用。
* 其实当线程结束后对应该线程的局部变量将自动被垃圾回收，所以无需我们调用remove，我们调用remove无非也就是加快内存回收速度。
  */
  public void remove() {
  // 获取当前线程的ThreadLocalMap对象，并将其移除。
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
  m.remove(this);
  }
  10、小结
  只要捋清楚如下几个类的关系，ThreadLocal将变得so easy！
  Thread、ThreadLocal、ThreadLocalMap、Entry
  一句话总结就是：Thread维护了ThreadLocalMap，而ThreadLocalMap里维护了Entry，而Entry里存的是以ThreadLocal为key，传入的值为value的键值对。
  五、答疑（面试题）
  1、和Synchronized的区别
  问：他和线程同步机制（如：Synchronized）提供一样的功能，这个很吊啊。
  答：放屁！同步机制保证的是多线程同时操作共享变量并且能正确的输出结果。ThreadLocal不行啊，他把共享变量变成线程私有了，每个线程都有独立的一个变量。举个通俗易懂的案例：网站计数器，你给变量count++的时候带上synchronized即可解决。ThreadLocal的话做不到啊，他没发统计，他只能说能统计每个线程登录了多少次。
  2、存储在jvm的哪个区域
  问：线程私有，那么就是说ThreadLocal的实例和他的值是放到栈上咯？
  答：不是。还是在堆的。ThreadLocal对象也是对象，对象就在堆。只是JVM通过一些技巧将其可见性变成了线程可见。
  3、真的只是当前线程可见吗
  问：真的只是当前线程可见吗？
  答：貌似不是，貌似通过InheritableThreadLocal类可以实现多个线程访问ThreadLocal的值，但是我没研究过，知道这码事就行了。
  4、会导致内存泄漏么
  问：会导致内存泄漏么？
  答：分析一下：
  1、ThreadLocalMap.Entry的key会内存泄漏吗？
  2、ThreadLocalMap.Entry的value会内存泄漏吗？
  先看下key-value的核心源码
  static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    Entry(ThreadLocal<?> k, Object v) {
  super(k);
  value = v;
  }
  }
  先看继承关系，发现是继承了弱引用，而且key直接是交给了父类处理super(key)，父类是个弱引用，所以key完全不存在内存泄漏问题，因为他不是强引用，它可以被GC回收的。
  弱引用的特点：如果这个对象只被弱引用关联，没有任何强引用关联，那么这个对象就可以被GC回收掉。弱引用不会阻止GC回收。这是jvm知识。
  再看value，发现value是个强引用，但是想了下也没问题的呀，因为线程终止了，我管你强引用还是弱引用，都会被GC掉的，因为引用链断了（jvm用的可达性分析法，线程终止了，根节点就断了，下面的都会被回收）。
  这么分析一点毛病都没有，但是忘了一个主要的角色，那就是线程池，线程池的存在核心线程是不会销毁的，只要创建出来他会反复利用，生命周期不会结束掉，但是key是弱引用会被GC回收掉，value强引用不会回收，所以形成了如下场面：
  Thread->ThreadLocalMap->Entry(key为null)->value
  由于value和Thread还存在链路关系，还是可达的，所以不会被回收，这样越来越多的垃圾对象产生却无法回收，早晨内存泄漏，时间久了必定OOM。
  解决方案ThreadLocal已经为我们想好了，提供了remove()方法，这个方法是将value移出去的。所以用完后记得remove()。
  5、为什么用Entry数组而不是Entry对象
  这个其实主要想考ThreadLocalMap是在Thread里持有的引用。
  问：ThreadLocalMap内部的table为什么是数组而不是单个对象呢？
  答：因为你业务代码能new好多个ThreadLocal对象，各司其职。但是在一次请求里，也就是一个线程里，ThreadLocalMap是同一个，而不是多个，不管你new几次ThreadLocal，ThreadLocalMap在一个线程里就一个，因为ThreadLocalMap的引用是在Thread里的，所以它里面的Entry数组存放的是一个线程里你new出来的多个ThreadLocal对象。
  6、你学习的开源框架哪些用到了ThreadLocal
  Spring框架。
  DateTimeContextHolder
  RequestContextHolder
  7、ThreadLocal里的对象一定是线程安全的吗
  未必，如果在每个线程中ThreadLocal.set()进去的东西本来就是多线程共享的同一个对象，比如static对象，那么多个线程的ThreadLocal.get()获取的还是这个共享对象本身，还是有并发访问线程不安全问题。
  8、笔试题
  问：下面这段程序会输出什么？为什么？
  public class TestThreadLocalNpe {
  private static ThreadLocal<Long> threadLocal = new ThreadLocal();

  public static void set() {
  threadLocal.set(1L);
  }

  public static long get() {
  return threadLocal.get();
  }

  public static void main(String[] args) throws InterruptedException {
  new Thread(() -> {
  set();
  System.out.println(get());
  }).start();
  // 目的就是为了让子线程先运行完
  Thread.sleep(100);
  System.out.println(get());
  }
  }
  答：
  1
  Exception in thread "main" java.lang.NullPointerException
  at com.chentongwei.study.thread.TestThreadLocalNpe.get(TestThreadLocalNpe.java:16)
  at com.chentongwei.study.thread.TestThreadLocalNpe.main(TestThreadLocalNpe.java:26)
  为什么？
  为什么输出个1，然后空指针了？
  首先输出1是没任何问题的，其次主线程空指针是为什么？
  如果你这里回答
  1
  1
  那我恭喜你，你连ThreadLocal都不知道是啥，这明显两个线程，子线程和主线程。子线程设置1，主线程肯定拿不到啊，ThreadLocal和线程是嘻嘻相关的。这个不多费口舌。
  说说为什么是空指针？
  因为你get方法用的long而不是Long，那也应该返回null啊，大哥，long是基本类型，默认值是0，没有null这一说法。ThreadLocal里的泛型是Long，get却是基本类型，这需要拆箱操作的，也就是会执行null.longValue()的操作，这绝逼空指针了。
  看似一道Javase的基础题目，实则隐藏了很多知识。
  六、ThreadLocal工具类
  package com.duoku.base.util;

import com.google.common.collect.Maps;
import org.springframework.core.NamedThreadLocal;

import java.util.Map;

/**
* Description:
*
* @author TongWei.Chen 2019-09-09 18:35:30
  */
  public class ThreadLocalUtil {

  private static final ThreadLocal<Map<String, Object>> threadLocal = new NamedThreadLocal("xxx-threadlocal") {
  @Override
  protected Map<String, Object> initialValue() {
  return Maps.newHashMap();
  }
  };

  public static Map<String, Object> getThreadLocal(){
  return threadLocal.get();
  }

  public static <T> T get(String key) {
  Map map = threadLocal.get();
  // todo:copy a new one
  return (T)map.get(key);
  }

  public static <T> T get(String key,T defaultValue) {
  Map map = threadLocal.get();
  return (T)map.get(key) == null ? defaultValue : (T)map.get(key);
  }

  public static void set(String key, Object value) {
  Map map = threadLocal.get();
  map.put(key, value);
  }

  public static void set(Map<String, Object> keyValueMap) {
  Map map = threadLocal.get();
  map.putAll(keyValueMap);
  }

  public static void remove() {
  threadLocal.remove();
  }

}
