# 关于jfinal的actionReporter
> jfinal 2.0

## 是什么
名字解释即行为报告，主要做的就是对于程序代码controller类运行时的日志打印。

## 做什么
唯一一个做事的方法是doReport()方法。这个方法主要就是打印request调用controller具体方法，方法执行的拦截器，以及传递的参数。

## 什么时候起作用
只有在开发模式中起作用，devMode=true时。在ActionHandler类中有调用，切只有在devMode为true时，才有用到ActionReporter类。

## 随便看看

### simpleDateFormat 
final类，定义了一个
	
	private static final ThreadLocal<SimpleDateFormat> sdf = new ThreadLocal<SimpleDateFormat>(){
		protected SimpleDateFormat initialValue(){
			return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		}
	}

这个是线程安全的SimpleDateFormat的使用。可以查看文章：
https://blog.csdn.net/csdn_ds/article/details/72984646

解决方法：

1. 如果是java8的话：可以使用Instant代替Date，LocalDateTime代替Calendar，DateTimeFormatter代替Simpledateformatter（推荐使用）
2. 使用ThreadLocal：每个线程拥有自己的SimpleDateFormat对象。（推荐使用）
3. 使用第三方库joda-time，由第三方考虑线程不安全的问题。（可以使用）
4. 方法加同步锁synchronized，在同一时刻，只有一个线程可以执行类中的某个方法。缺点：性能较差，每次都要等待锁释放后其他线程才能进入。
5. 将SimpleDateFormat定义成局部变量。缺点：每调用一次方法就会创建一个SimpleDateFormat对象，方法结束又要作为垃圾回收。