#两个小例子讲解System.out

其实本文章讲解的是System.out.println可能带来的误区

- **volatile与system.out组合产生的误区**
- **主内存与工作内存的“小桥梁“**
 


-------------------



## volatile与system.out组合产生的误区


Volatile关键字大家并不是很陌生，他有两个特性，一个是可见性，第二个就是禁止重排序（具体说明是重排序，感兴趣的话去搜下就有，我这里就不做讲解），但是大家也非常清楚，他并不保证原子性。 

下面有个例子就可以说明：
代码如下：

```
public class VolatileTest {
	public static volatile int race = 0;

	public static final Object obj = new Object();

	public static void increase() {
		race++;
	}

	private static final int THREADS_COUNT = 20;

	public static void main(String[] args) {
		Thread[] threads = new Thread[THREADS_COUNT];
		for (int i = 0; i < THREADS_COUNT; i++) {
			threads[i] = new Thread(new Runnable() {
				@Override
				public void run() {						
	        for (int i = 0; i < 10000; i++) {
								increase();
					}
				}
			});
			threads[i].start();
		}
		while (Thread.activeCount() > 1)
			Thread.yield();
		System.out.println(race);
	}
}
```
这个例子就是说我启用20个线程并且每个线程循环increase()方法10000次，如果保证原子性的情况下应该输出结果是200000次，但是结果往往不是这个数值（而且不变的）。

上面是从表现的层面上来说明了他不能保证原子性。
如果更深层次层面的去看这个问题：

```
 0: getstatic     #2
 3: iconst_1
 4: iadd
 5: putstatic     #2
```
这四个是执行race++的四个指令，就是说当我getstatic拿到值得时候是正确的，但是当我执行iadd指令的时候，其他线程可能已经执行putstatic指令，所以我再iadd的时候，其实值是变小了，从这里也看出他并非保证原子性。

```
increase();
System.out.println(race);
```
    如果你在increase()方法下加入这句话，就会发现结果不管怎么运行都是200000。
这不是很奇怪么，明明race不能保证原子性，为什么输出确实能保证是线程安全的。

问题就是出在System.out.println()这里，
点击查看他的源码会发现：

```
public void println(String x) {
        synchronized (this) {
            print(x);
            newLine();
        }
}
```
>会发现每次输出都是被同步加锁，其实读到这，很多读者还是会觉得不对劲，因为加锁的只是输出这句话，race++又没被加锁，按常理会出现值重复的情况，但是结果并非如此。
  
  Jvm虚拟机中有个锁优化原则之一就是“锁粗化”，何为锁粗化，但是如果一系列的连续操作都对同一个对象反 复加锁和解锁，甚至加锁操作是出现在循环体中的，那即使没有线程竞争，频繁地进行互斥 同步操作也会导致不必要的性能损耗。
所以加了System.ou.println()之后相当于代码变成了：

```
synchronized (obj) {
	for (int i = 0; i < 10000; i++) {
		increase();							       
	}
}
```
所以现象可能给初学者带到一种误区，会认为他存在原子性，我用这个例子就是想说明的是volatile的可见性但是不保证原子性。
### 主内存与工作内存的“小桥梁“
      第二个例子就是关于主内存和工作内存，先看下运行代码：
  

```
public class TestBooleanStop implements Runnable{

	public  boolean flag=true;

	@Override
	public void run() {
			while(flag){
				//...
		}
	}
	public static void main(String[] args) throws Exception {
		TestBooleanStop testBooleanStop = new TestBooleanStop();
		Thread t=new Thread(testBooleanStop);
		t.start();
		Thread.sleep(3000);
		testBooleanStop.flag=false;
		Thread.sleep(3000);
		System.out.println(testBooleanStop.flag);
		
	}
}
```
这代码很简单，启动一个线程进入死循环，然后修改flag观察线程是否停止并输出flag，但是程序运行结果是flag的确为false，但是程序并没有停止，还是一直在运行。

如果在循环体中放入：i++

```
int i=0;
while(flag){
 i++;
        }
```
这样子程序依然是不会停止。



但是你在循环体中加入System.out.println(...)就会发现程序就会停止。

```
while(flag){
   System.out.println(...);
        }
```


其实最关键的还是在System.out.println，他每次输出都会清除工作内存去同步主内存，由于我们修改的flag存在在主内存，所以每次输出去同步的话必然会把flag更新到我们的工作内存当中并且停止了程序运行。

而synchronized是做了什么的操作呢
获得同步锁；
1、清空工作内存；
2、从主内存拷贝对象副本到工作内存；
3、执行代码(计算或者输出等)；
4、刷新主内存数据；
5、释放同步锁。

所以system.out.println是主内存和工作内存之间的小桥梁。

    这是我个人对这些现象的看法和分析，如果大家觉得有更好的角度去分析多多指教，我也会去更正改正。