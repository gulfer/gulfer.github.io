# Lambda And Anonymous Class

曾经一直认为Java8引入的Lambda表达式与匿名类没什么差别，起码在功能上，抱着一点好奇给自己留个作业，试着去了解Lambda以及Java版的函数式编程。函数式编程与面向对象编程都是编程思想，一种面向对象语言引入函数式编程思想有什么意义，本文不做探讨。我只关注题目所示的Lambda表达式与匿名类。

Lambda并不是Java新增的某种函数类型，本质上Java编译器会把它解析为某种函数接口（Functional Interface）。其实Java在之前的版本就已经提供的一些接口可当做函数式，比如Runnable，就可以借助匿名类实现。其他框架如Guava，也提供了函数接口。

## 功能比较

OOP要求我们关注对象，而函数式编程让我们关注过程。匿名类在一定程度上简化了Java开发者对类的关注，但你仍然不能脱离接口，而Lambda表达式是匿名函数，你只要关心写好函数体就可以了。我觉得这是本质上的差别。

#### 代码精简

写个匿名类，你要么定义个接口，要么定义抽象类

这是Firefly的一段代码：

```java
factory.init(dataSourceConfig,
				new ValueLoader<String, Class<DataSource>>() {

					@Override
					public Class<DataSource> load(String className)
							throws LoaderException {
						return DataSource.class;
					}

				});
```

改成Lambda之后变成这样：

```java
ValueLoader<String, Class<DataSource>> loader = (s) -> DataSource.class;
factory.init(dataSourceConfig, loader);
```
有几点需要注意：

* 如果是抽象类，需要使用注解@FunctionalInterface

* 不是什么情况下都可以用Lambda改写，当你的接口只包含了一个抽象方法的时候才可以替换成Lambda表达式，静态方法除外

* 对于异常，unchecked异常在函数式接口中不用声明，抛出checked异常仍然需要声明

总的来说代码精简了不少，这是最主要的一个差异（还是语法糖）。

#### final变量

Java8对匿名类和Lambda解除了final变量的限制，但实际上，在Lambda中引用一个值变化过的变量仍然会报错。

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/lambda%20final%20error.png)

#### this

Lambda定义了一个闭包，this变量在闭包内部引用的是外部类实例，而在内部类中的this，引用的是内部类实例

```java
private String s = "Outter";

void sayHelloInInner(LambdaTest2 test){
		test.sayHello(new Hello2() {
			
			private String s = "inner";

			@Override
			public void print() {
				System.out.println(this.s);
			}
			
		});
	}
	
void sayHelloInLambda(LambdaTest2 test) {
		test.sayHello(() -> System.out.println(this.s));
	}
```
两个方法依次调用则打印：
inner
outter

引申而言，Lambda表达式没有任何隐藏变量，即不会从父类中集成任何变量，同名的变量在闭包内外语义相同。

> Like local and anonymous classes, lambda expressions can capture variables; they have the same access to local variables of the enclosing scope. However, unlike local and anonymous classes, lambda expressions do not have any shadowing issues (see Shadowing for more information). Lambda expressions are lexically scoped. This means that they do not inherit any names from a supertype or introduce a new level of scoping. Declarations in a lambda expression are interpreted just as they are in the enclosing environment. 

#### 使用场景

其实匿名类和Lambda表达式的使用场景并没有太大差异，功能上两者互通。不过上面也提到了，Lambda表达式有一定限制，首先就是Lambda表达式的目标类型必须是SAM（Single Abstract Method）接口，像Runnable、Callbable这样的。

## 性能
   
做了个简单的测试：

```java
public class LambdaTest3 {

	public static void main(String[] args) {
		LambdaTest3 test = new LambdaTest3();
		int ceiling = 10000;
		
		long start = System.currentTimeMillis();
		System.out.println("Count primes from 0 up to " + ceiling + " with Anonymous Class ===");
		System.out.println("count : " + test.countPrimesWithAC(ceiling));
		System.out.println("use : " + (System.currentTimeMillis() - start));
		
		System.out.println("======");
		
		start = System.currentTimeMillis();
		System.out.println("Count primes from 0 up to " + ceiling + " with Lambda ===");
		System.out.println("count : " + test.countPrimesWithLambda(ceiling));
		System.out.println("use : " + (System.currentTimeMillis() - start));
	}
	
	int countPrimesWithAC(int ceiling) {
		return new PrimeCounter() {

			@Override
			int countPrimes(int ceiling) {
				int count = 0;
				for (int i=0;i<ceiling;i++) {
					if (isPrime(i)) {
//						System.out.println(i);
						count++;
					}
				}
				return count;
			}
			
		}.countPrimes(ceiling);
	}
	
	long countPrimesWithLambda(int ceiling) {
		return IntStream.range(0, ceiling).filter(i -> isPrime(i)).count();
	}
	
	boolean isPrime(int i) {
		for (int j=2;j<i;j++) {
//		for (int j=2;j<=Math.sqrt(i);j++) {
			if (i%j == 0) {
				return false;
			}
		}
		return true;
	}
	
	abstract class PrimeCounter {
		
		abstract int countPrimes(int ceiling);
		
	}

}
```
统计10000以内的质数，请不要纠结算法本身的效率，要的就是稍微明显点的差异，多次统计结果基本相同：

```
Count primes from 0 up to 10000 with Anonymous Class ===
count : 1231
use : 20
======
Count primes from 0 up to 10000 with Lambda ===
count : 1231
use : 65
```
此用例中，Lambda的执行时间大约在匿名类的3倍以上，这似乎与官方的benchmark测试结果差异较大。

而当isPrime方法替换成以下isPrime1方法后：

```java
boolean isPrime1(int i) {
		return IntStream.range(2, i).allMatch(j -> (i%j != 0));
	}
```
更糟糕，超过4倍了：

```
Count primes from 0 up to 10000 with Anonymous Class ===
count : 1231
use : 20
======
Count primes from 0 up to 10000 with Lambda ===
count : 1231
use : 95
```

改写成并行流后：

```java
long countPrimesWithLambda(int ceiling) {
		return IntStream.range(0, ceiling).parallel().filter(i -> isPrime(i)).count();
	}
......
boolean isPrime1(int i) {
		return IntStream.range(2, i).parallel().allMatch(j -> (i%j != 0));
	}
```
似乎有所提升，又回到3倍了：

```
Count primes from 0 up to 10000 with Anonymous Class ===
count : 1231
use : 18
======
Count primes from 0 up to 10000 with Lambda ===
count : 1231
use : 53
```
我试着做一些假设，匿名类的性能成本在于类加载及构造对象，Lambda的性能成本在于编译器需要将表达式解析为函数接口，同样也需要执行类加载和构造对象等操作。构造对象操作每次都有，理论上问题不大，所以问题有可能在于表达式的解析或类加载。如果每一个相同的Lambda都需要执行这些操作，那么我只做一次会怎么样。

为了公平起见，我把匿名类和Lambda同时定义为类变量。

```java
final PrimeCounter primeCounter = new PrimeCounter() {

		@Override
		int countPrimes(int ceiling) {
			int count = 0;
			for (int i=0;i<ceiling;i++) {
				if (isPrime(i)) {
					count++;
				}
			}
			return count;
		}
		
	};
	
final IntPredicate predicate1 = (i) -> isPrime(i);
```
把4种方式测试结果都打出来：

```
Count primes from 0 up to 10000 with Anonymous Class ===
count : 1231
use : 18
======
Count primes from 0 up to 10000 with Lambda ===
count : 1231
use : 12
======
Count primes from 0 up to 10000 with final Anonymous Class ===
count : 1231
use : 15
======
Count primes from 0 up to 10000 with final Lambda ===
count : 1231
use : 16
```
结果基本证实了之前的假设。Java编译器对Lambda表达式的执行优化可能不像对匿名类那样成熟，性能损耗可能主要出现在Lambda表达式解析或类加载上。所以当Lambda定义为类变量后，表达式解析和类加载均只执行了一次，执行效率得到提升。不过这仍然只是猜测，需要详细看看编译器如何处理包含Lambda表达式的类才能进一步分析。

此外，官方提供的benchmark测试结果是Lambda在最坏的情况下比匿名类要好5倍，所以或许代码上还可以优化。

如果想了解官方对二者性能方面更详细的比较，E文靠谱的同学可围观以下视频，不过口音听起来很要命：
[Lambda vs Anonymous Class](http://medianetwork.oracle.com/video/player/2623576348001)

相关测试代码：
[Java8 Lambda Demo](https://github.com/gulfer/java8)

简单总结一下，到目前为止我仍然认为Lambda表达式对Java而言更多的是语法糖，Lambda能做的事匿名类同样可以做到，所以功能性方面并没有实际的增加。不过对于Java来说，我还是很认同引入函数式编程这种思想的，更简洁的代码，更优雅的集合类操作方式，（不提并行化是我觉得Java7已经有ForkJoinPool了），所以对语言本身，是进步。而回到本文的主题Lambda和匿名类，我还是持比较保守的态度，在使用上还是需要结合具体场景并关注性能消耗，不过我们的工作环境恐怕一时半会也用不上Java8。


