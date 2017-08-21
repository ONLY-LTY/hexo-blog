---
title: Reflect-Proxy
date: 2016-11-14 18:22:59
tags:
  - 反射
  - 动态代理
photos:
  - /img/20161114reflect.jpg
---
反射和动态代理
<!--more-->

>反射在java中是一个高级特性。它的用处很广泛，通过反射API可以获取程序在运行时刻的内部结构。本文不对反射基本API做详细的讲解。主要讲解反射在动态代理中应用。

#### 动态代理和静态代理
 > **代理** : 为某个对象提供一个代理，以控制对这个对象的访问。 代理类和委托类有共同的父类或父接口，这样在任何使用委托类对象的地方都可以用代理对象替代。代理类负责请求的预处理、过滤、将请求分派给委托类处理、以及委托类执行完请求后的后续处理。

   1.**静态代理** :由程序员创建或工具生成代理类的源码，再编译代理类。所谓静态也就是在程序运行前就已经存在代理类的字节码文件，代理类和委托类的关系在运行前就确定了。
```java
/**  
 * 代理接口。处理给定名字的任务。
 */  
public interface Subject {  
  /**
   * 执行给定名字的任务。
    * @param taskName 任务名
   */  
   public void dealTask(String taskName);   
}
```
```java
/**
 * 真正执行任务的类，实现了代理接口。
 */  
public class RealSubject implements Subject {  

 /**
  * 执行给定名字的任务。
  * @param taskName  
  */  
   @Override  
   public void dealTask(String taskName) {  
      System.out.println("正在执行任务："+taskName);  
      doSomething...
   }  
}
```
```java
/**
 *　代理类，实现了代理接口。
 */  
public class ProxySubject implements Subject {  
 //代理类持有一个委托类的对象引用  
 private Subject delegate;  

 public ProxySubject(Subject delegate) {  
  this.delegate = delegate;  
 }  

 /**
  * 将请求分派给委托类执行，记录任务执行前后的时间，时间差即为任务的处理时间
  *  
  * @param taskName
  */  
 @Override  
 public void dealTask(String taskName) {  
  long stime = System.currentTimeMillis();   
  //将请求分派给委托类处理  
  delegate.dealTask(taskName);  
  long ftime = System.currentTimeMillis();   
  System.out.println("执行任务耗时"+(ftime - stime)+"毫秒");  

 }  
}  
```
上面代码就是一个简单的静态代理模式，我们获取到代理类ProxySubject可以操作相关接口。然后由具体的实现类执行。进而在真正执行方法前后做一些处理。很明显在代码运行前，相关类就已经确定。而我们针对每个接口类都需要实现代理类，增加新的接口方法，相关代理类也需要修改。代码不易维护。

  2.**动态代理** : 顾名思义动态代理类的源码是在程序运行期间由JVM根据反射等机制动态的生成，所以不存在代理类的字节码文件。代理类和委托类的关系是在程序运行时确定。
```java
/**  
 * 代理接口。处理给定名字的任务。 (和静态代理一样没有什么变化)
 */  
public interface Subject {  
  /**
   * 执行给定名字的任务。
    * @param taskName 任务名
   */  
   public void dealTask(String taskName);   
}
```
```java
/**
 * 真正执行任务的类，实现了代理接口。 (和静态代理没有什么变化)
 */  
public class RealSubject implements Subject {  

 /**
  * 执行给定名字的任务。
  * @param taskName  
  */  
   @Override  
   public void dealTask(String taskName) {  
      System.out.println("正在执行任务："+taskName);  
      doSomething...
   }  
}
```
```java
/**
   * InvocationHandlerImple实现InvocationHandler接口，覆写invoke()方法
   * 代理主题的业务写在invoke()方法中
   */
 public class InvocationHandlerImpl implements InvocationHandler {

     //需要代理的对象
     private Object target;

     public InvocationHandlerImpl(Object target) {
         this.target = target;
     }

     @Override
     public Object invoke(Object proxy, Method method, Object[] args)
             throws Throwable {
         long stime = System.currentTimeMillis();   
         Object obj = method.invoke(target, args);
         long ftime = System.currentTimeMillis();   
         System.out.println("执行任务耗时"+(ftime - stime)+"毫秒");
         return obj;
     }
 }
```
```java
/**
 * 使用方法
 */  
public class DynProxyFactory {    
 public static void  main(String[] args){  
   //委托类，这里可以是很多个接口。
   Subject delegate = new RealSubject();  
   InvocationHandler handler = InvocationHandlerImpl(delegate);  
   Subject proxy = (Subject)Proxy.newProxyInstance(delegate.getClass().getClassLoader(),   delegate.getClass().getInterfaces(),   handler);  
   proxy.dealTask("Proxy Task");
}
```
由代码可以看出来，动态代理用到了InvocationHandler类和Proxy类。其基本模式就是将自己的方法功能实现交给InvocationHandler角色，外界Proxy角色中的每个方法的调用，Proxy角色都会交给InvocationHandler来处理，而InvocationHandler则调用具体的对象角色的方法。这样我们的代理类动态生成的，不需要像静态代理那样针对每个接口实现自己的代理类。那么Proxy是如何生成的代理类呢？我接下来看源码分析（很多，很枯燥）。

#### JDK动态代理实现
```java
@CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         * 具体的获取代理类Class对象调用方法，传入classloder和接口类也就是上面的Subject
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }
             //反射获取构造方法
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            //一句没用的代码，ih都没有用
            final InvocationHandler ih = h;
            //反射设置访问权限
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            //通过反射调用构造方法直接返回实例对象，h是我们的传入的InvocationHandler实例。
            //这里是动态生成是实现Subject接口的代理类，后面我会给大家看看这个类里面的东东
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```
```java
 /**
     * Generate a proxy class.  Must call the checkProxyAccess method
     * to perform permission checks before calling this.
     * 动态生成代理类Class对象调用方法
     */
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        //接口类数组的长度小于65535，65535是计算机16位二进制最大数，如果大于就会内存溢出.
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        //调用get方法,这里proxyClassCache对象是一个WeakCache（弱引用，用作缓存）对象。具体的get看下面。
        return proxyClassCache.get(loader, interfaces);
    }
```
```java
public V get(K key, P parameter) {
        //这里 key是classLoader P 是interfaces
        //判断是否为null
        Objects.requireNonNull(parameter);

        expungeStaleEntries();
        //将key转换成cacheKey
        Object cacheKey = CacheKey.valueOf(key, refQueue);

        // lazily install the 2nd level valuesMap for the particular cacheKey
        //第一次进来才初始化valuseMap保存代理类和代理的接口
        ConcurrentMap<Object, Supplier<V>> valuesMap = map.get(cacheKey);
        if (valuesMap == null) {
            ConcurrentMap<Object, Supplier<V>> oldValuesMap
                = map.putIfAbsent(cacheKey,
                                  valuesMap = new ConcurrentHashMap<>());
            if (oldValuesMap != null) {
                valuesMap = oldValuesMap;
            }
        }

        // create subKey and retrieve the possible Supplier<V> stored by that
        // subKey from valuesMap
        //这一步获取代理类,调用BiFunction.apply方法,具体看下面
        Object subKey = Objects.requireNonNull(subKeyFactory.apply(key, parameter));
        //从valuesMap中获取代理类，第一次或者新的classloder肯定为空
        Supplier<V> supplier = valuesMap.get(subKey);
        Factory factory = null;

        while (true) {
            //不为空的话直接取出来返回
            if (supplier != null) {
                // supplier might be a Factory or a CacheValue<V> instance
                V value = supplier.get();
                if (value != null) {
                    return value;
                }
            }
            // else no supplier in cache
            // or a supplier that returned null (could be a cleared CacheValue
            // or a Factory that wasn't successful in installing the CacheValue)

            // lazily construct a Factory
            if (factory == null) {
                factory = new Factory(key, parameter, subKey, valuesMap);
            }
            //为空的时候将上面生成的代理类放到valuesMap中
            if (supplier == null) {
                supplier = valuesMap.putIfAbsent(subKey, factory);
                if (supplier == null) {
                    // successfully installed Factory
                    supplier = factory;
                }
                // else retry with winning supplier
            } else {
                if (valuesMap.replace(subKey, supplier, factory)) {
                    // successfully replaced
                    // cleared CacheEntry / unsuccessful Factory
                    // with our Factory
                    supplier = factory;
                } else {
                    // retry with current supplier
                    supplier = valuesMap.get(subKey);
                }
            }
        }
    }

```
```java
 @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
            //IdentityHashMap是一个key可以重复的的Map，这里jdk当做set来用
            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    //反射获得接口类的实例
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 * 验证是否为接口 jdk动态代理只能代理接口
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                 //将interfaceClass放入到interfaceSet，这里interfaceSet是IdentityHashMap
                 //当我们put的时候发现已经有key了会添加后返回这个以前的key对应的value值，
                 //这里来判读interface是否重复。
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }

            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             * 若为非公有接口则需要记录包名且判断必须为同一包
             */
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                //无非公有接口则默认为com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             * 选择一个代理类名称 第一为$Proxy0 以后类推。num生成是线程安全的
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             * 生成class二进制文件，具体看下面
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
               //这个方法是native方法，加载class二进制
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }

```
```java
public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2) {
        ProxyGenerator var3 = new ProxyGenerator(var0, var1, var2);
        //具体调用ProxyGenerator.generateClassFile方法生成class二进制文件，具体看下面
        final byte[] var4 = var3.generateClassFile();
        //如果saveGeneratedFiles为true，将会把class文件保存到项目com/sun/proxy下,即就是上面的默认包名目录
        if(saveGeneratedFiles) {
            AccessController.doPrivileged(new PrivilegedAction() {
                public Void run() {
                    try {
                        int var1 = var0.lastIndexOf(46);
                        Path var2;
                        if(var1 > 0) {
                            Path var3 = Paths.get(var0.substring(0, var1).replace('.', File.separatorChar), new String[0]);
                            Files.createDirectories(var3, new FileAttribute[0]);
                            var2 = var3.resolve(var0.substring(var1 + 1, var0.length()) + ".class");
                        } else {
                            var2 = Paths.get(var0 + ".class", new String[0]);
                        }

                        Files.write(var2, var4, new OpenOption[0]);
                        return null;
                    } catch (IOException var4x) {
                        throw new InternalError("I/O exception saving generated file: " + var4x);
                    }
                }
            });
        }

        return var4;
    }

```
```java
private byte[] generateClassFile() {

        //添加Object三个方法
        this.addProxyMethod(hashCodeMethod, Object.class);
        this.addProxyMethod(equalsMethod, Object.class);
        this.addProxyMethod(toStringMethod, Object.class);
        Class[] var1 = this.interfaces;
        int var2 = var1.length;

        int var3;
        Class var4;
        //添加所有所有接口中的方法,idea源码直接把变量替换了
        for(var3 = 0; var3 < var2; ++var3) {
            var4 = var1[var3];
            Method[] var5 = var4.getMethods();
            int var6 = var5.length;

            for(int var7 = 0; var7 < var6; ++var7) {
                Method var8 = var5[var7];
                this.addProxyMethod(var8, var4);
            }
        }

        Iterator var11 = this.proxyMethods.values().iterator();

        List var12;
        //处理重载的方法
        while(var11.hasNext()) {
            var12 = (List)var11.next();
            checkReturnTypes(var12);
        }

        Iterator var15;
        //组装 FieldInfo 和 MethodInfo
        try {
            this.methods.add(this.generateConstructor());
            var11 = this.proxyMethods.values().iterator();

            while(var11.hasNext()) {
                var12 = (List)var11.next();
                var15 = var12.iterator();

                while(var15.hasNext()) {
                    ProxyGenerator.ProxyMethod var16 = (ProxyGenerator.ProxyMethod)var15.next();
                    this.fields.add(new ProxyGenerator.FieldInfo(var16.methodFieldName, "Ljava/lang/reflect/Method;", 10));
                    this.methods.add(var16.generateMethod());
                }
            }

            this.methods.add(this.generateStaticInitializer());
        } catch (IOException var10) {
            throw new InternalError("unexpected I/O Exception", var10);
        }
        //开始写数据了 能力有限看不懂了
        if(this.methods.size() > '\uffff') {
            throw new IllegalArgumentException("method limit exceeded");
        } else if(this.fields.size() > '\uffff') {
            throw new IllegalArgumentException("field limit exceeded");
        } else {
            this.cp.getClass(dotToSlash(this.className));
            this.cp.getClass("java/lang/reflect/Proxy");
            var1 = this.interfaces;
            var2 = var1.length;

            for(var3 = 0; var3 < var2; ++var3) {
                var4 = var1[var3];
                this.cp.getClass(dotToSlash(var4.getName()));
            }

            this.cp.setReadOnly();
            ByteArrayOutputStream var13 = new ByteArrayOutputStream();
            DataOutputStream var14 = new DataOutputStream(var13);

            try {
                var14.writeInt(-889275714);
                var14.writeShort(0);
                var14.writeShort(49);
                this.cp.write(var14);
                var14.writeShort(this.accessFlags);
                var14.writeShort(this.cp.getClass(dotToSlash(this.className)));
                var14.writeShort(this.cp.getClass("java/lang/reflect/Proxy"));
                var14.writeShort(this.interfaces.length);
                Class[] var17 = this.interfaces;
                int var18 = var17.length;

                for(int var19 = 0; var19 < var18; ++var19) {
                    Class var22 = var17[var19];
                    var14.writeShort(this.cp.getClass(dotToSlash(var22.getName())));
                }

                var14.writeShort(this.fields.size());
                var15 = this.fields.iterator();

                while(var15.hasNext()) {
                    ProxyGenerator.FieldInfo var20 = (ProxyGenerator.FieldInfo)var15.next();
                    var20.write(var14);
                }

                var14.writeShort(this.methods.size());
                var15 = this.methods.iterator();

                while(var15.hasNext()) {
                    ProxyGenerator.MethodInfo var21 = (ProxyGenerator.MethodInfo)var15.next();
                    var21.write(var14);
                }

                var14.writeShort(0);
                return var13.toByteArray();
            } catch (IOException var9) {
                throw new InternalError("unexpected I/O Exception", var9);
            }
        }
    }

```
#### 如何查看生成的代理类
>我们在上面的代码中可以知道当ProxyGenerator类中saveGeneratedFiles属性值为true时候会把生成的class保存在项目根目录com/sun/proxy目录下。但是一般saveGeneratedFiles的值是false。我们可以在方法调用前修改这个值为true。在项目根目录下新建com/sun/proxy这个目录。运行就可以看见生成的class文件了。拿如何设置这个值呢 ？？请看下面

```java
    //ProxyGenerator类中的saveGeneratedFiles属性值，是通过GetBooleanAction这个类获取的
    private static final boolean saveGeneratedFiles = ((Boolean)AccessController.doPrivileged(new GetBooleanAction("sun.misc.ProxyGenerator.saveGeneratedFiles"))).booleanValue();

```
```java

package sun.security.action;

import java.security.PrivilegedAction;

public class GetBooleanAction implements PrivilegedAction<Boolean> {
    private String theProp;

    public GetBooleanAction(String var1) {
        this.theProp = var1;
    }

    public Boolean run() {
        //这里通过getBoolean获取“sun.misc.ProxyGenerator.saveGeneratedFiles”这个值
        return Boolean.valueOf(Boolean.getBoolean(this.theProp));
    }
}
```
```java
  public static boolean getBoolean(String name) {
        boolean result = false;
        try {
            //这里我们知道直接从System里面获取的值。
            result = parseBoolean(System.getProperty(name));
        } catch (IllegalArgumentException | NullPointerException e) {
        }
        return result;
    }
```
到这里我们知道了saveGeneratedFiles这个属性值是通过System.getProptery("sun.misc.ProxyGenerator.saveGeneratedFiles")这样初始化的。我们可以在调用Proxy的方法前设置一下就好了。在我们使用代理类的时候可以这样：
```java
/**
 * 使用方法
 */  
public class DynProxyFactory {    
 public static void  main(String[] args){  
 //设置系统属性，为了能看见动态生成的class文件  
  System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
   //委托类，这里可以是很多个接口。
   Subject delegate = new RealSubject();  
   InvocationHandler handler = InvocationHandlerImpl(delegate);  
   Subject proxy = (Subject)Proxy.newProxyInstance(delegate.getClass().getClassLoader(),   delegate.getClass().getInterfaces(),   handler);  
   proxy.dealTask("Proxy Task");
}
```
生成的class文件如下
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.sun.proxy;

import com.lty.rpc.api.Subject;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements Subject {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void dealTask(String var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("com.lty.rpc.api.Subject").getMethod("dealTask", new Class[]{Class.forName("java.lang.String")});
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}

```
>从中我们可以看出动态生成的代理类是以$Proxy为类名前缀，继承自Proxy，并且实现了Proxy.newProxyInstance(…)第二个参数传入的所有接口的类。如果代理类实现的接口中存在非 public 接口，则其包名为该接口的包名，否则为com.sun.proxy。其中的dealTeask()函数都是直接交给h去处理，h在父类Proxy中定义为protected InvocationHandler h;即为Proxy.newProxyInstance(…)第三个参数。所以InvocationHandler的子类 C 连接代理类 A 和委托类 B，它是代理类 A 的委托类，委托类 B 的代理类。

#### 小结一下
>本想仔细分析一下 generateClassFile 的实现，但直接生成 class 文件这种方式确实较为复杂，我就简单说一下其原理吧。有一种比较土的生成 class 文件的方法就是直接使用源码的字符串，用 JavaCompiler 动态编译生成。这样就可以通过所需实现的接口运用反射获得其方法信息，方法实现则直接调用 InvocationHandler 实现类的 invoke 方法，拼出整个代理类的源码，编译生成。而直接组装 class 文件的这种方式更为直接，根据 JVM 规范所定义的 class 文件的格式，省去了组装源码的步骤，使用输出流直接按格式生成 class 文件，相当于自己实现了 JVM 的功能。这样生成 class 后通过 classloader 加载类，反射调用构造方法进行实例化，就可以得到代理类对象了。上面的代码（还有一部分没贴）就是组装 class 文件的过程，奈何我对其也不是很懂

---
<p align='center'><font color='blue'>先相信自己，然后别人才会相信你。</font></p><p align='right'>——罗曼·罗兰</p>

---
