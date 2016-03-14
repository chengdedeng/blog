title: Kryo2.22详解
date: 2014-5-19 21:3:30
categories: 技术
tags: [java]
------
>在2.22版本修正了许多之前反馈的问题，提高稳定性和性能。 它还引入了许多新的功能，最主要的是，它可以使用Unsafe的方式的直接读取和写入对象的内存。 这中绝对是最快的方式序列化方式，特别是在操作大型原始数组的时候。
Maven仓库中的JAR文件现在包含一个ObjectWeb的“阴影”版本ASM库，以避免与你项目中的ASM库出现兼容问题，从而导致你的应用程序粗线冲突。 这里不再需要一个单独的阴影jar了。

`由于本篇文章是之前翻译的，最后迁移到该博客。在迁移过程中可能出现某些地方出错，并且由于水平有限，如果发生上面的情况，请大家发邮件告诉我，谢谢！`

### 概述

**Kryo**是一种快速，高效的对象图序列化的Java框架。 该项目的目标是速度，效率，以及一个易于使用的API。该项目对那些在任何时间，对象需要被持久化，无论是文件，数据库，或通过网络的项目都是适用的。Kryo还可以自动实现深浅的复制/克隆。 就是直接复制一个对象对象到例外一个对象，而不是对象->字节->对象。
本文档是Kryo的V2版本。如果V1版本感兴趣，见[V1Documentation](https://github.com/EsotericSoftware/kryo/wiki/Documentation-for-Kryo-version-1.x)。
如果您打算使用Kryo进行网络通信，该[KryoNet](https://github.com/EsotericSoftware/kryonet)项目可能对你有用。

---
### 内容
   * Quickstart
   * IO
   * Unsafe-based IO
   * Serializers
   * Registration
   * Default serializers
   * FieldSerializer
   * KryoSerializable
   * Reading and writing
   * References
   * Object creation
   * Copying/cloning
   * Context
   * Compression and encryption
   * Chunked encoding
   * Compatibility
   * Interoperability
   * Stack size
   * Threading
   * Logging
   * Integration with Maven
   * Projects using Kryo

<!--more-->

 ---

### Quickstart快速入门
让我们从头开始来看看该库如何使用

``` java
Kryokryo=newKryo();
Outputoutput=newOutput(newFileOutputStream("file.bin"));
SomeClasssomeObject=...
kryo.writeObject(output,someObject);
output.close();
Inputinput=newInput(newFileInputStream("file.bin"));
SomeClasssomeObject=kryo.readObject(input,SomeClass.class);
input.close();
```

该Kryo类协调整个序列化。 输出和输入类处理缓冲字节和用可选的方式将其刷新到流。
本文档的其余部分详细介绍它是如何工作的和该库的高级用法。

---

### IO

输出类是OutputStream，它将数据写入一个字节数组缓冲区。 这个缓冲器中的内容可以直接获得和使用，如果一个字节数组是需要的。 如果输出是一个给定的OutputStream，当字节缓冲区满时会将其刷新到该输出流。 Output有很多方法能够高效地将原语和字符串转为字节。 它提供了类似DataOutputStream，BufferedOutputSteam，FilterOutputStream，和ByteArrayOutputStream类的功能。

因为Output buffers写入到OutputStream时，一定要调用flush()或close()当写入完成后，从而使缓冲字节写入底层流。

输入类是一个InputStream,它能够从一个字节数组缓冲区读取数据。这个buffer可以直接设置成缓冲区，如果需要从这个buffer中读取数据。 如果输入是一个给定的InputStream，这个buffer将会被填满直到buffer空间耗尽。Input有很多方法能够高效地读取原语和字符串通过字节的方式。  它提供了类似DataInputStream，BufferedInputStream，FilterInputStream，和ByteArrayInputStream类的功能。

如果读取或者写入不是一个字节数组，只需提供相应的InputStream和OutputStream就行了。

---

### Unsafe-based IO 不安全型的IO

Kryo提供了额外的IO类，它是基于sun.misc.Unsafe类公开的功能。 这些类是UnsafeInput，UnsafeOutput。对于来自Kryo的Input和Output，它们可以作为在那些支持sun.misc.Unsafe平台上的一种IO替换方案。

对于那些需要在物理内存和堆外内存上直接序列化和反序列的情况，这里有两个专用类UnsafeMemoryInputUnsafeMemoryOutput，它们可以替换平常的Input和Output类来实现该功能。
使用Unsafe-based IO可能导致在一个相当显著的性能提升，这主要取决于你的应用程序。 特别是那些需要序列化大型原始数组作为你的对象图的一部分的情况。

**关于使用Unsafe-based IO的免责申明 **

Unsafe-based IO不能100%的与Kryo的Input和Output 流兼容，当涉及到序列化数据的二进制格式时！
这意味着它们是由Unsafe-based产生的输出流只能由Unsafe-based输入流读取，而不是被通常的input stream读取。 这同样适用于相反的方向：通常的输出流中的数据不能被Unsafe-based IO读取。

只要序列化和反序列化两边都使用Unsafe IO或者在同样架构的处理器（更准确地说，如果字节顺序和内部表示整数和浮点类型是一样的）上使用，它们就是安全的。

---

### Serializers
Kryo是一个序列化框架。 它不会强求是否有一个schema或者关心数据到底是写入还是读取。 这是留给序列化器本身。 Serializers在默认情况下各种提供读取和写入数据的方式。 如果这些仍不符合特定的需要，它们可以部分或者全部被替换。 所提供的序列化器可以读写大多数对象，但是如果有必要，写一个新的序列化也是很容易。 序列化器的抽象类，定义了从对象到字节和字节到对象的方法。

``` java
public class ColorSerializer extends Serializer<Color>{
	public void write(Kryo kryo,Output output,Color object){
		output.writeInt(object.getRGB());
	}		
	public Color read(Kryo kryo,Input input,Class<T> type{
		return new Color(input.readInt(),true);
	}
}
```

序列化器具有可实现的两种方法。 write()将对象转化为字节。 read()创建该对象的一个新实例，并从输入中读取信息来填充它。

Kryo实例可用于写入和读取嵌套对象。 如果Kryo在read()中读取了嵌套对象，那么kryo.reference()必须首先与父对象一起调用，假如该嵌套对象引用了该父对象。假如嵌套对象没有引用父对象，Kryo不处理嵌套对象或者对象引用不存在，那么kryo.reference()是没有必要被调用的。 如果嵌套的对象使用相同的序列化器，序列化器必须是可重入的（什么是可重入的对象序列化? 就是说对象任意时候都可以进行序列化,包括保存和载入,而且不会影响当前程序运行和对象状态的异常，也就是说该序列化器不应该有特定的状态）。

代码不应该直接使用serializer，而是应该使用Kryo类的read和write方法来替代。 这样就能通过Kryo来编排序列化和处理新功能，如引用和空对象。

默认情况下，序列化程序并不需要处理null对象。 Kryo会根据需要通过一个byte来表示null或not null。如果一个serializer希望更高效和能够处理null，它可以调用Serializer#setAcceptsNull(true) 。 这个方法同样可以用来避免写入表示null的字节，当你知道一种类型的所有实例将永远不会是空的时候。

---

### Registration注册
当Kryo输出一个对象的实例时，它首先会输出该对象的Class的标识符。 默认情况下，类的完全限定名称会被写入，后面紧接的是对象的字节。 随后出现的是该对象图中的对象类型，它们通过可变长度的整数表示。 将类名写入序列化文件是有点效率低下的，因此类可以事先注册，从而避免写入类名：

``` java
	Kryo kryo = new Kryo();
    kryo.register(SomeClass.class);
    // ...
    Output output = ...
    SomeClass someObject = ...
    kryo.writeObject(output, someObject);
```

这里SomeClass的注册到了Kryo，该类与一个int型的ID相互关联。 当Kryo输出SomeClass实例时，它会输出这个整数ID。 这样比输出类的名称更高效，但要求被序列的类在序列化之前知道。 反序列化过程中，注册类id必须和序列化过程中的该类的id一致。 上面显示的注册方法分配下一个可用，最小的整数ID，这就意味着类的注册顺序非常重要。 该ID也可以明确指定，从而使顺序不重要：

``` java
	Kryo kryo = new Kryo();
    kryo.register(SomeClass.class, 0);
    kryo.register(AnotherClass.class, 1);
    kryo.register(YetAnotherClass.class, 2);
```


该ID很小时特别是正整数时，是非常高效的。 负整数ID不能高效的序列化。 -1和-2是保留的，不能使用。

已注册和未注册的类可以混合使用。 所有的基本数据类型，基础对象和String是默认注册的。

Kryo#setRegistrationRequired可以设置为true，那么任何未注册的类在序列化和反序列化都会抛出一个异常。 这样可以防止一个应用程序意外地使用类名的字符串。

如果使用未注册的类，缩短短包是一种优化方案。

---

### Default serializers 默认的序列化器
写完类标识符之后，Kryo会使用序列化器将对象转化成字节写入。 当一个类被在注册时可以指定一个序列化方式的的实例：

``` java
	Kryo kryo = new Kryo();
	kryo.register(SomeClass.class, new SomeSerializer());
	kryo.register(AnotherClass.class, new AnotherSerializer());
```
如果一个类没有注册或没有指定序列化器，会根据类的类型自动选择一个默认的序列化器。 下面的类含有一个默认的序列化器：

boolean|Boolean|byte|Byte|char
:---|:---|:---|:---|:---
Character|short|Short|int|Integer
long|Long|float|Float|double
Double|byte[]|String|BigInteger|BigDecimal
Collection|Date|Collections.emptyList|Collections.singleton|Map
StringBuilder|TreeMap|Collections.emptyMap|Collections.emptySet|KryoSerializable
StringBuffer|Class|Collections.singletonList|Collections.singletonMap|Currency
Calendar|TimeZone|Enum|EnumSet

可以为指定的类添加默认的序列化器：

``` java
	Kryo kryo = new Kryo();
    kryo.addDefaultSerializer(SomeClass.class, SomeSerializer.class);
    // ...
    Output output = ...
    SomeClass someObject = ...
    kryo.writeObject(output, someObject);
```
也可以使用DefaultSerializer注解来添加默认的序列化器：

``` java
	@DefaultSerializer(SomeClassSerializer.class)
    public class SomeClass {
       // ...
    }
```

如果一个类没有默认的序列化器与之相匹配，则默认情况下使用[FieldSerializer](https://github.com/EsotericSoftware/kryo#FieldSerializer)。 这也是可以改变：

``` java
	Kryo kryo = new Kryo();
	kryo.setDefaultSerializer(AnotherGenericSerializer.class);
```
一些序列化器允许提供额外的信息，从而减少字节的输出：

``` java
	Kryo kryo = new Kryo();
    FieldSerializer someClassSerializer = new FieldSerializer(kryo, SomeClass.class);
    CollectionSerializer listSerializer = new CollectionSerializer();
    listSerializer.setElementClass(String.class);
    listSerializer.setElementsCanBeNull(false);
    someClassSerializer.getField("list").setClass(LinkedList.class, listSerializer);
    kryo.register(SomeClass.class, someClassSerializer);
    // ...
    SomeClass someObject = ...
    someObject.list = new LinkedList();
    someObject.list.add("thishitis");
    someObject.list.add("bananas");
    kryo.writeObject(output, someObject);
```

在这个例子中，FieldSerializer序列化器将用于SomeClass的。 从FieldSerializer的配置可以看出，“list”字段将永远是一个LinkedList，并使用指定的CollectionSerializer。 该CollectionSerializer被配置成每个元素将是一个String，并没有任何元素都为null。 这使得序列化器更有效率。 在这种情况下，列表中的每个元素大概需要2〜3个字节。

---

### FieldSerializer

默认情况下，大多数的类最终使用FieldSerializer。 它本质上是手写的序列化方式，但是它会自动执行。FieldSerializer直接对对象的字段进行转换。 如果字段是public，protected，或默认的访问（包私有），使用最快速度生成字节码（见[ReflectASM](https://github.com/EsotericSoftware/reflectasm) ）。 对于私有字段，setAccessible和缓存反射状态，速度还是相当快的。
还提供了其他用途的序列化器，如的BeanSerializer，TaggedFieldSerializer和CompatibleFieldSerializer。 额外的序列化器，在GitHub上是一个单独的项目，[kryo-serializers](https://github.com/magro/kryo-serializers)。

---

### KryoSerializable
虽然FieldSerializer对于大多数类来说是理想的，但是有时你也想很方便的实现自己的序列化方式。 这可以通过实现KryoSerializable接口（类似于JDK的java.io.Externalizable接口）来完成。

``` java
	public class SomeClass implements KryoSerializable {
       // ...

       public void write (Kryo kryo, Output output) {
          // ...
       }

       public void read (Kryo kryo, Input input) {
          // ...
       }
    }
```

---

### Reading and writing

Kryo有三套方法集来读取和写入的对象。

如果对象的具体类不知道且对象可能为null：

``` java
	kryo.writeClassAndObject(output, object);
    // ...
    Object object = kryo.readClassAndObject(input);
    if (object instanceof SomeClass) {
       // ...
    }
```
如果类是已知的且该对象可能为null：

``` java
	kryo.writeObjectOrNull(output, someObject);
    // ...
    SomeClass someObject = kryo.readObjectOrNull(input, SomeClass.class);
```
如果类是已知的，该对象不能为空：

``` java
	kryo.writeObject(output, someObject);
    // ...
    SomeClass someObject = kryo.readObject(input, SomeClass.class);
```

---

### References
默认情况下，对象图中出现的对象会在首次已一个整数序号存储下来。 这样就能对多个引用指向同一个对象和循环图进行序列化。 这有一个小的开销，但是可以关闭以节省空间，如果你不需要保存引用：

``` java
	Kryo kryo = new Kryo();
    kryo.setReferences(false);
    // ...
```
当使用Kryo对嵌套对象进行序列化时， kryo.reference()必须在read() 中被调用。 见[Serializers](https://github.com/EsotericSoftware/kryo#serializers)获取更多信息。

---

### Object creation

对于特定类型的序列化器，会使用Java代码来创建该类型的新实例。 如FieldSerializer序列化器，它是通用的能够创建任何类的实例。 默认情况下，如果一个类有一个无参数的构造函数，然后它是通过ReflectASM来实现invoke或反射，否则将引发异常。 如果无参数的构造函数是私有的，会调用setAccessible方法通过反射来访问它。 如果这是可以接受的，私人的无参数的构造函数是一个很好的方式，让Kryo创建一个类的实例，而不会影响公共API。

当ReflectASM或反射不能用，Kryo可以被配置为使用InstantiatorStrategy来处理创建类的实例。 Objenesis 提供StdInstantiatorStrategy它使用JVM的特定API来创建一个类的实例，不需要调用任何构造函数都没有。 虽然这个作品多次在JVM上，一个无参的够着函数其实是更方便的。

``` java
	kryo.setInstantiatorStrategy(new StdInstantiatorStrategy());
```

注意，类必须被设计为以这种方式创建的。 如果一个类希望它的构造函数能够被调用，通过这一机制建立的，它可能是处于未初始化状态。
Objenesis也可以使用Java内置的序列化机制创建新的对象。 利用这一点，该类必须实现java.io.Serializable，并且无参的构造函数应该在超类中被调用。

``` java
		kryo.setInstantiatorStrategy(newSerializingInstantiatorStrategy());
```

您也可以编写自己的InstantiatorStrategy。
要自定义创建特定类型的方式，一个ObjectInstantiator可以设置。 这将覆盖ReflectASM，reflection，以及InstantiatorStrategy。

``` java
	Registrationregistration=kryo.register(SomeClass.class);
	registration.setObjectInstantiator(...);
```
另外，一些序列化器提供了可以被覆盖以自定义对象的创建方法。

``` java
	kryo.register(SomeClass.class, new FieldSerializer(kryo, SomeClass.class) {
       public Object create (Kryo kryo, Input input, Class type) {
          return new SomeClass("some constructor arguments", 1234);
       }
    });
```
---

### Copying/cloning
序列化库需要特定的知识在如何创建新的实例，获取和设置值，操作对象图等。这所有的一切都需要支持copying objects，因此Kryo自动支持深浅拷贝是顺理成章的事情。 注意，Kryo的拷贝不是序列化和反序列化，而是直接分配。

``` java
	Kryo kryo = new Kryo();
    SomeClass someObject = ...
    SomeClass copy1 = kryo.copy(someObject);
    SomeClass copy2 = kryo.copyShallow(someObject);
```

该序列化类有一个copy 方法来完成这项工作。这些方法能够被忽略，假如拷贝方法没有使用在实现应用程序中特定的序列化方式时。 Kryo的拷贝支持所有的序列化器。 多个引用指向同一个对象和循环引用会自动被框架理。

类似read()序列化方法， kryo.reference()必须先于Kryo复制子对象被调用。 通过[Serializers](https://github.com/EsotericSoftware/kryo#Serializers)获取更多信息。

类似KryoSerializable，可以实现KryoCopyable来实现自己的复制：

``` java
 public class SomeClass implements KryoCopyable<SomeClass> {
       // ...

       public SomeClass copy (Kryo kryo) {
          // Create new instance and copy values from this instance.
       }
    }
```

---

### Context
Kryo有两个context方法。 getContext()返回一个map，用于存储用户数据 。 因为Kryo实例提供给所有的序列化器使用，这个数据是现成的。 getGraphContext()类似，但它会在每个对象图形序列化和反序列化之后被清除。 这可以很容易地管理每对象图的状态。

---

### 压缩和加密
Kryo支持流，所以在序列化后字节上压缩和加密是非常简单的：

``` java
	OutputStream outputStream = new DeflaterOutputStream(new 	FileOutputStream("file.bin"));
    Output output = new Output(outputStream);
    Kryo kryo = new Kryo();
    kryo.writeObject(output, object);
    output.close();
```
如果需要,可以使用序列化器只压缩或加密对象图字节的一个子集。 例如，请参见DeflateSerializer或BlowfishSerializer。 这些序列化包装其他序列化器来编码和解码的字节。

---
### 分块编码

有时候先将数据的长度写入然后再将数据写入是有意义的。 如果不知道数据的长度，所有的数据将需要一个缓冲来确定其长度，以确定它的长度，则该长度可以被写入，然后才是数据。这种在防止流和潜在获取非常大的缓冲区方面不是很理想。
分块编码通过使用一个小缓冲解决这个问题。 当缓冲器满时，它的长度被写入，然后是数据。 这就是一个数据块。然后缓冲区被清除，这种情况持续下去，直到有没有更多的数据写入。 当一个长度为零的块表示所有块的结束。
Kryo提供了一些类，便于分块编码。 OutputChunked是用来写分块数据的。 它继承了Output，所以拥有所有方便的方法来写数据。 当OutputChunked缓冲区满时，它会将数据刷新到一个被包装过的OutputStream中。 该endChunks()方法用来标记一组块的结束。

``` java
	OutputStream outputStream = new FileOutputStream("file.bin");
    OutputChunked output = new OutputChunked(outputStream, 1024);
    // Write data to output...
    output.endChunks();
    // Write more data to output...
    output.endChunks();
    // Write even more data to output...
    output.close();
```
要读取分块的数据，可以使用InputChunked。 它继承了Input，因此它具有所有读取数据的方便方法。 当读取数据时，InputChunked会准确找出数据结束的数据块。 nextChunks()方法的作用是前进到下一组块，即使不是所有的数据要从当前组块集合中读出。

``` java
	InputStream outputStream = new FileInputStream("file.bin");
    InputChunked input = new InputChunked(inputStream, 1024);
    // Read data from first set of chunks...
    input.nextChunks();
    // Read data from second set of chunks...
    input.nextChunks();
    // Read data from third set of chunks...
    input.close();
```

---

### 兼容性
对于某些需求，尤其是长期存储的序列化字节，如何处理那些已经更改类的序列化是非常重要的。 这被称为前向和后向兼容性。 默认情况下，大多数用户类将使用FieldSerializer，它不支持添加，删除或更改类型，对于之前的序列化字节来说是无效字段。 这是可以接受的在许多情况下，例如在网络上发送数据。 如果有必要，可以使用一个简单的序列化器来替换：

``` java
	kryo.setDefaultSerializer(TaggedFieldSerializer.class);
```

TaggedFieldSerializer只序列化有一个@Tag注释的字段。 相比FieldSerializer不够灵活，因为它可以处理大多数类而无需注解，但允许TaggedFieldSerializer添加一个对之前序列化文件来说无效的字段。 如果一个字段被删除就会导致以前序列化的字节失效，所以字段应该用@Deprecated注解表示被删除。

或者，可以使用CompatibleFieldSerializer，在序列化对象时，会将第一次碰见的class用一个schema来表示，并写入序列化文件。 像FieldSerializer，它可以无需注解序列所有类。 字段可以添加或删除，无需关心以前的序列化字节，但不支持改变一个字段的类型。 这具有一些额外的开销，无论是在速度和空间，相对于FieldSerializer。

额外的具有向前和向后兼容序列化可能会被开发，例如使用一个外部的，手写模式的序列化程序。

---

### 互通性
Kryo序列化器默认假设Java将作为反序列化，所以他们并没有明确定义写入格式。 序列化器可以编写一个标准化的格式使别的语言更容易阅读,但这不是默认提供。

---

### 堆大小
Kryo能够直接访问堆当序列化嵌套对象时。 Kryo确实减少堆栈调用，但是对于极深的对象图，堆溢出可能发生。 这是大多数序列化库，包括Java内置的序列化都面临该问题。 堆大小的调节可以使用-Xss ，但是请注意，这是针对所有线程。 堆太大并且线程很多会导致JVM使用大量内存。

---

### 线程
Kryo不是线程安全的。 每个线程都应该有自己的Kryo，输入和输出的实例。 此外，byte []的输入用途可以被修改，然后反序列化过程返回到其原始状态，因此相同的字节不应该在多线程中并发使用。

---

### 日志
Kryo利用了低开销，轻量级的MinLog logging library.。 记录级别可以通过下列方法之一来设置：

``` java
	Log.ERROR();
    Log.WARN();
    Log.INFO();
    Log.DEBUG();
    Log.TRACE();
```

Kryo不会输出INFO （默认值）及以上水平的日志。 DEBUG方便开发过程中使用。 TRACE好调试一个具体问题时使用，但这种级别一般会有太多的信息要输出。

MinLog支持一个固定的日志级别，这会导致javac来在编译时去掉低于这一水平的日志记录语句。 在Kryo派发zip文件中，“debug”的JAR中被开启。 “production”JAR文件使用一个固定的日志记录级别NONE ，这意味着所有的日志代码已被删除。

---

### 使用Kryo的项目

* [KryoNet](http://code.google.com/p/kryonet/) (NIO networking)
* [Twitter's Scalding](https://github.com/twitter/scalding) (Scala API for Cascading)
* [Twitter's Chill](https://github.com/twitter/chill) (Kryo serializers for Scala)
* [Apache Hive](http://hive.apache.org/) (query plan serialization)
* [DataNucleus](https://github.com/datanucleus/type-converter-kryo) (JDO/JPA persistence framework)
* [CloudPelican](http://www.cloudpelican.com/)
* [Yahoo's S4](http://www.s4.io/) (distributed stream computing)
* [Storm](https://github.com/nathanmarz/storm/wiki/Serialization) (distributed realtime computation system, in turn used by many others)
* [Cascalog](https://github.com/nathanmarz/cascalog) (Clojure/Java data processing and querying details)
* [memcached-session-manager](https://code.google.com/p/memcached-session-manager/) (Tomcat high-availability sessions)
* [Mobility-RPC](http://code.google.com/p/mobility-rpc/) (RPC enabling distributed applications)
* [akka-kryo-serialization](https://github.com/romix/akka-kryo-serialization) (Kryo serializers for Akka)
* [Groupon](https://code.google.com/p/kryo/issues/detail?id=67)
* [Jive](http://www.jivesoftware.com/jivespace/blogs/jivespace/2010/07/29/the-jive-sbs-cache-redesign-part-3)
* [DestroyAllHumans](https://code.google.com/p/destroyallhumans/) (controls a [robot](http://www.youtube.com/watch?v=ZeZ3R38d3Cg)!)
* [kryo-serializers](https://github.com/magro/kryo-serializers) (additional serializers)
