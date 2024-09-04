---
title: Jvm Classloader
date: 2016-11-16 11:54:41
tags:
  - JVM
  - Classloader
---
Java 类加载机制

### 说在前面的话

JVM的相关知识很多,其实主要我们在平时接触到就三个部分一个是内存模型，一个是类加载，在一个就是垃圾回收，要看具体的内容可以去看《深入理解Java虚拟机》这本书,慢慢品味会有不少的收获，记得自己刚开始面试那会前面几章看了好多遍，奈何当时只是应付面试,里面好多东西都不是很理解,现在在来复习一下说说自己的见解。

谈到Java的类加载机制，首先我们要知道类加载到底加载的是什么，其实好多人看网上的文章说的什么双亲委派模型啊，一些基础的都没有说明白。**Class文件由类装载器装载后，在JVM中将形成一份描述Class结构的元信息对象，通过该元信息对象可以获知Class的结构信息：如构造函数，属性和方法等，Java允许用户借由这个Class相关的元信息对象间接调用Class对象的功能。** 这是比较官方的说法，我的理解是这样的，Java中我们可以对一切相同属性的东西抽象一个父类。比如宝马，奔驰，奥迪，他们都有四个轮子，有发动机等等，我们就可以抽象出一个汽车类。在Java中，进行装载的是一系列class文件，这些class文件都有构造函数，成员属性，成员方法等，所以Java就将这些抽象出来了Class对象。**一个class文件进行加载了之后，只会有一个Class对象，之后再次加载的时候，会去检查是否加载了。**

上面说完了类加载加载的是什么东西，在说说Java类加载的一个流程，**虚拟机把描述类的数据从class文件加载到内存，并对数据进行校验，转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。** 这些过程具体讲起来就很多了，本文就不做过多的详述了，具体大家去看看书本的知识。

### Java类加载的双亲委派模型

>1.  **Bootstrap ClassLoader**。将存放于<JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如 rt.jar 名字不符合的类库即使放在lib目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被Java程序直接引用
2.  **Extension ClassLoader** : 将<JAVA_HOME>\lib\ext目录下的，或者被java.ext.dirs系统变量所指定的路径中的所有类库加载。开发者可以直接使用扩展类加载器。     
3.  ** Application ClassLoader** : 负责加载用户类路径(ClassPath)上所指定的类库,开发者可直接使用。

上面是Java里面的三个类加载器。从上到下分别是由父到子。其中Bootstrap ClassLoader是由C++编写的。我们在Java代码中是看不到的。

如果一个类加载器接收到了类加载的请求，它首先把这个请求委托给他的父类加载器去完成，每个层次的类加载器都是如此，因此所有的加载请求都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它在搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。这就是Java的双亲委派模型。要理解这个模型我们可以去看ClassLoader这个类，其实又回到了刚开始的抽象，Java中三个类加载器可以抽象出来一个对象。我们可以去看这个这对象的源码查看到相关的原理。

```Java
//java.lang.ClassLoader
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        //我看的Java8的代码，这里给改成了同步代码块，以前的版本是同步方法
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            //检查是否被加载了，验证了一个类的Class对象永远只用一个
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //交给父类去加载
                        c = parent.loadClass(name, false);
                    } else {
                        //如果没有父类就交给BootstrapClass去加载
                        //这个方法是Native的方法。
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    //上面父类如果没有找到则交给你自己去加载。
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```
上面的双亲委派模型我们看代码已经很清楚了，再来看看AppClassLoader和ExClassLoader。这两个类在sun.misc.Launcher类中。

```Java
//sun.misc.Launcher类部分代码
public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
            //初始化ExtClassLoader
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
          //初始化AppClassLoader，并且将ExtClassLoader传入，作为父加载器
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }
        ......
    }
    //ExtClassLoader是Launcher的静态内部类
    static class ExtClassLoader extends URLClassLoader {
        public static Launcher.ExtClassLoader getExtClassLoader() throws IOException {
            //这里指定了ExtClassLoader加载的类路径，
            final File[] var0 = getExtDirs();
            try {
                return (Launcher.ExtClassLoader)AccessController.doPrivileged(new PrivilegedExceptionAction() {
                    public Launcher.ExtClassLoader run() throws IOException {
                        int var1 = var0.length;

                        for(int var2 = 0; var2 < var1; ++var2) {
                            MetaIndex.registerDirectory(var0[var2]);
                        }
                        //返回具体的ExtClassLoader对象，看下面构造方法
                        return new Launcher.ExtClassLoader(var0);
                    }
                });
            } catch (PrivilegedActionException var2) {
                throw (IOException)var2.getException();
            }
        }

        private static File[] getExtDirs() {
            //这里是java扩展包路径。
            String var0 = System.getProperty("java.ext.dirs");
            File[] var1;
            if(var0 != null) {
                StringTokenizer var2 = new StringTokenizer(var0, File.pathSeparator);
                int var3 = var2.countTokens();
                var1 = new File[var3];

                for(int var4 = 0; var4 < var3; ++var4) {
                    var1[var4] = new File(var2.nextToken());
                }
            } else {
                var1 = new File[0];
            }

            return var1;
        }

        public ExtClassLoader(File[] var1) throws IOException {
            //这里的第二参数是父加载器，这里是null。在Java里面，ExtClassLoader
            //是没有父加载器。我们在看ClassLoader的源码的时候，当没有父类加载器的时候
            //交给Bootstrap去加载的。所以我们可以认为ExtClassLoader的父类加载器是Bootstrap
            super(getExtURLs(var1), (ClassLoader)null, Launcher.factory);
            SharedSecrets.getJavaNetAccess().getURLClassPath(this).initLookupCache(this);
        }

        .....

      }

      ......

      //AppClassLoader也是Launcher的静态内部类
      static class AppClassLoader extends URLClassLoader {
        final URLClassPath ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);

        public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
            //这里指定了类加载加载class文件目录。这里说明了AppClassLoader是加载classath下面的jar包。
            final String var1 = System.getProperty("java.class.path");
            final File[] var2 = var1 == null?new File[0]:Launcher.getClassPath(var1);
            return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction() {
                public Launcher.AppClassLoader run() {
                    URL[] var1x = var1 == null?new URL[0]:Launcher.pathToURLs(var2);
                    //这里返回具体的AppClassLoader对象，看下面构造方法
                    return new Launcher.AppClassLoader(var1x, var0);
                }
            });
        }

        AppClassLoader(URL[] var1, ClassLoader var2) {
            //这里的第二个参数是指定父类加载器，我们最开始看见的传入的是ExtClassLoader
            super(var1, var2, Launcher.factory);
            this.ucp.initLookupCache(this);
        }
        ....
      }

```

上面我们看到了Java类加载器的初始化，Lanuncher类是rt.jar包中的类，在lib目录下，启动的时候就会加载初始化。由于BootstrapClassLoader是C++编写，我们在Java中是看到不的，由此Java的类加载器初始化完了，然后就可以去加载自己对应目录的下的class文件了。在这里大家可能又一个疑问。Java中的类加载器都是继承ClassLoader的，那ClassLoader是由什么加载的呢。其实ClassLoader对象和Class对象一样，他们的初始化是由Native方法去初始化的，Java层面我们是看不到的。由于我没有看C++代码对此也不敢乱下结论。

```Java
public abstract class ClassLoader {

    private static native void registerNatives();
    static {
        registerNatives();
    }
    .....
}
```
### 为什么要用双亲委派模型

因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次。考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义的类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免这种情况，因为String已经在启动时就被引导类加载器（Bootstrcp ClassLoader）加载，所以用户自定义的ClassLoader永远也无法加载一个自己写的String，除非你改变JDK中ClassLoader搜索类的默认算法。

这里推荐一篇博客写的很好 http://blog.csdn.net/xyang81/article/details/7292380
