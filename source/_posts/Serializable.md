---
title: Serializable
date: 2016-11-14 11:31:49
tags:
  - 序列化
photos:
  - /img/20161115.jpg
---
Java序列化
<!--more-->

&emsp;&emsp;最近在Rpc的一些内容，主要的序列化和传输两个模块挺重要的，然后详细的去了解了一些序列化的知识，也当复习下。

#### Java序列化
&emsp;&emsp;Java的序列化很简单，只需要实现Serializable接口，然后就可以用ObjectInpuStream、ObjectOutputSteam进行序列化和反序列化了。下面简单演示一下。
```Java
public static void main(String[] args) {
        User user = new User();
        user.setUserName("LTY");
        user.setPassword("password");
        try(ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("tempFile"))){
            outputStream.writeObject(user);
            System.out.println("Write User "+user.toString());
        }catch (Exception e){
            e.printStackTrace();
        }
        try(ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("tempFile"))){
            User read = (User) inputStream.readObject();
            System.out.println("Read User "+read.toString());
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    static class User implements Serializable{
        private String userName;
        private String password;

        public String getUserName() {
            return userName;
        }

        public void setUserName(String userName) {
            this.userName = userName;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }

        @Override
        public String toString() {
            return "User{" +
                    "userName='" + userName + '\'' +
                    ", password='" + password + '\'' +
                    '}';
        }
    }
    //output:
    //Write User User{userName='LTY', password='password'}
    //Read User User{userName='LTY', password='password'}
```
&emsp;&emsp;上面就是简单的，序列化和反序列化的过程。默认情况下Java是序列化所有成员变量的，除了静态变量。很好理解，静态变量并不属于某一个具体的对象。在有就是Transient关键字可以控制成员变量的序列化。
```Java
       private String userName;
       private transient String password;

       //output
       //Write User User{userName='LTY', password='password'}
       //Read User User{userName='LTY', password='null'}
```
&emsp;&emsp;刚才的代码，我在password变量前面加上transient关键字，可以看到反序列化出来的结果为空。
&emsp;&emsp;Java序列化是将对象转换成的二进制写入文件，并且是可逆的。如果我们在进行Rpc调用的时候，会将这样的二进制格式的数据进行传输。这样是不安全的。对此我们可以自定义序列化和反序列化的一些策略。对一些敏感数据（比如password）进行模糊化。示例如下:
```Java
        private void writeObject(ObjectOutputStream stream) throws IOException {
            password = password +"decode";  //模拟加密,模糊数据
            stream.defaultWriteObject();
            stream.writeObject(password);
        }
        private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException {
            stream.defaultReadObject();
            password = (String) stream.readObject();
            password = password.substring(0,password.lastIndexOf("decode")); //模拟解密,还原数据
        }

        //output
        //Write User User{userName='LTY', password='passworddecode'}
        //Read User User{userName='LTY', password='password'}
```
&emsp;&emsp;上面代码我们在User类中加入了两个方法。这两个方法会在写和读的时候调用，我们可以用这样的方式来自定义序列化和反序列号策略。代码中我们看出，我们在password变量前面加了transient关键字,但是结果仍然反序列化出来了结果。就是因为我自定义了序列化。有兴趣的人可以去看看ArrayList的源码,其中的自定义序列化策略代码很有参考性。所以知识不是死板的我们要知道其中缘由。上面就是我们知道的一些Java序列化的一些用法。知道了怎么用我们在来看看一些实现。为什么我只需要实现了Serializable这个接口就可以序列化以及上面的writeObject和readObject方法是什么时候执行的，代码中我们并没有显示的去调用。那就的看源码了。

```Java
//ObjectOutputStream
public final void writeObject(Object obj) throws IOException {
        if (enableOverride) {
            writeObjectOverride(obj);
            return;
        }
        try {
            //写对象，我们进去看看
            writeObject0(obj, false);
        } catch (IOException ex) {
            if (depth == 0) {
                writeFatalException(ex);
            }
            throw ex;
        }
    }

```
```Java
//writeObject0方法部分代码
          // remaining cases
           if (obj instanceof String) {
               writeString((String) obj, unshared);
           } else if (cl.isArray()) {
               writeArray(obj, desc, unshared);
           } else if (obj instanceof Enum) {
               writeEnum((Enum<?>) obj, desc, unshared);
           } else if (obj instanceof Serializable) {
               //判断了对象的类型，(Enum、Array和Serializable)如果没有实现Serializable,则会抛出异常。
               //进入具体的写方法
               writeOrdinaryObject(obj, desc, unshared);
           } else {
               if (extendedDebugInfo) {
                   throw new NotSerializableException(
                       cl.getName() + "\n" + debugInfoStack.toString());
               } else {
                   throw new NotSerializableException(cl.getName());
               }
           }
```

```Java
private void writeOrdinaryObject(Object obj,
                                     ObjectStreamClass desc,
                                     boolean unshared)
        throws IOException
    {
        if (extendedDebugInfo) {
            debugInfoStack.push(
                (depth == 1 ? "root " : "") + "object (class \"" +
                obj.getClass().getName() + "\", " + obj.toString() + ")");
        }
        try {
            desc.checkSerialize();

            bout.writeByte(TC_OBJECT);
            writeClassDesc(desc, false);
            handles.assign(unshared ? null : obj);
            //判断类型。Externalizable 后面我会讲到。
            if (desc.isExternalizable() && !desc.isProxy()) {
                writeExternalData((Externalizable) obj);
            } else {
                //序列化数据
                writeSerialData(obj, desc);
            }
        } finally {
            if (extendedDebugInfo) {
                debugInfoStack.pop();
            }
        }
    }
```

```Java
//writeSerialData方法中调用了invoke
/** class-defined writeObject method, or null if none */
private Method writeObjectMethod;
/** class-defined readObject method, or null if none */
private Method readObjectMethod;

void invokeWriteObject(Object obj, ObjectOutputStream out)
        throws IOException, UnsupportedOperationException
    {
        //这里的writeObjectMethod变量就是反射需要调用的方法
        if (writeObjectMethod != null) {
            try {
                writeObjectMethod.invoke(obj, new Object[]{ out });
            } catch (InvocationTargetException ex) {
                Throwable th = ex.getTargetException();
                if (th instanceof IOException) {
                    throw (IOException) th;
                } else {
                    throwMiscException(th);
                }
            } catch (IllegalAccessException ex) {
                // should not occur, as access checks have been suppressed
                throw new InternalError(ex);
            }
        } else {
            throw new UnsupportedOperationException();
        }
    }

```
&emsp;&emsp;上面我们走了一遍写数据的流程。读数据的流程是一样的这就不说了。在写数据的时候通过instanceof来限制是否实现了Serializable接口，然后通过反射的方式调用了writeObject和readObject方法。
&emsp;&emsp;看源码的时候我们发现了一个Externalizable，其实我们实现序列化不止一种方法。Serializable接口只是最方便的，不需要我们写过多的代码。下面我们看看怎么用Externalizable实现序列化。

```Java
static class User implements Externalizable{
        private String userName;
        private String password;

        public User(){}

        public String getUserName() {
            return userName;
        }

        public void setUserName(String userName) {
            this.userName = userName;
        }

        public String getPassword() {
            return password;
        }

        public void setPassword(String password) {
            this.password = password;
        }

        @Override
        public String toString() {
            return "User{" +
                    "userName='" + userName + '\'' +
                    ", password='" + password + '\'' +
                    '}';
        }

        @Override
        public void writeExternal(ObjectOutput out) throws IOException {
            out.writeObject(userName);
            out.writeObject(password);
        }

        @Override
        public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
            userName = (String) in.readObject();
            password = (String) in.readObject();
        }
    }
    //output:
    //Write User User{userName='LTY', password='password'}
    //Read User User{userName='LTY', password='password'}
```
&emsp;&emsp;我们通过实现Externalizable接口，并且必须复写writeExternal和readExter方法来实现序列化和反序列化。上面我们看源码的时候发现对Serializable和Externalizable做了判断。做不同处理。使用Externalizable意味着我们不能自动的序列化变量，需要将自己需要序列化的变量一个一个的写入到流当中,反序列的化的时候需要一个一个的读取,给成员变量进行初始化。还有使用Externalizable接口必须有有个无惨构造函数。

&emsp;&emsp;序列化的一些知识我们回顾的差不多了。然后我们谈谈Rpc中的序列化,在Rpc中我们的接口调用很频繁,要求我们在序列化方面需要很好的性能,如果使用Java原生的这种序列化方式那会很糟糕的,现在一些主流的Rpc框架早已经抛弃了。本人所在的公司用的自己研发的一个Rpc框架,他们没有使用二进制之类的序列化方法,为了从业务的考虑直接用的Json的序列化方法,这样我们可以方便开发人员的调试。后端技术已经很成熟了,比较好的一些序列化技术有Kryo,FST等针对Java语言的,还有一些跨语言的。比如Protostuff,ProtoBuf,Thrift等.当我们累个半死知道了Java的序列化之后,发现主流的一些技术已经把它抛弃了有时候我也不得不感觉郁闷啊。不过学无止境.

---
<p><font color='blue'>人生犹如一本书，愚蠢者草草翻过，聪明人细细阅读。为何如此 . 因为他们只能读它一次。</font></p><p align='right'>——保罗</p>

---
