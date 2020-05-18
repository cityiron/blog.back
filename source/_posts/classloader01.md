---
title: 类加载总结
date: 2019-10-07 15:29:11
tags: [classloader, springboot, 原创]
categories: 类加载
---

[classloader] 类加载日常总结

<!-- more --> 

> 本文内容暂时基于1.7

在上一篇文章提到一个JVM加载多个boot项目（https://blog.funnycode.cn/springboot/2019-09-28-springboot-one-jvm/），继续看这块内容涉及到了类加载，springboot内嵌tomcat，spring子容器等，在这篇文章做一个大而不一定全的总结。

# JVM类加载

> 类加载机制：虚拟机把描述类的数据从 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型

## 类的生命周期

![类加载生命周期](https://res.cloudinary.com/dogbaobao/image/upload/v1571546274/blog/classloader/classonload-3-wm-1571546246_ect8nx.png)

- 加载
- 连接（验证，准备，解析）
- 初始化
- 使用
- 卸载

如图所示，类加载有7个阶段。其中加载、验证、准备、初始化和卸载这5个阶段的顺序是确定的，类的加载过程必须按照这种顺序按部就班的执行。而解析阶段不一定，为了支持Java语言的运行时绑定（也称为动态绑定和晚期绑定），它可以在初始化阶段之后再开始。要注意这些阶段通常都是相互交叉地混合式进行的，通常会在一个阶段执行的过程中调用、激活另一个阶段。

## 类的加载过程

> 主要内容参考《深入理解Java虚拟机》

### 加载

> 类加载的第一个阶段

JVM会完成如下3件事情
- 通过一个类的全限定名来获取定义此类的二进制字节流
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
- 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口

因为没有指定字节流从哪里获取，常见的 ` .class ` 文件的加载方式如下：
- 从ZIP包中读取，现在主要发展为 JAR 、 WAR ，比如我们最常见的 springboot 的 fat.jar 和 Tomcat 用的 war
- 通过网络去获取
- 运行时计算生成，这种场景使用的最多的就是动态代理技术， JDK 的动态代理和 cglib
- 通过其他文件生成，比如 JSP 应用
- 从数据库中读取，很少见

相对于类加载的其他阶段，一个非数据组类的加载阶段（准确地说，是加载阶段中获取类的二进制字节流的动作）是开发人员可控性最强的，因为加载阶段既可以使用系统提供的引导类加载器来完成，也可以由用户自定义的类加载器去完成，在自定义类加载器中重写一个类加载器的 ` loadClass() ` 方法

对于数组类而言，数组类本身不通过类加载器创建，它是由JVM直接创建的。但是数组类的元素类型最终是要靠类加载器去创建。一个数组类会遵循如下规则：
- 如果数组的组件类型是引用类型，那就递归采用上面提到的加载过程去加载这个组件类型，会和这个加载该组件的类加载器关联
- 如果数组的组件类型不是引用类型（ int[] )， JVM 会把数组标记为与引导类加载器关联
- 数组类的可见性与它的组件类型可见性一致，如果组件类型不是引用类型， 那数组类型的可见性默认为 public 

> 类加载器加载.class文件后，JVM会把.class对应的二进制字节流按虚拟机所需的格式存储在方法区之中，方法区中的数据存储格式由虚拟机自定义，虚拟机规范未规定此区域的具体数据结构。然后在内存中实例化一个 java.lang.Class 类的对象（注意这个也没规定是在Java堆中，对于 HotSpot虚拟机而言，是存放在方法区的）。还有注意加载阶段和连接阶段的部分内容是交叉进行的，只是说这两个阶段的开始时间是保持有序的，加载阶段开始早于连接阶段。

### 验证

验证是连接阶段的第一步，这一阶段的目标是确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

> 对各种来源的二进制字节流做检查，是虚拟机自我保护的必要操作

验证阶段大致会完成如下4个阶段：

- 文件格式验证，验证字节流是否符合Class文件格式的规范。主要保证输入的字节流能正确地解析并存储于方法区内，只有通过这个环节字节流才会进入内存的方法区中进行存储，后面的3个步骤都只会基于方法区的存储结构进行，不会再直接操作字节流。如魔数0xCAFEBASE开头，常量池的常量否是支持类型等
- 元数据验证，对字节码描述的信息进行语义分析，保证不存在不符合Java语言规范的元数据信息。如类的父类，不是抽象类的类是否实现接口的所有方法等
- 字节码验证，整个验证过程最复杂的一个阶段，主要目的是通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。会对类的方法体进行校验分析，保证被校验类在运行时不会危害虚拟机安全
- 符号引用验证，这个阶段发生在虚拟机将符号引用转化为直接引用的时候，这个转换动作是在 解析阶段 发生。因此这一步的作用主要就是确保解析动作能正常执行

验证阶段虽然非常重要的，但是对于JVM而言并不是必要的，可以通过 -Xverify:none 参数来关闭大部分的类验证，缩短虚拟机类加载时间

### 准备

准备是正式为类变量分配内存并设置类变量初始值的阶段，这些变量所使用的内存都将在方法取中进行分配。

准备阶段的内存分配仅包括类变量（被static修饰的变量）而不包括实例变量，实例变量会在对象实例化的时候随着对象一起分配到 Java 堆中
初始化“通常情况”下是数据类型的零值。如 ` public static int value = 123; ` value 变量准备阶段过后的初始值是0而不是123，而把 value 设置成 123 的动作将在初始化阶段才会执行。
当类字段的字段属性表中存在 ConstantValue 属性，name在准备阶段变量 value 就会被初始化为 ConstantValue 属性所指定的值。如 ` public static final int value = 123; ` 编译时 Javac 将会为 value 生成 ConstantValue 属性，在准备阶段虚拟机就会根据 ConstantValue 的设置将 value 赋值为 123

### 解析

解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，通俗的说符号引用就是以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可；直接引用就是直接指向目标的指针、相对偏移量或是一个间接定位到目标的句柄。后者和JVM内存布局有关，前者无关。

解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行。

类型表如下：

![常量图](https://res.cloudinary.com/dogbaobao/image/upload/v1571545479/blog/classloader/classloader-1_wjcibs.png)

常量内容可以查看
https://www.jianshu.com/p/d8492e748c57

### 初始化

类初始化是类加载过程的最后一步，前面的类加载过程中，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（字节码）

初始化是执行类构造器 ` <client>() ` 方法的过程

#### ` <client> ` 和 ` <init> `

##### clinit
clinit指的是类构造器，这个构造器是jvm自动合并生成的，在jvm第一次加载class文件时调用，包括
- 静态变量初始化语句和静态块的执行，它合并static变量的赋值操作，注意是赋值操作，(仅声明，或者final static)不会触发，毕竟前面准备阶段已经默认赋过值为0了
- static{}语句块生成，且虚拟机保证执行前，父类的已经执行完毕，所以说父类如果定义static块的话，一定比子类先执行
- 如果一个类或接口中没有static变量的赋值操作和static{}语句块，那么不会被JVM生成
- static变量的赋值操作和static{}语句块合并的顺序是由语句在源文件中出现的顺序所决定的。

##### init
在实例创建出来的时候调用，也就是构造函数，包括:
- new操作符
- 普通代码块
- 调用Class或java.lang.reflect.Constructor对象的newInstance()方法；
- 调用任何现有对象的clone()方法；
- 通过java.io.ObjectInputStream类的getObject()方法反序列化

下面是一个例子，来补充上面的说明：

```java
public class DogMaoMao {

    private static DogMaoMao singleton = new DogMaoMao();

    public static int c1;

    public static int c2 = 10;

    private DogMaoMao() {
        c1++;
        c2++;
    }

    public static DogMaoMao getInstance(){
        return singleton;
    }

    @Override
    public String toString(){
        return "c1 : " + c1 + " -- c2 : " + c2 + " -- age : " + Dog.age;
    }

}

public class Run {

    public static void main(String[] args) {
        DogMaoMao instance = DogMaoMao.getInstance();
        System.out.println(instance);
    }

}
```

输出结果：
```bash
I'm Dog
c1 : 1 -- c2 : 10 -- age : 18
```

分析：
1. 通过main方法入口加载，Run类初始化
2. DogMaoMao.getInstance(); 开始DogMaoMao初始化，准备阶段的时候初始值
```java
    private static DogMaoMao singleton;

    public static int c1;

    public static int c2;
```
3. DogMaoMao执行到初始化阶段，生成类构造器
```java
public static int c1;
static {
	// 按类里面的位置正向运行
	private static DogMaoMao singleton = new DogMaoMao();
	public static int c2 = 10;
}
```

#### 初始化条件

Java程序对类的使用方式可以分为两种：
- 主动使用：会执行加载、连接、初始化静态域
- 被动使用：只执行加载、连接，不执行类的初始化静态域

JVM严格规定了有且只有5种情况必须立即进行初始化：

- 遇到 new 、 getstatic 、 putstatic 或 invokestatic 这4条字节码指令时，如果类没有进行过初始化，则需要先触发其初始化。常见场景：使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被 final 修饰、已在编译器把结果放入常量池的静态字段除外），以及调用一个类的静态方法的时候
- 使用 ` java.lang.reflect ` 包的方法对类进行反射调用的时候，如果类没有进行初始化，则需要先触发其初始化
- 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含 main() 方法的那个类），虚拟机会先初始化这个类
- 当使用 JDK 1.7 的动态语言支持时，如果一个 ` java.lang.invoke.MethodHanlder ` 实例最后的解析结果 REF_getStatic、REF_putStatic、REF_invokeStatic 的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化。

除去上面的5种场景，所有引用类的方式都不会触发初始化，称为被动引用

接口的规范和上面类的区别在于第三点，在接口初始化时，并不要求其父接口全部都完成了初始化。

#### Class.forName和ClassLoader.loadClass

> 这里讨论的Class.forName是Class类的方法public static Class<?> forName(String className) throws ClassNotFoundException

```java
public abstract class Dog {
  
    static {
        System.out.println("I'm Dog");
    }  
  
}

public class DogBaoBao extends Dog{

    static {
        System.out.println("I'm Dog Bao Bao");
    }

}
```

- Class.forName()会初始化

```java
public class DbbClassLoader {
  
    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> aClass = Class.forName("com.dbb.blog.DogBaoBao");
    }
  
}
```

输出

```bash
I'm Dog
I'm Dog Bao Bao
```

- ClassLoader.loadClass不会初始化

```java
public class DbbClassLoader {

    public static void main(String[] args) throws ClassNotFoundException {
        ClassLoader classLoader = DbbClassLoader.class.getClassLoader();
        Class<?> loadClass = classLoader.loadClass("com.dbb.blog.DogBaoBao");
    }

}
```

输出内容为空

#### 常见类加载器说明 

```java
class Dog {

    public void wangwang() {
        Dog.getClassLoader().loadClass(“Cat”);
    }
    
}
```

CurrentClassLoader，称之为当前类加载器，简称CCL，在代码中对应的就是类型 Dog 的类加载器。
SpecificClassLoader，称之为指定类加载器，简称SCL，在代码中对应的是  Dog.class.getClassLoader()，如果使用任意的 ClassLoader 进行加载，这个  ClassLoader 都可以称之为SCL。
ThreadContextClassLoader，称之为线程上下文类加载器，简称TCCL，每个线程都会拥有一个 ClassLoader 引用，而且可以通过 Thread.currentThread().setContextClassLoader(ClassLoader classLoader) 进行切换。

## 类加载器

> 类加载器一开始是为了 Java Applet 的需求而开发出来的，虽然这个技术已经 “死掉”，但类加载器在类层次划分、OSGI、热部署、代码加载等领域大放异彩

java语言系统内置了众多类加载器，从一定程度上讲，只存在两种不同的类加载器：一种是启动类加载器，此类加载由C++实现，是JVM的一部分；另一种就是所有其他的类加载器，这些类加载器均由java实现，且全部继承自` java.lang.ClassLoader `

- Bootstrap ClassLoader 启动类加载器，最顶层的加载类，由C++实现，负责加载 <JAVA_HOME>/lib 目录中或 -Xbootclasspath 中参数指定的路径中的，并且是虚拟机识别的（按名称，如rt.jar）类库，所以名字不符合的类库就算放在 lib 目录中也不会被加载）

- Extention ClassLoader 扩展类加载器，由启动类加载器加载，实现为` sun.misc.Launcher$ExtClassLoader `，负责加载目录 <JAVA_HOME>/lib/ext 目录中或 -Djava.ext.dirs 中参数指定的路径中的 jar 包和 class 文件

- Application ClassLoader 应用类加载器，也称为系统类加载器(System ClassLoader，可由` java.lang.ClassLoader.getSystemClassLoader() `获取)，实现为` sun.misc.Launcher$AppClassLoader `，由启动类加载器加载，负责加载当前应用 ClassPath 下的所有类，一般应用程序没有自定义自己的类加载器，这个就是程序的默认类加载器。

> All classes are loaded based on their names and if any of these classes are not found then it returns a NoClassDefFoundError or ClassNotFoundException.

我们一般项目的类加载器结构如下：

![双亲委派模型](https://res.cloudinary.com/dogbaobao/image/upload/v1571546275/blog/classloader/classloader-2-wm-1571546245_u7hebn.jpg)

### 类的唯一性

类全限定名称+加载它的类加载器来确立在 Java 虚拟机中的唯一性，通俗的说就是两个全路径都一样的类，必须要同一个类加载器加载才能比较

在JVM中，类型被定义在一个叫 SystemDictionary 的数据结构中，该数据结构接受类加载器和全类名作为参数，返回类型实例。

` SystemDictionary `，系统字典，这个数据结构是保存Java加载类型的数据结构，如下图所示：

![JVM](https://res.cloudinary.com/dogbaobao/image/upload/v1571546275/blog/classloader/classloader-4-wm-1571546246_lzd3c1.png)

###  双亲委派模型（Parents Delegation Model）

> 源自JDK 1.2

> 在双亲委派模型中除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。类加载器之间的父子关系一般不会以继承的关系来实现，而都是用组合的关系来复用父加载器

### 双亲委派如何工作

如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父类加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去加载

### 双亲委派的好处

Java类随着它的类加载器一起具备了带有优先级的层次关系。比如 ` java.lang.Object `，它存放在 rt.jar 之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器加载，保证了 ` Object ` 类在程序的各种类加载器环境中都是同一个类。（你无法通过自定义 ` java.lang.Object ` 来覆盖捣乱）

### 实现细节

` java.lang.ClassLoader#loadClass(java.lang.String, boolean) `

```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

如果父加载器不为空，会让父类去加载，否则自己去加载

# 自定义Classloader

编写自定义的Classloader，需要满足以下两点：
- 继承java.lang.ClassLoader
- 重写父类的findClass方法

## 普通的使用
```java
public class DbbClassLoader extends ClassLoader {

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String root
            = "/Users/tc/Documents/workspace/github/sample-parent/sample-java-base/java-advance/classloader/target"
            + "/classes/com/dbb/blog/";
        String file = root + name.substring(name.lastIndexOf(".") + 1) + ".class";
        try (InputStream is = new FileInputStream(file);
             ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            int buffSize = 1024;
            byte[] buffer = new byte[buffSize];

            int length;
            while ((length = is.read(buffer)) != -1) {
                baos.write(buffer, 0, length);
            }

            byte[] bytes = baos.toByteArray();
            return defineClass(name, bytes, 0, bytes.length);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return null;
    }
}
```

```java
public class DbbClassLoaderTest {

    public static void main(String[] args) throws Exception {
        DbbClassLoader dbbClassLoader = new DbbClassLoader();
        // Class<?> dogClass = dbbClassLoader.findClass("Dog");
        // error, NoClassFoundError
        Class<?> dogClass = dbbClassLoader.findClass("com.dbb.blog.DogMaoMao");

        Constructor<?> constructor = dogClass.getDeclaredConstructor();
        constructor.setAccessible(true);
        Object o = constructor.newInstance();
        System.out.println(o instanceof DogMaoMao);
    }

}
```

碰到的一些问题

### 初始化失败
```text
Exception in thread "main" java.lang.InstantiationException
	at sun.reflect.InstantiationExceptionConstructorAccessorImpl.newInstance(InstantiationExceptionConstructorAccessorImpl.java:48)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at java.lang.Class.newInstance(Class.java:442)
	at com.dbb.blog.DbbClassLoaderTest.main(DbbClassLoaderTest.java:15)
```

类需要有构造函数或者不能是 abstract

### 类型转换
```java
DogMaoMao dogMaoMao = (DogMaoMao)constructor.newInstance();
```

```text
Exception in thread "main" java.lang.ClassCastException: com.dbb.blog.DogMaoMao cannot be cast to com.dbb.blog.DogMaoMao
	at com.dbb.blog.DbbClassLoaderTest.main(DbbClassLoaderTest.java:20)
```

## Fat.jar使用

> 这里主要讲的是 springboot 的打包方式，因为这种是我们应用中实际用的

- URLClassLoader

```java
/**
 * This class loader is used to load classes and resources from a search
 * path of URLs referring to both JAR files and directories. Any URL that
 * ends with a '/' is assumed to refer to a directory. Otherwise, the URL
 * is assumed to refer to a JAR file which will be opened as needed.
 * <p>
 * The AccessControlContext of the thread that created the instance of
 * URLClassLoader will be used when subsequently loading classes and
 * resources.
 * <p>
 * The classes that are loaded are by default granted permission only to
 * access the URLs specified when the URLClassLoader was created.
 *
 * @author  David Connelly
 * @since   1.2
 */
public class URLClassLoader extends SecureClassLoader implements Closeable {
}
```

- 加载路径

源码中提到了可以获取Jar或目录的URLS引用，以 / 结尾，实际项目中我们可以发现路径类似于 XXXX/XXXX/XXXX/X.jar!/

org.springframework.boot.loader.archive.JarFileArchive#getNestedArchives
```java
	public List<Archive> getNestedArchives(EntryFilter filter) throws IOException {
		List<Archive> nestedArchives = new ArrayList<>();
		for (Entry entry : this) {
			if (filter.matches(entry)) {
				nestedArchives.add(getNestedArchive(entry));
			}
		}
		return Collections.unmodifiableList(nestedArchives);
	}
```

实际加载的文件路径如下：
![](https://res.cloudinary.com/dogbaobao/image/upload/v1571579284/blog/classloader/classloader-8-wm-1571579248_hm22l5.jpg)

- 资源加载

```java
  // org.springframework.boot.loader.jar.JarFile#registerUrlProtocolHandler
	public static void registerUrlProtocolHandler() {
		String handlers = System.getProperty(PROTOCOL_HANDLER, "");
		System.setProperty(PROTOCOL_HANDLER,
				("".equals(handlers) ? HANDLERS_PACKAGE : handlers + "|" + HANDLERS_PACKAGE));
		resetCachedUrlHandlers();
	}
	
	//  org.springframework.boot.loader.LaunchedURLClassLoader#definePackage
	private void definePackage(String className, String packageName) {
		try {
			AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
				String packageEntryName = packageName.replace('.', '/') + "/";
				String classEntryName = className.replace('.', '/') + ".class";
				for (URL url : getURLs()) {
					try {
						URLConnection connection = url.openConnection();
						// ......
					}
					catch (IOException ex) {
						// Ignore
					}
				}
				return null;
			}, AccessController.getContext());
		}
		catch (java.security.PrivilegedActionException ex) {
			// Ignore
		}
	}
	
	java.net.URL#openConnection()
	
	public URLConnection openConnection() throws java.io.IOException {
  		return handler.openConnection(this);
  }	
```

URL的openConnection()实际上被委托给了URLStreamHandler处理，针对不同的协议比如jar,file,http,调用不同的hander。

- spring官方

https://github.com/spring-projects/spring-boot.git

![boot项目结构](https://res.cloudinary.com/dogbaobao/image/upload/v1571578510/blog/classloader/classloader-7-wm-1571578473_gdkqlw.jpg)

官方的处理逻辑在 spring-boot-loader 模块中

> 官方 jar file 说明
https://docs.spring.io/spring-boot/docs/current/reference/html/executable-jar.html#executable-jar-jar-file-structure

# Tomcat

> 简单的介绍Tomcat中类加载器的设计（主流的Java Web服务器，都实现了自己定义的类加载器）

因为现在主流的微服务开发，我们常常是基于k8s把应用部署在docker里面运行，已经较少的会用到一个Tomcat下放多个war包去运行的情况。所以下面的内容看看就好。

类加载这一块，Web服务器一般会考虑的几点：
- 部署在同一个服务器上的两个Web应用程序所使用的Java类库可以实现相互隔离，应为2个Web程序对同一个类库的依赖的版本不一定一致
- 部署在同一个服务器上的两个Web应用程序所使用的Java类库可以共享，类库在使用的时候需要被加载到服务器内存，这样可以重复利用。主要是spring相关的类
- 服务器需要尽量不受部署的Web应用程序的影响，服务器依赖的类库会独立
- 一些如JSP等文件（变动频率较高）的热替换

Tomcat5.X的类加载器如下：
![](https://res.cloudinary.com/dogbaobao/image/upload/v1571554492/blog/classloader/classloader-5-wm-1571554457_k8w48e.png)

我开始用Tomcat的时候已经是7.X了，我们一般只会看到 /lib 目录，它是上面提到的3个目录的合并， CatalinaClassLoader 和 SharedClassLoader 需要在 tomcat/conf/catalina.properties 配置文件配置 server.loader 和 shared.loader 后才会建立，默认情况下都统一用 CommonClassLoader 代替

# 破坏双亲委派模型

## OSGI

> OSGI联盟制定的一个基于Java语言的动态模块化规范

OSGI能实现模块化热部署，它的每个程序模块（成为Bundle）都有一个自己的类加载器，而每个 Bundle 都可以声明它所依赖的 Java Package（Import-Package描述），也可以通过声明它允许导出发布的 Java Package（Export-Package描述）。Bundle之间的依赖从上下层的依赖关系变成了平级模块之间的依赖，变成一个复杂的网状结构。

OSGI比较复杂，虽然简单的看了查找规则，但是未实际使用过，暂时不做过多描述。

## 线程上下文类加载器

Java中所有涉及SPI的加载动作基本上都采用这种方式，如JNDI、JDBC、JAXB等。

这里拿JDBC驱动举例

![](https://res.cloudinary.com/dogbaobao/image/upload/v1571578496/blog/classloader/classloader-6-wm-1571578473_occqfm.jpg)
` java.sql.Driver ` 定义在rt.jar中，由JVM自行加载，接口可以由不同的数据库厂商实现，来提供具体数据库的驱动，并且需要按照 SPI 的规范，在 /META-INF/services/ 目录里，创建一个服务接口命名的文件

```java
public class DriverManager {

    /* Prevent the DriverManager class from being instantiated. */
    private DriverManager(){}
    
    /**
     * Load the initial JDBC drivers by checking the System property
     * jdbc.properties and then use the {@code ServiceLoader} mechanism
     */
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
    
    private static void loadInitialDrivers() {
        // ......
    
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
    
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
    
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });
    
        // ......
    }
}
```

通过 ` java.util.ServiceLoader#load(java.lang.Class<S>) `加载的类
```java
public final class ServiceLoader<S>
    implements Iterable<S> {
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
}
```

里面就是用了线程上下文类加载器，如果你不设置，默认就是AppClassLoader

这里补充说明一下 spring 加载的时候，资源也是通过线程上下文类加载器加载的，下面的类 ` DefaultResourceLoader ` 是资源加载的对象，它里面有一个 classLoader 属性

```java
public class DefaultResourceLoader implements ResourceLoader {

	private ClassLoader classLoader;
	
		@Override
	public ClassLoader getClassLoader() {
		return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
	}

}
```

```java
public abstract class ClassUtils {
		public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
			// Cannot access thread context ClassLoader - falling back...
		}
		if (cl == null) {
			// No thread context class loader -> use class loader of this class.
			cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				// getClassLoader() returning null indicates the bootstrap ClassLoader
				try {
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
				}
			}
		}
		return cl;
	}
}
```

在 springboot 使用 spring 子容器加载资源的时候，通过设置不同的线程上下文类加载器（加载不同路径的 jar 内容的 自定义类加载器）可以做到隔离

在springboot整合遇到的问题
https://blog.csdn.net/hengyunabc/article/details/79475505

# 参考文档

https://www.geeksforgeeks.org/classloader-in-java/
https://www.jianshu.com/p/aedee0e14319
https://docs.spring.io/spring-boot/docs/current/reference/html/executable-jar.html#executable-jar-jar-file-structure
https://mp.weixin.qq.com/s/yOktvsG8Cj7XBA6PjgX3Xg
《深入理解Java虚拟机》