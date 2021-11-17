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

`magic`

&emsp;&emsp;为`class`文件格式提供标识性的魔法数字；它的值为`0xCAFEBABE`。

`minor_version`、`major_version`

&emsp;&emsp;`class`文件的小版本和大版本号。二者结合起来才能确定这个`class`文件格式的版本号。如果大版本号是*M*，小版本号是*m*，记作*M.m*。

&emsp;&emsp;如果一个JVM的实现符合JavaSE*N*，那就必须要提供表4.1-A第四列“支持的大版本”中所对应的准确的`class`文件格式的大版本号。表中*A..B*意思就是*A*到*B*的大版本，且包含*A*和*B*。第三列“大版本”，表示每个JavaSE发行版对应的大版本号，也就是第一个能够接受`major_version`项所对应的发行版。

**表4.1-A. `class`文件格式的大版本们**

|**JavaSE**|**发行时间**|**大版本**|**支持的大版本**|
|-|-|-|-|
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

&emsp;&emsp;如果一个`class`文件的`major_version`大于等于56，那它的`minor_version`必须是0或65545。

&emsp;&emsp;如果一个`class`文件的`major_version`在45到55之间，且包含45和55，那么它的`minor_version`可以是任意值。

<pre>JDK对<code>class</code>文件格式版本的这种支持，从历史上看也说得过去的。JDK1.0.2支持45.0到45.3。JDK1.1支持45.0到45.65535。JDK1.2支持的大版本是46，对应的小版本只支持0。后来的JDK延续了这种规则，每次引入一个新的大版本（47、48等等）但小版本只有一个0。最后，随着JavaSE12预览版（见下方）的出现，赋予了小版本号的标准角色，因此JDK12支持的大版本号56对应的小版本号是0<i>和</i>65535。后续JDK引入的版本就都是<i>N</i>.0和<i>N</i>.65535。比如JDK13支持57.0和57.65535。</pre>

&emsp;&emsp;JavaSE平台可以定义*预览版*。如果一个JVM实现符合JavaSE*N*（*N*≥12），那就必须要支持JavaSE*N*所有预览版的特性，但不包括其他发行版的预览特性。具体实现的时候，必须默认关闭所有预览特性，必须提供能够开启全部预览特性的方法，不能只提供开启一部分预览特性的方法。

&emsp;&emsp;如果一个`class`文件说它*依赖于JavaSE N（N≥12）的预览特性*，那么它的`major_version`对应JavaSE*N*（参考表4.1-A），`minor_version`为65535。

&emsp;&emsp;一个JVM实现符合JavaSE*N*（N≥12）必须：

- 一个依赖于JavaSE*N*预览特性的`class`文件只有当JavaSE*N*的预览特性开启后才能被加载。
- 一个依赖于其他JavaSE发行版预览特性的`class`文件绝对无法被加载。
- 一个不依赖于任何JavaSE发行版预览特性的`class`文件可以被加载，不论JavaSE*N*是否开启了预览特性。

`constant_pool_count`

&emsp;&emsp;`constant_pool_count`的值等于`constant_pool`表中记录数量加一。在`constant_pool`表中的有效索引是大于零且小于`constant_pool_count`的，除了§4.4.5提到的特殊的`long`和`double`类型的常量。

`constant_pool[]`

&emsp;&emsp;`constant_pool`是一个结构体表（§4.4），用来表达各种不同的字符串常量、类和接口名、属性名，以及其他`ClassFile`结构及其子结构中引用的常量。每一条`constant_pool`表中的记录格式由该记录的第一个“标签”字节确定。

&emsp;&emsp;`constant_pool`表中的所有是从1到`constant_pool_count`-1。

`access_flags`

&emsp;&emsp;`access_flags`的值是访问权限以及一些类或接口属性信息的掩码。每个标记的含义见表4.1-B。

**表4.1-B. 类访问和属性修饰符**

|**标记名**|**值**|**解释**|
|-|-|-|
|ACC_PUBLIC|0x0001|`public`声明；可以从包外访问。
|ACC_FINAL|0x0010|`final`声明；不让有子类。
|ACC_SUPER|0x0020|被*invokespecial*指令调用时对父类方法要特殊对待。
|ACC_INTERFACE|0x0200|是个接口，不是类。
|ACC_ABSTRACT|0x0400|`abstract`声明；不能被实例化。
|ACC_SYNTHETIC|0x1000|合成声明；不直接存在于源代码中。
|ACC_ANNOTATION|0x2000|声明注解接口。
|ACC_ENUM|0x4000|`enum`类声明
|ACC_MODULE|0x8000|我是模块，不是类或接口。

&emsp;&emsp; `ACC_MODULE`说明这个`class`文件是定义模块的，不是类或者接口。如果设置了`ACC_MODULE`标记，就会为`class`文件添加本节末尾提到的特殊规则。如果没有设置`ACC_MODULE`标记，则为`class`文件添加下面这段规则。

 &emsp;&emsp;`ACC_INTERFACE`标记用于区分是否是接口。如果没有设置`ACC_INTERFACE`标记，说明该`class`文件定义的是类，而不是接口或模块。

 &emsp;&emsp;如果设置了`ACC_INTERFACE`标记，则不能设置`ACC_ABSTRACT`标记，也不能设置`ACC_FINAL`、`ACC_SUPER`、`ACC_ENUM`以及`ACC_MODULE`标记。

 &emsp;&emsp;如果没设置`ACC_INTERFACE`标记，表4.1-B中除了`ACC_ANNOTATION`和`ACC_MODULE`以外的所有标记都可以设置。但这种`class`文件不能同时设置`ACC_FINAL`和`ACC_ABSTRACT`（JLS §8.1.1.2）。

 &emsp;&emsp;如果类或接口中设置了`ACC_SUPER`，它用来指示`invokespecial`指令（§invokespecial）的具体语义。JVM指令集的编译器要负责设置`ACC_SUPER`标记。从JavaSE8开始，JVM会认为所有的`class`文件都设置的`ACC_SUPER`标记，不管`class`文件中该标记的真实值是什么，也不管`class`文件的版本是什么。

 <pre><code>ACC_SUPER</code>标记是为了跟早期的Java语言编译器做向后兼容。在JDK1.0.2之前，编译器生成的`access_flags`中并没有`ACC_SUPER`的含义，即便设置了也会被Oracle的JVM实现所忽略。</pre>

 &emsp;&emsp;`ACC_SYNTHETIC`标记表示这个类或接口是编译器生成的，源代码中看不到。

 &emsp;&emsp;注解接口（JLS §9.6）必须要设置`ACC_ANNOTATION`。如果设置了，那么必须要同时设置`ACC_INTERFACE`。

 &emsp;&emsp;`ACC_ENUM`标记指明该类或其父类被声明为一个枚举类（JLS §8.9）。

 &emsp;&emsp;表4.1-B中没有说明的比特位留给以后需要的时候再用。在生成的`class`文件中它们应该都设置成零，并且JVM实现也应该忽略这些比特位。

 `this_class`

 &emsp;&emsp;它的值必须是`constant_pool`表中的有效索引。该索引对应的记录必须是一个`COSNTANT_Class_info`结构体（§4.4.1），用于表达该`class`文件所定义的类或接口。

 `super_class`

 &emsp;&emsp;对于类来说，`super_class`的值要么是零要么是`constant_pool`表中的有效索引。如果非零，`constant_pool`表中该索引对应的记录必须是一个`CONSTANT_Class_info`结构体，用于表达该`class`文件定义的类的直接父类。不管是它的直接父类还是更高层的父类，它们的`ClassFile`中的`access_flags`中都不能设置`ACC_FINAL`标记。

 &emsp;&emsp;如果该值是零，那么该`class`文件就必须表达的是`Object`类，只有它没有直接父类。

 &emsp;&emsp;对于接口来说，该标记的值必须总是`constant_pool`表中的有效索引。`constant_pool`表中该索引对应的记录必须是一个`CONSTANT_Class_info`结构体，用来表达`Object`类。

 `interfaces_count`

 &emsp;&emsp;它的值用来表示该类或接口的直接父接口的数量。

 `interfaces[]`

 &emsp;&emsp;`interfaces`数组中的每个值必须是`constant_pool`表中的一个有效索引。每个`interfaces[i]`，`0 ≤ i < interfaces_count`，对应的`constant_pool`表中的记录必须是一个`CONSTANT_Class_info`结构体，用来表达该类或接口的一个直接父接口，顺序按照它们在该类型的源代码中声明的顺序排列。

 `fields_count`

 &emsp;&emsp;这个值就是告诉你`fields`表中有多少个`field_info`结构体。`field_info`结构体代表了所有的属性，包括当前类或接口类型声明的类属性和实例属性。

 `fields[]`

 &emsp;&emsp;`fields`表中的每个值都得是一个`field_info`结构体（§4.5），完整的描述了当前类或接口中的一个属性。`fields`表中仅包含当前类或接口声明的属性。不包含父类或者父接口的。

 `methods_count`

&emsp;&emsp;这个值就是`methods`表中有多少个`method_info`结构体。

 `methods[]`

&emsp;&emsp;这个表里面的每个值必须是一个`method_info`结构体（§4.6），完整描述了当前类或接口中的一个方法。如果`method_info`结构体中的`access_flags`没有设置`ACC_NATIVE`或`ACC_ABSTRACT`，也同样提供实现该方法的JVM指令。（最后这半句没搞明白）

&emsp;&emsp;`method_info`结构体用于表达当前类或接口中声明的所有方法，包括实例方法、类方法、实例初始化方法（§2.9.1），以及任何的类或接口的初始化方法（§2.9.2）。不包括父类或者父接口的。

 `attributes_count`

&emsp;&emsp;这个值代表`attributes`表中有多少个属性（attribute）。

 `attributes[]`

&emsp;&emsp;表中的每个值都得是一个`attribute_info`结构体（§4.7）。

&emsp;&emsp;其中包含的属性见表4.7-C。

&emsp;&emsp;预定义属性相关规则见§4.7。

&emsp;&emsp;非预定义属性相关规则见§4.7.1。

如果`access_flags`中设置了`ACC_MODULE`标记，那么就不会设置其他标记了，而且会针对该`ClassFile`结构体中的其他部分应用以下规则：

- `major_version`、`minor_version`：≥53.0（即至少得是JavaSE9）
- `this_class`：`module_info`
- `super_class`、`interfaces_count`、`fields_count`、`methods_count`：零
- `attributes`：必须要有个一个`Module`属性。除了`Module`、`ModulePackge`、`ModuleMainClass`、`InnerClasses`、`SourceFile`、`SourceDebugExtension`、`RuntimeVisibleAnnotations`、`RuntimeInvisibleAnnotations`以外，不能有其他预定义属性（§4.7）。

## 4.2 名称

### 4.2.1 二进制类与接口名

`class`文件结构中出现的类和接口名总是以完全限定的形式出现，也就是*二进制名*（JLS §13.1）。这些名字都是用`CONSTANT_Utf8_info`结构体（§4.4.7）来表达的，因此在不受其他约束的情况下可以从整个Unicode码空间取值。类和接口名会用于`CONSTANT_NameAndType_Info`结构体（§4.4.6），作为描述符（§4.3）的一部分，还会用于所有的`CONSTANT_Class_info`结构体（§4.4.1）。

出于历史原因，`class`文件结构中的二进制名语法跟JLS §13.1文档中的二进制名语法并不相同。通常用ASCII的点符号（.）对二进制名中的标识符进行分割，但这种内部形式上，会换成ASCII的斜杠（/）。标识符本身必须得是未限定名（§4.2.2）。

<pre>比如，Thread类的普通的二进制名是java.lang.Thread。在class文件格式的描述符中使用的内部形式就不一样，使用CONSTANT_Utf8_info结构来实现对Thread类名称的引用，就要表达成java/lang/Thread。</pre>

### 4.2.2 未限定名

方法、属性、本地变量、形参，这些都是用*未限定名*保存的。一个未限定名必须至少包含一个Unicode代码点，且不能包含任何ASCII字符.;[/（也就是点、分号、左方括号和斜杠）。

方法名的限制更多，除了特殊的`<init>`和`<clinit>`（§2.9）方法名，其他方法名不能包含ASCII字符<或>（左右尖括号）。

<pre>属性名和接口方法名可以是&lt;init&gt;或&lt;clinit&gt;，但是没有哪种方法调用指令可以引用&lt;clinit&gt;，且只有<i>invokespecial</i>指令（§invokespecial）可以引用&lt;init&gt;。</pre>

### 4.2.3 模块与包名

`Module`属性引用的模块名是保存在常量池的`COSTANT_Module_info`结构体中的（§4.4.11）。一个`CONSTANT_Module_info`结构体内封装了一个用来表示模块名的`CONSTANT_Utf8_info`结构体。模块名并不像类或接口名那样用“内部形式”进行编码，也就是模块名中用点（.）分隔标识符，而不是换成斜杠（/）。

模块名可以从整个Unicode码空间来取值，并服从以下约束：
- 模块名的不能包含闭区间`\u0000`到`\u001F`内的任何代码点。
- 模块名中的反斜杠（\）要保留用来做转义符号。反斜杠不能用在模块名中，除非它后面跟着一个反斜杠或冒号（:）或at符号（@）。可以用`\\`在模块名中编码出一个反斜杠。
- 冒号（:）和at符号（@）在模块名中是留给以后使用的。如果没有转义则不能直接用在模块名中。可以用`\:`和`\@`在模块名中编码出冒号和at符号。

`Module`属性中引用的包名是保存在常量池的`CONSTANT_Package_info`结构体中的（§4.4.12）。一个`CONSTANT_Package_info`结构体内封装了一个用来表示包名内部形式的`CONSTANT_Utf8_info`结构体。

## 4.3 描述符

一个*描述符*是用来表达属性或方法的类型的。描述符在`class`文件格式中使用修正的UTF-8（MUTF-8）字符串（§4.4.7）来表达，因此在没有其他约束的情况下可以从整个Unicode码空间内取值。

### 4.3.1 文法标记

描述符使用特定的文法。这种文法包含一组产生式，描述了如何使用字符序列构造出各种语义正确的描述符。该文法中的终结符我们使用`等宽`字体。非终结符用*斜体*。非终结符定义开始先写上这个非终结符的名字，然后跟一个分号。下面可以继续写一个或多个具体的定义。

在一个产生式右边的*{x}*表示*x*出现零或多次。

产生式右边的<i>（one of）</i>短语表示下面的一行或多行的终结符都是一个可选的定义。

（上面这些东西翻译的时候感觉回到了大学，想起了一些上辈子的知识。）

### 4.3.2 属性描述符

一个*属性描述符*表达了一个类、实例或本地变量的类型。

&emsp;&emsp;*FieldDescriptor:*<br/>
&emsp;&emsp;&emsp;&emsp;*FieldType*<br/>
&emsp;&emsp;*FieldType:*<br/>
&emsp;&emsp;&emsp;&emsp;*BaseType*<br/>
&emsp;&emsp;&emsp;&emsp;*ObjectType*<br/>
&emsp;&emsp;&emsp;&emsp;*ArrayType*<br/>
&emsp;&emsp;*BaseType:*<br/>
&emsp;&emsp;&emsp;&emsp;*(one of)*<br/>
&emsp;&emsp;&emsp;&emsp;`B C D F I J S Z`<br/>
&emsp;&emsp;*ObjectType:*<br/>
&emsp;&emsp;&emsp;&emsp;`L` *ClassName* `;`<br/>
&emsp;&emsp;*ArrayType:*<br/>
&emsp;&emsp;&emsp;&emsp;`[` *ComponentType*<br/>
&emsp;&emsp;*ComponentType:*<br/>
&emsp;&emsp;&emsp;&emsp;*FieldType*<br/>

*BaseType*中的字符，以及*ObjectType*中的`L`和`;`，还有*ArrayType*中的`[`，这些都是ASCII字符。

*ClassName*代表一个二进制类或接口名编码后的内部格式（§4.2.1）。

属性描述符中的类型解释见表4.3-A。

如果属性描述符要表达一个数组类型，那它的维度不能大于255。

**表4.3-A 属性描述符解释**

|***FieldType***|**类型**|**解释**
|-|-|-
|`B`|`byte`|有符号byte
|`C`|`char`|基本多文种平面中的Unicode字符代码点，以UTF-16编码
|`D`|`double`|双精度浮点数
|`F`|`float`|单精度浮点数
|`I`|`int`|整型
|`J`|`long`|长整型
|`L` *ClassName* `;`|`reference`|*ClassName*类的一个实例
|`S`|`short`|有符号short
|`Z`|`boolean`|`true`或`false`
|`[`|`reference`|一维数组

&emsp;&emsp;`int`类型实例变量的属性描述符就是一个`I`。

&emsp;&emsp;`Object`类型实例变量的属性描述符是`Ljava/lang/Object;`。注意这里使用的是内部格式。

&emsp;&emsp;`double[][][]`类型的多维数组实例变量的属性描述符是`[[[D`。

### 4.3.3 方法描述符

一个*方法描述符*包含零或多个*参数描述符*，表达方法携带的参数类型，还有一个*返回描述符*，表达方法返回指（如果有）的类型。

&emsp;&emsp;*MethodDescriptor:*<br/>
&emsp;&emsp;&emsp;&emsp;( *{ParameterDescriptor}* ) *ReturnDescriptor*<br/>
&emsp;&emsp;*ParameterDescriptor:*<br/>
&emsp;&emsp;&emsp;&emsp;*FieldType*<br/>
&emsp;&emsp;*ReturnDescriptor:*<br/>
&emsp;&emsp;&emsp;&emsp;*FieldType*<br/>
&emsp;&emsp;&emsp;&emsp;*VoidDescriptor*<br/>
&emsp;&emsp;*VoidDescriptor:*<br/>
&emsp;&emsp;&emsp;&emsp;`V`<br/>

字符`V`就代表方法没有返回指（返回结果是`void`）。

&emsp;&emsp;一个方法：<br/>
&emsp;&emsp;&emsp;&emsp;`Object m(int i, double d, Thread t) {...}`<br/>
的方法描述符：<br/>
&emsp;&emsp;&emsp;&emsp;`(IDLjava/lang/Thread;)Ljava/lang/Object;`<br/>
&emsp;&emsp;注意这里用了`Thread`和`Object`的内部格式。<br/>

方法描述符的方法参数不能超过255个，其中还包含实例方法或接口方法调用时的`this`参数。这里的个数计算是把每个单独参数的贡献值加起来，其中`long`和`double`都是占用了两个单位，其他类型的参数则只占用一个单位。

方法描述符对于类方法和实例方法都是一样的。尽管实例方法会传一个`this`，引用方法调用时的对象，再跟上其他参数，但这个事儿在方法描述符中看不出来。`this`引用是由JVM指令调用实例方法时隐式传递的（§2.6.1, §4.11）。

## 4.4 常量池

JVM不会依赖运行时的类、接口、类实例或数组的层次结构。而是要引用`constant_pool`表中的符号信息。

`constant_pool`表中的所有记录都具有一般形式：

```
cp_info {
    u1 tag;
    u1 info[];
}
```

`constant_pool`表中的每条记录的开头必须要有一个1字节标记，表示该记录所表达的常量类型。一共有17种常量，对应的标记也都列在了表4.4-A中，按照它们在本节中讲述的顺序排列。每个标记字节后面必须要跟至少两个字节，用来提供指定常量的信息。其他信息的格式依赖于标记字节，根据`tag`的不同，`info`数组的内容也随之变化。

**表4.4-A 常量池标记（按章节号排列）**

|**常量类型**|**标记**|**章节**
|-|-|-
|CONSTANT_Class|7|§4.4.1
|CONSTANT_Fieldref|9|§4.4.2
|CONSTANT_Methodref|10|§4.4.2
|CONSTANT_InterfaceMethodref|11|§4.4.2
|CONSTANT_String|8|§4.4.3
|CONSTANT_Integer|3|§4.4.4
|CONSTANT_Float|4|§4.4.4
|CONSTANT_Long|5|§4.4.5
|CONSTANT_Double|6|§4.4.5
|CONSTANT_NameAndType|12|§4.4.6
|CONSTANT_Utf8|1|§4.4.7
|CONSTANT_MethodHandle|15|§4.4.8
|CONSTANT_MethodType|16|§4.4.9
|CONSTANT_Dynamic|17|§4.4.10
|CONSTANT_InvokeDynamic|18|§4.4.10
|CONSTANT_Module|19|§4.4.11
|CONSTANT_Package|20|§4.4.12

如果一个`class`文件的版本号是*v*，那么`constant_pool`表中的每条记录的标记要么是*v*版本新定义的，要么是以前定义的（§4.1）（总感觉是个废话）。就是说，每条记录所代表的常量类型必须被`class`文件所允许。表4.4-B中列出了每种标记首次定义对应的`class`文件格式版本。以及对应的JavaSE平台版本。

**表4.4-B 常量池标记（按标记值排序）**

|**常量类型**|**标记**|`class`**文件格式**|**Java SE**
|-|-|-|-
|CONSTANT_Utf8|1|45.3|1.0.2
|CONSTANT_Integer|3|45.3|1.0.2
|CONSTANT_Float|4|45.3|1.0.2
|CONSTANT_Long|5|45.3|1.0.2
|CONSTANT_Double|6|45.3|1.0.2
|CONSTANT_Class|7|45.3|1.0.2
|CONSTANT_String|8|45.3|1.0.2
|CONSTANT_Fieldref|9|45.3|1.0.2
|CONSTANT_Methodref|10|45.3|1.0.2
|CONSTANT_InterfaceMethodref|11|45.3|1.0.2
|CONSTANT_NameAndType|12|45.3|1.0.2
|CONSTANT_MethodHandle|15|51.0|7
|CONSTANT_MethodType|16|51.0|7
|CONSTANT_Dynamic|17|55.0|11
|CONSTANT_InvokeDynamic|18|51.0|7
|CONSTANT_Module|19|53.0|9
|CONSTANT_Package|20|53.0|9

`constant_pool`表中的一些记录是*可加载*的，因为它们所表达的实体可以在运行时被压栈做运算。在版本*v*的`class`文件中，如果`contant_pool`表中的一条记录是可加载的，它要么是在版本*v*的时候被认为是可加载的，要么是之前的版本中就被认为是可加载的了（又是一句感觉上的废话）。表4.4-C列出了每个标记首次被认为是可加载时对应的`class`文件格式版本。以及对应的JavaSE平台版本。

&emsp;&emsp;除了`CONSTANT_Class`，标记被认为是可加载的时候都是它首次定义的时候。

**表4.4-C 可加载常量池标记**

|**常量类型**|**标记**|`class`**文件格式**|**Java SE**
|-|-|-|-
|CONSTANT_Integer|3|45.3|1.0.2
|CONSTANT_Float|4|45.3|1.0.2
|CONSTANT_Long|5|45.3|1.0.2
|CONSTANT_Double|6|45.3|1.0.2
|CONSTANT_Class|7|49.0|5.0
|CONSTANT_String|8|45.3|1.0.2
|CONSTANT_MethodHandle|15|51.0|7
|CONSTANT_MethodType|16|51.0|7
|CONSTANT_Dynamic|17|55.0|11

### 4.4.1 CONSTANT_Class_info结构

`CONSTANT_Class_info`结构体用来表达一个类或接口：

```
CONSTANT_Class_info {
    u1 tag;
    u2 name_index;
}
```

解释如下：

`tag`

&emsp;&emsp;值为`CONSTANT_Class`（7）。

`name_index`

&emsp;&emsp;必须是`constant_pool`表中的有效索引。对应的记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个有效的二进制类或接口名，以内部格式编码（§4.2.1）。

因为数组也是对象嘛，操作码*anewarray*和*multianewarray*——可不是*new*——可以通过`constant_pool`表的`CONSTANT_Class_info`结构体来引用数组的“类”。对于这些数组类，类的名字就是数组类型的描述符（§4.3.2）。

&emsp;&emsp;比如二维数组类型`int[][]`的类名是`[[I`，`Thread[]`的类名是`[Ljava/lang/Thread;`。

数组类型的描述符维度不能大于255。

### 4.4.2 CONSTANT_Fieldref_info，CONSTANT_Methodref_info，CONSTANT_InterfaceMethodref_info结构

属性、方法、接口方法，它们都用类似的结构体来表达：

```
CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}

CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}

CONSTANT_InterfaceMethodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

解释如下：

`tag`

&emsp;&emsp;在`CONSTANT_Fieldref_info`中它的值是`CONSTANT_Fieldref`（9）。

&emsp;&emsp;在`CONSTANT_Methodref_info`中它的值是`CONSTANT_Methodref`（10）。

&emsp;&emsp;在`CONSTANT_InterfaceMethodref_info`中它是`CONSTANT_InterfaceMethodref`（11）。

`class_index`

&emsp;&emsp;它的值必须是`constant_pool`表的有效索引。表中对应的记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），表达一个类或接口，它的成员包含了该`class_index`所属的属性或方法。

&emsp;&emsp;在`CONSTANT_Fieldref_info`中，它对应的值可以是一个类或接口类型。

&emsp;&emsp;在`CONSTANT_Methodref_info`中，它对应的值必须是一个类，不能是接口类型。

&emsp;&emsp;在`CONSTANT_InterfaceMethodref_info`中，它对应的值必须是一个接口类型，不能是类。

`name_and_type_index`

&emsp;&emsp;它的值必须是`contant_pool`表的有效索引。对应的记录必须是一个`CONSTANT_NameAndType_info`结构体（§4.4.6）。该记录代表了属性或方法的名字以及描述符。

&emsp;&emsp;在`CONSTANT_Fieldref_info`中，它指明的描述符必须是一个属性描述符（§4.3.2）。其他场景下必须是一个方法描述符（§4.3.3）。

&emsp;&emsp;如果`CONSTANT_Methodref_info`结构体中的方法名以`<`（`\u003c`）开头，那它就必须是特殊的`<init>`，代表实例初始化方法（§2.9.1）。这种方法的返回类型必须是`void`。

### 4.4.3 CONSTANT_String_info结构

该结构用来表达`String`类型的常量对象：

```
CONSTANT_String_info {
    u1 tag;
    u2 string_index;
}
```

解释如下：

`tag`

&emsp;&emsp;值为`CONSTANT_String`（8）。

`string_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。表中对应的记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个Unicode代码点序列，`String`对象用它来初始化。

### 4.4.4 CONSTANT_Integer_info和CONSTANT_Float_info结构

它们用力表达4字节的数字（`int`和`float`）常量：

```
CONSTANT_Integer_info {
    u1 tag;
    u4 bytes;
}
CONSTANT_Float_info {
    u1 tag;
    u4 bytes;
}
```

解释如下：

`tag`

&emsp;&emsp;在`CONSTANT_Integer_info`结构体中它的值是`CONSTANT_Integer`（3）。

&emsp;&emsp;在`CONSTANT_Float_info`结构体中它的值是`CONSTANT_Float`（4）。

`bytes`

&emsp;&emsp;在`CONSTANT_Integer_info`结构体中它的值用来表达`int`常量的值。该值的字节按照大端序进行存储（高位字节在前）。

&emsp;&emsp;在`CONSTANT_Float_info`结构体中它的值用来表达IEEE 754定义的binary32浮点格式（§2.3.2）的`float`常量。该值的字节按照大端序进行存储（高位字节在前）。

&emsp;&emsp;`CONSTANT_Float_info`结构体所表达的值有以下规则。该值的字节首先要转换成一个`int`常量*比特位*。然后：

- 如果*比特位*是`0x7f800000`，这个`float`是正无穷。
- 如果*比特位*是`0xff800000`，这个`float`是负无穷。
- 如果*比特位*的范围是`0x7f800001`到`0x7fffffff`或是`0xff800001`到`0xffffffff`，这个`float`是个NaN。
- 以上都不是的话，假设`s`、`e`、`m`是三个可以从*比特位*计算得到的值：

```
    int s = ((bits >> 31) == 0) ? 1 : -1;
    int e = ((bits >> 23) & 0xff);
    int m = (e == 0) ?
    (bits & 0x7fffff) << 1 :
    (bits & 0x7fffff) | 0x800000;
```

&emsp;&emsp;那么，该`float`值等于表达式<code>s · m · 2<sup>e-150</sup></code>的计算结果。

### 4.4.5 CONSTANT_Long_info和CONSTANT_Double_info结构

`CONSTANT_Long_info`和`CONSTANT_Double_info`代表8字节数字（`long`和`double`）常量：

```
CONSTANT_Long_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}
CONSTANT_Double_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}
```

所有的8字节常量在`class`文件的`constant_pool`表中都要占两条记录。如果一个`CONSTANT_Long_info`或`CONSTANT_Double_info`结构体在`constant_pool`表中的索引是*n*，那么表中下一个可用的索引则是*n*+2。索引*n*+1一定是有效的但是不可使用。

<pre>现在回头想想，让8字节常量占俩位置是一个糟糕的决定。</pre>

解释如下：

`tag`

&emsp;&emsp;在`CONSTANT_Long_info`中它的值是`CONSTANT_Long`（5）。

&emsp;&emsp;在`CONSTANT_Double_info`中它的值是`CONSTANT_Double`（6）。

`high_bytes`、`low_bytes`

&emsp;&emsp;在`CONSTANT_Long_info`中，它俩一起决定了`long`常量的值

```
((long) high_bytes << 32) + low_bytes
```

&emsp;&emsp;`high_bytes`和`low_bytes`的每个字节都是按大端序（高位在前）排列的。

&emsp;&emsp;在`CONSTANT_Double_info`中，他俩一起决定了一个IEEE 754定义的binary64浮点格式（§2.3.2）的`double`值。每个字节也是按大端序（高位在前）来的。

&emsp;&emsp;`CONSTANT_Double_info`表达的值按以下规则生成。首先要把它俩转成`long`常量*比特位*，就等于

```
((long) high_bytes << 32) + low_bytes
```

&emsp;&emsp;然后：

- 如果*比特位*是`0x7ff0000000000000L`，那么这个`double`值是正无穷。
- 如果*比特位*是`0xfff0000000000000L`，那么这个`double`值是负无穷。
- 如果*比特位*的范围是从`0x7ff0000000000001L`到`0x7fffffffffffffffL`，或者是`0xfff0000000000001L`到`0xffffffffffffffffL`，那么这个`double`就是个NaN。
- 以上都不是的话，假设`s`、`e`、`m`是三个可以从*比特位*计算得到的值：

```
int s = ((bits >> 63) == 0) ? 1 : -1;
int e = (int)((bits >> 52) & 0x7ffL);
long m = (e == 0) ?
            (bits & 0xfffffffffffffL) << 1 :
            (bits & 0xfffffffffffffL) | 0x10000000000000L;
```

&emsp;&emsp;那么，该`double`值等于表达式<code>s · m · 2<sup>e-1075</sup></code>的计算结果。

### 4.4.6 CONSTANT_NameAndType_info结构

这玩意用来表达一个属性或者方法，但不包含它所属的类或接口类型：

```
CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;
    u2 descriptor_index;
}
```

解释如下：

`tag`

&emsp;&emsp;值为`CONSTANT_NameAndType`（12）。

`name_index`

&emsp;&emsp;它的值必须得是`constant_pool`表的有效索引。对应的记录必须得是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个有效的属性或方法的未限定名（§4.2.2），或者是特殊的方法名`<init>`（§2.9.1）。

`descriptor_index`

&emsp;&emsp;它的值也得是`constant_pool`表的有效索引。对应的记录同样必须得是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个有效的属性描述符或方法描述符（§4.3.2，§4.3.3）。

### 4.4.7 CONSTANT_Utf8_info结构

该结构体用来表达一个常量字符串值：

```
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```

解释如下：

`tag`

&emsp;&emsp;它的值是`CONSTANT_Utf8`（1）。

`length`

&emsp;&emsp;该值给出了`bytes`数组的字节数量（非最终字符串长度）。

`bytes[]`

&emsp;&emsp;包含了字符串的字节。

&emsp;&emsp;其中不存在值为`(byte) 0`的字节。

&emsp;&emsp;其中不存在值为`(byte)0xf0`到`(byte)0xff`范围内的字节。

字符串内容采用修正的UTF-8编码。使用修正的UTF-8编码，如果代码点序列中只包含非null的ASCII字符，那么每个代码点用1个字节就可以表达了，而且还能保证Unicode码空间内的所有代码点都能表达出来。修正的UTF-8并不以null结尾。编码规则如下：

- `\u0001`到`\u007F`内的代码点用一个字节表达：

|*0*|*bits 6-0*
|-|-

&emsp;&emsp;用7个比特位来表达这个代码点。

- null代码点（`\u0000`）以及`\u0080`到`\u07FF`范围内的代码点用一对字节`x`和`y`表达：

&emsp;&emsp;`x`：

|*1*|*1*|*0*|*bits 10-6*
|-|-|-|-

&emsp;&emsp;`y`：

|*1*|*0*|*bits 5-0*
|-|-|-

&emsp;&emsp;这两个字节表达的代码点的值为：

```
((x & 0x1f) << 6) + (y & 0x3f)
```

- `\u0800`到`\uFFFF`范围内的代码点用3个字节`x`、`y`、`z`表达：

&emsp;&emsp;`x`：

|*1*|*1*|*1*|*0*|*bits 15-12*
|-|-|-|-|-

&emsp;&emsp;`y`：

|*1*|*0*|*bits 11-6*
|-|-|-

&emsp;&emsp;`z`：

|*1*|*0*|*bits 5-0*
|-|-|-

&emsp;&emsp;这三个字节表达的代码点的值为：

```
((x & 0xf) << 12) + ((y & 0x3f) << 6) + (z & 0x3f)
```

- 如果字符的代码点大于U+FFFF（称为*增补字符*），那就要以UTF-16的形式分别编码出两个代理码。每个代理码占三个字节。也就是说增补字符一共六个字节，`u`、`v`、`w`、`x`、`y`、`z`。

&emsp;&emsp;`u`：

|*1*|*1*|*1*|*0*|*1*|*1*|*0*|*1*
|-|-|-|-|-|-|-|-

&emsp;&emsp;`v`：

|*1*|*0*|*1*|*0*|*（bits 20-16）-1*
|-|-|-|-|-

&emsp;&emsp;`w`：

|*1*|*0*|*bits 15-10*
|-|-|-

&emsp;&emsp;`x`：

|*1*|*1*|*1*|*0*|*1*|*1*|*0*|*1*
|-|-|-|-|-|-|-|-

&emsp;&emsp;`y`：

|*1*|*0*|*1*|*0*|*bits 9-6*
|-|-|-|-|-

&emsp;&emsp;`z`：

|*1*|*0*|*bits 5-0*
|-|-|-

&emsp;&emsp;这六个字节表达的代码点值为：

```
0x10000 + ((v & 0x0f) << 16) + ((w & 0x3f) << 10) +
((y & 0x0f) << 6) + (z & 0x3f)
```

多字节字符在`class`文件中也是大端序（高位字节在前）的。

这种格式跟“标准的”UTF-8存在两个不同的地方。首先，空字符`(char) 0`用了2字节格式而不是1字节格式，因此修正的UTF-8字符串不存在嵌套的空字符。第二，标准UTF-8只用了1字节、2字节、3字节格式。JVM不认识四字节格式的标准UTF-8；它用的是双三字节格式。

<pre>关于标准UTF-8的详细介绍，请移步Unicode标准之Unicode编码格式，第13版，第3.9节。</pre>

### 4.4.8 CONSTANT_MethodHandle_info结构

该结构体用来表达一个方法句柄：

```
CONSTANT_MethodHandle_info {
    u1 tag;
    u1 reference_kind;
    u2 reference_index;
}
```

解释如下：

`tag`

&emsp;&emsp;值为`CONSTANT_MethodHandle`（15）。

`reference_kind`

&emsp;&emsp;值必须在1到9之间。代表该方法句柄的*种类*，从而确定它的字节码行为（§5.4.3.5）。

`reference_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应的记录必须：

- 如果`reference_kind`值为1（`REF_getField`）、2（`REF_getStatic`）、3（`REF_putField`）、4（`REF_putStatic`），那么对应的记录必须是一个`CONSTANT_Fieldref_info`结构体（），代表一个属性，该方法句柄就是给这个属性建的。
- 如果值为5（`REF_invokeVirtual`）或8（`REF_newInvokeSpecial`），对应记录必须是一个`CONSTANT_Methodref_info`结构体（§4.4.2）,表达一个类的方法或构造器（§2.9.1），该方法句柄就是给它们建的。
- 如果值为6（`REF_invokeStatic`）或7（`REF_invokeSpecial`），且`class`文件版本号小于52.0，那么对应的记录必须是一个`CONSTANT_Methodref_info`结构体，代表一个类的方法，该方法句柄为它而建；如果版本号大于等于52.0，对应的记录必须要么是一个`CONSTANT_Methodref_info`结构体，要么是一个`CONSTANT_InterfaceMethodref_info`结构体（§4.4.2），代表一个类或接口的方法，该方法句柄为它而建。
- 如果值等于9（`REF_invokeInterface`），对应记录必须是一个`CONSTANT_InterfaceMethodref_info`结构体，代表一个接口的方法，该方法句柄为它而建。

&emsp;&emsp;如果`reference_kind`值等于5（`REF_invokeVirtual`）、6（`REF_invokeStatic`）、7（`REF_invokeSpecial`）、9（`REF_invokeInterface`），对应`CONSTANT_Methodref_info`或`CONSTANT_InterfaceMethodref_info`结构体所表达的方法的名字绝不能是`<init>`或`<clinit>`。

&emsp;&emsp;如果值等于8（`REF_newInvokeSpecial`），对应`CONSTANT_Methodref_info`结构体表达的方法的名字必须是`<init>`。

### 4.4.9 CONSTANT_MethodType_info结构

该结构体用来表达一个方法类型：

```
CONSTANT_MethodType_info {
    u1 tag;
    u2 descriptor_index;
}
```

解释如下：

`tag`

&emsp;&emsp;值为`CONSTANT_MethodType`（16）。

`descriptor_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个方法描述符（§4.3.3）。

### 4.4.10 CONSTANT_Dynamic_info和CONSTANT_InvokeDynamic_info结构

`constant_pool`表中大部分的结构体都是直接用来表达实体的，把名字、描述符、值组合起来，它们都是表中的静态记录。相反，`CONSTANT_Dynamic_info`和`CONSTANT_InvokeDynamic_info`这两个结构体间接表达了实体，它们指向了用来动态计算实体的代码。这种代码，称为*引导方法*，在JVM解析这些结构体派生出的符号引用时，会调用这种代码（§5.1, §5.4.3.6）。每个结构体指明了一个引导方法、一个辅助的名字和类型，用来描述需要被计算的实体。细节如下：

- `CONSTANT_Dynamic_info`结构体用来表达一个*动态计算常量*，它是一个任意值，在*ldc*（§ldc）等其他指令执行的时候，调用引导方法产生了这个值。结构体指定的辅助类型用来约束动态计算出的常量的类型。
- `CONSTANT_InvokeDynamic_info`结构体用来表达一个*动态计算调用点*，它是`java.lang.invoke.CallSite`的一个实例，在*invokedynamic*指令（§invokedynamic）执行的时候调用引导方法所产生。结构体指定的辅助类型用来约束动态计算调用点的方法类型。

```
CONSTANT_Dynamic_info {
    u1 tag;
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;
}
CONSTANT_InvokeDynamic_info {
    u1 tag;
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;
}
```

解释如下：

`tag`

&emsp;&emsp;在`CONSTANT_Dynamic_info`结构体中它的值是`CONSTANT_Dynamic`（17）。

&emsp;&emsp;在`CONSTANT_InvokeDynamic_info`结构体中它的值是`CONSTANT_InvokeDynamic`（18）。

`bootstrap_method_attr_index`

&emsp;&emsp;必须是`class`文件引导方法表`bootstrap_methods`数组的有效索引（§4.7.23）。

&emsp;&emsp;`CONSTANT_Dynamic_info`结构体很特殊，因为在语法上它可以通过引导方法表引用它自己。我们并不强制在类加载时探测出这种循环（这种检查潜在的代价很大），我们允许一开始存在这种循环，但是在解析时要强制报错（§5.4.3.6）。

`name_and_type_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_NameAndType_info`结构体（§4.4.6）。该记录给出一个名字和一个描述符。

&emsp;&emsp;在`CONSTANT_Dynamic_info`中，给出的描述符必须是一个属性描述符（§4.3.2）。

&emsp;&emsp;在`CONSTANT_InvokeDynamic_info`中，给出的描述符必须是一个方法描述符（§4.3.3）。

### 4.4.11 CONSTANT_Module_info结构

这玩意用来表达一个模块：

```
CONSTANT_Module_info {
    u1 tag;
    u2 name_index;
}
```

解释如下：

`tag`

&emsp;&emsp;值为`CONSTANT_Module`（19）。

`name_index`

&emsp;&emsp;必须是`constant_pool`的有效索引。对应的记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个有效的模块名（§4.2.3）。

只有`class`文件声明了一个模块时，该结构体才能出现在它的常量池中，也就是说`ClassFile`结构体的`access_flags`中必须要设置`ACC_MODULE`标记。否则，`CONSTANT_Module_info`结构体的出现都是非法的。

### 4.4.12 CONSTANT_Package_info结构

该结构体用来表达一个被导出的包或是一个模块打开的包：

```
CONSTANT_Package_info {
    u1 tag;
    u2 name_index;
}
```

解释如下：

`tag`

&emsp;&emsp;值为`CONSTANT_Package`（20）。

`name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个有效的包名，并以内部形式编码（§4.2.3）。

只有`class`文件声明了一个模块时，该结构体才能出现在它的常量池中，也就是说`ClassFile`结构体的`access_flags`中必须要设置`ACC_MODULE`标记。否则，`CONSTANT_Package_info`结构体的出现都是非法的。

## 4.5 字段

每个字段都要用一个`field_info`结构体来描述。

同一个`class`文件中字段名和描述符（§4.3.2）唯一。

该结构体格式如下：

```
field_info {
    u2 access_flags;
    u2 name_index;
    u2 descriptor_index;
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

解释如下：

`access_flags`

&emsp;&emsp;它的值是该字段访问权限及属性的标记掩码。每种可能的标记情况见表4.5-A。

&emsp;&emsp;**表4.5-A 字段访问及属性标记**

|**标记名**|**值**|**解释**
|-|-|-
|ACC_PUBLIC|0x0001|`public`声明；可以从包外访问。
|ACC_PRIVATE|0x0002|`private`声明；只能在定义的类或相同嵌套层次内访问（§5.4.4）。
|ACC_PROTECTED|0x0004|`protected`声明；可以被子类访问。
|ACC_STATIC|0x0008|`static`声明。
|ACC_FINAL|0x0010|`final`声明；在对象构造后不能直接赋值（JLS §17.5）。
|ACC_VOLATILE|0x0040|`volatile`声明；不能被缓存。
|ACC_TRANSIENT|0x0080|`transient`声明；无法被持久化对象管理器读写。
|ACC_SYNTHETIC|0x1000|合成声明；源码里看不出来。
|ACC_ENUM|0x4000|声明这是一个`enum`类的元素。

&emsp;&emsp;类中的字段可以设置表4.5-A中的任意标记。但是对于`ACC_PUBLIC`、`ACC_PRIVATE`和`ACC_PROTECTED`标记（JLS §8.3.1），一个类中的一个字段最多只能设置其中的一个，对于`ACC_FINAL`和`ACC_VOLATILE`也不能同时出现（JLS §8.3.1.4）。

&emsp;&emsp;接口中的字段必须要设置`ACC_PUBLIC`、`ACC_STATIC`和`ACC_FINAL`标记；可能还会设置`ACC_SYNTHETIC`，但其他的不能再加了（JLS §9.3）。

&emsp;&emsp;`ACC_SYNTHETIC`标记表示这个字段是编译器生成的，不在源码里。

&emsp;&emsp;`ACC_ENUM`标记表示这个字段保存的是一个枚举类的元素（JLS §8.9）。

&emsp;&emsp;表4.5-A中没有给出的比特位留给以后用。生成`class`文件时它们都得设置成零，并且JVM实现也要忽略这些比特位。

`name_index`

&emsp;&emsp;比如是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达该字段有效的未限定名（§4.2.2）。

`descriptor_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达一个有效的字段描述符。

`attribute_count`

&emsp;&emsp;给出该字段其他属性的数量。

`attributes[]`

&emsp;&emsp;表中每个值都是一个`attribute_info`结构体（§4.7）。

&emsp;&emsp;一个字段可以有任意多个可选属性。

&emsp;&emsp;本书定义的这种属性见表4.7-C。

&emsp;&emsp;这里定义的属性要遵循一定的规则，见§4.7。

&emsp;&emsp;非预定义属性也要遵循一定的规则，见§4.7.1。

## 4.6 方法

每个方法，包括实例初始化方法（§2.9.1）以及类或接口的初始化方法（§2.9.2），都要用`method_info`结构体来描述。

同一个`class`文件中方法的名字和描述符（§4.3.3）唯一。

该结构体格式如下：

```
method_info {
    u2 access_flags;
    u2 name_index;
    u2 descriptor_index;
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

解释如下：

`access_flags`

&emsp;&emsp;它的值是该方法访问权限和属性的掩码。每种标记对应的解释见表4.6-A。

&emsp;&emsp;**表4.6-A 方法访问和属性标记**

|**标记名**|**值**|**解释**
|-|-|-
|ACC_PUBLIC|0x0001|`public`声明；可以从包外访问。
|ACC_PRIVATE|0x0002|`private`声明；只能在定义的类或相同嵌套层次内访问（§5.4.4）。
|ACC_PROTECTED|0x0004|`protected`声明；可以被子类访问。
|ACC_STATIC|0x0008|`static`声明。
|ACC_FINAL|0x0010|`final`声明；不能被重写（§5.4.5）。
|ACC_SYNCHRONIZED|0x0020|`synchronized`声明；调用时被监视器包着。
|ACC_BRIDGE|0x0040|桥方法，由编译器生成。
|ACC_VARARGS|0x0080|声明数量可变参数。
|ACC_NATIVE|0x0100|`native`声明；不是用Java语言实现的方法。
|ACC_ABSTRACT|0x0400|`abstract`声明；不提供实现。
|ACC_STRICT|0x0800|如果`class`文件大版本号是[46,60]范围内则用来声明`strictfp`。
|ACC_SYNTHETIC|0x1000|合成声明；源码中看不到。

&emsp;&emsp;`0x0800`只有在`class`文件大版本为[46,60]这个区间内的时候才会被解释成`ACC_STRICT`。此时，`ACC_STRICT`能够跟其它标记一同设置取决于以下规则。（从JavaSE 1.2到16中，`ACC_STRICT`标记会约束方法的浮点数指令（§2.8））。如果`class`文件大版本小于46或大于60，`0x0800`并不会解释成`ACC_STRICT`标记，而是一个未设置的状态；在这种`class`文件中“设置`ACC_STRICT`标记”并无任何意义。

&emsp;&emsp;类中的方法可以任意选择表4.6-A中的标记进行设置。但是`ACC_PUBLIC`、`ACC_PRIVATE`、`ACC_PROTECTED`最多只能设置其中一个（JLS §8.4.3）。

&emsp;&emsp;接口中的方法除了不能用`ACC_PROTECTED`、`ACC_FINAL`、`ACC_SYNCHRONIZED`、`ACC_NATIVE`以外其他的也都可以任选（JLS §9.4）。如果`class`文件版本号小于52.0，那么接口方法必须要设置`ACC_PUBLIC`和`ACC_ABSTRACT`标记；如果大于等于52.0，每个接口方法只能设置`ACC_PUBLIC`或`ACC_PRIVATE`其中一个。

&emsp;&emsp;如果一个类或接口方法设置了`ACC_ABSTRACT`，那就不能同时设置`ACC_PRIVATE`、`ACC_STATIC`、`ACC_FINAL`、`ACC_SYNCHRONIZED`、`ACC_NATIVE`，以及（`class`文件大版本在[46,60]之间时）`ACC_STRICT`也不行。

&emsp;&emsp;实例初始化方法（§2.9.1）最多只能设置`ACC_PUBLIC`、`ACC_PRIVATE`和`ACC_PROTECTED`其中的一个，而且还要同时设置`ACC_VARARGS`和`ACC_SYNTHETIC`标记，而且还要（`class`文件大版本号在[46,60]时）设置`ACC_STRICT`，而且表4.6-A中其他的任何标记都不能设置。

&emsp;&emsp;如果`class`文件版本大于等于51.0，方法名为`<clinit>`的方法必须要设置`ACC_STATIC`标记。

&emsp;&emsp;类或接口的初始化方法（§2.9.2）是由JVM隐式调用的。如果没有设置`ACC_STATIC`以及（`class`文件大版本号为[46,60]时）`ACC_STRICT`标记，`access_flags`项会被直接忽略，而且方法不受之前讲的那些标记组合规则的限制。

&emsp;&emsp;`ACC_BRIDGE`标记用来表示一个桥方法，它是Java语言编译器生成的。

&emsp;&emsp;`ACC_VARARGS`表示该方法在源码级可以接受可变数量的参数。这种方法编译时必须将`ACC_VARARGS`标记设置成1。所有其他的方法编译时`ACC_VARARGS`必须是0。

&emsp;&emsp;`ACC_SYNTHETIC`标记表示该方法是编译器生成的，在源码里看不到，除非它是§4.7.8列出的方法之一。

&emsp;&emsp;表4.6-A中没有说明的`access_flags`的其他的比特位都留以后用。（其中包括`class`文件大版本为[46,60]时的0x0800。）生成的`class`文件中这些位置都应该归零，并且被JVM实现所忽略。

`name_index`

&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须时要给`CONSTANT_Utf8_info`结构体（§4.4.7），用来表示方法的一个有效的非限定名（§4.2.2），或者（方法是类方法而非接口方法）是特殊的`<init>`或`<clinit>`方法名。

`descriptor_index`

&emsp;&emsp;呵呵呵呵，必须是`constant_pool`的有效索引。对应的值必须是一个`CONSTANT_Utf8_info`结构体，代表一个有效的方法描述符（§4.3.3）。而且：

- 如果是在类里面而不是接口中，并且这个方法名为`<init>`，那么描述符必须是一个`void`方法。
- 如果方法名为`<clinit>`，描述符必须是一个`void`方法，并且，如果`class`文件版本号大于等于51.0，该方法不能带参数。

&emsp;&emsp;未来可能会要求如果`access_flags`中设置了`ACC_VARARGS`，方法描述符的最后一个参数描述符得是一个数组类型。

`attribute_count`

&emsp;&emsp;代表该方法额外属性的数量。

`attributes[]`

&emsp;&emsp;其中的每个元素必须是一个`attribute_info`结构体（§4.7）。

&emsp;&emsp;方法可以有任意多个可选属性。

&emsp;&emsp;本书中定义的可以出现在此处的属性见表4.7-C。

&emsp;&emsp;相关的规则见§4.7。

&emsp;&emsp;非预定义属性的相关规则见§4.7.1。

## 4.7 属性

在`class`文件结构中，*属性*被用于`ClassFile`、`field_info`、`method_info`、`Code_attribute`、`record_component_info`结构体中（§4.1，§4.5，§4.6，§4.7.3，§4.7.30）。

所有的属性都具备以下一般形式：

```
attribute_info {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 info[attribute_length];
}
```

对于所有的属性来说，`attribute_name_index`必须是一个有效的、无符号的16位的常量池索引。`constant_pool`中对应的记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表属性的名字。`attribute_length`的值代表后续其他信息的字节数。该字节数不包括起始的六个字节，其中包含了`attribute_name_index`以及`attribute_length`。

本书中介绍了30个预定义的属性。为了方便查询，给出三种排列：

- 表4.7-A按属性所在章节号排序。每个属性跟着它首次定义、首次出现时的`class`文件版本号。以及该版本对应的JavaSE平台版本（§4.1）。
- 表4.7-B按属性首次定义时的`class`文件格式版本号排序。
- 表4.7-C按属性在`class`文件中出现的顺序排列。

在本书介绍的上下文中，也就是这些属性所在的`attributes`表中，这些预定义属性的名字都是被保留的名字。

`attributes`表中预定义属性出现的条件都明确的写在了介绍对应属性的小节中。如果没有说明对应的条件，那么该属性在`attributes`表中可以出现任意多次。

根据用途不同，预定义属性分为三组：

- 1. 这七种属性对于JVM正确解释`class`文件至关重要：
    - `ConstantValue`
    - `Code`
    - `StackMapTable`
    - `BootstrapMethods`
    - `NestHost`
    - `NestMembers`
    - `PermittedSubclasses`

&emsp;&emsp;如果`class`文件版本号为*v*，如果一个JVM实现支持该版本的`class`文件格式，那么这些属性必须能够被JVM识别并正确的读取，这些属性要么是*v*版本定义的要么是以前就有的，而且都出现在了它们该出现的位置上。

- 2. 这十种属性对于JVM正确解释`class`并不那么重要，但对于JavaSE平台类库正确解释`class`文件很重要，而且对于其他工具也很有用（此时对应的小节中会将属性描述成“可选的”）：
    - `Exceptions`
    - `InnerClasses`
    - `EnclosingMethod`
    - `Synthetic`
    - `Signature`
    - `Record`
    - `SourceFile`
    - `LineNumberTable`
    - `LocalVariableTable`
    - `LocalVariableTypeTable`

&emsp;&emsp;如果`class`文件版本号为*v*，如果一个JVM实现支持该版本的`class`文件格式，那么这些属性必须能够被JVM识别并正确的读取，这些属性要么是*v*版本定义的要么是以前就有的，而且都出现在了它们该出现的位置上。咯咯咯，我直接复制上面的了。

- 3. 这十三种属性对于JVM正确解释`class`同样不那么重要，但是包含了`class`文件的元数据，它们要么是由JavaSE平台的类库暴露出来，要么是被工具加以利用（此时对应的小节中会将属性描述成“可选的”）：
    - `SourceDebugExtension`
    - `Deprecated`
    - `RuntimeVisibleAnnotations`
    - `RuntimeInvisibleAnnotations`
    - `RuntimeVisibleParameterAnnotations`
    - `RuntimeInvisibleParameterAnnotations`
    - `RuntimeVisibleTypeAnnotations`
    - `RuntimeInvisibleTypeAnnotations`
    - `AnnotationDefault`
    - `MethodParameters`
    - `Module`
    - `ModulePackages`
    - `ModuleMainClass`

&emsp;&emsp;JVM实现中可以选择使用这些属性所包含的信息，也可以简单的忽略这些属性。

**表4.7-A 预定义的`class`文件属性（按章节号）**
|**属性**|**章节**|**`class`文件**|**JavaSE**
|-|-|-|-
|ConstantValue|§4.7.2|45.3|1.0.2
|Code|§4.7.3|45.3|1.0.2
|StackMapTable|§4.7.4|50.0|6
|Exceptions|§4.7.5|45.3|1.0.2
|InnerClasses|§4.7.6|45.3|1.1
|EnclosingMethod|§4.7.7|49.0|5.0
|Synthetic|§4.7.8|45.3|1.1
|Signature|§4.7.9|49.0|5.0
|SourceFile|§4.7.10|45.3|1.0.2
|SourceDebugExtension|§4.7.11|49.0|5.0
|LineNumberTable|§4.7.12|45.3|1.0.2
|LocalVariableTable|§4.7.13|45.3|1.0.2
|LocalVariableTypeTable|§4.7.14|49.0|5.0
|Deprecated|§4.7.15|45.3|1.1
|RuntimeVisibleAnnotations|§4.7.16|49.0|5.0
|RuntimeInvisibleAnnotations|§4.7.17|49.0|5.0
|RuntimeVisibleParameterAnnotations|§4.7.18|49.0|5.0
|RuntimeInvisibleParameterAnnotations|§4.7.19|49.0|5.0
|RuntimeVisibleTypeAnnotations|§4.7.20|52.0|8
|RuntimeInvisibleTypeAnnotations|§4.7.21|52.0|8
|AnnotationDefault|§4.7.22|49.0|5.0
|BootstrapMethods|§4.7.23|51.0|7
|MethodParameters|§4.7.24|52.0|8
|Module|§4.7.25|53.0|9
|ModulePackages|§4.7.26|53.0|9
|ModuleMainClass|§4.7.27|53.0|9
|NestHost|§4.7.28|55.0|11
|NestMembers|§4.7.29|55.0|11
|Record|§4.7.30|60.0|16
|PermittedSubclasses|§4.7.31|61.0|17

**表4.7-B 预定义的`class`文件属性（按`class`文件格式）**
|**属性**|**`class`文件**|**JavaSE**|**章节**
|-|-|-|-
|ConstantValue|45.3|1.0.2|§4.7.2
|Code|45.3|1.0.2|§4.7.3
|Exceptions|45.3|1.0.2|§4.7.5
|SourceFile|45.3|1.0.2|§4.7.10
|LineNumberTable|45.3|1.0.2|§4.7.12
|LocalVariableTable|45.3|1.0.2|§4.7.13
|InnerClasses|45.3|1.1|§4.7.6
|Synthetic|45.3|1.1|§4.7.8
|Deprecated|45.3|1.1|§4.7.15
|EnclosingMethod|49.0|5.0|§4.7.7
|Signature|49.0|5.0|§4.7.9
|SourceDebugExtension|49.0|5.0|§4.7.11
|LocalVariableTypeTable|49.0|5.0|§4.7.14
|RuntimeVisibleAnnotations|49.0|5.0|§4.7.16
|RuntimeInvisibleAnnotations|49.0|5.0|§4.7.17
|RuntimeVisibleParameterAnnotations|49.0|5.0|§4.7.18
|RuntimeInvisibleParameterAnnotations|49.0|5.0|§4.7.19
|AnnotationDefault|49.0|5.0|§4.7.22
|StackMapTable|50.0|6|§4.7.4
|BootstrapMethods|51.0|7|§4.7.23
|RuntimeVisibleTypeAnnotations|52.0|8|§4.7.20
|RuntimeInvisibleTypeAnnotations|52.0|8|§4.7.21
|MethodParameters|52.0|8|§4.7.24
|Module|53.0|9|§4.7.25
|ModulePackages|53.0|9|§4.7.26
|ModuleMainClass|53.0|9|§4.7.27
|NestHost|55.0|11|§4.7.28
|NestMembers|55.0|11|§4.7.29
|Record|60.0|16|§4.7.30
|PermittedSubclasses|61.0|17|§4.7.31

**表4.7-C 预定义的`class`文件属性（按位置）**
|**属性**|**位置**|**`class`文件**
|-|-|-
|SourceFile|ClassFile|45.3
|InnerClasses|ClassFile|45.3
|EnclosingMethod|ClassFile|49.0
|SourceDebugExtension|ClassFile|49.0
|BootstrapMethods|ClassFile|51.0
|Module,|ModulePackages,|ModuleMainClass|ClassFile|53.0
|NestHost,|NestMembers|ClassFile|55.0
|Record|ClassFile|60.0
|PermittedSubclasses|ClassFile|61.0
|ConstantValue|field_info|45.3
|Code|method_info|45.3
|Exceptions|method_info|45.3
|RuntimeVisibleParameterAnnotations,
|RuntimeInvisibleParameterAnnotations
|method_info|49.0
|AnnotationDefault|method_info|49.0
|MethodParameters|method_info|52.0
|Synthetic|ClassFile,field_info,method_info|45.3
|Deprecated|ClassFile,field_info,method_info|45.3
|Signature|ClassFile,field_info,method_info,record_component_info|49.0
|RuntimeVisibleAnnotations,RuntimeInvisibleAnnotations|ClassFile,field_info,method_info,record_component_info|49.0
|LineNumberTable|Code|45.3
|LocalVariableTable|Code|45.3
|LocalVariableTypeTable|Code|49.0
|StackMapTable|Code|50.0
|RuntimeVisibleTypeAnnotations,RuntimeInvisibleTypeAnnotations|ClassFile,field_info,method_info,Code,record_component_info|52.0