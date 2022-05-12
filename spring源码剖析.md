# IOC 容器的基本实现

## 1.1 功能分析：

- 首先读取配置文件 `beanFactory.xml`
- 根据 `beanFactoryTest.xml` 中的配置找到对应的类的配置，实例化
- 调用实例化后的实例。

为了达成这个功能，至少需要三个类：

- `ConfigReader`：用于读取及验证配置文件。我们要用配置文件中的东西，首先读取，然后配置在内存中。
- `ReflectionUtil`：根据配置文件中的配置进行反射实例化。
- `APP`：完成整个逻辑的串联。

<img src="C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220421154335256.png" alt="image-20220421154335256" style="zoom:67%;" />

## 1.2 Spring 结构组成

### 1.2.1 核心类介绍

#### 1. `DefaultListableBeanFactory`

综合它的接口，父类的各种功能，主要是对 bean 注册后的处理。

![image-20220421155616088](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220421155616088.png)

`DefaultListableBeanFactory` 数是整个 bean 加载的核心部分，是 Spring 注册及加载 Bean 的默认实现。

它与它的 Xml 实现类不同的地方是 Xml 实现类中使用了自定义的 Xml 读取器 `XmlBeanDefinitionReader`。

- `BeanFactory`：定义获取 Bean 及 Bean 的各种属性
- `BeanDefinitionRegistry`：定义对 `BeanDefinition` 的各种增删改操作。

#### 2. `XmlBeanDefinitionReader`

XML 配置文件读取是 Spring 中的重要功能，Spring 通过 `XmlBeanDefinitionReader` 完成了读取 XML 配置文件的功能。

<img src="C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220421161442405.png" alt="image-20220421161442405" style="zoom:67%;" />

主要流程：

- 通过继承自 `AbstractBeanDefinitionReader` 中的方法，使用 `ResourceLoader` 将资源文件路径转换为对应的 Resource 文件。
- 通过 `DocumentLoader` 对 `Resource` 文件进行转换，将 `Resource` 文件转换为 `Document` 文件。
- 通过实现接口 `BeanDefinitionDocumentReader` 的 `DefaultBeanDefitionDocumentReader` 类对 `Document` 进行解析，并使用 `BeanDefinitionParserDelegate` 对 `Element` 进行解析。

## 1.3 容器的基础 `XmlBeanFactory`

深入分析代码：

```java
BeanFactory bf = new XmlBeanFactory(new ClassPathResouce("beanFactoryTest.xml"));
```

时序图：

![image-20220421162626712](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220421162626712.png)

首先根据类路径返回 `Resource` 对象，根据 `Resource` 对象再由 `XmlBeanFactory` 解析返回 `BeanFactory` 对象

### 1.3.1 配置文件封装

这一步是读取文件然后封装成对应的对象到内存中。

在 Java 中，将不同来源的资源抽象成 URL，通过注册不同的 handler（`URLStreamHandler`）来处理不同来源的资源的读取逻辑。43

一般 handler 类型使用不同前缀来识别，例如：`file:、http:、jar:` 等。

然而 URL 没有默认定义相对 `Classpath` 或 `ServletContext` 等资源的 handler，虽然可以注册自己的 `URLStreamHandler` 来解析特定的 URL 前缀
但是这需要了解 URL 的实现机制，而且 URL 没有提供基本的方法，例如检查当前资源是否存在等方法。

因此 Spring 对其内部使用的资源实现了自己的抽象结构：**Resource** 接口封装底层资源。

> 这里 BB 这么多意思就是 JDK 中的 URL 资源占位符类功能有限，所以 Spring 实现了自己的资源接口：Resource

```java
public interface InputStreamSource {
    InputStream getInputStream() throws IOException;
}

public interface Resource extends InputStreamSource {
    boolean exists();
    boolean isReadable();
    boolean isOpen();
    URL getURL() throws IOException;
    //...
}
```

<img src="C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220421165416639.png" alt="image-20220421165416639" style="zoom:67%;" />

`InputStreamSource` 封装任何能返回 `InputStream` 的类，例如 `File、Classpath` 下的资源和 `Byte、Array` 等。

`Resource` 接口抽象了 Spring 内部使用到的底层资源：`File、URL、Classpath` 等。对于不同来源的资源文件都有相应的 `Resource` 实现：文件`FileSystemResource` 啥的。

日常开发也可以用 Spring 提供的类，例如希望加载文件可以用如下代码：

```java
Resource resource = new ClassPathResource("spring.xml");
InputStream inputStream = resource.getInputStream();
```

> `ClassPathResource.java` 中对 `Resource` 接口的使用：
>
> 通过 class 或者 `classLoader` 提供的底层方法进行调用。
>
> ```java
> if (this.clazz != null) {
>     is = this.clazz.getResourceAsStream(this.path);
> } else {
>     is = this.classLoader.getResourceAsStream(this.path);
> }
> ```

接下来介绍 `XmlBeanFactory` 的初始化

```java
public XmlBeanFactory(Resource resource) throws IOException {
    // 调用 XmlBeanFactory(Resource, BeanFactory)构造方法
    return this(resource, null);
}

// 构造函数内部再次调用内部构造函数：
// parentBeanFactory 为父类 BeanFactory 用于 factory 合并，可以为空
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
    super(parentBeanFactory);
    this.reader.loadBeanDefinitions(resource);  // 加载 Bean
}
```

上面代码中只有 `this.reader.loadBeanDefinitions(resource);` 才是资源加载的真正实现。

但是在 load 之前，还有一个调用父类初始化的动作：`super(parentBeanFactory)`

`XmlBeanFactory` 的父类是 `DefaultListableBeanFactory` 

因此这里的 super 其实调用的父类构造方法

```java
public DefaultListableBeanFactory(@Nullable BeanFactory parentBeanFactory) {
	super(parentBeanFactory);
}
    
public AbstractAutowireCapableBeanFactory(@Nullable BeanFactory parentBeanFactory) {
	this();
	setParentBeanFactory(parentBeanFactory);
}

public AbstractAutowireCapableBeanFactory() {
	super();
	ignoreDependencyInterface(BeanNameAware.class);
	ignoreDependencyInterface(BeanFactoryAware.class);
	ignoreDependencyInterface(BeanClassLoaderAware.class);
}
```

`ignoreDependencyInterface(Class)` 方法：忽略给定接口的自动装配功能。

当 Spring 获取指定 bean 时如果其属性没有初始化，那么 spring 会捎带手顺便将这个属性初始化。
但是某些情况这个属性不会被初始化，其中的一种就是这个属性实现了 `BeanNameAware` 接口，因此这里要忽略这个接口，这样就可以让这个属性顺利初始化了。

### 1.3.2 加载 Bean

在 `XmlBeanFactory` 构造方法中有 `this.reader.loadBeanDefinitions(resource);` 这行代码

这里的 reader 对象是 `XmlBeanDefinitionReader` 类型的

这个方法的时序图：

![image-20220422140704692](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220422140704692.png)

梳理过程：

1. 封装资源文件。当进入 `XmlBeanDefinitionReader` 后首先对参数 `Resource` 使用 `EncodedResource` 类进行封装。
2. 获取输入流。从 Resource 获取对应的 `InputStream` 并构造 `InputSource`
3. 通过构造的 `InputSource` 实例和 `Resource` 实例继续调用函数 `doLoadBeanDefinitions`

源码分析：

从上到下的调用关系：

```java
/**
* XmlBeanFactory 类中:
*/
public XmlBeanFactory(Resource resource) throws BeansException {
    this(resource, (BeanFactory)null);
}

public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
    super(parentBeanFactory);
    this.reader = new XmlBeanDefinitionReader(this);
    this.reader.loadBeanDefinitions(resource);
}

/**
* XmlBeanFactory 类中:
* EncodedResource 类是与资源编码有关的类，可以设置资源的编码，这样 Spring 就可以根据编码来记载资源
*/
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return this.loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Loading XML bean definitions from " + encodedResource);
    }
    
    // 通过属性来记录已经加载的资源
    Set<EncodedResource> currentResources = (Set)this.resourcesCurrentlyBeingLoaded.get();
    if (!currentResources.add(encodedResource)) {
        // 已经加载过了
        throw new BeanDefinitionStoreException("Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    } else {
        // 正式加载
        int var6;
        try {
            // 得到输入流
            InputStream inputStream = encodedResource.getResource().getInputStream();
            Throwable var4 = null;
            try {
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    // 设置编码
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
                var6 = this.doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            } catch (Throwable var24) {
                var4 = var24;
                throw var24;
            } finally {
                // 关闭资源
                if (inputStream != null) {
                    if (var4 != null) {
                        try {
                            inputStream.close();
                        } catch (Throwable var23) {
                            var4.addSuppressed(var23);
                        }
                    } else {
                        inputStream.close();
                    }
                }
            }
        } catch (IOException var26) {
            throw new BeanDefinitionStoreException("IOException parsing XML document from " + encodedResource.getResource(), var26);
        } finally {
            // 移除资源
            currentResources.remove(encodedResource);
            if (currentResources.isEmpty()) {
                this.resourcesCurrentlyBeingLoaded.remove();
            }
        }
        return var6;
    }
}
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) throws BeanDefinitionStoreException {
        try {
            // 加载 XML 文件，获取对应的 Document
            Document doc = this.doLoadDocument(inputSource, resource);  
            // 注册 Bean 信息  请看 1.6
            int count = this.registerBeanDefinitions(doc, resource);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Loaded " + count + " bean definitions from " + resource);
            }

            return count;
        } catch (BeanDefinitionStoreException var5) {
            throw var5;
        } catch (SAXParseException var6) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(), "Line " + var6.getLineNumber() + " in XML document from " + resource + " is invalid", var6);
        } catch (SAXException var7) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(), "XML document from " + resource + " is invalid", var7);
        } catch (ParserConfigurationException var8) {
            throw new BeanDefinitionStoreException(resource.getDescription(), "Parser configuration exception parsing XML from " + resource, var8);
        } catch (IOException var9) {
            throw new BeanDefinitionStoreException(resource.getDescription(), "IOException parsing XML document from " + resource, var9);
        } catch (Throwable var10) {
            throw new BeanDefinitionStoreException(resource.getDescription(), "Unexpected exception parsing XML document from " + resource, var10);
        }
    }
```

其实只做了三件事：

- 获取对 XML 文件的验证模式
- 加载 XML 文件，得到对应的 Document
- 根据返回的 Document 注册 Bean 信息

## 1.4 获取 XML 的验证模式

XML 文件常用验证模式主要有两种：DTD，XSD

### 1.4.1 DTD 与 XSD 区别

**DTD** 即文档类型定义，是一种 XML 约束模式语言，是 XML 文件验证机制，属于 XML 文件的组成的一部分。

一个 DTD 文档包含：元素的定义规则，元素间关系的定义规则，元素可以使用的属性，可使用的实体或符号规则。

要使用 DTD 验证模式需要在 XML 文件的头部声明

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//Spring//DTD BEAN 2.0//EN" "http://www.Springframework.org/dtd/ Spring-beans-2.0.dtd">

<beans>
... ...
</beans>
```

**XML Schema** 语言就是 XSD，描述了 XML 文档的结构和内容，可据此检查 XML 文档是否有效。

例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

... ...
</beans>
```

### 1.4.2 验证模式的读取

Spring 通过读取 XML 文件是否有 DOCTYPE 来判断是哪种验证模式。

## 1.5 获取 Document

验证后就可以加载 Document 对象了。

`XmlBeanFactoryReader` 类对于文档读取委托给了与它相关联的 `DocumentLoader` 去执行，这里真正调用的是 `DefaultDocumentLoader` 执行。

源码：

```java
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
        return this.documentLoader.loadDocument(inputSource, this.getEntityResolver(), this.errorHandler, this.getValidationModeForResource(resource), this.isNamespaceAware());
}


public Document loadDocument(InputSource inputSource, EntityResolver entityResolver, ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
        DocumentBuilderFactory factory = this.createDocumentBuilderFactory(validationMode, namespaceAware);
        if (logger.isTraceEnabled()) {
            logger.trace("Using JAXP provider [" + factory.getClass().getName() + "]");
        }

        DocumentBuilder builder = this.createDocumentBuilder(factory, entityResolver, errorHandler);
        return builder.parse(inputSource);
    }
```

首先创建 `DocumentBuilderFactory`，在通过 `DocumentBuilderFactory` 创建 `DocumentBulder`，进而解析 `inputSource` 来返回 Document对象。

不过要注意这里的 `EntityResolver`，Spring 这里通过 `this.getEntityResolver()` 来获取 `entityResolver`

源码：

```java
protected EntityResolver getEntityResolver() {
    if (this.entityResolver == null) {
        // 选择默认的 EntityResolver 去使用
        ResourceLoader resourceLoader = this.getResourceLoader();
        if (resourceLoader != null) {
            this.entityResolver = new ResourceEntityResolver(resourceLoader);
        } else {
            this.entityResolver = new DelegatingEntityResolver(this.getBeanClassLoader());
        }
    }
    return this.entityResolver;
}
```

### 1.5.1 `EntityResolver` 用法

对于解析一个 XML，应该首先读取 XML 文件的声明，根据声明找对应的 DTD 定义，便于对文档验证。
默认的寻找规则是通过 URI 下载对应的 DTD 声明，进行验证，下载过程首先比较耗时，当发生网络中断或不可用就会报错，这是因为相应的 DTD 声明没有找到。

而 `EntityResolver` 的作用是项目本身就可以提供一个如何寻找 DTD 声明的方法，由程序来实现寻找 DTD 声明的过程。

## 1.6 解析及注册 `BeanDefinitions`

把文件转换为 Document 后，接下来提取和注册 bean 就是关键。

主要是这个方法：

`this.reader.loadBeanDefinitions(resource);` 的调用

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 实例化 BeanDefinitionDocumentReader
    BeanDefinitionDocumentReader documentReader = this.createBeanDefinitionDocumentReader();
    // 记录统计前 BeanDefenition 的加载个数
    int countBefore = this.getRegistry().getBeanDefinitionCount();
    // 将环境变量设置其中
    documentReader.registerBeanDefinitions(doc, this.createReaderContext(resource));
    // 记录本次加载的 BeanDefinition 数量
    return this.getRegistry().getBeanDefinitionCount() - countBefore;
}
```

这个方法应用了 OOP 中面向对象中单一职责的原则，将逻辑处理委托给单一的类进行处理。

```java
// 上面第 7 行
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    // CORE! 
    // 提取出 root 作为参数继续 BeanDefinition 的注册
    this.doRegisterBeanDefinitions(doc.getDocumentElement());
}

protected void doRegisterBeanDefinitions(Element root) {
    // 处理解析    
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = this.createDelegate(this.getReaderContext(), root, parent);
    
    if (this.delegate.isDefaultNamespace(root)) {
        // 处理 profile 属性
        String profileSpec = root.getAttribute("profile");
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, ",; ");
            if (!this.getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec + "] not matching: " + this.getReaderContext().getResource());
                }

                return;
            }
        }
    }
	
    // 解析前处理，留给子类实现
    this.preProcessXml(root);
    this.parseBeanDefinitions(root, this.delegate);
    // 解析后处理，留给子类实现
    this.postProcessXml(root);
    
    this.delegate = parent;
}
```

如果点进前置处理与后置处理会发现这两个方法是空方法

![image-20220422210316017](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220422210316017.png)

这就是典型的模板方法模式，继承自 `DefaultBeanDefinitionDocumentReader` 的子类如果需要在 Bean 解析前后做一些处理的话，那么只需重写这两个方法即可。

### 1.6.1 profile 属性的使用

 例如在 xml 中，我们这样设置：

```xml
<beans profile="dev">
... ...
</beans>

<beans profile="prod">
... ...
</beans>
```

如果集成了 WEB 环境，我们可以在 web.xml 中这样配置：

```xml
<context-param>
    <param-name>Spring.profiles.active</param-name>
    <param-value>dev</param-value>
</context-param>
```

有这个配置我们就可以部署多套配置来适用不同环境了。

### 1.6.2 解析并注册 `BeanDefinition`

方法`parseBeanDefinitions(root, this.delegate)`源码：

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // 对 beans 的处理
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for(int i = 0; i < nl.getLength(); ++i) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element)node;
                if (delegate.isDefaultNamespace(ele)) {
                    this.parseDefaultElement(ele, delegate);
                } else {
                    // 对Bean处理
                    delegate.parseCustomElement(ele);
                }
            }
        }
    } else {
        delegate.parseCustomElement(root);
    }
}
```

在 XML 配置中有两大类 Bean 声明，一个是默认的，如：

```xml
<bean id="test" class="com.test"/>
```

一类是自定义的，如：

```xml
<tx:annotation-driven/>
```

如果是默认命名空间采用 `parseDefaultElement` 方法解析，否则采用 `delegate.parseCustomElement` 方法对自定义命名空间解析

## 1.7 总结

首先根据类路径文件转换为对应的 Resource 文件，这里 Spring 是封装了自己的 URL 对象—— Resource

接着根据 Resource 对象确定其编码，完成一系列细节工作，再加载 bean 信息

主要就是读取 XML 文件，Spring 根据两种不同验证 XML 文件格式：DTD、XSD 读取。

验证结束后即可根据文件获取 Document 对象，根据 Document 对象即可获取 XML 文件中的 bean 节点，然后解析标签（下面介绍）

### 设计模式

主要就是在 `DefaultBeanDefinitionDocumentReader` 默认 Bean 配置信息读取器的 `doRegisterBeanDefinitions` 方法使用了模板设计模式

解析 bean 配置信息前会有前置处理，后会有后置处理，在 `DefaultBeanDefinitionDocumentReader` 它的前置处理和后置处理都是空语句，所以主要让它的子类来实现这两个方法，如果有需要的话。



# 默认标签的解析

上面提过 ==如果是默认命名空间采用 parseDefaultElement 方法解析，否则采用 delegate.parseCustomElement 方法对自定义命名空间解析==

点进 `parseDefaultElement` 方法：

```java
// 对默认命名空间的处理
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, "import")) {
        // 对 import 标签的处理
        this.importBeanDefinitionResource(ele);
    } else if (delegate.nodeNameEquals(ele, "alias")) {
        // 对 alias 标签的处理
        this.processAliasRegistration(ele);
    } else if (delegate.nodeNameEquals(ele, "bean")) {
        // 对 bean 标签的处理
        this.processBeanDefinition(ele, delegate);
    } else if (delegate.nodeNameEquals(ele, "beans")) {
        // 对 beans 标签的处理
        this.doRegisterBeanDefinitions(ele);
    }
}
```

## 1.1 bean 标签的解析及注册

在四种标签中，对 beans 标签的解析是最为复杂也是最为重要的

方法 `DefaultBeanDefinitionDoucumentReader.processBeanDefinition(ele, delegate)` 源码

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	// 让 bdHolder 实例包含配置文件中的属性例如 class、name、id、alias
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
        // 对自定义标签解析
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// 对解析后的 bdHolder 实例注册
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// 发出注册完成响应时间
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

大致逻辑总结如下：

- 首先委托 `BeanDefinitionDelegate` 类的 `parseBeanDefinitionElement` 方法进行元素解析，返回 `BeanDefinitionHolder` 类型的实例。这个方法可以让 `bdHolder` 实例包含我们配置文件中配置的各种属性了，例如 class、name、id、alias
- 当返回的 `bdHolder` 不为空的情况下若存在默认标签的子节点再有自定义属性，还需要再次对自定义标签进行解析。
- 解析完成后需要对解析后的 `bdHolder` 进行注册，同样，注册操作委托给了 `BeanDefinitionReaderUtils.registerBeanDefinition` 方法
- 最后发出响应事件，通知相关的监听器，这个 bean 已经完成加载了。

### 1.1.1 解析 BeanDefinition

`delegate.parseBeanDefinitionElement(ele)` 源码：

```java
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return this.parseBeanDefinitionElement(ele, (BeanDefinition)null);
}

public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    // 解析 id 属性
    String id = ele.getAttribute("id");
    // 解析 name 属性
    String nameAttr = ele.getAttribute("name");
    
    // 分割 name 属性
    List<String> aliases = new ArrayList();
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, ",; ");
        aliases.addAll(Arrays.asList(nameArr));
    }

    String beanName = id;
    if (!StringUtils.hasText(id) && !aliases.isEmpty()) {
        beanName = (String)aliases.remove(0);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("No XML 'id' specified - using '" + beanName + "' as bean name and " + aliases + " as aliases");
        }
    }

    if (containingBean == null) {
        this.checkNameUniqueness(beanName, aliases, ele);
    }
	
    // 解析其他属性
    AbstractBeanDefinition beanDefinition = this.parseBeanDefinitionElement(ele, beanName, containingBean);
    
    if (beanDefinition != null) {
        if (!StringUtils.hasText(beanName)) {
            try {
                // 如果不存在 beanName 那么根据 Spring 中提供的命名规则为当前 bean 生成的 beanName
                if (containingBean != null) {
                    beanName = BeanDefinitionReaderUtils.generateBeanName(beanDefinition, this.readerContext.getRegistry(), true);
                } else {
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null && beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() && !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }

                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("Neither XML 'id' nor 'name' specified - using generated bean name [" + beanName + "]");
                }
            } catch (Exception var9) {
                this.error(var9.getMessage(), ele);
                return null;
            }
        }

        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    } else {
        return null;
    }
}
```

- 提取元素中的 id 以及 name 属性
- 进一步解析其他所有属性并统一封装至 `GenericBeanDefinition` 类型的实例中
- 如果检测到 bean 没有指定 `beanName`，那么使用默认规则为此 Bean 生成 `beanName`
- 将获取到的信息封装到 `BeanDefinitionHolder` 的实例中

上面 32 行的 `AbstractBeanDefinition beanDefinition = this.parseBeanDefinitionElement(ele, beanName, containingBean);` 这个方法源码：

```java
// 解析 bean 的其他属性
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(Element ele, String beanName, @Nullable BeanDefinition containingBean) {
    this.parseState.push(new BeanEntry(beanName));
    
    String className = null;
    // 解析 class 属性
    if (ele.hasAttribute("class")) {
        className = ele.getAttribute("class").trim();
    }
    
    String parent = null;
    // 解析 parent 属性
    if (ele.hasAttribute("parent")) {
        parent = ele.getAttribute("parent");
    }
    
    try {
        // 创建用于承载属性的 AbstractBeanDefinition 类型的 GenericBeanDefinition
        AbstractBeanDefinition bd = this.createBeanDefinition(className, parent);
        
        // 硬编码解析默认 bean 的各种属性
        this.parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        // 提取 description
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, "description"));
        
        // 解析元数据
        this.parseMetaElements(ele, bd);
        
        // 解析 lookup-method 属性
        this.parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        
        // 解析 replaced-method 属性
        this.parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
        
        // 解析构造函数
        this.parseConstructorArgElements(ele, bd);
        // 解析 Property 子元素
        this.parsePropertyElements(ele, bd);
        // 解析 Quealifier 子元素
        this.parseQualifierElements(ele, bd);
        
        bd.setResource(this.readerContext.getResource());
        bd.setSource(this.extractSource(ele));
        return bd;
    } catch (ClassNotFoundException var13) {
        this.error("Bean class [" + className + "] not found", ele, var13);
    } catch (NoClassDefFoundError var14) {
        this.error("Class that bean class [" + className + "] depends on not found", ele, var14);
    } catch (Throwable var15) {
        this.error("Unexpected failure during bean definition parsing", ele, var15);
    } finally {
        this.parseState.pop();
    }
    return null;
}
```

#### 1. 创建用于属性承载的 `BeanDefinition`

`BeanDefinition` 是一个接口，在 Spring 中存在三种实现：`RootBeanDefinition`、`ChildBeanDefinition`、`GenericBeanDefinition`

三种实现都继承了 `AbstractBeanDefinition`，其中 `BeanDefinition` 是配置文件 <bean> 元素标签在容器中的内部表现形式。

<bean> 元素标签拥有 class、scope、lazy-init 等属性，`BeanDefinition` 提供了相应的 `beanClass、scope、lazyInit` 属性。

`BeanDefinition` 与 <bean> 中的属性是一一对应的。而其中 `RootBeanDefinition` 是最常用的实现类，它对应一般性的 <bean> 元素标签。

在配置文件中可以定义父<bean> 和子<bean>，父<bean> 用 `RootBeanDefinition` 表示，子<bean> 用 `ChildBeanDefinition` 表示，
`AbstractBeanDefinition` 对两者共同的类信息进行抽象。

Spring 通过 `BeanDefinition` 将配置文件中的 <bean> 转换为容器的内部表示，并将 `BeanDefinition` 注册到 `BeanDefinitionRegistry` 中。

Spring 的 `BeanDefinitionRegistry` 就像 Spring 的内存数据库，主要以 map 形式保存，后续操作直接从 `BeanDefinitionRegistry` 中读取配置信息。

要解析属性首先要创建用于承载属性的实例，也就是创建 `GenericBeanDefinition` 类型的实例。

而上面源码的 20 行 `AbstractBeanDefinition bd = this.createBeanDefinition(className, parent);` 就是创建 `GenericBeanDefinition` 类型的实例。

源码：

```java
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName) throws ClassNotFoundException {
    return BeanDefinitionReaderUtils.createBeanDefinition(parentName, className, this.readerContext.getBeanClassLoader());
}

public static AbstractBeanDefinition createBeanDefinition(@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFou
    // 创建 GenericBeanDefinition
    GenericBeanDefinition bd = new GenericBeanDefinition();
    bd.setParentName(parentName);
    if (className != null) {
        if (classLoader != null) {
            // 如果类加载器不为空，则用传入的类加载器统一虚拟机加载类的对象，否则只是记录 className
            bd.setBeanClass(ClassUtils.forName(className, classLoader));
        } else {
            bd.setBeanClassName(className);
        }
    }
    return bd;
}
```

#### 2. 解析各种属性

当我们创建了 bean 信息的承载实例后，便可进行 bean 信息的解析了。

下面是 1.1.1 中的第 23 行 `this.parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);` 源码：

```java
public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName, @Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
    
    // 解析 scope 属性
    if (ele.hasAttribute("singleton")) {
        // 如果这个属性有 singleton 属性则抛异常
        this.error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
    } else if (ele.hasAttribute("scope")) {
        bd.setScope(ele.getAttribute("scope"));
    } else if (containingBean != null) {
        bd.setScope(containingBean.getScope());
    }
    
    // 解析 abstract 属性
    if (ele.hasAttribute("abstract")) {
        bd.setAbstract("true".equals(ele.getAttribute("abstract")));
    }
    
    // 解析 lazy-init 属性
    String lazyInit = ele.getAttribute("lazy-init");
    if (this.isDefaultValue(lazyInit)) {
        lazyInit = this.defaults.getLazyInit();
    }
    bd.setLazyInit("true".equals(lazyInit));
    
    // 解析 autowire 属性
    String autowire = ele.getAttribute("autowire");
    bd.setAutowireMode(this.getAutowireMode(autowire));
    String autowireCandidate;
    
    // 解析 depends-on 属性
    if (ele.hasAttribute("depends-on")) {
        autowireCandidate = ele.getAttribute("depends-on");
        bd.setDependsOn(StringUtils.tokenizeToStringArray(autowireCandidate, ",; "));
    }
    autowireCandidate = ele.getAttribute("autowire-candidate");
    
    // 解析 destroyMethodName 属性
    String destroyMethodName;
    if (this.isDefaultValue(autowireCandidate)) {
        destroyMethodName = this.defaults.getAutowireCandidates();
        if (destroyMethodName != null) {
            String[] patterns = StringUtils.commaDelimitedListToStringArray(destroyMethodName);
            bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
        }
    } else {
        bd.setAutowireCandidate("true".equals(autowireCandidate));
    }
    
    // 解析 primary 属性
    if (ele.hasAttribute("primary")) {
        bd.setPrimary("true".equals(ele.getAttribute("primary")));
    }
    if (ele.hasAttribute("init-method")) {
        destroyMethodName = ele.getAttribute("init-method");
        bd.setInitMethodName(destroyMethodName);
    } else if (this.defaults.getInitMethod() != null) {
        bd.setInitMethodName(this.defaults.getInitMethod());
        bd.setEnforceInitMethod(false);
    }
    if (ele.hasAttribute("destroy-method")) {
        destroyMethodName = ele.getAttribute("destroy-method");
        bd.setDestroyMethodName(destroyMethodName);
    } else if (this.defaults.getDestroyMethod() != null) {
        bd.setDestroyMethodName(this.defaults.getDestroyMethod());
        bd.setEnforceDestroyMethod(false);
    }
    if (ele.hasAttribute("factory-method")) {
        bd.setFactoryMethodName(ele.getAttribute("factory-method"));
    }
    if (ele.hasAttribute("factory-bean")) {
        bd.setFactoryBeanName(ele.getAttribute("factory-bean"));
    }
    return bd;
}
```

#### 3. 解析子元素 meta

meta 属性的使用

```xml
<bean id="testBean" class="com.TestBean">
    <meta key="testStr" value="abc" />
</bean>
```

当我们要使用里面信息时可以用 `BeanDefinition` 的 `getAttribute(key)` 方法进行获取

`parseMetaElements(ele, bd);` 源码：

```java
public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
	// 获取当前节点的所有元素
    NodeList nl = ele.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		// 提取 meta
        if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
			Element metaElement = (Element) node;
			String key = metaElement.getAttribute(KEY_ATTRIBUTE);
			String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
			// 使用 key、value 构造 BeanMetadataAttribute
            BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
			attribute.setSource(extractSource(metaElement));
			// 记录信息
            attributeAccessor.addMetadataAttribute(attribute);
		}
	}
}
```

#### 4. 解析子元素 `lookup-method`

这个元素不是很常用，我们称它为获取器注入。

获取器注入是一种特殊的方法注入，将一个方法声明为返回某种类型的 bean，但实际要返回的 bean 是在配置文件中配置的，此方法可用在设计有插拔的功能上，解除程序依赖。

我们首先创建一个父类 User：

```java
public class User {
    public void showMe() {
        System.out.println("User");
    }
}
```

在创建一个子类：

```java
public class Teacher extends User{
    public void showMe() {
        System.out.println("Teacher");
    }
}
```

创建调用方法：

```java
public abstract class GetBeanTest {
    public void showMe() {
        this.getBean().showMe();
    }

    public abstract User getBean();
}
```

bean 配置：

```xml
    <bean id="getBeanTest" class="com.bean.GetBeanTest">
        <lookup-method name="getBean" bean="teacher" />
    </bean>

    <bean id="teacher" class="com.bean.Teacher" />
```

测试类：

```java
@Test
public void testLookUpMethod() {
    ApplicationContext bf = new ClassPathXmlApplicationContext("bean1.xml");
    GetBeanTest test = (GetBeanTest) bf.getBean("getBeanTest");
    test.showMe();
}
```

运行结果：

````
Teacher
````

这个配置完成的功能是动态将 teacher 所代表的 bean 作为 `getBean` 的返回值

业务变更，要求进行替换，则我们需要新增类和修改配置文件即可。

源码：

```java
public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
    NodeList nl = beanEle.getChildNodes();
    for(int i = 0; i < nl.getLength(); ++i) {
        Node node = nl.item(i);
        // 仅当在 Spring 默认 bean 的子元素下且为 lookup-method 时有效
        if (this.isCandidateElement(node) && this.nodeNameEquals(node, "lookup-method")) {
            Element ele = (Element)node;
            String methodName = ele.getAttribute("name");
            String beanRef = ele.getAttribute("bean");
            LookupOverride override = new LookupOverride(methodName, beanRef);
            override.setSource(this.extractSource(ele));
            overrides.addOverride(override);
        }
    }
}
```

#### 5. 解析子元素 `replaced-method`

这个方法主要是对 bean 中 `replaced-method` 子元素的提取，先来介绍一下用法：

- 方法替换：可以在运行时用新的方法替换现有的方法。它与 look-up不同的是，不但可以动态替换返回实体 bean，还可以动态更改原有方法的逻辑。

创建原来的类：

```java
public class TestChangeMethod {
    public void changeMe() {
        System.out.println("changeMe");
    }
}
```

需要改变业务逻辑：

```java
public class TestMethodReplacer implements MethodReplacer {
    @Override
    public Object reimplement(Object o, Method method, Object[] objects) throws Throwable {
        System.out.println("我替换了原有的方法");
        return null;
    }
}
```

配置：

```xml
<bean id="testChangeMethod" class="com.bean.TestChangeMethod">
    <replaced-method name="changeMe" replacer="replacer" />
</bean>

<bean id="replacer" class="com.bean.TestMethodReplacer"/>
```

主方法：

```java
@Test
public void testReplacedMethod() {
    ApplicationContext bf = new ClassPathXmlApplicationContext("bean1.xml");
    TestChangeMethod test = (TestChangeMethod) bf.getBean("testChangeMethod");
    test.changeMe();
}
```

运行结果：

```
我替换了原有的方法
```

因此当我们业务变更需要替换实现类时，我们就可以新增逻辑类并让他实现接口 `MethodReplacer`，再新增配置，就可以在不改动代码的情况下动态修改逻辑了。

源码：

```java
public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
    NodeList nl = beanEle.getChildNodes();
    for(int i = 0; i < nl.getLength(); ++i) {
        Node node = nl.item(i);
        if (this.isCandidateElement(node) && this.nodeNameEquals(node, "replaced-method")) {
            Element replacedMethodEle = (Element)node;
            
            // 提取要更换的旧方法
            String name = replacedMethodEle.getAttribute("name");
            
            // 提取对应新的替换方法
            String callback = replacedMethodEle.getAttribute("replacer");
            ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
            List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, "arg-type");
            Iterator var11 = argTypeEles.iterator();
            while(var11.hasNext()) {
                Element argTypeEle = (Element)var11.next();
                // 记录参数
                String match = argTypeEle.getAttribute("match");
                match = StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle);
                if (StringUtils.hasText(match)) {
                    replaceOverride.addTypeIdentifier(match);
                }
            }
            replaceOverride.setSource(this.extractSource(replacedMethodEle));
            overrides.addOverride(replaceOverride);
        }
    }
}
```

#### 6. 解析子元素 `constructor-arg`

例如我们在 bean.xml 中配置：

```xml
<bean id="helloBean" class="com.bean.HelloBean">
    <constructor-arg index="0">
        <value>
            13
        </value>
    </constructor-arg>
    <constructor-arg index="1">
        <value>
            你好
        </value>
    </constructor-arg>
</bean>
```

那么 Spring 会自动为我们识别 value 的类型自动装配到 bean 中，大大简化了我们的开发。

源码：

```java
public void parseConstructorArgElements(Element beanEle, BeanDefinition bd) {
    NodeList nl = beanEle.getChildNodes();
    for(int i = 0; i < nl.getLength(); ++i) {
        Node node = nl.item(i);
        if (this.isCandidateElement(node) && this.nodeNameEquals(node, "constructor-arg")) {
            // 解析 constructor-arg
            this.parseConstructorArgElement((Element)node, bd);
        }
    }
}
```

遍历所有子元素，提取所有的 `constructor-arg` 然后解析，具体解析放置到重载的 `parseConstructorArgElement` 方法中

源码：

```java
public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
    // 提取 index 属性
    String indexAttr = ele.getAttribute("index");
    // 提取 type 属性
    String typeAttr = ele.getAttribute("type");
    // 提取 name 属性
    String nameAttr = ele.getAttribute("name");
    if (StringUtils.hasLength(indexAttr)) {
        try {
            int index = Integer.parseInt(indexAttr);  // 将 index 转为整数
            if (index < 0) {
                // 小于 0 报错
                this.error("'index' cannot be lower than 0", ele);
            } else {
                try {
                    this.parseState.push(new ConstructorArgumentEntry(index));
                    // 解析 ele 对应的属性元素
                    Object value = this.parsePropertyValue(ele, bd, (String)null);
                    ValueHolder valueHolder = new ValueHolder(value);
                    if (StringUtils.hasLength(typeAttr)) {
                        valueHolder.setType(typeAttr);
                    }
                    if (StringUtils.hasLength(nameAttr)) {
                        valueHolder.setName(nameAttr);
                    }
                    valueHolder.setSource(this.extractSource(ele));
                    // 不允许指定相同参数
                    if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
                        this.error("Ambiguous constructor-arg entries for index " + index, ele);
                    } else {
                        bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
                    }
                } finally {
                    this.parseState.pop();
                }
            }
        } catch (NumberFormatException var19) {
            this.error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
        }
    } else {
        // 没有 index 属性则忽略，自动寻找
        try {
            this.parseState.push(new ConstructorArgumentEntry());
            Object value = this.parsePropertyValue(ele, bd, (String)null);
            ValueHolder valueHolder = new ValueHolder(value);
            if (StringUtils.hasLength(typeAttr)) {
                valueHolder.setType(typeAttr);
            }
            if (StringUtils.hasLength(nameAttr)) {
                valueHolder.setName(nameAttr);
            }
            valueHolder.setSource(this.extractSource(ele));
            bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
        } finally {
            this.parseState.pop();
        }
    }
}
```

- 如果配置中指定 index 属性，操作步骤如下：
  1. 解析 `Constructor-arg` 的子元素
  2. 使用 `ConstructorArgumentValues.ValueHolder` 类型来封装解析出来的元素
  3. 将 type、name 和 index 属性一起封装到 `ConstructorArgumentValues.ValueHolder` 中，添加到当前 `BeanDefinition` 的 `constructorArgumentValues` 的 ==**`indexedArgumentValues`**== 属性中

- 如果没有指定 index 属性
  1. 解析 `Constructor-arg` 的子元素
  2. 使用 `ConstructorArgumentValues.ValueHolder` 封装解析出来的元素
  3. 将 type、name 和 index 属性一起封装到 `ConstructorArgumentValues.ValueHolder` 中，添加到当前 `BeanDefinition` 的 `constructorArgumentValues` 的 ==**`genericArgumentValue`**== 属性中

对于是否有 index 元素，Spring 的处理过程是不同的，进入 `parsePropertyValue`（上面的 18 行或者 44 行）

代码：

> 这里的 `NodeList` 就相当于一个 `constructor-arg` 的元素节点，例如 <value> 啥的
>
> 这里的 Element 就相当于一个 `constructor-arg` 标签

```java
public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
    String elementName = propertyName != null ? "<property> element for property '" + propertyName + "'" : "<constructor-arg> element";
    
    // 一个属性只能对应一种类型：ref、value、list 等
    NodeList nl = ele.getChildNodes();
    Element subElement = null;
    for(int i = 0; i < nl.getLength(); ++i) {
        Node node = nl.item(i);
        // 对应 description 或者 meta 不处理
        if (node instanceof Element && !this.nodeNameEquals(node, "description") && !this.nodeNameEquals(node, "meta")) {
            if (subElement != null) {
                this.error(elementName + " must not contain more than one sub-element", ele);
            } else {
                subElement = (Element)node;
            }
        }
    }
    // 有 ref 标签
    boolean hasRefAttribute = ele.hasAttribute("ref");
    // 有 value 标签
    boolean hasValueAttribute = ele.hasAttribute("value");
    if (hasRefAttribute && hasValueAttribute || (hasRefAttribute || hasValueAttribute) && subElement != null) {
        /**
        * 在 constructor-arg 上不存在：
        	既有 ref 属性又有 value 属性
        	存在 ref 属性或者 value 属性且又有子元素
        */
        this.error(elementName + " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
    }
    if (hasRefAttribute) {
        // 解析得到 ref 标签
        String refName = ele.getAttribute("ref");
        if (!StringUtils.hasText(refName)) {
            this.error(elementName + " contains empty 'ref' attribute", ele);
        }
        // ref 属性的处理，使用 RuntimeBeanReference 封装对应的 ref 名称
        RuntimeBeanReference ref = new RuntimeBeanReference(refName);
        ref.setSource(this.extractSource(ele));
        return ref;
    } else if (hasValueAttribute) {
        // value 的处理，使用 TypedStringValue 封装
        TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute("value"));
        valueHolder.setSource(this.extractSource(ele));
        return valueHolder;
    } else if (subElement != null) {
        // 解析子元素
        return this.parsePropertySubElement(subElement, bd);
    } else {
        // 既没有 ref 也没有 value 元素也没有子元素,不符合规范
        this.error(elementName + " must specify a ref or value", ele);
        return null;
    }
}
```

对 `constructor-age` 中属性元素的解析，经历以下几个过程：

1. 略过 description 或者 meta

2. 提取 ref 或者 value 属性，便于根据规则验证正确性。

3. ref 属性的处理。使用 `RuntimeBeanReference` 封装对应的 ref 名称。

4. value 属性的处理。使用 `TypedStringValue` 封装。

5. 子元素处理，例如：

   ```xml
   <constructor-arg>
   	<map>
           <!-- 注入 map 元素 -->
       	<entry key="key" value="value" />
       </map>
   </constructor-arg>
   ```

#### 7. 解析子元素 property

使用如下：

```xml
<bean id="test" class="com.learn.TestClass">
	<property name="testStr" value="123"></property>
</bean>
```

这里使用 `parsePropertyElements` 方法来解析

代码：

```java
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
    NodeList nl = beanEle.getChildNodes();
    for(int i = 0; i < nl.getLength(); ++i) {
        Node node = nl.item(i);
        if (this.isCandidateElement(node) && this.nodeNameEquals(node, "property")) {
            // CORE
            this.parsePropertyElement((Element)node, bd);
        }
    }
}
```

提取所有的 `property` 的子元素，然后解析：

源码：

```java
public void parsePropertyElement(Element ele, BeanDefinition bd) {
    // 成员变量名
    String propertyName = ele.getAttribute("name");
    if (!StringUtils.hasLength(propertyName)) {
        this.error("Tag 'property' must have a 'name' attribute", ele);
    } else {
        this.parseState.push(new PropertyEntry(propertyName));
        try {
            // 如果对同一属性多次赋值则报错
            if (bd.getPropertyValues().contains(propertyName)) {
                this.error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
                return;
            }
            Object val = this.parsePropertyValue(ele, bd, propertyName);
            PropertyValue pv = new PropertyValue(propertyName, val);
            this.parseMetaElements(ele, pv);
            pv.setSource(this.extractSource(ele));
            bd.getPropertyValues().addPropertyValue(pv);
        } finally {
            this.parseState.pop();
        }
    }
}
```

### 1.1.2 `AbstractBeanDefinition` 属性

在 1.1.1 的 1 中提到 3 种不同的 `BeanDefinition` 的定义，而其都继承了 `AbstractBeanDefinition` 抽象类，这个类就是 `BeanDefinition` 的规范，我们可以在这个类中找到所有与 <bean> 配置相关的属性，这里也用到了模板方法模式

源码：

```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor implements BeanDefinition, Cloneable {
    // 省去常量
    @Nullable
    private String scope;
    private boolean abstractFlag;
    @Nullable
    private Boolean lazyInit;
    private int autowireMode;
    private int dependencyCheck;
    @Nullable
    private String[] dependsOn;
    private boolean autowireCandidate;
    private boolean primary;
    private final Map<String, AutowireCandidateQualifier> qualifiers;
    @Nullable
    private Supplier<?> instanceSupplier;
    private boolean nonPublicAccessAllowed;
    private boolean lenientConstructorResolution;
    @Nullable
    private String factoryBeanName;
    @Nullable
    private String factoryMethodName;
    @Nullable
    private ConstructorArgumentValues constructorArgumentValues;
    @Nullable
    private MutablePropertyValues propertyValues;
    private MethodOverrides methodOverrides;
    @Nullable
    private String initMethodName;
    @Nullable
    private String destroyMethodName;
    private boolean enforceInitMethod;
    private boolean enforceDestroyMethod;
    private boolean synthetic;
    private int role;
    @Nullable
    private String description;
    @Nullable
    private Resource resource;
	// 省去 get/set/构造器 方法
}
```

###  1.1.3 解析默认标签中的自定义标签元素

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 解析 <bean> 标签
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 解析默认标签中的自定义元素
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 注册 beanDefinition
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, this.getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException var5) {
            this.getReaderContext().error("Failed to register bean definition with name '" + bdHolder.getBeanName() + "'", ele, var5);
        }
        this.getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

之前一直在说默认标签的解析与提取过程，也就是第 2 行：`delegate.parseBeanDefinitionElement(ele)`

接下来进行 ` bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);`（第 4 行）的分析，这句代码的作用是**如果需要的话就对 `beanDefinition` 进行装饰**。

这句代码适用于这样的场景：

```xml
<bean>
	<mybean:user userName="aaa">
</bean>
```

当 Spring 中的 bean 是使用默认的标签配置，但其中的子元素却使用了自定义的配置，这句代码便起了作用。

这个方法源码：

```java
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef) {
    return this.decorateBeanDefinitionIfRequired(ele, originalDef, (BeanDefinition)null);
}
```

第三个参数设置为空，第三个参数其实是**父类 bean**，当对某个嵌套配置进行分析时，这是需要传递父类 `beanDefinition`。

跟踪函数：

```java
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {
    
    BeanDefinitionHolder finalDefinition = originalDef;
    
    NamedNodeMap attributes = ele.getAttributes();
    
    // 遍历所有的属性，看看是否有适用于修饰的属性
    for(int i = 0; i < attributes.getLength(); ++i) {
        Node node = attributes.item(i);
        finalDefinition = this.decorateIfRequired(node, finalDefinition, containingBd);
    }
    NodeList children = ele.getChildNodes();
    
    // 遍历所有的子节点，看看是否有适用于修饰的子元素
    for(int i = 0; i < children.getLength(); ++i) {
        Node node = children.item(i);
        if (node.getNodeType() == 1) {
            finalDefinition = this.decorateIfRequired(node, finalDefinition, containingBd);
        }
    }
    return finalDefinition;
}
```

查看 `decorateIfRequired` 方法：

```java
public BeanDefinitionHolder decorateIfRequired(Node node, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {
    
    // 获取自定义标签的命名空间
    String namespaceUri = this.getNamespaceURI(node);
    // 对于非默认标签进行修饰
    if (namespaceUri != null && !this.isDefaultNamespace(namespaceUri)) {
        // 根据命名空间找到对应的处理器
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler != null) {
            // 进行修饰
            BeanDefinitionHolder decorated = handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
            if (decorated != null) {
                return decorated;
            }
        } else if (namespaceUri.startsWith("http://www.springframework.org/schema/")) {
            this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
        } else if (this.logger.isDebugEnabled()) {
            this.logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
        }
    }
    return originalDef;
}
```

- 首先获取属性或者元素的命名空间，以此来判断该元素或者属性是否适用于自定义标签的解析条件
- 找出对应的处理器进一步解析，如果可以解析则修饰并返回，否则抛异常。

### 1.1.4 注册解析的 `BeanDefinition`

对于配置文件，解析装饰完成后，对于得到的 `beanDefinition` 可以满足后续的使用要求了，唯一剩下的就是注册了。

也就是`BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, this.getReaderContext().getRegistry());`

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 解析 <bean> 标签
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 解析默认标签中的自定义元素
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 注册 beanDefinition
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, this.getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException var5) {
            this.getReaderContext().error("Failed to register bean definition with name '" + bdHolder.getBeanName() + "'", ele, var5);
        }
        this.getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

源码：

```java
// 注册 beanDefinition
public static void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) throws BeanDefinitionStoreException {
    String beanName = definitionHolder.getBeanName();  // 获取 beanName
    // 使用 beanName 做唯一标识注册
    // 注册 beanDefinition
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
    
    // 注册所有别名
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        String[] var4 = aliases;
        int length = aliases.length;
        for(int i = 0; i < length; ++i) {
            String alias = var4[i];
            registry.registerAlias(beanName, alias);
        }
    }
}
```

解析的 `beanDefinition` 都会注册到 `BeanDefinitionRegistry` 类型的实例 `registry` 中，对于 `beanDefinition` 的注册分为两部分：通过 `beanName` 的注册以及通过别名的注册。

#### 1. 通过 `beanName` 注册 `BeanDefinition`

简单来说就是将 `beanDefinition` 直接放入 `map` 中就好了，使用 `beanName` 作为 `key`。不过除此之外，Spring 还做了点别的事情：

```java
// 第 6 行 registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition()); 源码
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {
    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");
    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            // 注册前的最后一次校验，不同于 XML 校验
            // 主要是对 AbstractBeanDefinition 中的 methodOverrides 校验
            // 校验 methodOverrides 是否与工厂方法并存或者 methodOverrides 对应的方法根本不存在
            ((AbstractBeanDefinition)beanDefinition).validate();
        } catch (BeanDefinitionValidationException var8) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName, "Validation of bean definition failed", var8);
        }
    }
    // 之前存在的 beanDefinition
    BeanDefinition existingDefinition = (BeanDefinition)this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        if (!this.isAllowBeanDefinitionOverriding()) {
            // 不允许覆盖操作
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }
        if (existingDefinition.getRole() < beanDefinition.getRole()) {
            if (this.logger.isInfoEnabled()) {
                this.logger.info("Overriding user-defined bean definition for bean '" + beanName + "' with a framework-generated bean definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
            }
        } else if (!beanDefinition.equals(existingDefinition)) {
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Overriding bean definition for bean '" + beanName + "' with a different definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
            }
        } else if (this.logger.isTraceEnabled()) {
            this.logger.trace("Overriding bean definition for bean '" + beanName + "' with an equivalent definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
        }
        // 更新操作，实现类是 ConcurrentHashMap
        this.beanDefinitionMap.put(beanName, beanDefinition);
    } else {
        // 第一次注册 beanDefinition
        if (this.hasBeanCreationStarted()) {
            // 因为 beanDefinitionMap 是全局变量，这里会存在并发访问情况
            synchronized(this.beanDefinitionMap) {
                // 新增操作
                this.beanDefinitionMap.put(beanName, beanDefinition);
                // COW 写时复制技术，新增的时候需要加锁，读的时候不需要锁
                // 存储更新的 Definitions 集合，根据 beanName 查询
                List<String> updatedDefinitions = new ArrayList(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                this.removeManualSingletonName(beanName);
            }
        } else {
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            this.removeManualSingletonName(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }
    if (existingDefinition == null && !this.containsSingleton(beanName)) {
        if (this.isConfigurationFrozen()) {
            this.clearByTypeCache();
        }
    } else {
        this.resetBeanDefinition(beanName);
    }
}
```

- 对 `AbstractBeanDefinitions` 校验。
- 对 `beanName` 已经注册的情况的处理，如果设置了不允许 bean 覆盖，抛出异常，否则直接覆盖。
- 加入 map 缓存
- 重置 `beanName` 缓存

#### 2. 通过别名注册 `BeanDefinition`

源码：

```java
// aliasMap 是 <String,String> 类型的 ConcurrentHashMap 集合 alias -> beanName
public void registerAlias(String name, String alias) {
    Assert.hasText(name, "'name' must not be empty");
    Assert.hasText(alias, "'alias' must not be empty");
    // 共享变量，需要加锁
    synchronized(this.aliasMap) {
        // 如果 beanName 与 alias 相同的话不记录 alias，并删除对应的 alias
        if (alias.equals(name)) {
            this.aliasMap.remove(alias);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
            }
        } else {
            // 得到
            String registeredName = (String)this.aliasMap.get(alias);
            if (registeredName != null) {
                // 如果这个别名之前注册过
                if (registeredName.equals(name)) {
                    // 内容相等，直接返回即可
                    return;
                }
                if (!this.allowAliasOverriding()) {
                    // 不允许覆盖，抛异常
                    throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" + name + "': It is already registered for name '" + registeredName + "'.");
                }
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Overriding alias '" + alias + "' definition for registered name '" + registeredName + "' with new target name '" + name + "'");
                }
            }
            // alias 循环检查
            this.checkForAliasCircle(name, alias);
            // 更新别名
            this.aliasMap.put(alias, name);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
            }
        }
    }
}
```

- alias 与 `beanName` 相同情况处理。如果相同就不需要处理，并删除原有的 alias。
- alias 覆盖处理。
- alias 循环检查。当 A -> B 存在时，若再次出现 A -> C -> B 就会抛出异常。
- 注册 alias

### 1.1.5 通知监听器解析及注册完成

`this.getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));`

这里的实现只为口占，当程序员需要对注册 `beanDefinition` 事件监听是，可以通过注册监听器的方式并将处理逻辑写入监听器中，目前 Spring 没有对此时间做任何逻辑处理。

<img src="C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220501232639331.png" alt="image-20220501232639331" style="zoom: 50%;" />

都是空方法。

## 1.2 alias 标签的解析

在对 bean 标签定义时，除了使用 id 来指定名称，为了提供多个名称，可以使用 alias 标签来指定，所有的这些名称都指向同一个 bean。

在 XML 中我们可以用如下方式添加别名：

```xml
<bean id="testBean" name="testBean1,testBean2" class="com.test" />

<bean id="testBean" class="com.test" />
<alias name="testBean" alias="testBean1,testBean2">
```

在 `DefaultBeanDefinitionDocumentReader.parseDefaultElement()` 方法中

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		importBeanDefinitionResource(ele);
	}
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		processAliasRegistration(ele);
	}
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
		processBeanDefinition(ele, delegate);
	}
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		// recurse
		doRegisterBeanDefinitions(ele);
	}
}
```

`processAliasRegistration(ele);` 源码：

```java
protected void processAliasRegistration(Element ele) {
	// 获取 beanName
    String name = ele.getAttribute(NAME_ATTRIBUTE);
	// 获取 alias
    String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
	boolean valid = true;
	if (!StringUtils.hasText(name)) {
		getReaderContext().error("Name must not be empty", ele);
		valid = false;
	}
	if (!StringUtils.hasText(alias)) {
		getReaderContext().error("Alias must not be empty", ele);
		valid = false;
	}
	if (valid) {
		try {
			getReaderContext().getRegistry().registerAlias(name, alias);
		}
		catch (Exception ex) {
			getReaderContext().error("Failed to register alias '" + alias +
					"' for bean with name '" + name + "'", ele, ex);
		}
		getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
	}
}
```

和之前的 bean alias 解析大同小异，都是将别名和 `beanName` 一起注册到 registry 中。

- alias 与 `beanName` 相同情况处理。如果相同就不需要处理，并删除原有的 alias。
- alias 覆盖处理。
- alias 循环检查。当 A -> B 存在时，若再次出现 A -> C -> B 就会抛出异常。
- 注册 alias

## 1.3 import 标签解析

对于一个庞大的项目，我们需要将其配置文件切分成不同的小块，来提高可读性，使用 import 是个好办法

```xml
<import resource="customerContext.xml" />
<import resource="systemContext.xml" />
```

Spring 是如何解析 import 标签的呢：

```java
protected void importBeanDefinitionResource(Element ele) {
	// 获取 resource 属性
    String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
	// 如果不存在 resource 属性则不作任何处理
    if (!StringUtils.hasText(location)) {
		getReaderContext().error("Resource location must not be empty", ele);
		return;
	}
	// 解析系统属性: 例如 "${user.dir}"
	location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);
	Set<Resource> actualResources = new LinkedHashSet<>(4);
	boolean absoluteLocation = false;
	try {
        // 判定 location 是决定 URI 还是 相对 URI
		absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
	}
	catch (URISyntaxException ex) {
		// cannot convert to an URI, considering the location relative
		// unless it is the well-known Spring prefix "classpath*:"
	}
	// Absolute or relative?
	if (absoluteLocation) {
        // 绝对 Location 直接根据地址加载对应的配置文件
		try {
			int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
			if (logger.isDebugEnabled()) {
				logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
			}
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error(
					"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
		}
	}
	else {
		// 相对地址根据相对地址计算出绝对地址
		try {
			int importCount;
			// Resource 存在多个子实现类，如 VfsResource、FileSystemResource 等
            // 而每个 resource 的 createRelative 方式实现都不同，所以这里先使用子类的方式尝试解析
            Resource relativeResource = getReaderContext().getResource().createRelative(location);
			if (relativeResource.exists()) {
                // 解析成功
				importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
				actualResources.add(relativeResource);
			}
			else {
                // 解析失败
                // 使用默认的解析器 ResourcePattrenResolver 解析 
				String baseLocation = getReaderContext().getResource().getURL().toString();
				importCount = getReaderContext().getReader().loadBeanDefinitions(
						StringUtils.applyRelativePath(baseLocation, location), actualResources);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
			}
		}
		catch (IOException ex) {
			getReaderContext().error("Failed to resolve current resource location", ele, ex);
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
					ele, ex);
		}
	}
    // 解析后启动监听器激活处理
	Resource[] actResArray = actualResources.toArray(new Resource[0]);
	getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```

1. 获取 resource 属性所表示的路径
2. 解析路径中的系统属性
3. 判定是绝对路径还是相对路径
4. 绝对路径则递归调用 bean 的解析过程，进行另一次解析
5. 相对路径则计算出绝对路径进行 4 
6. 通知监听器，解析完成

# 自定义标签的解析

在 `DefaultBeanDefinitionDocumentReader` 的 `parseBeanDefinitions`

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for(int i = 0; i < nl.getLength(); ++i) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element)node;
                if (delegate.isDefaultNamespace(ele)) {
                    // 解析默认标签
                    this.parseDefaultElement(ele, delegate);
                } else {
                    // 解析用户自定义标签
                    delegate.parseCustomElement(ele);
                }
            }
        }
    } else {
        // 解析用户自定义标签
        delegate.parseCustomElement(root);
    }
}
```

当 Spring 拿到一个元素时首先根据命名空间解析。如果是默认的命名空间，则使用 `parseDefaultElement` 方法进行元素解析，
否则使用 `parseCustomElement` 方法进行解析。

之前已经撕完了解析默认标签的源码。

那么解析用户自定义标签的逻辑又是怎样的呢。

## 4.1 自定义标签的使用

- 创建一个需要扩展的组件
- 定义一个 XSD 文件描述组件内容
- 创建一个 java类，实现 `BeanDefinitionParser` 接口，用来解析 XSD 文件中的定义和组件的定义
- 创建一个 Handler 类，扩展自 `NamespaceHandlerSupport`，目的是将组件注册到 Spring 容器。
- 编写 `Spring.handlers` 和 `Spring.schemas` 文件。

1. 首先创建一个普通的 POJO，这个 POJO没有什么特别处，只是用来接收配置文件。

   ```java
   public class Student {
       private String name;
       private String email;
   	// get 和 set 方法省略
   }
   ```

2. 定义一个 XSD 文件描述组件内容（创建在 `src` 下的 META-INF 中）

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <schema xmlns="http://www.w3.org/2001/XMLSchema"
           targetNamespace="http://www.lexueba.com/schema/student" 
           xmlns:tns="http://www.lexueba.com/schema/student"
           elementFormDefault="qualified"
   >
       
   <element name="student" >
       <complexType>
           <attribute name="id" type="string"/>
           <attribute name="name" type="string"/>
           <attribute name="email" type="string"/>
       </complexType>
   </element>    
   
   </schema>
   ```
   XSD 文件是 XML、DTD 的替代者。

3. 创建一个类，实现 `BeanDefinitionParser` 接口，用来解析 XSD 文件中的定义和组件定义

   ```java
   import com.spring5.Student;
   import org.springframework.beans.factory.support.BeanDefinitionBuilder;
   import org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser;
   import org.springframework.util.StringUtils;
   import org.w3c.dom.Element;
   
   public class StudentBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {
       // Element 对应的类
       @Override
       protected Class getBeanClass(Element element) {
           return Student.class;
       }
       
       // 从 element 解析并提取对应的元素
       @Override
       protected void doParse(Element element, BeanDefinitionBuilder bean) {
           String name = element.getAttribute("name");
           String email = element.getAttribute("email");
           // 将提取的数据放入 BeanDefinitionBuilder，待完成所有 bean 的解析后统一注册到 beanFactory 中
           if (StringUtils.hasText(name)) {
               bean.addPropertyValue("name", name);
           }
           if (StringUtils.hasText(email)) {
               bean.addPropertyValue("email", email);
           }
       }
   }
   ```

4. 创建一个 Handler 类，扩展 `NamespaceHandlerSupport`，目的是将组件注册到 Spring 容器

   ```java
   public class MyNamespaceHandler extends NamespaceHandlerSupport {
       @Override
       public void init() {
           registerBeanDefinitionParser("student", new StudentBeanDefinitionParser());
       }
   }
   ```
   以上代码很简单，当遇到自定义标签 `<student:aaa >` 这样类型以 `student` 开头的元素，就会把这个元素交给对应的 `StudentBeanDefinitionParser` 去解析

5. 编写 `Spring.handlers` 和 `Spring.schemas` 文件，默认位置：`src/META-INF` 下

   - `spring.handlers`：

     ```
     http\://www.lexueba.com/schema/student=com.test.MyNamespaceHandler
     ```

   - `spring.schemas`：

     ```
     http\://www.lexueba.com/schema/student.xsd=META-INF/Spring-test.xsd
     ```

   Spring 加载自定义的大致流程是遇到自定义标签然后就去 `spring.handlers` 和 `spring.schemas` 中去找对应的 handler 和 XSD，默认在 `META-INF` 下，进而找对应的 Handler 和解析元素的 Parser，从而完成整个自定义元素的解析，也就是说自定义和 Spring 默认配置的不同在于 Spring 将自定义标签解析的工作交给用户实现，Spring 默认的配置也是通过这样的方式来实现的。

6. 创建配置文件，在配置文件引入对应的命名空间和 XSD 后，可以直接使用自定义标签

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          
          xmlns:abc="http://www.lexueba.com/schema/student"
          
          xsi:schemaLocation="
          http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd
          
          http://www.lexueba.com/schema/student
          http://www.lexueba.com/schema/student.xsd
   
   ">
   
       <abc:student id="testBean" name="aaa" email="bbb" />
   
   </beans>
   ```

7. 测试：

   ```java
   @Test
   public void testConstructor() {
       ApplicationContext bf = new ClassPathXmlApplicationContext("test.xml");
       Student bean = (Student) bf.getBean("testBean");
       System.out.println(bean);
   }
   ```

   输出：

   ```
   Student{name='aaa', email='bbb'}
   ```

## 4.2 自定义标签解析

源码：

```java
@Nullable
public BeanDefinition parseCustomElement(Element ele) {
    return this.parseCustomElement(ele, (BeanDefinition)null);
}

@Nullable
// containingBd 为父类 bean，对顶层元素的解析应该设置为 null
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    // 获取对应的命名空间
    String namespaceUri = this.getNamespaceURI(ele);
    if (namespaceUri == null) {
        return null;
    } else {
        // 根据命名空间找到对应的 NamespaceHandler
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler == null) {
            this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        } else {
            // 调用自定义 NamespaceHandler 进行解析
            return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
        }
    }
}
```

### 4.2.1 获取标签命名空间

`String namespaceUri = this.getNamespaceURI(ele);` 上面的第10行代码

源码：

```java
@Nullable
public String getNamespaceURI(Node node) {
    return node.getNamespaceURI();
}
```

### 4.2.2 提取自定义标签处理器

`NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);` 在 `readerContext` 初始化时属性 `namespaceHandlerResolver` 已经被初始化了 `DefaultNamespaceHandlerResolver` 实例，这里调用的 `resolve` 方法其实是 `DefaultNamespaceHandlerResolver` 的方法

源码：

```java
@Nullable
public NamespaceHandler resolve(String namespaceUri) {
    // 获取所有已经配置的 handler 映射
    Map<String, Object> handlerMappings = this.getHandlerMappings();
    // 根据命名空间找到对应信息
    Object handlerOrClassName = handlerMappings.get(namespaceUri);
    if (handlerOrClassName == null) {
        return null;
    } else if (handlerOrClassName instanceof NamespaceHandler) {
        // 已经做过解析，直接从缓存中读取
        return (NamespaceHandler)handlerOrClassName;
    } else {
        // 没有解析，返回类路径
        String className = (String)handlerOrClassName;
        try {
            // 使用反射将类路径转化为类
            Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
            if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri + "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
            } else {
                // 初始化类
                NamespaceHandler namespaceHandler = (NamespaceHandler)BeanUtils.instantiateClass(handlerClass);
                // 调用自定义的 NamespaceHandler 的初始化方法
                namespaceHandler.init();  // 之前我们就继承了 NamespaceHandlerSupport 类，重写了 init 方法
                // 记录在缓存
                handlerMappings.put(namespaceUri, namespaceHandler);
                return namespaceHandler;
            }
        } catch (ClassNotFoundException var7) {
            throw new FatalBeanException("Could not find NamespaceHandler class [" + className + "] for namespace [" + namespaceUri + "]", var7);
        } catch (LinkageError var8) {
            throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" + className + "] for namespace [" + namespaceUri + "]", var8);
        }
    }
}
```

上面展示了解析自定义 `NamespaceHandler` 的过程，如果使用自定义标签，其中一项必不可少的操作就是在 `spring.handlers` 中配置命名空间和命名空间处理器的映射关系。Spring 就可以根据映射关系找到匹配的处理器，找到后就可以执行初始化并解析了，之前的内容：

```java
public class MyNamespaceHandler extends NamespaceHandlerSupport {
    @Override
    public void init() {
        registerBeanDefinitionParser("student", new StudentBeanDefinitionParser());
    }
}
```

这里可以注册多个标签解析器，当前实例中只支持`<myname:student>` 写法

注册后，命名空间处理器就可以根据标签的不同来调用不同的解析器进行解析。

`getHandlerMappings` 主要功能就是读取 `spring.handlers` 配置文件并将配置信息缓存在 map 中

源码：

```java
private Map<String, Object> getHandlerMappings() {
    Map<String, Object> handlerMappings = this.handlerMappings;
    // 没有缓存就开始进行缓存
    if (handlerMappings == null) {
        synchronized(this) {
            handlerMappings = this.handlerMappings;
            if (handlerMappings == null) {
                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
                }
                try {
                    // this.handlerMappingsLocation 默认是 META-INF.spring.handlers
                    Properties mappings = PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("Loaded NamespaceHandler mappings: " + mappings);
                    }
                    handlerMappings = new ConcurrentHashMap(mappings.size());
                    // 将 properties 合并到 Map 格式的 handdlerMappings 中
                    CollectionUtils.mergePropertiesIntoMap(mappings, (Map)handlerMappings);
                    this.handlerMappings = (Map)handlerMappings;
                } catch (IOException var5) {
                    throw new IllegalStateException("Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", var5);
                }
            }
        }
    }
    return (Map)handlerMappings;
}
```

### 4.2.3 标签解析

得到了解析器后，Spring 就可以让解析器去解析了

`return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));` 源码：

handler 被实例化成我们自定义的 handler 了，而我们自定义的 handler 是继承了 `NamespaceHandler` 抽象类，所以这里实例实现的应该是 `NamespaceHandler` 抽象类的方法：

```java
@Nullable
public BeanDefinition parse(Element element, ParserContext parserContext) {
    // 寻找解析器进行解析操作
    BeanDefinitionParser parser = this.findParserForElement(element, parserContext);
    return parser != null ? parser.parse(element, parserContext) : null;
}

@Nullable
private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
    // 获取元素名称，也就是 <myname:student> 中的 student，实例中的 localName 为 student
    String localName = parserContext.getDelegate().getLocalName(element);
    // 根据 student 找到对应的解析器
    // 也就是在 registerBeanDefinitionParser("student", new StudentBeanDefinitionParser()); 时注册的
    BeanDefinitionParser parser = (BeanDefinitionParser)this.parsers.get(localName);
    if (parser == null) {
        parserContext.getReaderContext().fatal("Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
    }
    return parser;
}
```

关于`parser.parse(element, parserContext)` 源码：

```java
@Nullable
public final BeanDefinition parse(Element element, ParserContext parserContext) {
    AbstractBeanDefinition definition = this.parseInternal(element, parserContext);
    if (definition != null && !parserContext.isNested()) {
        try {
            String id = this.resolveId(element, definition, parserContext);
            if (!StringUtils.hasText(id)) {
                parserContext.getReaderContext().error("Id is required for element '" + parserContext.getDelegate().getLocalName(element) + "' when used as a top-level tag", element);
            }
            String[] aliases = null;
            if (this.shouldParseNameAsAliases()) {
                String name = element.getAttribute("name");
                if (StringUtils.hasLength(name)) {
                    aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
                }
            }
            // 将 AbstractBeanDefinition 转为 BeanDefinitionHolder 并注册
            BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
            this.registerBeanDefinition(holder, parserContext.getRegistry());
            if (this.shouldFireEvents()) {
                // 需要通知监听器则进行处理
                BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
                this.postProcessComponentDefinition(componentDefinition);
                parserContext.registerComponent(componentDefinition);
            }
        } catch (BeanDefinitionStoreException var8) {
            String msg = var8.getMessage();
            parserContext.getReaderContext().error(msg != null ? msg : var8.toString(), element);
            return null;
        }
    }
    return definition;
}
```

真正解析成为 `AbstractBeanDefinition` 的其实是调用方法：`this.parseInternal(element, parserContext);`，上面这个方法只是将 `AbstractBeanDefinition` 转化为 `BeanDefinitionHolder` 并注册

`this.parseInternal(element, parserContext)` 源码：

```java
protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
    String parentName = this.getParentName(element);
    if (parentName != null) {
        builder.getRawBeanDefinition().setParentName(parentName);
    }
    // 获取自定义标签中的 class,此时会调用自定义解析器如 StudentBeanDefinitionParser 中的 getBeanClass 方法
    Class<?> beanClass = this.getBeanClass(element);
    if (beanClass != null) {
        builder.getRawBeanDefinition().setBeanClass(beanClass);
    } else {
        // 如果子类没有重写 getBeanClass 方法，则尝试子类是否重写 getBeanClassName 方法
        String beanClassName = this.getBeanClassName(element);
        if (beanClassName != null) {
            builder.getRawBeanDefinition().setBeanClassName(beanClassName);
        }
    }
    builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
    BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
    if (containingBd != null) {
        // 存在父类则使用父类的 scope 属性。
        builder.setScope(containingBd.getScope());
    }
    if (parserContext.isDefaultLazyInit()) {
        // 懒加载
        builder.setLazyInit(true);
    }
    // 调用子类重写的 doParser 方法进行解析
    this.doParse(element, parserContext, builder);
    return builder.getBeanDefinition();
}
```

这里的 `doParser` 就是我们之前的：

<img src="C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220503015711217.png" alt="image-20220503015711217" style="zoom: 67%;" />

# Bean 的加载

bean 的加载比 bean 的解析要复杂的多。

对于加载 bean，在 spring 中的调用方式为：

```
Student bean = (Student) bf.getBean("testBean");
```

查看其源码`AbstractBeanFactory.getBean`：

```java
public Object getBean(String name) throws BeansException {
    return this.doGetBean(name, (Class)null, (Object[])null, false);
}

protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
    // 提取对应的 beanName
    String beanName = this.transformedBeanName(name);
    Object sharedInstance = this.getSingleton(beanName);
    // 检查缓存中或者实例工厂中是否有对应的实例。
    // 在创建单例 bean 的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，
    // spring 创建 bean 的原则是不等 bean 创建完成就会创建 bean 的 ObjectFactory 提早曝光
    // 也就是将 ObjectFactory 加入到缓存中，一旦下个 bean 创建时就需要依赖上个 bean 则直接使用 ObjectFactory
    Object bean;
    if (sharedInstance != null && args == null) {
        if (this.logger.isTraceEnabled()) {
            if (this.isSingletonCurrentlyInCreation(beanName)) {
                this.logger.trace("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
            } else {
                this.logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        // 返回对应的实例，有时候存在诸如 BeanFactory 的情况并不是直接返回实例本身而是返回指定方法返回的实例
        bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
    } else {
        // 只有在单例情况才会尝试解决循环依赖
        // 原型模式情况下，如果存在 A 中有 B 的属性，B 中有 A 的属性，当依赖注入时
        // 就会产生当 A 还没创建完时因为对 B 的创建再次返回创建 A，造成循环依赖
        if (this.isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        BeanFactory parentBeanFactory = this.getParentBeanFactory();
        // 如果 beanDefinitionMap 中也就是所有已经加载的类中不包括 beanName 则尝试从 parentBeanFactory 中检测
        if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
            String nameToLookup = this.originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory)parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
            }
            // 递归到 beanFactory 中查找
            if (args != null) {
                return parentBeanFactory.getBean(nameToLookup, args);
            }
            if (requiredType != null) {
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            return parentBeanFactory.getBean(nameToLookup);
        }
        // 如果不是仅仅做类型检查而是创建 bean，这里要进行记录
        if (!typeCheckOnly) {
            this.markBeanAsCreated(beanName);
        }
        try {
            // 将存储 XML 配置文件的 GernericBeanDefiniton 转为 RootBeanDefinition
            // 如果指定 BeanName 是子 Bean 的话同时会合并父类相关属性
            RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
            this.checkMergedBeanDefinition(mbd, beanName, args);
            String[] dependsOn = mbd.getDependsOn();
            String[] var11;
            // 若存在依赖则需要递归实例化依赖的 bean
            if (dependsOn != null) {
                var11 = dependsOn;
                int var12 = dependsOn.length;
                for(int var13 = 0; var13 < var12; ++var13) {
                    String dep = var11[var13];
                    if (this.isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 缓存依赖调用
                    this.registerDependentBean(dep, beanName);
                    try {
                        this.getBean(dep);
                    } catch (NoSuchBeanDefinitionException var24) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "'" + beanName + "' depends on missing bean '" + dep + "'", var24);
                    }
                }
            }
            // 实例化依赖的 bean 后便可以实例化 mbd 本身了
            // 单例模式创建
            if (mbd.isSingleton()) {
                sharedInstance = this.getSingleton(beanName, () -> {
                    try {
                        return this.createBean(beanName, mbd, args);
                    } catch (BeansException var5) {
                        this.destroySingleton(beanName);
                        throw var5;
                    }
                });
                bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            } else if (mbd.isPrototype()) {
                // 原型模式创建 new
                var11 = null;
                Object prototypeInstance;
                try {
                    this.beforePrototypeCreation(beanName);
                    prototypeInstance = this.createBean(beanName, mbd, args);
                } finally {
                    this.afterPrototypeCreation(beanName);
                }
                bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            } else {
                // 指定的 scope 上实例化 bean
                String scopeName = mbd.getScope();
                if (!StringUtils.hasLength(scopeName)) {
                    throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
                }
                Scope scope = (Scope)this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        this.beforePrototypeCreation(beanName);
                        Object var4;
                        try {
                            var4 = this.createBean(beanName, mbd, args);
                        } finally {
                            this.afterPrototypeCreation(beanName);
                        }
                        return var4;
                    });
                    bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                } catch (IllegalStateException var23) {
                    throw new BeanCreationException(beanName, "Scope '" + scopeName + "' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var23);
                }
            }
        } catch (BeansException var26) {
            this.cleanupAfterBeanCreationFailure(beanName);
            throw var26;
        }
    }
    // 检查需要的类型是否符合 bean 的实际类型
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = this.getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            } else {
                return convertedBean;
            }
        } catch (TypeMismatchException var25) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Failed to convert bean '" + name + "' to required type '" + ClassUtils.getQualifiedName(requiredType) + "'", var25);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    } else {
        return bean;
    }
}
```

对于加载过程中设计的步骤大致如下：

- **转换对应 `beanName`**

  注意传入的参数可能是别名，也可能是 `FactoryBean`，所以要进行一系列的解析

  - 去除 `FactoryBean` 的修饰符，也就是如果 `name = "&aa"`，那么首先取出 & 而使 `name = "aa"`
  - 取指定 alias 所表示的最终 `beanName`。

- **尝试从缓存中加载单例**

  单例在 Spring 的同一个容器只会创建一次，后续再获取 bean，就直接从单例缓存中获取了。

  这里也只是尝试加载，首先尝试从缓存中加载，如果加载不成功再从 `singletonFactories` 中加载。

  因为在创建单例 bean 时会存在依赖注入问题，创建依赖时为了避免循环依赖，在 Spring 创建 bean 的原则是不等 bean 创建完成就会将创建 bean 的 `ObjectFactory` 提早曝光加入缓存中，一旦下一个 bean 创建时需要依赖上一个 bean 则直接使用 `ObjectFactory`

- **bean的实例化** 

  如果从缓存中得到了 bean 的原始状态，则需要对 bean 进行实例化。

  > 缓存中记录的只是最原始的 bean 状态，不是我们最终想要的 bean。
  >
  > 举个例子：假如我们需要对工厂 bean 进行处理，那么这里得到的其实是工厂 bean 的初始状态，但我们真正需要的是工厂 bean 中定义的 `factory-method` 方法中返回的 bean，而 `getObjectForBeanInstance` 就是完成这个工作的。

- **原型模式的依赖检查**

  只有单例模式才会解决循环依赖。

- **检测 `parentBeanFactory`**

  如果缓存没有数据直接转到父工厂加载

  > 如果当前加载的 XML 配置文件中不包含 `beanName` 的配置，就只能到 `parentBeanFactory`（首先得有）去尝试，然后再递归调用 `getBean`

- **将存储 XML 配置文件的 `GernericBeanDefinition` 转为 `RootBeanDefinition`**

  因为从 XML 读取的 bean 信息存储在 `GernericBeanDefinition` 中，但所有的 bean 后续处理都是针对于 `RootBeanDefinition`，所以这里要进行一个转换，转换的同时如果父类 bean 不为空，会一并合并父类属性。

- **寻找依赖**

  bean 初始化可能用到某些属性，这些属性可能是动态配置，且配置成依赖于其他 bean，这时有必要先加载依赖的 bean，所以在 spring 的加载顺序中，初始化某一个 bean 会先初始化这个 bean 对应的依赖。

- **针对不同的 scope 进行 bean 创建**

  spring 中存在不同的 scope，spring 要根据不同的配置进行不同的初始化策略。

- **类型转换**

  它的功能是将返回的 bean 转换为 `requiredType` 指定的类型。

![image-20220505024903922](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220505024903922.png)

## 5.1 `FactoryBean` 的使用

一般情况下，Spring 通过反射利用 bean 的 class 属性指定实现类来实例化 bean。某些情况下，实例化 bean 比较复杂，如果按照传统方式需要在 bean 中提供大量配置信息，配置方式的灵活性受限，这时采用编码方式会得到一个简单方案。

spring 为此提供了一个 `org.springframework.beans.factory.FactoryBean` 的工厂类接口，用户可以实现该接口定制实例化 bean 逻辑。

`FactoryBean` 接口对于 Spring 来说就占有很重要的地位，spring 自身就提供了多个实现。

可以隐藏实例化一些复杂 bean 的细节，给上层应用带来便利。

`FactoryBean<T>` 源码：

```java
public interface FactoryBean<T> {
	// 返回由 FactoryBean 创建的实例，如果 isSingleton() 返回 true，则该实例会放到 Spring 单例缓存池中
    @Nullable
	T getObject() throws Exception;

    // 返回 FactoryBean 创建的 bean 类型
	@Nullable
	Class<?> getObjectType();

    // 作用域是单例还是原型
	default boolean isSingleton() {
		return true;
	}

}
```

当配置文件中 bean 的 class 属性配置的实现类时 `FactoryBean` 时，通过 `getBean()` 方法返回的不是 `FactoryBean` 本身，而是 `getObject()` 方法返回的对象，相当于 `Factory.getObject()` 代理了 `getBean()` 方法。

例如：

```java
// 定义 Car 类
public class Car {
    private int maxSpeed;
    private String brand;
    private double price;
	// get、set 方法
}
```

如果用 `FactoryBean` 的方式实现就会灵活一些：

```java
public class CarFactoryBean implements FactoryBean<Car> {

    String info;

    @Override
    public Car getObject() throws Exception {
        Car car = new Car();
        String[] split = info.split(",");
        car.setBrand(split[0]);
        car.setMaxSpeed(Integer.parseInt(split[1]));
        car.setPrice(Double.parseDouble(split[2]));
        return car;
    }

    @Override
    public Class<?> getObjectType() {
        return Car.class;
    }

    @Override
    public boolean isSingleton() {
        return false;
    }

    public void setInfo(String info) {
        this.info = info;
    }
}
```

xml 中配置：

```xml
<bean class="com.test.CarFactoryBean" id="car" >
    <property name="info" value="超跑,400,2000000" />
</bean>
```

测试：

```java
@Test
public void testCar() {
    ApplicationContext bf = new ClassPathXmlApplicationContext("bean1.xml");
    Car car = (Car)bf.getBean("car");
    System.out.println(car);
}
```

打印：

```
Car{maxSpeed=400, brand='超跑', price=2000000.0}
```

当调用 `getBean("car")` 时，Spring 通过反射发现 `CarFactoryBean` 实现了 `FactoryBean`，这是 spring 就调用 `getObject` 返回，如果此时我们使用 `getBean(beanName)` 在 `beanName` 前加上 & 符号，那么就可以获取 `FactoryBean` 实例，例如 `geetBean("&car")`

## 5.2 缓存中获取单例 bean

Spring 获取单例 bean 时会先尝试从单例缓存中获取，如果获取不到再去 `singletonFactories` 中尝试加载，如果都没有说明第一次创建，则创建。

spring 单例为了解决循环依赖，spring 不等 bean 创建完就会将创建 bean 的 `ObjectFactory` 提早曝光到加入到缓存中，一旦下一个 bean 创建时需要依赖上个 bean，直接使用 `ObjectFactory`

源码：

```java
@Nullable
public Object getSingleton(String beanName) {
    return this.getSingleton(beanName, true);
}

@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 检查缓存中是否存在实例
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            // 如果为空，锁定全局变量并进行处理
            synchronized(this.singletonObjects) {
                // 如果此 bean 正在加载则不处理
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    // 某些方法需要提前初始化时会调用 addSingletomFactory 方法将对应的 ObjectFactory 初始化策略
                    // 存储在 singletonFactories
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        
                        ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            // 调用预先设定的 getObject()
                            singletonObject = singletonFactory.getObject();
                            // 记录在缓存中，earlySingletonObjects 和 singletonFactories 互斥
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

这个方法涉及循环依赖的检测，首先阐明几个变量的意思：

传说中的三级缓存：

![image-20220505233551938](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220505233551938.png)

- `singletonObjects`：用于保存 `beanName -> bean instance` 的映射关系，一级缓存。
- `singletonFactories`：用于保存 `beanName -> ObjectFactory`，bean name 和创建 bean 的工厂之间的映射关系。
- `earlySingletonObjects`：保存 bean name 和创建 bean 实例之间的关系，与 `singletonObjects` 不同在于：当一个单例 bean 放到里面后，当 bean 还在创建过程中，就可以通过 `getBean` 获取到了，目的用来检测循环引用。
- `registeredSingletons`：保存当前所有已注册的 bean。

**过程：**

- 先从 `singletonObjects` 中获取，如果没有
- 从 `earlySingletonObjects` 中获取，如果还没有
- 尝试从 `singletonFactories` 中获取 bean name 对应的 `ObjectFactory` ，然后调用它的 `getObject` 来创建 bean，并放入 `earlySingletonObjects` 中，并且从 `singletonFactories` 中 remove 这个 `ObjectFactory`

## 5.3 从 bean 实例中获取对象

在 `getBean` 方法中，`getObjectForBeanInstance` 是个高频使用的方法，无论从缓存中获得 bean 还是根据不同的 scope 策略加载 bean。我们得到 bean 的实例后要做的第一件事就是调用这个方法来检测正确性，就是检测当前 bean 是否是 `FactoryBean` 类型的 bean，如果是，那么需要调用 bean 对应的 `FactoryBean` 实例中的 `getObject` 作为返回值。

无论从缓存中获取 bean 还是不同的 scope 加载 bean 都只是原始的 bean 状态，不是我们最终想要的 bean。例如我们想要对工厂 bean 进行处理，那么这里得到的其实是工厂 bean 的初始状态，我们真正需要的是工厂 bean 中定义的 `factory-method` 方法中返回的 bean，而 `getObjectForBeanInstance` 方法就是完成这个工作的。

源码：

```java
protected Object getObjectForBeanInstance(Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        // 如果指定的 name 是工厂相关（& 开头）
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        } else if (!(beanInstance instanceof FactoryBean)) {
            // 不是 FactoryBean 实例，报异常（命名不符要求）
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        } else {
            if (mbd != null) {
                mbd.isFactoryBean = true;
            }
            // 用户就想获得 FactoryBean 实例，直接返回
            return beanInstance;
        }
    } else if (!(beanInstance instanceof FactoryBean)) {
        // 不是工厂类，直接返回
        return beanInstance;
    } else {
        Object object = null;
        if (mbd != null) {
            // 父节点是 FactoryBean
            mbd.isFactoryBean = true;
        } else {
            // 当前实例就是父节点
            object = this.getCachedObjectForFactoryBean(beanName);
        }
        if (object == null) {
            // 明确知道 beanInstance 是工厂类型
            FactoryBean<?> factory = (FactoryBean)beanInstance;
            // containsBeanDefinition 检测 beanDefinitionMap 中也就是在所有已经加载的类中检测是否定义 beanName
            if (mbd == null && this.containsBeanDefinition(beanName)) {
                // 将存储 XML 配置文件的 GernericBeanDefinition 转换为 RootBeanDefinition，如果指定 beanName 				 // 是子 bean 的话同时会合并父类相关属性。
                mbd = this.getMergedLocalBeanDefinition(beanName);
            }
            // 是否是用户定义的而不是程序本身定义的
            boolean synthetic = mbd != null && mbd.isSynthetic();
            // 委托给 getObjectFromFactoryBean
            object = this.getObjectFromFactoryBean(factory, beanName, !synthetic);
        }
        return object;
    }
}
```

以上代码做了如下工作：

- 对 `FactoryBean` 正确性的验证

- 对非 `FactoryBean` 不做任何处理

- 对 bean 进行转换

- 将从 `Factory` 中解析 bean 的工作委托给 `getObjectFromFactoryBean`

  ```java
  protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
      if (factory.isSingleton() && this.containsSingleton(beanName)) {
          // 是单例
          synchronized(this.getSingletonMutex()) {
              Object object = this.factoryBeanObjectCache.get(beanName);
              if (object == null) {
                  // 委托给 doGetObjectFromFactoryBean
                  object = this.doGetObjectFromFactoryBean(factory, beanName);
                  Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                  if (alreadyThere != null) {
                      object = alreadyThere;
                  } else {
                      if (shouldPostProcess) {
                          // 后置处理
                          if (this.isSingletonCurrentlyInCreation(beanName)) {
                              return object;
                          }
                          this.beforeSingletonCreation(beanName);
                          try {
                              object = this.postProcessObjectFromFactoryBean(object, beanName);
                          } catch (Throwable var14) {
                              throw new BeanCreationException(beanName, "Post-processing of FactoryBean's singleton object failed", var14);
                          } finally {
                              this.afterSingletonCreation(beanName);
                          }
                      }
                      if (this.containsSingleton(beanName)) {
                          this.factoryBeanObjectCache.put(beanName, object);
                      }
                  }
              }
              return object;
          }
      } else {
          // 原型
          Object object = this.doGetObjectFromFactoryBean(factory, beanName);
          if (shouldPostProcess) {
              try {
                  object = this.postProcessObjectFromFactoryBean(object, beanName);
              } catch (Throwable var17) {
                  throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", var17);
              }
          }
          return object;
      }
  }
  ```

这个方法中只做了一件事：返回的 bean 如果是单例，要求全局唯一，同时因为单例，所以不用重复创建，可以用缓存提高性能

```java
  // doGetObjectFromFactoryBean
  private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
      Object object;
      try {
          // 需要权限认证
          if (System.getSecurityManager() != null) {
              AccessControlContext acc = this.getAccessControlContext();
              try {
                  object = AccessController.doPrivileged(factory::getObject, acc);
              } catch (PrivilegedActionException var6) {
                  throw var6.getException();
              }
          } else {
              // 直接调用 getObject 方法
              object = factory.getObject();
          }
      } catch (FactoryBeanNotInitializedException var7) {
          throw new BeanCurrentlyInCreationException(beanName, var7.toString());
      } catch (Throwable var8) {
          throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", var8);
      }
      if (object == null) {
          if (this.isSingletonCurrentlyInCreation(beanName)) {
              throw new BeanCurrentlyInCreationException(beanName, "FactoryBean which is currently in creation returned null from getObject");
          }
          object = new NullBean();
      }
      return object;
  }
```

如果 bean 声明为 `FacotryBean` 类型，则当提取 bean 时提取的不是 `FactoryBean`，而是 `FactoryBean` 中对应的 `getObject` 方法返回的 bean，上面这段代码也就完成了这个工作。

除此之外，创建 bean 后还要后置处理：

```java
  // AbstractAutowireCapableBeanFactory.postProcessObjectFromFactoryBean
  @Override
  protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
  	return applyBeanPostProcessorsAfterInitialization(object, beanName);
  }
  
  @Override
  public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
  		throws BeansException {
  	Object result = existingBean;
  	for (BeanPostProcessor processor : getBeanPostProcessors()) {
  		Object current = processor.postProcessAfterInitialization(result, beanName);
  		if (current == null) {
  			return result;
  		}
  		result = current;
  	}
  	return result;
  }
```

spring 在获取 bean 中有这样的规则：尽可能保证所有 bean 初始化后都会调用注册的 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法进行处理，实际开发中可以针对此特性设计自己的业务逻辑。

## 5.4 获取单例

如果缓存中不存在已经加载的单例 bean 就需要从头开始 bean 的加载了，而 spring 中使用 `getSingleton` 的重载方法实现 bean 的加载过程：

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    // 全局变量需要同步
    synchronized(this.singletonObjects) {
        // 首先检查对应的 bean 是否已经加载过，因为 singleton 模式就是复用已经创建的 bean
        // 这一步是必须的
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            // 为空才可以初始化
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName, "Singleton bean creation not allowed while singletons of this factory are in destruction (Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            // 加载单例前
            this.beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = this.suppressedExceptions == null;
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet();
            }
            try {
                // 初始化 bean
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            } catch (IllegalStateException var16) {
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw var16;
                }
            } catch (BeanCreationException var17) {
                BeanCreationException ex = var17;
                if (recordSuppressedExceptions) {
                    Iterator var8 = this.suppressedExceptions.iterator();
                    while(var8.hasNext()) {
                        Exception suppressedException = (Exception)var8.next();
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            } finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                // 加载单例后
                this.afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 加入缓存
                this.addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```

上述代码其实使用了回调方法，程序可以在单例创建前后做一些准备和处理操作，真正获取单例 bean 不是在此方法中实现，实现逻辑在 `ObjectFactory.singletonFactory` 中实现的，这些准备及处理操作包括：

- 检查缓存是否已经加载了。

- 没有加载则记录 `beanName` 正在加载状态

- 加载单例前记录加载状态

  加载单例前：

  ```java
  protected void beforeSingletonCreation(String beanName) {
      if (!this.inCreationCheckExclusions.contains(beanName) && 
          // 通过 singletonsCurrentlyInCreation.add(beanName)) 将当前正要创建的 bean 记录在缓存中，可以对循环依         // 赖进行检测
          !this.singletonsCurrentlyInCreation.add(beanName)) {
          throw new BeanCurrentlyInCreationException(beanName);
      }
  }
  ```

- 通过调用参数传入的 `ObjectFactory` 的个体 `Object` 方法实例化 bean

- 加载单例后的处理方法调用

  同加载单例前的处理方法调用

  ```java
  protected void afterSingletonCreation(String beanName) {
      if (!this.inCreationCheckExclusions.contains(beanName) && 
          !this.singletonsCurrentlyInCreation.remove(beanName)) {
          throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
      }
  }
  ```

- 将结果记录值缓存并删除加载 bean 过程中所记录的各种辅助状态

  ```java
  protected void addSingleton(String beanName, Object singletonObject) {
      synchronized(this.singletonObjects) {
          // 记录缓存
          this.singletonObjects.put(beanName, singletonObject);
          // 移除 二三级缓存中的记录
          this.singletonFactories.remove(beanName);
          this.earlySingletonObjects.remove(beanName);
          // 标记该 beanName 已经加载了
          this.registeredSingletons.add(beanName);
      }
  }
  ```

- 返回处理结果

  ```java
  // bean 的加载逻辑其实是在传入的 ObjectFactory 类型的参数 singletonFactory 中定义
  sharedInstance = this.getSingleton(beanName, () -> {
      try {
          // 核心
          return this.createBean(beanName, mbd, args);
      } catch (BeansException var5) {
          this.destroySingleton(beanName);
          throw var5;
      }
  });
  ```

  

## 5.5 准备创建 bean

Spring 的一个规律：一个真正干活的方法其实是以 do 开头的，例如 `doGetObjectFromFactoryBean`，而 `getObjectFromFactoryBean` 则其实是从全局角度做些统筹工作。这个规则对于 `createBean` 也相同。

`createBean` 源码：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;
    // 锁定 class,根据设置的 class 属性或者根据 className 来解析 class
    Class<?> resolvedClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }
    try {
        // 验证及准备覆盖的方法
        mbdToUse.prepareMethodOverrides();
    } catch (BeanDefinitionValidationException var9) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(), beanName, "Validation of method overrides failed", var9);
    }
    Object beanInstance;
    try {
        // 给 BeanPostProcessors 一个机会来返回代理来替代真正的实例
        beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
        
        if (beanInstance != null) {
            return beanInstance;
        }
    
    } catch (Throwable var10) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "BeanPostProcessor before instantiation of bean failed", var10);
    }
    try {
        // 真正干活的方法
        beanInstance = this.doCreateBean(beanName, mbdToUse, args);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    } catch (ImplicitlyAppearedSingletonException | BeanCreationException var7) {
        throw var7;
    } catch (Throwable var8) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", var8);
    }
}
```

总结函数完成的具体步骤：

- 根据设置的 class 属性或者根据 `className` 来解析 Class

- 对 override 属性进行标记及验证

  在 Spring 中没有 override-method 这样的配置，但是在 Spring 配置中存在 `lookup-method` 以及 `replace-method` 的，而这两个配置的加载其实就是将配置统一存放在 `BeanDefinition` 中的 `methodOverrides` 属性中的，这个函数的操作其实也就是针对于这两个配置。

- 应用初始化前的后处理器，解析指定 bean 是否存在初始化前的短路操作。

- 创建 bean

### 5.5.1 处理 override 属性

```java
public void prepareMethodOverrides() throws BeanDefinitionValidationException {
    if (this.hasMethodOverrides()) {
        this.getMethodOverrides().getOverrides().forEach(this::prepareMethodOverride);
    }
}

protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
    // 获取对应类中对应方法名的个数
    int count = ClassUtils.getMethodCountForName(this.getBeanClass(), mo.getMethodName());
    if (count == 0) {
        throw new BeanDefinitionValidationException("Invalid method override: no method with name '" + mo.getMethodName() + "' on class [" + this.getBeanClassName() + "]");
    } else {
        if (count == 1) {
            // 标记 MethodOverride 暂时没有覆盖，避免参数类型检查的开销
            mo.setOverloaded(false);
        }
    }
}
```

在 spring 配置中存在 `lookup-method` 和 `replace-method` 两个配置功能，这两个配置的加载其实就是将配置统一存放在 `BeanDefinition` 中的 `methodOverrides` 属性中，这两个功能实现原理其实在 bean 实例化的时候如果检测到存在 `methodOverrides` 属性，会动态为当前 bean 生成代理并使用对应拦截器为 bean 做增强操作。

在这里 Spring 还做了一个优化，如果一个方法名只有一个方法说明没有重载，就不需要进行参数验证了，这样可以提高效率。

### 5.5.2 实例化的前置处理

真正调用 `doCreate` 方法创建 bean 的实例前使用了这样一个方法 `resolveBeforeInstantiation(beanName, mbd)` 对 `BeanDefinition` 中的属性做前置处理。在这个函数中提供了一个短路判断，这才是最关键的部分：

```java
if (bean != null) {
    return bean;
}
```

 当经过前置处理后返回的结果不为空，那么会直接略过后面的 bean 创建而直接返回结果，这一特性起着很重要的作用，AOP 功能就是基于这里的判断。

```java
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    // 尚未被解析
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
            // 确保 bean class 在这里被解析过了
            Class<?> targetType = this.determineTargetType(beanName, mbd);
            if (targetType != null) {
                // 实例化前采用 Bean 后处理器
                bean = this.applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    // 实例化后采用 bean 后处理器
                    bean = this.applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = bean != null;
    }
    return bean;
}
```

这个方法中有两个方法：`applyBeanPostProcessorsBeforeInstantiation`、`applyBeanPostProcessorsAfterInitialization`

这两个方法的实现很简单，无非是对后处理器中所有的 `InstantiationAwareBeanPostProcessor` 类型的后处理器进行 `postProcessBeforeInstantiation` 方法和 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法的调用

#### 实例化前的后处理器应用

将 `AbstractBeanDefinition` 转换为 `BeanWrapper` 前的处理。给子类一个修改 `BeanDefinition` 的机会，程序经过这个方法后，bean 已经不是我们的 bean了，或许成了一个经过处理的代理 bean，可能通过 `cglib` 生成

```java
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
    Iterator var3 = this.getBeanPostProcessors().iterator();
    while(var3.hasNext()) {
        BeanPostProcessor bp = (BeanPostProcessor)var3.next();
        if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
            // 实例化前的后置处理
            Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
            if (result != null) {
                return result;
            }
        }
    }
    return null;
}
```

#### 实例化后的后处理器

spring 中的规则是在 bean 初始化后尽可能保证将注册的后处理器的 `postProcessAfterInitialization` 方法应用到该 bean 中，因为如果返回的 bean 不为空，不会再次经历普通 bean 的创建过程，所以只能在这里应用后处理器的 `postProcessAfterInitializaiton` 方法

```java
 public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
     Object result = existingBean;
     Object current;
     for(Iterator var4 = this.getBeanPostProcessors().iterator(); var4.hasNext(); result = current) {
         BeanPostProcessor processor = (BeanPostProcessor)var4.next();
         current = processor.postProcessAfterInitialization(result, beanName);
         if (current == null) {
             return result;
         }
     }
     return result;
 }
```

## 5.6 循环依赖

### 5.6.1 什么是循环依赖

循环依赖就是循环引用，就是多个 bean 互相之间持有对方，它们最终反映为一个环。

<img src="C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220507015430546.png" alt="image-20220507015430546" style="zoom: 50%;" />

### 5.6.2 spring 如何解决循环依赖

spring 循环依赖包括构造器循环依赖和 setter 循环依赖。

spring 将循环依赖的处理分成 3 种情况：

#### 构造器循环依赖

**通过构造器注入构成的循环依赖，是无法解决的**，只能抛出 `BeanCurrentlyInCreationException` 异常

在创建 A 类时，构造器需要创建 B 类，将去创建 B，创建 B 时又发现需要 C，又去创建 C，最终创建 C 时发现需要 A，从而形成一个环，无法创建。

原理：spring 容器将每一个正在创建的 bean 标识符放在一个 "当前创建 bean 池" 中，bean 标识符在创建过程中将一直保持在这个池中，因此如果在创建 bean 过程中发现自己已经在 "当前创建 bean 中"时，将抛出 `BeanCurrentlyInCreationException` 异常表示循环依赖，而对于创建完毕的 bean 将从 "当前创建 bean 池" 中清除掉。

用例：

```xml
<bean id="A" class="com.test.A">
    <constructor-arg index="0" ref="B" />
</bean>

<bean id="B" class="com.test.B">
    <constructor-arg index="0" ref="C" />
</bean>

<bean id="C" class="com.test.C">
    <constructor-arg index="0" ref="A" />
</bean>
```

分析：

- Spring 容器创建 "A" bean，首先去 "当前创建 bean 池" 查找是否当前 bean 正在创建，如果没发现，则继续准备其需要的构造器参数 "B"，并将 "A" 放入 "当前创建 bean 池" 中。
- Spring 容器创建 "B" bean，首先去 "当前创建 bean 池" 查找是否当前 bean 正在创建，如果没发现，则继续准备其需要的构造器参数 "C"，并将 "B" 放入 "当前创建 bean 池" 中。
- Spring 容器创建 "C" bean，首先去 "当前创建 bean 池" 查找是否当前 bean 正在创建，如果没发现，则继续准备其需要的构造器参数 "A"，并将 "C" 放入 "当前创建 bean 池" 中。
- 到此为止 Spring 容器又要去创建 "A"，发现该 bean 标识符在 "当前创建 bean 池" 中，表示循环依赖，抛出 `BeanCurrentlyInCreationException`.

#### setter 循环依赖

**通过 setter 注入方式构成的循环依赖，是可以解决的，通过三级缓存**

setter 注入造成的循环依赖只能解决单例作用域的 bean 循环依赖。通过提前暴露一个单例工厂方法，从而使其他 bean 能引用到该 bean。

如下代码：

```java
addSingletonFactory(beanName, new Object Factory() {
   public Object getObject() throws BeanException {
		return getEarlyBeanRefrence(beanName, mbd, bean);
   } 
});
```

具体步骤：

- Spring 容器创建单例 "A"，首先根据无参构造器创建 bean，并暴露一个 "`ObjectFactory`" 用于返回一个提前暴露一个创建中的 bean，并将 "A" 标识符放到 "当前创建 bean 池"，然后 setter 注入 "B"
- Spring 容器创建单例 "B"，首先根据无参构造器创建 bean，并暴露一个 "`ObjectFactory`" 用于返回一个提前暴露一个创建中的 bean，并将 "B" 标识符放到 "当前创建 bean 池"，然后 setter 注入 "C"
- Spring 容器创建单例 "C"，首先根据无参构造器创建 bean，并暴露一个 "`ObjectFactory`" 用于返回一个提前暴露一个创建中的 bean，并将 "C" 标识符放到 "当前创建 bean 池"，然后 setter 注入 "A"。进行注入 "A" 时由于提前暴露了 "`ObjectFactory`" 工厂，从而使用它返回提前暴露一个创建中的 bean
- 最后再依赖注入 "B" 和 "A"，完成 setter 注入

#### 原型范围的依赖处理

对于原型作用域 bean，Spring 容器无法完成依赖注入，因为 Spring 容器不缓存 原型作用域的 bean，因此无法提前暴露一个创建中的 bean。

所以如果原型模式下的循环依赖就会抛出循环依赖异常。

## 5.7 创建 bean

如果创建了代理或者重写了 `InstantiationAwareBeanPostProcessor` 的 `postProcessBeforeInstantiation` 方法并在方法 `postProcessBeforeInstantiation` 中改变了 bean，直接返回就可以了，否则就进行常规 bean 的创建。

在 `doCreateBean(beanName, mbdToUse, args)` 中完成：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    // 实例化 bean
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        // 如果是单例则需要先清除缓存
        instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 根据指定 bean 使用对应的策略创建新的实例，如：工厂方法、构造函数自动注入、简单初始化
        instanceWrapper = this.createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }
    synchronized(mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            } catch (Throwable var17) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
            }
            mbd.postProcessed = true;
        }
    }
    // 是否需要提早曝光：单例并且允许循环依赖并且当前 bean 正在创建中
    boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
    if (earlySingletonExposure) {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
        }
        // 在 bean 初始化前创建实例的 ObjectFactory 加入工厂
        this.addSingletonFactory(beanName, () -> {
            return this.getEarlyBeanReference(beanName, mbd, bean);
        });
    }
    Object exposedObject = bean;
    try {
        // 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性，则会递归初始依赖 bean
        this.populateBean(beanName, mbd, instanceWrapper);
        // 调用初始化方法，例如 init-method
        exposedObject = this.initializeBean(beanName, exposedObject, mbd);
    } catch (Throwable var18) {
        if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
            throw (BeanCreationException)var18;
        }
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
    }
    if (earlySingletonExposure) {
        Object earlySingletonReference = this.getSingleton(beanName, false);
        // earlySingletonReference 只有检测到有循环依赖情况下才不为空
        if (earlySingletonReference != null) {
            // 如果 exposedObject 没有在初始化方法中被改变，也就是没有被增强
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                String[] dependentBeans = this.getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                String[] var12 = dependentBeans;
                int var13 = dependentBeans.length;
                for(int var14 = 0; var14 < var13; ++var14) {
                    String dependentBean = var12[var14];
                    // 检测依赖
                    if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                // 因为 bean 创建后其所依赖的 bean 一定是已经创建的
                // actualDependentBeans 不为空表示当前 bean 创建后其依赖的 bean 却没有全部创建完成，也就是说存在 
                // 循环依赖
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }
    try {
        // 根据 scope 注册 bean
        this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
        return exposedObject;
    } catch (BeanDefinitionValidationException var16) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
    }
}
```

整个方法的思路：

1. 如果是单例则需要先清除缓存

2. 实例化 bean，将 `beanDefinition` 转换为 `beanWrapper` 

   这是一个复杂的过程，大致内容如下：

   - 如果存在工厂方法则使用工厂方法进行初始化
   - 一个类有多个构造函数，每个构造函数有不同的参数，需要根据参数锁定构造函数并进行初始化
   - 如果既不存在工厂方法又不存在带有参数的构造函数，则使用默认构造函数进行 bean 的实例化

3. `MergedBeanDefinitionPostProcessor` 的应用

   bean 合并后的处理，`Autowired` 注解正是通过此方法实现注入类型的预解析。

4. 依赖处理

   在 spring 中会有循环依赖的情况，在这里的处理方式就是当创建 B 时，设计自动注入 A 的步骤时，并不是直接去再次创建 A，而是通过放入缓存中的 `ObjectFactory` 来创建实例，这样就可以解决循环依赖的问题

5. 属性填充。将所有属性填充到 bean 实例中

6. 循环依赖检查

   对于 `prototype` 的 bean，Spring 没有好的循环依赖检查方法，只有直接抛出异常。

   这个步骤中会检测已经加载的 bean 是否已经出现了依赖循环，并判断是否需要抛出异常。

7. 注册 `DisposableBean`

   如果配置了 `destroy-method`，这里需要注册以便于在销毁的时候能调用

8. 完成创建并返回

下面我们就来详细剖析一下其中过程

### 5.7.1 创建 bean 的实例

首先从 `doCreateBean` 中的 `createBeanInstance` 开始：

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // 解析 class
    Class<?> beanClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    } else {
        Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
        if (instanceSupplier != null) {
            return this.obtainFromSupplier(instanceSupplier, beanName);
        } else if (mbd.getFactoryMethodName() != null) {
            // 如果工厂方法不为空则使用工厂方法初始化策略
            return this.instantiateUsingFactoryMethod(beanName, mbd, args);
        } else {
            boolean resolved = false;
            boolean autowireNecessary = false;
            if (args == null) {
                synchronized(mbd.constructorArgumentLock) {
                    // 一个类有多个构造函数，每个构造函数有不同的参数，所以调用前需要根据参数锁定构造函数或对应的工厂方法
                    if (mbd.resolvedConstructorOrFactoryMethod != null) {
                        resolved = true;
                        autowireNecessary = mbd.constructorArgumentsResolved;
                    }
                }
            }
            if (resolved) {
                // 如果已经解析过了则使用解析好的构造函数方法，不需要再次锁定了
                if (autowireNecessary) {
                    // 构造函数自动注入
                    return this.autowireConstructor(beanName, mbd, (Constructor[])null, (Object[])null);
                } else {
                    // 使用默认构造函数构造
                    return this.instantiateBean(beanName, mbd);
                }
            } else {
                // 需要根据参数解析构造函数
                Constructor<?>[] ctors = this.determineConstructorsFromBeanPostProcessors(beanClass, beanName);
                if (ctors == null && mbd.getResolvedAutowireMode() != 3 && !mbd.hasConstructorArgumentValues() && ObjectUtils.isEmpty(args)) {
                    ctors = mbd.getPreferredConstructors();
                    return ctors != null ? this.autowireConstructor(beanName, mbd, ctors, 
                  // 默认函数构造                                                   
(Object[])null) : this.instantiateBean(beanName, mbd);
                } else {
                    // 构造函数自动注入
                    return this.autowireConstructor(beanName, mbd, ctors, args);
                }
            }
        }
    }
}
```

1. 如果在 `RootBeanDefinition` 中存在 `factoryMethodName` 属性，或者说在配置文件中配置 `factory-method`，那么 spring 会尝试使用 `instantiateUsingFactoryMethod(beanName, mbd, args);`，根据 `RootBeanDefinition` 中的配置生成 bean 实例。

2. 解析构造函数并进行构造函数实例化。spring 会根据参数及其类型来判断最终会使用哪个构造函数进行实例化。

   但是判断的过程是消耗性能的，所以采用缓存，如果已经解析了则不需要重复解析，而是直接从 `RootBeanDefinition` 中的属性 `resolvedConstructorOrFactoryMethod` 缓存的值去取，否则需要再次解析，并将解析的结果添加到 `RootBeanDefinition` 中的属性 `resolvedConstructorOrFacotryMethod` 中

#### `autowireConstructor`

对于实例的创建 spring 分成了两种情况，一种是默认的实例化，另一种是带有参数的实例化。带有参数的实例化过程相当复杂。

源码：

```java
protected BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {
    return (new ConstructorResolver(this)).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}

public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {
    BeanWrapperImpl bw = new BeanWrapperImpl();
    this.beanFactory.initBeanWrapper(bw);
    Constructor<?> constructorToUse = null;
    ConstructorResolver.ArgumentsHolder argsHolderToUse = null;
    Object[] argsToUse = null;
    // 如果 getBean 方法调用时直接指定方法参数那么直接使用
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    } else {
        // getBean 方法没有指定则从配置文件中解析
        Object[] argsToResolve = null;
        synchronized(mbd.constructorArgumentLock) {
            // 从缓存中获取
            constructorToUse = (Constructor)mbd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse != null && mbd.constructorArgumentsResolved) {
                // 从缓存中取
                argsToUse = mbd.resolvedConstructorArguments;
                if (argsToUse == null) {
                    // 配置的构造函数参数
                    argsToResolve = mbd.preparedConstructorArguments;
                }
            }
        }
        // 如果缓存中存在
        if (argsToResolve != null) {
            // 解析参数类型，如给定方法的构造函数A(int, int) 则通过此方法就会把配置中的("1", "1")，转换为 (1, 1)
            // 缓存中的值可能是原始值也可能是最终值
            argsToUse = this.resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve, true);
        }
    }
    // 没有被缓存
    if (constructorToUse == null || argsToUse == null) {
        // 需要解析构造器
        Constructor<?>[] candidates = chosenCtors;
        if (chosenCtors == null) {
            Class beanClass = mbd.getBeanClass();
            try {
                candidates = mbd.isNonPublicAccessAllowed() ? beanClass.getDeclaredConstructors() : beanClass.getConstructors();
            } catch (Throwable var26) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Resolution of declared constructors on bean Class [" + beanClass.getName() + "] from ClassLoader [" + beanClass.getClassLoader() + "] failed", var26);
            }
        }
        if (candidates.length == 1 && explicitArgs == null && !mbd.hasConstructorArgumentValues()) {
            Constructor<?> uniqueCandidate = candidates[0];
            if (uniqueCandidate.getParameterCount() == 0) {
                synchronized(mbd.constructorArgumentLock) {
                    mbd.resolvedConstructorOrFactoryMethod = uniqueCandidate;
                    mbd.constructorArgumentsResolved = true;
                    mbd.resolvedConstructorArguments = EMPTY_ARGS;
                }
                bw.setBeanInstance(this.instantiate(beanName, mbd, uniqueCandidate, EMPTY_ARGS));
                return bw;
            }
        }
        boolean autowiring = chosenCtors != null || mbd.getResolvedAutowireMode() == 3;
        ConstructorArgumentValues resolvedValues = null;
        int minNrOfArgs;
        if (explicitArgs != null) {
            minNrOfArgs = explicitArgs.length;
        } else {
            // 提取配置文件中的配置的构造函数参数
            ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            // 用于承载解析后的构造函数参数的值
            resolvedValues = new ConstructorArgumentValues();
            // 能解析到的参数个数
            minNrOfArgs = this.resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
        }
        // 排序构造器， public 构造函数优先参数数量降序，非 public 构造函数参数数量降序
        AutowireUtils.sortConstructors(candidates);
        int minTypeDiffWeight = 2147483647;  // Integer.MAX_VALUE
        Set<Constructor<?>> ambiguousConstructors = null;
        LinkedList<UnsatisfiedDependencyException> causes = null;
        Constructor[] var16 = candidates;
        int var17 = candidates.length;
        for(int var18 = 0; var18 < var17; ++var18) {
            Constructor<?> candidate = var16[var18];
            int parameterCount = candidate.getParameterCount();
            if (constructorToUse != null && argsToUse != null && argsToUse.length > parameterCount) {
                // 因为已经找到选用的构造函数或者需要的参数个数个数小于当前构造函数参数个数则终止。
                break;
            }
            if (parameterCount >= minNrOfArgs) {
                Class<?>[] paramTypes = candidate.getParameterTypes();
                ConstructorResolver.ArgumentsHolder argsHolder;
                if (resolvedValues != null) {
                    // 有参数则根据值构造对应参数类型的参数
                    try {
                        // 获取参数名称
                        String[] paramNames = ConstructorResolver.ConstructorPropertiesChecker.evaluate(candidate, parameterCount);
                        if (paramNames == null) {
                            // 获取参数名称探索器
                            ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                            if (pnd != null) {
                                // 获取指定构造函数的参数名称
                                paramNames = pnd.getParameterNames(candidate);
                            }
                        }
                        // 根据名称和数据类型创建参数持有者
                        argsHolder = this.createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames, this.getUserDeclaredConstructor(candidate), autowiring, candidates.length == 1);
                    } catch (UnsatisfiedDependencyException var27) {
                        if (this.logger.isTraceEnabled()) {
                            this.logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + var27);
                        }
                        if (causes == null) {
                            causes = new LinkedList();
                        }
                        causes.add(var27);
                        continue;
                    }
                } else {
                    if (parameterCount != explicitArgs.length) {
                        continue;
                    }
                    // 构造函数没有参数的情况
                    argsHolder = new ConstructorResolver.ArgumentsHolder(explicitArgs);
                }
                // 探测是否有不确定性的构造函数存在，例如不同构造函数的参数为父子关系
                int typeDiffWeight = mbd.isLenientConstructorResolution() ? argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes);
                // 如果它代表着当前最接近的匹配则选择作为构造函数
                if (typeDiffWeight < minTypeDiffWeight) {
                    constructorToUse = candidate;
                    argsHolderToUse = argsHolder;
                    argsToUse = argsHolder.arguments;
                    minTypeDiffWeight = typeDiffWeight;
                    ambiguousConstructors = null;
                } else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
                    if (ambiguousConstructors == null) {
                        ambiguousConstructors = new LinkedHashSet();
                        ambiguousConstructors.add(constructorToUse);
                    }
                    ambiguousConstructors.add(candidate);
                }
            }
        }
        if (constructorToUse == null) {
            if (causes == null) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Could not resolve matching constructor (hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
            }
            UnsatisfiedDependencyException ex = (UnsatisfiedDependencyException)causes.removeLast();
            Iterator var34 = causes.iterator();
            while(var34.hasNext()) {
                Exception cause = (Exception)var34.next();
                this.beanFactory.onSuppressedException(cause);
            }
            throw ex;
        }
        if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Ambiguous constructor matches found in bean '" + beanName + "' (hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " + ambiguousConstructors);
        }
        if (explicitArgs == null && argsHolderToUse != null) {
            // 将解析的构造函数放入缓存
            argsHolderToUse.storeCache(mbd, constructorToUse);
        }
    }
    Assert.state(argsToUse != null, "Unresolved constructor arguments");
    // 将构建的实例加入 BeanWrapper 中
    bw.setBeanInstance(this.instantiate(beanName, mbd, constructorToUse, argsToUse));
    return bw;
}
```

该方法很长，不符合 spring 一贯的风格，总结这个方法实现的功能：

- 构造函数参数的确定

  - 根据 `explicitArgs` 参数判断

    如果传入的参数 `explicitArgs` 不为空，这边可以直接确定参数，因为 `explicitArgs` 是调用 bean 时用户自己指定的，在 `BeanFactory` 中存在这样的方法：

    ```java
    Object getBean(String name, Object... args) throws BeansException;
    ```

    获取 bean 时不但可以指定 bean 的名称，还可以指定 bean 对应类的构造函数或者工厂方法的方法参数。

  - 缓存中获取

    如果构造函数已经记录在缓存中，可以直接拿来使用。不过要注意缓存中缓存的可能是参数的最终类型也可能是参数的初始类型，例如构造函数要求的是 `int` 类型，但是原始的参数是字符串 "1"，那么即使在缓存中得到了参数，也需要经过类型转换器的过滤以确保参数类型与对应的构造函数参数类型完全对应。

  - 配置文件获取

    分析从获取配置文件中配置的构造函数信息开始，经过之前的分析，spring 中配置文件的信息经过转换都会通过 `BeanDefinition` 实例承载，也就是参数 `mbd` 中包含，那么可以通过调用 `mbd.getConstructorArgumentValues()` 来获取配置的构造函数信息。有了构造函数信息就可以获取对应的参数值信息，获取参数值的信息包括直接指定值，如直接指定构造函数中某个值为原始类型 String 类型，或者是一个对其他 bean 的引用，这一处理委托给 `resolveConstructorArguments` 方法，并返回能解析到的参数个数。

- 构造函数的确定

  第一步已经确定了参数，接下来就是根据参数在所有构造函数中锁定对应的构造函数，匹配的方法就是根据参数个数匹配。

  匹配前需要先排序 bean 的构造方法，这样有利于迅速判断排在后面的构造函数参数个数是否符合条件。

  配置文件还支持指定参数名称进行设定参数值，这种情况就需要先确定构造函数中参数名称。

  获取参数名称可以通过**注解**和 **spring 提供的工具类 `ParameterNameDiscover`** 来获取。

- 根据确定的构造函数转换对应的参数类型

  使用 spring 中提供的类型转换器或者用户提供的自定义类型转换器进行转换。

- 构造函数不确定性的验证

  spring 最后还要做一次验证

- 根据实例化策略以及得到的构造函数及参数实例化 Bean

####  `instantiateBean`

默认构造函数注入

```java
protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        if (System.getSecurityManager() != null) {
            // 安全模式，需要认证
            beanInstance = AccessController.doPrivileged(() -> {
                return this.getInstantiationStrategy().instantiate(mbd, beanName, this);
            }, this.getAccessControlContext());
        } else {
            // 直接按照策略模式创建
            beanInstance = this.getInstantiationStrategy().instantiate(mbd, beanName, this);
        }
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        this.initBeanWrapper(bw);
        return bw;
    } catch (Throwable var5) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", var5);
    }
}
```

如果没有参数的话是一件很简单的事，直接调用实例化策略进行实例化即可。

#### 实例化策略

我们在之前已经得到了实例化的所有相关信息（构造函数，参数，Class），完全可以用最简单的反射构造实例，但是 spring 没有这么干

```java
// SimpleInstantiationStrategy.java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
    // 如果有需要覆盖或者动态替换的方法则需要用 cglib 进行动态代理，因为可以在创建代理同时将动态方法织入类中
    // 但是如果没有需要动态改变的方法，直接反射即可
    if (!bd.hasMethodOverrides()) {
        // 有需要覆盖的方法，动态代理
        Constructor constructorToUse;
        synchronized(bd.constructorArgumentLock) {
            constructorToUse = (Constructor)bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                Class<?> clazz = bd.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    if (System.getSecurityManager() != null) {
                        clazz.getClass();
                        constructorToUse = (Constructor)AccessController.doPrivileged(() -> {
                            return clazz.getDeclaredConstructor();
                        });
                    } else {
                        constructorToUse = clazz.getDeclaredConstructor();
                    }
                    bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                } catch (Throwable var9) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", var9);
                }
            }
        }
        return BeanUtils.instantiateClass(constructorToUse, new Object[0]);
    } else {
        // 直接反射
        return this.instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

```java
// CglibSubclassingInstantiationStrategy.java
public Object instantiate(@Nullable Constructor<?> ctor, @Nullable Object... args) {
	Class<?> subclass = createEnhancedSubclass(this.beanDefinition);
	Object instance;
	if (ctor == null) {
		instance = BeanUtils.instantiateClass(subclass);
	}
	else {
		try {
			Constructor<?> enhancedSubclassConstructor = subclass.getConstructor(ctor.getParameterTypes());
			instance = enhancedSubclassConstructor.newInstance(args);
		}
		catch (Exception ex) {
			throw new BeanInstantiationException(this.beanDefinition.getBeanClass(),
					"Failed to invoke constructor for CGLIB enhanced subclass [" + subclass.getName() + "]", ex);
		}
	}
	// SPR-10785: set callbacks directly on the instance instead of in the
	// enhanced class (via the Enhancer) in order to avoid memory leaks.
	Factory factory = (Factory) instance;
	factory.setCallbacks(new Callback[] {NoOp.INSTANCE,
			new LookupOverrideMethodInterceptor(this.beanDefinition, this.owner),
			new ReplaceOverrideMethodInterceptor(this.beanDefinition, this.owner)});
	return instance;
}
```

程序中，首先判断如果 `beanDefinition.getMethodOverrides()` 为空也就是用户没有使用 `replace` 或者 `lookup` 的配置方法，那么直接反射，但是如果使用了这两个特性，就需要将这两个配置提供的功能切入进去，所以就必须使用动态代理的方式将**包含两个特性所对应的逻辑的拦截增强器设置进去**，这样才可以保证在调用方法时会被相应拦截器增强，返回值为**包含拦截器的代理实例**。

### 5.7.2 记录创建的 bean 的 `ObjectFactory`

在 `docreateBean` 函数中有这样一段代码：

```java
boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
if (earlySingletonExposure) {
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
    }
    // 避免后期循环依赖，可以在 bean 初始化完成前将创建实例的 ObjectFactory 加入工厂
    this.addSingletonFactory(beanName, () -> {
        // 对 bean 再一次依赖引用，主要应用 SnartInstantiationAwareBeanPostProcessor
        // 我们知道的 AOP 就是在这里将 advice 动态织入 bean 中，若没有则直接返回 bean，不做任何处理
        return this.getEarlyBeanReference(beanName, mbd, bean);
    });
}
```

- `earlySingletonExposure`：提早曝光单例

- `mbd.isSingletonn()`：判断 bean 是否是单例

- `this.allowCircularReferences`：是否允许循环依赖，不能在配置文件中配置，可以通过编码设置：

  ```java
  BeanFactory bf = new ClassPathXmlApplication("test.xml");
  bf.setAllowBeanDefinitionOverriding(false);
  ```

- `isSingletonCurrentlyIncreation(beanName)`：该 bean 是否在创建中。在 Spring 中，有个专门的属性默认为 `DefaultSingletonBeanRegistry` 的 `singletonCurrentlyCreation` 来记录 bean 的加载状态，在 bean 开始创建前会将 `beanName` 记录在属性中，bean 创建后会将这个属性移除。

当 **bean 是单例、允许循环依赖、对应的 bean 正在创建**，那么 `earlySingletonExposure` 就为 true，那么就可以执行 `addSinglletonFactory` 操作，那么这个方法作用是什么呢：

<img src="C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220509004758027.png" alt="image-20220509004758027" style="zoom:50%;" />

> 创建 A 时首先记录 A 对应的 `beanName`，将 `beanA` 的创建工厂加入缓存中，在对 A 的属性填充也就是调用 populate 方法时又会再一次对 B 进行递归创建。
>
> 同样 B 因为存在 A 属性，实例化 B 的 populate 方法会再次初始化 A，也就是这里的 `getBean(A)`。
>
> 这个方法不是直接去实例化 A，而是先检测缓存中是否有已经创建好的 bean，或者是否已经创建好的 `ObjectFactory`，而此时对于 A 的 `ObjectFactory` 我们已经创建好了，所以不会再去向后执行，而是直接调用 `ObjectFactory` 去创建 A。

`ObjectFactory`的实现：

```java
this.addSingletonFactory(beanName, new ObjectFactory() {
    // 对 bean 再一次依赖引用，主要应用 SnartInstantiationAwareBeanPostProcessor
    // 我们知道的 AOP 就是在这里将 advice 动态织入 bean 中，若没有则直接返回 bean，不做任何处理
    return this.getEarlyBeanReference(beanName, mbd, bean);
});
```

其中 `getEarlyBeanReference` 的代码如下：

```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
        Iterator var5 = this.getBeanPostProcessors().iterator();
        while(var5.hasNext()) {
            BeanPostProcessor bp = (BeanPostProcessor)var5.next();
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor)bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}
```

Spring 处理循环依赖的解决方法：

B 中创建依赖 A 时通过 `ObjectFactory` 提供的实例化方法来中断 A 的属性填充，使 B 中持有的 A 仅仅是刚初始化并没有填充任何属性的 A，而正初始化 A 的步骤还是在最开始创建 A 时进行的，因为 A 和 B 中的 A 表示的属性地址相同，所以 A 中创建好的属性填充可以通过 B 中的 A 获取。

### 5.7.3 属性注入

那么 Spring 是如何进行属性填充的呢：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
    } else {
        if (!mbd.isSynthetic() && this.hasInstantiationAwareBeanPostProcessors()) {
            Iterator var4 = this.getBeanPostProcessors().iterator();
            while(var4.hasNext()) {
                BeanPostProcessor bp = (BeanPostProcessor)var4.next();
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                    if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                        // 没有属性填充
                        return;
                    }
                }
            }
        }
        
        // 给 InstantiationAwareBeanPostProcessors 最后一次机会在属性设置前来改变 bean
        PropertyValues pvs = mbd.hasPropertyValues() ? mbd.getPropertyValues() : null;
        int resolvedAutowireMode = mbd.getResolvedAutowireMode();
        if (resolvedAutowireMode == 1 || resolvedAutowireMode == 2) {
            MutablePropertyValues newPvs = new MutablePropertyValues((PropertyValues)pvs);
            // 根据名称自动注入
            if (resolvedAutowireMode == 1) {
                this.autowireByName(beanName, mbd, bw, newPvs);
            }
            // 根据类型自动注入
            if (resolvedAutowireMode == 2) {
                this.autowireByType(beanName, mbd, bw, newPvs);
            }
            pvs = newPvs;
        }
        
        // 后置处理器已经初始化
        boolean hasInstAwareBpps = this.hasInstantiationAwareBeanPostProcessors();
        // 需要依赖检查
        boolean needsDepCheck = mbd.getDependencyCheck() != 0;
        PropertyDescriptor[] filteredPds = null;
        if (hasInstAwareBpps) {
            if (pvs == null) {
                pvs = mbd.getPropertyValues();
            }
            Iterator var9 = this.getBeanPostProcessors().iterator();
            while(var9.hasNext()) {
                BeanPostProcessor bp = (BeanPostProcessor)var9.next();
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                    PropertyValues pvsToUse = ibp.postProcessProperties((PropertyValues)pvs, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        if (filteredPds == null) {
                            filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                        }
                        // 对所有需要依赖检查的属性进行后置处理
                        pvsToUse = ibp.postProcessPropertyValues((PropertyValues)pvs, filteredPds, bw.getWrappedInstance(), beanName);
                        if (pvsToUse == null) {
                            return;
                        }
                    }
                    pvs = pvsToUse;
                }
            }
        }
        if (needsDepCheck) {
            if (filteredPds == null) {
                filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            }
            // 依赖检查
            this.checkDependencies(beanName, mbd, filteredPds, (PropertyValues)pvs);
        }
        if (pvs != null) {
            // 将属性应用到 bean 中
            this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
        }
    }
}
```

流程：

- `InstantiationAwareBeanPostProcessor` 处理器的 `postProcessAfterInstantiation` 方法调用，控制程序是否继续进行属性填充
- 根据注入类型 `byName/byType`，提取依赖的 bean，统一存入 `PropertyValues`
- 应用 `InstantiationAwareBeanPostProcessor` 处理器的 `postProcessPropertyValues` 方法，对属性获取完毕填充前对属性再次处理。
- 将所有 `PropertyValues` 中的属性填充到 `BeanWrapper` 中

#### `autowireByName`

根据名称自动注入：

```java
protected void autowireByName(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    // 寻找 bw 中需要注入的属性
    String[] propertyNames = this.unsatisfiedNonSimpleProperties(mbd, bw);
    for(int i = 0; i < propertyNames.length; ++i) {
        String propertyName = propertyNames[i];
        if (this.containsBean(propertyName)) {
            // 递归初始化相关的 bean
            Object bean = this.getBean(propertyName);
            pvs.add(propertyName, bean);
            // 注册依赖
            this.registerDependentBean(propertyName, beanName);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Added autowiring by name from bean name '" + beanName + "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
            }
        } else if (this.logger.isTraceEnabled()) {
            this.logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName + "' by name: no matching bean found");
        }
    }
}
```

在传入的参数中找到已经加载的 bean，递归实例化，最终放入内存中

#### `autowireByType`

根据类型自动注入：

```java
protected void autowireByType(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    TypeConverter converter = this.getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }
    Set<String> autowiredBeanNames = new LinkedHashSet(4);
    // 寻找 bw 中需要依赖注入的属性
    String[] propertyNames = this.unsatisfiedNonSimpleProperties(mbd, bw);
    for(int i = 0; i < propertyNames.length; ++i) {
        String propertyName = propertyNames[i];
        try {
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            if (Object.class != pd.getPropertyType()) {
                // 探测指定属性的 set 方法
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
                DependencyDescriptor desc = new AbstractAutowireCapableBeanFactory.AutowireByTypeDependencyDescriptor(methodParam, eager);
                // 解析指定 beanName 属性匹配的值，并把解析到的属性名称存储在 autowiredBeanNames 中，
                // 当属性存在多个封装 bean 时如：@Autowired private List<A> aList
                // 会找到所有匹配 A 类型的 bean 并将其注入
                Object autowiredArgument = this.resolveDependency(desc, beanName, autowiredBeanNames, (TypeConverter)converter);
                if (autowiredArgument != null) {
                    pvs.add(propertyName, autowiredArgument);
                }
                Iterator var17 = autowiredBeanNames.iterator();
                while(var17.hasNext()) {
                    String autowiredBeanName = (String)var17.next();
                    // 注册依赖
                    this.registerDependentBean(autowiredBeanName, beanName);
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" + propertyName + "' to bean named '" + autowiredBeanName + "'");
                    }
                }
                autowiredBeanNames.clear();
            }
        } catch (BeansException var19) {
            throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, var19);
        }
    }
}
```

与根据名称自动匹配相同，根据类型匹配第一步就是寻找 `bw(BeanWrapper)` 中需要依赖注入的属性，然后遍历这些属性并寻找类型匹配的 bean，最复杂的就是寻找类型匹配的 bean。同时，Spring 中提供了对集合类型注入的支持：

```java
@Autowired
private List<Test> tests;
```

会把所有 Test 类型注入到 tests 中，因为如此，在 `autowireByType` 方法中，新建了局部遍历 `autowiredBeanNames`，用于存储所有依赖的 bean，如果只是非集合的属性注入来说，此属性没用。

寻找类型匹配的逻辑实现封装在 `resolveDependency` 方法中

```java
@Nullable
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName, @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
    descriptor.initParameterNameDiscovery(this.getParameterNameDiscoverer());
    if (Optional.class == descriptor.getDependencyType()) {
        return this.createOptionalDependency(descriptor, requestingBeanName);
    } else if (ObjectFactory.class != descriptor.getDependencyType() && ObjectProvider.class != descriptor.getDependencyType()) {
        if (javaxInjectProviderClass == descriptor.getDependencyType()) {
            // javaxInjectPorviderClass 类注入的特殊处理
            return (new DefaultListableBeanFactory.Jsr330Factory()).createDependencyProvider(descriptor, requestingBeanName);
        } else {
            Object result = this.getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(descriptor, requestingBeanName);
            if (result == null) {
                // 通用处理逻辑
                result = this.doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
            }
            return result;
        }
    } else {
        return new DefaultListableBeanFactory.DependencyObjectProvider(descriptor, requestingBeanName);
    }
}

@Nullable
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName, @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    Object var7;
    try {
        Object shortcut = descriptor.resolveShortcut(this);
        if (shortcut == null) {
            Class<?> type = descriptor.getDependencyType();
            // 支持 @Value
            Object value = this.getAutowireCandidateResolver().getSuggestedValue(descriptor);
            Object var23;
            if (value != null) {
                if (value instanceof String) {
                    String strVal = this.resolveEmbeddedValue((String)value);
                    BeanDefinition bd = beanName != null && this.containsBean(beanName) ? this.getMergedBeanDefinition(beanName) : null;
                    value = this.evaluateBeanDefinitionString(strVal, bd);
                }
                TypeConverter converter = typeConverter != null ? typeConverter : this.getTypeConverter();
                try {
                    var23 = converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
                    return var23;
                } catch (UnsupportedOperationException var18) {
                    Object var25 = descriptor.getField() != null ? converter.convertIfNecessary(value, type, descriptor.getField()) : converter.convertIfNecessary(value, type, descriptor.getMethodParameter());
                    return var25;
                }
            }
            
            
            Object multipleBeans = this.resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
            if (multipleBeans != null) {
                var23 = multipleBeans;
                return var23;
            }
            Map<String, Object> matchingBeans = this.findAutowireCandidates(beanName, type, descriptor);
            String autowiredBeanName;
            if (matchingBeans.isEmpty()) {
                if (this.isRequired(descriptor)) {
                    this.raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
                }
                autowiredBeanName = null;
                return autowiredBeanName;
            }
            Object instanceCandidate;
            Object result;
            if (matchingBeans.size() > 1) {
                autowiredBeanName = this.determineAutowireCandidate(matchingBeans, descriptor);
                if (autowiredBeanName == null) {
                    if (!this.isRequired(descriptor) && this.indicatesMultipleBeans(type)) {
                        result = null;
                        return result;
                    }
                    result = descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
                    return result;
                }
                instanceCandidate = matchingBeans.get(autowiredBeanName);
            } else {
                Entry<String, Object> entry = (Entry)matchingBeans.entrySet().iterator().next();
                autowiredBeanName = (String)entry.getKey();
                instanceCandidate = entry.getValue();
            }
            if (autowiredBeanNames != null) {
                autowiredBeanNames.add(autowiredBeanName);
            }
            if (instanceCandidate instanceof Class) {
                instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
            }
            result = instanceCandidate;
            if (instanceCandidate instanceof NullBean) {
                if (this.isRequired(descriptor)) {
                    this.raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
                }
                result = null;
            }
            if (!ClassUtils.isAssignableValue(type, result)) {
                throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
            }
            Object var14 = result;
            return var14;
        }
        var7 = shortcut;
    } finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
    return var7;
}

@Nullable
private Object resolveMultipleBeans(DependencyDescriptor descriptor, @Nullable String beanName, @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) {
    Class<?> type = descriptor.getDependencyType();
    if (descriptor instanceof DefaultListableBeanFactory.StreamDependencyDescriptor) {
        Map<String, Object> matchingBeans = this.findAutowireCandidates(beanName, type, descriptor);
        if (autowiredBeanNames != null) {
            autowiredBeanNames.addAll(matchingBeans.keySet());
        }
        Stream<Object> stream = matchingBeans.keySet().stream().map((name) -> {
            return descriptor.resolveCandidate(name, type, this);
        }).filter((bean) -> {
            return !(bean instanceof NullBean);
        });
        if (((DefaultListableBeanFactory.StreamDependencyDescriptor)descriptor).isOrdered()) {
            stream = stream.sorted(this.adaptOrderComparator(matchingBeans));
        }
        return stream;
    } else {
        Class valueType;
        Map matchingBeans;
        Class elementType;
        // 数组类型
        if (type.isArray()) {
            elementType = type.getComponentType();
            ResolvableType resolvableType = descriptor.getResolvableType();
            valueType = resolvableType.resolve(type);
            if (valueType != type) {
                elementType = resolvableType.getComponentType().resolve();
            }
            if (elementType == null) {
                return null;
            } else {
                matchingBeans = this.findAutowireCandidates(beanName, elementType, new DefaultListableBeanFactory.MultiElementDescriptor(descriptor));
                if (matchingBeans.isEmpty()) {
                    return null;
                } else {
                    if (autowiredBeanNames != null) {
                        autowiredBeanNames.addAll(matchingBeans.keySet());
                    }
                    TypeConverter converter = typeConverter != null ? typeConverter : this.getTypeConverter();
                    Object result = converter.convertIfNecessary(matchingBeans.values(), valueType);
                    if (result instanceof Object[]) {
                        Comparator<Object> comparator = this.adaptDependencyComparator(matchingBeans);
                        if (comparator != null) {
                            Arrays.sort((Object[])((Object[])result), comparator);
                        }
                    }
                    return result;
                }
            }
        } else if (Collection.class.isAssignableFrom(type) && type.isInterface()) {
            // Collection 类型
            elementType = descriptor.getResolvableType().asCollection().resolveGeneric(new int[0]);
            if (elementType == null) {
                return null;
            } else {
                Map<String, Object> matchingBeans = this.findAutowireCandidates(beanName, elementType, new DefaultListableBeanFactory.MultiElementDescriptor(descriptor));
                if (matchingBeans.isEmpty()) {
                    return null;
                } else {
                    if (autowiredBeanNames != null) {
                        autowiredBeanNames.addAll(matchingBeans.keySet());
                    }
                    TypeConverter converter = typeConverter != null ? typeConverter : this.getTypeConverter();
                    Object result = converter.convertIfNecessary(matchingBeans.values(), type);
                    if (result instanceof List && ((List)result).size() > 1) {
                        Comparator<Object> comparator = this.adaptDependencyComparator(matchingBeans);
                        if (comparator != null) {
                            ((List)result).sort(comparator);
                        }
                    }
                    return result;
                }
            }
        } else if (Map.class == type) {
            // Map 类型
            ResolvableType mapType = descriptor.getResolvableType().asMap();
            Class<?> keyType = mapType.resolveGeneric(new int[]{0});
            if (String.class != keyType) {
                return null;
            } else {
                valueType = mapType.resolveGeneric(new int[]{1});
                if (valueType == null) {
                    return null;
                } else {
                    matchingBeans = this.findAutowireCandidates(beanName, valueType, new DefaultListableBeanFactory.MultiElementDescriptor(descriptor));
                    if (matchingBeans.isEmpty()) {
                        return null;
                    } else {
                        if (autowiredBeanNames != null) {
                            autowiredBeanNames.addAll(matchingBeans.keySet());
                        }
                        return matchingBeans;
                    }
                }
            }
        } else {
            return null;
        }
    }
}
```

首先尝试使用解析器进行解析，如果解析器没有成功解析，可能是使用默认的解析器没有做任何处理，或是用自定义的解析器，对于集合并不在解析范围，所以对不同类型进行不同情况处理。

#### `applyPropertyValues`

程序运行到这里虽然注入属性都获取了，但还是以 `PropertyValue` 形式存在，还没有应用到已经实例化的 bean 中，而 `applyPropertyValues` 就是实例化 bean 的

`appliPropertyValues`代码：

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (!pvs.isEmpty()) {
        if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
            ((BeanWrapperImpl)bw).setSecurityContext(this.getAccessControlContext());
        }
        MutablePropertyValues mpvs = null;
        List original;
        if (pvs instanceof MutablePropertyValues) {
            mpvs = (MutablePropertyValues)pvs;
            if (mpvs.isConverted()) {
                try {
                    bw.setPropertyValues(mpvs);
                    return;
                } catch (BeansException var18) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Error setting property values", var18);
                }
            }
            original = mpvs.getPropertyValueList();
        } else {
            original = Arrays.asList(pvs.getPropertyValues());
        }
        TypeConverter converter = this.getCustomTypeConverter();
        if (converter == null) {
            converter = bw;
        }
        // 获取对应的解析器
        BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, (TypeConverter)converter);
        List<PropertyValue> deepCopy = new ArrayList(original.size());
        boolean resolveNecessary = false;
        Iterator var11 = original.iterator();
        while(true) {
            while(var11.hasNext()) {
                PropertyValue pv = (PropertyValue)var11.next();
                // 遍历属性，将属性转换为对应类的对应属性的类型
                if (pv.isConverted()) {
                    deepCopy.add(pv);
                } else {
                    String propertyName = pv.getName();
                    Object originalValue = pv.getValue();
                    if (originalValue == AutowiredPropertyMarker.INSTANCE) {
                        Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
                        if (writeMethod == null) {
                            throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
                        }
                        originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
                    }
                    Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
                    Object convertedValue = resolvedValue;
                    boolean convertible = bw.isWritableProperty(propertyName) && !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
                    if (convertible) {
                        convertedValue = this.convertForProperty(resolvedValue, propertyName, bw, (TypeConverter)converter);
                    }
                    if (resolvedValue == originalValue) {
                        if (convertible) {
                            pv.setConvertedValue(convertedValue);
                        }
                        deepCopy.add(pv);
                    } else if (convertible && originalValue instanceof TypedStringValue && !((TypedStringValue)originalValue).isDynamic() && !(convertedValue instanceof Collection) && !ObjectUtils.isArray(convertedValue)) {
                        pv.setConvertedValue(convertedValue);
                        deepCopy.add(pv);
                    } else {
                        resolveNecessary = true;
                        deepCopy.add(new PropertyValue(pv, convertedValue));
                    }
                }
            }
            if (mpvs != null && !resolveNecessary) {
                mpvs.setConverted();
            }
            try {
                bw.setPropertyValues(new MutablePropertyValues(deepCopy));
                return;
            } catch (BeansException var19) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Error setting property values", var19);
            }
        }
    }
}
```

### 5.7.4 初始化 bean

bean 配置时可以设置 bean 的 `init-method` 属性，作用是在 bean 实例化前调用 `init-method` 来根据用户业务进行相应的实例化

![image-20220509015115371](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220509015115371.png)

源码：

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(() -> {
            // 激活 Aware 方法
            this.invokeAwareMethods(beanName, bean);
            return null;
        }, this.getAccessControlContext());
    } else {
        // 对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
        this.invokeAwareMethods(beanName, bean);
    }
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 应用后处理器
        wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
    }
    try {
        // 激活用户自定义的 init 方法
        this.invokeInitMethods(beanName, wrappedBean, mbd);
    } catch (Throwable var6) {
        throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 后处理器应用
        wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

#### 激活 Aware 方法

Spring 中提供一些 Aware 相关的接口，实现这些 Aware 接口的 bean 在初始化后，可以获得一些对应的资源，例如实现 `BeanFactoryAware` 的 bean 初始后，Spring 会直接注入 `BeanFactory` 实例。

用法：

1. 定义普通 bean

   ```java
   public class Hello {
       public void sayHi() {
           System.out.println("Hi");
       }
   }
   ```

2. 定义 `BeanFactoryAware` 类型的 bean

   ```java
   public class Test implements BeanFactoryAware {
   	private BeanFactory bf;
   	
       // 声明 bean 时 Spring 会自动注入 beanFactory
       @Override
       public void setBeanFactory(BeanFactory bf) throws BeansException {
           this.bf = bf;
       }
       
       public void test() {
           Hello h = (Hello) bf.getBean("hello");
           hello.sayHi();
       }
   }
   ```

3. 测试

   ```java
   Test test = (Test) bf.getBean("test");
   test.test();
   ```

   输出：

   ```java
   hi
   ```

   

按照上面的方法我们可以获取到 Spring 中的 `BeanFactory`，可以根据 bf 获取所有 bean，以及相关设置。

实现：

```java
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware)bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = this.getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware)bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware)bean).setBeanFactory(this);
        }
    }
}
```

#### 处理器的应用

`BeanPostProcessor` 后置处理器是 Spring 必不可少的亮点，它给用户充足的权限去修改或者扩展 Spring，除了 `BeanPostProcessors` 还有很多其他的 `PostProcessor`。

调用客户自定义初始化方法前以及调用自定义初始化方法后会分别调用 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 和 `postProcessAfterInitialization`，用户可以根据自己的业务设置相应的逻辑。

![image-20220509020704222](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220509020704222.png)

源码：

```java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
    // 实例化前
    Object result = existingBean;
    Object current;
    for(Iterator var4 = this.getBeanPostProcessors().iterator(); var4.hasNext(); result = current) {
        BeanPostProcessor processor = (BeanPostProcessor)var4.next();
        current = processor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result;
        }
    }
    return result;
}
```

```java
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {
    Object result = existingBean;
    Object current;
    for(Iterator var4 = this.getBeanPostProcessors().iterator(); var4.hasNext(); result = current) {
        // 实例化后
        BeanPostProcessor processor = (BeanPostProcessor)var4.next();
        current = processor.postProcessAfterInitialization(result, beanName);
        if (current == null) {
            return result;
        }
    }
    return result;
}
```

#### 激活自定义的 `init` 方法

客户自定义的初始化方法除了 `init-method` 外，还有使自定义的 bean 实现 `InitializingBean` 接口，并在 `afterPropertiesSet` 中实现自己的初始化业务逻辑。

执行顺序是先执行 `afterOrioertuesSet` 后执行 `init-method`

源码：

```java
protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd) throws Throwable {
    // 会先检查是否是 InitializingBean,如果是则执行 afterPropertiesSet 方法
    boolean isInitializingBean = bean instanceof InitializingBean;
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged(() -> {
                    ((InitializingBean)bean).afterPropertiesSet();
                    return null;
                }, this.getAccessControlContext());
            } catch (PrivilegedActionException var6) {
                throw var6.getException();
            }
        } else {
            // 属性初始化后的处理
            ((InitializingBean)bean).afterPropertiesSet();
        }
    }
    if (mbd != null && bean.getClass() != NullBean.class) {
        // 调用 init-method
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) && (!isInitializingBean || !"afterPropertiesSet".equals(initMethodName)) && !mbd.isExternallyManagedInitMethod(initMethodName)) {
            this.invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```

#### 注册 `DisposableBean`

Spring 提供了销毁方法的扩展入口，除了 `destroy-method` 设置之外，还可以注册后处理器 `DestructionAwareBeanPostProcessor` 来统一处理 bean 的销毁方法

源码：

```java
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
    AccessControlContext acc = System.getSecurityManager() != null ? this.getAccessControlContext() : null;
    if (!mbd.isPrototype() && this.requiresDestruction(bean, mbd)) {
        if (mbd.isSingleton()) {
            // 单例模型下注册需要销毁的 bean，此方法中会处理实现 DisposableBean 的 bean
            // 对所有的 bean 使用 DestructionAwareBeanPostProcessor 处理
            // DisposableBean DestructionAwareBeanPostProcessors
            this.registerDisposableBean(beanName, new DisposableBeanAdapter(bean, beanName, mbd, this.getBeanPostProcessors(), acc));
        } else {
            // 自定义 scope 的处理
            Scope scope = (Scope)this.scopes.get(mbd.getScope());
            if (scope == null) {
                throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
            }
            scope.registerDestructionCallback(beanName, new DisposableBeanAdapter(bean, beanName, mbd, this.getBeanPostProcessors(), acc));
        }
    }
}
```

# 容器的功能扩展

前面我们一直以 `BeanFactory` 接口以及它的默认实现类 `XmlBeanFactory` 为例进行分析，但是 Spring 还提供了另一个接口 `ApplicationContext`，用于扩展 `BeanFactory` 的功能。

`ApplicationContext` 和 `BeanFactory` 都是用来加载 Bean 的，但是 `ApplicationContext` 提供了更多的扩展功能。它继承了 `BeanFactory`接口，并且扩充了更多功能，通常建议比 `BeanFactory` 优先，除非在一些限制场合，例如字节长度对内存有很大影响时。

用两个不同类去加载配置文件在写法上的不同：

- 使用 `BeanFactory` 方式加载 XML

  ```java
  BeanFactory bf = new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
  ```

- 使用 `ApplicationContext` 加载 XML

  ```java
  ApplicationContext bf = new ClassPathXmlApplicationContext("beanFactory.xml");
  ```

以 `ClassPathXmlApplication` 作为切入点，对整体功能进行分析。

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[]{configLocation}, true, (ApplicationContext)null);
}

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
    super(parent);
    this.setConfigLocations(configLocations);  // !
    if (refresh) {
        this.refresh();  // !
    }
}
```

设置路径是必不可少的，`ClassPathXmlApplicationContext` 可以将配置文件路径以数组方式传入，`ClassPathXmlApplicationContext` 可以对数组进行解析并加载。而对于解析和功能实现都在 `refresh()` 中实现。

## 6.1 设置配置路径

`ClassPathXmlApplicationContext` 中支持多个配置文件以数组方式同时传入。

```java
public void setConfigLocations(@Nullable String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for(int i = 0; i < locations.length; ++i) {
            // 解析给定路径
            this.configLocations[i] = this.resolvePath(locations[i]).trim();
        }
    } else {
        this.configLocations = null;
    }
}
```

主要用于解析给定的路径数组，如果数组中包含特殊符号如 `${var}`，那么在 `resolvePath` 中会搜寻匹配的系统变量并替换。

## 6.2 扩展功能

设置路径后，可以根据路径做配置文件的解析及各种功能的实现，`refresh` 函数包含了几乎 `ApplicationContext` 的全部功能，而且此函数逻辑非常清晰：

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized(this.startupShutdownMonitor) {
        // 准备上下文刷新
        this.prepareRefresh();
        
        // 初始化 BeanFactory 并进行 XML 文件读取
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        // 对 BeanFactory 进行各种功能填充
        this.prepareBeanFactory(beanFactory);
        try {
            // 子类覆盖方法做额外处理
            this.postProcessBeanFactory(beanFactory);
            
            // 激活各种 BeanFactory 处理器
            this.invokeBeanFactoryPostProcessors(beanFactory);
            
            // 注册拦截 Bean 创建的 Bean 处理器，这里只是注册，真正调用是在 getBean 时
            this.registerBeanPostProcessors(beanFactory);
            
            // 初始化 Message源，不同语言的消息体，国际化处理
            this.initMessageSource();
            
            // 初始化应用消息广播器，放入 "applicationEventMulticaster" bean 中
            this.initApplicationEventMulticaster();
            
            // 留给子类初始化其他 bean
            this.onRefresh();
            
            // 在所有注册的 bean 中查找 Listener bean，注册到消息广播器中
            this.registerListeners();
            
            // 初始化剩下的单实例（非惰性）
            this.finishBeanFactoryInitialization(beanFactory);
            
            // 完成刷新过程，通知生命周期处理器 lifecycleProcessor 刷新过程，同时发出 ContextRefreshEvent 通知别人
            this.finishRefresh();
        } catch (BeansException var9) {
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
            }
            
            // 销毁已经创建的悬而未决的 bean
            this.destroyBeans();
            
            // 重置 active 标记
            this.cancelRefresh(var9);
            throw var9;
        } finally {
            // 重置缓存
            this.resetCommonCaches();
        }
    }
}
```

步骤及功能：

1. 初始化前的准备工作。例如对系统属性或者环境变量进行准备和验证。

   它可以在 Spring 启动时对这些必需的变量进行存在性验证。

2. 初始化 `BeanFactory`，进行 XML 文件读取

   这一步会复用 `BeanFactory` 中配置文件读取解析和其他功能，这一步之后 `ClassPathXmlApplicationContext` 实际上就已经包含了 `BeanFactory` 提供的功能，也就是可以进行 bean 的提取等操作了。

3. 对 `BeanFactory` 进行各种功能填充

   `@Qualifier` 和 `@Autowired` 这两个注解就是在这个步骤中增加的支持。

4. 子类覆盖方法做额外的处理

   Spring 给程序员提供了开放式的架构，使程序员可以根据业务需要扩展已经存在的功能，例如这里的 `postProcessBeanFactory` 就可以方便程序员在业务上进一步扩展。

5. 激活各种 `BeanFactory` 处理器

6. 注册拦截 bean 创建的 bean 处理器，这里只是注册，真正调用是在 `getBean` 时候

7. 为上下文初始化 Message 源，对不同语言的国际化处理。

8. 初始化应用消息广播器，放入 `applicationEventMulticaster` bean 中

9. 留给子类初始化其他 bean

10. 所有注册的 bean 中查找 listener bean，注册到消息广播器中

11. 初始化剩下的单实例

12. 完成刷新过程，通知 生命周期处理器 `lifecycleProcessor ` 刷新过程，同时发出 `ContextRefreshEvent` 通知别人

## 6.3 环境准备 `prepareRefresh`

`prepareRefresh` 主要是做些准备工作，例如对系统属性和环境变量的初始化和验证。

```java
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);
    if (this.logger.isDebugEnabled()) {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Refreshing " + this);
        } else {
            this.logger.debug("Refreshing " + this.getDisplayName());
        }
    }
    
    // 子类覆盖
    this.initPropertySources();
    
    // 验证需要的属性文件是否都已经放入环境中
    this.getEnvironment().validateRequiredProperties();
    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet(this.applicationListeners);
    } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }
    this.earlyApplicationEvents = new LinkedHashSet();
}
```

1. `initPropertySources()` 是空方法，正符合 Spring 的开放式结构，给用户最大扩展 Spring 的能力，用户可以根据需要重写此方法，进行个性化的属性处理和设置。

2. `validateRequiredProperties()` 是对属性进行验证。

   举例：工程运行过程中需要用到某个设置例如 VAR，从系统环境变量获得，如果用户没有在系统环境变量中配置这个参数，那么工程可能不会工作。我们可以对源码进行扩展，自定义类：

   ```java
   public class MyClassPathXmlApplicationContext extends ClassPathXmlApplicationContext {
       public MyClassPathXmlApplicationContext(String... configLocations) {
           super(configLocations);
       }
       
       protected void initPropertySources() {
           // 添加验证要求
           getEnvironment().setRequiredProperties("VAR");
       }
   }
   ```

   程序走到 `this.getEnvironment().validateRequiredProperties();` 时发现系统缺少 VAR 常量，就会抛出异常。

## 6.4 加载 `BeanFactory`：`obtainFreshBeanFactory`

`obtainFreshBeanFactory` 从字面上来看是获取 `BeanFactory`，`ApplicationContext` 是对 `BeanFactory` 的扩展，不但包含了 bf 基本的功能，还在其之上添加了大量的扩展应用，而 `obtainFreshBeanFactory` 就是实现 `BeanFactory` 的地方，经过这个函数后 `ApplicationContext` 就具有了 `BeanFactory` 全部功能。

````java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 初始化 BeanFactory，并进行 XML 文件读取，将得到的 BeanFactory 记录在当前实体属性中
    this.refreshBeanFactory();
    // 返回 bf 属性
    return this.getBeanFactory();
}

// 方法中将核心实现委托给了 refreshBeanFactory:
protected final void refreshBeanFactory() throws BeansException {
    if (this.hasBeanFactory()) {
        this.destroyBeans();
        this.closeBeanFactory();
    }
    try {
        // 创建 DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = this.createBeanFactory();
        
        // 为了序列化指定 id，如果需要，这个 beanFactory 从 id 反序列化到 BeanFactory 对象
        beanFactory.setSerializationId(this.getId());
        // 定制 beanFactory，设置相关属性，包括是否允许覆盖同名称的不同定义的对象以及循环依赖
        // 以及设置 @Autowired 和 @Qualifier 注解解析器 QualifierAnnotationAutowireCandidateResolver
        this.customizeBeanFactory(beanFactory);
        
        // 初始化 DocumentReader，并进行 XML 文件读取及解析
        this.loadBeanDefinitions(beanFactory);
        this.beanFactory = beanFactory;
    } catch (IOException var2) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var2);
    }
}
````

步骤：

1. 创建 `DefaultListableBeanFactory`

   `DefaultListableBeanFactory` 是容器的基础，必须首先要实例化。

2. 指定序列化 ID

3. 定制 `BeanFactory`

4. 加载 `BeanDefinition`

5. 使用全局变量记录 `BeanFactory` 类实例

### 6.4.1 定制 `beanFactory`：`customizeBeanFactory(beanFactory)`

这里已经开始对 `BeanFactory` 进行扩展，在基本的容器基础上，增加了是否允许覆盖、是否允许扩展的设置，并提供了注解 `@Qualifier` 和 `@Autowired` 的支持

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    // 如果 allowBeanDefinitionOverriding 不为空，设置给 beanFactory 对象相关属性
    // 含义：是否允许覆盖同名称的不同定义的对象
    if (this.allowBeanDefinitionOverriding != null) {
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // 是否允许 bean 之间存在循环依赖
    if (this.allowCircularReferences != null) {
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```

对于是否允许覆盖和允许依赖的设置这里只是判断是否为空，如果不为空要进行设置，但是没有看到在哪里进行设置，我们自己实现的话需要子类覆盖方法：

```java
public class MyClassPathXmlApplicationContext extends ClassPathXmlApplicationContext {
    protected void customizeBeanFactory(DefaultListableBeanFactory bf) {
        super.setAllowBeanDefinitionOverriding(false);
        super.setAllowCircularReferences(false);
        super.customizeBeanFactory(bf);
    }
}
```

### 6.4.2 加载 `BeanDefinition`：`loadBeanDefinitions`

由于需要读取 XML 文件，所以需要 `XmlBeanDefinitionReader` 类的支持：

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 创建 XmlBeanDefinitionReader 对象
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    
    // 进行环境变量设置
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    
    // 对 beanDefinitionReader 进行设置
    this.initBeanDefinitionReader(beanDefinitionReader);
    
    // 正式开始读取
    this.loadBeanDefinitions(beanDefinitionReader);
}

// 读取配置文件
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = this.getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = this.getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```

复用以前读取配置文件的代码即可，就是 `BeanFactory` 的套路，所以 `XmlBeanDefinitionReader` 读取的 `BeanDefinitionHolder` 都会注册到之前初始化的 `DefaultListableBeanFactory` 中，经过此步骤，`beanFactory` 已经包含了所有解析好的配置。

##  6.5 功能扩展 `prepareBeanFactory(beanFactory)`

进入 `prepareBeanFactory(beanFactory)` 之前，Spring 已经完成了配置解析，而 `ApplicationContext` 在功能上的扩展也由此展开

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置 bf 的类加载器为当前 context 的类加载器
    beanFactory.setBeanClassLoader(this.getClassLoader());
    
    // 设置 bf 的表达式语言处理器，Spring3 增加了表达式语言的支持，默认可以用 #{bean.xxx} 的实行来调用相关属性
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    
    // 为 bf 增加一个 PropertyEditor，对 bean 的属性等设置管理的一个工具
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, this.getEnvironment()));
    
    // 添加 bean 后置处理器
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    
    // 设置几个忽略自动装配的接口
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    
    // 设置几个自动装配的特殊规则
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
    
    // 添加对 AspectJ 的支持
    if (beanFactory.containsBean("loadTimeWeaver")) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
    
    // 添加默认的系统环境 bean
    if (!beanFactory.containsLocalBean("environment")) {
        beanFactory.registerSingleton("environment", this.getEnvironment());
    }
    if (!beanFactory.containsLocalBean("systemProperties")) {
        beanFactory.registerSingleton("systemProperties", this.getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean("systemEnvironment")) {
        beanFactory.registerSingleton("systemEnvironment", this.getEnvironment().getSystemEnvironment());
    }
}
```

函数在几个方面的扩展：

- 增加对 `SqEL` 语言的支持
- 增加对属性编辑器的支持
- 增加对一些内置类，例如 `EnvironmentAware、MessageSourceAware` 的信息注入
- 设置了依赖功能可忽略的接口
- 注册一些固定依赖的属性
- 增加 `AspectJ` 的支持
- 将相关环境变量以及属性注册以单例模式注入

### 6.5.1 增加 `SqEL` 语言的支持

`SqEL`（Spring Expression Language），能在运行时构建复杂表达式，存取对象图属性，对象方法调用等，能与 Spring 功能完美整合。

使用：

```xml
<bean id="saxophone" value="com.xxx"/>
<bean>
	<property name="intrument" value="#{saxophone}"/>
</bean>
```

等价于：

```xml
<bean id="saxophone" value="com.xxx"/>
<bean>
	<property name="intrument" ref="saxophone"/>
</bean>
```

在源码中通过 `beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));` 就可以对 `SqEL` 进行解析了，那么注册解析器后 Spring 什么时候调用这个解析器进行解析呢？

Spring 在 bean 初始化时会有属性填充，这一步 Spring 会调用 `AbstractAutowireCapableBeanFactory` 类的 `applyPropertyValues` 函数来完成功能。在这个函数中，会通过构造 `BeanDefinitionValueResolver` 类型实例 `valueResolver` 来进行属性值的解析。

### 6.5.2 增加属性注册编辑器

在 Spring 依赖注入时可以把普通属性注入进来，但是像 Date 类型的就无法识别，例如：

```java
public class UserManager {
    private Date date;
    public Date getDate() {
        return date;
    }
    public void setDate(Date date) {
        this.date = date;
    }
    @Override
    public String toString() {
        return "UserManager{" +
                "date=" + date +
                '}';
    }
}
```

```xml
<!-- 如果要对 date 属性进行注入 -->
<bean id="userManager" class="com.spring5.UserManager">
    <property name="date">
        <value>2022-05-12</value>
    </property>
</bean>
```

结果：

![image-20220512003920154](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220512003920154.png)

Spring 提供两种解决方法：

#### 使用自定义属性编辑器

通过继承 `PropertyEditorSupport` 重写 `setAsText` 方法

1. 编写自定义的属性编辑器

   ```java
   public class DatePropertyEditor extends PropertyEditorSupport {
       private String format = "yyy-MM-dd";
   
       public void setFormat(String format) {
           this.format = format;
       }
   
       @Override
       public void setAsText(String text) throws IllegalArgumentException {
           System.out.println("text: " + text);
           SimpleDateFormat sdf = new SimpleDateFormat(format);
           try {
               Date date = sdf.parse(text);
               this.setValue(date);
           } catch (ParseException e) {
               e.printStackTrace();
           }
       }
   }
   ```

2. 将自定义属性编辑器注册到 Spring 中

   ```xml
   <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
       <property name="customEditors">
           <map>
               <entry key="java.util.Date">
                   <bean class="com.test.DatePropertyEditor">
                       <property name="format" value="yyyy-MM-dd" />
                   </bean>
               </entry>
           </map>
       </property>
   </bean>
   ```

	在配置文件中引入类型为 `org.springframework.beans.factory.config.CustomEditorConfigurer` 的 bean，并在属性 `customEditors` 中加入自定义的属性编辑器，其中 key 为属性编辑器对应的类型，通过这样的配置，当 Spring 注入 bean 属性时一旦遇到 `java.util.Date` 类型的属性会自动调用自定义的 `DatePropertyEditor` 解析器进行解析。

#### 2. 注入 Spring 自带的属性编辑器 `CustomDateEditor`

1. 定义属性编辑器

   ```java
   public class DatePropertyEditorRegistrar implements PropertyEditorRegistrar {
       @Override
       public void registerCustomEditors(PropertyEditorRegistry propertyEditorRegistry) {
           propertyEditorRegistry.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
       }
   }
   ```

2. 注册到 Spring 中

   ```xml
   <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
       <property name="propertyEditorRegistrars">
           <list>
               <bean class="com.test.DatePropertyEditorRegistrar" />
           </list>
       </property>
   </bean>
   ```

   通过在配置文件中将自定义的 `DatePropertyEditorRegistrar` 注册进入 `CustomEditorConfigurer` 的 `propertyEditorRegistrars` 可以具有相同效果。

深入探索一下 `ResourceEditorRegistrar` 的内部实现 `registerCustomEditors`：

```java
// 注册一系列常用类型的属性编辑器，
public void registerCustomEditors(PropertyEditorRegistry registry) {
    ResourceEditor baseEditor = new ResourceEditor(this.resourceLoader, this.propertyResolver);
    this.doRegisterEditor(registry, Resource.class, baseEditor);
    this.doRegisterEditor(registry, ContextResource.class, baseEditor);
    this.doRegisterEditor(registry, InputStream.class, new InputStreamEditor(baseEditor));
    this.doRegisterEditor(registry, InputSource.class, new InputSourceEditor(baseEditor));
    this.doRegisterEditor(registry, File.class, new FileEditor(baseEditor));
    this.doRegisterEditor(registry, Path.class, new PathEditor(baseEditor));
    this.doRegisterEditor(registry, Reader.class, new ReaderEditor(baseEditor));
    this.doRegisterEditor(registry, URL.class, new URLEditor(baseEditor));
    ClassLoader classLoader = this.resourceLoader.getClassLoader();
    this.doRegisterEditor(registry, URI.class, new URIEditor(classLoader));
    this.doRegisterEditor(registry, Class.class, new ClassEditor(classLoader));
    this.doRegisterEditor(registry, Class[].class, new ClassArrayEditor(classLoader));
    if (this.resourceLoader instanceof ResourcePatternResolver) {
        this.doRegisterEditor(registry, Resource[].class, new ResourceArrayPropertyEditor((ResourcePatternResolver)this.resourceLoader, this.propertyResolver));
    }
}

// 注册 Class 类对应的属性编辑器
// 注册后一旦某个实体 bean 中存在一些 Class 类型的属性，那么 Spring 会调用 ClassEditor 将配置中定义的 String 类型转换为 Class 类型并进行赋值
private void doRegisterEditor(PropertyEditorRegistry registry, Class<?> requiredType, PropertyEditor editor) {
    if (registry instanceof PropertyEditorRegistrySupport) {
        ((PropertyEditorRegistrySupport)registry).overrideDefaultEditor(requiredType, editor);
    } else {
        registry.registerCustomEditor(requiredType, editor);
    }
}
```

在 `doRegisterEditor` 方法中，可以看到 `registry.registerCustomEditor(requiredType, editor);` 方法。

不过现在问题来了，`prepareBeanFactory` 仅仅是 add 了一下 `PropertyEditorRegistrar`，即仅仅注册了，但没有调用 `ResourceEditorRegistrar` 的 `registerCustomEditors`，那么是什么时候注册的呢？

![image-20220512013050065](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220512013050065.png)

我们可以看到原来在 `AbstarctBeanFactory` 中调用了 `registerCustomEditors`

![image-20220512013220376](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220512013220376.png)

原来在 `initBeanWrapper` 就已经调用过它了，这是 bean 初始化时使用的一个方法，主要是将 `BeanDefinition` 转换为 `BeanWrapper` 后对属性的填充。

Spring 中用于封装 bean 的是 `BeanWrapper` 类型，而它又间接继承了 `PropertyEditorRegistry` 类型，对于 `BeanWrapper` 在 Spring 中的默认实现是 `BeanWrapperImpl`，除了实现 `BeanWrapper` 接口，它还继承了 `PropertyEditorRegistrySupport`

### 6.5.3 添加 `ApplicationContextAwareProcessor` 处理器

`beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));` 主要目的就是注册个 `BeanPostProcessor`，真正的逻辑还是在 `ApplicationContextAwareProcessor` 中

`ApplicationContextAwareProcessor` 实现 `BeanPostProcessor` 接口，bean 实例化时，`init-method` 前后会嗲用 `BeanPostProcessor` 的 `postProcessorBeforeInitialization` 方法和 `postProcessorAfterInitialization` 方法，对于 `ApplicationContextAwareProcessor` 我们也主要关注这两个方法：

#### After 方法

```java
default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    return bean;
}
```

after 方法没有做改动，还是 `BeanPostProcessor`

#### Before 方法

```java
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (!(bean instanceof EnvironmentAware) && !(bean instanceof EmbeddedValueResolverAware) && !(bean instanceof ResourceLoaderAware) && !(bean instanceof ApplicationEventPublisherAware) && !(bean instanceof MessageSourceAware) && !(bean instanceof ApplicationContextAware)) {
        return bean;
    } else {
        AccessControlContext acc = null;
        if (System.getSecurityManager() != null) {
            acc = this.applicationContext.getBeanFactory().getAccessControlContext();
        }
        if (acc != null) {
            AccessController.doPrivileged(() -> {
                // !
                this.invokeAwareInterfaces(bean);
                return null;
            }, acc);
        } else {
            // !
            this.invokeAwareInterfaces(bean);
        }
        return bean;
    }
}

private void invokeAwareInterfaces(Object bean) {
    // 全都是 set，显然 Spring 想让这些 bean 获取一些资源
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware)bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof EmbeddedValueResolverAware) {
        ((EmbeddedValueResolverAware)bean).setEmbeddedValueResolver(this.embeddedValueResolver);
    }
    if (bean instanceof ResourceLoaderAware) {
        ((ResourceLoaderAware)bean).setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationEventPublisherAware) {
        ((ApplicationEventPublisherAware)bean).setApplicationEventPublisher(this.applicationContext);
    }
    if (bean instanceof MessageSourceAware) {
        ((MessageSourceAware)bean).setMessageSource(this.applicationContext);
    }
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware)bean).setApplicationContext(this.applicationContext);
    }
}
```

`postProcessBeforeInitialization` 方法中调用了 `invokeAwareInterfaces`，实现这些 Aware 接口的 bean 初始化后可以获取一些对应的资源。

### 6.5.4 设置忽略依赖

Spring 将 后置处理器注册后，在 `invokeAwareInterfaces` 中调用的 Aware 类不是普通的 bean，需要在 Spring 做依赖注入时忽略它们，`ignoreDependencyInterface` 作用正是在此。

```java
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

### 6.5.5 注册依赖

spring 也会有注册依赖的功能

```java
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

例如注册了 `BeanFactory.class` 的解析依赖后，当 bean 的属性注入时，一旦检测到属性为 `BeanFactory` 的类便会将 `beanFactory` 的实例注入。 

## 6.6 `BeanFactory` 的后处理 `postProcessBeanFactory(beanFactory)`

Spring 为了保证程序的高可扩展性，针对 `BeanFactory` 做了大量的扩展，例如 `PostProcessor` 等都是在这里实现的。

### 6.6.1 激活注册的 `BeanFactoryPostProcessor`

先了解一下 `BeanFactoryPostProcessor` 的用法：

`BeanFactoryPostProcessor` 接口与 `BeanPostProcessor` 类似，可以对 bean 定义进行处理，Spring IOC 允许 `BeanFactroyPostProcessor` 在容器实际实例化任何其他的 bean 之前读取配置元数据，并有可能修改它。

如果愿意的话可以配置多个 `BeanFactoryPostProcessor`，还能通过 `order` 属性来控制 `BeanFactoryPostProcessor` 的执行次序（仅当 `BeanFactoryPostProcessor` 实现 `Ordered` 接口才可以设置）。

如果想改变实际的 bean 实例，最好使用 `BeanPostProcessor`，`BeanFactoryPostProcessor` 的作用域范围是容器级的，它只和你使用的容器有关。如果你在容器中定义一个 `BeanFactoryPostProcessor`，它仅仅对容器中的 bean 做后置处理，`BeanFactoryPostProcessor` 不会对定义在另一个容器中的 bean 进行后置处理，即使这两个容器都是在同一层次上。

#### `BeanFactoryPostProcessor` 的典型应用：`PropertyPlaceholderConfigurer`

有时阅读 Spring 的 Bean 描述文件时，你会遇到类似如下的一些配置：

```xml
<bean id="message" class="distConfig.HelloMessage">
	<property name="mes">
        <value>${bean.message}</value>
    </property>
</bean>
```

其中出现了变量引用：`${bean.message}`，这就是 Spring 的分散配置，可以在另外的配置文件中为 `bean.message` 指定值。如在 `bean.properties` 配置如下定义：

```properties
bean.message=Hi,can you find me?
```

当访问名为 message 的 bean 时，`mes` 属性就会被置为字符串 "Hi, can you find me?"，但 Spring 框架如何直到存在这样的配置文件呢？要靠 `PropertyPlaceholderConfigurer` 这个类的 bean：

```xml
<bean id="mesHandler" class="org.Springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
    	<list>
        	<value>config/bean.properties</value>
        </list>
    </property>
</bean>
```

就是在这里指定了配置文件的位置。不过还有个问题，这个 `mesHandler` 只是个普通的 bean，Spring 的 `BeanFactory` 如何直到从这个 bean 中获取配置信息呢？

![image-20220512140801956](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220512140801956.png)

原来它间接实现了 `BeanFactoryPostProcessor` 接口，这是一个很特别的接口，当 Spring 加载任何实现了这个接口的 bean 的配置，都会在 bean 工厂载入所有的 bean 的配置之后执行 `postProcessBeanFactory`方法，在 `PropertyResourceConfigurer` 类中实现了 `postProcessBeanFactory` 方法，在方法中先后调用了 `mergeProperties、convertProperties、processProperties` 这 3 个方法，分别得到配置、将得到的配置转换为合适的类型，最后将配置内容告知 `BeanFactory`。

正是通过实现 `BeanFactoryPostProcessor` 接口，`BeanFactory` 会在实例化任何 bean 之前获得配置信息，从而能正确解析 bean 描述文件中的变量引用。

#### 使用自定义 `BeanFactoryPostProcessor`

我们可以实现一个 `BeanFactoryPostProcessor`

```java
public class ObscenityRemovingBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    private final Set<String> obscenties;

    public ObscenityRemovingBeanFactoryPostProcessor() {
        obscenties = new HashSet<>();
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        String[] beanNames = configurableListableBeanFactory.getBeanDefinitionNames();
        StringValueResolver stringValueResolver = new StringValueResolver() {
            @Override
            public String resolveStringValue(String s) {
                // 将在 obscenties 中有的字符串替换为 ******
                if (isObscene(s)) return "******";
                return s;
            }
        };
        for (String beanName : beanNames) {
            BeanDefinition bd = configurableListableBeanFactory.getBeanDefinition(beanName);
            BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(stringValueResolver);
            // visitor 根据 stringValueResolver 修改 beanDefinition 字段值
            visitor.visitBeanDefinition(bd);
        }
    }

    public boolean isObscene(Object value) {
        String s = value.toString().toUpperCase();
        return this.obscenties.contains(s);
    }

    public void setObscenties(Set<String> obscenties) {
        this.obscenties.clear();
        for (String obscenty : obscenties) {
            this.obscenties.add(obscenty.toUpperCase());
        }
    }
}
```

配置文件：

```xml
<bean id="bfpp" class="com.test.ObscenityRemovingBeanFactoryPostProcessor">
    <property name="obscenties">
        <set>
            <value>bollocks</value>
            <value>winky</value>
            <value>bum</value>
            <value>Microsoft</value>
        </set>
    </property>
</bean>

<bean class="com.spring5.SimplePostProcessor" id="simpleBean">
    <property name="connectionString" value="bollocks" />
    <property name="userName" value="Microsoft" />
    <property name="password" value="imagine" />
</bean>
```

测试打印：

```java
SimplePostProcessor{connectionString='******', password='imagine', userName='******'}
```

#### 激活 `BeanFactoryPostProcessor`

深入 `BeanFactoryPostProcessor` 的调用过程：

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, this.getBeanFactoryPostProcessors());
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean("loadTimeWeaver")) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}

public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    Set<String> processedBeans = new HashSet();
    ArrayList regularPostProcessors;
    ArrayList registryProcessors;
    int var9;
    ArrayList currentRegistryProcessors;
    String[] postProcessorNames;
    
    if (beanFactory instanceof BeanDefinitionRegistry) {
        // 对 BeanDefinitionRegistry 类型的处理
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry)beanFactory;
        regularPostProcessors = new ArrayList();
        registryProcessors = new ArrayList();
        
        // 硬编码的注册的后处理器
        Iterator var6 = beanFactoryPostProcessors.iterator();
        while(var6.hasNext()) {
            BeanFactoryPostProcessor postProcessor = (BeanFactoryPostProcessor)var6.next();
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor = (BeanDefinitionRegistryPostProcessor)postProcessor;
                
                // 对于 BeanDefinitionRegistryPostProcessor 类型，在 BeanFactoryPostProcessor 的基础上还有自己定义的方法，需要先调用
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            } else {
                // 记录常规 BeanFactoryPostProcessor
                regularPostProcessors.add(postProcessor);
            }
        }
        currentRegistryProcessors = new ArrayList();
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        String[] var16 = postProcessorNames;
        var9 = postProcessorNames.length;
        int var10;
        String ppName;
        for(var10 = 0; var10 < var9; ++var10) {
            ppName = var16[var10];
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        var16 = postProcessorNames;
        var9 = postProcessorNames.length;
        for(var10 = 0; var10 < var9; ++var10) {
            ppName = var16[var10];
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();
        boolean reiterate = true;
        while(reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            String[] var19 = postProcessorNames;
            var10 = postProcessorNames.length;
            for(int var26 = 0; var26 < var10; ++var26) {
                String ppName = var19[var26];
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }
        
        // 激活 postProcessBeanFactory 方法，之前激活的是 postProcessBeanDefinitionRegistry
        // 硬编码设置的 BeanDefinitionRegistryPostProcessor
        invokeBeanFactoryPostProcessors((Collection)registryProcessors, (ConfigurableListableBeanFactory)beanFactory);
        invokeBeanFactoryPostProcessors((Collection)regularPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    } else {
        invokeBeanFactoryPostProcessors((Collection)beanFactoryPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    }
    
    // 对于配置中读取的 BeanFactoryPostProcessor 的处理
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
    regularPostProcessors = new ArrayList();
    registryProcessors = new ArrayList();
    currentRegistryProcessors = new ArrayList();
    postProcessorNames = postProcessorNames;
    int var20 = postProcessorNames.length;
    String ppName;
    // 对后处理器进行分类
    for(var9 = 0; var9 < var20; ++var9) {
        ppName = postProcessorNames[var9];
        if (!processedBeans.contains(ppName)) {
            // 没有处理过
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                regularPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                registryProcessors.add(ppName);
            } else {
                currentRegistryProcessors.add(ppName);
            }
        }
    }
    // 按照优先级排序
    sortPostProcessors(regularPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors((Collection)regularPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList(registryProcessors.size());
    Iterator var21 = registryProcessors.iterator();
    while(var21.hasNext()) {
        String postProcessorName = (String)var21.next();
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    
    // 按照 order 排序
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors((Collection)orderedPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    
    // 无序，直接使用
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList(currentRegistryProcessors.size());
    Iterator var24 = currentRegistryProcessors.iterator();
    while(var24.hasNext()) {
        ppName = (String)var24.next();
        nonOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors((Collection)nonOrderedPostProcessors, (ConfigurableListableBeanFactory)beanFactory);
    beanFactory.clearMetadataCache();
}
```

对于 `BeanFactoryPostProcessor` 的处理主要分两种情况进行：一个是对于 `BeanDefinitionRegistry` 类的特殊处理，另一种是对普通的 `BeanFactoryPostProcessor` 进行处理。对于每种情况都要考虑硬编码注入注册的后处理器以及通过配置注入的后处理器。

对于 `BeanDefinitionRegistry` 类型的处理类的处理主要包括以下内容：

1. 对于硬编码注册的后处理器的处理，主要是通过 `AbstractApplicationContext` 中的添加处理器方法 `addBeanFactoryProcessor` 进行添加。

   ```java
   public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
       Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
       this.beanFactoryPostProcessors.add(postProcessor);
   }
   ```

   添加后的后处理器会存放在 `beanFactoryPostProcessors` 中，而在处理 `BeanFactoryPostProcessor` 时会首先检测 `beanFactoryPostProcessors` 是否有数据。

2. 记录后处理器只要使用 3 个 List 完成

   - `registryPostProcessors`：记录通过硬编码方法注册的 `BeanDefinitionRegistryPostProcessor` 类型的处理器
   - `regularPostProcessors`：记录通过硬编码方法注册的 `BeanFactoryPostProcessor` 类型的处理器
   - `registryPostProcessorBeans`：记录通过配置方式注册的 `BeanDefinitionRegistryPostProcessor` 类型的处理器。

3. 对以上记录的 List 中的后处理器统一调用 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法

4. 对 `beanFactoryPostProcessors` 中非 `BeanDefinitionRegistryPostPorcessor` 类型的后处理器进行统一的 `BeanFactoryPostProcessor` 的 `postPorcessBeanFactory` 方法调用

5. 普通 `beanFactory` 处理

这里需要注意：对于硬编码方式手动添加的后处理器不需要做任何排序，但是在配置文件中读取的处理器，Spring 不保证读取的顺序，所以为了保证用户的调用顺序要求，Spring 对于后处理器的调用支持按照 `PriorityOrdered` 或者 `Ordered` 顺序调用。

### 6.6.2 注册 `BeanPostProcessor`

这里不是调用，而是注册，真正的调用是在 bean 的实例化阶段进行，这是很重要的步骤，也是很多功能 `BeanFactory` 不支持的重要原因，Spring 大部分的功能都是通过后处理器的方式进行扩展，这是 Spring 的一个特性，但是在 `BeanFactory` 并没有实现后处理器的自动注册，所以调用时如果没有手动注册其实是不能使用的，但是在 `ApplicationContext` 中添加了自动注册功能。

如定义这样一个后处理器：

```java
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    @Override
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        System.out.println("====");
        return null;
    }
}
```

在配置文件中配置：

```xml
<bean class="com.test.MyInstantiationAwareBeanPostProcessor" />
```

使用 `BeanFactory` 进行 Spring 的 bean 的加载时是不会有任何改变的，但是使用 `ApplicationContext` 方式获取 bean 时会在获取每个 bean 时打印出 "===="，这个特性就是在 `registerBeanPostProcessors` 方法实现的。

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}

public static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    
    // BeanPostProcessorChecks 是一个普通的打印信息，可能会有些情况
    // 当 Spring 的配置中的后处理器还没有被注册就已经开始了 bean 的初始化时
    // 会打印出 BeanPostProcessorChecker 中设定的信息
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(new PostProcessorRegistrationDelegate.BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
    
    // 使用 PriorityOrdered 保证顺序
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList();
    
    // 使用 Ordered 保证顺序
    List<String> orderedPostProcessorNames = new ArrayList();
    
    // 无序 BeanPostProcessor
    List<String> nonOrderedPostProcessorNames = new ArrayList();
    String[] var8 = postProcessorNames;
    int var9 = postProcessorNames.length;
    String ppName;
    BeanPostProcessor pp;
    for(int var10 = 0; var10 < var9; ++var10) {
        ppName = var8[var10];
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        } else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }
    // 注册所有实现 PriorityOrdered 的 BeanPostProcessor
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, (List)priorityOrderedPostProcessors);
    
    // 注册所有实现 Ordered 的 BeanPostProcessor
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList(orderedPostProcessorNames.size());
    Iterator var14 = orderedPostProcessorNames.iterator();
    while(var14.hasNext()) {
        String ppName = (String)var14.next();
        BeanPostProcessor pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, (List)orderedPostProcessors);
    
    // 注册无序的 BeanPostProcessor
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList(nonOrderedPostProcessorNames.size());
    Iterator var17 = nonOrderedPostProcessorNames.iterator();
    while(var17.hasNext()) {
        ppName = (String)var17.next();
        pp = (BeanPostProcessor)beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    registerBeanPostProcessors(beanFactory, (List)nonOrderedPostProcessors);
    
    // 注册所有 MergedBeanDefinitionPostProcessor 类型的 BeanPostProcessor，并非重复注册
    // 在 beanFactory.addBeanPostProcessor 中会先移除已经存在的 BeanPostProcessor
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, (List)internalPostProcessors);
    
    // 添加 ApplicationListener 探测器
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

对于 `BeanPostProcessor` 的处理似乎与 `BeanFactoryPostProcessor` 的处理即为相似，但是也有一些不同的地方，对于 `BeanFactoryPostProcessor` 的处理要区分两种情况，一种是通过硬编码方式处理，另一种是通过配置文件方式处理。

那为什么在 `BeanPostProcessor` 只考虑配置文件而不考虑硬编码呢？

因为对于 `BeanFactoryPostProcessor` 的处理，不但要实现注册功能，还要实现对后处理器的激活操作，所以需要载入配置中的定义，进行激活，而对于 `BeanPostProcessor` 不需要马上调用，硬编码方式实现的功能是将后处理器提取并调用，这里不需要调用，所以不需要考虑硬编码的方法，只要将配置文件的 `BeanPostProcessor` 提取出来并注册到 `beanFactory` 即可。

### 6.6.3 初始化消息资源

回顾下 Spring 国际化的使用方法

国际化简单来说就是根据客户端的系统语言返回对应的界面，英文操作系统返回英文界面，中文操作系统返回中文界面，这就是典型的 i18n 国际化问题。对于有国际化要求的应用系统，不能简单采用硬编码的方式编写用户界面信息，必须要国际化的信息进行特殊处理，简单来说就是为每种诞提供一套相应的资源文件，并以规范化命名的方式保存在特定的目录中，系统可以自动根据客户端语言选择适合的资源文件。

一般需要两个条件才可以确定一个特定类型的本地化信息，分别是"语言类型" 和 "国家/地区的类型"。 Java 通过 `java.util.Locale` 类表示一个本地化对象，它允许通过语言参数和国家/地区参数创建一个确定的本地化对象。

示例：

```java
// 带有语言和国家/地区信息的本地化对象
Locale locale1 = new Locale("zh", "CN");
// 只有语言信息的本地化对象
Locale locale2 = new Locale("zh");
// 等于 Locale locale1 = new Locale("zh", "CN")
Locale locale3 = Locale.CHINA;
// 等于 new Locale("zh")
Locale locale4 = Locale.CHINESE;
// 获取本地系统默认的本地化对象
Locale locale5 = Locale.getDefault();
```

Java 的 `java.util` 包提供几个支持本地化的格式化操作工具类：`NumberFormat、DateFormat、MessageFormat`，而在 Spring 中的国际化资源操作无非是对这些类的封装操作，介绍 `MessageFormat` 的用法：

```java
// 信息格式字符串
String pattern1 = "{0}, 你好！你于{1}在工商银行存入{2}元";
String pattern2 = "At {1, time, short} On {1, date, long} {0} paid {2, number, currency}.";

// 动态替换占位符的参数
Object[] params = {"John", new GregorianCalendar().getTime(), 1.0E3};

// 使用默认本地化对象格式化信息
String msg1 = MessageFormat.format(pattern1, params);

// 使用指定本地化对象格式化信息
MessageFormat mf = new MessageFormat(pattern2, Locale.US);
String msg2 = mf.format(params);

System.out.println(msg1);
System.out.println(msg2);
```

输出：

```
John, 你好！你于22-5-12 下午4:21在工商银行存入1,000元
At 4:21 PM On May 12, 2022 John paid $1,000.00.
```

Spring 定义了访问国际化信息的 `MessageSource` 接口，并提供了几个易用的实现类。
`MessageSource` 分别被 `HierarchicalMessageSource` 和 `ApplicationContext` 接口扩展，主要看一下 `HierarchicalMessageSource` 接口的几个实现类：

![image-20220512162711513](C:\Users\HASEE\Desktop\读书笔记\spring源码剖析.assets\image-20220512162711513.png)

`HierachicalMessageSource` 最重要的两个实现类是 `ResourceBundleMessageSource` 和 `ReloadableResourceBundleMessageSource`，它们基于 Java 的 `ResourceBundle` 基础类实现，允许通过资源名加载国际化资源。

#### 定义资源文件

- `messages.properties` （默认英文）

  ```properties
  test=test
  ```

- `messages_zh_CN.properties`（简体中文）

  ```properties
  test=测试
  ```

#### 配置文件

```xml
<bean class="org.springframework.context.support.ResourceBundleMessageSource" id="messageSource">
    <property name="basenames">
        <list>
            <value>messages</value>
        </list>
    </property>
</bean>
```

#### 使用 `ApplicationContext`访问国际化信息

```java
String[] configs = {"bean1.xml"};
ApplicationContext ctx = new ClassPathXmlApplicationContext(configs);
Object[] params = {"John", new GregorianCalendar().getTime()};
String str1 = ctx.getMessage("test", params, Locale.US);
String str2 = ctx.getMessage("test", params, Locale.CHINA);
System.out.println(str1);
System.out.println(str2);
```

messages 中配置就是

```properties
test={0},{1}
```

打印：

```
John,5/12/22 4:48 PM
John,22-5-12 下午4:48
```



在 `initMessageSource` 中的方法主要功能是提取配置中定义的 `messageSource`，并将其记录在 Spring 容器中，也就是 `AbstractApplicationContext`。如果用户未设置资源文件，Spring 提供了默认配置 `DelegatingMessageSource`。

在 `initMessageSource` 中获取自定义资源文件的方式为 `beanFactory.getBean(MESSAGE_SOURE_BEAN_NAME, MessageSource.class)`，在这里 Spring 做了硬编码硬性规定了自定义资源文件必须为 message，否则会获取不到自定义资源配置。

```java
protected void initMessageSource() {
    ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
    if (beanFactory.containsLocalBean("messageSource")) {
        // 如果在配置中已经配置了 messageSource，那么将 messageSource 提取并记录在 this.messageSource 中
        this.messageSource = (MessageSource)beanFactory.getBean("messageSource", MessageSource.class);
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
            HierarchicalMessageSource hms = (HierarchicalMessageSource)this.messageSource;
            if (hms.getParentMessageSource() == null) {
                hms.setParentMessageSource(this.getInternalParentMessageSource());
            }
        }
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Using MessageSource [" + this.messageSource + "]");
        }
    } else {
        // 用户没有定义配置文件，使用临时的 DelegatingMessageSource 以便于作为调用 getMessage 方法的返回
        DelegatingMessageSource dms = new DelegatingMessageSource();
        dms.setParentMessageSource(this.getInternalParentMessageSource());
        this.messageSource = dms;
        beanFactory.registerSingleton("messageSource", this.messageSource);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("No 'messageSource' bean, using [" + this.messageSource + "]");
        }
    }
}
```

### 6.6.4 初始化 `ApplicationEventMulticaster`

看一下 Spring 事件监听的简单用法

#### 定义监听事件

```java
public class TestEvent extends ApplicationEvent {
    public String msg;
    
    public TestEvent(Object source) {
        super(source);
    }
    
    public TestEvent(Object source, String msg) {
        super(source);
        this.msg = msg;
    }
    
    public void print() {
        System.out.println(msg);
    }
}
```

#### 定义监听器

```java
public class TestListener implements ApplicationListener {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof TestEvent) {
            TestEvent testEvent = (TestEvent) event;
            testEvent.print();
        }
    }
}
```

#### 配置文件

```xml
<bean class="com.test.TestListener" id="testListener"/>
```

#### 测试

```java
@Test
public void testEvent() {
    ApplicationContext ctx = new ClassPathXmlApplicationContext("bean1.xml");
    TestEvent event = new TestEvent("hello", "msg");
    ctx.publishEvent(event);
}
```

输出：

```
msg
```

程序运行时，Spring 会将发出的 `TestEvent` 事件转给我们自定义的 `TestListener` 进行进一步处理。

这里是设计模式中的观察者模式的运用。

我们看看 `ApplicationEventMulticaster` 是如何被初始化以确保功能的正确运行：

```java
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
    if (beanFactory.containsLocalBean("applicationEventMulticaster")) {
        this.applicationEventMulticaster = (ApplicationEventMulticaster)beanFactory.getBean("applicationEventMulticaster", ApplicationEventMulticaster.class);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    } else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton("applicationEventMulticaster", this.applicationEventMulticaster);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("No 'applicationEventMulticaster' bean, using [" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
        }
    }
}
```

这里教简单，考虑两种情况：

- 如果用户自定义事件广播器，那么使用用户自定义事件广播器。
- 否则使用默认的 `ApplicationEventMulticaster`

作为广播器，一定是用于存放监听器并在合适时调用监听器，不妨进入默认广播器实现 `SimpleApplicationEventMulticaster` 看看源码：

```java
public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
    Executor executor = this.getTaskExecutor();
    Iterator var5 = this.getApplicationListeners(event, type).iterator();
    while(var5.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var5.next();
        if (executor != null) {
            executor.execute(() -> {
                this.invokeListener(listener, event);
            });
        } else {
            this.invokeListener(listener, event);
        }
    }
}
```

当产生 Spring 事件时会默认使用 `SimpleApplicationEventMulticaster` 的 `multicastEvent` 来广播事件，遍历所有监听器，使用监听器中的 `obApplicationEvent` 方法进行监听器处理，对于每个监听器都可以获取到产生的事件，但是是否进行处理由事件监听器来决定。

### 6.6.5 注册监听器

```java
protected void registerListeners() {
    // 硬编码方式注册的监听器处理
    Iterator var1 = this.getApplicationListeners().iterator();
    while(var1.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var1.next();
        this.getApplicationEventMulticaster().addApplicationListener(listener);
    }
    
    // 配置文件注册的监听器处理
    String[] listenerBeanNames = this.getBeanNamesForType(ApplicationListener.class, true, false);
    String[] var7 = listenerBeanNames;
    int var3 = listenerBeanNames.length;
    for(int var4 = 0; var4 < var3; ++var4) {
        String listenerBeanName = var7[var4];
        this.getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }
    
    
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
        Iterator var9 = earlyEventsToProcess.iterator();
        while(var9.hasNext()) {
            ApplicationEvent earlyEvent = (ApplicationEvent)var9.next();
            this.getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

## 6.7 初始化非延迟加载单例 `finishBeanFactoryInitialization`

完成 `BeanFactory` 的初始化工作，包括 `ConversionService` 的设置、配置冻结以及非延迟加载的 bean 的初始化工作：

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    if (beanFactory.containsBean("conversionService") && beanFactory.isTypeMatch("conversionService", ConversionService.class)) {
        beanFactory.setConversionService((ConversionService)beanFactory.getBean("conversionService", ConversionService.class));
    }
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver((strVal) -> {
            return this.getEnvironment().resolvePlaceholders(strVal);
        });
    }
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    String[] var3 = weaverAwareNames;
    int var4 = weaverAwareNames.length;
    for(int var5 = 0; var5 < var4; ++var5) {
        String weaverAwareName = var3[var5];
        this.getBean(weaverAwareName);
    }
    beanFactory.setTempClassLoader((ClassLoader)null);
    
    // 冻结所有的 bean 定义，说明注册的 bean 定义不能修改或任何进一步的处理
    beanFactory.freezeConfiguration();
    
    // 初始化剩下的单例（非惰性）
    beanFactory.preInstantiateSingletons();
}
```

### `ConversionService` 的设置

之前提过使用自定义类型转换器从 String 转换为 Date 方式，那么在 Spring 中提供了另一种转换方式：使用 Converter。

实例：

1. 定义转换器

   ```java
   public class StringToDateConverter implements Converter<String, Date> {
       @Override
       public Date convert(String s) {
           SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
           try {
               return sdf.parse(s);
           } catch (ParseException e) {
               return null;
           }
       }
   }
   ```

   

2. 注册

   ```xml
   <bean class="org.springframework.context.support.ConversionServiceFactoryBean">
       <property name="converters">
           <list>
               <bean class="com.test.StringToDateConverter" />
           </list>
       </property>
   </bean>
   ```

   

3. 测试

   ```java
   @Test
   public void testConvert() {
       DefaultConversionService conversionService = new DefaultConversionService();
       // 增加 converter
       conversionService.addConverter(new StringToDateConverter());
       
       String dateString = "2022-05-12 17:19:30";
       // 进行转换
       Date convert = conversionService.convert(dateString, Date.class);
       System.out.println(convert);
   }
   ```

### 冻结配置

冻结所有 `BeanDefinition`，说明注册的 bean 定义不能被修改或者进一步处理

```java
public void freezeConfiguration() {
    this.configurationFrozen = true;
    this.frozenBeanDefinitionNames = StringUtils.toStringArray(this.beanDefinitionNames);
}
```

### 初始化非延迟加载

`ApplicationContext` 实现的默认行为就是在启动时将所有单例 bean 提前实例化，这就意味着提前实例化作为初始化过程的一部分，`ApplicationContext` 实例会创建并配置所有的单例 bean，通常是一件好事，这样配置中的任何错误就会被立即发现。

```java
public void preInstantiateSingletons() throws BeansException {
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Pre-instantiating singletons in " + this);
    }
    List<String> beanNames = new ArrayList(this.beanDefinitionNames);
    Iterator var2 = beanNames.iterator();
    while(true) {
        String beanName;
        Object bean;
        do {
            while(true) {
                RootBeanDefinition bd;
                do {
                    do {
                        do {
                            if (!var2.hasNext()) {
                                var2 = beanNames.iterator();
                                while(var2.hasNext()) {
                                    beanName = (String)var2.next();
                                    Object singletonInstance = this.getSingleton(beanName);
                                    if (singletonInstance instanceof SmartInitializingSingleton) {
                                        SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton)singletonInstance;
                                        if (System.getSecurityManager() != null) {
                                            AccessController.doPrivileged(() -> {
                                                smartSingleton.afterSingletonsInstantiated();
                                                return null;
                                            }, this.getAccessControlContext());
                                        } else {
                                            smartSingleton.afterSingletonsInstantiated();
                                        }
                                    }
                                }
                                return;
                            }
                            beanName = (String)var2.next();
                            bd = this.getMergedLocalBeanDefinition(beanName);
                        } while(bd.isAbstract());
                    } while(!bd.isSingleton());
                } while(bd.isLazyInit());
                if (this.isFactoryBean(beanName)) {
                    bean = this.getBean("&" + beanName);
                    break;
                }
                this.getBean(beanName);
            }
        } while(!(bean instanceof FactoryBean));
        FactoryBean<?> factory = (FactoryBean)bean;
        boolean isEagerInit;
        if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
            SmartFactoryBean var10000 = (SmartFactoryBean)factory;
            ((SmartFactoryBean)factory).getClass();
            isEagerInit = (Boolean)AccessController.doPrivileged(var10000::isEagerInit, this.getAccessControlContext());
        } else {
            isEagerInit = factory instanceof SmartFactoryBean && ((SmartFactoryBean)factory).isEagerInit();
        }
        if (isEagerInit) {
            this.getBean(beanName);
        }
    }
}
```

## 6.8 `finishRefresh`

Spring 中还提供了 `Lifecycle` 接口，包含 `start/stop` 方法，实现此接口后 Spring 启动时调用其 `start` 方法开始生命周期，在 Spring 关闭时调用 stop 结束生命周期，通常来配置后台程序，启动后一直运行（如对 MQ 进行轮询）。而 `ApplicationContext` 初始化最后就是保证这个功能的实现。

```java
protected void finishRefresh() {
    // 清除资源缓存
    this.clearResourceCaches();
    
    // 初始化 LifecycleProcessor
    this.initLifecycleProcessor();
    
    // 实现 Lifecycle 的 onRefresh()
    this.getLifecycleProcessor().onRefresh();
   
    // 发布 ApplicationContext 初始化信息
    this.publishEvent((ApplicationEvent)(new ContextRefreshedEvent(this)));
    
    // 注册 ApplicationContext
    LiveBeansView.registerApplicationContext(this);
}
```

### `initLifecycleProcessor`

`ApplicationContext` 启动或者停止时，都会通过 `LifecycleProcessor` 来与所有声明的 bean 的周期做状态更新，而在 `LifecycleProcessor` 使用前需要初始化

```java
protected void initLifecycleProcessor() {
    ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
    if (beanFactory.containsLocalBean("lifecycleProcessor")) {
        this.lifecycleProcessor = (LifecycleProcessor)beanFactory.getBean("lifecycleProcessor", LifecycleProcessor.class);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
        }
    } else {
        DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
        defaultProcessor.setBeanFactory(beanFactory);
        this.lifecycleProcessor = defaultProcessor;
        beanFactory.registerSingleton("lifecycleProcessor", this.lifecycleProcessor);
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("No 'lifecycleProcessor' bean, using [" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
        }
    }
}
```

### `onRefresh`

启动所有实现了 `Lifecycle` 接口的 bean

```java
public void onRefresh() {
    this.startBeans(true);
    this.running = true;
}

private void startBeans(boolean autoStartupOnly) {
    Map<String, Lifecycle> lifecycleBeans = this.getLifecycleBeans();
    Map<Integer, DefaultLifecycleProcessor.LifecycleGroup> phases = new HashMap();
    lifecycleBeans.forEach((beanName, bean) -> {
        if (!autoStartupOnly || bean instanceof SmartLifecycle && ((SmartLifecycle)bean).isAutoStartup()) {
            int phase = this.getPhase(bean);
            DefaultLifecycleProcessor.LifecycleGroup group = (DefaultLifecycleProcessor.LifecycleGroup)phases.get(phase);
            if (group == null) {
                group = new DefaultLifecycleProcessor.LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
                phases.put(phase, group);
            }
            group.add(beanName, bean);
        }
    });
    if (!phases.isEmpty()) {
        List<Integer> keys = new ArrayList(phases.keySet());
        Collections.sort(keys);
        Iterator var5 = keys.iterator();
        while(var5.hasNext()) {
            Integer key = (Integer)var5.next();
            ((DefaultLifecycleProcessor.LifecycleGroup)phases.get(key)).start();
        }
    }
}
```

### `publishEvent`

完成 `ApplicationContext` 初始化时，通过 Spring 中的事件发布机制来发出 `ContextRefershedEvent` 事件，保证对应的监听器做进一步逻辑处理

```java
public void publishEvent(ApplicationEvent event) {
    this.publishEvent(event, (ResolvableType)null);
}

protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
    Assert.notNull(event, "Event must not be null");
    Object applicationEvent;
    if (event instanceof ApplicationEvent) {
        applicationEvent = (ApplicationEvent)event;
    } else {
        applicationEvent = new PayloadApplicationEvent(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
        }
    }
    if (this.earlyApplicationEvents != null) {
        this.earlyApplicationEvents.add(applicationEvent);
    } else {
        this.getApplicationEventMulticaster().multicastEvent((ApplicationEvent)applicationEvent, eventType);
    }
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext) {
            ((AbstractApplicationContext)this.parent).publishEvent(event, eventType);
        } else {
            this.parent.publishEvent(event);
        }
    }
}
```

