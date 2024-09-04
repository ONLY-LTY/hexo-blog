---
title: Spring bean（一）
date: 2016-11-24 15:46:43
tags:
  - Spring
top: true
---
Spring bean 配置文件初始化

&emsp;&emsp;Spring框架基础之模块-IOC，主要提供了依赖注入的实现。甚为重要，小洋闲来无事，再此专研一番。记之、

### 事前准备

&emsp;&emsp;1.  新建一个maven项目，导入最基本的spring包依赖。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>rpc</artifactId>
        <groupId>com.xxx.rpc</groupId>
        <version>1.3.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>spring-bean-init</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
        </dependency>
    </dependencies>
</project>
```
&emsp;&emsp;写个一简单的bean类。

```Java
public class User {
    private String userName;
    private String password;

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getUserName() {

        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    @Override
    public String toString() {
        return "User{" +
                "userName='" + userName + '\'' +
                ", password='" + password + '\'' +
                '}';
    }
}

```
&emsp;&emsp;3.  新建spring.xml文件，配置一下User。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.lty.bean"/>

    <bean id="user" class="com.lty.bean.User">
        <property name="userName" value="LTY"/>
        <property name="password" value="PASSWORD"/>
    </bean>

</beans>
```
&emsp;&emsp;4.  写个启动类，加载spring.xml文件，并获取配置的bean。

```Java
public class Boot {
    public static void main(String[] args) {
        System.out.println(new ClassPathXmlApplicationContext("spring.xml").getBean("user",User.class).toString());
    }

    //ouptput: User{userName='LTY', password='PASSWORD'}
}
```
### 简单设想

&emsp;&emsp;上面是很简单的Spring用法，可以看到小洋自定义的bean已经交给Spring管理了，看输出结果属性的值也初始化了。小洋默默深思，这一切是怎么发生的呢，在Java的世界里能通过运行时去加载的类的，也就只有反射了。是不是通过标签的class属性去加载User，然后在根据配置的属性通过反射设置进去呢。为了解决这个疑问和设想，小洋开始了源码之旅。

### 源码之旅

```Java
//ClassPathXmlApplicationContext构造方法。

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) throws BeansException {
        super(parent);
        //设置配置文件的位置。
        this.setConfigLocations(configLocations);
        if(refresh) {
            //核心方法
            this.refresh();
        }

    }
```
```Java
//ClassPathXmlApplicationContext.refresh()

public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
            //这个是做一些准备工作，打打日志啊，获取一下环境啊什么的，没有做什么重要的事情，不关注
            this.prepareRefresh();
            //初始化一个beanFactory，这个beanFactory很重要，核心的东东。看看他都干了什么。
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                //下面这些方法是Spring初始化需要做的一些事情，小洋只是想看看bean的初始化
                //下面好多方法是不会讲的。等那天小洋闲了在来看看。小洋只会讲finishBeanFactoryInitialization
                //他是bean实例化的一个过程。不过先来看看beanFactory吧。
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var5) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt", var5);
                this.destroyBeans();
                this.cancelRefresh(var5);
                throw var5;
            }

        }
    }

```

```Java
//ClassPathXmlApplicationContext.obtainFreshBeanFactory

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        //核心方法，初始化beanFactory,这个方法是抽象方法，具体的实现是AbstractRefreshableApplicationContext
        this.refreshBeanFactory();
        //这里this.getBeanFactory获取上一步初始化的beanFactory，和这个方法也是抽象的，具体实现和上面一样
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if(this.logger.isDebugEnabled()) {
            this.logger.debug("Bean factory for " + this.getDisplayName() + ": " + beanFactory);
        }

        return beanFactory;
    }

```
```Java
//AbstractRefreshableApplicationContext.refreshBeanFactory

protected final void refreshBeanFactory() throws BeansException {
        if(this.hasBeanFactory()) {
            this.destroyBeans();
            this.closeBeanFactory();
        }

        try {
            //这里创建了一个beanFactory,代码就不用看了直接new DefaultListableBeanFactory这个对象的。
            DefaultListableBeanFactory ex = this.createBeanFactory();
            ex.setSerializationId(this.getId());
            //这个方法是设置初始化bean的一些策略，相同的bean是否允许覆盖，是否允许循环引用
            this.customizeBeanFactory(ex);
            //这个是核心方法，一个bean的配置在Spring内部结构就是BeanDefinition对象，下面看怎么加载的
            //这个调用很长我直接给出最终的调用方法，中间都是些无关紧要的跳转。
            this.loadBeanDefinitions(ex);
            Object var2 = this.beanFactoryMonitor;
            synchronized(this.beanFactoryMonitor) {
                //然后设置一下给成员变量。
                this.beanFactory = ex;
            }
        } catch (IOException var5) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
        }
    }

```

```Java
//DefaultBeanDefinitionDocumentReader.doRegisterBeanDefinitions

protected void doRegisterBeanDefinitions(Element root) {
        //创建一个代理对象去解析bean
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = this.createDelegate(this.getReaderContext(), root, parent);
        if(this.delegate.isDefaultNamespace(root)) {
            String profileSpec = root.getAttribute("profile");
            if(StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, ",; ");
                if(!this.getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    return;
                }
            }
        }
        //预处理xml
        this.preProcessXml(root);
        //解析bean，核心方法。
        this.parseBeanDefinitions(root, this.delegate);
        //后处理xml
        this.postProcessXml(root);
        this.delegate = parent;
    }

```
```Java
//DefaultBeanDefinitionDocumentReader.parseBeanDefinitions

protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        //这里root是xml解析后的根元素，判断是否是默认的命名空间。也就是http://www.springframework.org/schema/beans
        //仔细看xml文件就能发现这个命名空间是配置bean相关的，比如<bean><import><alias><description>等标签
        if(delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            //循环去解析xml配置的标签
            for(int i = 0; i < nl.getLength(); ++i) {
                Node node = nl.item(i);
                if(node instanceof Element) {
                    Element ele = (Element)node;
                    if(delegate.isDefaultNamespace(ele)) {
                        //处理默认的命名空间，我们在前期准备中是通过配置<bean>，是默认的命名空间下的标签，所以这里进入是这个方法。
                        this.parseDefaultElement(ele, delegate);
                    } else {
                        //处理自定义的 比如我们常用的http://www.springframework.org/schema/context
                        //<context:component-scan base-package="com.xxx.rpc.sample.server"/>
                        //这个就是扫描包的配置。我们列举这一个命名空间的解析。后面文章会讲到。
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        } else {
            delegate.parseCustomElement(root);
        }

    }

```
&emsp;&emsp;上我们可以看到，当解析到具体的某个标签的时候，Spring有两个分支，自定义和默认的。很好理解我们可以通过在xml文件配置<bean>来让Spring初始化，也可以什么都不写直接在类上面加Component注解就行，然后在xml文件指定一些Spring的扫描路径。下面我先看通过<bean>配置的解析。

```Java
//DefaultBeanDefinitionDocumentReader.parseDefaultElement

private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if(delegate.nodeNameEquals(ele, "import")) {
            //处理import标签
            this.importBeanDefinitionResource(ele);
        } else if(delegate.nodeNameEquals(ele, "alias")) {
            this.processAliasRegistration(ele);
        } else if(delegate.nodeNameEquals(ele, "bean")) {
            //处理bean标签，我们看这个方法。
            this.processBeanDefinition(ele, delegate);
        } else if(delegate.nodeNameEquals(ele, "beans")) {
            //处理beans标签
            this.doRegisterBeanDefinitions(ele);
        }

    }
```
```Java
//DefaultBeanDefinitionDocumentReader.processBeanDefinition

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //通过代理去解析bean标签，然后组装成BeanDefinitionHolder，这个BeanDefinitionHolder是BeanDefinition的包装类。
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if(bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);

            try {
                //讲上面解析到的BeanDefinition注册到beanFactory当中。最开始我们创建的DefaultListableBeanFactory有个成员变量
                //private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap(64);
                //这里的注册就是讲解析出来的BeanDefinition放入到这个map当中，这个map后面我们实例化bean的时候会用到。切记我们解析
                //出来的BeanDefinition不是bean的实例对象，只是把bean标签里面的内容，以Java对象的数据结构来表示。
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, this.getReaderContext().getRegistry());
            } catch (BeanDefinitionStoreException var5) {
                this.getReaderContext().error("Failed to register bean definition with name \'" + bdHolder.getBeanName() + "\'", ele, var5);
            }

            this.getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }

    }
```

```Java
//BeanDefinitionParserDelegate.parseBeanDefinitionElement

public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
        //小洋看到这里，看到了希望，走了那么远，终于看见<bean>里面的id啊name了。这里就是解析了
        String id = ele.getAttribute("id");
        String nameAttr = ele.getAttribute("name");
        ArrayList aliases = new ArrayList();
        if(StringUtils.hasLength(nameAttr)) {
            String[] beanName = StringUtils.tokenizeToStringArray(nameAttr, ",; ");
            aliases.addAll(Arrays.asList(beanName));
        }

        String beanName1 = id;
        if(!StringUtils.hasText(id) && !aliases.isEmpty()) {
            beanName1 = (String)aliases.remove(0);
            if(this.logger.isDebugEnabled()) {
                this.logger.debug("No XML \'id\' specified - using \'" + beanName1 + "\' as bean name and " + aliases + " as aliases");
            }
        }

        if(containingBean == null) {
            //检查一下bean名称是否重复啊
            this.checkNameUniqueness(beanName1, aliases, ele);
        }
        //这里解析配置<bean>里面属性。这里是关键。
        AbstractBeanDefinition beanDefinition = this.parseBeanDefinitionElement(ele, beanName1, containingBean);
        if(beanDefinition != null) {
            if(!StringUtils.hasText(beanName1)) {
                try {
                    if(containingBean != null) {
                        beanName1 = BeanDefinitionReaderUtils.generateBeanName(beanDefinition, this.readerContext.getRegistry(), true);
                    } else {
                        beanName1 = this.readerContext.generateBeanName(beanDefinition);
                        String aliasesArray = beanDefinition.getBeanClassName();
                        if(aliasesArray != null && beanName1.startsWith(aliasesArray) && beanName1.length() > aliasesArray.length() && !this.readerContext.getRegistry().isBeanNameInUse(aliasesArray)) {
                            aliases.add(aliasesArray);
                        }
                    }

                    if(this.logger.isDebugEnabled()) {
                        this.logger.debug("Neither XML \'id\' nor \'name\' specified - using generated bean name [" + beanName1 + "]");
                    }
                } catch (Exception var9) {
                    this.error(var9.getMessage(), ele);
                    return null;
                }
            }

            String[] aliasesArray1 = StringUtils.toStringArray(aliases);
            //然后包装beanDefinition
            return new BeanDefinitionHolder(beanDefinition, beanName1, aliasesArray1);
        } else {
            return null;
        }
    }

```
```Java
public AbstractBeanDefinition parseBeanDefinitionElement(Element ele, String beanName, BeanDefinition containingBean) {
       //这里就是解析的最底层了，小洋喝了杯水，看了看天空的月亮，看见了嫦娥。
        this.parseState.push(new BeanEntry(beanName));
        String className = null;
        if(ele.hasAttribute("class")) {
            className = ele.getAttribute("class").trim();
        }

        try {
            String ex = null;
            if(ele.hasAttribute("parent")) {
                ex = ele.getAttribute("parent");
            }
            //创建BeanDefinition对象，用来封装解析处理的数据。
            AbstractBeanDefinition bd = this.createBeanDefinition(className, ex);
            //解析属性，这里就是解析我们配置singleton啊scope啊lazy-init啊这些等等。
            this.parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
            //设置描述
            bd.setDescription(DomUtils.getChildElementValueByTagName(ele, "description"));
            //解析meta标签
            this.parseMetaElements(ele, bd);
            //解析lookup-method标签
            this.parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            //机械replaced-method标签
            this.parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
            //解析构造函数的参数
            this.parseConstructorArgElements(ele, bd);
            //解析property，我们要注入的属性值
            this.parsePropertyElements(ele, bd);
            //解析qualifier标签
            this.parseQualifierElements(ele, bd);
            bd.setResource(this.readerContext.getResource());
            bd.setSource(this.extractSource(ele));
            AbstractBeanDefinition var7 = bd;
            return var7;
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
&emsp;&emsp;到此bean解析成BeanDefinition数据对象，就到此结束了，小洋也感觉到筋疲力尽了，但感觉还是很有成就感的，正准备歇息一番，突然一想，不妙啊，这里只是把bean标签解析成了BeanDefinition对象，可是我们getBean拿到的是一个实例对象啊。还有上面只看到了配置bean处理，还没有看扫描注解包的处理。小洋对此只能仰天长叹，道路还很长啊。后面看小洋如何披荆斩棘，雄霸天下。

---

<p  align='center'><font color='blue' >我读的书愈多，就愈亲近世界，愈明了生活的意义，愈觉得生活的重要。</font></p><p align='right'> —— 高尔基</p>

---
