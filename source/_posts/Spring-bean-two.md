---
title: Spring bean（二）
date: 2016-11-24 19:20:06
tags:
  - Spring
photos:
  - /img/20161129-2.png
---
Spring bean 扫描注解初始化
<!--more-->
&emsp;&emsp;天和日丽，阳光明媚，小洋接着看Spring bean。上篇文章我们说了，Spring初始化bean的时候有两种方式，一个是在xml里面配置<bean>相关内容。一个是用注解扫描。

```Java
//DefaultBeanDefinitionDocumentReader.parseBeanDefinitions

protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if(delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();

            for(int i = 0; i < nl.getLength(); ++i) {
                Node node = nl.item(i);
                if(node instanceof Element) {
                    Element ele = (Element)node;
                    if(delegate.isDefaultNamespace(ele)) {
                        this.parseDefaultElement(ele, delegate);
                    } else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        } else {
            delegate.parseCustomElement(root);
        }

    }

```
&emsp;&emsp;上次小洋看到这里的时候，进入的是parseDefaultElement这个方法，这个方法是处理默认命名空间的标签的，当我们配置扫描包的时候，比如context:component-scan base-package="" 这个不是默认命名空间下的标签，他的命名空间是http://www.springframework.org/schema/context。 所以会进入parseCustomElement这个方法。

```Java
//BeanDefinitionParserDelegate.parseCustomElement

public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
        String namespaceUri = this.getNamespaceURI(ele);
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if(handler == null) {
            this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        } else {
            //具体的解析方法。交给NamespaceHandlerSupport去处理的。
            return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
        }
    }
```
```Java
//NamespaceHandlerSupport.parse

public BeanDefinition parse(Element element, ParserContext parserContext) {
       //先根据而element找出合适的BeanDefinitionParser，然后在调用parse解析。
       return this.findParserForElement(element, parserContext).parse(element, parserContext);
   }
```
```Java

private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
       //根据element得的解析器的名称。小洋这里得到的是component-scan，这里很好理解配置的就是扫描包嘛
       String localName = parserContext.getDelegate().getLocalName(element);
       //直接从map里面获取对应的BeanDefinition。spring这里给我初始化了8个具体的解析bean用的对象。
       //具体的看下面的debug图，这里我们是ComponentScanBeanDefinitionParser这个解析.
       BeanDefinitionParser parser = (BeanDefinitionParser)this.parsers.get(localName);
       if(parser == null) {
           parserContext.getReaderContext().fatal("Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
       }

       return parser;
   }
```
![Alt text](/img/QQ20161128-0.png)

```Java
//ComponentScanBeanDefinitionParser.parse

public BeanDefinition parse(Element element, ParserContext parserContext) {
        //获取配置的package
        String basePackage = element.getAttribute("base-package");
        basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
        //将配置的对个package放到String数组里面
        String[] basePackages = StringUtils.tokenizeToStringArray(basePackage, ",; \t\n");
        //新建一个ClassPathBeanDefinitionScanner。
        ClassPathBeanDefinitionScanner scanner = this.configureScanner(parserContext, element);
        //去扫描包了。具体看这里
        Set beanDefinitions = scanner.doScan(basePackages);
        this.registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
        return null;
    }
```
```Java
//ClassPathBeanDefinitionScanner.doScan

protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        LinkedHashSet beanDefinitions = new LinkedHashSet();
        String[] var3 = basePackages;
        int var4 = basePackages.length;
        //扫描配置的每个package
        for(int var5 = 0; var5 < var4; ++var5) {
            String basePackage = var3[var5];
            //根据名称我们就能看到找到候选人，就是加Components注解的bean。核心
            Set candidates = this.findCandidateComponents(basePackage);
            Iterator var8 = candidates.iterator();

            while(var8.hasNext()) {
                BeanDefinition candidate = (BeanDefinition)var8.next();
                ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
                candidate.setScope(scopeMetadata.getScopeName());
                String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
                //根据不同的类型去做不同的处理
                if(candidate instanceof AbstractBeanDefinition) {
                    this.postProcessBeanDefinition((AbstractBeanDefinition)candidate, beanName);
                }

                if(candidate instanceof AnnotatedBeanDefinition) {
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition)candidate);
                }

                if(this.checkCandidate(beanName, candidate)) {
                   //封装成BeanDefinitionHolder
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    //注册到beanFactory
                    this.registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }

        return beanDefinitions;
    }

```
```Java
//ClassPathBeanDefinitionScanner.findCandidateComponents

public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        LinkedHashSet candidates = new LinkedHashSet();

        try {
            String ex = "classpath*:" + this.resolveBasePackage(basePackage) + "/" + this.resourcePattern;
            //这里去获取package包下面的class资源，核心，接着看
            Resource[] resources = this.resourcePatternResolver.getResources(ex);
            boolean traceEnabled = this.logger.isTraceEnabled();
            boolean debugEnabled = this.logger.isDebugEnabled();
            Resource[] var7 = resources;
            int var8 = resources.length;
            //循环每个class资源
            for(int var9 = 0; var9 < var8; ++var9) {
                Resource resource = var7[var9];
                if(traceEnabled) {
                    this.logger.trace("Scanning " + resource);
                }

                if(resource.isReadable()) {
                    try {
                        //这里通过ASM去解析class文件。
                        MetadataReader ex1 = this.metadataReaderFactory.getMetadataReader(resource);
                        //判断是是否有Component注解
                        if(this.isCandidateComponent(ex1)) {
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(ex1);
                            sbd.setResource(resource);
                            sbd.setSource(resource);
                            if(this.isCandidateComponent((AnnotatedBeanDefinition)sbd)) {
                                if(debugEnabled) {
                                    this.logger.debug("Identified candidate component class: " + resource);
                                }
                                //加入到候选人集合
                                candidates.add(sbd);
                            } else if(debugEnabled) {
                                this.logger.debug("Ignored because not a concrete top-level class: " + resource);
                            }
                        } else if(traceEnabled) {
                            this.logger.trace("Ignored because not matching any filter: " + resource);
                        }
                    } catch (Throwable var13) {
                        throw new BeanDefinitionStoreException("Failed to read candidate component class: " + resource, var13);
                    }
                } else if(traceEnabled) {
                    this.logger.trace("Ignored because not readable: " + resource);
                }
            }

            return candidates;
        } catch (IOException var14) {
            throw new BeanDefinitionStoreException("I/O failure during classpath scanning", var14);
        }
    }
```
```Java
//PathMatchingResourcePatternResolver.doFindAllClassPathResources

protected Set<Resource> doFindAllClassPathResources(String path) throws IOException {
        LinkedHashSet result = new LinkedHashSet(16);
        ClassLoader cl = this.getClassLoader();
        //这里通过classLoader去加载资源。
        Enumeration resourceUrls = cl != null?cl.getResources(path):ClassLoader.getSystemResources(path);

        while(resourceUrls.hasMoreElements()) {
            URL url = (URL)resourceUrls.nextElement();
            result.add(this.convertClassLoaderURL(url));
        }

        if("".equals(path)) {
            this.addAllClassLoaderJarRoots(cl, result);
        }

        return result;
    }

```

&emsp;&emsp;小洋这就看完了处理非默认命名空间的元素解析，这里具体用component-scan这个标签来举例说的，Spring针对不同的标签，会用具体的对象去处理解析，改天再去看看其他标签的解析。小洋这里要嘱咐一下，这里我们不管是配置的bean还是扫描的bean，现在在Spring里面就只是一个java对象的数据结构BeanDefinition,并没有实例化。

---

<p  align='center'><font color='blue' >笨蛋自以为聪明，聪明人才知道自己是笨蛋。</font></p><p align='right'> —— 莎士比亚</p>

---
