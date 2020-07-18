---
layout:     post
title:      "Spring源码(二) -- xml文件解析"
subtitle:   ""
date:       2020-07-18
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring源码
---

# 背景
上一节我们编写了一个Ioc的简单测试用例，通过xml文件配置bean文件并使用spring的BeanFactory去获取，我们这次就来解析下spring是如何加载xml中的配置并实现返回的。
# Resource
```
public interface Resource extends InputStreamSource {

    //资源是否存在
    boolean exists();

    //资源是否可读
    default boolean isReadable() {
        return true;
    }

    //资源是否是个open的流以供获取
    default boolean isOpen() {
        return false;
    }

    //资源是否是一个文件
    default boolean isFile() {
        return false;
    }

    //资源是否可以表示成URL对象
    URL getURL() throws IOException;

    //资源是否可以表示成URI对象
    URI getURI() throws IOException;

    //资源是否可以表示成File对象
    File getFile() throws IOException;

    //每次读取创建一个Channel
    default ReadableByteChannel readableChannel() throws IOException {
        return Channels.newChannel(getInputStream());
    }

    //资源内容的长度
    long contentLength() throws IOException;

    //资源最后修改的时间
    long lastModified() throws IOException;

    //创建一个复制资源
    Resource createRelative(String relativePath) throws IOException;

    //获取文件名称
    @Nullable
    String getFilename();

    //获取资源的描述
    String getDescription();
}
```
我们先看下Resource接口，Resource接口定义了资源的一些属性的方法，我们使用XmlBeanFactory(Resource resource)构造出XmlBeanFactory对象，那我们看下代码使用ClassPathResource(String path)够造出Resource对象，那么我就看下ClassPathResouce类
# ClassPathResouce
```
public class ClassPathResource extends AbstractFileResolvingResource {
    private final String path;

    @Nullable
    private ClassLoader classLoader;

    @Nullable
    private Class<?> clazz;

    public ClassPathResource(String path) {
        this(path, (ClassLoader) null);
    }

    public ClassPathResource(String path, @Nullable ClassLoader classLoader) {
        Assert.notNull(path, "Path must not be null");
        String pathToUse = StringUtils.cleanPath(path);
        if (pathToUse.startsWith("/")) {
            pathToUse = pathToUse.substring(1);
        }
        this.path = pathToUse;
        this.classLoader = (classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader());
    }
    //通过复写InputStreamSource接口的getInputStream方法获取资源的InputStream流
    @Override
    public InputStream getInputStream() throws IOException {
        InputStream is;
        if (this.clazz != null) {
            is = this.clazz.getResourceAsStream(this.path);
        }
        else if (this.classLoader != null) {
            is = this.classLoader.getResourceAsStream(this.path);
        }
        else {
            is = ClassLoader.getSystemResourceAsStream(this.path);
        }
        if (is == null) {
            throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
        }
        return is;
    }
}
```
我们这边看到ClassPathResource类的构造函数通过构造函数将对象中的path和classLoader赋值，此时path是我们指定的beanFactoryTest.xml，classLoader通过classUtils.getDefaultClassLoader()方法获得，我们这边简单看下这个方法
```
public abstract class ClassUtils {
    @Nullable
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
我们可以看到上面获取加载器的类型，这里的类加载器应该是AppClassLoader。接下来我们来看下XmlBeanFactory如何构造的。
# XmlBeanFactory
```
public class XmlBeanFactory extends DefaultListableBeanFactory {
    private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);

    public XmlBeanFactory(Resource resource) throws BeansException {
        this(resource, null);
    }

    public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
        super(parentBeanFactory);
        this.reader.loadBeanDefinitions(resource);
    }
}
```
我们看到XmlBeanFactory构造函数首先通过super(parentBeanFactory)方法初始化从父类继承的全局属性，然后通过XmlBeanDefinitionReader对象调用loadBeanDefinitions方法。那我们先看下DefaultListableBeanFactory的类关系图
![image.png](/img/doc/springcode/spring2one.png)
我们可以看到DefaultListableBeanFactory有很多上层的接口及类，我们这边暂时不去探究。我们接着往下走到XmlBeanDefinitionReader类
# XmlBeanDefinitionReader
```
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    protected final Log logger = LogFactory.getLog(getClass());

    private final ThreadLocal<Set<EncodedResource>> resourcesCurrentlyBeingLoaded =
            new NamedThreadLocal<>("XML bean definition resources currently being loaded");

    private DocumentLoader documentLoader = new DefaultDocumentLoader();

    private ErrorHandler errorHandler = new SimpleSaxErrorHandler(logger);
    
    @Override
    public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
        return loadBeanDefinitions(new EncodedResource(resource));
    }

    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if (logger.isInfoEnabled()) {
            logger.info("Loading XML bean definitions from " + encodedResource);
        }

        Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        if (currentResources == null) {
            currentResources = new HashSet<>(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }
        if (!currentResources.add(encodedResource)) {
            throw new BeanDefinitionStoreException(
                    "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        }
        try {
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
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

    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {
        try {
            Document doc = doLoadDocument(inputSource, resource);
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

    protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
        return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
                getValidationModeForResource(resource), isNamespaceAware());
    }
    //获取一个Resolver
    protected EntityResolver getEntityResolver() {
        if (this.entityResolver == null) {
            // Determine default EntityResolver to use.
            ResourceLoader resourceLoader = getResourceLoader();
            if (resourceLoader != null) {
                this.entityResolver = new ResourceEntityResolver(resourceLoader);
            }
            else {
                this.entityResolver = new DelegatingEntityResolver(getBeanClassLoader());
            }
        }
        return this.entityResolver;
    }
}
```
我们可以看到上面的步骤分为五步。1：对source进行encode封装，考虑到可能需要编码。2：通过sax读取xml文件方式构建一个inputSource。3:getValidationModeForResource获取xml的验证模式。4：加载xml文件，获得对应的Document。5：通过Document注册Bean信息。至此就完成了一个xm文件到document的转换，这里转换使用的sax解析xml文件的方法，有兴趣的可以debug进去看看。