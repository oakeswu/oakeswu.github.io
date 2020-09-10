---
layout:     post
title:      "Spring源码(二) -- Xml文件封装成Resource"
subtitle:   ""
date:       2020-07-18
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring源码
---

# 背景
上一节我们编写了一个Ioc的简单测试用例，通过xml文件配置bean文件并使用spring的BeanFactory去获取，我们可以从很多地方查到流程其实是1：ClassPathResource将文件加载成Resource类。2：XmlBeanFactory通过Resource构造XmlBeanFactory。3：XmlBeanFactory调用继承而来的getBean方法获取Bean对象。我们先来看一下第一个步骤：

#Resource
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
我们先看下Resource接口，Resource接口定义了资源的一些属性的方法。那我们看下使用ClassPathResource如何够造出Resource对象。

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

    public ClassPathResource(String path, @Nullable Class<?> clazz) {
        Assert.notNull(path, "Path must not be null");
        this.path = StringUtils.cleanPath(path);
        this.clazz = clazz;
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

    @Deprecated
    protected ClassPathResource(String path, @Nullable ClassLoader classLoader, @Nullable Class<?> clazz) {
        this.path = StringUtils.cleanPath(path);
        this.classLoader = classLoader;
        this.clazz = clazz;
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
    
    //获取当前文件的URL
    @Override
    public URL getURL() throws IOException {
        URL url = resolveURL();
        if (url == null) {
            throw new FileNotFoundException(getDescription() + " cannot be resolved to URL because it does not exist");
        }
        return url;
    }
  
    @Nullable
    protected URL resolveURL() {
        if (this.clazz != null) {
            return this.clazz.getResource(this.path);
        }
        else if (this.classLoader != null) {
            return this.classLoader.getResource(this.path);
        }
        else {
            return ClassLoader.getSystemResource(this.path);
        }
    }
    
    //根据相对路径再创建一个Resource
    @Override
    public Resource createRelative(String relativePath) {
        String pathToUse = StringUtils.applyRelativePath(this.path, relativePath);
        return (this.clazz != null ? new ClassPathResource(pathToUse, this.clazz) :
                new ClassPathResource(pathToUse, this.classLoader));
    }
}
```
我们可以看到ClassPathResource有四个构造函数，最后都是给全局变量path，classLoader，clazz赋值。那我们看下xml加载的流程
1：StringUtils.cleanPath标准化路径
2：对path和classLoader赋值，其中classLoader需要ClassUtils.getDefaultClassLoader()获取
# ClassUtils 
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
我们可以看到上面获取加载器的类型，这里的类加载器应该是AppClassLoader。