# 《Java并发编程实践-第一部分》-读书笔记

大家好，我是烤鸭：
	《Java并发编程实战-第一部分》-读书笔记。

## 第一章：介绍

### 1.1 并发历史：

多个程序在各自的进程中执行，由系统分配资源，如：内存、文件句柄、安全证书。进程间通信方式：Socket、信号处理（Signal Handlers）、共享内存（Shared Memory）、信号量（Semaphores）和文件。

促进因素：

- 资源利用：等待的时候，其他程序运行会提高效率。
- 公平：多个用户或程序可能对系统资源有平等的优先级别。
- 方便：多个程序各自执行单独的任务比一个程序执行多个任务方便调度和协调。

早期分时共享系统，每一个进程都是一个虚拟的冯诺依曼机：拥有内存空间、存储指令和数据，根据机器语言来顺序执行指令，通过操作系统的 I/O 实现与外部世界交互。对于每一条指令的执行，都有对 "下一条指令" 的明确定义，并根据程序中的指令集来进行流程控制。

线程允许程序控制流(control flow)的多重分支同时存在于一个进程。共享进程范围内的资源，比如内存和文件句柄，但是每个线程有自己的程序计数器(program counter)、栈(stack)和本地变量。

### 1.2 线程优点：

适当多线程降低开发和维护开销，提高性能。

- 多核处理器：程序调度基本单元是线程，一个单线程应用程序一次只能运行在一个处理器上。
- 模型简化：一个复杂、异步的流程可以被分解为一系列更简单的同步流程，每一个在相互独立的线程中运行，只有在特定同步点才彼此交互。比如servlets或者RMI。
- 异步事件的简单处理：历史上，操作系统把一个进程能够创建的线程限制在比较少的数量(几百个)。因此，操作系统为多元化的I/O开发了一些高效机制，比如unix的select和poll系统调用，java类库对非阻塞I/O提供了一组包(java.nio)。
- 用户界面的更加响应性：AWT 和 Swing 框架

### 1.3 线程风险：

- 安全危险：多线程操作共享变量，有可能指令重排序，会导致发生混乱。
- 活跃度风险：单线程没问题，多线程可能出现 死锁、饥饿、活锁 （https://blog.csdn.net/qq_22054285/article/details/87911464）。
- 性能风险：上下文切换可能引起巨大的系统开销。

### 1.4 线程无处不在：

比如 AWT和Swing、Timer、Servlets

## 第二章 线程安全

编写线程安全的代码，本质上是管理对状态的访问，而且通常都是共享、可变的状态。

状态：对象的状态就是数据存储在状态变量，比如实例域或静态域。共享指多线程访问、可变指生命周期内可以改变。线程安全的性质，取决于程序中如何使用对象，而不是对象完成了什么。(没读懂)

无论何时，只要有多于一个线程访问给定的状态变量，而且其中某个线程会写入该变量，此时必须使用同步来协调线程对该变量的访问。

Java中首要的同步机制是 synchronized ，提供了独占锁。volatile 用来声明变量可见性，但不能保证原子性。

消除多线程访问同一个变量的隐患：不跨线程共享变量、使状态变量不可变、在任何状态访问变量使用同步。

**设计时考虑线程安全，比后期修复容易。设计线程安全的类，封装、不可变性以及明确的不变约束会提供帮助。**

### 2.1 线程安全性：

类与它的规约保持一致。良好的规约定义了用于强制对象状态的不变约束以及描述操作影响的后验条件。
定义：**当多个线程访问一个类时，如果不用考虑这些线程在运行时环境下的调度和交替执行，并且不需要额外的同步及在调用方代码不必做其他的协调，这个类行为仍然是正确的，那么这个类是线程安全的。
对于线程安全类的实例进行顺序或并发的一系列操作，都不会导致实例处于无效状态。**

- 示例：一个无状态(stateless)的servlet

  ```
  @ThreadSafe
  public class StatelessFactorizer implements Servlet{
  	public void Service(ServletRequest req,ServletResponse resp){
  		BigInteger i = extractFormRequest(req);
  		BigInteger[] factors = factor(i);
  		//...
  	}
  }
  ```

  StatelessFactorizer 和 大多数 Servlet 一样，是无状态的：不包含域也没有引用其他类的域。一次特定计算的瞬时状态，会唯一地存在本地变量中，这些变量存储在线程的栈中，只有执行线程才能访问。

  **无状态对象永远是线程安全的。**

### 2.2 原子性
比如下面的代码，单线程时运行良好。但是多线程会有问题，++count不是原子操作。
自增操作是"读-改-写"操作的实例。

```
@NotThreadSafe
public class StatelessFactorizer implements Servlet{
	private long count = 0;
	public void Service(ServletRequest req,ServletResponse resp){
		BigInteger i = extractFormRequest(req);
		BigInteger[] factors = factor(i);
		++ count;
		//...
	}
}
```

- 竞争条件：线程交替执行，会产生竞争，正确的答案依赖"幸运"的时序。最常见的竞争是"检查再运行"(check-then-act)。比如单例模式，如果没有double check synchronized，多线程场景就可能出现不止一个对象。

- 示例：惰性初始化中的竞争条件，延迟对象的初始化，直到程序真正用到它，同时确保只初始化一次。
  (单例模式推荐使用的是double check  synchronized)

- 复合操作：**操作A、B，当其他线程执行B时，要么B全部执行完成，要么一点没有执行。A、B互为原子操作。**比如前面的自增操作，如果是原子操作的话就没有问题了。可以考虑Java内置的原子性机制-锁。

  ```
  @ThreadSafe
  public class StatelessFactorizer implements Servlet{
  	private final AtomicLong count = new AtomicLong(0);
  	
  	public void Service(ServletRequest req,ServletResponse resp){
  		BigInteger i = extractFormRequest(req);
  		BigInteger[] factors = factor(i);
  		count.incrementAndGet();
  		//...
  	}
  }
  ```

  juc 包下包含了原子变量类，实现数字和对象引用的原子状态转换。把long类型的计数器替换为 AtomicLong 类型的。

### 2.3 锁
比如下面这段代码，虽然变量本身是线程安全的，但是多个set操作不是原子性的，所以结果是有问题的。
**为了保护状态的一致性，要在单一的原子操作中更新相互关联的状态变量。**

```
@NotThreadSafe
public class StatelessFactorizer implements Servlet{
	private final AtomicReference<BigInteger> lastNumber = new AtomicReference<BigInteger>();
	private final AtomicReference<BigInteger[]> lastFactors = new AtomicReference<BigInteger[]>();
	public void Service(ServletRequest req,ServletResponse resp){
		BigInteger i = extractFormRequest(req);
		if(i.equals(lastNumber.get())){
			//...
		}else{
			BigInteger[] factors = factor(i);
            lastNumber.set(i);
            lastFactors.set(factors);
		}
	}
}
```

- 内部锁：synchronized 块。
  每个Java对象都可以隐式扮演一个用于同步的锁角色，这些内置锁被称为内部锁(intrinsic locks)或监视器锁(monitor locks)。执行线程进入 synchronized 块之前会自动获得锁，无论是正常退出，还是抛出异常，都会在失去 synchronized 块 的控制时释放锁。
  内部锁在Java中扮演了互斥锁，只有至多一个线程可以拥有锁。

  ```
  @ThreadSafe
  public class SynchronizedFactorizer implements Servlet{
  	private BigInteger lastNumber;
  	private BigInteger[] lastFactors;
  	public synchronized void Service(ServletRequest req,ServletResponse resp){
  		BigInteger i = extractFormRequest(req);
  		if(i.equals(lastNumber)){
  			//...
  		}else{
  			BigInteger[] factors = factor(i);
              lastNumber = i;
              lastFactors = factors;
  		}
  	}
  }
  ```

- 重进入 (Reentrancy)：当一个线程请求其他线程已经占有的锁时，请求线程将被阻塞。然而内部锁时可重进入的，重进入意味着基于“每线程”，而不是“每调用”。（这句话没看明白，重入说的就是持有锁的线程再次持有锁，也就是多线程场景下的单线程场景，每持有锁一次就重入一次，计数递增）**重进入的实现通过为每个锁关联一个请求计数和一个占有它的线程。没锁的时候，计数为0，相同的线程每请求一次锁，计数就会递增。**

### 2.4 用锁来保护状态

锁使得线程能够串行地访问它所保护的代码路径，可以用锁创建相关的协议，保证对共享状态的独占访问。比如 ++count 的递增或 惰性初始化，操作共享状态的复合操作必须是原子的。

**对于每个可被多个线程访问的可变状态变量，如果所有访问它的线程在执行时都占有同一个锁，这种情况下，我们称这个变量由这个锁保护。**

**对象的内部锁和它的状态之前没有内在关系。每个共享的可变变量都需要由唯一一个确定的锁保护，而维护者应该清楚这个锁。**
不是所有数据都需要锁保护—只有那些被多个线程访问的可变数据。

**对于每一个涉及多个变量的不变约束，需要同一个锁保护其所有的变量。**

### 2.5 活跃度与性能

上面的代码 SynchronizedFactorizer 类，在 Service方法加 synchronized，每次只能有一个线程执行它，违背了 Servlet 框架的初衷。多个请求排队等待并依次被处理。我们把这种Web应用的运行方式描述为**弱并发(poor concurrency)的一种表现：限制并发调用数量的，并非可用的处理器资源，而恰恰是应用程序自身的结构。**

可以通过缩小 synchronized 块的范围来提升并发性。

**通过简单性与性能之间是相互牵制的。实现一个同步策略时，不要过早为了性能而牺牲简单性（这是对安全性潜在的威胁）。**

**有些耗时的计算或操作，比如网络或控制台 I/O，难以快速完成。这些操作期间不要占有锁。**

## 第三章 共享对象

synchronized 不仅保证原子性，还保证 内存可见性。

### 3.1 可见性：读、写操作发生在不同线程，“重排序”可能会影响读线程看到的结果。
**在没有同步的情况下，编译器、处理器，运行时安排操作的执行顺序可能完全出人意料。在没有进行适当同步的多线程程序中，尝试推断那些“必然”发生在内存中的动作时，你总是会判断错误。** 

- 过期数据：类似数据库隔离级别的读未提交，也就是读到了更新前的数据。
- 非原子的64位操作：对于非 volatile的long和double变量，JVM将64位的读或写划分为两个32位的操作。如果读、写在不同的线程，这种情况读取一个非 volatile类型的long就可能出现一个值的高32位和另一个值的低32位。
- 锁和可见性：锁不仅仅是关于同步与互斥的，也是关于内存可见的。为了保证所有线程都能看到共享的、可变变量的最新值，读取和写入线程必须使用公共的锁进行同步。
- Volatile 变量：同步的弱形式：确保对一个变量的更新以可预见的方式告知其他线程。当一个域声明volatile 后，编译器会监视这个变量，并且对它的操作不会与其他的内存操作一起被重排序。
  **只有当 volatile变量能够简化实现和同步粗略的验证时，才使用它们。当验证正确性必须推断可见性问题时，应该避免使用volatile变量。正确使用volatile变量的方式：用于确保它们引用的对象状态的可见性，或者用于标识重要的生命周期事件(比如初始化或关闭)的发生。**
  读取 volatile 变量比 读取非 volatile 变量的开销略高。
  **加锁可以保证可见性和原子性；volatile 变量只能保证可见性。**

### 3.2 发布和逸出：

- 安全构建的实践：不要让this引用在构造期间逸出。在构造函数中注册监听器或启动线程，使用私有构造函数和一个公共的工厂方法。

### 3.3 线程封闭：
类似JDBC连接池的Connection对象，线程总是从池中获得一个Connection对象，并且用它处理单一请求，最后归还。这种连接管理模式隐式地将Connection对象限制在处于请求处理期间的线程中。

- Ad-hoc 线程限制（未经设计的线程封闭行为）： 比如使用volatile关键字，外加读写在同一线程内。

- 栈限制：栈限制中，只能通过本地变量才能触及对象。

- ThreadLocal：线程与持有数值的对象关联，提供get和set方法。


### 3.4不可变性：

不可变对象永远是线程安全的。

- final：将所有的域声明为final型，除非它们是可变的。

- 使用volatile 发布不可变对象：使用可变的容器对象，必须加锁以确保原子性。使用不可变对象，不必担心其他线程修改对象状态，如果更新变量，会创建新的容器对象，不过在此之前任何线程都还和原先的容器打交道，仍然看到它处于一致的状态。(对应下面的代码 OneValueCache 和       VolatileCachedFactorizer)

  ```
  @Immutable
  class OneValueCache{
  	private final BigInteger lastNumber;
  	private final BigInteger[] lastFactors;
  	
  	public OneValueCache(BigInteger i,BigInteger[] factors){
  		lastNumber = i;
  		lastFactors = Arrays.copyOf(factors,factors.length);
  	}
  	public BigInteger[] getFactors(BigInteger i) {
  		if(lastNumber == null || !lastNumber.equals(i)){
  			return null;
  		}else{
  			return Arrays.copyOf(lastFactors, lastFactors.length);
  		}
  	}
  }
  ```

### 3.5 安全发布：

```
@ThreadSafe
public class VolatileCachedFactorizer implements Servlet{
	private volatile OneValueCache cache = new OneValueCache(null, null);
	
	public void service(ServletRequest req, ServletResponse resp){
		BigInteger i = extractFromRequest(req);
		BigInteger[] factors = cache.getFactors(i);
		if (factors == null){
			facotrs = factors(i);
			cache = new OneValueCache(i, factors);
		}
		//
	}
}
```

- 不正确发布：当好对象变坏时，代码如下：

  ```
  public class Holder {
  	private int n;
  	public Holder(int n){
  		this.n = n;
  	}
  	public void assertSanity(){
  		if(n != n){
  			throw new AssertionError("This statement is false.");
  		}
  	}
  }
  ```

  发布线程以外的任何线程都可以看到Holder域的过期值，因而看到的是一个null引用或者旧值。（反之线程同理），这种写法本身是不安全的。

- 不可变对象与初始化安全性：即使发布对象引用时没有使用同步，不可变对象仍然可以被安全地访问。

  不可变对象可以在没有额外同步的情况下，安全地用于任意线程；甚至发布它们也不需要同步。

- 安全发布的模式：
  发布对象的安全可见性。

  - 通过静态初始化器初始化对象的引用；
  - 将它的引用存储到volatile域或AtomicReference；
  - 将它的引用存储到正确创建的对象的final域中；
  - 或者将它的引用存储到由锁正确保护的域中。

  线程安全容器。

  ​	HashTable、SynchronizedMap、ConcurrentMap、Vector.CopyOnWriteArrayList、CopyOnWriteArraySet、syncronized-List、BlockingQueue或者ConcurrentListQueue

- 高效不可变对象（Effectively immutable objects）：任何线程都可以在没有额外的同步下安全地使用一个安全发布的高效不可变对象。比如正在维护一个Map存储每位用户的最近登录时间：

  ```
  public Map<String,Date> lastLogin = Collections.synchronizedMap(new HashMap<String, Date>)());
  ```

  访问Date值时不需要额外的同步。

- 可变对象：
  不可变对象可以通过任意机制发布；
  高效不可变对象必须要安全发布；
  可变对象必须要安全发布，同时必须要线程安全或者被锁保护。

- 安全地共享对象：

  共享策略：

  - 线程限制：一个线程限制的对象，通过限制在线程中，而被线程独占，且只能被占有它的线程修改。

  - 共享只读（shared read-only）：一个共享的只读对象，在没有额外同步的情况下，可以被多个线程并发访问，但是任何线程都不能修改它。共享只读对象包括可变对河与高效不可变对象。
  - 共享线程安全(shared thread-safe) ：一个线程安全的对象在内部进行同步，所以其他线程无须额外同步，就可以通过公共接口随意访问它。
  - 被守护的(Guarded) ：一个被守护的对象只能通过特定的锁来访问。被守护的对象包括那些被线程安全对象封装的对象，和已知被待定的锁保护起来的已发布的对象。

## 第四章 组合对象

### 4.1 设计线程安全的类：
确定对象状态由哪些变量构成；
确定限制状态变量的不变约束；
制定一个管理并发访问对象状态的策略。

```
@ThreadSafe
public final class Counter {
	@GuradedBy("this") private long value = 0;
	
	public synchronized long getValue(){
		return value;
	}
	public synchronized long increment(){
		//...
		return ++value;
	}
}
```

同步策略定义对象如何协调对其状态访问，并且不会违反它的不变约束或后验条件。

- 收集同步需求：对象与变量拥有一个状态空间，即它们可能处于的状态范围。状态空间越小，越容易判断它们。尽量使用final类型的域，可以简化对对象可能状态进行分析。

  比如 Long的区间是 Long.MIN_VALUE 到 Long.MAX_VALUE，后验条件会指出某种状态转换是非法的。比如当前状态17，下一个合法状态是18。
  **不理解对象的不变约束和后验条件，就不能保证线程安全性。要约束状态变量的有效值或者状态转换，就需要原子性与封装性**。

- 状态依赖的操作
  和后验条件对应的是先验条件，比如移除队列中的元素，队列必须是“非空”状态。

- 状态所有权
  创建的对象归属谁所有，所有权意味着控制权，一旦将引用发布到一个可变对象上，就不再拥有独占的控制权，充其量只可能有"共享控制权"。容器类通常表现出一种"所有权分离"的形式。以 servlet中的 ServletContext 为例。 ServletContext 为 Servlet 提供了类似于 Map的对象容器服务。ServletContext可以调用 setAttribute 和 getAttribute，由于被多线程访问，ServletContext 必须是线程安全的，而 setAttribute 和 getAttribute 不必是同步的。

### 4.2 实例限制
**将数据封装在对象内部，把对数据的访问限制在对象的方法上**，更易确保线程在访问数据时总能获得正确的锁。比如 私有的类变量、本地变量或者线程内部的变量。
比如 PersonSet中的 mySet 只能通过 addPerson 和 containsPerson 访问，而这两个方法都加了锁。

```
@ThreadSafe
puclic class PersonSet{
	@GuardedBy("this")
	private final Set<Person> mySet = new HashSet<Person>();
	
	public synchronized void addPerson(Person p){
		mySet.add(p);
	}
	
	public synchronized boolean containsPerson(Person p){
		return mySet.contains(p);
	}
}
```

使用线程安全的类，分析安全性时更容易。

- 监视器模式
  Java monitor pattern，遵循Java监视器模式的对象封装了所有的可变状态，并由对象自己的内部锁保护。私有锁对象可以封装锁，客户代码无法得到它。共有锁允许客户代码涉足它的同步策略，不正确地使用可能引起活跃度问题。要验证是否正确使用，需要检查整个程序，而不是单个类。比如内部方法加 synchronized 。

### 4.3 委托线程安全

无状态的类中加入一个 AtomicLong类型的属性，组合对象安全，因为线程安全性委托给了 AtomicLong。比如使用 ConcurrentHashMap。

- 非状态依赖变量

  使用 CopyOnWriteArrayList 存储每个监听器清单。

- 委托无法胜任

  ```
  public class NumberRange{
  	// 不变约束: lower <= upper
  	private final AtomicInteger lower = new AtomicInteger(0);
  	private final AtomicInteger upper = new AtomicInteger(0);
  	
  	public void setLower(int i){
  		// 警告 —— 不安全的 "检查再运行"
  		if (i > uppper.get()){
  			throw new IllegalArgumentException("can't set lower to ...");
  		}
  		lower.set(i);
  	}
  	
  	public void setUpper(int i){
  		// 警告 —— 不安全的 "检查再运行"
  		if (i < lower.get()){
  			throw new IllegalArgumentException("can't set upper to ...");
  		}
  		upper.set(i);
  	}
  	
  	public boolean isInRange(int i){
  		return (i >= lower.get() && i <= upper.get());
  	}
  }
  ```

  NumberRange 不是线程安全的，"检查再运行"没有保证原子性。

  底层的AtomicInteger是线程安全的，但是组合类不是，因为状态变量lower和upper不是彼此独立的。

  **如果一个类由多个彼此独立的线程安全的状态变量组成，并且类的操作不包含任何无效状态转换时，可以将线程安全委托给这些变量。**

- 发布底层的状态变量

  **如果一个状态变量是线程安全的，没有任何的不变约束限制它的值，并且没有任何状态转换限制它的操作，可以被安全发布。** 

  比如发布一个 public 的AtomicLong变量，没有上述多余的判断，就是安全的。

### 4.4 向已有的线程安全类添加功能

"缺少就加入"，比如list的!contains，再add，由于操作非原子性，可能同一个元素会执行多次add。比如使用Vector可以解决，或者改写add方法，增加synchronized。

- 4.4.1 客户端加锁

  ```
  public classs ListHelper<E>{
  	public List<E> list = Collections.synchronizedList(new ArrayList<E>());
  	// ...
  	public synchronized boolean putIfAbsent(E x) {
  		boolean absent = !list.contains(x);
  		if (absent){
  			list.add(x);
  		}
  		return absent;
  	}
  }
  ```
  
  上面这个例子对list并不是线程安全的，虽然在方法上加了synchronized。因为锁的对象不对(**客户端加锁和外部加锁所用的不是一个锁**)，操作的是list(多线程下对list的操作不是原子性的)。修改如下：
  
  ```
  public classs ListHelper<E>{
  	public List<E> list = Collections.synchronizedList(new ArrayList<E>());
  	// ...
  	public boolean putIfAbsent(E x) {
  		synchronized (list) {
  			boolean absent = !list.contains(x);
  			if (absent){
  				list.add(x);
  			}
  			return absent;
  		}
  	}
  }
  ```

- 4.4.2 组合

  客户端没法直接使用list，只能通过 ImprovedList 操作list的方法。客户端并无关心底层的list是否安全，由ImprovedList的锁来实现保证就行。

  ```
  public classs ImprovedList<T> implements List<T>{
  	public final List<T> list;
  	
  	public ImprovedList(List<T> list){
  		this.list = list;
  	}
  	
  	public synchronized boolean putIfAbsent(E x) {
  			boolean absent = !list.contains(x);
  			if (absent){
  				list.add(x);
  			}
  			return absent;
  	}
  }
  ```

- 4.5 同步策略的文档化

  **为类的用户编写类线程安全性担保的问答；为类的维护者编写类的同步策略文档。**

  - 4.5.1 含糊不清的文档

    比如 Servlet 规范没有建议任何用来协调对这些共享属性并发访问的机制。所以容器代替Web Application所存储的这些对象应该是线程安全的或者是搞笑不可变的。(**容器不可能知道你的锁协议，需要自己保证这些对象是线程安全的。**)

    再比如 DataSource的JDBC Connection，如果一个获得了JDBC Connection 的及活动跨越了多个线程，那么必须确保利用同步正确地保护到 Connection的访问。

    


## 第五章 构建块

在实践中，委托是创建线程安全类最有效的策略之一：只需要用已有的线程安全类来管理所有状态即可。

### 5.1 同步容器

同步包装类，Collections.synchronizedXxx 工厂方法创建。

#### 5.1.1 同步容器之中出现的问题

同步容器都是线程安全的。

```
public static Object getLast(Vector list) {
	int lastIndex = list.size() - 1;
	return list.get(lastIndex);
}
public static void deleteLast(Vector list) {
	int lastIndex = list.size() - 1;
	list.remove(lastIndex);
}
```

不同线程调用 size和get/remove时可能出现 ArrayIndexOutOfBoundsException，对list加锁可以避免这个问题，但会增加性能开销。

```
public static Object getLast(Vector list) {
	sychroized(list) {
		int lastIndex = list.size() - 1;
		return list.get(lastIndex);
	}
}
public static void deleteLast(Vector list) {
	sychroized(list) {
		int lastIndex = list.size() - 1;
		list.remove(lastIndex);	
	}

}
```

#### 5.1.2 迭代器和ConcurrentModificationException

无论单线程还是多线程，操作容器，都可能出现ConcurrentModificationException，这是迭代器 Iterator 对修改检查抛出的异常。

对容器加锁可以避免这个问题，但会影响性能，替代方法是复制容器，线程隔离操作安全，会有明显的性能开销。

#### 5.1.3 隐藏迭代器

容器本身作为一个元素，或者作为另一容器的key时，containsAll、removeAll、retainAll方法以及把容器作为参数的构造函数，都会对容器进行迭代。这些对迭代的间接调用，都可能引起 ConcurrentModificationException。

### 5.2 并发容器

JUC的并发容器和队列。

#### 5.2.1 ConcurrentHashMap

#### 5.2.2 Map附加的原子操作

#### 5.2.2 CopyOnWriteArrayList

写入时复制，为了保证数组内容的可见性（不会考虑后续的修改）

查询频率远高于修改频率时，适合使用 CopyOnWriteArrayList

### 5.3 阻塞队列和生产者-消费者模式

生产者-消费者模式可以把生产者和消费者的代码解耦合，但还是通过共享工作队列耦合在一起。

设计初期就使用阻塞队列建立对资源的管理。

LinkedBlockingQueue和ArrayBlockingQueue是FIFO队列，PriorityBlockingQueue是一个按优先级排序的队列(可以使用Comparator进行排序)。

#### 5.3.3 双端队列和窃取工作

双端队列和窃取工作(work stealing)模式。一个生产消费者设计中，所有的消费者共享一个工作队列；在窃取工作的设计中，每一个消费者都有一个自己的双端队列。如果一个消费完，可以偷取其他消费者的双端队列中的末尾任务。

### 5.4 阻断和可中断的方法

关于 InterruptedException的两种处理方式：

传递 InterruptedException，抛出给调用者。

恢复中断。捕获 InterruptedException ，并且在当前线程中调用通过 interrupt 中断中恢复。

恢复中断状态，避免掩盖中断：

```
public class TaskRunnable implements Runnable{
	BlockingQueue<Task> queue;
	//...
	public void run(){
		try{
			processTask(queue.take());
		} catch (InterruptedException e){
			//恢复中断状态
			Thread.currentThread().interrupt();
		}
	}
}
```

### 5.5 Synchronizer

Synchronizer 是一个对象，包含 信号量(semaphore)、关卡(barrier)以及闭锁(latch)。

#### 5.5.1 闭锁

闭锁(latch)是一种Synchronizer，可以延迟线程的进度直到线程到达终止状态。闭锁可以确保特定活动直到其他活动完成后才发生。

- 资源初始化，使用二元闭锁可以保证 先后加载。
- 服务依赖，使用二元闭锁可以保证 服务的先后启动。

CountDownLatch 是一个灵活的闭锁实现。countDown 方法对计数器做减操作，表示一个事件已经发生了，而 await 方法等待计数器打到零，此时所有需要等待的事件都已发生。如果计数器入口值为非零，await会一直阻塞到计数器为零，或者等待线程中断以及超时。(**await一定要设置超时时间**)

 #### 5.5.2 FutureTask

Future.get() 可以立刻得到返回结果。通常可以把多个FutureTask放到一个list，循环 get()

#### 5.5.3 信号量

计数信号量(Counting semaphore)用来控制能够同时访问特定资源的活动的数量，或者同时执行某一给定操作的数量。

一个Semaphore管理一个有效的许可集(permit)，活动获取许可，使用之后释放。如果没有可用的许可，acquire会阻塞。

release方法向信号量返回一个许可。信号量的退化形式：二元信号量(互斥)。

比如可以用信号量维护连接池，池为空的时候阻塞，反之解除。

#### 5.5.4 关卡







