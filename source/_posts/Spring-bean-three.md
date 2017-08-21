---
title: Spring bean（三）
date: 2016-11-24 19:24:11
tags:
 - Spring
photos:
 - /img/20161201.jpg
---
Spring bean 实例化
<!--more-->

&emsp;&emsp;小洋前面的已经看到了Spring bean的解析了。可是聪明的小洋发现那时候的bean并没有实例化。抱着不到黄河不死心的想法，小洋又开始了艰难的旅程。

```Java
//AbstractApplicationContext.refresh

public void refresh() throws BeansException, IllegalStateException {
    Object var1 = this.startupShutdownMonitor;
    synchronized(this.startupShutdownMonitor) {
        this.prepareRefresh();
        //解析bean
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        this.prepareBeanFactory(beanFactory);

        try {
            this.postProcessBeanFactory(beanFactory);
            this.invokeBeanFactoryPostProcessors(beanFactory);
            this.registerBeanPostProcessors(beanFactory);
            this.initMessageSource();
            this.initApplicationEventMulticaster();
            this.onRefresh();
            this.registerListeners();
            //预实例化bean，本文主要方法
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
&emsp;&emsp;小洋又回到了开始Spring初始化的时候，获取beanFactory之后还有好多个方法，根据名称其实已经能看出来都干什么的，小洋主要看了finishBeanFactoryInitialization这个方法，是做预实例化bean的，为什么叫预实例化呢。经过小洋一番研究，发现Spring在初始化的时候将配置的bean或者扫描的bean解析成BeanDefinition数据对象，之后并不是将所有解析出来的bean在Spring初始化的就实例化。只有特定条件的bean才会在Spring初始化的时候实例化。什么条件呢？容小洋娓娓道来。

```Java
//AbstractApplicationContext.finishBeanFactoryInitialization

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    if(beanFactory.containsBean("conversionService") && beanFactory.isTypeMatch("conversionService", ConversionService.class)) {
        beanFactory.setConversionService((ConversionService)beanFactory.getBean("conversionService", ConversionService.class));
    }

    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    String[] var3 = weaverAwareNames;
    int var4 = weaverAwareNames.length;

    for(int var5 = 0; var5 < var4; ++var5) {
        String weaverAwareName = var3[var5];
        this.getBean(weaverAwareName);
    }

    beanFactory.setTempClassLoader((ClassLoader)null);
    beanFactory.freezeConfiguration();
    //实例化bean，看这个东东，前面就是一些其他的处理，不关注。
    beanFactory.preInstantiateSingletons();
}

```
```Java
//DefaultListableBeanFactory.preInstantiateSingletons

public void preInstantiateSingletons() throws BeansException {
        if(this.logger.isDebugEnabled()) {
            this.logger.debug("Pre-instantiating singletons in " + this);
        }
        //这里构造了一List，this.beanDefinitionNames是前面解析bean的时候，将所有bean的名字
        //放入一个list
        ArrayList beanNames = new ArrayList(this.beanDefinitionNames);
        Iterator var2 = beanNames.iterator();

        while(true) {
            while(true) {
                String beanName;
                RootBeanDefinition singletonInstance;
                do {
                    do {
                        do {
                            if(!var2.hasNext()) {
                                var2 = beanNames.iterator();

                                while(var2.hasNext()) {
                                    beanName = (String)var2.next();
                                    Object singletonInstance1 = this.getSingleton(beanName);
                                    if(singletonInstance1 instanceof SmartInitializingSingleton) {
                                        final SmartInitializingSingleton smartSingleton1 = (SmartInitializingSingleton)singletonInstance1;
                                        if(System.getSecurityManager() != null) {
                                            AccessController.doPrivileged(new PrivilegedAction() {
                                                public Object run() {
                                                    smartSingleton1.afterSingletonsInstantiated();
                                                    return null;
                                                }
                                            }, this.getAccessControlContext());
                                        } else {
                                            smartSingleton1.afterSingletonsInstantiated();
                                        }
                                    }
                                }

                                return;
                            }

                            beanName = (String)var2.next();
                            //getMergedLocalBeanDefinition这个方法就是根据beanName去获取BeanDefinition.
                            //其实就是从前面解析bean的时候构造的map里面获取。
                            singletonInstance = this.getMergedLocalBeanDefinition(beanName);
                            //上面的处理无关紧要，主要是几个while的条件。满足这几条件就会一直while循环，直到里面return
                        } while(singletonInstance.isAbstract());
                        //不是单例的
                    } while(!singletonInstance.isSingleton());
                    //是懒加载的
                } while(singletonInstance.isLazyInit());
                //上面可以看到，只有bean是 单例的并且不是懒加载的 才会执行下面的方法。
                //判断是不是FactoryBean,我们配置的bean会返回false。直接走else
                if(this.isFactoryBean(beanName)) {
                    final FactoryBean smartSingleton = (FactoryBean)this.getBean("&" + beanName);
                    boolean isEagerInit;
                    if(System.getSecurityManager() != null && smartSingleton instanceof SmartFactoryBean) {
                        isEagerInit = ((Boolean)AccessController.doPrivileged(new PrivilegedAction() {
                            public Boolean run() {
                                return Boolean.valueOf(((SmartFactoryBean)smartSingleton).isEagerInit());
                            }
                        }, this.getAccessControlContext())).booleanValue();
                    } else {
                        isEagerInit = smartSingleton instanceof SmartFactoryBean && ((SmartFactoryBean)smartSingleton).isEagerInit();
                    }

                    if(isEagerInit) {
                        this.getBean(beanName);
                    }
                } else {
                    //核心方法，获取bean实例。
                    this.getBean(beanName);
                }
            }
        }
    }

```
&emsp;&emsp;小洋发现在只有是单例模式的并且不是懒加载的bean才会在Spring初始化的实例化。

```Java
//AbstractBeanFactory.doGetBean

protected <T> T doGetBean(String name, Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {
        final String beanName = this.transformedBeanName(name);
        //AbstractBeanFactory的父类FactoryBeanRegistrySupport有个成员属性singletonObjects
        //private final Map<String, Object> singletonObjects = new ConcurrentHashMap(64);
        //我们实例化的单利模式的bean都会放入到这个集合中，Spring初始化的时候肯定没有，返回的是null，
        Object sharedInstance = this.getSingleton(beanName);
        Object bean;
        if(sharedInstance != null && args == null) {
            if(this.logger.isDebugEnabled()) {
                if(this.isSingletonCurrentlyInCreation(beanName)) {
                    this.logger.debug("Returning eagerly cached instance of singleton bean \'" + beanName + "\' that is not fully initialized yet - a consequence of a circular reference");
                } else {
                    this.logger.debug("Returning cached instance of singleton bean \'" + beanName + "\'");
                }
            }

            bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
        } else {
            if(this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }
            //如果有父BeanFactory则交给父BeanFactory去处理，和Java的类加载机制很像。
            //我们这里DefaultListableBeanFactory没有父BeanFactory所以这里的条件满足不了
            BeanFactory ex = this.getParentBeanFactory();
            if(ex != null && !this.containsBeanDefinition(beanName)) {
                String var24 = this.originalBeanName(name);
                if(args != null) {
                    return ex.getBean(var24, args);
                }

                return ex.getBean(var24, requiredType);
            }

            if(!typeCheckOnly) {
                this.markBeanAsCreated(beanName);
            }

            try {
                //现根据beanName获取BeanDefinition
                final RootBeanDefinition ex1 = this.getMergedLocalBeanDefinition(beanName);
                this.checkMergedBeanDefinition(ex1, beanName, args);
                //获取bean的依赖
                String[] dependsOn = ex1.getDependsOn();
                String[] scopeName;
                if(dependsOn != null) {
                    scopeName = dependsOn;
                    int scope = dependsOn.length;
                    //循环初始化bean的依赖
                    for(int ex2 = 0; ex2 < scope; ++ex2) {
                        String dependsOnBean = scopeName[ex2];
                        if(this.isDependent(beanName, dependsOnBean)) {
                            throw new BeanCreationException(ex1.getResourceDescription(), beanName, "Circular depends-on relationship between \'" + beanName + "\' and \'" + dependsOnBean + "\'");
                        }

                        this.registerDependentBean(dependsOnBean, beanName);
                        this.getBean(dependsOnBean);
                    }
                }
                //bean如果是单利模式的
                if(ex1.isSingleton()) {
                    sharedInstance = this.getSingleton(beanName, new ObjectFactory() {
                        public Object getObject() throws BeansException {
                            try {
                                //创建bean核心方法
                                return AbstractBeanFactory.this.createBean(beanName, ex1, args);
                            } catch (BeansException var2) {
                                AbstractBeanFactory.this.destroySingleton(beanName);
                                throw var2;
                            }
                        }
                    });
                    bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, ex1);
                    //如果是Prototype类型的
                } else if(ex1.isPrototype()) {
                    scopeName = null;

                    Object var25;
                    try {
                        this.beforePrototypeCreation(beanName);
                        //创建bean
                        var25 = this.createBean(beanName, ex1, args);
                    } finally {
                        this.afterPrototypeCreation(beanName);
                    }

                    bean = this.getObjectForBeanInstance(var25, name, beanName, ex1);
                  //其他类型的
                } else {
                    String var26 = ex1.getScope();
                    Scope var27 = (Scope)this.scopes.get(var26);
                    if(var27 == null) {
                        throw new IllegalStateException("No Scope registered for scope \'" + var26 + "\'");
                    }

                    try {
                        Object var28 = var27.get(beanName, new ObjectFactory() {
                            public Object getObject() throws BeansException {
                                AbstractBeanFactory.this.beforePrototypeCreation(beanName);

                                Object var1;
                                try {
                                    //创建bean
                                    var1 = AbstractBeanFactory.this.createBean(beanName, ex1, args);
                                } finally {
                                    AbstractBeanFactory.this.afterPrototypeCreation(beanName);
                                }

                                return var1;
                            }
                        });
                        bean = this.getObjectForBeanInstance(var28, name, beanName, ex1);
                    } catch (IllegalStateException var21) {
                        throw new BeanCreationException(beanName, "Scope \'" + var26 + "\' is not active for the current thread; " + "consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var21);
                    }
                }
            } catch (BeansException var23) {
                this.cleanupAfterBeanCreationFailure(beanName);
                throw var23;
            }
        }

        //......省略一部分代码

        return bean;

    }

```
&emsp;&emsp;创建bean调用的核心方法都一样，只是区分了isSingleton或isPrototype或者其他的。对于singleton的会放在缓存里面，下次直接获取。而prototype则没有放入缓存。下面看创建bean的方法。

```Java
//AbstractAutowireCapableBeanFactory.doCreateBean

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, Object[] args) {
        //先创建一个bean的包装类
        BeanWrapper instanceWrapper = null;
        if(mbd.isSingleton()) {
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }
        //初始化的时候肯定为空
        if(instanceWrapper == null) {
            //创建bean的实例
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }

        final Object bean = instanceWrapper != null?instanceWrapper.getWrappedInstance():null;

        //......省略部分代码

        Object exposedObject = bean;

        try {
            //这里核心 bean实例的属性注入
            this.populateBean(beanName, mbd, instanceWrapper);
            if(exposedObject != null) {
                exposedObject = this.initializeBean(beanName, exposedObject, mbd);
            }
        } catch (Throwable var17) {
            if(var17 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var17).getBeanName())) {
                throw (BeanCreationException)var17;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var17);
        }
        //下面还有部分代码 不重要不看了。
}
```
```Java
//AbstractAutowireCapableBeanFactory.createBeanInstance

protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
        //根据反射创建class对象，mbd里面包含了我们解析出来bean的一些属性，这里其实就是根据class属性，通过Class.forName
        Class beanClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        if(beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Bean class isn\'t public, and non-public access not allowed: " + beanClass.getName());
        } else if(mbd.getFactoryMethodName() != null) {
            return this.instantiateUsingFactoryMethod(beanName, mbd, args);
        } else {
            boolean resolved = false;
            boolean autowireNecessary = false;
            if(args == null) {
                Object ctors = mbd.constructorArgumentLock;
                synchronized(mbd.constructorArgumentLock) {
                    if(mbd.resolvedConstructorOrFactoryMethod != null) {
                        resolved = true;
                        autowireNecessary = mbd.constructorArgumentsResolved;
                    }
                }
            }

            if(resolved) {
                return autowireNecessary?this.autowireConstructor(beanName, mbd, (Constructor[])null, (Object[])null):this.instantiateBean(beanName, mbd);
            } else {
                //得到bean的class对象后，会执行到这个分支，对于无参构造函数，instantiateBean就是创建bean的实例。
                //对于配置了构造函数参数的，autowireConstructor进行实例化。其实底层都是通过反射。只是多了参数处理
                Constructor[] ctors1 = this.determineConstructorsFromBeanPostProcessors(beanClass, beanName);
                return ctors1 == null && mbd.getResolvedAutowireMode() != 3 && !mbd.hasConstructorArgumentValues() && ObjectUtils.isEmpty(args)?this.instantiateBean(beanName, mbd):this.autowireConstructor(beanName, mbd, ctors1, args);
            }
        }
    }

```
```Java
//AbstractBeanDefinition.resolveBeanClass

public Class<?> resolveBeanClass(ClassLoader classLoader) throws ClassNotFoundException {
        String className = this.getBeanClassName();
        if(className == null) {
            return null;
        } else {
            //这里就是通过forName去获取Class对象。
            Class resolvedClass = ClassUtils.forName(className, classLoader);
            //将得到的Class对象赋值给beanClass给后面实例化的用。
            this.beanClass = resolvedClass;
            return resolvedClass;
        }
    }
```
```Java
//SimpleInstantiationStrategy.instantiate 这里的实例化是无参构造函数

public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
        if(bd.getMethodOverrides().isEmpty()) {
            Object var5 = bd.constructorArgumentLock;
            Constructor constructorToUse;
            synchronized(bd.constructorArgumentLock) {
                constructorToUse = (Constructor)bd.resolvedConstructorOrFactoryMethod;
                if(constructorToUse == null) {
                    //获取到上一步创建的bean的Class对象
                    final Class clazz = bd.getBeanClass();
                    //判断一下，不能是接口
                    if(clazz.isInterface()) {
                        throw new BeanInstantiationException(clazz, "Specified class is an interface");
                    }

                    try {
                        if(System.getSecurityManager() != null) {

                            constructorToUse = (Constructor)AccessController.doPrivileged(new PrivilegedExceptionAction() {
                                public Constructor<?> run() throws Exception {
                                   //获取默认的构造方法
                                    return clazz.getDeclaredConstructor((Class[])null);
                                }
                            });
                        } else {
                            constructorToUse = clazz.getDeclaredConstructor((Class[])null);
                        }
                        //将获取到的构造方法设置到BeanDefinition里面
                        bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                    } catch (Exception var9) {
                        throw new BeanInstantiationException(clazz, "No default constructor found", var9);
                    }
                }
            }
            //反射实例化bean 可以看到参数是Object[0]
            return BeanUtils.instantiateClass(constructorToUse, new Object[0]);
        } else {
            //这里是CGLIB实例化，不关注
            return this.instantiateWithMethodInjection(bd, beanName, owner);
        }
    }

```
```Java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
        Assert.notNull(ctor, "Constructor must not be null");

        try {
            //小洋发现其实看了那么多，其实核心就这一句
            ReflectionUtils.makeAccessible(ctor);
            return ctor.newInstance(args);
        } catch (InstantiationException var3) {
            throw new BeanInstantiationException(ctor.getDeclaringClass(), "Is it an abstract class?", var3);
        } catch (IllegalAccessException var4) {
            throw new BeanInstantiationException(ctor.getDeclaringClass(), "Is the constructor accessible?", var4);
        } catch (IllegalArgumentException var5) {
            throw new BeanInstantiationException(ctor.getDeclaringClass(), "Illegal arguments for constructor", var5);
        } catch (InvocationTargetException var6) {
            throw new BeanInstantiationException(ctor.getDeclaringClass(), "Constructor threw exception", var6.getTargetException());
        }
    }

```
&emsp;&emsp;看到这里已经看到了bean的实例化了。其实就是通过反射获取构造方法然后执行newInstance来实例化的。实例化就该干什么呢，属性注入了。

```Java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, Object[] args) {
        BeanWrapper instanceWrapper = null;
        if(mbd.isSingleton()) {
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }

        if(instanceWrapper == null) {
            //上面这里就返回了bean的实例对象了。然后接着往下看
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }

        final Object bean = instanceWrapper != null?instanceWrapper.getWrappedInstance():null;
        Class beanType = instanceWrapper != null?instanceWrapper.getWrappedClass():null;
        Object earlySingletonExposure = mbd.postProcessingLock;
        synchronized(mbd.postProcessingLock) {
            if(!mbd.postProcessed) {
                this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                mbd.postProcessed = true;
            }
        }

        //........


        Object exposedObject = bean;

        try {
            //这里是具体的对实例对象进行属性注入了。小洋进去看看
            this.populateBean(beanName, mbd, instanceWrapper);
            if(exposedObject != null) {
                exposedObject = this.initializeBean(beanName, exposedObject, mbd);
            }
        } catch (Throwable var17) {
            if(var17 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var17).getBeanName())) {
                throw (BeanCreationException)var17;
            }

            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var17);
        }

        //........
}
```
```Java
//AbstractAutowireCapableBeanFactory.populateBean

protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
       //首先获取解析出来的需要注入的属性，小洋叮嘱 这里只是我们xml文件配置bean的property。
       Object pvs = mbd.getPropertyValues();
       if(bw == null) {
           if(!((PropertyValues)pvs).isEmpty()) {
               throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
           }
       } else {

           //....... 省略部分代码

               boolean hasInstAwareBpps2 = this.hasInstantiationAwareBeanPostProcessors();
               boolean needsDepCheck1 = mbd.getDependencyCheck() != 0;
               if(hasInstAwareBpps2 || needsDepCheck1) {
                   PropertyDescriptor[] filteredPds1 = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                   if(hasInstAwareBpps2) {
                       //这里获取BeanPostProcessors去处理属性注入。Spring会每个都去尝试
                       Iterator var9 = this.getBeanPostProcessors().iterator();

                       while(var9.hasNext()) {
                           BeanPostProcessor bp = (BeanPostProcessor)var9.next();
                           //判断是不是用来处理自动注入的bean，也就是InstantiationAwareBeanPostProcessor
                           if(bp instanceof InstantiationAwareBeanPostProcessor) {
                               InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
                               //这里调用方法去注入属性，这里有好多个具体的实现。
                               //比如AutowiredAnnotationBeanPostProcessor 专门处理 Autowired注解的
                               //比如CommonAnnotationBeanPostProcessor 专门处理Resource注解的
                               //这里小洋看了下Autowired的 因为平时用的多嘛，小洋叮嘱，这里只注入我们加了注解的，像在xml文件配置的bean属性注入，是不在这里处理的
                               pvs = ibp.postProcessPropertyValues((PropertyValues)pvs, filteredPds1, bw.getWrappedInstance(), beanName);
                               if(pvs == null) {
                                   return;
                               }
                           }
                       }
                   }

                   if(needsDepCheck1) {
                       this.checkDependencies(beanName, mbd, filteredPds1, (PropertyValues)pvs);
                   }
               }
               //处理xml文件配置的bean属性注入
               this.applyPropertyValues(beanName, mbd, bw, (PropertyValues)pvs);
           }
       }
   }
```
```Java
//AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues

public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        //找到加了Autowired注解的信息。这个InjectionMetadata对象封装了这个bean里面所有加Autowired注解的元素信息
        InjectionMetadata metadata = this.findAutowiringMetadata(beanName, bean.getClass(), pvs);

        try {
            //然后注入，这个方法会循环每一个InjectedElement，然后执行具体的inject方法
            //循环我们就不看了 我们直接看InjectionMetadata.InjectedElement.inject
            metadata.inject(bean, beanName, pvs);
            return pvs;
        } catch (Throwable var7) {
            throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", var7);
        }
    }
```
```Java
//InjectionMetadata.InjectedElement.inject

protected void inject(Object target, String requestingBeanName, PropertyValues pvs) throws Throwable {
    if(this.isField) {
        //注解加在成员变量上面，反射设置属性值
        Field ex = (Field)this.member;
        ReflectionUtils.makeAccessible(ex);
        ex.set(target, this.getResourceToInject(target, requestingBeanName));
    } else {
        //注解加在方法上面，反射调用方法
        if(this.checkPropertySkipping(pvs)) {
            return;
        }

        try {
            Method ex1 = (Method)this.member;
            ReflectionUtils.makeAccessible(ex1);
            //核心也就这么一句代码
            ex1.invoke(target, new Object[]{this.getResourceToInject(target, requestingBeanName)});
        } catch (InvocationTargetException var5) {
            throw var5.getTargetException();
        }
    }

}

```
&emsp;&emsp;上面我们看了AutowiredAnnotationBeanPostProcessor的处理属性注入，其实其他的BeanPostProcessor处理都一样，都首先找到自己处理的注解的信息，构造成InjectionMetadata对象去处理。不同的地方就是查找注解的地方。注解的属性注入处理完了，对于xml文件配置bean属性的注入，小洋大概看了下，最后也是通过反射执行set方法初始化的。而且只有反射执行方法，所以xml配置的bean属性注入，必须有对应的set方法。不然会报错的。

&emsp;&emsp;小洋的这一番折腾，大概了解了Spring初始化的时候对bean的解析、实例化、属性注入等流程。虽然不是特别的详细，但是具体的流程还是走通的，期间收获不小。
