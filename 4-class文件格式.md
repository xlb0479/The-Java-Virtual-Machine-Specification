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
|Module,ModulePackages,ModuleMainClass|ClassFile|53.0
|NestHost,NestMembers|ClassFile|55.0
|Record|ClassFile|60.0
|PermittedSubclasses|ClassFile|61.0
|ConstantValue|field_info|45.3
|Code|method_info|45.3
|Exceptions|method_info|45.3
|RuntimeVisibleParameterAnnotations,RuntimeInvisibleParameterAnnotations|method_info|49.0
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

### 4.7.1 定义和命名新的属性

编译器可以定义并生成包含新属性的`class`文件，这些属性包括`class`文件结构体中的属性、`field_info`结构体中的属性、`method_info`结构体中的属性以及`Code`属性（§4.7.3）。JVM实现可以识别并使用`attributes`表中新出现的属性。但是如果属性并非本书定义的属性之一，那么它就不能影响到`class`文件的语义。JVM实现对于它们不认识的属性可以直接忽略。

比如可以定义一个支持特定调试工具的新属性。因为JVM实现的时候可以忽略它们不认识的属性，即便是专门为特定JVM设计的`class`文件，也可以被其他JVM实现所使用，即便这些实现并不能利用到`class`文件中包含的调试信息。

如果遇到了新的属性，JVM实现不能抛出异常或拒绝使用该`class`文件。当然，如果`class`文件没有提供足够的属性，对应的工具可能无法正确执行。

两个不同属性之间应该是有所区分的，但也会碰巧使用了相同的名字并且长度也一样，此时在实现中就会出现冲突，只能识别其中一个属性。如果不是本书中定义的属性，起名应当参考*Java语言规范，JavaSE 第17版*（JLS §6.1）中的包名规范。

本规范后续的版本也可能会定义出新的属性。

### 4.7.2 ConstantValue属性

该属性是`field_info`结构体（§4.5）的`attributes`表中的一个固定长度的属性。该属性代表了一个常量表达式（JLS §15.28）的值，用法如下：

- 如果`field_info`结构体的`access_flags`中设置了`ACC_STATIC`标记，那么该`field_info`结构体所表达的字段就会被赋值为它的`ConstantValue`属性所表达的值，此过程发生在声明该字段的类或接口进行初始化的时候（§5.5）。此过程发生在该类或接口的初始化方法被调用之前（§2.9.2）。
- 否则，JVM必须要忽略掉这个属性。

一个`field_info`结构体的`attributes`表中最多只能有一个`ConstantValue`属性。

该属性格式如下：

```
ConstantValue_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 constantvalue_index;
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;该值必须为`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达的字符串值为“`ConstantValue`”

`attribute_length`

&emsp;&emsp;它的值必须是二。

`constantvalue_index`

&emsp;&emsp;该值必须为`constant_pool`表的有效索引。对应记录给出了该属性的值。且该记录的类型必须能跟该字段的类型对应上，见表4.7.2-A。

**表4.7.2-A 常量值属性类型**

|**字段类型**|**记录类型**
|-|-
|int,short,char,byte,boolean|CONSTANT_Integer
|float|CONSTANT_Float
|long|CONSTANT_Long
|double|CONSTANT_Double
|String|CONSTANT_String

### 4.7.3 Code属性

该属性是`method_info`结构体（§4.6）的`attributes`表中的一个变长属性。该属性中包含了JVM指令以及方法的一些辅助信息，这些方法包括实例初始化方法以及类或接口初始化方法（§2.9.1, §2.9.2）。

如果该方法是`native`或`abstract`，且不是类或接口初始化方法，那么它的`method_info`结构体的`attributes`表中就不能包含`Code`属性。否则，它的`attributes`表中就必须仅包含一个`Code`属性。

`Code`属性格式如下：

```
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），表达的字符串值为“`Code`”。

`attribute_length`

&emsp;&emsp;代表该属性的长度，包含开头的六个字节。

`max_stack`

&emsp;&emsp;给出该方法的操作数栈在任意执行点时的最大深度（§2.6.2）。

`max_locals`

&emsp;&emsp;给出调用该方法时分配的局部变量数组中的局部变量个数（§2.6.1），包括在调用时用来传参的局部变量。

&emsp;&emsp;`long`或`double`类型的值的局部变量索引最大为`max_locals - 2`。其他类型值的局部变量索引最大为`max_locals - 1`。

`code_length`

&emsp;&emsp;代表该方法的`code`数组的长度。

&emsp;&emsp;必须大于零（`code`数组不能为空）且小于65536。

`code[]`

&emsp;&emsp;包含用来实现该方法的JVM代码的真实字节。

&emsp;&emsp;当该数组被读入到一个字节可寻址机器的内存中时，如果该数组的首字节对齐到了4字节边界，那么*tableswitch*和*lookupswitch*的32位偏移量也会对齐到4字节。（关于`code`数组对齐的结果，可以到这些指令的描述信息中查看详细的介绍。）

`exception_table_length`

&emsp;&emsp;代表`exception_table`数组包含的记录数。

`exception_table[]`

&emsp;&emsp;这个表里的每条记录代表`code`数组中的一个异常处理器。它们在该表中的顺序时很重要滴（§2.10）。

&emsp;&emsp;每条记录包含以下字段：

&emsp;&emsp;`start_pc`，`end_pc`

&emsp;&emsp;&emsp;&emsp;它们俩的值代表了该异常处理器在`code`数组中的激活范围。`start_pc`必须是`code`数组中某个指令操作码的有效索引。`end_pc`同样必须是这样，或者也可以等于`code_length`，也就是`code`数组的长度。`start_pc`必须小于`end_pc`。

&emsp;&emsp;&emsp;&emsp;包含`start_pc`但不包含`end_pc`；也就是程序计数器位于`[start_pc,end_pc)`范围内时，该异常处理器必须处于激活状态。

&emsp;&emsp;&emsp;&emsp;这个范围为什么不能包含`end_pc`是JVM在设计上的一个历史性的错误：如果一个方法的JVM代码正好是65535个字节，并且以一个1字节指令结束，那么这个指令就无法被异常处理器保护起来。编译器的开发者可以通过限制各种方法，包括实例初始化方法或静态初始化器，生成的JVM代码大小（`code`数组的长度 ）来规避这个bug，限制到65534个字节。

&emsp;&emsp;`handler_pc`

&emsp;&emsp;&emsp;&emsp;表示异常处理器的开始。它的值必须是`code`数组中某条指令操作码的有效索引。

&emsp;&emsp;`catch_type`

&emsp;&emsp;&emsp;&emsp;如果它的值非零，那就必须是`constant_pool`的有效索引。对应的记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），代表该异常处理器要捕获的异常类。只有当抛出的异常时指定类或其子类的一个实例时才会调用该异常处理器。

&emsp;&emsp;&emsp;&emsp;校验器会检查这个类是否是`Throwable`或其子类（§4.9.2）。

&emsp;&emsp;&emsp;&emsp;如果它的值是零，该异常处理器则用来捕获所有的异常。

&emsp;&emsp;&emsp;&emsp;这是用来实现`finally`的（§3.13）。

`attributes_count`

&emsp;&emsp;代表`Code`属性中属性的数量。（额。。。属性的属性。属属属属属属属。。。）

`attributes[]`

&emsp;&emsp;其中每个值必须是一个`attribute_info`结构体（§4.7）。

&emsp;&emsp;一个`Code`属性可以有任意多个可选的关联属性。

&emsp;&emsp;本书中定义的可以出现在此处的属性见表4.7-C。

&emsp;&emsp;相关规则见§4.7。

&emsp;&emsp;非预定义属性的相关规则见§4.7.1。

### 4.7.4 StackMapTable属性

它是`Code`属性（§4.7.3）的`attributes`表中的一个变长属性。该属性用于类型检查校验（§4.10.1）。

一个`Code`属性的`attributes`表中最多只能有一个`StackMapTable`属性。

当`class`文件版本号大于等于50.0，如果方法的`Code`属性不包含`StackMapTable`属性，那它会有一个*隐式的栈映射属性*（§4.10.1）。隐式栈映射属性相当于一个`number_of_entries`等于零的`StackMapTable`。

该属性格式如下：

```
StackMapTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_entries;
    stack_map_frame entries[number_of_entries];
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;它的值必须是`constant_pool`表的有效索引。对应的记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串“`StackMapTable`”。

`attribute_length`

&emsp;&emsp;代表属性的长度，包括开头的六个字节。

`number_of_entries`

&emsp;&emsp;代表`entries`表中`stack_map_frame`记录的个数。

`entries[]`

&emsp;&emsp;表中的每条记录代表该方法的一个栈映射帧。它们在该表中的顺序是很重要滴。

一个*栈映射帧*声明了（显式或隐式的）它所应用的字节码偏移量，以及在对应偏移量上的局部变量以及操作数栈记录的校验类型。

`entries`表中的每一个栈映射帧的部分语义都会依赖于它的前一帧。方法的第一个栈映射帧是隐式的，是类型检查器（§4.10.1.6）根据方法描述符计算得出的。因此`entries[0]`处的`stack_map_frames`结构体描述的是方法中的第二个栈映射帧。

栈映射帧使用的*字节码偏移量*是根据帧中的`offset_delta`（显式或隐式的）计算出来的，并且通过`offset_delta + 1`就可以得出前一帧的字节码偏移量，除非前一帧是方法的初始帧。此时，这个栈映射帧应用的字节码偏移量就等于帧中声明的`offset_delta`。

<pre>我们用这种偏移量的变化量，而不是直接保存真正的字节码偏移量，这样我们可以从定义上保证栈映射帧的顺序是正确的。而且，对所有的显式帧（相对于开头的隐式帧）使用offset_delta + 1这种公式，我们能够保证不会出现重复数据。</pre>

我们说一个指令在字节码中有一个*关联栈映射帧*，前提是这个指令位于一个`Code`属性的`code`数组中的偏移量*i*，并且这个`Code`属性有一个`StackMapTable`属性，它的`entries`数组中有一个栈映射帧是应用在字节码偏移量*i*处的。

一个*校验类型*声明了一处或两处位置的类型，这个*位置*对应一个局部变量或者一条操作数栈的记录。一个校验类型通过一个可区分共用体来表达，`verification_type_info`，它包含了一个单字节标记，代表共用体中正在使用的元素，然后跟着零或多个字节，给出该标记的其他详细信息。

```
union verification_type_info {
    Top_variable_info;
    Integer_variable_info;
    Float_variable_info;
    Long_variable_info;
    Double_variable_info;
    Null_variable_info;
    UninitializedThis_variable_info;
    Object_variable_info;
    Uninitialized_variable_info;
}
```

一个校验类型声明一处局部变量表或操作数栈中的位置，需要通过`verification_type_info`共用体的以下属性进行表达：

- `Top_variable_info`代表该局部变量拥有校验类型`top`。

```
Top_variable_info {
    u1 tag = ITEM_Top; /* 0 */
}
```

- `Integer_variable_info`代表该位置拥有校验类型`int`。

```
Integer_variable_info {
    u1 tag = ITEM_Integer; /* 1 */
}
```

- `Float_variable_info`代表该位置拥有校验类型`float`。

```
Float_variable_info {
    u1 tag = ITEM_Float; /* 2 */
}
```

- `Null_variable_info`代表该位置拥有校验类型`null`。

```
Null_variable_info {
    u1 tag = ITEM_Null; /* 5 */
}
```

- `UninitializedThis_variable_info`代表该位置拥有校验类型`uninitializedThis`。

```
UninitializedThis_variable_info {
    u1 tag = ITEM_UninitializedThis; /* 6 */
}
```

- `Object_variable_info`代表该位置的校验类型要通过`constant_pool`表中`cpool_index`索引处的`CONSTANT_Class_info`结构体（§4.4.1）来进行表达。

```
Object_variable_info {
    u1 tag = ITEM_Object; /* 7 */
    u2 cpool_index;
}
```

- `Uninitialized_variable_info`代表该位置的校验类型为`uninitialized(Offset)`。其中`Offset`代表`Code`属性中`code`数组包含`StackMapTable`属性的偏移量，对应的*new*指令用来创建保存在该位置中的对象。

```
Uninitialized_variable_info {
    u1 tag = ITEM_Uninitialized; /* 8 */
    u2 offset;
}
```

一个校验类型如果要声明两处局部变量数组或操作数栈中的位置，那就要使用`verification_type_info`共用体的以下属性：

- `Long_variable_info`代表两个位置中的第一个拥有校验类型`long`。

```
Long_variable_info {
    u1 tag = ITEM_Long; /* 4 */
}
```

- `Double_variable_info`代表两个位置中的第一个拥有校验类型`double`。

```
Double_variable_info {
    u1 tag = ITEM_Double; /* 3 */
}
```

- `Long_variable_info`和`Double_variable_info`按照以下方法表达两个位置中第二个位置的校验类型：
    - 如果第一个位置是一个局部变量，那么：
        - 它不能是最高索引处的局部变量。
        - 次高位的局部变量的校验类型为`top`。
    - 如果第一个位置是一条操作数栈记录，那么：
        - 它不能是操作数栈顶位置。
        - 操作数栈的次顶位置的校验类型为`top`。

一个栈映射帧要用一个可区分共用体表达，`stack_map_frame`，它包含一个单字节标记，表示当前使用的是哪个元素，然后跟着零或多个字节，给出关于此标记的详细信息。

```
union stack_map_frame {
    same_frame;
    same_locals_1_stack_item_frame;
    same_locals_1_stack_item_frame_extended;
    chop_frame;
    same_frame_extended;
    append_frame;
    full_frame;
}
```

标记是用来表示该栈映射帧的*帧类型*：

- 标记值范围在[0-63]之间表示的帧类型为`same_frame`。这种类型表示这一帧跟前一帧拥有相同的局部变量，并且操作数栈是空的。这一帧的`offset_delta`值就是标记的值，`frame_type`。

```
same_frame {
    u1 frame_type = SAME; /* 0-63 */
}
```

- 标记值在[64,127]之间表示的帧类型为`same_locals_1_stack_item_frame`。这种类型表示这一帧跟前一帧拥有相同的局部变量，并且操作数栈只有一条记录。这种帧的`offset_delta`值的计算方式是`frame_type - 64`。这一条栈记录的校验类型出现在帧类型的后面。

```
same_locals_1_stack_item_frame {
    u1 frame_type = SAME_LOCALS_1_STACK_ITEM; /* 64-127 */
    verification_type_info stack[1];
}
```

- [128-246]范围内的标记值保留，以后再用。

- `same_locals_1_stack_item_frame_extended`帧类型的标记值为247。这种类型表示这一帧跟前一帧拥有相同的局部变量，并且操作数栈只有一条记录。这种帧的`offset_delta`值是直给的，不像`same_locals_1_stack_item_frame`。这一条栈记录的校验类型出现在`offset_delta`的后面。

```
same_locals_1_stack_item_frame_extended {
    u1 frame_type = SAME_LOCALS_1_STACK_ITEM_EXTENDED; /* 247 */
    u2 offset_delta;
    verification_type_info stack[1];
}
```

- `chop_frame`帧类型的标记值范围是[248-250]。这种类型表示这一帧跟前一帧拥有相同的局部变量，但缺少最后*k*个局部变量，并且操作数栈是空的。*k*的值由`251 - frame_type`得出。这种帧的`offset_delta`值也是直给的。

```
chop_frame {
    u1 frame_type = CHOP; /* 248-250 */
    u2 offset_delta;
}
```

&emsp;&emsp;假设前一帧的局部变量的校验类型由`locals`给出，该数组结构同`full_frame`帧类型。如果前一帧中的`locals[M-1]`代表局部变量*X*，`locals[M]`代表局部变量*Y*，那么移除一个局部变量的结果就是`locals[M-1]`在新的帧中代表局部变量*X*，而`locals[M]`则是未定义的。

&emsp;&emsp;如果*k*大于前一帧`locals`中的局部变量的个数则是不正确的，会导致新帧中的局部变量的个数小于零。

- `same_frame_extended`帧类型的标记值是251。这种类型表示这一帧跟前一帧拥有相同的局部变量，并且操作数栈为空。该帧的`offset_delta`值也是直给的，不像`same_frame`。

```
same_frame_extended {
    u1 frame_type = SAME_FRAME_EXTENDED; /* 251 */
    u2 offset_delta;
}
```

- `append_frame`帧类型的标记值范围是[252-254]。这种类型表示这一帧跟前一帧拥有相同的局部变量，但是额外定义了*k*个局部变量，并且操作数栈为空。*k*的值由`frame_type - 251`给出。该帧的`offset_delta`值也是直给的。

```
append_frame {
    u1 frame_type = APPEND; /* 252-254 */
    u2 offset_delta;
    verification_type_info locals[frame_type - 251];
}
```

&emsp;&emsp;`locals`中的第0个记录代表首个额外局部变量的校验类型。假设`locals[M]`代表局部变量`N`，那么：

&emsp;&emsp;- 如果`locals[M]`属于`Top_variable_info`、`Integer_variable_info`、`Float_variable_info`、`Null_variable_info`、`UninitializedThis_variable_info`、`Object_variable_info`、`Uninitialized_variable_info`之一，那么`locals[M+1]`代表局部变量`N+1`；并且
&emsp;&emsp;- 如果`locals[M]`属于`Long_variable_info`或`Double_variable_info`，那么`locals[M+1]`代表局部变量`N+2`。

&emsp;&emsp;对于任意索引*i*，如果`locals[i]`代表的局部变量的索引大于该方法局部变量数量最大值的话，那就是不正确的。

- `full_frame`帧类型的标记值是255。该帧的`offset_delta`值也是直给的。

```
full_frame {
    u1 frame_type = FULL_FRAME; /* 255 */
    u2 offset_delta;
    u2 number_of_locals;
    verification_type_info locals[number_of_locals];
    u2 number_of_stack_items;
    verification_type_info stack[number_of_stack_items];
}
```

&emsp;&emsp;`locals`中的第0个记录代表局部变量0的校验类型。假设`locals[M]`代表局部变量`N`，那么：

&emsp;&emsp;- 如果`locals[M]`属于`Top_variable_info`、`Integer_variable_info`、`Float_variable_info`、`Null_variable_info`、`UninitializedThis_variable_info`、`Object_variable_info`、`Uninitialized_variable_info`之一，那么`locals[M+1]`代表局部变量`N+1`；并且
&emsp;&emsp;- 如果`locals[M]`属于`Long_variable_info`或`Double_variable_info`，那么`locals[M+1]`代表局部变量`N+2`。

&emsp;&emsp;对于任意索引*i*，如果`locals[i]`代表的局部变量的索引大于该方法局部变量数量最大值的话，那就是不正确的。

&emsp;&emsp;`stack`中的第0条记录代表操作数栈底的校验类型，`stack`中越往后的记录代表的校验类型对应的栈记录离栈顶越近。我们将栈底记录记为0记录，然后往后就是记录1，2这样。假设`stack[M]`代表栈记录`N`，那么：

&emsp;&emsp;- 如果`stack[M]`属于`Top_variable_info`、`Integer_variable_info`、`Float_variable_info`、`Null_variable_info`、`UninitializedThis_variable_info`、`Object_variable_info`、`Uninitialized_variable_info`之一，那么`stack[M+1]`代表栈记录`N+1`；并且

&emsp;&emsp;- 如果`stack[M]`属于`Long_variable_info`或`Double_variable_info`，那么`stack[M+1]`代表栈记录`N+2`。

&emsp;&emsp;对于任意索引*i*，如果`stack[i]`代表的栈记录的索引大于该方法操作数栈大小的最大值的话，那就是不正确的。

### 4.7.5 Exceptions属性

它是`method_info`结构体（§4.6）的`attributes`表中的一个变长属性。该属性揭示了一个方法可能会抛出的已检查异常。

在一个`method_info`结构体的`attributes`表中，最多只能有一个`Exceptions`属性。

该属性格式如下：

```
Exceptions_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_exceptions;
    u2 exception_index_table[number_of_exceptions];
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），字符串值为“`Exceptions`”。

`attribute_length`

&emsp;&emsp;表示该属性的长度，包含开头的六个字节。

`number_of_exceptions`

&emsp;&emsp;表示`exception_index_table`表中的记录数。

`exception_index_table[]`

&emsp;&emsp;数组中的每个值必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），即该方法声明的要抛出的异常类型。

&emsp;&emsp;方法要想抛出一个异常，必须至少满足以下三个条件之一：

&emsp;&emsp;- 异常是`RuntimeException`或其子类的实例。

&emsp;&emsp;- 异常是`Error`或其子类的实例。

&emsp;&emsp;- 异常是`exception_index_table`表中给出的异常类或其子类的实例。

&emsp;&emsp;JVM并不强制要求这些条件；它们是编译器要求的。

### 4.7.6 InnerClasses属性

它是`ClassFile`结构体（§4.1）的`attributes`表中的变长属性。

对于一个类或接口*C*的常量池来说，如果它至少包含了一条`CONSTANT_Class_info`记录（§4.4.1），并且该记录表达的类或接口并不属于某个包的成员，那么*C*的`ClassFile`结构体的`attributes`表中必须且只能有一个`InnerClasses`属性。

格式如下：

```
InnerClasses_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_classes;
    {   u2 inner_class_info_index;
        u2 outer_class_info_index;
        u2 inner_name_index;
        u2 inner_class_access_flags;
    } classes[number_of_classes];
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），字符串值为“`InnerClasses`”。

`attribute_length`

&emsp;&emsp;表示该属性的长度，包含开头的六个字节。

`number_of_classes`

&emsp;&emsp;表示`classes`表中的记录数。

`classes[]`

&emsp;&emsp;如果`constant_pool`表中的`CONSTANT_Class_info`记录表达的类或接口*C*非包成员，那么在`classes`数组中它必须且只能有一条对应的记录。

&emsp;&emsp;如果一个类或接口的成员中也有类或接口，它的`constant_pool`表（以及它的`InnerClasses`属性）必须要引用每一个这种成员（JLS §13.1），即便该成员在其他地方没有用到。

&emsp;&emsp;而且，每个被嵌套的类和接口的`constant_pool`表中必须要引用它外层的类，这样把一切归拢到一起来看，每个被嵌套的类和接口的`InnerClasses`信息就包含每一个外层类以及每一个内部的嵌套类或嵌套接口。（这段我有点懵，你去看看原文吧。）

&emsp;&emsp;`classes`数组中的每个记录都包含以下四个字段：

&emsp;&emsp;`inner_class_info_index`

&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Class_info`结果透体，代表*C*。

&emsp;&emsp;`outer_class_info_index`

&emsp;&emsp;&emsp;&emsp;如果*C*不是某个类或接口的成员——即顶层类或接口（JLS §7.6），或局部类（JLS §14.3），或匿名类（JLS §15.9.5）——那么该值必须为零。

&emsp;&emsp;&emsp;&emsp;否则，该值必须是`constant_pool`表的有效索引，对应记录必须是一个`CONSTANT_Class_info`结构体，代表一个类或接口，而*C*是它的一个成员。`outer_class_info_index`不能等于`inner_class_info_index`。

&emsp;&emsp;`inner_name_index`

&emsp;&emsp;&emsp;&emsp;如果*C*是匿名的（JLS §15.9.5），该值必须是零。

&emsp;&emsp;&emsp;&emsp;否则，该值必须是`constant_pool`表的有效索引，对应记录必须是一个`CONSTANT_Utf8_info`结构体，代表*C*的原始基本名（译者注：original simple name），就跟该`class`文件对应的源码中名字的一样。

&emsp;&emsp;`inner_class_access_flags`

&emsp;&emsp;&emsp;&emsp;它是一个标记掩码，用来标记类或接口*C*的访问权限和属性，也就是`class`文件对应源码中声明的那样。在源码不可用的时候，编译器可以用它们来恢复原始信息。各种标记见表4.7.6-A。

&emsp;&emsp;&emsp;&emsp;**表4.7.6-A 嵌套类的访问和属性标记**

|**标记名**|**值**|**解释**
|-|-|-
|`ACC_PUBLIC`|0x0001|在源码中主动标记或隐式的`public`。
|`ACC_PRIVATE`|0x0002|在源码中标记`private`。
|`ACC_PROTECTED`|0x0004|在源码中标记`protected`。
|`ACC_STATIC`|0x0008|在源码中主动标记或隐式的`static`。
|`ACC_FINAL`|0x0010|在源码中主动标记或隐式的`final`。
|`ACC_INTERFACE`|0x0200|在源码中是一个`interface`。
|`ACC_ABSTRACT`|0x0400|在源码中主动标记或隐式的`abstract`。
|`ACC_SYNTHETIC`|0x1000|复合声明；源码中看不出来。
|`ACC_ANNOTATION`|0x2000|注解接口声明。
|`ACC_ENUM`|0x4000|`enum`类声明。

&emsp;&emsp;&emsp;&emsp;表中没有给出的比特位留着以后用。生成`class`文件的时候这些位置都应该置零，而且JVM实现也应该忽略它们。

&emsp;&emsp;如果`class`文件的版本号大于等于51.0，并且`attributes`表中有一个`InnerClasses`属性，那么对于这个`InnerClasses`属性来说，它的`classes`表中的所有记录，如果`inner_name_index`是零，`outer_class_info_index`就也得是零。

&emsp;&emsp;Oracle的JVM并不会检查`InnerClasses`属性和它所引用的类或接口的`class`文件的一致性。

### 4.7.7 EnclosingMethod属性

这个属性是`ClassFile`结构体（§4.1）的`attributes`表中的一个定长属性。一个类必须要有一个`EnclosingMethod`属性，当且仅当它表达的是一个局部类或匿名类（JLS §14.3, JLS
§15.9.5）。

在一个`ClassFile`结构体的`attributes`表中最多只能有一个`EnclosingMethod`属性。

该属性格式如下：

```
EnclosingMethod_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 class_index;
    u2 method_index;
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串“`EnclosingMethod`”。

`attribute_length`

&emsp;&emsp;这个值必须等于四。

`class_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），代表包裹当前类最近的那个类。

`method_index`

&emsp;&emsp;如果当前类不是被一个方法或构造方法立即包裹住，那么这个值必须是零。

>特别的，如果当前类在源码中被实例初始化器、静态初始化器、实例变量初始化器或类变量初始化器立即包裹住，那么该值必须是零。（前两个考虑的是局部类和匿名类，后两个考虑的是在字段赋值的时候表达式右边声明的那种匿名类。）

&emsp;&emsp;否则，这个值必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_NameAndType_info`结构体（§4.4.6），代表上面的`class_index`属性引用的类中的一个方法的名字和类型。

>要保证`method_index`标识的方法的确是包裹当前类最近的那个方法，这个事儿是编译器的责任。

### 4.7.8 Synthetic属性

该属性属于`ClassFile`、`field_info`、`method_info`结构体（§4.1, §4.5, §4.6）的`attributes`表，是一个定长属性。如果一个类的成员没有出现在源码中，那就必须要用`Synthetic`属性来标记，或者必须要设置`ACC_SYNTHETIC`标记。唯一的特例是那些编译器生成的成员，且在实现物中不做考虑，例如：

- 一个实例初始化方法，代表Java语言中的默认构造器（§2.9.1）
- 一个类或接口的初始化方法（§2.9.2）
- 枚举和record类中隐式声明的成员（JLS §8.9.3, JLS §8.10.3）

>`Synthetic`属性是打JDK1.1引入的，用来支持嵌套类和嵌套接口。

在`class`文件中有一处限制就是说只有形参和模块可以打`ACC_MANDATED`标记（§4.7.24, §4.7.25），这样的意思就是尽管是编译器生成的，但它们也不在实现物的考虑范围之内。其他编译器生成的玩意儿是没有办法对其进行标记从而不做实现物考虑的（JLS §13.1）。这种限制就意味着JavaSE中的反射API可能无法准确的给出这种玩意儿的“被mandated”的状态。

该属性结构如下：

```
Synthetic_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`Synthetic`”。

`attribute_length`

&emsp;&emsp;必须是零。

### 4.7.9 Signature属性

它是`ClassFile`、`field_info`、`method_info`、`record_component_info`结构体（§4.1, §4.5, §4.6, §4.7.30）`attributes`表中的一个定长属性。它为类、接口、构造器、方法、字段或record成分保存了一个签名信息，而且它们在Java语言中声明的时候都用了类型变量或参数化类型。关于这些结构的知识可以去看*Java语言规范，JavaSE 第17版*。

在一个`ClassFile`、`field_info`、`method_info`、`record_component_info`结构体中，`attributes`表中最多只能有一个`Signature`属性。

格式如下：

```
Signature_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 signature_index;
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`Signature`”。

`attribute_length`

&emsp;&emsp;必须是二。

`signature_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），如果该`Signature`属性是一个`ClassFile`结构体的属性，那它就代表一个类的签名；如果是`method_info`结构体的属性，那它就代表一个方法签名；或者就是字段签名。

>Oracle的JVM在类加载或链接的过程中不会去检查`Signature`属性结构是否完好。它的检查是JavaSE平台类库中那些暴露泛型类、接口、构造器、方法、字段签名的方法去检查的。比如`Class`中的`getGenericSuperclass`和`java.lang.reflect.Executable`中的`toGenericString`。

#### 4.7.9.1 *签名*

*签名*可以对Java语言中使用JVM类型体系之外的类型声明进行编码。支持反射和调试，以及只有`class`文件时的编译场景。

对于每一个类、接口、构造器、方法、字段或record成分，在声明中使用了类型变量或参数化类型，Java编译器必须要给出一个签名。特别是，一个Java编译器必须要给出：

- 类签名。对于任意类或接口声明，如果是泛型的，或者是有参数化类型作为父类或父接口，或者两者兼有，此时必须要给出类签名。
- 方法签名。对于任意方法或构造器声明，如果是泛型的，或者是有类型参数或参数化类型作为返回类型或形参类型的，又或者是在`throws`语法中带类型变量的，又或者是这些情况的任意组合，此时必须要给出方法签名。
- 字段签名。对于任意字段、形参、局部变量或record成分声明，如果类型中使用了类型变量或参数化类型，此时必须要给出字段签名。

签名使用§4.3.1节讲的标记语法进行声明。另外：

- 在一个产生式的右边，*[x]*语法表示x出现零次或一次。意即，x是一个*可选符号*。如果包含了可选符号，实际上就定义了两种情况：带和不带。
- 如果右边特别长，可以在下一行继续写，下一行会有明确的缩进。

语法中包含了终结符*标识符*，表示由Java编译器生成的类型、字段、方法、形参、局部变量或类型变量的名字。这种名字不能包含任何ASCII字符`.;[/<>:`（也就是说，方法名中禁止的字符（§4.2.2）再加上冒号）但是可以包含Java语言标识符中禁止出现的字符（JLS §3.8）。

签名是一种多层非终结符，即*类型签名*：

- 一个*Java类型签名*代表Java语言中的一个引用类型或基本类型。

&emsp;&emsp;&emsp;&emsp;*JavaTypeSignature:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*ReferenceTypeSignature*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*BaseType*<br/>

<sub>&emsp;&emsp;&emsp;&emsp;为了方便参考，我们把§4.3.2节的产生式搬过来了：</sub>

<sub>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;<i>BaseType:</i></sub><br/>
<sub>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;<i>(one of)</i></sub><br/>
<sub>&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;<code>B C D F I J S Z</code></sub>

- 一个*引用类型签名*代表一个Java语言中的引用类型，也就是一个类或一个接口、类型变量或一个数组类型。

&emsp;&emsp;一个*类类型签名*代表一个（可能是参数化的）类或接口类型。一个类类型签名必须得规划好了，这样才能在擦除类型参数并将每个`.`字符转换成`$`字符后可靠的映射到类的二进制名上。

&emsp;&emsp;一个*类型变量签名*代表一个类型变量。

&emsp;&emsp;一个*数组类型签名*代表一个数组类型中的一维。

&emsp;&emsp;&emsp;&emsp;*ReferenceTypeSignature:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*ClassTypeSignature*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*TypeVariableSignature*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*ArrayTypeSignature*

&emsp;&emsp;&emsp;&emsp;*ClassTypeSignature:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`L` *[PackageSpecifier]*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;&nbsp;&nbsp;*SimpleClassTypeSignature {ClassTypeSignatureSuffix} ;*

&emsp;&emsp;&emsp;&emsp;*PackageSpecifier:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*Identifier / {PackageSpecifier}*

&emsp;&emsp;&emsp;&emsp;*SimpleClassTypeSignature:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*Identifier [TypeArguments]*

&emsp;&emsp;&emsp;&emsp;*TypeArguments:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`<` *TypeArgument {TypeArgument}* `>`

&emsp;&emsp;&emsp;&emsp;*TypeArgument:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;*[WildcardIndicator] ReferenceTypeSignature*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`*`

&emsp;&emsp;&emsp;&emsp;*WildcardIndicator:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`+`<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`-`

&emsp;&emsp;&emsp;&emsp;*ClassTypeSignatureSuffix:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`.` *SimpleClassTypeSignature*

&emsp;&emsp;&emsp;&emsp;*TypeVariableSignature:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`T` *Identifier ;*

&emsp;&emsp;&emsp;&emsp;*ArrayTypeSignature:*<br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;`[` *JavaTypeSignature*

一个*类签名*编码了一个（可能是泛型的）类或接口声明的类型信息。它描述了类或接口中的所有类型参数，并且列出了它的（可能是参数化的）直接父类和直接父接口，如果有的话。一个类型参数是用它的名字加类边界或接口边界进行描述的。

&emsp;&emsp;*ClassSignature:*<br/>
&emsp;&emsp;&nbsp;&nbsp;*[TypeParameters] SuperclassSignature {SuperinterfaceSignature}*

&emsp;&emsp;*TypeParameters:*<br/>
&emsp;&emsp;&nbsp;&nbsp;`<` *TypeParameter {TypeParameter}* `>`

&emsp;&emsp;*TypeParameter:*<br/>
&emsp;&emsp;&nbsp;&nbsp;*Identifier ClassBound {InterfaceBound}*

&emsp;&emsp;*ClassBound:*<br/>
&emsp;&emsp;&nbsp;&nbsp;`:` *[ReferenceTypeSignature]*

&emsp;&emsp;*InterfaceBound:*<br/>
&emsp;&emsp;&nbsp;&nbsp;`:` *ReferenceTypeSignature*

&emsp;&emsp;*SuperclassSignature:*<br/>
&emsp;&emsp;&nbsp;&nbsp;*ClassTypeSignature*

&emsp;&emsp;*SuperinterfaceSignature:*<br/>
&emsp;&emsp;&nbsp;&nbsp;*ClassTypeSignature*

一个*方法签名*编码了（可能是泛型的）方法声明相关的类型信息。它描述了方法的所有类型参数；（可能是参数化的）形参类型；（可能是参数化的）返回类型，如果有的话；以及`throws`声明的所有异常的类型。

&emsp;&emsp;*MethodSignature:*<br/>
&emsp;&emsp;&nbsp;&nbsp;*[TypeParameters] ( {JavaTypeSignature} ) Result {ThrowsSignature}*

&emsp;&emsp;*Result:*<br/>
&emsp;&emsp;&nbsp;&nbsp;*JavaTypeSignature*<br/>
&emsp;&emsp;&nbsp;&nbsp;*VoidDescriptor*

&emsp;&emsp;*ThrowsSignature:*<br/>
&emsp;&emsp;&nbsp;&nbsp;^ *ClassTypeSignature*<br/>
&emsp;&emsp;&nbsp;&nbsp;^ *TypeVariableSignature*

&emsp;&emsp;<sub>下面是§4.3.3节的产生式拿过来看着方便一些：</sub><br/>

&emsp;&emsp;&emsp;&emsp;<sub>*VoidDescriptor:*</sub><br/>
&emsp;&emsp;&emsp;&emsp;&nbsp;&nbsp;<sub>`V`</sub>

&emsp;&emsp;<sub>用`Signature`编码的方法签名可能没法跟`method_info`结构体（§4.3.3）中的方法描述符准确的对应上。特别是，无法保证方法签名中的形参类型数量和方法描述符中的参数描述符的数量相等。这俩数在大部分方法中都是相等的，但是某些Java语言中的构造器是带隐式声明参数的，编译器会用参数描述符来进行表达，但方法签名中却没有这个东西。§4.7.18中讲的参数注解就有这种情况。</sub>

一个*字段签名*编码了（可能是参数化的）字段类型、形参、局部变量或record成分声明。

&emsp;&emsp;*FieldSignature:*<br/>
&emsp;&emsp;*ReferenceTypeSignature*

### 4.7.10 SourceFile属性

它是`ClassFile`结构体（§4.1）的`attribtues`表中的一个可选的定长属性。

在一个`ClassFile`结构体的`attributes`表中，最多只能有一个`SourceFile`属性。

格式如下：

```
SourceFile_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 sourcefile_index;
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应值必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`SourceFile`”。

`attribute_length`

&emsp;&emsp;必须是二。

`sourcefile_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应的记录必须是一个`CONSTANT_Utf8_info`结构体，代表一个字符串。

&emsp;&emsp;<sub>`sourcefile_index`所引用的字符串会被解释为该`class`文件对应源文件的名字。它可不是包含文件的文件夹名或者是该文件的绝对路径名；在真正要用这些文件名的时候，这种平台相关的额外信息必须要由运行时解释器或开发工具提供出来。</sub>

### 4.7.11 SourceDebugExtension属性

它是`ClassFile`结构体（§4.1）的`attributes`表中的一个可选属性。

在一个`ClassFile`结构体的`attributes`表中，最多只能有一个`SourceDebugExtension`属性。

格式如下：

```
SourceDebugExtension_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 debug_extension[attribute_length];
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`SourceDebugExtension`”。

`attribute_length`

&emsp;&emsp;代表属性的长度，不包括开头的六个字节。

`debug_extension[]`

&emsp;&emsp;保存了扩展的调试信息，这些信息对JVM来说没有语义上的影响。这些信息通过修正的UTF-8字符串（§4.4.7）来表达，结尾不带零字节。

&emsp;&emsp;<sub>注意，它代表的字符串长度可能超出了一个`String`类实例所能表达的长度。</sub>

### 4.7.12 LineNumberTable属性

它是`Code`属性（§4.7.3）的`attributes`表中的一个可选的变长属性。调试器可以用它来判断`code`数组中的某一部分对应源码文件中的行号是多少。

如果`Code`属性的`attributes`表中出现了多个`LineNumberTable`属性，它们的顺序是不固定的。

在一个`Code`属性的`attribtues`表中，*源码文件的每一行*可能对应多个`LineNumberTable`属性。也就是说多个该属性可能才能表达源码文件中的一行，和源码文件不需要一行对一个。

格式如下：

```
LineNumberTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 line_number_table_length;
    {   u2 start_pc;
        u2 line_number;
    } line_number_table[line_number_table_length];
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应值必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`LineNumberTable`”。

`attribute_length`

&emsp;&emsp;属性的长度，不包含开头的六个字节。

`line_number_table_length`

&emsp;&emsp;代表`line_number_table`表中的记录数量。

`line_number_table[]`

&emsp;&emsp;每条记录表示源码中的行号在`code`数组中某个位置上发生了变化。每个`line_number_table`记录必须包含以下两个属性：

&emsp;&emsp;`start_pc`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`Code`属性的`code`数组中的有效索引。该索引指定的`code`数组的位置表示对应源码中新的一行开始了。

&emsp;&emsp;`line_number`<br/>
&emsp;&emsp;&emsp;&emsp;给出对应的源码行号。

### 4.7.13 LocalVariableTable属性

这玩意是`Code`属性（§4.7.3）的`attributes`表的一个可选的变长属性。调试器可以用它在方法执行过程中判断某个局部变量的值。

如果一个`Code`属性的`attributes`表中出现了多个`LocalVariableTable`属性，它们的顺序是不固定的。

在一个`Code`属性的`attributes`表中，*每个局部变量*最多只能有一个*LocalVariableTable*属性。

格式如下：

```
LocalVariableTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_table_length;
    {   u2 start_pc;
        u2 length;
        u2 name_index;
        u2 descriptor_index;
        u2 index;
    } local_variable_table[local_variable_table_length];
}
```

解释如下：

`attribute_name_index`

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`LocalVariableTable`”。

`attribute_length`

&emsp;&emsp;表示该属性的长度，不包括开头的六个字节。

`local_variable_table_length`

&emsp;&emsp;表示`local_variable_table`数组的长度。

`local_variable_table[]`

&emsp;&emsp;表中每个记录代表一个`code`数组的偏移量范围，该范围对应了某个局部变量的值，还代表该局部变量在当前帧的局部变量数组中的索引位置。表中的每个记录必须包含以下五个属性：

&emsp;&emsp;`start_pc`、`length`<br/>
&emsp;&emsp;&emsp;&emsp;`start_pc`的值必须是该`Code`属性的`code`数组的一个有效索引，而且必须是一条指令的操作码索引。

&emsp;&emsp;&emsp;&emsp;`start_pc + length`的值必须要么是该`Code`属性的`code`数组的有效索引，要么是一条指令的操作码索引，再要么必须是超出`code`数组末尾的第一个索引。

&emsp;&emsp;&emsp;&emsp;`start_pc`和`length`属性表示某个局部变量在`code`数组的[`start_pc`, `start_pc + length`)区间内的某个索引位置上有一个值，区间是左闭右开的。

&emsp;&emsp;`name_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体，代表一个局部变量的非限定名（§4.2.2）。

&emsp;&emsp;`descriptor_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体，代表一个字段描述符，该字段描述符编码了源程序中的一个局部变量的类型（§4.2.2）。

&emsp;&emsp;`index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是当前帧的局部变量表中的有效索引。对应索引处有一个局部变量。呦呵，反正话啊，这可是相声的经典表演。

&emsp;&emsp;&emsp;&emsp;如果指定的局部变量类型是`double`和`long`，那它要占用`index`和`index + 1`。

### 4.7.14 LocalVariableTypeTable属性

该属性是`Code`属性（§4.7.3）的`attributes`表中的一个可选的变长属性。调试器可以在方法执行过程中使用该属性来判断局部变量的值。

如果在`Code`属性的`attributes`表中出现了多个`LocalVariableTypeTable`属性，它们的顺序是不固定的。

在一个`Code`属性的`attributes`表中，*每个局部变量*不能有多个`LocalVariableTypeTable`属性。

&emsp;&emsp;<sub>`LocalVariableTypeTable`跟`LocalVariableTable`（§4.7.13）不一样，它给的是签名信息而非描述符信息。这种不同只对于那些使用了类型变量或参数化类型的变量才显得比较重要。这种变量可能在这俩表中都会出现，而其它变量只会出现在`LocalVariableTable`中。</sub>

格式如下：

```
LocalVariableTypeTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_type_table_length;
    {   u2 start_pc;
        u2 length;
        u2 name_index;
        u2 signature_index;
        u2 index;
    } local_variable_type_table[local_variable_type_table_length];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`LocalVariableTypeTable`”。

`attribute_length`<br/>
&emsp;&emsp;代表属性的长度，不包括开头的六个字节。

`local_variable_type_table_length`<br/>
&emsp;&emsp;代表`local_variable_type_table`数组中记录的数量。

`local_variable_type_table[]`<br/>
&emsp;&emsp;该数组中的每条记录代表一个`code`数组中的偏移量区间，该区间对应一个局部变量的值，还代表该局部变量在当前帧的局部变量数组中的索引位置。表中的每个记录必须包含以下五个属性：

&emsp;&emsp;`start_pc`、`length`<br/>
&emsp;&emsp;&emsp;&emsp;必须是该`Code`属性的`attributes`表的有效索引，而且必须是一个指令操作码的索引。

&emsp;&emsp;&emsp;&emsp;`start_pc + length`的值必须要么是该`Code`属性的`code`数组的有效索引，要么是一条指令的操作码索引，再要么必须是超出`code`数组末尾的第一个索引。

&emsp;&emsp;&emsp;&emsp;`start_pc`和`length`属性表示某个局部变量在`code`数组的[`start_pc`, `start_pc + length`)区间内的某个索引位置上有一个值，区间是左闭右开的。

&emsp;&emsp;`name_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体，代表一个局部变量的非限定名（§4.2.2）。

&emsp;&emsp;`signature_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体，代表一个字段签名，该字段签名编码了源程序中的一个局部变量的类型（§4.7.9.1）。

&emsp;&emsp;`index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是当前帧的局部变量表中的有效索引。对应索引处有一个局部变量。呦呵，反正话啊，这可是相声的经典表演。

&emsp;&emsp;&emsp;&emsp;如果指定的局部变量类型是`double`和`long`，那它要占用`index`和`index + 1`。

### 4.7.15 Deprecated属性

它是`ClassFile`、`field_info`、`method_info`结构体（§4.1, §4.5, §4.6）的`attributes`表中的一个可选的定长属性。如果一个类、接口、方法或字段用`Deprecated`属性做了标记，说明这个类、接口、方法或字段已经废了。

能够读取`class`文件格式的运行时解释器或其他工具，比如一个编译器吧，可以使用这个标记告诉用户他引用了一个废弃的类、接口、方法或字段。`Deprecated`属性的出现不会对类或接口产生语义上的影响。

格式如下：

```
Deprecated_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`Deprecated`”。

`attribute_length`<br/>
&emsp;&emsp;必须是零。

### 4.7.16 RuntimeVisbleAnnotations属性

它是`ClassFile`、`field_info`、`method_info`、或`record_component_info`结构体（§4.1, §4.5, §4.6, §4.7.30）的`attributes`表中的一个变长属性。该属性可以保存类、字段、方法或record成分声明上的那些运行时可见的注解。

在一个`ClassFile`、`field_info`、`method_info`或`record_component_info`结构体的`attributes`表中，最多只能有一个`RuntimeVisibleAnnotations`属性。

格式如下：

```
RuntimeVisibleAnnotations_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 num_annotations;
    annotation annotations[num_annotations];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`RuntimeVisbleAnnotations`”。

`attribute_length`<br/>
&emsp;&emsp;代表属性的长度，不包括开头的六个字节。

`num_annotations`<br/>
&emsp;&emsp;给出了该结构体表达的运行时可见注解的数量。

`annotations[]`<br/>
&emsp;&emsp;表中的每条记录代表某个声明上的一个运行时可见注解。`annotation`结构体的格式如下：

```
    annotation {
        u2 type_index;
        u2 num_element_value_pairs;
        {   u2 element_name_index;
            element_value value;
        } element_value_pairs[num_element_value_pairs];
    }
```

&emsp;&emsp;解释如下：

&emsp;&emsp;`type_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表一个字段描述符（§4.3.2）。该字段描述符指明了这个`annotation`结构体所表达的注解的类型。

&emsp;&emsp;`num_element_value_pairs`<br/>
&emsp;&emsp;&emsp;&emsp;给出了该`annotation`结构体所表达的注解中的元素键值对的个数。

&emsp;&emsp;`element_value_pairs[]`<br/>
&emsp;&emsp;&emsp;&emsp;其中的每个值代表该`annotation`结构体所表达的注解中的一个元素键值对。每个记录包含以下两个属性：

&emsp;&emsp;&emsp;&emsp;`element_name_index`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7）。给出了当前元素键值对的名字，也就是键。

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;<sub>给出了名字，其实也就是给出了元素</sub>

&emsp;&emsp;&emsp;&emsp;`value`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;该记录所表达的元素键值对的值。

#### 4.7.16.1 element_value结构

它是一个可区分共用体，代表一个元素键值对的值。格式如下：

```
element_value {
    u1 tag;
    union {
        u2 const_value_index;
        
        {   u2 type_name_index;
            u2 const_name_index;
        } enum_const_value;

        u2 class_info_index;

        annotation annotation_value;
        
        {   u2 num_values;
            element_value values[num_values];
        } array_value;
    } value;
}
```

`tag`是一个ASCII字符，代表这个值的类型。它决定了`value`共用体中要使用哪个元素。表4.7.16.1-A给出了`tag`的有效值，以及每个值代表的类型，以及对应生效的共用体中的元素。表中第四列是共用体中某个属性生效的时候才用到，下面会讲。

**表4.7.16.1-A `tag`值的解释**

|`tag`**值**|**类型**|`value`**项**|**常量类型**
|-|-|-|-
|`B`|`byte`|`const_value_index`|`CONSTANT_Integer`
|`C`|`char`|`const_value_index`|`CONSTANT_Integer`
|`D`|`double`|`const_value_index`|`CONSTANT_Double`
|`F`|`float`|`const_value_index`|`CONSTANT_Float`
|`I`|`int`|`const_value_index`|`CONSTANT_Integer`
|`J`|`long`|`const_value_index`|`CONSTANT_Long`
|`S`|`short`|`const_value_index`|`CONSTANT_Integer`
|`Z`|`boolean`|`const_value_index`|`CONSTANT_Integer`
|`s`|`String`|`const_value_index`|`CONSTANT_Utf8`
|`e`|枚举类|`enum_const_value`|*不适用*
|`c`|`Class`|`class_info_index`|*不适用*
|`@`|注解接口|`annotation_value`|*不适用*
|`[`|数组类型|`array_value`|*不适用*

*value*项代表元素键值对的值。它是一个共用体，解释如下：

`constant_value_index`<br/>
&emsp;&emsp;代表一个基本类型或`String`类型的常量，作为元素键值对的值。

&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录的类型必须适用于`tag`所表达的类型，也就是表的第四列。

`enum_const_value`<br/>
&emsp;&emsp;说明这个元素键值对的值是一个枚举常量。

&emsp;&emsp;由两个属性组成：

&emsp;&emsp;`type_name_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表一个字段描述符（§4.3.2）。`constant_pool`表的对应记录给出了这个`element_value`结构体表达的枚举常量类型的二进制名的内部形式（§4.2.1）。

&emsp;&emsp;`constant_name_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7）。`constant_pool`表的对应记录给出了这个`element_value`结构体表达的枚举常量的一般名称。

`class_info_index`<br/>
&emsp;&emsp;该属性表示这个元素键值对的值是一个类字面量。

&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表一个返回描述符（§4.3.3）。这个返回描述符给出了该`element_value`结构体表达的类字面量的类型。类型和类字面量对应方式如下：

<ul>
<li>对于一个类字面量<code><i>C</i>.class</code>，<i>C</i>是一个类、接口、或数组类型的名字，此时对应的类型就是<i>C</i>。在<code>constant_pool</code>中的返回描述符则是<i>ObjectType</i>或<i>ArrayType</i>。</li>
<li>对于一个类字面量<code><i>p</i>.class</code>，<i>p</i>是一个基本类型的名字，此时对应的类型就是<i>p</i>。在<code>constant_pool</code>中的返回描述符则是一个<i>BaseType</i>字符。</li>
<li>对于一个类字面量<code>void.class</code>，对应的类型就是<code>void</code>。在<code>constant_pool</code>中的返回描述符则是<i>V</i></li>
</ul>

&emsp;&emsp;<sub>比如`Object.class`类字面量对应的类型就是`Object`，对应的`constant_pool`记录是`Ljava/lang/Object;`，而`int.class`类字面量对应的是类型`int`，`constant_pool`记录是`I`。</sub>

&emsp;&emsp;<sub>类字面量`void.class`对应`void`，`constant_pool`记录是*V*，然而`Void.class`类字面量对应类型`Void`，所以`constant_pool`记录是`Ljava/lang/Void;`。</sub>

`annotation_value`<br/>
&emsp;&emsp;表示这个元素键值对的值是一个“嵌套的”注解。

&emsp;&emsp;它的值是一个`annotation`结构体（§4.7.16），给出这个`element_value`结构体所表达的注解。

`array_value`<br/>
&emsp;&emsp;表示这个元素键值对的值是一个数组。

&emsp;&emsp;包含以下两个属性：

&emsp;&emsp;`num_values`<br/>
&emsp;&emsp;&emsp;&emsp;表示该`element_value`结构体表达的数组中的元素个数。

&emsp;&emsp;`values[]`<br/>
&emsp;&emsp;&emsp;&emsp;`values`表中的每个值给出了该`element_value`结构体所表达的数组中对应的元素。

### 4.7.17 RuntimeInvisibleAnnotations属性

它是`ClassFile`、`field_info`、`method_info`或`record_component_info`结构体（§4.1, §4.5, §4.6, §4.7.30）的`attributes`表中的一个变长属性。该属性保存了类、方法、字段或record成分声明上的运行时不可见的注解。

在一个`ClassFile`、`field_info`、`method_info`、或`record_component_info`结构体的`attributes`表中最多只能有一个`RuntimeInvisibleAnnotations`属性。

&emsp;&emsp;<sub>该属性跟`RuntimeVisibleAnnotations`属性（§4.7.16）类似，只不过该属性表达的注解不能通过反射API返回，除非JVM实现时通过某种特定的机制把这些注解保留下来了，比如某种命令行选项。如果没有此类指令的话，JVM要忽略这个属性。</sub>

格式如下：

```
RuntimeInvisibleAnnotations_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 num_annotations;
    annotation annotations[num_annotations];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`RuntimeInvisibleAnnotations`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`num_annotations`<br/>
&emsp;&emsp;该结构体所表达的运行时不可见注解的数量。

`annotations[]`<br/>
&emsp;&emsp;表中每条记录都是某个声明上的一个运行时不可见注解。`annotation`结构体见§4.7.16。

### 4.7.18 RuntimeVisibleParameterAnnotations属性

它是`method_info`结构体（§4.6）的`attributes`表中的一个变长属性。该属性保存了对应方法的形参声明上的运行时可见的注解。

在一个`method_info`结构体的`attributes`表中最多只能有一个`RuntimeVisibleParameterAnnotations`属性。

格式如下：

```
RuntimeVisibleParameterAnnotations_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 num_parameters;
    {   u2 num_annotations;
        annotation annotations[num_annotations];
    } parameter_annotations[num_parameters];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`RuntimeVisibleParameterAnnotations`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`num_parameters`<br/>
&emsp;&emsp;代表该结构体所表达的运行时可见的参数注解的数量。

&emsp;&emsp;<sub>不保证这个数量跟方法描述符中的参数描述符的数量相等。</sub>

`parameter_annotations[]`<br/>
表中的每条记录代表一处形参声明上的所有运行时可见的注解。每个记录包含以下两个属性：

&emsp;&emsp;`num_annotations`<br/>
&emsp;&emsp;&emsp;&emsp;给出该记录包含的对应形参声明上的运行时可见注解的数量。

&emsp;&emsp;`annotations[]`<br/>
&emsp;&emsp;&emsp;&emsp;表中的每条记录代表对应形参声明上的一个运行时可见的注解。`annotation`结构体见§4.7.16。

&emsp;&emsp;表中的第*i*条记录可以但并不强制跟方法描述符（§4.3.3）中的第*i*个参数描述符对应。

&emsp;&emsp;<sub>比如，编译器可能只会为那些用来表达在源码中显式声明的参数的参数描述符创建该表对应的记录。在Java语言中，一个内部类的构造器要求在显式声明的参数之前要有一个隐式声明的参数（JLS §8.8.1），这样，在`class`文件中对应的`<init>`方法在所有显式声明参数的参数描述符之前会有一个参数描述符用来表达这个隐式声明的参数。如果第一个显式声明的参数在源码中加了注解，那么编译器可能用`parameter_annotations[0]`来保存*第二个*参数描述符对应的注解。</sub>

### 4.7.19 RuntimeInvisibleParameterAnnotations属性

它是`method_info`结构体（§4.6）的`attributes`表中的一个变长属性。它保存了对应方法的形参声明上的运行时不可见的注解。

在一个`method_info`结构体的`attributes`表中最多只能有一个`RuntimeInvisibleParameterAnnotations`属性。

&emsp;&emsp;<sub>这玩意跟`RuntimeVisibleParameterAnnotations`属性类似（§4.7.18），只不过该属性表达的注解不能通过反射API返回，除非JVM实现时通过某种特定的机制把这些注解保留下来了，比如某种命令行选项。如果没有此类指令的话，JVM要忽略这个属性。</sub>

格式如下：

```
RuntimeInvisibleParameterAnnotations_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 num_parameters;
    {   u2 num_annotations;
        annotation annotations[num_annotations];
    } parameter_annotations[num_parameters];
}
```

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`RuntimeInvisibleParameterAnnotations`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`num_parameters`<br/>
&emsp;&emsp;代表该结构体所表达的运行时不可见的参数注解的数量。

&emsp;&emsp;<sub>不保证这个数量跟方法描述符中的参数描述符的数量相等。</sub>

`parameter_annotations[]`<br/>
表中的每条记录代表一处形参声明上的所有运行时不可见的注解。每个记录包含以下两个属性：

&emsp;&emsp;`num_annotations`<br/>
&emsp;&emsp;&emsp;&emsp;给出该记录包含的对应形参声明上的运行时不可见注解的数量。

&emsp;&emsp;`annotations[]`<br/>
&emsp;&emsp;&emsp;&emsp;表中的每条记录代表对应形参声明上的一个运行时不可见的注解。`annotation`结构体见§4.7.16。

&emsp;&emsp;表中的第*i*条记录可以但并不强制跟方法描述符（§4.3.3）中的第*i*个参数描述符对应。

&emsp;&emsp;<sub>§4.7.18节中给出了一个例子，其中`parameter_annotations[0]`和方法描述符中的第一个参数描述符就不是对应的。</sub>

### 4.7.20 RuntimeVisibleTypeAnnotations属性

它是`ClassFile`、`field_info`、`method_info`、`record_component_info`、`Code`结构体（§4.1, §4.5, §4.6, §4.7.30, §4.7.3）的`attributes`表中的一个变长属性。该属性保存了类、字段、方法或record成分声明上的，或对应方法体中某个表达式上的运行时可见的类型注解。该属性还保存了泛型类、接口、方法和构造器的类型参数声明上的运行时可见的注解。

在一个`ClassFile`、`field_info`、`method_info`或`record_component_info`结构体或`Code`属性的`attributes`表中最多只能有一个`RuntimeVisibleTypeAnnotations`属性。

如果声明或表达式中的类型加了注解，只有跟父级的结构体或`attributes`表中的属性对应起来，才会被加入到`attributes`表的`RuntimeVisibleTypeAnnotations`属性中。（很难理解是不是？看下面）。

&emsp;&emsp;<sub>比如类声明时的`implements`语法中所有类型的注解都记录在该类的`ClassFile`结构体的`RuntimeVisibleTypeAnnotations`属性中。但是，某个字段声明的类型注解则是记录在该字段的`field_info`结构体的`RuntimeVisibleTypeAnnotations`属性中。</sub>

格式如下：

```
RuntimeVisibleTypeAnnotations_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 num_annotations;
    type_annotation annotations[num_annotations];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体，代表字符串值“`RuntimeVisibleTypeAnnotations`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`num_annotations`<br/>
&emsp;&emsp;该结构体所表达的运行时可见的类型注解的数量。

`annotations[]`<br/>
&emsp;&emsp;表中每条记录都是某个声明或表达式中的一个运行时可见的类型注解。`type_annotation`结构体格式如下：

```
type_annotation {
    u1 target_type;
    union {
        type_parameter_target;
        supertype_target;
        type_parameter_bound_target;
        empty_target;
        formal_parameter_target;
        throws_target;
        localvar_target;
        catch_target;
        offset_target;
        type_argument_target;
    } target_info;
    type_path target_path;
    u2 type_index;
    u2 num_element_value_pairs;
    {   u2 element_name_index;
        element_value value;
    } element_value_pairs[num_element_value_pairs];
}
```

&emsp;&emsp;头三个属性`target_type`、`target_info`、`target_path`给出了这个被注解类型的准确定位。后三个属性`type_index`、`num_element_value_pairs`、`element_value_pairs[]`给出了注解自身的类型以及元素键值对。

&emsp;&emsp;`type_annotation`结构体解释如下：

&emsp;&emsp;`target_type`<br/>
&emsp;&emsp;&emsp;&emsp;给出了该注解出现在了哪种目标上。这里的各种目标对应的*类型上下文*是Java语言中声明和表达式中用到的各种类型（JLS §4.11）。

&emsp;&emsp;&emsp;&emsp;`target_type`合法的值见表4.7.20-A和表4.7.20-B。每个值都是一个单字节标记，指明了紧跟着的`target_info`中由哪个属性负责对目标进行详细描述。

&emsp;&emsp;&emsp;&emsp;<sub>表4.7.20-A和表4.7.20-B中列出的目标种类对应了JLS §4.11中的目标上下文。也就是说，`target_type`的0x10-0x17和0x40-0x42对应类型上下文1-10，`target_type`的0x43-0x4B对应类型上下文11-16。</sub>

&emsp;&emsp;&emsp;&emsp;`target_type`属性的值决定了`ClassFile`、`field_info`、`method_info`结构体或`Code`属性中的`RuntimeVisibleTypeAnnotations`属性中是否会出现`type_annotation`结构体。表4.7.20-C中给出了`type_annotaion`的`target_type`值合法时的`RuntimeVisibleTypeAnnotations`属性的位置。

&emsp;&emsp;`target_info`<br/>
&emsp;&emsp;&emsp;&emsp;这个值准确的给出了一个声明或表达式中的哪个类型被加了注解了。

&emsp;&emsp;&emsp;&emsp;该共用体的属性见§4.7.20.1。

&emsp;&emsp;`target_path`<br/>
&emsp;&emsp;&emsp;&emsp;准确的指明了`target_info`指定的类型中的哪一部分被注解了。

&emsp;&emsp;&emsp;&emsp;`target_path`结构体格式详见§4.7.20.2。

&emsp;&emsp;`type_index`、`num_element_value_pairs`、`element_value_pairs[]`<br/>
&emsp;&emsp;&emsp;&emsp;这几个属性在`type_annotation`结构体中的含义跟它们在`annotation`结构体中的含义是一样的（§4.7.16）。

**表4.7.20-A `target_type`释义（第1部分）**

|**值**|**目标种类**|`target_info`**属性**
|-|-|-
|0x00|<sub>泛型类或接口的类型参数声明</sub>|`type_parameter_target`
|0x01|<sub>泛型方法或构造器的类型参数声明</sub>|`type_parameter_target`
|0x10|<sub>类声明（包括匿名类声明的直接父类或父接口）时的`extends`或`implements`语法中的类型，或接口声明时的`extends`语法中的类型</sub>|`supertype_target`
|0x11|<sub>泛型类或接口的类型参数绑定声明中的类型</sub>|`type_parameter_bound_target`
|0x12|<sub>泛型方法或构造器的类型参数绑定声明中的类型</sub>|`type_parameter_bound_target`
|0x13|<sub>字段或record成分声明中的类型</sub>|`empty_target`
|0x14|<sub>方法的返回类型，或刚构造完的对象的类型</sub>|`empty_target`
|0x15|<sub>方法或构造器的接收者类型</sub>|`empty_target`
|0x16|<sub>方法、构造器或lambda表达式中的形参声明中的类型</sub>|`formal_parameter_target`
|0x17|<sub>方法或构造器的`throws`语法中的类型</sub>|`throws_target`

**表4.7.20-B `target_type`释义（第2部分）**

|**值**|**目标种类**|`target_info`**属性**
|-|-|-
|0x40|<sub>局部变量声明中的类型</sub>|`localvar_target`
|0x41|<sub>资源变量声明中的类型</sub>|`localvar_target`
|0x42|<sub>异常参数声明中的类型</sub>|`catch_target`
|0x43|<sub>*instanceof*表达式中的类型</sub>|`offset_target`
|0x44|<sub>*new*表达式中的类型</sub>|`offset_target`
|0x45|<sub>使用*::new*时的方法引用表达式中的类型</sub>|`offset_target`
|0x46|<sub>使用*::Identifier*时的方法引用表达式中的类型</sub>|`offset_target`
|0x47|<sub>类型转换表达式中的类型</sub>|`type_argument_target`
|0x48|<sub>*new*表达式中的泛型构造器或显式调用构造器语句中的类型参数</sub>|`type_argument_target`
|0x49|<sub>方法调用表达式中的泛型方法的类型参数</sub>|`type_argument_target`
|0x4A|<sub>使用*::new*的方法引用表达式中的泛型构造器的类型参数</sub>|`type_argument_target`
|0x4B|<sub>使用*::Idetifier*的方法引用表达式中的泛型方法的类型参数</sub>|`type_argument_target`

**表4.7.20-C `target_type`外层属性的位置**

|**值**|**目标种类**|**位置**
|-|-|-
|0x00|<sub>泛型类或接口的类型参数声明</sub>|`ClassFile`
|0x01|<sub>泛型方法或构造器的类型参数声明</sub>|`method_info`
|0x10|<sub>类或接口声明的`extends`，或接口声明时的`implements`语法中的类型</sub>|`ClassFile`
|0x11|<sub>泛型类或接口的类型参数绑定声明中的类型</sub>|`ClassFile`
|0x12|<sub>泛型方法或构造器的类型参数绑定声明中的类型</sub>|`method_info`
|0x13|<sub>字段或record成分声明中的类型</sub>|`field_info`、`record_component_info`
|0x14|<sub>方法或构造器的返回类型</sub>|`method_info`
|0x15|<sub>方法或构造器的接收者类型</sub>|`method_info`
|0x16|<sub>方法、构造器或lambda表达式的形参声明中的类型</sub>|`method_info`
|0x17|<sub>方法或构造器的`throws`语法中的类型</sub>|`method_info`
|0x40-0x4B|<sub>局部变量声明、资源变量声明、异常参数声明、表达式中的类型</sub>|`Code`

#### 4.7.20.1 target_info共用体

该共用体中的属性（除了第一个）准确的指明了一个声明或表达式中的哪个类型被注解了。而第一个属性指明的并非是哪个类型，而是类型参数的哪一出声明被注解了。属性如下：

- `type_parameter_target`属性表示一个泛型类、泛型接口、泛型方法或泛型构造器的第*i*个类型参数声明上出现了一个注解。

```
type_parameter_target {
    u1 type_parameter_index;
}
```

&emsp;&emsp;`type_parameter_index`的值指明了哪一个类型参数声明有注解。`0`表示第一个类型参数声明。

- `supertype_target`属性说明类或接口声明中的`extends`或`implements`语法中的一个类型上出现了一个注解。

```
supertype_target {
    u2 supertype_index;
}
```

&emsp;&emsp;如果`supertype_index`等于65535，说明注解出现在了类声明的`extends`语法中的父类上面。

&emsp;&emsp;其他的值则代表外层`ClassFile`结构体的`interfaces`数组的一个索引，告诉你这个注解要么是出现在类声明的`implements`语法中的父接口上，要么是出现在接口声明的`extends`语法的父接口上。

- `type_parameter_bound_target`属性是说这个注解出现在一个泛型类、接口、方法或构造器的第*j*个类型参数声明的第*i*个绑定上。

```
type_parameter_bound_target {
    u1 type_parameter_index;
    u1 bound_index;
}
```

&emsp;&emsp;`type_parameter_index`元素指的是哪一个类型参数声明上有加了注解的绑定。`0`代表第一个类型参数声明。

&emsp;&emsp;`bound_index`指的是`type_parameter_index`指定的类型参数声明的哪一个绑定带了注解。`0`代表第类型参数声明的第一个绑定。

&emsp;&emsp;&emsp;&emsp;<sub>`type_parameter_bound_target`记录了一个绑定带注解了，但是没有记录该绑定是啥类型组成的。这个类型信息可以去相关的`Signature`属性中的类签名或方法签名中去找。</sub>

&emsp;&emsp;`empty_target`指的是注解要么出现在了字段声明的类型上，要么是出现在record成分声明的类型上，再要么是方法的返回类型上，还要么是新构造的对象的类型上，最后要么是方法或构造器的接收者类型上。各种要么。

```
empty_target {
}
```

&emsp;&emsp;&emsp;&emsp;<sub>每个位置上只能出现一种类型，所以在`target_info`中用不着区分每种类型。</sub>

- `formal_parameter_target`是说注解出现在了方法、构造器或lambda表达式的形参声明的类型上。

```
formal_parameter_target {
    u1 formal_parameter_index;
}
```

&emsp;&emsp;`formal_parameter_index`的值告诉你哪一个形参声明带类型注解。`formal_parameter_index`的值*i*可能但不强制对应方法描述符（§4.3.3）中的第*i*个参数描述符。

&emsp;&emsp;&emsp;&emsp;<sub>该属性记录了一个形参的类型被加了注解，但并不记录类型本身。类型信息可以通过方法描述符去找，尽管`formal_parameter_index`的`0`并不一定指的是方法描述符中的第一个参数描述符；§4.7.18中介绍了一个涉及到`parameter_annotations`表的类似的情况。</sub>

- `throws_target`告诉你一个方法或构造器声明的`throws`语法中的第*i*个类型上出现了一个注解。
```
throws_target {
    u2 throws_type_index;
}
```
&emsp;&emsp;`throws_type_index`的值是一个索引，是一个什么索引呢？是这个`RuntimeVisibleTypeAnnotations`属性外层的`method_info`结构体的`Exceptions`属性的`exception_index_table`数组的一个索引。好家伙，这份儿绕啊。

- `localvar_target`属性指的是注解出现在了局部变量声明的类型上，包括带资源`try`语句中的资源变量声明。

```
localvar_target {
    u2 table_length;
    {   u2 start_pc;
        u2 length;
        u2 index;
    } table[table_length];
}
```

&emsp;&emsp;`table_length`指的是`table`数组中有多少条记录。每条记录代表`code`数组偏移量的一个区间，一个局部变量在这个区间内存在一个值。同时还告诉你这个局部变量在当前帧的局部变量数组的哪个索引位置上可以找到。每条记录包含以下三个属性：

&emsp;&emsp;`start_pc`、`length`<br/>
&emsp;&emsp;&emsp;&emsp;在`code`数组的[`start_pc`, `start_pc + length`)区间内，指定的局部变量有一个值在这儿，区间是左闭右开的。

&emsp;&emsp;`index`<br/>
&emsp;&emsp;&emsp;&emsp;指定局部变量必须在当前帧的局部变量数组的`index`处。

&emsp;&emsp;&emsp;&emsp;如果`index`处的局部变量是`double`或者`long`类型的，那它要占用`index`和`index + 1`两个位置。

&emsp;&emsp;&emsp;&emsp;<sub>必须得拿一个表把加了注解的局部变量完全的说清楚才行，因为一个局部变量可能在多个活动的区间内通过不同的局部变量索引进行表达。每条记录中的`start_pc`、`length`、`index`都是在表达作为`LocalVariableTable`属性的信息。</sub>

&emsp;&emsp;&emsp;&emsp;<sub>`localvar_target`记录了某个局部变量的类型被加了注解了，但是并不记录类型本身。类型信息可以去相关的`LocalVariableTable`属性中找。</sub>

- `catch_target`属性告诉你一个异常参数声明的第*i*个类型上有个注解。

```
catch_target {
    u2 exception_table_index;
}
```

&emsp;&emsp;这个值是一个索引，啥索引呢，是`RuntimeVisibleTypeAnnotations`属性外层的`Code`属性的`exception_table`数组的一个索引。

&emsp;&emsp;&emsp;&emsp;<sub>之所以一个异常参数声明中会出现多个类型，是因为有带多个`catch`块的`try`语句，而且异常参数的类型是一个类型共用体（JLS §14.20）。编译器通常会为共用体中的每个类型创建一个`exception_table`的记录，`catch_target`可以用它来进行区分。这种方式保留了类型和其注解的关联性。</sub>

- `offset_target`告诉你*instanceof*或*new*表达式中的类型上有个注解，或者是方法引用表达式的`::`前面的类型上有个注解。

```
offset_target {
    u2 offset;
}
```

&emsp;&emsp;这个`offset`值指的是`code`数组中的偏移量，可能是对应*instanceof*表达式的字节码指令，也可能是对应*new*表达式的*new*字节码指令，还可能是对应方法引用表达式的字节码指令。

- `type_argument_target`属性告诉你类型转换表达式的第*i*个类型上有个注解，或者是以下场景中的显式类型参数列表的第*i*个类型参数上：*new*表达式、显式构造器调用语句、方法调用表达式、方法引用表达式。

```
type_argument_target {
    u2 offset;
    u1 type_argument_index;
}
```

&emsp;&emsp;`offset`给的是`code`数组中的偏移量，可能是类型转换表达式对应的字节码指令，或者是*new*表达式对应的*new*字节码指令，或者是显式构造器调用语句对应的字节码指令，或者是方法调用表达式对应的字节码指令，或者是方法引用表达式对应的字节码指令。

&emsp;&emsp;对于类型转换表达式，`type_argument_index`的值给出了类型转换操作符中的哪个类型加了注解。`0`表示第一个（或唯一的）类型。

&emsp;&emsp;&emsp;&emsp;<sub>类型转换表达式中会出现多个类型是因为可以转换成一个交叉类型。</sub>

&emsp;&emsp;对于显式类型参数列表，`type_argument_index`告诉你哪个类型参数加了注解了。`0`代表第一个类型参数。

#### 4.7.20.2 type_path结构体

声明或表达式中但凡出现了某个类型，`type_path`结构体都可以标识出哪部分被加了注解了。注解可能直接出现在类型上，但如果类型是一个引用类型，注解也可能出现在其他地方：

- 如果声明或表达式中使用了数组类型<i>`T[]`</i>，那么注解可能出现在数组类型中的任何组成类型上，包括元素类型。
- 如果声明或表达式中使用了嵌套类型<i>`T1.T2`</i>，注解可能出现在最内侧成员类型的名字上，以及任何可以用类型注解的外侧类型上（JLS §9.7.4）。
- 如果声明或表达式中使用了参数化类型<i>`T<A>`</i>或<i>`T<? extends A>`</i>或<i>`T<? super A>`</i>，那么注解可能出现在任意类型参数上，或者任意通配类型参数的结合上。

&emsp;&emsp;<sub>比如`String[][]`的不同部分搞上注解：</sub>

```
@Foo String[][] // 给String类加了注解
String @Foo [][] // 给String[][]数组类加了注解
String[] @Foo [] // 给String[]数组类加了注解
```

&emsp;&emsp;<sub>或者是嵌套类型`Outer.Middle.Inner`的不同部分：</sub>

```
@Foo Outer.Middle.Inner
Outer.@Foo Middle.Inner
Outer.Middle.@Foo Inner
```

&emsp;&emsp;<sub>又或者是参数化类型`Map<String,Object>`或`List<...>`的不同部分搞了注解：</sub>

```
@Foo Map<String,Object>
Map<@Foo String,Object>
Map<String,@Foo Object>
List<@Foo ? extends String>
List<? extends @Foo String>
```

`type_path`结构体的格式如下：

```
type_path {
    u1 path_length;
    {   u1 type_path_kind;
        u1 type_argument_index;
    } path[path_length];
}
```

`path_length`的值代表`path`数组中元素的个数：

- 如果等于`0`，而且被注解的类型是一个嵌套类型，那么注解是出现在最外侧的能允许加注解的部分上。
- 如果等于`0`，而且被注解的类型不是嵌套类型，那么该注解直接就出现在该类型上。
- 如果非零，则`path`数组中的每条记录代表数组类型、嵌套类型、参数化类型中的一个迭代式的、自左向右递进的注解位置。（在数组类型中，就是迭代式访问数组类型本身，然后是它的组成类型，然后是组成类型的组成类型，一步一步递进，直到到达元素类型。）每个记录包含以下两个属性：

&emsp;&emsp;`type_path_kind`<br/>
&emsp;&emsp;&emsp;&emsp;合法值见表4.7.20.2-A。

**表4.7.20.2-A `type_path_kind`释义**

|**值**|**解释**
|-|-
|`0`|注解在数组类型更深的地方
|`1`|注解在嵌套类型更深的地方
|`2`|注解在参数化类型的通配类型参数的结合处
|`3`|注解在参数化类型的类型参数上

&emsp;&emsp;`type_argument_index`<br/>
&emsp;&emsp;&emsp;&emsp;如果`type_path_kind`的值是`0`、`1`、`2`，则`type_argument_index`的值是`0`。

&emsp;&emsp;&emsp;&emsp;如果`type_path_kind`的值是`3`，那么`type_argument_index`的值就会告诉你参数化类型中的哪个类型参数加了注解了，`0`代表参数化类型中的第一个类型参数。

**表4.7.20.2-B `@A Map<@B ? extends @C String, @D List<@E Object>>`的`type_path`结构体**

|**注解**|`path_length`|`path`
|-|-|-
|`@A`|`0`|`[]`
|`@B`|`1`|`[{type_path_kind: 3; type_argument_index: 0}]`
|`@C`|`2`|`[{type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 2; type_argument_index: 0}]`
|`@D`|`1`|`[{type_path_kind: 3; type_argument_index: 1}]`
|`@E`|`2`|`[{type_path_kind: 3; type_argument_index: 1}, {type_path_kind: 3; type_argument_index: 0}]`

**表4.7.20.2-C `@I String @F [] @G [] @H []`的`type_path`结构体**

|**注解**|`path_length`|`path`
|-|-|-
|`@F`|`0`|`[]`
|`@G`|`1`|`[{type_path_kind: 0; type_argument_index: 0}]`
|`@H`|`2`|`[{type_path_kind: 0; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}]`
|`@I`|`3`|`[{type_path_kind: 0; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}]`

**表4.7.20.2-D `@A List<@B Comparable<@F Object @C [] @D [] @E []>>`的`type_path`结构体**

|**注解**|`path_length`|`path`
|-|-|-
|`@A`|`0`|`[]`
|`@B`|`1`|`[{type_path_kind: 3; type_argument_index: 0}]`
|`@C`|`2`|`[{type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}]`
|`@D`|`3`|`[{type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}]`
|`@E`|`4`|`[{type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}]`
|`@F`|`5`|`[{type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}]`

**表4.7.20.2-E `@A Outer . @B Middle . @C Inner`的`type_path`结构体**<br/>
**假设：  class Outer {<br/>
&emsp;&emsp;&emsp;&emsp;class Middle {<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;class Inner {}<br/>
&emsp;&emsp;&emsp;&emsp;}<br/>
&emsp;&emsp;&nbsp;&nbsp;&nbsp;&nbsp;}**

|**注解**|`path_length`|`path`
|-|-|-
|`@A`|`0`|`[]`
|`@B`|`1`|`[{type_path_kind: 1; type_argument_index: 0}]`
|`@C`|`2`|`[{type_path_kind: 1; type_argument_index: 0}, {type_path_kind: 1; type_argument_index: 0}]`

**表4.7.20.2-F `Outer . @A MiddleStatic . @B Inner`的`type_path`结构体**<br/>
**假设：  class Outer {<br/>
&emsp;&emsp;&emsp;&emsp;static class MiddleStatic {<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;class Inner {}<br/>
&emsp;&emsp;&emsp;&emsp;}<br/>
&emsp;&emsp;&nbsp;&nbsp;&nbsp;&nbsp;}**

|**注解**|`path_length`|`path`
|-|-|-
|`@A`|`0`|`[]`
|`@B`|`1`|`[{type_path_kind: 1; type_argument_index: 0}]
|||在`Outer . MiddleStatic . Inner`类型中，不能在`Outer`名字上加注解，因为它右侧的类名`MiddleStatic`并不代表`Outer`的一个内部类。

**表4.7.20.2-G `Outer . MiddleStatic . @A InnerStatic`的`type_path`结构体**<br/>
**假设：  class Outer {<br/>
&emsp;&emsp;&emsp;&emsp;static class MiddleStatic {<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;static class InnerStatic {}<br/>
&emsp;&emsp;&emsp;&emsp;}<br/>
&emsp;&emsp;&nbsp;&nbsp;&nbsp;&nbsp;}**

|**注解**|`path_length`|`path`
|-|-|-
|`@A`|`0`|`[]`
|||在`Outer . MiddleStatic . InnerStatic`类型中，不能在`Outer`名字上加注解，因为它右侧的类名`MiddleStatic`并不代表`Outer`的一个内部类。同样，`MiddleStatic`名字上也不能加注解，因为它右侧的`InnerStatic`也不代表`MiddleStatic`的一个内部类。

**表4.7.20.2-H `Outer . Middle<@A Foo . @B Bar> . Inner<@D String @C []>`的`type_path`结构体**<br/>
**假设：  class Outer {<br/>
&emsp;&emsp;&emsp;&emsp;class Middle&lt;T&gt; {<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;class Inner&lt;U&gt; {}<br/>
&emsp;&emsp;&emsp;&emsp;}<br/>
&emsp;&emsp;&nbsp;&nbsp;&nbsp;&nbsp;}**

|**注解**|`path_length`|`path`
|-|-|-
|`@A`|`2`|`[{type_path_kind: 1; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}]`
|`@B`|`3`|`[{type_path_kind: 1; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 1; type_argument_index: 0}]`
|`@C`|`3`|`[{type_path_kind: 1; type_argument_index: 0}, {type_path_kind: 1; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}]`
|`@D`|`4`|`[{type_path_kind: 1; type_argument_index: 0}, {type_path_kind: 1; type_argument_index: 0}, {type_path_kind: 3; type_argument_index: 0}, {type_path_kind: 0; type_argument_index: 0}]`

关于属性的内容实在是太琐碎无聊了，我已经翻译的没有感情了，希望赶紧结束属性之旅。

### 4.7.21 RuntimeInvisibleTypeAnnotations属性

它是`ClassFile`、`field_info`、`method_info`、`record_component_info`结构体，或`Code`属性（§4.1, §4.5, §4.6, §4.7.30, §4.7.3）的`attributes`表中的一个变长属性。该属性保存了类、字段、方法或record成分声明上的，或对应方法体中某个表达式上的运行时不可见的类型注解。该属性还保存了泛型类、接口、方法和构造器的类型参数声明上的运行时不可见的注解。

在一个`ClassFile`、`field_info`、`method_info`或`record_component_info`结构体或`Code`属性的`attributes`表中最多只能有一个`RuntimeInvisibleTypeAnnotations`属性。

如果声明或表达式中的类型加了注解，只有跟父级的结构体或`attributes`表中的属性对应起来，才会被加入到`attributes`表的`RuntimeInvisibleTypeAnnotations`属性中。（很难理解是不是？别管了）。

格式如下：

```
RuntimeInvisibleTypeAnnotations_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 num_annotations;
    type_annotation annotations[num_annotations];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体，代表字符串值“`RuntimeInvisibleTypeAnnotations`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`num_annotations`<br/>
&emsp;&emsp;该结构体所表达的运行时不可见的类型注解的数量。

`annotations[]`<br/>
&emsp;&emsp;表中每条记录都是某个声明或表达式中的一个运行时不可见的类型注解。`type_annotation`结构体见§4.7.20。

### 4.7.22 AnnotationDefault属性

它是`method_info`结构体（§4.6）的`attributes`表中的一个变长属性，也就是用来表达注解接口中的元素的（JLS §9.6.1）。该属性保存了`method_info`结构体表达的元素的默认值（JLS §9.6.2）。

在一个`method_info`结构体的`attributes`表中最多只能有一个`AnnotationDefault`属性，代表注解接口的一个元素。

格式如下：

```
AnnotationDefault_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    element_value default_value;
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`AnnotationDefault`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`default_value`<br/>
&emsp;&emsp;这个值代表该`AnnotationDefault`属性外层`method_info`结构体表达的注解接口元素的默认值。（绕来绕去讲废话。）

### 4.7.23 BootstrapMethods属性

它是`ClassFile`结构体（§4.1）的`attributes`表中的一个变长属性。该属性记录了用来产生动态计算常量和动态计算调用点的引导方法（§4.4.10）。

当`ClassFile`结构体的`constant_pool`表中至少存在一个`CONSTANT_Dynamic_info`或`CONSTANT_InvokeDynamic_info`记录时，该结构体的`attributes`表中必须有且仅能有一个`BootstrapMethods`属性。

一个`ClassFile`结构体的`attributes`表中最多只能有一个`BootstrapMethods`属性。

格式如下：

```
BootstrapMethods_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 num_bootstrap_methods;
    {   u2 bootstrap_method_ref;
        u2 num_bootstrap_arguments;
        u2 bootstrap_arguments[num_bootstrap_arguments];
    } bootstrap_methods[num_bootstrap_methods];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`BootstrapMethods`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`num_bootstrap_methods`<br/>
&emsp;&emsp;表示`bootstrap_methods`数组中引导方法的个数。

`bootstrap_methods`<br/>
&emsp;&emsp;表中每条记录包含一个`CONSTANT_MethodHandle_info`结构体的索引，它定义了一个引导方法和一个引导方法的*静态参数*的索引序列（可能为空）。

&emsp;&emsp;表中每个记录包含以下三个属性：

&emsp;&emsp;`bootstrap_method_ref`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_MethodHandle_info`结构体（§4.4.8）。

&emsp;&emsp;&emsp;&emsp;<sub>方法句柄在动态计算常量或调用点（§5.4.3.6）的解析过程中就弄好了，然后就可以通过`java.lang.invoke.MethodHandle`的`invokeWithArguments`调用了。方法句柄必须能够接受§5.4.3.6中定义的参数数组，否则解析过程就会失败。</sub>

&emsp;&emsp;`num_bootstrap_arguments`<br/>
&emsp;&emsp;&emsp;&emsp;`bootstrap_arguments`数组中元素的个数。

&emsp;&emsp;`bootstrap_arguments[]`<br/>
&emsp;&emsp;&emsp;&emsp;该表中的每条记录必须是`constant_pool`表的有效索引。对应的`constant_pool`中的记录必须时可加载的（§4.4）。

### 4.7.24 MethodParameters属性

它是`method_info`结构体（§4.6）的`attributes`表中的一个变长属性。该属性记录了方法的形参信息，比如参数的名字。

在一个`mehtod_info`结构体的`attributes`表中最多只能有一个`MethodParameters`属性。

格式如下：

```
MethodParameters_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 parameters_count;
    {   u2 name_index;
        u2 access_flags;
    } parameters[parameters_count];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`MethodParameters`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`parameters_count`<br/>
&emsp;&emsp;它给出了外层`method_info`结构体中`descriptor_index`引用的方法描述符（§4.3.3）中的参数描述符的数量。

&emsp;&emsp;<sub>这个事儿并不需要让JVM在格式检查（§4.8）的过程中做强制要求。让方法描述符中的参数描述符跟下面的`parameters`表对应起来是JavaSE平台反射库的工作。</sub>

`parameters[]`<br/>
&emsp;&emsp;每条记录包含以下两个属性：

&emsp;&emsp;`name_index`<br/>
&emsp;&emsp;&emsp;&emsp;要么是零，要么是`constant_pool`的有效索引。

&emsp;&emsp;&emsp;&emsp;如果是零，那么这个`elements`元素代表的是一个没名字的形参。

&emsp;&emsp;&emsp;&emsp;如果非零，对应`constant_pool`中的记录必须是一个`CONSTANT_Utf8_info`结构体，代表一个有效的形参的非限定名（§4.2.2）。

&emsp;&emsp;`access_flags`<br/>
&emsp;&emsp;&emsp;&emsp;可能的值如下：

&emsp;&emsp;&emsp;&emsp;`0x0010（ACC_FINAL）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;说明形参是`final`的。

&emsp;&emsp;&emsp;&emsp;`0x1000（ACC_SYNTHETIC）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;说明根据源码使用的语言规范来看（JLS §13.1），形参没有显式或隐式的声明在源码中。（编译器生成这个`class`文件的时候，形参也是它的一个实现产物。）

&emsp;&emsp;&emsp;&emsp;`0x8000（ACC_MANDATED）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;说明根据源码使用的语言规范来看（JLS §13.1），形参隐式的声明在了源码中。（形参是由语言规范强制的，因此该语言的所有编译器都要生成它。）

&emsp;&emsp;<sub>`parameters`数组中的第*i*条记录对应外层方法描述符的第*i*个参数描述符。（`parameter_count`之所以是一个字节是因为方法描述符最多只能有255个参数。）也就是意味着`parameters`数组中保存了该方法所有参数的信息。如果我们把形式变一变，比如`parameters`数组中的记录保存了它们对应的参数描述符，但这样会使`MethodParameter`属性变得过于复杂。</sub>

&emsp;&emsp;<sub>`parameters`数组中的第*i*条记录并不一定对应着外层方法的`Signature`属性（如果有的话）中的第*i*个类型，或者是外层方法参数注解中的第*i*个注解。</sub>

### 4.7.25 Module属性

它是`ClassFile`结构体（§4.1）的`attributes`表中的一个变长属性。这个属性表明某一模块依赖了其它模块；对应的包有导出并且被某个模块打开了；模块中使用和提供的服务。

在一个`ClassFile`结构体的`attributes`表中最多只能有一个`Module`属性。

格式如下：

```
Module_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 module_name_index;
    u2 module_flags;
    u2 module_version_index;
    u2 requires_count;
    {   u2 requires_index;
        u2 requires_flags;
        u2 requires_version_index;
    } requires[requires_count];
    u2 exports_count;
    {   u2 exports_index;
        u2 exports_flags;
        u2 exports_to_count;
        u2 exports_to_index[exports_to_count];
    } exports[exports_count];
    u2 opens_count;
    {   u2 opens_index;
        u2 opens_flags;
        u2 opens_to_count;
        u2 opens_to_index[opens_to_count];
    } opens[opens_count];
    u2 uses_count;
    u2 uses_index[uses_count];
    u2 provides_count;
    {   u2 provides_index;
        u2 provides_with_count;
        u2 provides_with_index[provides_with_count];
    } provides[provides_count];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`Module`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包括开头的六个字节。

`module_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Module_info`结构体（§4.4.11），表示当前模块。

`module_flags`<br/>
&emsp;&emsp;可能的值如下：

&emsp;&emsp;`0x0020（ACC_OPEN）`<br/>
&emsp;&emsp;&emsp;&emsp;说明这个模块是开放的。

&emsp;&emsp;`0x1000（ACC_SYNTHETIC）`<br/>
&emsp;&emsp;&emsp;&emsp;说明这个模块没有被显式或隐式的声明。

&emsp;&emsp;`0x8000（ACC_MANDATED）`<br/>
&emsp;&emsp;&emsp;&emsp;说明这个模块是隐式声明的。

`module_version_index`<br/>
&emsp;&emsp;必须是零或者是`constant_pool`的有效索引。如果是零，那么当前模块就没有版本信息。如果非零，`constant_pool`中对应的记录必须得是一个`CONSTANT_Utf8_info`结构体，代表当前模块的版本号。

`requires_count`<br/>
&emsp;&emsp;`requires`表的记录数。

&emsp;&emsp;如果当前模块是`java.base`，该值必须是零。

&emsp;&emsp;如果当前模块不是`java.base`，该值至少是一。

`requires[]`<br/>
&emsp;&emsp;每条记录表示当前模块的一个依赖。每条记录包含的属性如下：

&emsp;&emsp;`requires_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Module_info`结构体，代表当前模块依赖的一个模块。

&emsp;&emsp;&emsp;&emsp;`requires`表中每个`requires_index`对应的模块名只能出现一次。

&emsp;&emsp;`requires_flags`<br/>
&emsp;&emsp;&emsp;&emsp;可能的值如下：

&emsp;&emsp;&emsp;&emsp;`0x0020（ACC_TRANSITIVE）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示依赖当前模块的任何其它模块通过该记录隐式声明了一个模块依赖。

&emsp;&emsp;&emsp;&emsp;`0x0040（ACC_STATIC_PHASE）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示这个依赖在静态阶段是强制的，比如在编译时，但是在动态阶段是可选的，比如在运行时。

&emsp;&emsp;&emsp;&emsp;`0x1000（ACC_SYNTHETIC）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示这个依赖在模块的源码声明中没有显式或隐式的声明。

&emsp;&emsp;&emsp;&emsp;`0x8000（ACC_MANDATED）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示这个依赖在模块源码声明中做了隐式声明。

&emsp;&emsp;&emsp;&emsp;如果当前模块不是`java.base`，而且`class`文件版本号大于等于54.0，那么`requires_flags`中不能设置`ACC_TRANSITIVE`或`ACC_STATIC_PHASE`。

&emsp;&emsp;`requires_version_index`<br/>
&emsp;&emsp;&emsp;&emsp;要么是零要么是`constant_pool`的有效索引。如果是零，这个依赖就没有版本信息。如果非零，`constant_pool`对应的记录必须是一个`CONSTANT_Utf8_info`结构体，代表`requires_index`指定的模块的版本。

&emsp;&emsp;除非当前模块是`java.base`，否则`requires`表中只能有一条记录可以让`requires_index`指向`java.base`同时还不为`requires_flags`设置`ACC_SYNTHETIC`。

`exports_count`<br/>
&emsp;&emsp;`exports`表的记录数。

`exports[]`<br/>
&emsp;&emsp;表中每条记录代表当前模块导出的一个包，这样就能让这个包中的`public`和`protected`类型，以及它们的`public`和`protected`成员在当前模块之外访问到，比如在一个有限的“友军”模块集合中进行访问。

&emsp;&emsp;每条记录中包含的属性如下：

&emsp;&emsp;`exports_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Package_info`结构体（§4.4.12），代表当前模块导出的一个包。

&emsp;&emsp;&emsp;&emsp;`exports`表中的每条记录声明的包名只能出现一次。

&emsp;&emsp;`exports_flags`<br/>
&emsp;&emsp;&emsp;&emsp;可能的值如下：

&emsp;&emsp;&emsp;&emsp;`0x1000（ACC_SYNTHETIC）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示这个导出没有在模块的源码声明中显式或隐式的进行声明。

&emsp;&emsp;&emsp;&emsp;`0x8000（ACC_MANDATED）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示这个导出在模块的源码声明中做了隐式声明。

&emsp;&emsp;`exports_to_count`<br/>
&emsp;&emsp;&emsp;&emsp;`exports_to_index`表的记录数。

&emsp;&emsp;&emsp;&emsp;如果等于零，那么当前模块就是以*非限定*的形式导出了这个包；其他模块中的任意代码都可以访问这个包中的类型和成员。

&emsp;&emsp;&emsp;&emsp;如果非零，那么当前模块就是以*限定*形式导出了这个包；只有`exports_to_index`表中对应模块中的代码才能访问这个包中的类型和成员。

&emsp;&emsp;`exports_to_index[]`<br/>
&emsp;&emsp;&emsp;&emsp;表中的每条记录必须是`constant_pool`表的有效索引。`constant_pool`中对应的记录必须是一个`CONSTANT_Module_info`结构体，表示指定的模块中的代码可以访问这个被导出包中的类型和成员。

&emsp;&emsp;&emsp;&emsp;对于`exports`表中的每条记录，`exports_to_index`表中对应的模块名只能出现一次。

`opens_count`<br/>
&emsp;&emsp;代表`opens`表中的记录数。

&emsp;&emsp;如果当前模块是开放的，那么这个值必须是零。

`opens`<br/>
&emsp;&emsp;表中每条记录代表当前模块开放的一个包，如果包开放了，包中所有的类型以及它们的成员都可以在包外部被JavaSE平台的反射库访问，比如从一个有限的“友军”模块集合中访问。

&emsp;&emsp;每条记录包含的属性如下：

&emsp;&emsp;`opens_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Package_info`结构体，代表当前模块开放的一个包。

&emsp;&emsp;&emsp;&emsp;在`opens`表中，`opens_index`对应的每个包名只能出现一次。

&emsp;&emsp;`open_flags`<br/>
&emsp;&emsp;&emsp;&emsp;可能的值如下：

&emsp;&emsp;&emsp;&emsp;`0x1000（ACC_SYNTHETIC）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示这种开放并没有显式或隐式的声明在模块的源码声明中。

&emsp;&emsp;&emsp;&emsp;`0x8000（ACC_MANDATED）`<br/>
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;表示这种开放隐式的声明在了模块的源码声明中。

&emsp;&emsp;`opens_to_count`<br/>
&emsp;&emsp;&emsp;&emsp;代表`opens_to_index`表的记录数。

&emsp;&emsp;&emsp;&emsp;如果是零，那么当前模块就是以“非限定”的方式开放了这个包；任何其他模块中的代码都可以通过反射的当时访问这个包中的类型和成员。

&emsp;&emsp;&emsp;&emsp;如果非零，那么当前模块就是以“限定”形式开放了这个包；只有`export_to_index`表（本宝宝注：这里感觉是写错了，应该是`opens_to_index`表吧）中列出的模块中的代码才能通过反射的方式访问这个包中的类型和成员。

&emsp;&emsp;`opens_to_index[]`<br/>
&emsp;&emsp;&emsp;&emsp;每条记录必须是`constant_pool`表的有效索引。`constant_pool`表中对应的记录必须是一个`CONSTANT_Module_info`结构体，代表一个模块，这个模块中的代码可以访问这个被开放的包中的类型和成员。

&emsp;&emsp;&emsp;&emsp;对于`opens`表中的每条记录，`opens_to_index`对应的模块名只能出现一次。

`uses_count`<br/>
&emsp;&emsp;`uses_index`表的记录数。

`uses_index[]`<br/>
&emsp;&emsp;每条记录必须是`constant_pool`表的一个有效索引。`constant_pool`表中对应的记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），代表一个服务接口，当前模块可以通过`java.util.ServiceLoader`发现这个服务接口。

&emsp;&emsp;对于`uses_index`表的每条记录，每个服务接口的名字只能出现一次。

`provides_count`<br/>
&emsp;&emsp;`provides`表的记录数。

`provides[]`<br/>
&emsp;&emsp;表中的每条记录代表一个指定服务接口的一个服务实现。

&emsp;&emsp;每条记录包含的属性如下：

&emsp;&emsp;`provides_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Class_info`结构体，代表一个服务接口，当前模块为该服务接口提供了一个服务实现。

&emsp;&emsp;&emsp;&emsp;对于`provides`表中的每条记录，`provides_index`对应的服务接口名只能出现一次。

&emsp;&emsp;`provides_with_count`<br/>
&emsp;&emsp;&emsp;&emsp;表示`provides_with_index`表的记录数。

&emsp;&emsp;&emsp;&emsp;必须非零。

&emsp;&emsp;`provides_with_index[]`<br/>
&emsp;&emsp;&emsp;&emsp;表中每条记录必须是`constant_pool`的有效索引。`constant_pool`中对应的记录必须是一个`CONSTANT_Class_info`结构体，代表一个服务实现，对应的服务接口由`provides_index`指定。

&emsp;&emsp;&emsp;&emsp;对于`provides`表中的每条记录，`provides_with_index`表中的服务实现对应的名字是唯一的。

### 4.7.26 ModulePackages属性

它是`ClassFile`结构体（§4.1）的`attributes`表中的一个变长属性。这个属性的意思是一个模块中的所有包，通过`Module`属性全都该导出或开放了，包括`Module`属性中记录的所有服务实现的包。

在一个`ClassFile`结构体的`attributes`表中，最多只能有一个`ModulePackages`属性。

格式如下：

```
ModulePackages_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 package_count;
    u2 package_index[package_count];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`ModulePackages`”。

`attribute_length`<br/>
&emsp;&emsp;表示属性的长度，不包括开头的六个字节。

`package_count`<br/>
&emsp;&emsp;代表`package_index`表中的记录数。

`package_index[]`<br/>
&emsp;&emsp;表中的记录必须是`constant_pool`的有效索引。`constant_pool`中对应的记录必须是一个`CONSTANT_Package_info`结构体（§4.4.12），代表当前模块中的一个包。

&emsp;&emsp;`package_index`表中的包名唯一。

### 4.7.27 ModuleMainClass属性

它是`ClassFile`结构体的`attributes`表中的一个定长属性（§4.1）。该属性代表模块的主类。

在一个`ClassFile`结构体的`attributes`表中最多只能有一个`ModuleMainClass`属性。

格式如下：

```
ModuleMainClass_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 main_class_index;
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串“`ModuleMainClass`”。

`attribute_length`<br/>
&emsp;&emsp;必须是二。

`main_class_index`<br/>
&emsp;&emsp;必须是`constant_pool`表的有效索引。对应记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），代表当前模块的主类。

### 4.7.28 NestHost属性

它是`ClassFile`结构体的`attributes`表中的一个定长属性。该属性记录了当前类或接口所属嵌套的嵌套主（§5.4.4）。

在一个`ClassFile`结构体的`attributes`表中最多只能有一个`NestHost`属性。

格式如下：

```
NestHost_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 host_class_index;
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串值“`NestHost`”。

`attribute_length`<br/>
&emsp;&emsp;必须是二。

`host_class_index`<br/>
&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），代表当前类或接口的嵌套主的类或接口。

&emsp;&emsp;<sub>如果嵌套主无法被加载，或者跟当前类或接口不在同一个运行时包中，又或者没有为当前类或接口的嵌套成员进行授权，那么在访问控制阶段可能会出现错误（§5.4.4）。</sub>

### 4.7.29 NestMembers属性

它是`ClassFile`结构体（§4.1）的`attributes`表中的一个变长属性。这里面保存了有权限访问当前类或接口管理的嵌套的成员信息的类和接口（§5.4.4）。（我天，这个太拗口了。）

在一个`ClassFile`结构体的`attributes`表中，最多只能有一个`NestMembers`属性。

在一个`ClassFile`结构体的`attributes`表中，不能同时包含`NestMembers`和`NestHost`属性。

&emsp;&emsp;<sub>这条规则可以阻止一个嵌套主试图去获取另一个嵌套的成员关系。它实际上是它所管理的嵌套中的一个隐式成员。</sub>

格式如下：

```
NestMembers_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_classes;
    u2 classes[number_of_classes];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串“`NestMembers`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包含开头的六个字节。

`number_of_classes`<br/>
&emsp;&emsp;代表`classes`数组的记录数。

`classes[]`<br/>
&emsp;&emsp;数组中每条记录必须是`constant_pool`的有效索引。`constant_pool`表中对应的记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），代表一个类或接口，它是当前类或接口管理的嵌套中的一个成员。

&emsp;&emsp;<sub>访问控制（§5.4.4)）会参考`classes`数组中的数据。它应当包含其它类和接口的引用，这些类和接口得在相同的运行时包中，而且它们都得有`NestHost`属性，引用的就是当前类或接口。数组中不满足这些要求的元素都会被访问控制所忽略。</sub>

### 4.7.30 Record属性

它是`ClassFile`结构体（§4.1）的`attributes`表的一个变长属性。该属性告诉你当前类就是一个record类（JLS §8.10），并且还保存了这个record类的record成分信息（JLS §8.10.1）。

在一个`ClassFile`结构体的`attributes`表中最多只能有一个`Record`属性。

格式如下：

```
Record_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 components_count;
    record_component_info components[components_count];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串“`Record`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包含开头的六个字节。

`components_count`<br/>
&emsp;&emsp;代表`components`表的记录数。

`components[]`<br/>
&emsp;&emsp;每条记录代表当前类的一个record成分，按照声明的顺序排序。`record_component_info`结构体格式如下：

```
record_component_info {
    u2 name_index;
    u2 descriptor_index;
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

&emsp;&emsp;解释如下：

&emsp;&emsp;`name_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表record成分的一个有效的非限定名（§4.2.2）。

&emsp;&emsp;`descriptor_index`<br/>
&emsp;&emsp;&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表用来对record成分的类型做编码的字段描述符（§4.3.2）。

&emsp;&emsp;`attributes_count`<br/>
&emsp;&emsp;&emsp;&emsp;代表该record成分的额外属性的数量。

&emsp;&emsp;`attributes[]`<br/>
&emsp;&emsp;&emsp;&emsp;每条记录必须是一个`attribute_info`结构体（§4.7）。

&emsp;&emsp;&emsp;&emsp;一个record成分可以包含任意数量的可选属性。

&emsp;&emsp;&emsp;&emsp;这里能出现的属性见表4.7-C。

&emsp;&emsp;&emsp;&emsp;预定义属性相关规则见§4.7。

&emsp;&emsp;&emsp;&emsp;非预定义属性相关规则见§4.7.1。

### 4.7.31 PermittedSubclasses属性

它是`ClassFile`结构体（§4.1）的`attribtues`表的一个变长属性。该属性纪录了被授权可以直接继承或实现当前类或接口的那些类和接口（§5.3.5）。啦啦啦啦。

&emsp;&emsp;<sub>在Java语言中可以使用`sealed`修饰符来表示一个类或接口可以限制它的直接子类或直接子接口。你可能觉得这个修饰符应该就是在`class`文件中有一个`ACC_SEALED`标记，因为比如`final`的话就是有一个`ACC_FINAL`标记。啦啦啦这你可想错了，在`class`文件中，`sealed`类是通过`PermittedSubclasses`属性来实现的。</sub>

在一个`ClassFile`结构体的`attributes`表中最多只能有一个`PermittedSubclasses`属性，而且这个`ClassFile`的`access_flags`属性中不能设置`ACC_FINAL`标记。

如果`ClassFile`的`access_flags`设置了`ACC_FINAL`标记，那就不能有`PermittedSubclasses`属性。

&emsp;&emsp;<sub>`sealed`跟`final`可不一样：前者是有一个被授权子类的列表的，而后者是不能有子类的。因此一个`ClassFile`可以有`PermittedSubclasses`属性，也可以设置`ACC_FINAL`标记，但两者不能同时出现。</sub>

格式如下：

```
PermittedSubclasses_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_classes;
    u2 classes[number_of_classes];
}
```

解释如下：

`attribute_name_index`<br/>
&emsp;&emsp;必须是`constant_pool`的有效索引。对应记录必须是一个`CONSTANT_Utf8_info`结构体（§4.4.7），代表字符串“`PermittedSubclasses`”。

`attribute_length`<br/>
&emsp;&emsp;属性的长度，不包含开头的六个字节。

`number_of_classes`<br/>
&emsp;&emsp;`classes`数组中的记录数。

`classes[]`<br/>
&emsp;&emsp;数组中的每个值必须是`constant_pool`的有效索引。`constant_pool`表中对应的记录必须是一个`CONSTANT_Class_info`结构体（§4.4.1），代表一个类或接口，它被授权可以直接继承或实现当前类或接口。

&emsp;&emsp;<sub>当一个类或接口被创建出来，并且尝试直接继承或实现当前类或接口（§5.3.5），此时就会参考`classes`数组中的数据。如果里面的类或接口没有尝试直接继承或实现当前类或接口，那么就直接被忽略了。</sub>

## 4.8 格式检查

当一个潜在的`class`文件要开始被JVM加载的时候（§5.3），JVM首先要确保这个文件具备`class`文件的基本格式要求（§4.1）。这个过程就是传说中的*格式检查*。具体是这样的：

- 开头四个字节必须包含正确的魔数。
- 所有预定义属性（§4.7）的长度必须正确，除了`StackMapTable`、`RuntimeVisibleAnnotations`、`RuntimeInvisibleAnnotations`、`RuntimeVisibleParameterAnnotations`、`RuntimeInvisibleParameterAnnotations`、`RuntimeVisibleTypeAnnotations`、`RuntimeInvisibleTypeAnnotations`、`AnnotationDefault`。
- `class`文件不能被截断或者结尾处有多余的字节。
- 常量池必须要满足§4.4中讲的所有约束条件。

&emsp;&emsp;<sub>比如常量池中的每个`CONSTANT_Class_info`结构体的`name_index`属性必须要对应一个有效的常量池索引，对应着一个`CONSTANT_Utf8_info`结构体。</sub>

- 常量池中所有的字段引用和方法引用必须拥有有效的名字、有效的类、有效的描述符（§4.3）。

&emsp;&emsp;格式检查并不确保指定的字段或方法存在于指定的类中，也不确保指定的描述符引用了一个真正的类。格式检查只是确保格式上是正确的。更详细的检查是在对字节码做校验和解析的时候弄。

要想解释好`class`文件中的内容，这些基本的完整性校验是必需的。格式检查不同于字节码校验，尽管历史上它们曾经一度混淆，因为它们都属于是完整性的校验。

## 4.9 JVM代码约束

这句话有意思了，我要开始写了。方法、实例初始化方法（§2.9.1）、类或接口初始化方法（§2.9.2），它们的代码都是保存在`class`文件的`method_info`结构体的`Code`属性的`code`数组中（§4.7.3）。本节就是要讲一讲`Code_attribute`结构体中的约束。

### 4.9.1 静态约束

`class`文件的*静态约束*要的就是文件的格式正确性。前面的章节中已经介绍过这些约束，除了`class`文件中代码的静态约束。这种静态约束定义了`code`数组中JVM指令的形式，以及每个指令的操作数是什么样的。

`code`数组中的指令静态约束如下：
- 只允许§6.5中列出的指令出现在`code`数组中。如果指令使用保留的操作码（§6.2）或者不在本规范给出的范围内，打死也不能放到`code`数组中。

&emsp;&emsp;如果`class`文件版本号大于等于51.0，那么`code`数组中不能出现<i>`jsr`</i>或<i>`jsr_w`</i>指令。

- `code`数组中第一个指令的操作码必须从索引`0`开始。（没看懂）
- 除了最后一条指令，`code`数组中的每条指令的下一条指令操作码的索引等于当前指令操作码的索引加上指令的长度，包括所有的操作数。

&emsp;&emsp;下面这段话我实在读不懂。*wide*指令也是如此；*wide*指令要修改的操作的操作码也属于该*wide*指令的操作数之一。这个操作码永远无法被运算过程直接接触到。

- `code`数组中最后一个指令的最后一个字节必须位于索引`code_length - 1`处。

`code`数组中的指令操作数的静态约束如下：

- 所有跳跃和分叉指令（*jsr*, *jsr_w*, *goto*, *goto_w*, *ifeq*, *ifne*, *ifle*, *iflt*, *ifge*, *ifgt*, *ifnull*, *ifnonnull*, *if_icmpeq*, *if_icmpne*, *if_icmple,if_icmplt*, *if_icmpge*, *if_icmpgt*, *if_acmpeq*, *if_acmpne*）的目标必须是该方法内部某条指令的操作码。

&emsp;&emsp;跳跃或分叉指令的目标永远不能是一个*wide*指令要修改的操作的操作码；一个跳跃或分叉的目标可以是*wide*指令本身。

- *tableswitch*指令的每个目标，包括默认目标，必须是当前方法中的指令的操作码。

&emsp;&emsp;每个*tableswitch*指令的跳跃表中必须要有一系列的记录，而且必须要跟跳跃表的*low*和*high*操作数的值保持一致，*low*值必须小于等于*high*值。

&emsp;&emsp;*tableswitch*指令的目标不能是*wide*指令要修改的操作的操作码；目标可以是*wide*指令本身。

- *lookupswitch*指令的每个目标，包括默认目标，必须是当前方法中的指令的操作码。

&emsp;&emsp;每条*lookupswitch*指令都有一系列*match-offset*对儿，它们的值跟操作数*npairs*的值保持一致。这些*match-offset*对儿必须按照带符号匹配值的数值升序排序。

&emsp;&emsp;*lookupswitch*指令的目标不能是*wide*指令要修改的操作的操作码；目标可以是*wide*指令本身。

- 每条*ldc*和*ldc_w*指令的操作数必须代表一个`constant_pool`表中的有效索引。`constant_pool`表中对应的记录必须是可加载的（§4.4），而且不能是以下内容：
    - `CONSTANT_Long`或`CONSTANT_Double`记录。
    - `CONSTANT_Dynamic`记录，并且它引用了一个`CONSTANT_NameAndType_info`结构体，这个结构体又指出了`J`（代表`long`）或`D`（代表`double`）的描述符。
- 每条*ldc2_w*指令的操作数必须代表一个`constant_pool`表中的有效索引。`constant_pool`表中对应的记录必须是可加载的（§4.4），而且得是以下内容之一：
    - `CONSTANT_Long`或`CONSTANT_Double`记录。
    - `CONSTANT_Dynamic`记录，并且它引用了一个`CONSTANT_NameAndType_info`结构体，这个结构体又指出了`J`（代表`long`）或`D`（代表`double`）的描述符。

&emsp;&emsp;&emsp;&emsp;后续的常量池索引也得是有效的索引，而且池中对应的记录绝对不能被用到。

- 每条*getfield*, *putfield*, *getstatic*, *putstatic*指令的操作数必须代表一个`constant_pool`表中的有效索引。常量池中的记录必须是`CONSTANT_Fieldref`。
- 每条*invokevirtual*指令的*indexbyte*操作数必须代表一个`constant_pool`表中的有效索引。常量池中对应的记录必须是`CONSTANT_Methodref`。
- 每条*invokespecial*指令和*invokestatic*指令的*indexbyte*操作数必须代表一个`constant_pool`表中的有效索引。如果`class`文件版本号小于52.0，池中对应的记录必须是`CONSTANT_Methodref`；如果大于等于52.0，池中对应的记录必须是`CONSTANT_Methodref`或`CONSTANT_InterfaceMethodref`。
- 每条*invokeinterface*指令的*indexbyte*操作数必须代表一个`constant_pool`表中的有效索引。常量池中对应的记录必须是`CONSTANT_InterfaceMethodref`。

&emsp;&emsp;每条*invokeinterface*指令的*count*操作数的值必须要给出传给接口方法的参数需要存储的局部变量的数量，正如`CONSTANT_InterfaceMethodref`引用的`CONSTANT_NameAndType_info`结构体的描述所暗示的那样。啧啧，都出来暗示了，我这翻译水平可以，拟人小王子。

&emsp;&emsp;每个*invokeinterface*指令的第四个操作数字节必须是零。

- 每条*invokedynamic*指令的*indexbyte*操作数必须代表一个`constant_pool`表中的有效索引。常量池中对应的记录必须是`CONSTANT_InvokeDynamic`。

&emsp;&emsp;每条*invokedynamic*指令的第三第四个参数必须是零。

- 只有*invokespecial*指令才能调用实例初始化方法（§2.9.1）。

&emsp;&emsp;方法调用指令是不能调用以'`<`'（'`\u003c`'）开头的方法的。特别是类或接口的初始化方法，以特殊的`<clinit>`命名，从来都不会直接被JVM指令调用，只是隐式的被JVM本身所调用。

- 每条*instanceof*, *checkcast*, *new*, *anewarray*指令的操作数，以及每条*multianewarray*指令的*indexbyte*操作数，必须代表`constant_pool`表的一个有效索引。表中对应记录必须是`CONSTANT_Class`。
- *new*指令引用的常量池记录不能是代表数组类型的`CONSTANT_Class`（§4.3.2）。该指令不能用来创建数组。
- *anewarray*指令创建的数组维度不能大于255。
- *multianewarray*指令创建的多维数组的维度至少要等于*dimensions*操作数的值。也就是说，虽然一个*multianewarray*指令不需要创建它的*indexbyte*操作数引用的数组类型的所有维度，但是创建的维度不能大于数组类型中的维度。

&emsp;&emsp;每条*multianewarray*指令的*dimensions*操作数的值不能是零。

- 每条*newarray*指令的*atype*操作数的值必须是`T_BOOLEAN`（4）, `T_CHAR`（5）, `T_FLOAT`（6）, `T_DOUBLE`（7）, `T_BYTE`（8）, `T_SHORT`（9）, `T_INT`（10）, `T_LONG`（11）之一。
- 每条*iload*、*fload*、*aload*、*istore*、*fstore*、*astore*、*iinc*、*ret*指令的*index*操作数必须是一个非负整数，而且不能大于`max_locals - 1`。

&emsp;&emsp;每条*iload_&lt;n&gt;*、*fload_&lt;n&gt;*、*aload_&lt;n&gt;*、*istore_&lt;n&gt;*、*fstore_&lt;n&gt;*、*astore_&lt;n&gt;*指令的隐式索引必须不能大于`max_locals - 1`。

- 每条*lload*，*dload*，*lstore*，*dstore*指令的*index*操作数不能大于`max_locals - 2`。

&emsp;&emsp;每条*lload_&lt;n&gt;*，*dload_&lt;n&gt;*，*lstore_&lt;n&gt;*，*dstore_&lt;n&gt;*指令的隐式索引必须不能大于`max_locals - 2`。

- 每条*wide*指令，如果修改的是*iload*，*fload*，*aload*，*istore*，*fstore*，*astore*，*iinc*，*ret*指令，那么它的*indexbyte*操作数必须要代表一个非负整数，且不能大于`max_locals - 1`。

&emsp;&emsp;每条*wide*指令，如果修改的是*lload*，*dload*，*lstore*，*dstore*指令，那么它的*indexbyte*操作数必须代表一个非负整数，且不能大于`max_locals - 2`。
### 4.9.2 结构化约束

关于`code`数组的结构化约束，它定义了JVM指令之间的关系。结构化约束如下：

- 每条指令只有在操作数栈和局部变量数组给出适当类型和数量的参数时才能执行，不管当前调用路径是咋流转过来的。

&emsp;&emsp;操作`int`类型值的指令也可以用来操作`boolean`、`byte`、`char`、`short`类型的值。

&emsp;&emsp;&emsp;&emsp;<sub>正如§2.3.4、§2.11.1中提到的，JVM会在内部将`boolean`、`byte`、`short`、`char`类型转换成`int`。</sub>

- 如果一条指令可以通过不同的路径执行，那么在指令执行前，不管走哪条路，操作数栈必须具有相同的深度（§2.6.2）。
- 在执行过程中，操作数栈深度不能大于`max_stack`。
- 在执行过程中，操作数栈能出栈的值的数量不能大于它所包含的数量。
- 在执行过程中，持有`long`或`double`类型值的局部变量对的顺序不能反转或分裂。这种局部变量对不能拆开使用。
- 局部变量（或局部变量对，比如`long`或`double`类型的值）在赋值前不能访问。
- 每条*invokespecial*指令必须声明以下几项之一：
    - 一个实例初始化方法（§2.9.1）
    - 当前类或接口中的一个方法
    - 当前类的父类中的一个方法
    - 当前类或接口的一个直接父接口中的方法
    - `Object`中的一个方法

&emsp;&emsp;如果*invokespecial*指令带的是一个实例初始化方法，那么它对操作数栈的引用目标必须是一个未初始化的类实例。

&emsp;&emsp;绝对不能在一个已经初始化过的类实例上调用实例初始化方法。而且：

&emsp;&emsp;- 如果对操作数栈的引用目标是*当前类的一个未初始化的类实例*，那么*invokespecial*必须要带着的是当前类或它的直接父类的实例初始化方法。

&emsp;&emsp;- 如果*invokespecial*指令带着一个实例初始化方法，并且对操作数栈的引用目标是*在此之前的`new`指令创建的一个类实例*，那么这个*invokespecial*带着的必须是内个类实例的类的实例初始化方法。（大张伟出现了！内内个内内，内个内个内内）。

&emsp;&emsp;如果*invokespecial*指令带着的不是一个实例初始化方法，那么对于操作数栈的引用目标必须是一个类实例，而且它的类型对于当前类来说也得是可以赋值的（JLS §5.2）。

&emsp;&emsp;&emsp;&emsp;<sub>*invokespecial*的一般规则是，它带着的类或接口必须在调用者类或接口“之上”，而*invokespecial*的接收者对象必须“在”调用者类或接口，或在调用者类或接口“之下”。后两句尤为关键：一个类或接口只能对它自己的对象执行invokespecial。在§invokespecial中可以看到后两句在Prolog中是如何实现的。</sub>

- 每个实例初始化方法，除了从`Object`类的构造器继承过来的实例初始化方法，在实例成员被*访问*之前，必须要调用`this`的另一个实例初始化方法或它的直接父类`super`的实例初始化方法。

&emsp;&emsp;但是，当前类声明的`this`的实例字段可以在其它实例初始化方法调用之前通过*putfield*来进行*赋值*。

- 当任意实例方法被调用，或任意实例变量被方法时，包含它们的类实例必须已经被初始化过。
- 如果异常处理器代码中的一个局部变量中包含了一个未初始化的类实例，那么（i）该异常处理器在`<init>`方法中，该异常处理器必须抛出一个异常，或者打一个死循环；（ii）如果异常处理器不在`<init>`方法中，这个未初始化的类实例必须保持未初始化的状态。
- 当执行*jsr*或*jsr_w*指令时，操作数栈上或者局部变量中不能出现未初始化的类实例。
- 方法调用指令目标的每一个类实例（也就是引用的操作数栈上的目标类型）必须跟指令中声明的类或接口类型是赋值兼容的。
- 每个方法调用的参数类型必须跟方法描述符是方法调用兼容的（JLS §5.3, §4.3.3）。
- 每个返回指令必须匹配方法的返回类型：
    - 如果方法返回一个`boolean`、`byte`、`char`、`short`、`int`，那么就只能用*ireturn*指令。
    - 如果方法返回一个`float`、`long`、`double`，那么对应只能用*freturn*、*lreturn*、*dreturn*指令。
    - 如果方法返回一个`reference`类型，那么只能用*areturn*指令，而且返回值的类型必须跟方法返回描述是赋值兼容的（§4.3.3）。
    - 所有的实例初始化方法、类或接口初始化方法、以及叫嚣要返回`void`的方法，都只能使用*return*指令。
- 通过*getfield*指令访问或通过*putfield*指令进行修改的每一个类实例的类型（也就是对操作数栈引用目标的类型），必须跟指令中声明的类类型是赋值兼容的。
- 通过*putfield*或*putstatic*指令存储的每一个值的类型必须跟类实例或类中该字段（§4.3.2）的描述符是兼容的：
    - 如果描述符类型是`boolean`、`byte`、`char`、`short`或`int`，那么这个值必须是个`int`。
    - 如果描述符类型是`float`、`long`、`double`，那么这个值对应必须是一个`float`、`long`、`double`。
    - 如果描述符类型是一个`reference`类型，那么这个值的类型必须得跟描述符类型是赋值兼容的。
- 使用*aastore*指令存储到数组中的每个值的类型必须是一个`reference`类型。

&emsp;&emsp;通过*aastore*指令存入数组的成分类型同样必须得是一个`reference`类型。

- 每个*athrow*指令扔出来的值必须是`Throwable`类或其子类的实例。

&emsp;&emsp;方法的`Code_attribute`结构体的`exception_table`数组中的`catch_type`属性中的类，必须得是`Throwable`或其子类。

- 如果*getfield*或*putfield*访问的是一个超类中的`protected`字段，而且它所在的运行时包跟当前类还不一样，那么这个被访问的类实例的类型（也就是对操作数栈的引用目标的类型）必须跟当前类是赋值兼容的。

&emsp;&emsp;如果*invokevirtual*或*invokespecial*指令访问的是一个超类中声明的`protected`方法，而且它跟当前类的运行时包也不一样，那么这个被访问的类实例的类型（也就是对操作数栈引用目标的类型）必须跟当前类是赋值兼容的。

- 执行流永远不会跌穿`code`数组。
- 不能从局部变量中加载返回地址（`returnAddress`类型的值）。
- `jsr`或`jsr_w`指令后面跟着的指令只能用单个`ret`指令进行返回。
- 如果是被返回到了`jsw`或`jsr_w`指令，那么这两个指令就不能用来递归调用一个子过程，而且这个子过程已经在子过程调用链中出现了。（如果是在一个`finally`块中可以继续使用`try-finally`块来嵌套子过程。）这说的这都是啥啊，不懂呀。
- 每个`returnAddress`类型的实例最多可以被返回一次。

&emsp;&emsp;如果一个*ret*指令的返回点是在子过程调用链中位于该指令的上方某处，对应了一个`returnAddress`类型的实例，那么这个实例永远不能被用来当返回地址。

## 4.10 class文件校验

尽管Java语言的编译器必须要生成满足上一节所讲的静态和结构化约束的`class`文件，但是JVM依然无法保证要加载的文件是由该编译器所生成的而且是格式正确的。比如像浏览器这种应用，它们不会下载源码然后编译；而是下载已经编译好的`class`文件。浏览器需要判断这个`class`文件是否由一个值得信任的编译器生成，还是说它是抵抗组织派来利用JVM的。

&emsp;&emsp;<sub>编译时检查的另一个问题是版本问题。用户可能已经成功编译了一个类，比如`PurchaseStockOptions`，它是`TradingClass`的一个子类。但是`TradingClass`的定义在编译之后可能发生了变化，无法兼容之前已存在的二进制。方法可能被删了或者是返回类型或修饰符被改了。字段类型可能变了，可能从实例变量变成了类变量。方法或变量的访问修饰符可能变了，从`public`变成了`private`。关于这种问题的讨论，见<i>Java语言规范，JavaSE 第17版</i>的第13章“二进制兼容”。</sub>

由于这些潜在的问题，JVM需要对它要使用的`class`文件进行校验，看它是否满足约束条件。JVM在链接时（§5.4）校验每个`class`文件是否满足必需的约束条件。

链接时校验提升了运行时解释器的性能。否则就要在运行时对每一条指令花费高昂的代价来进行约束校验。JVM会假设这些校验已经完成了。比如，JVM已经知道了以下事实：

- 不存在操作数栈的上溢或下溢。
- 使用和存储的局部变量都是有效的。
- JVM指令的参数的类型都是有效的。

JVM实现时，可能会使用两种策略进行校验：

- 对于版本号大于等于50.0的`class`文件必须使用类型检查进行校验。
- 所有的JVM实现必须都得支持类型推断校验，除了那些符合Java ME CLDC和Java Card的实现，这样才能校验那些版本号小于50.0的`class`文件。

&emsp;&emsp;对于那些符合Java ME CLDC和Java Card的JVM实现，它们的校验请看它们各自的规范，我不管。

这两种策略主要是用来强调§4.9节中讲的`Code`属性（§4.7.3）的`code`数组中的那些静态和结构化的约束。但是在校验过程中，`Code`属性之外还有三个检查也需要做：

- 确保`final`类没有子类。
- 确保`final`方法没有被重写（§5.4.5）。
- 检查是否每一个类（除了`Object`）都有一个直接超类。

### 4.10.1 类型检查校验

如果`class`文件的版本号大于等于50.0（§4.1）那么就必须要用本节介绍的类型检查规则进行校验。

当且仅当`class`文件的版本号等于50.0时，如果类型检查失败，JVM实现可以选择尝试用类型推断进行校验（§4.10.2）。

&emsp;&emsp;<sub>这是一种实用性的妥协，为了让新的校验规则过渡的更加丝滑。许多需要操作`class`文件的工具可能会通过调整方法的栈映射帧对方法的字节码进行修改。如果某个工具没有完成所需的栈映射帧调整，可能会导致检查失败，即便它的字节码在规则上是有效的（然后就会使用旧的类型推断机制进行校验）。为了让实现者们有机会去适配他们的工具，JVM实现可以回落到旧的校验规则上，但仅限于一小段时间。</sub>

&emsp;&emsp;<sub>如果类型检查失败，启用了类型推断，然后成功了，此时自然会出现性能上的损耗。这种损耗无法避免。这也提醒了工具的开发者，他们需要调整他们的产物了，也算是提供了一种额外的刺激手段。</sub>

&emsp;&emsp;<sub>总的来说吧，采用类型推断做降级校验，既可以让JavaSE平台的栈映射帧平滑的接入进来（如果在版本号为50.0的`class`中没有出现，那就可以降级），而且还可以从JavaSE平台中平滑的删掉*jsr*和*jsr_w*指令（如果在版本号为50.0的`class`中没有出现，那就可以降级）。</sub>

如果一个JVM实现要在版本号50.0的类文件上做类型推断校验，那么就必须在所有类型检查校验失败时都要这么做。

&emsp;&emsp;<sub>这就意味着JVM实现时如果某时用了类型推断校验，那么就不能再变回去了。它要么拒绝那些不用类型检查校验的`class`文件，要么就在类型检查失败时持续降级用类型推断校验。</sub>

类型检查器会将Prolog语法声明的类型规则进行强制要求。其中用英语文本对类型规则做非正规描述，用Prolog语法做正规描述。

类型检查器需要每个带`Code`属性（§4.7.3）的方法提供一个栈映射帧列表。这个列表是在`Code`属性的`StackMapTable`属性（§4.7.4）中的。这样做的目的就是栈映射帧必须出现在方法中每个基本块的开头。栈映射帧在每个基本块的开头就能声明每个操作数栈记录和每个局部变量的校验类型。类型检查器读取这些栈映射帧，使用这些映射就可以生成`Code`属性中指令类型安全的证明。

如果类的方法都是类型安全的，那么这个类就是类型安全的，而且它不是一个`final`类的子类。

```
classIsTypeSafe(Class) :-
    classClassName(Class, Name),
    classDefiningLoader(Class, L),
    superclassChain(Name, L, Chain),
    Chain \= [],
    classSuperClassName(Class, SuperclassName),
    loadedClass(SuperclassName, L, Superclass),
    classIsNotFinal(Superclass),
    classMethods(Class, Methods),
    checklist(methodIsTypeSafe(Class), Methods).
classIsTypeSafe(Class) :-
    classClassName(Class, 'java/lang/Object'),
    classDefiningLoader(Class, L),
    isBootstrapLoader(L),
    classMethods(Class, Methods),
    checklist(methodIsTypeSafe(Class), Methods).
```

Prolog预言`classIsTypeSafe`假设`Class`是一个Prolog词条，代表一个二进制类，已经被成功解析和加载。这份声明没有强调该词条的精确结构，但是要求基于它做出某些预言（Predicate）。

&emsp;&emsp;<sub>比如我们假设这样一个语言`classMethods(Class, Methods)`，第一个词条参数是上面描述的类，第二个参数是一个列表，由该类中的所有方法组成，表达的方式也很简易，后面会说。</sub>

当且仅当预言`classIsTypeSafe`不为真，那么类型检查器必须要抛出一个`VerifyError`异常，表示这个`class`文件格式有问题。否则，该`class`文件即通过类型检查，字节码校验顺利完成。

本节剩余的内容会详细介绍类型检查的过程：

- 首先，我们为核心JVM产物给出Prolog预言，比如类和方法（§4.10.1.1）。
- 然后，我们定义好类型检查器已知的类型系统（§4.10.1.2）。
- 第三，我们定义出指令和栈映射帧的Prolog表现式（§4.10.1.3，§4.10.1.4）。
- 第四，我们给出一个方法是如何做类型检查的，包括有代码（§4.10.1.6）和无代码（§4.10.1.5）的方法。
- 第五，我们要讨论对于所有加载和存储指令常见的类型检查问题（§4.10.1.7），以及访问`protected`成员时的问题（§4.10.1.8）。
- 最后，我们定义出每条指令的类型检查规则（§4.10.1.9）。

#### 4.10.1.1 JVM制品的访问器

我们明确规定了28个Prolog预言（“访问器”），它们具有预期的行为，但是本书中并不会给出它们的形式化定义。

`classClassName(Class, ClassName)`<br/>
&emsp;&emsp;提取`Class`类的名字，`ClassName`。

`classIsInterface(Class)`<br/>
&emsp;&emsp;当且仅当`Class`类是接口时为真。

`classIsNotFinal(Class)`<br/>
&emsp;&emsp;当且仅当`Class`类不是一个`final`类。

`classSuperClassName(Class, SuperClassName)`<br/>
&emsp;&emsp;提取`Class`类的超类的名字，`SuperClassName`

`classInterfaces(Class, Interfaces)`<br/>
&emsp;&emsp;提取`Class`类的直接父接口列表，`Interfaces`

`classMethods(Class, Methods)`<br/>
&emsp;&emsp;提取`Class`类中声明的方法的列表，`Methods`。

`classAttributes(Class, Attributes)`<br/>
&emsp;&emsp;提取`Class`类的属性的列表，`Attributes`。

&emsp;&emsp;每个属性都被表达成了`attribute(AttributeName, AttributeContents)`形式的函子引用，其中`AttributeName`是属性的名字。属性内容的格式这里没有定义。

`classDefiningLoader(Class, Loader)`<br/>
&emsp;&emsp;提取`Class`类的定义类加载器，`Loader`。

`isBootstrapLoader(Loader)`<br/>
&emsp;&emsp;当且仅当类加载器`Loader`是引导类加载器。

`loadedClass(Name, InitiatingLoader, ClassDefinition)`<br/>
&emsp;&emsp;当且仅当存在一个名为`Name`的类，它在被类加载器`InitiatingLoader`加载时的表现形式（按照本书要求）是`ClassDefinition`。

`methodName(Method, Name)`<br/>
&emsp;&emsp;提取方法`Method`的名字，`Name`。

`methodAccessFlags(Method, AccessFlags)`<br/>
&emsp;&emsp;提取方法`Method`的访问标记，`AccessFlags`。

`methodDescriptor(Method, Descriptor)`<br/>
&emsp;&emsp;提取方法`Method`的描述符，`Descriptor`。

`methodAttributes(Method, Attributes)`<br/>
&emsp;&emsp;提取方法`Method`的属性的列表，`Attributes`。

`isInit(Method)`<br/>
&emsp;&emsp;当且仅当`Method`（不管类是啥）是`<init>`时为真。

`isNotInit(Method)`<br/>
&emsp;&emsp;当且仅当`Method`（不管类是啥）不是`<init>`时为真。

`isNotFinal(Method, Class)`<br/>
&emsp;&emsp;当且仅当`Class`类中的`Method`不是`final`时为真。

`isStatic(Method, Class)`<br/>
&emsp;&emsp;当且仅当`Class`类中的`Method`是`static`时为真。

`isNotStatic(Method, Class)`<br/>
&emsp;&emsp;当且仅当`Class`类中的`Method`不是`static`时为真。

`isPrivate(Method, Class)`<br/>
&emsp;&emsp;当且仅当`Class`类中的`Method`是`private`时为真。

`isNotPrivate(Method, Class)`<br/>
&emsp;&emsp;当且仅当`Class`类中的`Method`不是`private`时为真。

`isProtected(MemberClass, MemberName, MemberDescriptor)`<br/>
&emsp;&emsp;当且仅当`MemberClass`类中有一个成员名为`MemberName`且它的描述符为`MemberDescriptor`，并且它还是`protected`时为真。

`isNotProtected(MemberClass, MemberName, MemberDescriptor)`<br/>
&emsp;&emsp;当且仅当`MemberClass`类中有一个成员名为`MemberName`且它的描述符为`MemberDescriptor`，并且它不是`protected`时为真。

`parseFieldDescriptor(Descriptor, Type)`<br/>
&emsp;&emsp;将字段描述符`Descriptor`转换成对应的校验类型`Type`（§4.10.1.2）。

`parseMethodDescriptor(Descriptor, ArgTypeList, ReturnType)`<br/>
&emsp;&emsp;将方法描述符`Descriptor`转换成一个校验类型列表`ArgTypeList`，对应方法的参数类型，以及一个校验类型`ReturnType`，对应返回类型。

`parseCodeAttribute(Class, Method, FrameSize, MaxStack, ParsedCode, Handlers, StackMap)`<br/>
&emsp;&emsp;提取`Class`的`Method`方法中的指令流`ParsedCode`，以及操作数栈大小的最大值`MaxStack`，局部变量数量最大值`FrameSize`，异常处理器`Handlers`，以及栈映射`StackMap`。

&emsp;&emsp;指令流和栈映射属性的表现形式必须跟§4.10.1.3和§4.10.1.4中说的一样。

`samePackageName(Class1, Class2)`<br/>
&emsp;&emsp;当且仅当`Class1`和`Class2`的包名一样时为真。

`differentPackageName(Class1, Class2)`<br/>
&emsp;&emsp;当且仅当`Class1`和`Class2`的包名不一样时为真。

对方法体做类型检查的时候，很容易访问到与方法相关的信息。为了实现这种需求，我们定义了一个*环境变量*，一个六元组：

- 一个类
- 一个方法
- 方法声明的返回类型
- 方法中的指令
- 操作数栈大小的最大值
- 异常处理器列表

我们定义了从环境变量提取信息的访问器。

```
allInstructions(Environment, Instructions) :-
    Environment = environment(_Class, _Method, _ReturnType,
                                Instructions, _, _).

exceptionHandlers(Environment, Handlers) :-
    Environment = environment(_Class, _Method, _ReturnType,
                                _Instructions, _, Handlers).

maxOperandStackLength(Environment, MaxStack) :-
    Environment = environment(_Class, _Method, _ReturnType,
                                _Instructions, MaxStack, _Handlers).

thisClass(Environment, class(ClassName, L)) :-
    Environment = environment(Class, _Method, _ReturnType,
                                _Instructions, _, _),
    classDefiningLoader(Class, L),
    classClassName(Class, ClassName).

thisMethodReturnType(Environment, ReturnType) :-
    Environment = environment(_Class, _Method, ReturnType,
                                _Instructions, _, _).
```

我们还定义了其他的预言，可以从环境变量中获取高阶信息。

```
offsetStackFrame(Environment, Offset, StackFrame) :-
    allInstructions(Environment, Instructions),
    member(stackMap(Offset, StackFrame), Instructions).
currentClassLoader(Environment, Loader) :-
    thisClass(Environment, class(_, Loader)).
```

最终，我们定义了一个一般预言贯穿类型规则：

```
notMember(_, []).
notMember(X, [A | More]) :- X \= A, notMember(X, More).
```

&emsp;&emsp;<sub>如何规定哪些访问器应该被明确要求，哪些应该被完整定义，我们的根本宗旨是，不去过度要求`Class`文件的表达形式。为`Class`或`Method`词条（term）提供专用的访问器的话，就会强制我们去完整定义出用来表达`class`文件的Prolog词条的格式。</sub>

#### 4.10.1.2 校验类型系统

类型检查器基于*校验类型*层级强加了一套类型系统，如图所示：

```
校验类型层级:
                   top
             ______/\______
             /            \
            /              \
        oneWord          twoWord
       /   |   \           /  \
      /    |    \         /    \
    int float reference long double
                /    \
               /      \_________
              /                 \
             /                   \
        未初始化的               +------------------+
      /            \            |    Java引用       |
     /              \           |    类型层级       |
未初始化的this  未初始化(Offset)  +------------------+
                                         |
                                         |
                                        null
```

大部分校验类型跟表4.3-A中用字段描述符表达的基本类型和引用类型都有一个直接关联：

- 基本类型`double`、`float`、`int`、`long`（字段描述符`D`、`F`、`I`、`J`）对应它们的同名校验类型。
- 基本类型`byte`、`char`、`short`、`boolean`（字段描述符`B`、`C`、`S`、`Z`）都对应同样的校验类型`int`。
- 类和接口类型（字段描述符以`L`开头）对应使用`class`函子的校验类型。校验类型`class(N, L)`代表的类的二进制名为*N*，被加载器*L*加载。这里`L`是`class(N, L)`所表达的类的初始化加载器（§5.3），那么它可以是也可以不是该类的定义加载器。

&emsp;&emsp;<sub>比如`Object`这种类类型可以表达成`class('java/lang/Object', BL)`，其中`BL`是引导加载器。</sub>

- 数组类型（字段描述符以`[`开头）对应的校验类型使用`arrayOf`函子。虽然基本类型`byte`、`char`、`short`、`boolean`没有单独对应的校验类型，但是元素类型为`byte`、`char`、`short`、`boolean`的数组类型*确有*单独对应的校验类型；这些校验类型支持*baload*、*bastore*、*caload*、*castore*、*saload*、*sastore*、*newarray*指令。
    - 校验类型`arrayOf(T)`代表的数组类型，它的组成类型是校验类型*T*。
    - 校验类型`arrayOf(byte)`代表的数组类型，它的元素类型是`byte`。
    - 校验类型`arrayOf(char)`代表的数组类型，它的元素类型是`char`。
    - 校验类型`arrayOf(short)`代表的数组类型，它的元素类型是`short`。
    - 校验类型`arrayOf(boolean)`代表的数组类型，它的元素类型是`boolean`。

&emsp;&emsp;&emsp;&emsp;<sub>比如数组类型`int[]`和`Object[]`可以通过校验类型`arrayOf(int)`和`arrayOf(class('java/lang/Object', BL))`来表达。数组类型`byte[]`和`boolean[][]`可以通过校验类型`arrayOf(byte)`和`arrayOf(arrayOf(boolean))`来表达。</sub>

下面给出剩下的校验类型：

- 校验类型`top`、`oneWord`、`twoWord`、`reference`通过Prolog原子表达，这些名字代表询问中的校验类型。（由于不懂Prolog，这里的询问原文是question，不知道是啥意思。）
- 校验类型`uninitialized(Offset)`的表达方式是一个函子`uninitialized`加一个参数，这个参数代表`Offset`的数值。

校验类型的子类型规则如下。

子类型是自反的。

```
isAssignable(X, X).
```

如果校验类型不是Java语言中的引用类型，那么它们具有的子类型规则形式是：

```
isAssignable(v, X) :- isAssignable(the_direct_supertype_of_v, X).
```

意思是如果`v`的直接父类型是`X`的一个子类型，那么`v`是`X`的一个子类型。规则如下：

```
isAssignable(oneWord, top).
isAssignable(twoWord, top).

isAssignable(int, X) :- isAssignable(oneWord, X).
isAssignable(float, X) :- isAssignable(oneWord, X).
isAssignable(long, X) :- isAssignable(twoWord, X).
isAssignable(double, X) :- isAssignable(twoWord, X).

isAssignable(reference, X) :- isAssignable(oneWord, X).
isAssignable(class(_, _), X) :- isAssignable(reference, X).
isAssignable(arrayOf(_), X) :- isAssignable(reference, X).

isAssignable(uninitialized, X) :- isAssignable(reference, X).
isAssignable(uninitializedThis, X) :- isAssignable(uninitialized, X).
isAssignable(uninitialized(_), X) :- isAssignable(uninitialized, X).

isAssignable(null, class(_, _)).
isAssignable(null, arrayOf(_)).
isAssignable(null, X) :- isAssignable(class('java/lang/Object', BL), X),
                        isBootstrapLoader(BL).
```

&emsp;&emsp;<sub>这些子类型规则并非得是子类型的最明显的公式。（完了，bbq了，这说的这是啥啊）对于Java语言中的引用类型的子类型规则，和其他校验类型之间存在着清晰的子类型规则划分。这种划分可以让我们在Java语言引用类型和其他校验类型之间声明一般性的子类型关系。这种关系独自持有Java引用类型在类型层级中的位置，有助于防止JVM实现对类进行过度加载。比如，为了`class(foo, L) <: twoWord`这种形式的查询，我们可不想顺着Java父类层级往上爬。（真的bbq了，一脸懵逼。）</sub>

&emsp;&emsp;<sub></sub>