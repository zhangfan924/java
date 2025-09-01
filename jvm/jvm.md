<h1 align="center">JVM虚拟机</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Markdown-模板-blue.svg" />
  <img src="https://img.shields.io/badge/Version-1.0-green.svg" />
</p>

# 📚JAVA类加载机制

## 📘 快速梳理JAVA类加载机制

 三句话总结JDK8的类加载机制：

1. 类缓存：每个类加载器对他加载过的类都有一个缓存。
2. 双亲委派：向上委托查找，向下委托加载。
3. 沙箱保护机制：不允许应用程序加载JDK内部的系统类。

### 1、JDK8的类加载体系

先来一个简单的Demo，看下JDK8的类加载体系：

```java
public class LoaderDemo {
    public static String a ="aaa";
    public static void main(String[] args) throws ClassNotFoundException {
        // 父子关系 AppClassLoader <- ExtClassLoader <- BootStrap Classloader
        ClassLoader cl1 = LoaderDemo.class.getClassLoader();
        System.out.println("cl1 > " + cl1);
        System.out.println("parent of cl1 > " + cl1.getParent());
        // BootStrap Classloader由C++开发，是JVM虚拟机的一部分，本身不是JAVA类。
        System.out.println("grant parent of cl1 > " + cl1.getParent().getParent());
        // String,Int等基础类由BootStrap Classloader加载。
        ClassLoader cl2 = String.class.getClassLoader();
        System.out.println("cl2 > " + cl2);
        System.out.println(cl1.loadClass("java.util.List").getClass().getClassLoader());

        // java指令可以通过增加-verbose:class -verbose:gc 参数在启动时打印出类加载情况
       // 这些参数来自于 sun.misc.Launcher 源码
        // BootStrap Classloader，加载java基础类。
        System.out.println("BootStrap ClassLoader加载目录：" + System.getProperty("sun.boot.class.path"));
        // Extention Classloader 加载一些扩展类。 可通过-D java.ext.dirs另行指定目录
        System.out.println("Extention ClassLoader加载目录：" + System.getProperty("java.ext.dirs"));
        // AppClassLoader 加载CLASSPATH，应用下的Jar包。可通过-D java.class.path另行指定目录
        System.out.println("AppClassLoader加载目录：" + System.getProperty("java.class.path"));
    }
}
```

 可以看到J![java类加载机制](img\java类加载机制.png)

 左侧是JDK中实现的类加载器，通过parent属性形成父子关系。应用中自定义的类加载器的parent都是AppClassLoader

 右侧是JDK中的类加载器实现类。通过类继承的机制形成体系。未来我们就可以通过继承相关的类实现自定义类加载器。

> 简而言之，左侧是对象，右侧是类。

 JDK8中的类加载器都继承于一个统一的抽象类ClassLoader，类加载的核心也在这个父类中。其中，加载类的核心方法如下：

```java
//类加载器的核心方法
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 每个类加载起对他加载过的类都有一个缓存，先去缓存中查看有没有加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {】
               //没有加载过，就走双亲委派，找父类加载器进行加载。
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                }

                if (c == null) {
                    long t1 = System.nanoTime();
                   // 父类加载起没有加载过，就自行解析class文件加载。
                    c = findClass(name);
                  
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
           //这一段就是加载过程中的链接Linking部分，分为验证、准备，解析三个部分。
           // 运行时加载类，默认是无法进行链接步骤的。
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

 这个方法就是最为核心的双亲委派机制。并且这个方法是protected声明的，这意味着，这个方法是可以被子类覆盖的。所以，双亲委派机制也是可以被打破的。

 当一个类加载器要加载一个类时，整体的过程就是通过双亲委派机制向上委托查找，如果没有查找到，就向下委托加载。整个过程整理如下图：

![双亲委派](img\双亲委派.png)

### 2、沙箱保护机制

 双亲委派机制有一个最大的作用就是要保护JDK内部的核心类不会被应用覆盖。而为了保护JDK内部的核心类，JAVA在双亲委派的基础上，还加了一层保险。就是ClassLoader中的下面这个方法。

```java
private ProtectionDomain preDefineClass(String name,
                                            ProtectionDomain pd)
    {
        if (!checkName(name))
            throw new NoClassDefFoundError("IllegalName: " + name);
        // 不允许加载核心类
        if ((name != null) && name.startsWith("java.")) {
            throw new SecurityException
                ("Prohibited package name: " +
                 name.substring(0, name.lastIndexOf('.')));
        }
        if (pd == null) {
            pd = defaultDomain;
        }
        if (name != null) checkCerts(name, pd.getCodeSource());
        return pd;
    }
```

 这个方法会用在JAVA在内部定义一个类之前。这种简单粗暴的处理方式，当然是有很多时代的因素。也因此在JDK中，你可以看到很多javax开头的包。这个奇怪的包名也是跟这个沙箱保护机制有关系的。

### 3、Linking链接过程

 在ClassLoader的loadClass方法中，还有一个不起眼的步骤，resolveClass。这是一个native方法。而其实现的过程称为linking-链接。链接过程的实现功能如下图：![Linking链接过程](img\Linking链接过程.png)

 其中关于半初始化状态就是**JDK在处理一个类的static静态属性时，会先给这个属性分配一个默认值，作用是占住内存**。然后等连接过程完成后，在后面的初**始化阶段，再将静态属性从默认值修改为指定的初始值**。

> 这里注意，static静态的属性，是属于类的，他是在类初始化过程中维护的。而普通的属性是属于对象的，他是在创建对象的过程中维护的。这两个不要搞混了。
>
> 对应到class文件当中，一个是<init>方法，一个是<cinit>方法。

 例如参照一下下面这个案例：

```java
class Apple{
    static Apple apple = new Apple(10);
    static double price = 20.00;
    double totalpay;

    public Apple (double discount) {
        System.out.println("===="+price);
        totalpay = price - discount;
    }
}
public class PriceTest01 {
    public static void main(String[] args) {
        System.out.println(Apple.apple.totalpay);
    }
}
```

 程序打印出的结果是-10 ，而不是10。 这感觉有点反直觉，为什么呢？就是因为这个半初始化状态。

1. **类加载时**，`Apple` 的静态变量按代码顺序依次初始化。
   - 先执行 `static Apple apple = new Apple(10);`
     - 此时 `price` 还没有初始化（还停留在默认值 **0.0**）。
     - 所以在构造函数里 `System.out.println("====" + price);` 打印的是 `====0.0`。
     - 然后 `totalpay = price - discount = 0.0 - 10 = -10.0`。
     - `Apple.apple` 被赋值成这个对象，`totalpay = -10.0`。
2. 接着再执行 `static double price = 20.00;`
   - 到这里 `price` 才真正变成 `20.0`，但 `apple` 已经初始化过了，不会再调用构造函数。
3. 在 `main` 里打印 `Apple.apple.totalpay`，结果就是 `-10.0`。

---

 后面解析的过程有两个核心的概念：符号引用和直接引用。这两个概念了解即可。

 如果A类中有一个静态属性，引用了另一个B类。那么在对类进行初始化的过程中，因为A和B这两个类都没有初始化，JVM并不知道A和B这两个类的具体地址。所以这时，在A类中，只能创建一个不知道具体地址的引用，指向B类。这个引用就称为**符号引用**。而当A类和B类都完成初始化后，JVM自然就需要将这个符号引用转而指向B类具体的内存地址，这个引用就称为**直接引用**。

> 思考问题：为什么在ClassLoader的这个loadClass方法中，reslove参数只能传个false，而不让传true？

**不能在递归调用父加载器时传 `true`**，因为：

1. 解析动作必须在类被“最终加载”后统一执行，而不是在中间委派过程中。
2. 传 `true` 可能导致循环依赖 / 死锁。
3. 符合双亲委派的设计原则：**先确定类归属，再决定是否解析**。

---

## ✨ 一个用类加载机制加薪的故事

### 📖 故事背景

模拟一个 **OA 系统**，每个月需要定时计算大家的工资。

```java
public class OADemo1 {
    public static void main(String[] args) throws InterruptedException {
        Double salary = 15000.00;
        Double money = 0.00;
        // 模拟不停机状态
        while (true) {
            try {
                money = calSalary(salary);
                System.out.println("实际到手Money:" + money);
            } catch(Exception e) {
                System.out.println("加载出现异常 ：" + e.getMessage());
            }
            Thread.sleep(5000);
        }
    }

    private static Double calSalary(Double salary) {
        SalaryCaler caler = new SalaryCaler();
        return caler.cal(salary);
    }
}
```

而具体计算工资的方法，根据面向对象的设计思想，会交由一个单独的 `SalaryCaler` 类来处理：

```java
public class SalaryCaler {
    public Double cal(Double salary) {
        return salary;
    }
}
```

😏 这时，程序员老王想要给大家偷偷加点工资，于是修改了计算方法：

```java
public class SalaryCaler {
    public Double cal(Double salary) {
        return salary * 1.4;
    }
}
```

老王暗自窃喜，但经理肯定不同意。
 👉 程序员与资本家的 **斗智斗勇** 就这样拉开了序幕。

------

## 📦 通过类加载器引入外部 Jar 包

### 🎯 背景

计算工资的方法都在 OA 系统里，经理随时能在仓库中看到。
 于是老王想到：
 👉 把工资计算逻辑抽出来，放到 **外部 Jar 包** 中，再通过类加载器加载。

### 🔧 基于 URLClassLoader

```java
public class OADemo2 {
    public static void main(String[] args) throws Exception {
        Double salary = 15000.00;
        Double money = 0.00;

        URL jarPath = new URL("file:/Users/roykingw/DevCode/ClassLoadDemo/out/artifacts/SalaryCaler_jar/SalaryCaler.jar");
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[] {jarPath});

        // 模拟不停机状态
        while (true) {
            try {
                money = calSalary(salary, urlClassLoader);
                System.out.println("实际到手Money:" + money);
            } catch(Exception e) {
                e.printStackTrace();
                System.out.println("加载出现异常 ：" + e.getMessage());
            }
            Thread.sleep(5000);
        }
    }

    private static Double calSalary(Double salary, ClassLoader classloader) throws Exception {
        Class<?> clazz = classloader.loadClass("com.roy.oa.SalaryCaler");
        if (null != clazz) {
            Object object = clazz.newInstance();
            return (Double) clazz.getMethod("cal", Double.class).invoke(object, salary);
        }
        return -1.00;
    }
}
```

### 💡 拓展思考

1. **哪些 Jar 包适合外部加载？**
   - 规则引擎
   - 统一审批规则
   - 订单状态规则
2. **外部 Jar 包可以放到哪里？**
   - 本地磁盘
   - 远程 Web 服务器（`URLClassLoader` 支持 `http://`）
   - Maven 仓库（如 Drools 的远程规则加载）

------

## 🔐 自定义类加载器实现 Class 代码混淆

虽然经理看不到 OA 系统源码，但仍能反编译 Jar 包。
 👉 老王决定对 `class` 文件进行 **代码混淆**。

### ⚙️ 解决思路

1. 简单：修改文件后缀（`.class → .myclass`）
2. 稳妥：在二进制内容中加点“干扰”数据

由于 JDK 只能加载标准 `.class` 文件，所以需要 **自定义类加载器**。

### 🛠️ 自定义类加载器

```java
public class SalaryClassLoader extends SecureClassLoader {
    private String classPath;
    public SalaryClassLoader(String classPath) {
        this.classPath = classPath;
    }

    @Override
    protected Class<?> findClass(String fullClassName) throws ClassNotFoundException {
        String filePath = this.classPath + fullClassName.replace(".", "/").concat(".myclass");
        int code;
        try (FileInputStream fis = new FileInputStream(filePath);
             ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
            while ((code = fis.read()) != -1) {
                bos.write(code);
            }
            byte[] data = bos.toByteArray();
            return defineClass(fullClassName, data, 0, data.length);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

> 🔎 这样 `.myclass` 文件就能被加载，而普通反编译工具却无法直接还原。

然后，在OA系统中通过这个自定义类加载器加载计算工资的SalaryCaler类。

```java
public class OADemo3 {
    public static void main(String[] args) throws Exception {
        Double salary = 15000.00;
        Double money = 0.00;
        SalaryClassLoader salaryClassLoader = new SalaryClassLoader("/Users/roykingw/DevCode/ClassLoadDemo/out/production/SalaryCaler/");

        //模拟不停机状态
        while (true) {
            try {
                money = calSalary(salary,salaryClassLoader);
                System.out.println("实际到手Money:" + money);
            }catch(Exception e) {
                System.out.println("加载出现异常 ："+e.getMessage());
                System.exit(-1);
            }
            Thread.sleep(5000);
        }
    }

    private static Double calSalary(Double salary,ClassLoader classloader) throws Exception {
        Class<?> clazz = classloader.loadClass("com.roy.oa.SalaryCaler");
        if(null != clazz) {
            Object object = clazz.newInstance();
            return (Double)clazz.getMethod("cal", Double.class).invoke(object, salary);
        }
        return -1.00;
    }
}
```

> 这个简单的示例并没有修改class文件的内容，所以,myclass文件，可以通过修改.class文件生成。

 这个.myclass文件并没有修改文件的内容。如果要修改内容呢？二进制文件不太好直接编辑，可以使用流的方式做一点修改。

```java
public class FileTransferTest {
    public static void main(String[] args) throws Exception {
        FileInputStream fis = new FileInputStream("/Users/roykingw/DevCode/ClassLoadDemo/out/production/SalaryCaler/com/roy/oa/SalaryCaler.class");

        File targetFile = new File("/Users/roykingw/DevCode/ClassLoadDemo/out/production/SalaryCaler/com/roy/oa/SalaryCaler.myclass");
        if(targetFile.exists()) {
            targetFile.delete();
        }
        FileOutputStream fos = new FileOutputStream(targetFile);

        int code = 0;
       //在读文件之前，先写一个没有意义的1
        fos.write(1);
        while((code = fis.read())!= -1 ) {
            fos.write(code);
        }
        fis.close();
        fos.close();
        System.out.println("文件转换完成");
    }
}
```

> 这样就能生成一个简单加密后的.myclass文件了。在class文件的标准内容前面加了一个没用的1。对应的类加载器只需要把这个1忽略掉就可以了。

**拓展思考**

1、如何进一步提升关键代码的安全性？

 我们这个算法太简单了，经理看看类加载器的源码就知道，只要把.myclass文件前面的1去掉，就能拿到原来的class文件内容，从而进行反编译。有没有什么算法，可以让经理推导不出原始的class文件内容呢？

 常用的加密算法就派上用场了。MD5、对称加密、非对称加密...

 或者是不是能够有更多奇怪的思路，比如将类加载器的class文件也加密呢？通过自定义类加载器A，从一个加密class文件当中加载出一个类加载器B，再用后面这个类加载器B，加载加密过的核心代码。

2、如何在真实项目中用上这种机制？

 真实项目当中不会拿class文件直接部署，都是拿jar包进行部署。所以，我们要做的是，在自定义类加载器中，将从硬盘上读取class文件的实现方式，改为从jar包当中读取class文件。这个通过文件流照样很容易实现。

```java
public class SalaryJARLoader extends SecureClassLoader {
 private String jarFile;

 public SalaryJARLoader(String jarFile) {
  this.jarFile = jarFile;
 }

 @Override
 protected Class<?> findClass(String fullClassName) throws ClassNotFoundException {
  String classFilepath = fullClassName.replace('.', '/').concat(".class");
  System.out.println("重新加载类："+classFilepath);
  int code;
  try {
   // 访问jar包的url
   URL jarURL = new URL("jar:file:" + jarFile + "!/" + classFilepath);
//			InputStream is = jarURL.openStream();
   URLConnection urlConnection = jarURL.openConnection();
   // 不使用缓存 不然有些操作系统下会出现jar包无法更新的情况
   urlConnection.setUseCaches(false);
   InputStream is = urlConnection.getInputStream();
   ByteArrayOutputStream bos = new ByteArrayOutputStream();
   while ((code = is.read()) != -1) {
    bos.write(code);
   }
   byte[] data = bos.toByteArray();
   is.close();
   bos.close();
   return defineClass(fullClassName, data, 0, data.length);
  } catch (Exception e) {
   e.printStackTrace();
   System.out.println("加载出现异常 ："+e.getMessage());
   throw new ClassNotFoundException(e.getMessage());
//			return null;
  }
 }
}
```



------

## ♻️ 自定义类加载器实现热加载

老王又遇到新问题：
 👉 修改 Jar 包后，必须重启 OA 系统才能生效。

原因：类加载器缓存了已加载的类。
 解决方案：**每次都新建一个类加载器实例**。

```java
public class OADemo5 {
    public static void main(String[] args) throws Exception {
        Double salary = 15000.00;
        Double money = 0.00;

        // 模拟不停机状态
        while (true) {
            try {
                money = calSalary(salary);
                System.out.println("实际到手Money:" + money);
            } catch(Exception e) {
                System.out.println("加载出现异常 ：" + e.getMessage());
            }
            Thread.sleep(5000);
        }
    }

    private static Double calSalary(Double salary) throws Exception {
        SalaryJARLoader salaryClassLoader = new SalaryJARLoader("/Users/roykingw/lib/SalaryCaler.jar");
        System.out.println(salaryClassLoader.getParent());
        Class<?> clazz = salaryClassLoader.loadClass("com.roy.oa.SalaryCaler");
        if (null != clazz) {
            Object object = clazz.newInstance();
            return (Double) clazz.getMethod("cal", Double.class).invoke(object, salary);
        }
        return -1.00;
    }
}
```

通过这种方式，每次都是创建出一个新的SalaryJARLoader对象，那么他的缓存肯定是空的。那么他自然就只能每次都从jar包当中加载类了。于是，老王可以愉快的随时切换jar包，实现热更新了。

**拓展思考**

1、这个热加载机制看似很好用，为什么在开源项目中没有见过这种用法？

 很显然，这种热加载机制需要创建出非常多的ClassLoader对象。而这些不用的ClassLoader对象加载过的缓存对象也会随之成为垃圾。这会让JVM中本来就不大的元数据区带来很大的压力，极大的增加GC线程的压力。

 但是在项目开发时，其实是有一些办法可以实现这种类似的热更新机制。例如IDEA中的JRebel插件，还有之前介绍过的Arthas。

2、加载SalaryCaler的时候真的只加载一个类吗？

 把SalaryJARLoader加载过的类打印出来，你会发现，在加载SalaryCaler时，其实不光加载了这个类，同时还加载了Double和Object两个类。这两个类哪里来的？这就是JVM实现的懒加载机制。

 JVM为了提高类加载的速度，并不是在启动时直接把进程当中所有的类一次加载完成，而是在用到的时候才去加载。也就是懒加载。

### ⚠️ 注意

- 会不断产生新的 ClassLoader，增加 **GC 压力**
- 开源工具中常用：
  - 🔥 JRebel
  - 🔍 Arthas

---

## 🔥 打破双亲委派，实现同类多版本共存

就在老王跟资本家们斗得不亦乐乎的时候，另一个新手程序员小王突然给老王来了个**背刺**。

不知道为什么，小王在 OA 系统中提交了一个 `SalaryCaler` 类。这时，老王突然发现，这个看似没用的类却导致他刚刚实现的**热加载机制失效**。

> 不管 JAR 包如何更新，OA 系统总是只加载小王提交的那个 `SalaryCaler` 类。

------

### ⚠️ 为什么会出现这种情况？

原因就在于 **JDK 的双亲委派机制**。

自定义的 `SalaryJARLoader` 的 `parent` 属性指向 JDK 内的 `AppClassLoader`，而 `AppClassLoader` 会加载 OA 系统的所有代码，包括小王提交的 `SalaryCaler` 类。

于是，当 `SalaryJARLoader` 尝试加载 `SalaryCaler` 类时，通过双亲委派，自然加载的是 **AppClassLoader 中的版本**。

![双亲1](img\双亲1.png)

------

### 🛠️ 如何保持热加载机制不失效？

逻辑很简单：**让自定义类优先从 JAR 包中加载**。

#### 🔹 示例代码：打破双亲委派顺序

```java
public class SalaryJARLoader6 extends SecureClassLoader {
    private String jarFile;

    public SalaryJARLoader6(String jarFile) {
        this.jarFile = jarFile;
    }

    @Override
    public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        //MAC 下会不断加载 Object 类，出现栈溢出的问题.Windows下测试是没有问题的。
//		if(name.startsWith("com.roy")) {
//			return this.findClass(name);
//		}else {
//			return super.loadClass(name);
//		}
        // 将双亲委派机制反过来：先子类加载器，再父类加载器
        Class<?> c = null;
        synchronized (getClassLoadingLock(name)) {
            c = findLoadedClass(name);
            if (c == null) {
                c = findClass(name);
                if (c == null) {
                    c = super.loadClass(name, resolve);
                }
            }
        }
        return c;
    }

    @Override
    protected Class<?> findClass(String fullClassName) throws ClassNotFoundException {
        String classFilepath = fullClassName.replace('.', '/').concat(".class");
        System.out.println("重新加载类：" + classFilepath);
        try {
            URL jarURL = new URL("jar:file:" + jarFile + "!/" + classFilepath);
            URLConnection urlConnection = jarURL.openConnection();
            urlConnection.setUseCaches(false);
            InputStream is = urlConnection.getInputStream();
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            int code;
            while ((code = is.read()) != -1) {
                bos.write(code);
            }
            byte[] data = bos.toByteArray();
            is.close();
            bos.close();
            return defineClass(fullClassName, data, 0, data.length);
        } catch (Exception e) {
            // 出现异常时交由父加载器加载
            return null;
        }
    }
}
```

------

### 💡 拓展思考

#### 1️⃣ 是否可以通过打破双亲委派绕过 JDK 沙箱？

> ❌ 不可以

JDK 内部的三个核心类加载器无法修改，因此核心类仍然安全。

- 从 JDK9 开始引入模块化机制，每个类加载器只负责部分模块，但自定义类加载器依然沿用双亲委派机制。
- JDK17 内部加载机制变化不大，案例几乎可以平滑迁移。

> ⚠️ 注意：是“几乎”，而不是“完全”，因为模块化影响全局，但核心加载流程仍然安全。

------

#### 2️⃣ 真实项目中什么时候需要打破双亲委派？

双亲委派机制是基础体系，很多框架定制时需要打破它。

#### 🔹 Tomcat 类加载体系示意

![tomcat类加载体系](img\tomcat类加载体系.png)



主要类加载器：

- `commonLoader`：Tomcat最基本的类加载器，加载路径中的class可以被Tomcat容器本身以及各个Webapp访问；
- `catalinaLoader`：Tomcat容器私有的类加载器，加载路径中的class对于Webapp不可见；
- `sharedLoader`：各个Webapp共享的类加载器，加载路径中的class对于所有Webapp可见，但是对于Tomcat容器不可见；
- `WebappClassLoader`：各个Webapp私有的类加载器，加载路径中的class只对当前Webapp可见，比如加载war包里相关的类，每个war包应用都有自己的WebappClassLoader，实现相互隔离，比如不同war包应用引入了不同的spring版本，这样实现就能加载各自的spring版本；
- `Jsp 类加载器`：针对每个JSP页面创建一个加载器。这个加载器比较轻量级，所以Tomcat还实现了热加载，也就是JSP只要修改了，就创建一个新的加载器，从而实现了JSP页面的热更新。

> 现在可以理解 Tomcat 设计类加载体系的原因了！

------

#### 3️⃣ SpringBoot 案例

SpringBoot 有自己的 **SPI 服务注入机制**，可以加载 `ApplicationContextInitializer` 接口的所有实现：

```
public class SPITest {
    public static void main(String[] args) {
        List<String> names = SpringFactoriesLoader.loadFactoryNames(
            ApplicationContextInitializer.class, null
        );
        names.forEach(System.out::println);

        List<ApplicationContextInitializer> applicationContextInitializers =
            SpringFactoriesLoader.loadFactories(ApplicationContextInitializer.class, null);
        applicationContextInitializers.forEach(System.out::println);
    }
}
```

> 💡 奇怪现象：`loadFactories` 第二个参数本来是 ClassLoader，但传 null 也能处理。
>  这个设计背后的思路可能与双亲委派和加载器隔离有关，值得后续验证。

---

## ⚡ 使用类加载器能不能不用反射？

对于一般程序员，故事到这里就结束了。接下来的部分，则属于**有追求的程序员**——继续打磨技术，追求真理的阶段。

> 💡 `没事找事的无聊时间`
>  如果觉得有点烧脑，不要强行跟进，可以先跳过。

------

老王分析了热加载器失效的原因，其实就是因为 OA 应用的多个类加载器中，存在了 **SalaryCaler 类的多个版本**。

![双亲1](D:\workspace\javaworkspace\img\双亲1.png)

- **AppClassLoader** 中的 `SalaryCaler` 对象可以直接 `new`。
- **SalaryJARLoader** 中的 `SalaryCaler` 对象，则只能通过**反射**操作。

> 🤔 为什么不能像正常类一样直接使用？

老王尝试了一种“简单粗暴”的方式：

```
SalaryCaler caler2 = (SalaryCaler)obj;
money = caler2.cal(salary);
```

但是，结果惨烈：

```
Exception in thread "main" java.lang.ClassCastException: com.roy.oa.SalaryCaler cannot be cast to com.roy.oa.SalaryCaler
```

> 是的，你没看错，“我不能转换成我自己”，这就是类加载器导致的类型隔离问题。

------

### 🧩 SPI 扩展机制救场

为了摆脱反射，JDK 提供了 **SPI（Service Provider Interface）机制**：

- 核心 API：`ServiceLoader.load(SalaryCalService.class)`
- 配置文件位置：`${classpath}/META-INF/services/`
- 文件名：接口全类名
- 文件内容：实现类全类名（每行一个）

> ⚡ `${classpath}` 可以是项目依赖路径，也可以是当前项目路径
>  SPI 是一种非常好的扩展机制，很多开源框架（如 ShardingSphere）和 SpringBoot 都大量使用。

SPI 的实现也依赖 ClassLoader：

```
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```

------

### 🔹 使用 SPI 统一接口调用

定义一个接口 `SalaryCalService`，不同实现类分别由不同 JAR 提供：

```java
public class OADemo8 {
    public static void main(String[] args) throws Exception {
        Double salary = 15000.00;

        while (true) {
            SalaryJARLoader6 salaryJARLoader = new SalaryJARLoader6("/Users/roykingw/lib/SalaryCaler.jar");
            SalaryCalService salaryService = getSalaryService(salaryJARLoader);
            System.out.println("应该到手Money:" + salaryService.cal(salary));

            SalaryJARLoader6 salaryJARLoader2 = new SalaryJARLoader6("/Users/roykingw/lib2/SalaryCaler.jar");
            SalaryCalService salaryService2 = getSalaryService(salaryJARLoader2);
            System.out.println("实际到手Money:" + salaryService2.cal(salary));

            SalaryCalService salaryService3 = getSalaryService(null);
            System.out.println("OA系统计算的Money:" + salaryService3.cal(salary));

            Thread.sleep(5000);
        }
    }

    private static SalaryCalService getSalaryService(ClassLoader classloader){
        ServiceLoader<SalaryCalService> services;
        if(classloader == null){
            services = ServiceLoader.load(SalaryCalService.class);
        } else {
            ClassLoader original = Thread.currentThread().getContextClassLoader();
            Thread.currentThread().setContextClassLoader(classloader);
            services = ServiceLoader.load(SalaryCalService.class);
            Thread.currentThread().setContextClassLoader(original);
        }

        Iterator<SalaryCalService> iterator = services.iterator();
        return iterator.hasNext() ? iterator.next() : null;
    }
}
```

> ✅ 这样就能**摆脱别扭的反射调用**，通过统一接口访问不同类加载器下的实现。

------

### 💡 拓展思考

1. **SPI 配置文件能放在 JAR 包中吗？**
   - 如果想在 JAR 内定义 `SalaryCalService` 的实现，需要确保 ClassLoader 可以找到 META-INF/services 下的文件。
   - 示例 OADemo9 就提供了一种做法。
2. **SpringBoot 与 SPI 的关系**
   - SpringBoot 在 SPI 基础上做了大量封装，提供了自己的扩展点。
   - 后续可以尝试从 SPI 角度理解 SpringBoot 的功能扩展机制。

---

# 📘 JVM内存模型深度剖析与优化

## **📖 Java语言的跨平台特性**

![Java跨语言特性](img\Java跨语言特性.png)

## 🔐 **JVM整体结构及内存模型**

![JVM内存模型](D:\workspace\javaworkspace\img\JVM内存模型.png)

### 1️⃣ JVM 运行时数据区（JDK8 内存模型）

- **🧩 堆（Heap）**
  - 存放对象实例（如 `math`、`user` 对象）。
  - 包含 **新生代 (Eden + Survivor)** 和 **老年代**。
  - 垃圾回收：
    - **Minor GC**：回收新生代。
    - **Full GC**：回收整个堆和方法区。
  - 空间不足 → GC 后仍不足 → **OOM (OutOfMemoryError)**。
- **📚 方法区（元空间 Metaspace）**
  - 存放类信息、常量、静态变量、方法元数据。
  - 示例：`Math.class` 被加载到方法区。
- **📦 栈（线程栈）**
  - 每个线程独立。
  - 存放方法调用的 **栈帧**（局部变量表、操作数栈、动态链接、方法出口）。
  - 示例：`main()` 栈帧、`compute()` 栈帧。
- **⏺ 程序计数器（PC Register）**
  - 记录线程下一条待执行的字节码指令地址。
- **🔧 本地方法栈**
  - 调用 **native 方法** 时使用。
- **📦 直接内存**
  - 属于非堆内存。
  - 用于 NIO（零拷贝），通过 `ByteBuffer.allocateDirect` 分配。

------

### 2️⃣ 方法调用过程

- **▶️ main 线程执行**
  - `main()` 方法入栈，生成局部变量表（如 `math`）。
  - 调用 `compute()` → 形成新栈帧，包含：
    - 局部变量表：`a=1, b=2, c=30`
    - 操作数栈：用于指令计算（FILO 原则）。
    - 动态链接：指向运行时常量池。
    - 方法出口：返回 `main()`。
- **📥 类加载**
  - `new Math()` → 类加载器将 `Math.class` 加载至方法区。
  - 在堆中创建对象，局部变量表 `math` 指向该对象。
- **⚙️ 字节码执行引擎**
  - 通过 **解释执行/即时编译 (JIT)** 执行字节码指令。
  - 执行时可能修改方法区中的运行时常量池或类元数据。

------

### 3️⃣ 垃圾回收 (GC)

- **♻️ 分代回收机制**
  - 新生代：
    - Eden 占 **8/10**，两个 Survivor 各 **1/10**。
    - 对象经历多次 GC → 晋升到老年代。
  - 老年代满 → **Full GC** → 仍不足 → **OOM**。
- **🛑 STW (Stop-The-World)**
  - GC 过程中会 **暂停用户线程**。

------

### 4️⃣ 多线程结构

- 每个线程独立：
  - **PC 寄存器**
  - **线程栈**
  - **本地方法栈**
- 所有线程共享：
  - **堆**
  - **方法区**

> Minor GC 并不是直接“挪动”对象，而是 **复制到新的内存区域**；           
>
> 原区域通过**移动分配指针到区域开始位置**来“清空”；在复制过程中，**所有引用会被更新为新地址**，确保对象引用的正确性。



## 📦 JVM内存参数设置

![内存参数设置](img\内存参数设置.png)

### ⚙️ Spring Boot 程序的 JVM 参数设置

📌 **Tomcat 启动方式**：在 `bin/catalina.sh` 文件中添加 JVM 参数
 📌 **Spring Boot Jar 启动方式**：直接在命令行追加参数

------

#### 🚀 启动示例

```
java \
  -Xms2048M \
  -Xmx2048M \
  -Xmn1024M \
  -Xss512K \
  -XX:MetaspaceSize=256M \
  -XX:MaxMetaspaceSize=256M \
  -jar microservice-eureka-server.jar
```

------

### 🗂 JVM 参数说明

#### 🔹 内存参数

- **`-Xss`**：每个线程的栈大小（默认约 1M，设置过大会影响线程数量）。
- **`-Xms`**：堆的初始大小，默认物理内存的 **1/64**。
- **`-Xmx`**：堆的最大大小，默认物理内存的 **1/4**。
- **`-Xmn`**：新生代大小（Eden + 2 Survivor）。

------

#### 🔹 分代比例

- **`-XX:NewRatio`**
  - 默认值 = 2
  - 含义：新生代 : 老年代 = 1 : 2 → 新生代占堆内存的 **1/3**。
- **`-XX:SurvivorRatio`**
  - 默认值 = 8
  - 含义：一个 Survivor 区占 Eden 的 1/8，即 **新生代的 1/10**。

------

#### 🔹 元空间参数（Metaspace）

> 元空间替代了 JDK8 之前的永久代（PermGen）

- **`-XX:MaxMetaspaceSize`**
  - 设置元空间最大值。
  - 默认：`-1` → 不限制，仅受本地内存大小限制。
- **`-XX:MetaspaceSize`**
  - 指定 **触发 Full GC 的初始阈值**（不是固定初始大小）。
  - 默认约 **21M**。
  - 达到该值 → 触发 Full GC，卸载类型信息，并动态调整：
    - 若释放了大量空间 → 降低阈值。
    - 若释放空间少 → 在不超过 `MaxMetaspaceSize` 的情况下，提高阈值。

------

#### 💡 调优建议

- 由于 **调整元空间大小需要 Full GC（代价昂贵）**，如果应用启动时频繁 Full GC，大多是 **Metaspace 动态调整导致**。
- 建议：
  - **`MetaspaceSize` 与 `MaxMetaspaceSize` 设置为相同值**。
  - **数值大于默认值**，避免频繁触发 Full GC。
  - 例如：
    - 机器内存 8G → 一般设为 `256M`。

---

# JVM对象创建与内存分配机制深度剖析

## 🛠️ 类加载与内存分配流程

![对象的创建](img\对象的创建.png)

### **🔍 1. 类加载检查**

虚拟机遇到 `new` 指令时，首先检查参数是否能在常量池中找到类的符号引用，并且确认该类是否已经加载、解析和初始化。如果没有，必须执行相应的类加载过程。
 `new` 指令包括语言层面的 `new` 关键字、对象克隆、对象序列化等。

------

### **💾 2. 分配内存**

在类加载检查通过后，虚拟机为新生对象分配内存。对象所需内存大小在类加载后已经确定，内存分配任务就是从 Java 堆中划分出一块内存。

#### 划分内存方法：

- **🪙 指针碰撞**
   如果 Java 堆内存是规整的，已使用内存和空闲内存分开，指针作为分界点指示，分配内存时仅需将指针移动。
- **📋 空闲列表**
   如果内存交错，虚拟机维护一个空闲内存块的列表，在分配时从中找到足够大的空间。

#### 解决并发问题的方法：

- **🔄 CAS (Compare And Swap)**
   使用 CAS 配合失败重试机制保证内存分配的原子性。
- **🧵 本地线程分配缓冲（TLAB）**
   每个线程有独立的小块内存进行分配，虚拟机默认开启。
   使用 `-XX:+UseTLAB` 参数控制是否启用，`-XX:TLABSize` 设置 TLAB 的大小。

------

### **⚙️ 3. 初始化零值**

内存分配后，虚拟机会将内存空间初始化为零值（不包括对象头）。如果使用 TLAB，这一步可以提前完成。
初始化零值确保对象字段在未赋值时使用零值，可以直接在代码中访问。

------

### 🔑 4. **设置对象头**

虚拟机对对象设置必要信息，例如：

- 对象所属类的信息
- 对象的哈希码
- GC 分代年龄等
   这些信息存储在 **对象头** 中。

在 HotSpot 虚拟机中，对象内存布局包括三个区域：

1. **🛠️ 对象头**
2. **📊 实例数据**
3. **🔳 对齐填充**

**对象头（Header）** 包括：

- 运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁等
- 类型指针，指向对象所属类的元数据

#### 32位对象头示意：

![对象头](img\对象头.png)

------

### ⚡ 5. **执行方法**

对象初始化后，根据程序员的意图执行赋值和构造方法。
 此步骤与零值初始化不同，程序员明确给属性赋值。

------

### 🔒 6. **对象半初始化**

- **📏 对象大小与指针压缩**
   对象的大小可以通过使用 `jol-core` 包进行查看。

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.9</version>
</dependency>
```

```java
import org.openjdk.jol.info.ClassLayout;

/**
 * 计算对象大小
 */
public class JOLSample {

    public static void main(String[] args) {
        ClassLayout layout = ClassLayout.parseInstance(new Object());
        System.out.println(layout.toPrintable());

        System.out.println();
        ClassLayout layout1 = ClassLayout.parseInstance(new int[]{});
        System.out.println(layout1.toPrintable());

        System.out.println();
        ClassLayout layout2 = ClassLayout.parseInstance(new A());
        System.out.println(layout2.toPrintable());
    }

    // -XX:+UseCompressedOops           默认开启的压缩所有指针
    // -XX:+UseCompressedClassPointers  默认开启的压缩对象头里的类型指针Klass Pointer
    // Oops : Ordinary Object Pointers
    public static class A {
                       //8B mark word
                       //4B Klass Pointer   如果关闭压缩-XX:-UseCompressedClassPointers或-XX:-UseCompressedOops，则占用8B
        int id;        //4B
        String name;   //4B  如果关闭压缩-XX:-UseCompressedOops，则占用8B
        byte b;        //1B 
        Object o;      //4B  如果关闭压缩-XX:-UseCompressedOops，则占用8B
    }
}


运行结果：
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)    //mark word
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)    //mark word     
      8     4        (object header)                           e5 01 00 f8 (11100101 00000001 00000000 11111000) (-134217243)    //Klass Pointer
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total


[I object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           6d 01 00 f8 (01101101 00000001 00000000 11111000) (-134217363)
     12     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     0    int [I.<elements>                             N/A
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total


com.tuling.jvm.JOLSample$A object internals:
 OFFSET  SIZE               TYPE DESCRIPTION                               VALUE
      0     4                    (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                    (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                    (object header)                           61 cc 00 f8 (01100001 11001100 00000000 11111000) (-134165407)
     12     4                int A.id                                      0
     16     1               byte A.b                                       0
     17     3                    (alignment/padding gap)                  
     20     4   java.lang.String A.name                                    null
     24     4   java.lang.Object A.o                                       null
     28     4                    (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 3 bytes internal + 4 bytes external = 7 bytes total
```

什么是java对象的**指针压缩**？

1.jdk1.6 update14开始，在64bit操作系统中，JVM支持指针压缩

2.jvm配置参数:UseCompressedOops，compressed--压缩、oop(ordinary object pointer)--对象指针

3.启用指针压缩:-XX:+UseCompressedOops(**默认开启**)，禁止指针压缩:-XX:-UseCompressedOops

为什么要进行指针压缩？

1.在64位平台的HotSpot中使用32位指针(实际存储用64位)，内存使用会多出1.5倍左右，使用较大指针在主内存和缓存之间移动数据，**占用较大宽带，同时GC也会承受较大压力**

2.为了减少64位平台下内存的消耗，启用指针压缩功能

3.在jvm中，32位地址最大支持4G内存(2的32次方)，可以通过对对象指针的存入**堆内存**时压缩编码、取出到**cpu寄存器**后解码方式进行优化(对象指针在堆中是32位，在寄存器中是35位，2的35次方=32G)，使得jvm只用32位地址就可以支持更大的内存配置(小于等于32G)

4.堆内存小于4G时，不需要启用指针压缩，jvm会直接去除高32位地址，即使用低虚拟地址空间

5.堆内存大于32G时，压缩指针会失效，会强制使用64位(即8字节)来对java对象寻址，这就会出现1的问题，所以堆内存不要大于32G为好

**关于对齐填充：**对于大部分处理器，对象以8字节整数倍来对齐填充都是最高效的存取方式。

## 🛠️ **Java 对象的指针压缩**

### 🔍1. **什么是指针压缩？**

指针压缩是 JVM 在 64 位操作系统中，优化对象指针存储的技术。从 **JDK 1.6 Update 14** 开始，JVM 支持在 64 位平台上使用压缩指针，目的是减少内存的使用。

- **32 位指针**：在堆内存中使用 32 位指针（4 字节），而在 CPU 寄存器中保持 64 位指针。
- **启用指针压缩**：通过在堆内存中存储 32 位指针，能够节省大量内存。

### ⚙️2. JVM 配置参数

- **`UseCompressedOops`**: 启用指针压缩。
  - 启用：`-XX:+UseCompressedOops`（默认开启）
  - 禁用：`-XX:-UseCompressedOops`

------

### 💡3. **为什么要进行指针压缩？**

#### 💰 1. **减少内存消耗**

- **64 位指针**：每个对象会占用 8 字节的指针内存。
- **32 位指针**：通过压缩，每个对象只占用 4 字节的指针内存，减少了大约 1.5 倍的内存消耗。

#### 🚀 2. **提高性能**

- **带宽问题**：较大的指针占用更多带宽，且 GC 会承受更大压力。
- **减少内存带宽压力**：指针压缩减少了内存占用，有助于提高内存带宽和缓存的效率。

#### 🧑‍💻 3. **支持更大内存配置**

- **32 位指针**：能够支持最大 **32GB** 的内存，通过对象指针的压缩编码和解码来优化内存。
  - 在堆内存中，指针为 32 位，而在 CPU 寄存器中为 64 位，能够处理更大的内存空间。

------

### 📏 4. **内存大小与指针压缩使用**

#### 🔽 1. **堆内存小于 4G**

指针压缩不需要启用，JVM 会直接使用低虚拟地址空间。

#### 🔼 2. **堆内存大于 4G 小于 32G**

启用指针压缩，JVM 使用 32 位指针，能够有效减少内存使用。

#### ⚠️ 3. **堆内存大于 32G**

指针压缩会失效，强制使用 64 位指针，导致内存占用增加，建议堆内存不要超过 32GB。

------

### 📊5. 如何工作

- **编码与解码**：在堆内存中，指针是 32 位；在寄存器中，指针解码为 64 位。
- **最大支持 32GB 内存**：通过 32 位压缩指针和 64 位解压指针，可以在 64 位系统上有效地支持更大的内存空间。

------

### 🧰6. 对齐填充

- **对齐优化**：大多数处理器在内存访问时，采用 8 字节对齐来提高效率。JVM 会确保对象按 8 字节对齐，从而提升对象的访问速度。

---

## 🏗️ **对象内存分配**

### 📊 1. **对象内存分配流程图**

![对象内存分配流程图](img\对象内存分配流程图.png)

### 💡 2. 对象栈上分配

Java 中的对象通常是在 **堆上** 进行分配的。当对象没有被引用时，垃圾回收器（GC）会负责回收它们的内存。然而，当对象数量较多时，GC 的负担会增加，从而影响应用程序的性能。

为了减少临时对象在堆内分配的数量，JVM 通过 **逃逸分析** 来判断某个对象是否会被外部访问。如果对象不会逃逸，则可以将该对象 **栈上分配**。栈上分配的对象会随着栈帧的出栈而销毁，从而减轻 GC 的压力。

**对象逃逸分析**

对象逃逸分析就是通过分析对象的动态作用域，判断它是否会被外部方法所引用。例如，如果对象作为参数传递到其他方法，就算它在当前方法中创建，作用域就会“逃逸”到外部。

------

### 🧑‍💻 3. **示例：逃逸分析与栈上分配**

```java
public User test1() {
   User user = new User();
   user.setId(1);
   user.setName("zhuge");
   //TODO 保存到数据库
   return user;
}

public void test2() {
   User user = new User();
   user.setId(1);
   user.setName("zhuge");
   //TODO 保存到数据库
}
```

- **`test1` 方法** 中的 `user` 对象被返回，作用域不明确，可能会逃逸。
- **`test2` 方法** 中的 `user` 对象，在方法结束后不再被引用，可以认为它是无效对象，适合栈上分配。

JVM 在开启逃逸分析的情况下，可以通过 **标量替换** 将这些临时对象分配到栈上，避免堆内存分配和 GC 的压力。

------

### 🔧 4. **JVM 参数**

- **开启逃逸分析**：`-XX:+DoEscapeAnalysis`（JDK 7+ 默认开启）
- **关闭逃逸分析**：`-XX:-DoEscapeAnalysis`
- **开启标量替换**：`-XX:+EliminateAllocations`（JDK 7+ 默认开启）
- **关闭标量替换**：`-XX:-EliminateAllocations`

------

### 🔍 5. **标量替换**

当逃逸分析确认一个对象不会被外部访问且能够进一步分解时，JVM 不会创建该对象，而是将其成员变量分解为多个独立的成员变量，在栈帧或寄存器中分配内存。这样可以避免因为没有连续的大块内存而导致对象内存分配失败。

**标量与聚合量**：

- **标量**：不可进一步分解的基本类型（如 `int`, `long`）和引用类型。
- **聚合量**：可以进一步分解的复杂对象（如 `User` 类对象）。

------

### 🏃‍♂️ 6. **栈上分配示例**

```java
/**
 * 栈上分配，标量替换
 * 代码调用了1亿次alloc()，如果是分配到堆上，大概需要1GB以上堆空间，
 * 如果堆空间小于该值，必然会触发GC。
 * 
 * 使用如下参数不会发生GC：
 * -Xmx15m -Xms15m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations
 * 使用如下参数都会发生大量GC：
 * -Xmx15m -Xms15m -XX:-DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations
 * -Xmx15m -Xms15m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:-EliminateAllocations
 */
public class AllotOnStack {

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100000000; i++) {
            alloc();
        }
        long end = System.currentTimeMillis();
        System.out.println(end - start);
    }

    private static void alloc() {
        User user = new User();
        user.setId(1);
        user.setName("zhuge");
    }
}
```

**结论：栈上分配依赖于逃逸分析和标量替换**

---

## 🏗️ **对象在 Eden 区分配**

### 📊 1. **Eden 区分配与 Minor GC**

大多数情况下，对象会在 **新生代的 Eden 区** 中进行分配。当 Eden 区没有足够的空间时，JVM 会发起一次 **Minor GC**。在 **Minor GC** 过程中，新生代的对象会被清理，未被回收的存活对象会被移到 **Survivor 区** 或 **老年代**。

------

### ⚙️ 2. **Minor GC vs Full GC**

**Minor GC / Young GC**：

- 发生在新生代的垃圾回收。
- 频繁且回收速度较快。

**Major GC / Full GC**：

- 发生在新生代、老年代和方法区。
- 通常比 Minor GC 慢 10 倍以上。

------

### 💡 3. **Eden 区与 Survivor 区比例 (默认 8:1:1)**

- 大多数对象被分配到 Eden 区，当 Eden 区满了时，会触发 **Minor GC**，大部分对象会被回收。
- 存活的对象会被移动到空闲的 Survivor 区。
- 默认比例为 **8:1:1**，即 Eden 区占 8 倍于 Survivor 区的大小。
  - **Eden 区大，Survivor 区小**，这样可以最大限度地利用 Eden 区的空间。
- 默认启用 **-XX:+UseAdaptiveSizePolicy**，该参数会动态调整比例。如果需要禁用自动调整，使用 **-XX:-UseAdaptiveSizePolicy**。

### 🧑‍💻 4. 示例：Eden 区的内存分配

```java
//添加运行JVM参数： -XX:+PrintGCDetails
public class GCTest {
   public static void main(String[] args) throws InterruptedException {
      byte[] allocation1, allocation2/*, allocation3, allocation4, allocation5, allocation6*/;
      allocation1 = new byte[60000*1024];

      //allocation2 = new byte[8000*1024];

      /*allocation3 = new byte[1000*1024];
     allocation4 = new byte[1000*1024];
     allocation5 = new byte[1000*1024];
     allocation6 = new byte[1000*1024];*/
   }
}

运行结果：
Heap
 PSYoungGen      total 76288K, used 65536K [0x000000076b400000, 0x0000000770900000, 0x00000007c0000000)
  eden space 65536K, 100% used [0x000000076b400000,0x000000076f400000,0x000000076f400000)
  from space 10752K, 0% used [0x000000076fe80000,0x000000076fe80000,0x0000000770900000)
  to   space 10752K, 0% used [0x000000076f400000,0x000000076f400000,0x000000076fe80000)
 ParOldGen       total 175104K, used 0K [0x00000006c1c00000, 0x00000006cc700000, 0x000000076b400000)
  object space 175104K, 0% used [0x00000006c1c00000,0x00000006c1c00000,0x00000006cc700000)
 Metaspace       used 3342K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 361K, capacity 388K, committed 512K, reserved 1048576K
```

我们可以看出eden区内存几乎已经被分配完全（即使程序什么也不做，新生代也会使用至少几M内存）。**假如我们再为allocation2分配内存会出现什么情况呢？**

```java
//添加运行JVM参数： -XX:+PrintGCDetails
public class GCTest {
   public static void main(String[] args) throws InterruptedException {
      byte[] allocation1, allocation2/*, allocation3, allocation4, allocation5, allocation6*/;
      allocation1 = new byte[60000*1024];

      allocation2 = new byte[8000*1024];

      /*allocation3 = new byte[1000*1024];
      allocation4 = new byte[1000*1024];
      allocation5 = new byte[1000*1024];
      allocation6 = new byte[1000*1024];*/
   }
}

运行结果：
[GC (Allocation Failure) [PSYoungGen: 65253K->936K(76288K)] 65253K->60944K(251392K), 0.0279083 secs] [Times: user=0.13 sys=0.02, real=0.03 secs] 
Heap
 PSYoungGen      total 76288K, used 9591K [0x000000076b400000, 0x0000000774900000, 0x00000007c0000000)
  eden space 65536K, 13% used [0x000000076b400000,0x000000076bc73ef8,0x000000076f400000)
  from space 10752K, 8% used [0x000000076f400000,0x000000076f4ea020,0x000000076fe80000)
  to   space 10752K, 0% used [0x0000000773e80000,0x0000000773e80000,0x0000000774900000)
 ParOldGen       total 175104K, used 60008K [0x00000006c1c00000, 0x00000006cc700000, 0x000000076b400000)
  object space 175104K, 34% used [0x00000006c1c00000,0x00000006c569a010,0x00000006cc700000)
 Metaspace       used 3342K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 361K, capacity 388K, committed 512K, reserved 1048576K
```

**解释：**

当 Eden 区内存几乎被分配完时，虚拟机会触发 **Minor GC**。在 GC 过程中，存活的对象会被移动到 Survivor 区。如果 Survivor 区空间不足，一部分对象会被提前转移到 **老年代**。这次 **Minor GC** 后，分配的对象可以继续在 Eden 区中分配内存。

### 🔧 5. **示例：更多对象分配**

```java
public class GCTest {
   public static void main(String[] args) throws InterruptedException {
      byte[] allocation1, allocation2, allocation3, allocation4, allocation5, allocation6;
      allocation1 = new byte[60000 * 1024];  // 60MB
      allocation2 = new byte[8000 * 1024];   // 8MB
      allocation3 = new byte[1000 * 1024];   // 1MB
      allocation4 = new byte[1000 * 1024];   // 1MB
      allocation5 = new byte[1000 * 1024];   // 1MB
      allocation6 = new byte[1000 * 1024];   // 1MB
   }
}
```

**运行结果**：

```java
[GC (Allocation Failure) [PSYoungGen: 65253K->952K(76288K)] 65253K->60960K(251392K), 0.0311467 secs]
Heap
 PSYoungGen      total 76288K, used 13878K [0x000000076b400000, 0x0000000774900000, 0x00000007c0000000)
  eden space 65536K, 19% used [0x000000076b400000,0x000000076c09fb68,0x000000076f400000)
  from space 10752K, 8% used [0x000000076f400000,0x000000076f4ee030,0x000000076fe80000)
  to   space 10752K, 0% used [0x0000000773e80000,0x0000000773e80000,0x0000000774900000)
 ParOldGen       total 175104K, used 60008K [0x00000006c1c00000, 0x00000006cc700000, 0x000000076b400000)
  object space 175104K, 34% used [0x00000006c1c00000,0x00000006c569a010,0x00000006cc700000)
 Metaspace       used 3343K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 361K, capacity 388K, committed 512K, reserved 1048576K
```

---

## 📦 大对象直接进入老年代

**大对象**是指需要大量连续内存空间的对象，例如字符串或数组。为了优化内存分配效率，JVM提供了参数-XX:PretenureSizeThreshold来设置大对象的大小阈值。如果对象大小超过该阈值，将直接进入**老年代**，而不会进入年轻代。此参数仅在**Serial**和**ParNew**垃圾收集器下有效。

### 🛠 示例

设置JVM参数：
-XX:PretenureSizeThreshold=1000000（单位：字节）
-XX:+UseSerialGC

运行程序后，若对象大小超过1000000字节，该对象将直接分配到老年代。

### ❓ 为什么要这样做？

为避免大对象在年轻代进行复制操作（Copying算法）时降低效率，直接分配到老年代可以减少性能开销。

------

## ⏳ 长期存活的对象进入老年代

JVM采用**分代收集**策略管理内存，需区分对象应分配在**新生代**还是**老年代**。为此，JVM为每个对象设置了一个**年龄计数器（Age）**。

### 🔄 对象年龄机制

1. 对象在**Eden区**创建，经过第一次**Minor GC**后若仍存活且能被**Survivor区**容纳，则被移至Survivor区，年龄设为1。
2. 每次在Survivor区经历Minor GC，年龄+1。
3. 当对象年龄达到阈值（默认15岁，CMS收集器默认6岁，可通过-XX:MaxTenuringThreshold设置），对象晋升到**老年代**。

### 🎯 动态年龄判断

当**Survivor区**中一批对象的总大小超过该区域内存的50%（可通过-XX:TargetSurvivorRatio设置），年龄**大于等于**该批对象中最大年龄的对象将直接进入老年代。

**示例**：
若Survivor区中年龄1+年龄2+...+年龄n的对象总和超过Survivor区50%，则年龄≥n的对象直接晋升老年代。此机制在**Minor GC**后触发，旨在让可能长期存活的对象尽早进入老年代。

---

## 🛡️ 老年代空间分配担保机制

在**Minor GC**之前，JVM会检查老年代的剩余可用空间：

1. 若老年代可用空间 < 年轻代所有对象（包括垃圾对象）大小之和，JVM会检查是否设置了-XX:-HandlePromotionFailure（JDK 1.8默认开启）。
2. 若设置了该参数，JVM会比较老年代可用空间与每次Minor GC后进入老年代对象的**平均大小**：
   - 若老年代可用空间 ≥ 平均大小，继续执行Minor GC。
   - 若小于或未设置该参数，触发**Full GC**，同时回收老年代和年轻代。
3. 若Minor GC后存活对象仍无法放入老年代（老年代空间不足），再次触发Full GC。
4. 若Full GC后仍无足够空间，则抛出**OutOfMemoryError (OOM)**。

![空间分配担保](img\空间分配担保.png)

---

## 🗑️ 对象内存回收

垃圾回收的第一步是判断哪些对象已“死亡”（即无法被任何途径使用）。

### 🔢 引用计数法

为对象添加一个引用计数器：

- 每次被引用，计数器+1。
- 引用失效，计数器-1。
- 计数器为0的对象视为不可用。

**缺点**：无法解决对象间的**循环引用**问题。例如：

```java
public class ReferenceCountingGc {
   Object instance = null;

   public static void main(String[] args) {
      ReferenceCountingGc objA = new ReferenceCountingGc();
      ReferenceCountingGc objB = new ReferenceCountingGc();
      objA.instance = objB;
      objB.instance = objA;
      objA = null;
      objB = null;
   }
}
```

### 🌳 可达性分析算法

**可达性分析算法**用于判断对象是否为垃圾对象。算法以**GC Roots**为起点，沿引用链向下搜索，所有被标记的对象为**非垃圾对象**，未标记的对象则为**垃圾对象**。

#### 🔗 GC Roots

GC Roots包括以下节点：

- **线程栈的本地变量**（如方法中的局部变量）
- **静态变量**（类中定义的static变量）
- **本地方法栈的变量**（Native方法引用的对象）
- 其他（如常量池中的引用）

![可达性](img\可达性.jpeg)

### 🔗 常见引用类型

Java提供四种引用类型，强度从高到低依次为：

1. **强引用**
   普通变量引用，GC不会回收。

   ```java
   public static User user = new User();
   ```

2. **软引用**
   使用SoftReference包裹，正常情况下不会被回收，但若GC后内存仍不足，则会被回收。常用于**内存敏感的缓存**（如浏览器后退功能）。

   ```java
   public static SoftReference<User> user = new SoftReference<User>(new User());
   ```

   **应用场景**：

   - 若网页在浏览结束时回收，按后退需重新请求页面。
   - 若全部存储在内存，可能导致内存浪费或溢出。软引用可平衡性能与内存使用。

3. **弱引用**
   使用WeakReference包裹，GC时直接回收，类似无引用状态，使用较少。

   ```java
   public static WeakReference<User> user = new WeakReference<User>(new User());
   ```

4. **虚引用**
   也称幽灵引用或幻影引用，是最弱的引用类型，几乎不用。

------

### ⚖️ finalize()方法与对象存活判定

即使对象在可达性分析中不可达，仍有机会通过finalize()方法“自救”。对象死亡需经历**两次标记**：

1. **第一次标记与筛选**
   - 若对象未覆盖finalize()方法，直接回收。
   - 若覆盖了finalize()，进入“缓刑”阶段，等待第二次标记。
2. **第二次标记**
   - 在finalize()中，对象可通过与引用链上的对象建立关联（例如赋给类变量）避免回收。
   - 若未成功自救，对象将被回收。
   - **注意**：finalize()仅执行一次，自救机会只有一次。

```java
public class OOMTest {

   public static void main(String[] args) {
      List<Object> list = new ArrayList<>();
      int i = 0;
      int j = 0;
      while (true) {
         list.add(new User(i++, UUID.randomUUID().toString()));
         new User(j--, UUID.randomUUID().toString());
      }
   }
}


//User类需要重写finalize方法
@Override
protected void finalize() throws Throwable {
    OOMTest.list.add(this);
    System.out.println("关闭资源，userid=" + id + "即将被回收");
}
```

### 🧹 如何判断一个类是无用的类

在JVM中，**方法区**主要负责回收**无用类**。一个类被认为是**无用类**，需同时满足以下三个条件：

1. **该类的所有对象实例已被回收** Java堆中不存在该类的任何实例，即该类的对象全部被垃圾回收器回收。
2. **加载该类的ClassLoader已被回收** 该类的类加载器（如自定义ClassLoader）本身必须已被垃圾回收，否则类无法被卸载。
3. **该类对应的java.lang.Class对象未被引用** 该类的java.lang.Class对象在任何地方都没有被引用，且无法通过反射（如Class.forName()或方法调用）访问该类的任何方法。
