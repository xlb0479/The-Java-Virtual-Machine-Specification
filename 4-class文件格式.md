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

