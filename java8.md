# Java8相关记录

### 相关文章原文在[Benjamin's Blog](http://winterbe.com/blog/) 这个博客里，感谢Benjamin一系列关于Java8的文章。

### 多线程相关

    Callable<Integer> task = ()->{

    TimeUnit.SECONDS.sleep(1);
    	return 123;
	};
	ExecutorService executorService = Executors.newFixedThreadPool(1);
	Future<Integer> future = executorService.submit(task);

	System.out.println("future done? " + future.isDone());
	Integer integer = future.get();

	System.out.println("future done? " + future.isDone());
	System.out.println("result = "+integer);
future.get()方法会锁住当前线程并且等待返回result后再执行。
executorService用完应该关闭shutdown(),便于资源的回收利用。


下面这种写法相当于用了synchronized关键字。

	ReentrantLock lock = new ReentrantLock();
	int count = 0;

	void increment() {
	    lock.lock();
	    try {
	        count++;
	    } finally {
	        lock.unlock();
	    }
	}

普通的读写锁，如果写锁没有释放，那么是读取不到map中的值的。原因是写操作是原子性的，但读操作并不会产生脏数据，等写操作完成后，读操作才能读到最后修改的正确数据。


	ExecutorService executor = Executors.newFixedThreadPool(2);
	Map<String, String> map = new HashMap<>();
	ReadWriteLock lock = new ReentrantReadWriteLock();

	executor.submit(() -> {
	    lock.writeLock().lock();
	    try {
	        sleep(1);
	        map.put("foo", "bar");
	    } finally {
	        lock.writeLock().unlock();
	    }

	});
	Runnable readTask = () -> {
	    lock.readLock().lock();
	    try {
	        System.out.println(map.get("foo"));
	        sleep(1);
	    } finally {
	        lock.readLock().unlock();
	    }
	};

	executor.submit(readTask);
	executor.submit(readTask);

	stop(executor);

StampedLock，功能上和读写锁一样，但是在上锁时候多了一个Long的返回值


	ExecutorService executor = Executors.newFixedThreadPool(2);
	Map<String, String> map = new HashMap<>();
	StampedLock lock = new StampedLock();

	executor.submit(() -> {
	    long stamp = lock.writeLock();
	    try {
	        sleep(1);
	        map.put("foo", "bar");
	    } finally {
	        lock.unlockWrite(stamp);
	    }
	});

	Runnable readTask = () -> {
	    long stamp = lock.readLock();
	    try {
	        System.out.println(map.get("foo"));
	        sleep(1);
	    } finally {
	        lock.unlockRead(stamp);
	    }
	};

	executor.submit(readTask);
	executor.submit(readTask);

	stop(executor);

### 常用的小功能

concurrent map的一些计算操作

	map.compute("foo", (key, value) -> value + value);
	System.out.println(map.get("foo"));   // barbar
	map.merge("foo", "boo", (oldVal, newVal) -> newVal + " was " + oldVal);
	System.out.println(map.get("foo"));   // boo was foo


### Java8对String的一些新方法：

1.相当于split的反向用法

	String.join(":", "foobar", "foo", "bar");
	// => foobar:foo:bar

2.字符串去重

	"foobar:foo:bar"
	    .chars()
	    .distinct()
	    .mapToObj(c -> String.valueOf((char)c))
	    .sorted()
	    .collect(Collectors.joining());
	// => :abfor

3.将”foobar:foo:bar"字符串用”:"分割开，然后查找包含关键字”bar"的字符串，排序并重新用”:”分割，”foo"由于没有关键字”bar”所以被去掉了


	Pattern.compile(":")
	    .splitAsStream("foobar:foo:bar")
	    .filter(s -> s.contains("bar"))
	    .sorted()
	    .collect(Collectors.joining(":"));
	// => bar:foobar

4.一种筛选的方法

	Pattern pattern = Pattern.compile(".*@gmail\\.com");
	Stream.of("bob@gmail.com", "alice@hotmail.com")
	    .filter(pattern.asPredicate())
	    .count();
	// => 1

---
后续陆续更新...