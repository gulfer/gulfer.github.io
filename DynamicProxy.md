# Java动态代理

代理模式通常被我们分为静态代理和动态代理，静态代理的委托类和代理类在编译期就已经确定，而动态代理的委托类仍然是编译期确定，但代理类就是运行期动态生成的了。正好最近在项目中应用到了动态代理的技术，就顺便总结下Java动态代理使用的场景及几种创建动态代理的方式。

## 为什么要使用动态代理

GoF对代理模式的适用性是这么描述的：Remote Proxy（远程代理），个人理解主要应用在实现网络通信模型时，如CXF等WebService框架会为远程对象创建本地代理对象；Virtual Proxy（虚代理），可作为一种延迟实例化或调用的方法，比如流媒体播放等场景；Protection Proxy（保护代理），对委托对象提供访问控制，是一种保护机制。

See [Proxy pattern](https://en.wikipedia.org/wiki/Proxy_pattern)

而动态代理和静态代理本质上只是代理类实现的阶段不同，进而提供了更大的灵活性，在增加**额外**功能的同时，对原始代码透明，我认为这就是我们使用动态代理最主要的理由。

## 动态代理的原理

或许各种动态代理的实现方法在实现细节上存在差异，但原理基本都是一样的，就是运行时生成代理类的字节码。我们以JDK提供的动态代理机制为例做个简单说明，动态代理的例子网上太多了，这里不对写法上做过多描述：

业务接口及实现类：

```
interface A {

	void doSomething();

}

class Impl implements A {

	@Override
	public void doSomething() {
		System.out.println("do something");
	}
}
```

代理类：

```
class Proxy implements InvocationHandler {

	private Object target;

	public void setTarget(Object target) {
		this.target = target;
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) 
			throws Throwable {
		System.out.println("before");
		Object ret = method.invoke(target, args);
		System.out.println("after");
		return ret;
	}

}
```

代理类使用：

```
public static void main(String[] args) {
		A a = new Impl();
		Proxy proxy = new Proxy();
		proxy.setTarget(a);
		A a1 = (A) java.lang.reflect.Proxy.newProxyInstance(a.getClass().getClassLoader(), 
				a.getClass().getInterfaces(),
				proxy);
		a1.doSomething();
	}
```
第一个需要关注的是InvocationHandler接口，动态代理类会实现此接口，接口唯一的方法：

`Object invoke(Object proxy, Method method, Object[] args) throws Throwable`

形参proxy是被代理对象，method是要调用的被代理对象的方法，args就是调用方法接受的参数数组。在代理类的代码中我们可以看到并未使用被代理对象，而是直接在调用方法前后增加了输出。

第二个需要关注的就是创建动态代理的方法，最简单的就是使用Proxy.newProxyInstance方法：

`public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException`

loader是指定加载代理对象的classloader，interfaces是被代理对象实现的接口数组，这个数组的最大长度被限制为65535，h是实现InvocationHandler接口的代理对象。

查看newProxyInstance方法可以看到代理对象是如何被创建的：

跳过追溯过程直接查看生成代理类的关键代码：

```
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```
其中proxyClassCache是一个WeakCache实例，对代理类对象进行缓存：

```
/**
     * a cache of proxy classes
     */
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```

重点在于这些Class是如何创建并被缓存的，省略掉WeakCache中的代码，直接定位我们关注点，即ProxyClassFactory类的apply方法：

```
...
			String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
...
```
动态代理类的字节码是通过ProxyGenerator.generateProxyClass生成的，而Class对象又是通过defineClass0方法加载到classloader并创建。了解前者的机制需要查看sun.misc.ProxyGenerator的源码，而后者是native方法。另一个有趣的事情是我们创建的代理类的Class是$Proxy0（数字），这是因为这个类型是虚拟机动态创建的，它会实现在创建代理类时传入的接口数组。

有了Class就可以创建代理类的实例了，回到newProxyInstance方法，可以看到对象的实例化实际上仍然是通过构造方法执行的，只不过这个构造方法比较特别，它的入参是我们之前创建的InvocationHandler对象，显然这个构造方法也是ProxyGenerator动态创建的。

```
final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
```
至此动态代理的创建过程才真正完成。

## 创建动态代理的方式及对比

目前比较主流的创建动态代理的方式可以说有两种：一是上文介绍的JDK动态代理，二是通过直接操作字节码创建动态代理，如ASM、cglib、Javassist等。

CXF Spring cglib




