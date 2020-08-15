title: BTrace笔记
date: 2015-6-8 19:31:29
categories: 技术
tags: [Thread]
----

>BTrace作为线上问题定位神器，它在侵入、安全、资源占用等方面表现的都非常出色。本文记录了作者平时工作中使用Btrace的场景，以供大家参考。


### BTrace限制
关于BTrace的安装配置使用，网络上有太多的教程了，本文就不再重复造轮子了，但是还是非常有必要再次重申BTrace的限制。因为不正当的使用Btrace可能根本拿不到自己想要的结果，甚至导致JVM崩溃，建议脚本使用之前先经过验证，然后再使用到线上环境。

使用Btrace时，需要确保追踪的行动是只读的（即：追踪行动不能修改程序的状态）和有限的（即：追踪行动需要在有限的时间内终止），一个追踪行为需要满足如下的限制：

* 不能创建新的对象
* 不能创建新的数组
* 不能抛出异常
* 不能捕获异常
* 不能对实例和静态方法随意调用，只有**com.sun.btrace.BTraceUtils**中的public static方法或者当前脚本中申明的方法，可以被BTrace程序调用。
* (1.2之前)不能存在实例字段和方法。BTrace类只允许静态公共void返回方法，并且所有字段必须是静态的。
* 不能将目标程序中的类和对象中的静态或者实例字段指派给BTrace程序。但是，可以将BTrace类自己的静态字段（“trace state”是可以改变的）指派给自己。因为引用的传递可能导致目标程序别修改，而BTrace自身的跟踪字段只有BTrace自己在使用，所以不怕修改。
* 不能有外部，内部，嵌套或者本地的类。
* 不能有**synchronized**块和方法。
* 不能有循环（for，while，do.....while）。
* 不能继承抽象类（父类只能为java.lang.Object）。
* 不能实现接口。
* 不能有断言语句。
* 不能使用**class**保留字。

<!--more-->

以上的限制可以通过通过**unsafe**模式绕过。追踪脚本和引擎都必须设置为**unsafe**模式。脚本需要使用注解为**@BTrace(unsafe = true)**，需要修改BTrace安装目录下bin中btrace脚本将**-Dcom.sun.btrace.unsafe=false**改为**-Dcom.sun.btrace.unsafe=true**。

注：关于**unsafe**的使用，如果你的程序一旦被btrace追踪过，那么unsafe的设置会一直伴随该进程的整个生命周期。如果你修改了**unsafe**的设置，只有通过重启目标进程，才能获得想要的结果。所以该用法不是很好使用，如果你的应用不能随便重启，那么你在第一次使用btrace最终目标进程之前，先想好到底使用那种模式来启动引擎。

-------

### 性能分析
大家经常会发现自己某一个服务变慢，但是由于这个服务背后由很多的业务逻辑或者方法块构成，这个时候就不好定位到底慢在哪个地方。**Profiling**就可以很好的解决该问题，我们只需要大概的定位问题可能存在的地方，通过包路径的模糊匹配，就可以很快找到问题所在。

```java
@BTrace
class Profiling {
    @Property
    Profiler profiler = BTraceUtils.Profiling.newProfiler();

    @OnMethod(clazz = "/info\\.yangguo\\..*/", method = "/.*/")
    void entry(@ProbeMethodName(fqn = true) String probeMethod) {
        BTraceUtils.Profiling.recordEntry(profiler, probeMethod);
    }

    @OnMethod(clazz = "/info\\.yangguo\\..*/", method = "/.*/", location = @Location(value = Kind.RETURN))
    void exit(@ProbeMethodName(fqn = true) String probeMethod, @Duration long duration) {
        BTraceUtils.Profiling.recordExit(profiler, probeMethod, duration);
    }

    @OnTimer(5000)
    void timer() {
        BTraceUtils.Profiling.printSnapshot("Performance profile", profiler);
    }
}
```

当然如果你怀疑一个类的某一个方法，也可以采用精准定位的方式来进行追踪。

```java
@BTrace
public class HttpClientSystemDefaultDnsResolverTracer {
    @TLS
    static long beginTime;

    @OnMethod(
            clazz = "org.apache.http.impl.conn.SystemDefaultDnsResolver",
            method = "resolve"
    )
    public static void traceGetByNameBegin() {
        beginTime = timeMillis();
    }

    @OnMethod(
            clazz = "org.apache.http.impl.conn.SystemDefaultDnsResolver",
            method = "resolve",
            location = @Location(Kind.RETURN)
    )
    public static void traceGetByNameEnd() {
        println(strcat(strcat("org.apache.http.impl.conn.SystemDefaultDnsResolver.resolve 耗时:", str(timeMillis() - beginTime)), "ms"));
    }
}
```

如果你只想定位执行时间超过某个阀值的函数，也可以使用**@Duration**来解决。

```java
@BTrace
public class DurationTracer {
    @OnMethod(
            clazz = "/info\\.yangguo\\.demo\\..*/",
            method = "/.*/",
            location = @Location(Kind.RETURN)
    )
    public static void trace(@ProbeClassName String pcn,
                             @ProbeMethodName String pmn,
                             @Duration long duration) {
        //duration的单位是纳秒
        if (duration > 1000 * 1000 * 3) {
            print(Strings.strcat(Strings.strcat(pcn, "."), pmn));
            print(" 耗时:");
            print(duration);
            println("纳秒,堆栈信息如下");
            jstack();
        }
    }
}
```

-----

### 异常分析
有时候，开发人员对异常的处理不合理，导致某些重要异常别人为吃掉，并且没有任何日志或者日志不详细，导致问题定位困难，那么我们可以使用下面的Tracer来处理。

```java
@BTrace
public class OnThrow {
    // store current exception in a thread local
// variable (@TLS annotation). Note that we can't
// store it in a global variable!
    @TLS
    static Throwable currentException;

    // introduce probe into every constructor of java.lang.Throwable
// class and store "this" in the thread local variable.
    @OnMethod(
            clazz = "java.lang.Throwable",
            method = "<init>"
    )
    public static void onthrow(@Self Throwable self) {
        currentException = self;
    }

    @OnMethod(
            clazz = "java.lang.Throwable",
            method = "<init>"
    )
    public static void onthrow1(@Self Throwable self, String s) {
        currentException = self;
    }

    @OnMethod(
            clazz = "java.lang.Throwable",
            method = "<init>"
    )
    public static void onthrow1(@Self Throwable self, String s, Throwable cause) {
        currentException = self;
    }

    @OnMethod(
            clazz = "java.lang.Throwable",
            method = "<init>"
    )
    public static void onthrow2(@Self Throwable self, Throwable cause) {
        currentException = self;
    }

    // when any constructor of java.lang.Throwable returns
// print the currentException's stack trace.
    @OnMethod(
            clazz = "java.lang.Throwable",
            method = "<init>",
            location = @Location(Kind.RETURN)
    )
    public static void onthrowreturn() {
        if (currentException != null) {
            Threads.jstack(currentException);
            println("=====================");
            currentException = null;
        }
    }
}

```

-----

### 参数及返回结果

```java
@BTrace
public class MethodArgsAndReturnTracer {
    @OnMethod(
            clazz = "info.yangguo.demo.btrace.Adder",
            method = "execute",
            location = @Location(Kind.RETURN)
    )
    public static void traceExecute(@Self Adder instance, int arg1,int arg2, @Return int result) {
        println("Adder.execute");
        println(strcat("arg1 is:", str(arg1)));
        println(strcat("arg2 is:", str(arg2)));
        println(strcat("maxResult is:", str(get(field("info.yangguo.demo.btrace.Adder", "maxResult"), instance))));
        println(strcat("return value is:", str(result)));
    }
}
```
--------

### jdbc问题追踪

```java
@BTrace
public class JdbcQueries {
    private static Map<Statement, String> preparedStatementDescriptions = Collections.newWeakMap();
    private static Aggregation histogram = Aggregations.newAggregation(AggregationFunction.QUANTIZE);
    private static Aggregation average = Aggregations.newAggregation(AggregationFunction.AVERAGE);
    private static Aggregation max = Aggregations.newAggregation(AggregationFunction.MAXIMUM);
    private static Aggregation min = Aggregations.newAggregation(AggregationFunction.MINIMUM);
    private static Aggregation sum = Aggregations.newAggregation(AggregationFunction.SUM);
    private static Aggregation count = Aggregations.newAggregation(AggregationFunction.COUNT);
    private static Aggregation globalCount = Aggregations.newAggregation(AggregationFunction.COUNT);
    @TLS
    private static String preparingStatement;
    @TLS
    private static String executingStatement;
    /**
     * If "--stack" is passed on command line, print the Java stack trace of the JDBC statement.
     * <p/>
     * Otherwise we print the SQL.
     */
    private static boolean useStackTrace = Sys.$(2) != null && Strings.strcmp("--stack", Sys.$(2)) == 0;
    // The first couple of probes capture whenever prepared statement and callable statements are
    // instantiated, in order to let us track what SQL they contain.

    /**
     * Capture SQL used to create prepared statements.
     *
     * @param args the list of method parameters. args[1] is the SQL.
     */
    @OnMethod(clazz = "+java.sql.Connection", method = "/prepare.*/")
    public static void onPrepare(AnyType[] args) {
        preparingStatement = useStackTrace ? Threads.jstackStr() : str(args[0]);
    }

    /**
     * Cache SQL associated with a prepared statement.
     *
     * @param preparedStatement the return value from the prepareXxx() method.
     */
    @OnMethod(clazz = "+java.sql.Connection", method = "/prepare.*/", location = @Location(Kind.RETURN))
    public static void onPrepareReturn(@Return Statement preparedStatement) {
        if (preparingStatement != null) {
            print("P"); // Debug Prepared
            Collections.put(preparedStatementDescriptions, preparedStatement, preparingStatement);
            preparingStatement = null;
        }
    }

    // The next couple of probes intercept the execution of a statement. If it execute with no-args,
    // then it must be a prepared statement or callable statement. Get the SQL from the probes up above.
    // Otherwise the SQL is in the first argument.
    @OnMethod(clazz = "+java.sql.Statement", method = "/execute.*/")
    public static void onExecute(@Self Object currentStatement, AnyType[] args) {
        if (args.length == 0) {
    // No SQL argument; lookup the SQL from the prepared statement
            executingStatement = Collections.get(preparedStatementDescriptions, currentStatement);
        } else {
    // Direct SQL in the first argument
            executingStatement = useStackTrace ? Threads.jstackStr() : str(args[0]);
        }
    }

    @OnMethod(clazz = "+java.sql.Statement", method = "/execute.*/", location = @Location(Kind.RETURN))
    public static void onExecuteReturn(@Duration long durationL) {
        if (executingStatement == null) {
            return;
        }

        print("X"); // Debug Executed

        AggregationKey key = Aggregations.newAggregationKey(executingStatement);
        int duration = (int) durationL / 1000;

        Aggregations.addToAggregation(histogram, key, duration);
        Aggregations.addToAggregation(average, key, duration);
        Aggregations.addToAggregation(max, key, duration);
        Aggregations.addToAggregation(min, key, duration);
        Aggregations.addToAggregation(sum, key, duration);
        Aggregations.addToAggregation(count, key, duration);
        Aggregations.addToAggregation(globalCount, duration);

        executingStatement = null;
    }


    @OnEvent
    public static void onEvent() {
    // Top 10 queries only
        Aggregations.truncateAggregation(histogram, 10);
        println("---------------------------------------------");
        Aggregations.printAggregation("Count", count);
        Aggregations.printAggregation("Min", min);
        Aggregations.printAggregation("Max", max);
        Aggregations.printAggregation("Average", average);
        Aggregations.printAggregation("Sum", sum);
        Aggregations.printAggregation("Histogram", histogram);
        Aggregations.printAggregation("Global Count", globalCount);
        println("---------------------------------------------");
    }
}

```

------

### 死锁排查
很多时候，我们怀疑程序是否有死锁，可以用下面的脚本扫描下，非常简单方便。

```java
@BTrace
public class Deadlock {
    @OnTimer(4000)
    public static void print() {
        deadlocks();
    }
}
```


我只是将我工作中经常用到的一些列举出来了，还有很多没有列举，可以详见[链接](https://kenai.com/projects/btrace/pages/UserGuide)。



参考文章：
[Btrace一些你不知道的事](http://agapple.iteye.com/blog/1005918)
