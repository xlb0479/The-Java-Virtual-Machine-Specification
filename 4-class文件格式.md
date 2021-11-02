# 4 class文件格式

本章讲的是JVM的`class`文件格式。每个`class`文件仅包含一个类或接口或模块的定义。尽管类、接口或模块不需要将外部表达直接包含在文件中（比如类是类加载器生成的），但我们一般都是把任意有效的类、接口或模块表达方式认为是`class`文件格式要求的那样。

一个`class`文件包含的是一个8比特字节的流。16或32比特则是要读取两个或四个连续的8比特字节来组合起来。多字节数据一定是按照大端序来排，高位字节在前。本章定义的`u1`、`u2`和`u4`数据类型用来表达无符号的一字节、双字节或四字节的数据。

<pre>在JavaSE的API中，和class文件格式相关的有java.io.DataInput和java.io.DataOutput，以及java.io.DataInputStream和java.io.DataOutputStream。比如u1、u2、u4类型的值可以用java.io.DataInput接口中的readUnsignedByte、readUnsignedShort、readInt方法来获取。</pre>

在本章中，`class`文件格式我们要用类似C风格的伪结构来描述。为了避免跟类属性、类实例等其他描述发生混淆，`class`文件结构中的内容我们称为*项*。连续的项在`class`文件中是一个挨一个排起来的，不存在填充或对齐。

*表*，包含了零或多个长度可变的项，用于多种`class`文件结构。尽管我们会用C风格的数组语法来描述表中的项，但由于这些表实际上都是由长度可变的项组成起来的，所以并不能将表中的索引直接翻译成一个直接偏移量。

当我们把一种数据结构称为*数组*的时候，是说它包含了零或多个连续的长度固定的项，并且可以像数组一样进行索引。

本章中引用的ASCII字符应当被解释成对应的Unicode代码点。

## 4.1 ClassFile结构

一个`class`文件只包含一个`ClassFile`结构：

```c
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

`ClassFile`中的项：

#### `magic`

为`class`文件格式提供标识性的魔法数字；它的值为`0xCAFEBABE`。

#### `minor_version`、`major_version`

`class`文件的小版本和大版本号。二者结合起来才能确定这个`class`文件格式的版本号。如果大版本号是*M*，小版本号是*m*，记作*M.m*。

如果一个JVM的实现符合JavaSE*N*，那就必须要提供表4.1-A第四列“支持的大版本”中所对应的准确的`class`文件格式的大版本号。表中*A..B*意思就是*A*到*B*的大版本，且包含*A*和*B*。第三列“大版本”，表示每个JavaSE发行版对应的大版本号，也就是第一个能够接受`major_version`项所对应的发行版。

**表4.1-A. `class`文件格式的大版本们**

|-|-|-|-
|**JavaSE**|**发行时间**|**大版本**|**支持的大版本**|
|1.0.2|May 1996|45|45
|1.1|February 1997|45|45
|1.2|December 1998|46|45 .. 46
|1.3|May 2000|47|45 .. 47
|1.4|February 2002|48|45 .. 48
|5.0|September 2004|49|45 .. 49
|6|December 2006|50|45 .. 50
|7|July 2011|51|45 .. 51
|8|March 2014|52|45 .. 52
|9|September 2017|53|45 .. 53
|10|March 2018|54|45 .. 54
|11|September 2018|55|45 .. 55
|12|March 2019|56|45 .. 56
|13|September 2019|57|45 .. 57
|14|March 2020|58|45 .. 58
|15|September 2020|59|45 .. 59
|16|March 2021|60|45 .. 60
|17|September 2021|61|45 .. 61

如果一个`class`文件的`major_version`大于等于56，那它的`minor_version`必须是0或65545。

如果一个`class`文件的`major_version`在45到55之间，且包含45和55，那么它的`minor_version`可以是任意值。

<pre>JDK对<code>class</code>文件格式版本的这种支持，从历史上看也说得过去的。JDK1.0.2支持45.0到45.3。JDK1.1支持45.0到45.65535。JDK1.2支持的大版本是46，对应的小版本只支持0。后来的JDK延续了这种规则，每次引入一个新的大版本（47、48等等）但小版本只有一个0。最后，随着JavaSE12预览版（见下方）的出现，赋予了小版本号的标准角色，因此JDK12支持的大版本号56对应的小版本号是0<i>和</i>65535。后续JDK引入的版本就都是*N*.0和*N*.65535。比如JDK13支持57.0和57.65535。</pre>

JavaSE平台可以定义*预览版*。如果一个JVM实现符合JavaSE*N*（*N*≥12），那就必须要支持JavaSE*N*所有预览版的特性，但不包括其他发行版的预览特性。具体实现的时候，必须默认关闭所有预览特性，必须提供能够开启全部预览特性的方法，不能只提供开启一部分预览特性的方法。

如果一个`class`文件说它*依赖于JavaSE N（N≥12）的预览特性*，那么它的`major_version`对应JavaSE*N*（参考表4.1-A），`minor_version`为65535。

一个JVM实现符合JavaSE*N*（N≥12）必须：

- 一个依赖于JavaSE*N*预览特性的`class`文件只有当JavaSE*N*的预览特性开启后才能被加载。
- 一个依赖于其他JavaSE发行版预览特性的`class`文件绝对无法被加载。
- 一个不依赖于任何JavaSE发行版预览特性的`class`文件可以被加载，不论JavaSE*N*是否开启了预览特性。