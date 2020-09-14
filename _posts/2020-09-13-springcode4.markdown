---
layout:     post
title:      "Spring源码(四) -- BeanDefinition的注册"
subtitle:   ""
date:       2020-09-13
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spring源码
---

# 背景
上一节我们对XML文件通过SAX解析成Document进行了分析，这一节我们将学习Spring如何将已转成Document对象的XML转化成BeanDefinition。

# 注册BeanDefinition
```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 通过全局变量documentReaderClass获得一个DefaultBeanDefinitionDocumentReader对象
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    // 获得解析前BeanDefinition的加载个数
    int countBefore = getRegistry().getBeanDefinitionCount();
    //加载注册BeanDefinition
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    //返回本次加载的BeanDefinion个数
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
我们看到上次获取Document之后调用registerBeanDefinitions方法解析并注册成BeanDefinition。这里documentReader是DefaultBeanDefinitionDocumentReader实例，所以我们现在再看下DefaultBeanDefinitionDocumentReader里面的registerBeanDefinitions方法。

```
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
    @Override
    public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
        this.readerContext = readerContext;
        logger.debug("Loading bean definitions");
        //获得Doc的root节点
        Element root = doc.getDocumentElement();
        doRegisterBeanDefinitions(root);
    }
    protected void doRegisterBeanDefinitions(Element root) {
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);
        //判断是否是root节点，如果是就需要判断profile是否配置并解析
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
        //子类实现解析前的处理
        preProcessXml(root);
        parseBeanDefinitions(root, this.delegate);
        //子类实现解析后的处理
        postProcessXml(root);

        this.delegate = parent;
    }
    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    if (delegate.isDefaultNamespace(ele)) {
                        //解析常用Element
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        //解析自定义Element
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        else {
            delegate.parseCustomElement(root);
        }
    }
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
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            //通过delegate获取解析到的BeanDefinitionHolder 
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                //将BeanDefinitionHolder放到BeanFactory里面的beanDefinitionMap中
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
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
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
}
```
我们这里梳理下逻辑：
- 判断root节点是通过节点的namespaceUri来判断的，如果为空或者uri为xml中定义的http://www.springframework.org/schema/beans。
- profile也是beans根节点里面的配置，如果是根节点需要判断，profile可以百度了解一下。
- 开始parseBeanDefinitions方法，解析具体的节点配置。我们这里只看下常用Element里面的Bean的解析。
- processBeanDefinition里面就能看出BeanDefinition对象和标签节点的关系。将delegate解析到的BeanDefinitionHolder放入实现过BeanDefinitionRegistry接口里面的beanDefinitionMap

```
public class BeanDefinitionParserDelegate {
    public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef) {
		return decorateBeanDefinitionIfRequired(ele, originalDef, null);
	}
    public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
			Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

		BeanDefinitionHolder finalDefinition = originalDef;

		// Decorate based on custom attributes first.
		NamedNodeMap attributes = ele.getAttributes();
		for (int i = 0; i < attributes.getLength(); i++) {
			Node node = attributes.item(i);
			finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
		}

		// Decorate based on custom nested elements.
		NodeList children = ele.getChildNodes();
		for (int i = 0; i < children.getLength(); i++) {
			Node node = children.item(i);
			if (node.getNodeType() == Node.ELEMENT_NODE) {
				finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
			}
		}
		return finalDefinition;
	}
}
```
```
public class BeanDefinitionReaderUtils {
    public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
}
```
从上面的代码就能看出最后会放到register里面的BeanDefinitionMap中，并且会将别名Alias也放进去，这里很重要，因为在最后getBean主要用到这里的BeanDefinitionMap等数据。