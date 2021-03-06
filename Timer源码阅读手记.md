
	定时器Timer类是一个最基础的定时调度任务的类。
	这个类的参数有以下几个：
         //定时器任务队列，上面的定时器线程就是从这个队列中取任务来执行。有新任务需要加入也是放到这个队列中
         private final TaskQueue queue = new TaskQueue();
				 //定时器线程，负责调度定时器任务
         private final TimerThread thread = new TimerThread(queue);
	---------------------------------------------------------------------------------------------------------------        
	默认的构造函数有以下几个：
	（1）
	``` java
   	 public Timer() {
       	 this("Timer-" + serialNumber());
    	}
	```
	这个构造函数调用了另一个构造函数，从而给线程设置一个默认的名字。这个名字就是Timer-加上一个serialNumber();
	我们来看看这个serialNumber()是什么鬼？
	``` java
	private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
	    private static int serialNumber() {
	return nextSerialNumber.getAndIncrement();
	}
	```
	原来这个serialNumber就是一个自增的数据，这个调用了AtomicInteger类来实现技术的。
	这个设置为静态，说明我们在用户代码中假如创建多个定时器，也能保证每个定时器能得到一个独一无二的、递增的序列号。
	（2）这个构造函数可以用户自定义线程名称
	   ```java
	      public Timer(String name) {
		thread.setName(name);
		thread.start();
	    }
	    ```
 	（3）这个构造函数使用默认的名字和设置守护线程
	```java
	     public Timer(boolean isDaemon) {
		this("Timer-" + serialNumber(), isDaemon);
	    }
    ```
	 （4）这个构造函数可以设置名字和设置守护线程
	 ```java
		public Timer(String name, boolean isDaemon) {
		thread.setName(name);
		thread.setDaemon(isDaemon);
		thread.start();
	    }
          ```
	 *小结：创建了定时器对象，在构造函数中会帮我们启动线程的。接下来就是我们调用schedule函数来调度任务。
	-------------------------------------------------------------------------------------------------------------------
 	 接着，我们来看一下schedule这个方法，有哪些方法和参数。
  	（1）这个方法就是提交一个任务，并告诉线程多久以后执行。只执行一次任务
	```java
	      public void schedule(TimerTask task, long delay) {
		if (delay < 0)
		    throw new IllegalArgumentException("Negative delay.");
		sched(task, System.currentTimeMillis()+delay, 0);
	    }
	```
 	（2）这个方法提交一个任务，并指定什么日期执行，只执行一次任务
	```java
	       public void schedule(TimerTask task, Date time) {
		sched(task, time.getTime(), 0);
	    }
         ```
	（3）这个方法提交一个任务，并指定延长delay时间执行，周期是period
	```java
	    public void schedule(TimerTask task, long delay, long period) {
		if (delay < 0)
		    throw new IllegalArgumentException("Negative delay.");
		if (period <= 0)
		    throw new IllegalArgumentException("Non-positive period.");
		sched(task, System.currentTimeMillis()+delay, -period);
	    }
	```
	（4）这个方法和上面的类似，只是传递的是日期。
	```java
	     public void schedule(TimerTask task, Date firstTime, long period) {
		if (period <= 0)
		    throw new IllegalArgumentException("Non-positive period.");
		sched(task, firstTime.getTime(), -period);
	    }
        ```
	同样的，Timer类还提供了两个scheduleAtFixedRate方法。
	区别在于，schedule 下一次的执行时间点=上一次程序执行完成的时间点+间隔时间 
	而scheduleAtFixedRate 下一次的执行时间点=上一次程序执行完成的开始时间点+间隔时间
	我们看scheduleAtFixedRate的方法可以看出，传递第三个参数有正反分。
      public void scheduleAtFixedRate(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, System.currentTimeMillis()+delay, period);
    }
	*小结：Timer类提供了定时调度任务的schedule和scheduleAtFixedRate方法，有点区别。
	------------------------------------------------------------------------------------------------------------------
	接下来我们看看这个TimerTask类，看看这个类有啥特殊的？
	这个TimerThread是一个抽象的类，这个类继承了Thread类。其中抽象方法为：
	                 ```java
			 public abstract void run();
			 ```
	我们看看这个类中还有其他什么？通过下面几个变量标识线程的不同工作状态，默认值为0.
	             ```java
		     int state = VIRGIN;
		     static final int VIRGIN = 0;
		     static final int SCHEDULED   = 1;
		     static final int EXECUTED    = 2;
		     static final int CANCELLED   = 3;
		     ```
	这个类中还有两个和调度相关的变量： long nextExecutionTime;//下一次任务执行时间
					long period = 0; //任务调度周期，也就是每多长时间调度一次
	我们看看这个类中还有个取消的方法，这个方法只是将线程状态标识为cannel状态。
	               ```java
		       public boolean cancel() {
			synchronized(lock) {
			    boolean result = (state == SCHEDULED);
			    state = CANCELLED;
			    return result;
			}
		    }
		        ```
	------------------------------------------------------------------------------------------------------------------    
	下面是定时器调度算法的核心：
	```java
	    private void sched(TimerTask task, long time, long period) {
		if (time < 0)
		    throw new IllegalArgumentException("Illegal execution time.");

		//首先检查一下参数
		if (Math.abs(period) > (Long.MAX_VALUE >> 1))
		    period >>= 1;

		synchronized(queue) {
		    if (!thread.newTasksMayBeScheduled)
			throw new IllegalStateException("Timer already cancelled.");
		    //每个任务有一把锁，这个锁其实就是一个Object对象
		    synchronized(task.lock) {
			if (task.state != TimerTask.VIRGIN) //查看一下工作线程TimerThread的状态是否合法
			    throw new IllegalStateException(
				"Task already scheduled or cancelled");
			task.nextExecutionTime = time;
			task.period = period;
			task.state = TimerTask.SCHEDULED;
		    }
		    //加入任务到队列中
		    queue.add(task);
		    if (queue.getMin() == task) //该取任务了，如果取到当前任务，说明需要执行，立马唤醒！！！
			queue.notify();
		}
	    }
    ```
	-------------------------------------------------------------------------------------------------------------------
	接着，我们来看看TimerThread中的run方法。
	```java
	public void run() {
	    try {
		mainLoop();
	    } finally {
		// Someone killed this Thread, behave as if Timer cancelled
		synchronized(queue) {
		    newTasksMayBeScheduled = false;
		    queue.clear();  // Eliminate obsolete references
		}
	    }
	}
        ```
	这里，我们重点分析mainLoop方法，这个是定时器调度任务的基础。
	```java
	private void mainLoop() {
	    while (true) {
		try {
		    TimerTask task;
		    boolean taskFired;
		    synchronized(queue) {
			// 如果队列是空 且 newTasksMayBeScheduled 是 true，阻塞等待
			while (queue.isEmpty() && newTasksMayBeScheduled)
			    queue.wait();
			// 如果被唤醒了，且队列还是空，跳出循环结束。
			if (queue.isEmpty())
			    break; // Queue is empty and will forever remain; die
			// Queue nonempty; look at first evt and do the right thing
			long currentTime, executionTime;
			// 拿到队列中第一个任务。
			task = queue.getMin();
			synchronized(task.lock) {// 对这个任务进行同步
			    // 如果取消了，就删除这个任务，并跳过这次循环
			    if (task.state == TimerTask.CANCELLED) {
				queue.removeMin();
				continue;  // No action required, poll queue again
			    }
			    currentTime = System.currentTimeMillis();
			    executionTime = task.nextExecutionTime;
			    // 如果任务的下次执行时间小于当前时间，
			    if (taskFired = (executionTime<=currentTime)) {
				// 且任务是不重复的
				if (task.period == 0) { // Non-repeating, remove
				    // 删除这个任务
				    queue.removeMin();
				    // 修改状态
				    task.state = TimerTask.EXECUTED;
				} else { // Repeating task, reschedule
				    // 如果任务是重复的。重新调度任务时间，以便下次执行。
				    queue.rescheduleMin(
				      task.period<0 ? currentTime   - task.period
						    : executionTime + task.period);
				}
			    }
			}
			// 如果时间没到，就等代指定时间
			if (!taskFired) // Task hasn't yet fired; wait
			    queue.wait(executionTime - currentTime);
		    }
		    // 如果时间到了，就执行任务。
		    if (taskFired)  // Task fired; run it, holding no locks
			task.run();
		} catch(InterruptedException e) {
		    // 如果有中断异常就忽略。
		}
	    }
	}
        ```
	*总结：在我们创建Timer对象的时候，其实就启动了TimerThread线程。在我们调用schedule或者scheduleAtFixedRate时候，
             我们将我们的任务（也就是Runnable对象）包装成TimerTask对象，以及延迟参数和周期参数传递进task队列中。
             这样TimerThread在执行任务的时候拿到这个任务的这些参数，从而判断是一次性任务还是周期性任务（通过period是否为0），
             然后继续将任务添加到Queue中。每次去Queue中去任务的时候，都是取最小的任务，也就是离当前时间最近的任务进行处理。
	-------------------------------------------------------------------------------------------------------------------
	##总结
	（1）创建Timer定时器，有4个构造函数，默认会设置一个“Timer-自增数字”的名字。其他构造函数可以设置名字、设置守护线程。
	（2）每个Timer中包含了一个TimerThread，也就是说一个定时器就是一个线程。
             每个Timer中还包含了一个TaskQueue，从上面我们知道这个Queue就是装着TimerTask的对象。
	（3）在调用schedule/scheduleAtFixedRate的时候,会将这些参数放到自定义的TimerTask对象中，然后丢到Queue中。
        （4）在TimerThread进行调度任务的时候，会取到队列中的每一个任务，然后阻塞任务知道执行完成。
             如果是周期性任务，会重新包装成task放到Queue中。否则执行完成，退出线程了。
	（5）mainLoop方法忽略了线程中断异常。当 wait 方法中断异常的时候，是不起作用的。
             mainLoop方法值只捕获线程中断异常，如果发生了其他异常，整个 Timer 就会停止。
 	------------------------------------------------------------------------------------------------------------------
