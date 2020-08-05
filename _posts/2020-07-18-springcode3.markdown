---
layout:     post
title:      "Spring源码(三) -- xml文件解析"
subtitle:   ""
date:       2020-08-05
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring源码
---

# 背景
我们上一节探究了ClassPathResource通过输入Path构造出一个Resource对象，我们这一节继续学习XmlBeanFactory如何解析Resource，将Resource转化成一个BeanDefinition。

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
我们看到XmlBeanFactory构造函数首先通过super(parentBeanFactory)方法初始化从父类继承的全局属性，然后通过XmlBeanDefinitionReader对象调用loadBeanDefinitions方法将Resource转化成BeanDefinition。我们先来看下XmlBeanDefinitionReader的类关系图，![](http://upload-images.jianshu.io/upload_images/9082703-63764eb107dd3d42.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)，我们可以看到XmlBeanDefinitionReader上层是BeanDefinitionReader接口，EnvironmentCapable接口已经父类AbstractBeanDefinitionReader。
- BeanDefinitionReader定义资源文件的加载及转换（转成BeanDefinition）
- EnvironmentCapable就是定义获取Environment的接口，而Environment就是可以通过profile或者property来定义Bean在不同场景中取用哪个Bean数据。
- AbstractBeanDefinitionReader是实现BeanDefinitionReader和EnvironmentCapable的抽象类，并且增加ResourceLoader变量加载Resouce。

# XmlBeanDefinitionReader
```
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    //通过ThreadLocal来防止重复加载
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
            //通过上一节定义的getInputStream方法将文件转换成字节流
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                //将字节流封装成InputSource并加上编码
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
            //sax将xml文件解析成Document，然后将doc注册到BeanDefinition
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
我们可以看到上面的步骤分为五步。1：对source进行encode封装，考虑到可能需要编码。2：通过sax读取xml文件方式构建一个inputSource。3:getValidationModeForResource获取xml的验证模式。4：加载xml文件，获得对应的Document。5：通过Document注册BeanDefinition。至此就完成了一个xml文件到document，document到BeanDefinition的转换，这里转换使用的sax解析xml文件的方法，有兴趣的可以debug进去看看。