[TOC]
## 什么是Spring IOC/DI
IOC(Inversion of Control，控制反转)是一种思想，并不是什么组件，Spring中基于IOC思想设计了IOC容器，用来存放所有的bean，管理bean的声明周期和依赖关系。IOC的目的是为了解耦。
想要理解IOC，要明白反转了什么？谁控制谁？控制什么？

* 反转了什么？依赖对象的获得被反转了。在传统的应用中，在一个对象A中想用调用另一个对象B的某些方法或者字段，那么需要通过new这样的方式创建一个对象B，然后进行调用。这种方式是高耦合度的，如果想要调用别的对象，那么需要修改类A文件的源码，当业务复杂起来会变的十分繁琐。IOC容器是一种第三方的构建，它将所有的应用中的bean都统一集中进行管理，当对象需要调用其他对象的时候，通过IOC容器进行注入，这样实现了代码的解耦。
* 谁控制谁？IOC容器控制对象
* 控制什么？控制对象对外部资源的获取

## IOC容器
IOC容器有两个系列，一个是从BeanFactory接口开始的，这一系列是最原始的创建IOC容器；还有一个系列是ApplicationContext，这一系列在BeanFactory的基础上加了很多别的功能，比如国际化、通用资源加载和对事件的相应机制，比BeanFactory更高级。
#### IOC容器接口设计
![IOC_interfaces](https://github.com/CoderAssassin/markdownImg/blob/master/Spring/IOC_Interfaces.png?raw=true)
上图是IOC容器的主要的接口，总共分为两个系，一个是BeanFactory系，一个是ApplicationContext系。
BeanFactory系，主要有4个子接口：

* ListableBeanFactory：在BeanFactory的基础之上，细化了很多方法，比如声明了getBeanDefinitionNames()
* AutowireCapableBeanFactory：声明了基于注解的配置的一些方法
* HierarchicalBeanFactory：再BeanFactory的基础上，增加了getParentBeanFactory()等接口方法
* ConfigurableBeanFactory：定义了一些对BeanFactory的配置功能，比如setParentBeanFactory()、addBeanPostProcessor()等

对于ApplicationContext，又实现了3个接口，使得ApplicationContext可以支持更多的功能，因此相比于BeanFactory显得更加高富帅，这三个接口为：

* MessageSource：支持不同的信息源。这个接口使得ApplicationContext能支持不同的信息源
* ResourceLoader：访问资源。可以从不同的地方得到Bean定义资源。
* ApplicationEventPublisher：支持应用事件。引入事件机制，这些事件和Bean的生命周期的结合为Bean的管理提供了便利。

ApplicationContext还继承了ListableBeanFactory和AutowireCapableBeanFactory，所以说BeanFactory和ApplicationContext有同样的祖先。

#### BeanFactory容器的设计
以XMLBeanFactory为例说明BeanFactory一系的类继承关系，如下图：
![IOC_BeanFactory](https://github.com/CoderAssassin/markdownImg/blob/master/Spring/IOC_BeanFactory.png?raw=true)
上图是XMLBeanFactory的类集成关系，其他的BeanFactory实现类也是类似的继承关系，这里关键的是最后的**DefaultListableBeanFactory**，这个类已经包含了IOC容器所有的重要的功能。
上述的ConfigurableListableBeanFactory继承ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory，因此BeanFactory接口的下属三个类已经全部继承了。
接下来看下XmlBeanFactory的源码：
``` java
public class XmlBeanFactory extends DefaultListableBeanFactory {
//定义Reader
	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
    public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}
    public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);//开始加载BeanDefinition
	}
}
```
上述代码是XmlBeanFactory的所有代码，主要做了两件事：

* 创建XmlBeanDefinitionReader
* 在构造函数中调用XmlBeanDefinitionReader的**loadBeanDefinitions(resource)**来开始加载BeanDefinition，这里的参数是Resource的实例

这里出现的两个接口说下，一个是BeanDefinition，这个很重要，BeanDefinition抽象了对Bean的定义，是让容器起作用的主要数据类型。还有一个是Resource，Resource是对资源文件的抽象封装。
举个使用BeanFactory的使用样例模板：
``` java
		//1.创建Resource，封装配置文件
		ClassPathResource res=new ClassPathResource("beans.xml");
        //2.创建BeanFactory
        DefaultListableBeanFactory factory=new DefaultListableBeanFactory();
        //3.创建BeanDefinitionReader，传入BeanFactory参数，回调的方式设置属性
        XmlBeanDefinitionReader reader=new XmlBeanDefinitionReader(factory);
        //4.调用BeanDefinition读取配置
        reader.loadBeanDefinitions(res);
```

#### ApplicationContext容器的设计
以FileSystemXmlApplicationContext为例，类继承结构图如下：
![IOC_ApplicationContext](https://github.com/CoderAssassin/markdownImg/blob/master/Spring/IOC_ApplicationContext.png?raw=true)
进入FileSystemXmlApplicationContext类来看一下主要的入口：
``` java
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
	//...
    //上边一系列的构造函数最后是调用这个构造函数
    public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();//主入口
		}
	}
    //解析文件系统资源，返回Resource对象
    protected Resource getResourceByPath(String path) {
		if (path != null && path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}
}
```
通过观察FileSystemXmlApplicationContext类的代码，可以知道主入口是**refresh()**函数。

## IOC容器的初始化
ApplicationContext是最常用的IOC容器，所以这里以ApplicationContext来讲解，从上一节可知，**ApplicationContext的容器创建过程是从refresh()函数开始的**。
IOC容器的启动过程主要包括三个步骤：

* BeanDefinition的Resource定位：主要接口是Resource，针对不同类型的Bean信息，有不同的资源抽象。这个过程主要是定位Bean的配置信息。
* Resource的载入：将包含定义好的Bean的Resource转换成BeanDefinition，BeanDefinition是Spring中对POJO对象的抽象，因为不同的配置方式生成的Resource不一样，Spring只针对BeanDefinition来做处理，不认其他的类。
* BeanDefinition的注册：主要接口是BeanDefinitionRegistry，向IOC容器注册BeanDefinition。

refresh()方法其实实在AbstractApplicationContext这个抽象类里实现的，代码如下：
``` java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备初始化环境，设置容器初始化开始日期，激活标记等
			prepareRefresh();

			// 在子类中启动refreshBeanFactory()
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 准备BeanFactory的默认容器环境
			prepareBeanFactory(beanFactory);

			try {
				// 设置BeanFactory的后处理器，后处理器主要用来对BeanDefinition的配置做修改
				postProcessBeanFactory(beanFactory);

				// 调用BeanFactory的后处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// 调用所有已注册的后处理器对BeanFactory做处理
				registerBeanPostProcessors(beanFactory);

				// 初始化消息源
				initMessageSource();

				// 初始化事件机制
				initApplicationEventMulticaster();

				// 进行一些特定环境的refresh工作，由子类实现
				onRefresh();

				// 注册实现ApplicationListener的监听器
				registerListeners();

				// 初始化所有非懒加载的Bean
				finishBeanFactoryInitialization(beanFactory);

				// 发布容器时间，结束Refresh()
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
    //获得BeanFactory
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();//由子类实现
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```
#### 时序图
![IOC_ResourcePosition](https://github.com/CoderAssassin/markdownImg/blob/master/Spring/IOC_ResourcePosition.png?raw=true)
上图是大致的Resource定位时序图，DefaultListableBeanFactory是在AbstractRefreshableApplicationContext创建的，XmlBeanDefinitionReader是在AbstractXmlApplicationContext创建的。在XmlBeanDefinitionReader里，对资源的获取有两个过程，一个是直接获取到当前环境的所有Resource对象数组，还有一种是获取到资源的位置String数组，这就需要定位到String表示的位置然后加载配置文件为Resource对象。
![IOC_BeanLoader](https://github.com/CoderAssassin/markdownImg/blob/master/Spring/IOC_BeanLoader.png?raw=true)
上图是对Resource的加载和解析的大致流程，具体的对不同标签的解析是再BeanDefinitionParserDelegate里边进行，当BeanDefinition加载完成后需要注册到IOC容器里：
![IOC_BeanRegister](https://github.com/CoderAssassin/markdownImg/blob/master/Spring/IOC_BeanRegister.png?raw=true)
注册最终是回到DefaultListableBeanFactory里，以beanName为键，BeanDefinition为值存储到Map中。

#### Resource定位、加载、解析和注册
上述的obtainFreshBeanFactory()主要是获取BeanFactory，里边的refreshBeanFactory()是一个抽象方法，在子类**AbstractRefreshableApplicationContext**中实现：
``` java
	protected final void refreshBeanFactory() throws BeansException {
    	//若已经存在BeanFactory，那么将原来的去掉
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();//根据容器现有的双亲容器的信息创建一个新的DefaultListableBeanFactory
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);//设置BeanFactory是否允许BeanDefinition同名覆盖以及循环引用
            //载入过程的入口，这里开始载入所有的BeanDefinition，是个抽象方法，由子类实现
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```
上边这段代码主要做了两件事：

* 调用createBeanFactory()根据当前IOC容器的双亲容器来创建一个DefaultListableBeanFactory实例
* 调用loadBeanDefinitions(beanFactory)方法，这个和前边的加载BeanDefinition模板很类似，传入BeanFactory做回调，也就是说，**loadBeanDefinitions(beanFactory)是加载BeanDefinition的入口**。这个方法是个抽象方法，具体的实现是在子类AbstractXmlApplicationContext里

那么这个方法是在哪里实现的呢？其实是在**AbstractXmlApplicationContext**里实现的，看一下源码：
``` java
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 创建一个XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// 配置上下文资源加载环境
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// 初始化BeanDefinitionReader
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
    //在这个方法将加载交给BeanDefinitionReader
    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    	//以Resource的方式获取配置文件的资源位置
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
        //以String的形式获得配置文件的位置
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
```
上述两个方法主要是完成两件事：

* 创建一个XmlBeanDefinitionReader，并初始化
* 获取资源文件的位置，然后交给XmlBeanDefinitionReader来加载资源，所以说，**资源的加载是由DefinitionReader来实现的**。这里的两种获取资源的方式都是由子类重写。

接下来进入XmlBeanDefinitionReader里看一下具体的加载：
``` java
	//对应上边的对Resource方式配置的资源文件加载
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int counter = 0;
		for (Resource resource : resources) {
			counter += loadBeanDefinitions(resource);//关键方法
		}
		return counter;
	}
    //对应上边的对String方式配置的资源文件加载
    public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int counter = 0;
		for (String location : locations) {
			counter += loadBeanDefinitions(location);//关键方法
		}
		return counter;
	}
```
上述代码中，分别对应AbstractXmlApplicationContext中loadBeanDefinitions()的按照两种方式配置资源来进行加载，这里的loadBeanDefinitions()是接口方法，由子类**XmlBeanDefinitionReader**来实现，那么具体看看两种方式的实现，具体看代码：
``` java
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}
    //真正的实现地方
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

		//正在加载的Resource集
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<EncodedResource>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
            //得到输入流
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                //关键方法，加载BeanDefinition
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
    //具体加载资源的过程
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
        	//取得Xml文件的Document对象，这里类似Xml文件加载的常规流程
			Document doc = doLoadDocument(inputSource, resource);
            //关键方法，启动对BeanDefinition的解析过程
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
    --------------------------------下面是根据String配置资源位置
    //进入AbstractBeanDefinitionReader类里看实现方法
    public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int counter = 0;
		for (String location : locations) {
			counter += loadBeanDefinitions(location);
		}
		return counter;
	}
    public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(location, null);
	}
    //这里是真正的根据location来加载资源
    public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();//定义资源加载器对象
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}
		//对路径模式解析，得到多个Resource对象的集合，这些Resource指向定义好的bean信息
		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);//关键方法，根据位置获取Resource文件
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);//将得到的资源加入到列表中
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
    //这里是DefaultResourceLoader对特定位置资源的加载
    public Resource getResource(String location) {
		Assert.notNull(location, "Location must not be null");
		//调用不同的资源模式协议解析器对某个位置的资源进行尝试加载
		for (ProtocolResolver protocolResolver : this.protocolResolvers) {
			Resource resource = protocolResolver.resolve(location, this);
			if (resource != null) {
				return resource;
			}
		}

		if (location.startsWith("/")) {
			return getResourceByPath(location);
		}
		else if (location.startsWith(CLASSPATH_URL_PREFIX)) {//处理带有classpath标识的Resource
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
		}
		else {
			try {
				// 处理URL标识的Resource定位
				URL url = new URL(location);
				return new UrlResource(url);
			}
			catch (MalformedURLException ex) {
				// 既不是classpath开头，也不是URL标识的Resource定位，那么调用getResourceByPath()方法，这个方法在FileSystemXmlApplicationContext中实现
				return getResourceByPath(location);
			}
		}
	}
```
上述的根据String加载会定位到具体的Resource位置，根据路径情况进行解析转化成Resource文件。
上述的根据Resource加载会获取输入流，然后根据Resource和InputSource流来获得解析Xml文件来获得Document，具体的解析Xml文档的过程不多说，关键方法是**registerBeanDefinitions(doc, resource)**，这里是对BeanDefinition解析的入口，来看一下这个方法：
``` java
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    	//创建BeanDefinitionDocumentReader对象
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
        //关键方法
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```
观察上述代码，主要完成两件事：

* 创建BeanDefinitionDocumentReader实例，这个主要是用来将文档的Bean真正转化成BeanDefinition
* 调用BeanDefinitionDocumentReader实例的**registerBeanDefinitions()**方法来真正开始解析，而这个方法只是个接口方法，具体实现在DefaultBeanDefinitionDocumentReader中

具体看DefaultBeanDefinitionDocumentReader中的代码：
``` java
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) 	{
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		Element root = doc.getDocumentElement();
		doRegisterBeanDefinitions(root);
	}
    //根据每个root元素获取对应的BeanDefinition
    protected void doRegisterBeanDefinitions(Element root) {
    	//获取BeanDefinitionParserDelegate对象
		BeanDefinitionParserDelegate parent = this.delegate;
        //临时创建BeanDefinitionParserDelegate进行转化
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		//预处理转化，接口方法
		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
        //后处理转化，接口方法
		postProcessXml(root);

		this.delegate = parent;
	}
    //转化节点元素，这里会根据节点的孩子节点进行遍历，然后根据节点是默认的还是自定义的分别进行转化
    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
    //对默认元素进行转化
    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
        //这里遇到了bean标签，将该元素转化成BeanDefinition
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
        //这里是Beans标签，需要继续遍历孩子节点调用，最终还是会调用上边的处理
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
    //对Bean标签元素处理，转化成BeanDefinition
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    	//这里是入口，进入BeanDefinitionParserDelegate的对Bean元素的转化方法，最后返回BeanDefinitionHolder对象
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册的入口，向容器注册得到的BeanDefinition
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```
总结一下上边的流程，Document对象传过来后，接下来从根元素开始层层遍历，先是根据元素是自定义元素还是默认元素进行划分；然后继续解析，根据元素标签类型进行解析，碰到bean标签的进行bean的解析；具体的解析是交付给BeanDefinitionParserDelegate对象来进行的，在解析完了后封装成BeanDefinitionHolder对象。最后调用**BeanDefinitionReaderUtils.registerBeanDefinition()**注册BeanDefinition，这是注册的入口方法。
接下来看一下BeanDefinitionParserDelegate里的parseBeanDefinitionElement()方法是如何转化bean标签元素的：
``` java
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}
    public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		List<String> aliases = new ArrayList<String>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}

		//这里是真正对bean标签元素进行转化，最后会返回AbstractBeanDefinition对象类
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
    //详细解析bean元素
    public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}

		try {
			String parent = null;
			if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
				parent = ele.getAttribute(PARENT_ATTRIBUTE);
			}
            //创建BeanDefinition对象
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
            //对bean标签元素进行解析
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
            //对各种bean元素的而信息解析
			parseMetaElements(ele, bd);
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			parseConstructorArgElements(ele, bd);
			parsePropertyElements(ele, bd);//对<property>标签解析
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```
上述代码关键地方在于**parseBeanDefinitionElement(ele, beanName, containingBean)**方法，该方法会对bean元素解析，最终会返回AbstractBeanDefinition对象，最后封装成BeanDefinitionHolder返回。
下面以对&lt;property&gt;的解析为例进行说明：
``` java
	public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
				parsePropertyElement((Element) node, bd);
			}
		}
	}
    //对单个<property>进行解析
    public void parsePropertyElement(Element ele, BeanDefinition bd) {
		String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
		if (!StringUtils.hasLength(propertyName)) {
			error("Tag 'property' must have a 'name' attribute", ele);
			return;
		}
		this.parseState.push(new PropertyEntry(propertyName));
		try {
			if (bd.getPropertyValues().contains(propertyName)) {
				error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
				return;
			}
			Object val = parsePropertyValue(ele, bd, propertyName);
            //PropertyValue对象来保存转化后的值，每个属性值都是一个PropertyValue对象
			PropertyValue pv = new PropertyValue(propertyName, val);
			parseMetaElements(ele, pv);
			pv.setSource(extractSource(ele));
			bd.getPropertyValues().addPropertyValue(pv);
		}
		finally {
			this.parseState.pop();
		}
	}
    //转化属性的值，返回可能是单个值，也可能是list等
    public Object parsePropertyValue(Element ele, BeanDefinition bd, String propertyName) {
		String elementName = (propertyName != null) ?
						"<property> element for property '" + propertyName + "'" :
						"<constructor-arg> element";

		// Should only have one child element: ref, value, list, etc.
		NodeList nl = ele.getChildNodes();
		Element subElement = null;
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
					!nodeNameEquals(node, META_ELEMENT)) {
				// Child element is what we're looking for.
				if (subElement != null) {
					error(elementName + " must not contain more than one sub-element", ele);
				}
				else {
					subElement = (Element) node;
				}
			}
		}

		boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
		boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
		if ((hasRefAttribute && hasValueAttribute) ||
				((hasRefAttribute || hasValueAttribute) && subElement != null)) {
			error(elementName +
					" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
		}
        //值是ref，用RuntimeBeanReference来封装
		if (hasRefAttribute) {
			String refName = ele.getAttribute(REF_ATTRIBUTE);
			if (!StringUtils.hasText(refName)) {
				error(elementName + " contains empty 'ref' attribute", ele);
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName);
			ref.setSource(extractSource(ele));
			return ref;
		}
		else if (hasValueAttribute) {
			TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
			valueHolder.setSource(extractSource(ele));
			return valueHolder;
		}
		else if (subElement != null) {//若是还有子元素，对子元素进行解析
			return parsePropertySubElement(subElement, bd);
		}
		else {
			// Neither child element nor "ref" or "value" attribute found.
			error(elementName + " must specify a ref or value", ele);
			return null;
		}
	}
    public Object parsePropertySubElement(Element ele, BeanDefinition bd) {
		return parsePropertySubElement(ele, bd, null);
	}
    //对属性子元素进行解析，这里就是对list、set、map等不同类型的子元素进行解析
    public Object parsePropertySubElement(Element ele, BeanDefinition bd, String defaultValueType) {
		if (!isDefaultNamespace(ele)) {
			return parseNestedCustomElement(ele, bd);
		}
		else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
			BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
			if (nestedBd != null) {
				nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
			}
			return nestedBd;
		}
		else if (nodeNameEquals(ele, REF_ELEMENT)) {
			// A generic reference to any name of any bean.
			String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
			boolean toParent = false;
			if (!StringUtils.hasLength(refName)) {
				// A reference to the id of another bean in the same XML file.
				refName = ele.getAttribute(LOCAL_REF_ATTRIBUTE);
				if (!StringUtils.hasLength(refName)) {
					// A reference to the id of another bean in a parent context.
					refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
					toParent = true;
					if (!StringUtils.hasLength(refName)) {
						error("'bean', 'local' or 'parent' is required for <ref> element", ele);
						return null;
					}
				}
			}
			if (!StringUtils.hasText(refName)) {
				error("<ref> element contains empty target attribute", ele);
				return null;
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
			ref.setSource(extractSource(ele));
			return ref;
		}
		else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
			return parseIdRefElement(ele);
		}
		else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
			return parseValueElement(ele, defaultValueType);
		}
		else if (nodeNameEquals(ele, NULL_ELEMENT)) {
			// It's a distinguished null value. Let's wrap it in a TypedStringValue
			// object in order to preserve the source location.
			TypedStringValue nullHolder = new TypedStringValue(null);
			nullHolder.setSource(extractSource(ele));
			return nullHolder;
		}
		else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
			return parseArrayElement(ele, bd);
		}
		else if (nodeNameEquals(ele, LIST_ELEMENT)) {
			return parseListElement(ele, bd);
		}
		else if (nodeNameEquals(ele, SET_ELEMENT)) {
			return parseSetElement(ele, bd);
		}
		else if (nodeNameEquals(ele, MAP_ELEMENT)) {
			return parseMapElement(ele, bd);
		}
		else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
			return parsePropsElement(ele);
		}
		else {
			error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
			return null;
		}
	}
    //以对list的解析为例
    public List<Object> parseListElement(Element collectionEle, BeanDefinition bd) {
		String defaultElementType = collectionEle.getAttribute(VALUE_TYPE_ATTRIBUTE);
		NodeList nl = collectionEle.getChildNodes();
		ManagedList<Object> target = new ManagedList<Object>(nl.getLength());
		target.setSource(extractSource(collectionEle));
		target.setElementTypeName(defaultElementType);
		target.setMergeEnabled(parseMergeAttribute(collectionEle));
        //开始对list容器进行解析
		parseCollectionElements(nl, target, bd, defaultElementType);
		return target;
	}
    //对list容器进行解析
    protected void parseCollectionElements(
			NodeList elementNodes, Collection<Object> target, BeanDefinition bd, String defaultElementType) {
        //遍历每个元素进行解析
		for (int i = 0; i < elementNodes.getLength(); i++) {
			Node node = elementNodes.item(i);
			if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT)) {
				target.add(parsePropertySubElement((Element) node, bd, defaultElementType));
			}
		}
	}
```
上面是一个完整的对&lt;property&gt;属性进行解析的例子，对其他标签元素的解析也类似。
在DefaultBeanDefinitionDocumentReader中当我们获取到了BeanDefinitionHolder之后，将bean标签转化成BeanDefinition的过程就结束了，接下来就是将这个BeanDefinition注册到IOC容器中，注册的话调用的是BeanDefinitionReaderUtils.registerBeanDefinition()方法，这是主入口。注册其实就是将BeanDefinition添加到一个Map中的过程，具体看BeanDefinitionReaderUtils中代码：
``` java
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// 获取bean的主要名字
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());//利用注册器进行注册

		// 获取所有的别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);//再将主名和别名映射起来
			}
		}
	}
```
上述代码中会分别根据bean的主要名字和别名来注册映射关系，这里的关键方法registerBeanDefinition()是个接口方法，需要被子类实现，这里是回到DefaultListableBeanFactory中实现，具体代码如下：
``` java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							oldBeanDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(oldBeanDefinition)) {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// 以beanName为key，BeanDefinition对象为value存入到IOC容器中
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```
上述代码就是对BeanDefinition的注册，其实就是以beanName为键，BeanDefinition对象为值存入map中。至此，整个bean的定位、加载、解析和注册过程已经完成，但现在的bean还只是一个配置好的对象而已，真正创建实例需要等到调用这个bean的时候，或者根据配置文件设置是否在容器启动的时候创建来判断是否在启动容器的时候就创建实例。

