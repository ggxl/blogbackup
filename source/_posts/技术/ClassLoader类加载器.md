---
date: 2018-8-2 18:44:49
title: ClassLoader类加载器
no_word_count: false
tags: [技术,后端,java,ClassLoader]
---

### ClassLoader介绍
>- 类加载器是负责加载类的对象。ClassLoader 类是一个抽象类。如果给定类的二进制名称，那么类加载器会试图查找或生成构成类定义的数据。一般策略是将名称转换为某个文件名，然后从文件系统读取该名称的“类文件”。 
>
>- 每个 Class 对象都包含一个对定义它的 ClassLoader 的引用。
>
>- 数组类的 Class 对象不是由类加载器创建的，而是由 Java 运行时根据需要自动创建。数组类的类加载器由 Class.getClassLoader() 返回，<font color="red">该加载器与其元素类型的类加载器是相同的；如果该元素类型是基本类型，则该数组类没有类加载器。</font>
>- ClassLoader 类试用委托机制来加载类，每个ClassLoader实例都有一个父类加载器。
>
>- 当需要查找类或者资源时ClassLoader实例会将任务委托给自己的父类加载器去处理。
>- 虚拟机内置的类加载器（称为 "bootstrap class loader"）本身没有父类加载器，但是可以将它用作 ClassLoader 实例的父类加载器。
>
>- 网络类加载器子类必须定义方法 findClass 和 loadClassData，以实现从网络加载类。下载组成该类的字节后，它应该使用方法 defineClass 来创建类实例。代码如下
<!--more-->
>```java
class NetworkClassLoader extends ClassLoader {
         String host;
         int port;

         public Class findClass(String name) {
             byte[] b = loadClassData(name);
             return defineClass(name, b, 0, b.length);
         }

         private byte[] loadClassData(String name) {
             // load the class data from the connection
              . . .
         }
     }

>```

### ClassLoader.loadClass方法源码
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException{

    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
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

/**
 * Finds the class with the specified <a href="#name">binary name</a>.
 * This method should be overridden by class loader implementations that
 * follow the delegation model for loading classes, and will be invoked by
 * the {@link #loadClass <tt>loadClass</tt>} method after checking the
 * parent class loader for the requested class.  The default implementation
 * throws a <tt>ClassNotFoundException</tt>.
 *
 * @param  name
 *         The <a href="#name">binary name</a> of the class
 *
 * @return  The resulting <tt>Class</tt> object
 *
 * @throws  ClassNotFoundException
 *          If the class could not be found
 *
 * @since  1.2
 */
protected Class<?> findClass(String name) throws ClassNotFoundException {
	throw new ClassNotFoundException(name);
}
```

1. `Class<?> c = findLoadedClass(name);`首先检查类是否已加载
2. `c = parent.loadClass(name, false);`如果没有被加载委托父类加载器加载,父类加载器执行同样的代码委托它的父类加载器加载直到没有父类加载器为止。
3. `c = findBootstrapClassOrNull(name);`调用虚拟机的类加载器
4. `c = findClass(name);`如果经过以上的委托机制还未加载到类调用自己本身的findClass方法加载类。
5. <font color=red>findClass</font>方法是protected的 本身没有任何实现代码，而是直接抛出ClassNotFoundException异常，所以这个方法就是为了让我们自己实现类加载器时重写的，而且必须重写。

### 自定义类加载器
```java
/**
*自定义类加载器
*/
public class MyClassLoader1 extends ClassLoader {

    @Override
    protected Class<?> findClass(java.lang.String name) throws ClassNotFoundException {
        java.lang.String classPath = MyClassLoader1.class.getResource("/").getPath();
        java.lang.String fileName = name.replace(".", "/") + ".class";
        File classFile = new File(classPath, fileName);

        if (!classFile.exists()) {
            throw new ClassNotFoundException(classFile.getPath() + "不存在");
        }
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ByteBuffer bf = ByteBuffer.allocate(1024);
        FileInputStream fis = null;
        FileChannel fc = null;
        try {
            fis = new FileInputStream(classFile);
            fc = fis.getChannel();
            while (fc.read(bf) > 0) {
                bf.flip();
                baos.write(bf.array(), 0, bf.limit());
                bf.clear();
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fis.close();
                fc.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        //解析字节数组，将字节数组转换为Class的实例。
        return defineClass(null, baos.toByteArray(), 0, baos.toByteArray().length);
    }
}

/**
 * 自定义的String类
 */
public class String {

	@Override
	public java.lang.String toString() {
		return "my String...";
	}
}


@Test
public void test1() throws Exception {
    MyClassLoader1 myClassLoader = new MyClassLoader1();
	
    Class<?> clazz = myClassLoader.findClass("com.iguozhe.wonhot.experiment.java.lang.classloader.String");
    Object instance = clazz.newInstance();
    Method method = clazz.getMethod("toString");

    Object invoke = method.invoke(instance);
    System.out.println(invoke);//my String...
}

@Test
public void test2() throws ClassNotFoundException {
    MyClassLoader1 myClassLoader = new MyClassLoader1();
    Class<?> clazz = myClassLoader.findClass("com.iguozhe.wonhot.experiment.java.lang.classloader.String");
    ClassLoader loader = clazz.getClassLoader();
    ClassLoader loader1 =loader.getParent();
    ClassLoader loader2 = loader1.getParent();
    ClassLoader loader3 = loader2.getParent();
    System.out.println(loader);//com.iguozhe.wonhot.experiment.java.lang.classloader.MyClassLoader1@14f9431
    System.out.println(loader1);//sun.misc.Launcher$AppClassLoader@644d46
    System.out.println(loader2);//sun.misc.Launcher$ExtClassLoader@1961c42
    System.out.println(loader3);//null
}
```
>test1测试我们可以通过使用自己的类加载器加载类
>
>test2测试
>根据打印的类加载器。发现一共有3个类加载MyClassLoader1、AppClassLoader、ExtClassLoader、其实还有一个BootstrapClassLoader（在没有父classLoader时使用）当loader2没有父类加载器时就会委托BootstrapClassLoader加载。
>
- 启动类加载器(Bootstrap ClassLoader)：负责加载 JAVA_HOME\lib 目录中的，或通过-Xbootclasspath参数指定路径中的，且被虚拟机认可（按文件名识别，如rt.jar）的类
- 扩展类加载器(Extension ClassLoader)：负责加载 JAVA_HOME\lib\ext 目录中的，或通过java.ext.dirs系统变量指定路径中的类库。
- 应用程序类加载器(Application ClassLoader)：负责加载用户路径（classpath）上的类库。

### 总结
>- 自定义类加载必须实现findClass方法
>- Java装载类使用“全盘负责委托机制”。
>“全盘负责”是指当一个ClassLoder装载一个类时，除非显示的使用另外一个ClassLoder，该类所依赖及引用的类也由这个ClassLoder载入；
>“委托机制”是指先委托父类装载器寻找目标类，只有在找不到的情况下才从自己的类路径中查找并装载目标类。
>- 这一点是从安全方面考虑的，试想如果一个人写了一个恶意的基础类（如java.lang.String）并加载到JVM将会引起严重的后果，但有了全盘负责制，java.lang.String永远是由根装载器来装载，避免以上情况发生 除了JVM默认的三个ClassLoder以外，第三方可以编写自己的类装载器，以实现一些特殊的需求。
>- 类文件被装载解析后，在JVM中都有一个对应的java.lang.Class对象，提供了类结构信息的描述。数组，枚举及基本数据类型，甚至void都拥有对应的Class对象。Class类没有public的构造方法，Class对象是在装载类时由JVM通过调用类装载器中的defineClass()方法自动构造的。








 

