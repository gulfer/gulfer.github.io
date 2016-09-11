# Lambda And Anonymous Class

曾经一直认为Java8引入的函数式编程与匿名类没什么差别，起码在功能上，直到给自己留了作业试着去了解、接受函数式编程。

函数式编程与面向对象编程都是编程思想，一种面向对象语言引入函数式编程思想有什么意义，本文不做探讨。我只关注题目所示的Lambda表达式与匿名类。

其实Java在之前的版本就已经提供的一些接口可当做函数式，比如Runnable，就可以借助匿名类实现。其他框架如Guava，也提供了函数式接口。

## 功能比较

OOP要求我们关注对象，而函数式编程让我们关注过程。匿名类在一定程度上简化了Java开发者对类的关注，但你仍然不能脱离接口，而Lambda表达式是匿名函数，你只要关心写好函数体就可以了。我觉得这是本质上的差别。

#### 减少代码冗余

写个匿名类，你要么定义个接口，要么定义抽象类

这是Firefly的一段代码：

```
factory.init(dataSourceConfig,
				new ValueLoader<String, Class<DataSource>>() {

					@Override
					public Class<DataSource> load(String className)
							throws LoaderException {
						return DataSource.class;
					}

				});
```

改成Lambda之后变成这个鬼样：

```
ValueLoader<String, Class<DataSource>> loader = (s) -> DataSource.class;
factory.init(dataSourceConfig, loader);
```
有几点需要注意：

* 如果是抽象类，需要使用注解@FunctionalInterface

* 不是什么情况下都可以用Lambda改写，当你的接口只包含了一个抽象方法的时候才可以替换成Lambda表达式，静态方法除外

* 对于异常，unchecked异常在函数式接口中不用声明，抛出checked异常仍然需要声明

总的来说代码精简了不少，这是最主要的一个差异。不过如果仅此而已，那么充其量也就是个语法糖。

#### final变量

Java8对匿名类和Lambda解除了final变量的限制，但实际上，在Lambda中引用一个值变化过的变量仍然会报错。

![](https://github.com/gulfer/gulfer.github.io/blob/master/pic/lambda%20final%20error.png)

## 使用场景

其实使用场景下匿名类和Lambda表达式没有太大差异。不过上面也提到了，Lambda表达式有一定限制，首先就是Lambda表达式的目标类型必须是SAM（Single Abstract Method）接口，像Runnable、Callbable这样的。

## 性能
   

