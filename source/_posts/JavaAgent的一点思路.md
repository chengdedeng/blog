title: Java agent笔记
date: 2015-7-18 21:42:29
categories: 技术
tags: [Java]
----

>最近对Java的**Profiling**和**Debugging**非常感兴趣，特别是对线上问题的定位方案进行了较深的调研和分析。因为在**SOA**架构体系中线上问题经常在测试环境不能复现，所以问题的定位具有非常大的挑战。我将业界定位问题的工具和方案都大概的研究了一遍，不论是**[JProfiler](http://www.ej-technologies.com/products/jprofiler/overview.html)**、**[YourKit](http://www.yourkit.com/)**、**[BTrace]()**，还是更底层的**[Serviceability技术](http://openjdk.java.net/groups/hotspot/docs/Serviceability.html)**......都广泛用到了`Agent`技术。

<!--more-->

在开始之前有必要将HotSpot JVM的**Serviceability**做个简单的说明。**Serviceability**可以翻译为**服务性**，它可以理解为一种能力。这种能力表现在HotSpot虚拟机具备跟踪调试Java进程的能力，可以针对故障分析和隔离。当然作为一名Java程序员，绝大多数时候我们都不需要关注到此层面，但是当我们需要做一个更加深层次的调试工作时我们就绕不开Serviceability，具体的技术有如下几项。

1. **The Serviceability Agent(SA)**.Sun私有的一个组件，它可以帮助开发者调试Hotspot。
2. **jvmstat performance counters**.HotSpot通过Sun私有的共享内存机制实现的性能计数器，它是公开给外部给外部进程的。它们也称为**perfdata**。
3. **The Java Virtual Machine Tool Interface (JVM TI)**.它是一套标准的C接口，是[ JSR 163 - JavaTM Platform Profiling Architecture ](http://jcp.org/en/jsr/detail?id=163)参考实现。JVMTI提供了可用于 debug和profiler的接口；同时，在Java 5/6 中，虚拟机接口也增加了监听（Monitoring），线程分析（Thread analysis）以及覆盖率分析（Coverage Analysis）等功能。JVMTI并不一定在所有的Java虚拟机上都有实现，不同的虚拟机的实现也不尽相同。不过在一些主流的虚拟机中，比如 Sun 和 IBM，以及一些开源的如 Apache Harmony DRLVM 中，都提供了标准 JVMTI 实现。
4. **The Monitoring and Management interface**.Sun的私有实现，它能够实现对HotSpot的监控和管理。
5. **Dynamic Attach**.Sun的私有机制，它允许外部进程在HotSpot中启动一个agent线程，该agent可以将该HotSPot的信息发送给外部进程。
6. **DTrace**.全称Dynamic Tracing，也称为动态跟踪，是由Sun™开发的一个用来在生产和试验性生产系统上找出系统瓶颈的工具，可以对内核(kernel)和用户应用程序(user application)进行动态跟踪并且对系统运行不构成任何危险的技术。在任何情况下它都不是一个调试工具，而是一个实时系统分析寻找出性能及其他问题的工具。DTrace是个特别好的分析工具，带有大量的帮助诊断系统问题的特性。还可以使用预先写好的脚本利用它的功能。用户也可以通过使用 DTrace D语言创建他们自己定制的分析工具，以满足特定的需求。除Solaris系列以外，Dtrace已先后被移植到FreeBSD、NetBSD及Max OS X等操作系统上
7. **pstack support**.pstack是从Solaris发源的实用程序,可以查看core dump文件，调试进程，查看线程运行情况等等。现在，pstack工具也移植到了Linux系统中，比如Red Hat Linux系统、Ubuntu Linux系统等等。HotSpot允许pstack显示Java堆栈帧。


在这一系列的技术中，**SA**和**JVM TI**容易产生混淆，**SA**主要是给JVM开发人员调试虚拟机用的，**JVM TI**是为Java应用层开发者而设计的，它是标准API，主要用来开发各种调试和分析工具。也就是说**SA**更底层，如果不触碰JVM虚拟机底层开发，其实绝大多数时候是用不到的。当然也不是说，我们平常完全不会用到**SA**的功能。比如，**jstack -F/m**就会使用到**SA**，我们平常不加**-F/-m**的时候默认走的是**Instrumentation**的**attach**操作。**jmap**中的部分功能也是**SA**实现的，具体的没有详细研究。**Attach**是进程内操作，**SA**是进程外操作，这便是他们的区别之处。大家可能碰见过这样的例子，就是Java进程hung住了，使用jstack是拿不到threaddump的，这个时候只有加**-F**才可以拿到，并且拿到的dump数据跟不加**-F**拿到的数据是有差别的。那是因为attach进JVM之后，要拿到threaddump，需要JVM执行操作才行，但是这个时候JVM已经不能响应任何操作了，所以拿不到。但是进程外就不一样了，它是以一个观察者的方式去查看目标进程，所以能够拿到想要的信息。


虽然**Serviceability**很强大，但是本篇文章真正想讲的是**Instrumentation**特性。虽然**JDK5**中就已经有了`premain`，但是真正具有划时代意义的还是**JDK6**中的`agentmain`。因为当不容易重现的问题出现时，你可以不用重启应用，便可以对应用进行问题定位和调试，当然目前只有**HotSpot**和**JRockit**虚拟机支持。下面给一个通过环绕织入来实现获取方法执行时间的例子。

**1.**首先来一个注解，它是用在方法上，表示该方法我需要织入代码来获取执行时间。

```
package info.yangguo.demo.attch_api.test4;

public @interface Point {
}
```

**2.**来一个被织入的测试代码，为了方便测试，classpath动态加载**[javassist](http://www.javassist.org/)**，因为该类库在我后面类的transform中使用到了。

```java
package info.yangguo.demo.attch_api.test4;

import java.io.File;
import java.lang.reflect.Method;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.Random;

public class TestExample {
    private Random random = new Random();

    @Point
    public void doSleep() {
        try {
            Thread.sleep(15000);
            System.out.println("*");
        } catch (InterruptedException e) {
        }
    }

    @Point
    private void doTask() {
        try {
            Thread.sleep(random.nextInt(1500));
            System.out.println(".");
        } catch (InterruptedException e) {
        }
    }

    @Point
    public void doWork() {
        for (int i = 0; i < random.nextInt(12); i++) {
            doTask();
        }
    }

    public static void main(String[] args) {
        /**
         * 为了减少设置classpath的做法，具体的类库路径需要自己设置
         */
        ExtClasspathLoader.loadClasspath("/Users/yangguo/.gradle/caches/modules-2/files-2.1/org.javassist/javassist/3.20.0-GA/a9cbcdfb7e9f86fbc74d3afae65f2248bfbf82a0");
        TestExample test = new TestExample();
        while (true) {
            test.doWork();
            test.doSleep();
        }
    }


    /**
     * 根据properties中配置的路径把jar和配置文件加载到classpath中。
     *
     * @author guo.yang
     */
    private static class ExtClasspathLoader {
        /**
         * URLClassLoader的addURL方法
         */
        private static Method addURL = initAddMethod();

        private static URLClassLoader classloader = (URLClassLoader) ClassLoader.getSystemClassLoader();

        /**
         * 初始化addUrl 方法.
         *
         * @return 可访问addUrl方法的Method对象
         */
        private static Method initAddMethod() {
            try {
                Method add = URLClassLoader.class.getDeclaredMethod("addURL", new Class[]{URL.class});
                add.setAccessible(true);
                return add;
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }

        private static void loadClasspath(String filepath) {
            File file = new File(filepath);
            loopFiles(file);
        }

        private static void loadResourceDir(String filepath) {
            File file = new File(filepath);
            loopDirs(file);
        }

        /**
         * 循环遍历目录，找出所有的资源路径。
         *
         * @param file 当前遍历文件
         */
        private static void loopDirs(File file) {
            // 资源文件只加载路径
            if (file.isDirectory()) {
                addURL(file);
                File[] tmps = file.listFiles();
                for (File tmp : tmps) {
                    loopDirs(tmp);
                }
            }
        }

        /**
         * 循环遍历目录，找出所有的jar包。
         *
         * @param file 当前遍历文件
         */
        private static void loopFiles(File file) {
            if (file.isDirectory()) {
                File[] tmps = file.listFiles();
                for (File tmp : tmps) {
                    loopFiles(tmp);
                }
            } else {
                if (file.getAbsolutePath().endsWith(".jar") || file.getAbsolutePath().endsWith(".zip")) {
                    addURL(file);
                }
            }
        }

        /**
         * 通过filepath加载文件到classpath。
         *
         * @param file 文件路径
         * @return URL
         * @throws Exception 异常
         */
        private static void addURL(File file) {
            try {
                addURL.invoke(classloader, new Object[]{file.toURI().toURL()});
            } catch (Exception e) {
            }
        }
    }
}

```

**3.**当然少不了最重要的**Transform**，在这里面我使用**javassist**动态修改了字节码，实现了对目标方法的环绕织入。更多更强大的功能就得各位的具体需求了，比如我们还可以在此处加一个监控的report功能，从而实现实时监控。

```java
package info.yangguo.demo.attch_api.test4;

import javassist.ByteArrayClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class Transformer implements ClassFileTransformer {
    private ClassPool classPool;

    public Transformer() {
        classPool = new ClassPool();
        classPool.appendSystemPath();
        try {
            classPool.appendPathList(System.getProperty("java.class.path"));
        } catch (Exception e) {
            System.out.println("异常:" + e);
            throw new RuntimeException(e);
        }
    }

    public byte[] transform(ClassLoader loader, String fullyQualifiedClassName, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain, byte[] classBytes) throws IllegalClassFormatException {
        String className = fullyQualifiedClassName.replace("/", ".");

        classPool.appendClassPath(new ByteArrayClassPath(className, classBytes));

        try {
            CtClass ctClass = classPool.get(className);
            if (ctClass.isFrozen()) {
                System.out.println("Skip class " + className + ": is frozen");
                return null;
            }

            if (ctClass.isPrimitive() || ctClass.isArray() || ctClass.isAnnotation()
                    || ctClass.isEnum() || ctClass.isInterface()) {
                System.out.println("Skip class " + className + ": not a class");
                return null;
            }

            boolean isClassModified = false;
            for (CtMethod method : ctClass.getDeclaredMethods()) {
                // if method is annotated, add the code to measure the time
                if (method.hasAnnotation(Point.class)) {
                    if (method.getMethodInfo().getCodeAttribute() == null) {
                        System.out.println("Skip method " + method.getLongName());
                        continue;
                    }
                    System.out.println("Instrumenting method " + method.getLongName());
                    //此处实现了一个AOP的环绕织入，在我们常规的需求中，可以在新增监控的report逻辑，
                    //如果需要加载第三方的jar，可以通过动态加载classpath来实现

                    method.addLocalVariable("__metricStartTime", CtClass.longType);
                    method.insertBefore("__metricStartTime = System.currentTimeMillis();");
                    method.insertAfter("System.out.println( System.currentTimeMillis() - __metricStartTime);");
                    isClassModified = true;
                }
            }
            if (!isClassModified) {
                return null;
            }
            return ctClass.toBytecode();
        } catch (Exception e) {
            System.out.println("Skip class : " + className + ",Error Message:" + e.getMessage());
            return null;
        }
    }
}
```

**4.** **Agent**代码主要是设置`transformer`和`retransformClasses`。

```java
package info.yangguo.demo.attch_api.test4;


import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;
import java.lang.management.ManagementFactory;
import java.lang.management.RuntimeMXBean;

public class Agent {
    public static void agentmain(String agentArguments, Instrumentation instrumentation) throws UnmodifiableClassException {
        RuntimeMXBean runtimeMxBean = ManagementFactory.getRuntimeMXBean();
        System.out.println("Runtime:" + runtimeMxBean.getName() + " : " + runtimeMxBean.getInputArguments());
        System.out.println("Starting agent with arguments " + agentArguments);

        instrumentation.addTransformer(new Transformer(),true);
        instrumentation.retransformClasses(TestExample.class);
    }
}
```

**5.**为了方便加了一个自动打包的类，自动生成agent jar，主要是对**Transformer**、**Agent**打包，并设置**Manifest**。这样做的好处是可以通过代码自动化，方便二次开发成内部工具。

```java
package info.yangguo.demo.attch_api.test4;


import javax.tools.JavaCompiler;
import javax.tools.ToolProvider;
import java.io.*;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.jar.Attributes;
import java.util.jar.JarEntry;
import java.util.jar.JarOutputStream;
import java.util.jar.Manifest;

/**
 * Created by IntelliJ IDEA
 * User:杨果
 * Date:15/7/8
 * Time:上午10:26
 * <p/>
 * Description:
 */
public class JavacCompileAndJar {
    String javaSourcePath;
    String javaClassPath;
    String targetPath;
    String jarClassPath;
    HashMap<Object, Object> attributes;

    public JavacCompileAndJar(String javaSourcePath, String javaClassPath, String jarClassPath, String targetPath, HashMap attributes) {
        this.javaSourcePath = javaSourcePath;
        this.javaClassPath = javaClassPath;
        this.targetPath = targetPath;
        this.jarClassPath = jarClassPath;
        this.attributes = attributes;
    }

    public void complier() throws IOException {
        System.out.println("*** --> 开始编译java源代码...");

        File javaclassDir = new File(javaClassPath);
        if (!javaclassDir.exists()) {
            javaclassDir.mkdirs();
        }

        List<String> javaSourceList = new ArrayList<>();
        getFileList(new File(javaSourcePath), javaSourceList);

        JavaCompiler javaCompiler = ToolProvider.getSystemJavaCompiler();
        int result = -1;
        for (int i = 0; i < javaSourceList.size(); i++) {
            if (javaSourceList.get(i).endsWith("java")) {
                result = javaCompiler.run(null, null, null, "-d", javaClassPath, javaSourceList.get(i));
                System.out.println(result == 0 ? "*** 编译成功 : " + javaSourceList.get(i) : "### 编译失败 : " + javaSourceList.get(i));
            }
        }
        System.out.println("*** --> java源代码编译完成。");
    }

    private void getFileList(File file, List<String> fileList) throws IOException {
        if (file.isDirectory()) {
            File[] files = file.listFiles();
            for (int i = 0; i < files.length; i++) {
                if (files[i].isDirectory()) {
                    getFileList(files[i], fileList);
                } else {
                    fileList.add(files[i].getPath());
                }
            }
        }
    }

    public void generateJar() throws IOException {
        System.out.println("*** --> 开始生成jar包...");
        String targetDirPath = targetPath.substring(0, targetPath.lastIndexOf("/"));
        File targetDir = new File(targetDirPath);
        if (!targetDir.exists()) {
            targetDir.mkdirs();
        }

        Manifest manifest = new Manifest();
        manifest.getMainAttributes().put(Attributes.Name.MANIFEST_VERSION, "1.0");
        for (Map.Entry<Object, Object> attribute : attributes.entrySet()) {
            manifest.getMainAttributes().put(attribute.getKey(), attribute.getValue());
        }


        JarOutputStream target = new JarOutputStream(new FileOutputStream(targetPath), manifest);
        writeClassFile(new File(jarClassPath), target);
        target.close();
        System.out.println("*** --> jar包生成完毕。");
    }

    private void writeClassFile(File source, JarOutputStream target) throws IOException {
        BufferedInputStream in = null;
        try {
            if (source.isDirectory()) {
                String name = source.getPath().replace("\\", "/");
                if (!name.isEmpty()) {
                    if (!name.endsWith("/")) {
                        name += "/";
                    }
                    name = name.substring(jarClassPath.length());
                    JarEntry entry = new JarEntry(name);
                    entry.setTime(source.lastModified());
                    target.putNextEntry(entry);
                    target.closeEntry();
                }
                for (File nestedFile : source.listFiles())
                    writeClassFile(nestedFile, target);
                return;
            }

            String middleName = source.getPath().replace("\\", "/").substring(javaClassPath.length());
            JarEntry entry = new JarEntry(middleName);
            entry.setTime(source.lastModified());
            target.putNextEntry(entry);
            in = new BufferedInputStream(new FileInputStream(source));

            byte[] buffer = new byte[1024];
            while (true) {
                int count = in.read(buffer);
                if (count == -1)
                    break;
                target.write(buffer, 0, count);
            }
            target.closeEntry();
        } finally {
            if (in != null)
                in.close();
        }
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        String javaSourcePath = "/Users/yangguo/work/code/demo/src/main/java/info/yangguo/demo/attach_api/test4";
        String javaClassPath = "/Users/yangguo/Downloads/demo/classes/info/yangguo/demo/attach_api/test4";
        String jarClassPath = "/Users/yangguo/Downloads/demo/classes";
        String targetPath = "/Users/yangguo/Downloads/demo/target/libs/demo-1.0-SNAPSHOT-fat.jar";

        HashMap<Object, Object> attributes = new HashMap<>();
        Attributes.Name agentClass = new Attributes.Name("Agent-Class");
        attributes.put(agentClass, "info.yangguo.demo.attch_api.test4.Agent");
        Attributes.Name canRedineClasses = new Attributes.Name("Can-Redefine-Classes");
        attributes.put(canRedineClasses, "true");
        Attributes.Name canRetransformClasses = new Attributes.Name("Can-Retransform-Classes");
        attributes.put(canRetransformClasses, "true");

        JavacCompileAndJar javacCompileAndJar = new JavacCompileAndJar(javaSourcePath, javaClassPath, jarClassPath, targetPath, attributes);
        javacCompileAndJar.complier();
        javacCompileAndJar.generateJar();
    }

}
```


**6.**最后便是**Instrumentation**的的入口类，将`Agent`attach到前面的测试代码中去。代码中我将机器上所有的Java进程给遍历一遍，选择自己感兴趣的进程（TestExample）`attach`。

```java
package info.yangguo.demo.attch_api.test4;

/**
 * Created by IntelliJ IDEA
 * User:杨果
 * Date:15/6/29
 * Time:下午10:06
 * <p/>
 * Description:
 */

import com.sun.tools.attach.VirtualMachine;
import com.sun.tools.attach.VirtualMachineDescriptor;

import java.util.List;

public class AttachMain extends Thread {
    private final String jar;

    AttachMain(String attachJar) {
        jar = attachJar;
    }

    public void run() {
        VirtualMachine vm = null;
        List<VirtualMachineDescriptor> listAfter = null;
        try {
            int count = 0;
            while (true) {
                listAfter = VirtualMachine.list();
                for (VirtualMachineDescriptor vmd : listAfter) {
                    if (vmd.displayName().contains("TestExample")) {
                        // 如果 VM 有增加，我们就认为是被监控的 VM 启动了
                        // 这时，我们开始监控这个 VM
                        vm = VirtualMachine.attach(vmd);
                        break;
                    }
                }
                Thread.sleep(5000);
                count++;
                if (null != vm || count >= 10) {
                    break;
                }
            }
            vm.loadAgent(jar);
            vm.detach();
        } catch (Exception e) {
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new AttachMain("/Users/yangguo/Downloads/demo/target/libs/demo-1.0-SNAPSHOT-fat.jar").start();
        Thread.sleep(10000000);
    }
}
```
