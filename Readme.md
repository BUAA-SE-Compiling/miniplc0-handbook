# miniplc0 实验指导书

[TOC]

## 1. 概述

本指导书为 miniplc0 指导书。

由于本指导书还只是一个 beta 版本，因此很可能存在一些错误，如果你发现本书中有以下问题

- 逻辑性/知识性错误
- 表意模糊/错误
- 前后矛盾
- 代码不对应/错误
- 示例输出过时
- ...

欢迎积极联系助教，也可以直接提 issue 甚至 pr，勘误会有一定的加分。

## 2. 编译过程概述

**编译**（compilation）的目的是将指定语言的源代码输入翻译成目标语言输出，通常来说目标语言是某种汇编指令集。

> 粗略地说，编译过程包含了五个顺序执行的主要过程：[词法分析](#词法分析)->[语法分析](#语法分析与语义分析)->[语义分析](#语法分析与语义分析)->[代码优化](#代码优化)->[代码生成](#代码生成)，以及贯穿它们的两个辅助过程：[符号表管理](#符号表管理)和[错误处理](#错误处理)。 
>
> 词法分析、语法分析和语义分析，也被称为编译过程的**前端**（front end）；
> 代码优化和代码生成，也被称为编译过程的**后端**（back end）。

实际编程语言的应用场景会更复杂一些，比如[C语言的主要编译过程](https://en.cppreference.com/w/c/language/translation_phases)就有八个阶段；而我们实验中采用的**基于递归下降分析的语法制导分析**则是在语法分析的同时进行语义分析和代码生成。在对简单过程模型的学习后，你可以更深入地去考虑现代编程语言的编译过程，事实上C语言编译过程也是面试经典题。

通过本章节的阅读，你将：

- 了解编译过程的基本组成
- 了解一些编译技术相关的术语
- 了解栈式虚拟机的基本概念

### 2.1 词法分析

> **词法分析**（lexical analysis）是编译器读入源代码文本，并将代码划分为有具体含义的单词序列的过程，这些单词被称为**token**。
>
> 词法分析作为将代码token化的过程，也被称为tokenization；相应地，词法分析器（lexer）也被称为tokenizer。

比如在C语言的编译过程中，一行代码`int a =1;`，经过预处理可能会被视为一组token：`关键字int`、`空白符序列`、`标识符a`、`空白符序列`、`赋值运算符=`、`无符号整数字面量1`、`分号;`。

C语言在词法分析之前还专门会有对源代码的预处理以及将代码划分为逻辑行的过程，而我们的实验文法相对简单，这些过程可能会通过语法分析的一些工具函数代为实现。

#### 2.1.1 最大吞噬规则

通常来说我们的文法会有一定的二义性，但是文法本身已经足够简洁而且实用，这种情况下我们会让编译器遵循**最大吞噬规则**。

在词法分析阶段，最大吞噬可以理解为：**一个token要尽可能多地识别它可以接受的字符**。比如C语言中`returna`不会被理解为`return`和`a`，`inta`也不会被理解为`int`和`a`。

有些时候最大吞噬规则也会有例外，如果本指导书中出现了例外情况，我们会特别指出，其他情况下默认遵循最大吞噬规则。

### 2.2 语法分析与语义分析

> **语法分析**（syntactic analysis）是编译器根据词法分析结果（比如token序列）检查代码是否符合语法规则的过程，通常会生成一棵描述源代码程序语法结构的树。
>
> 语法分析也被称为parsing；相应地，语法分析器也被称为parser。
>
> **语义分析**（semantic analysis）是编译器根据语法分析的结果检查代码是否符合语义规则的过程。此过程一般会构建符号表，并向语法树添加额外的语义信息，有些时候还会执行中间代码的生成。

文法规则定义了编程语言的基本结构，就好像汉语语法中对主谓宾的顺序和组合有各种要求一样。语法分析的目的就是检查是否存在一段代码，它不符合文法规则。

对于无法通过文法规则直接表述的问题，比如变量不应该被重定义，语法分析在这种情况下就会显得鞭长莫及，此时就需要交给语义分析来完成。下面这几种为我们所熟知的C语言编译错误都是通过语义分析发现的：

- 判断变量是否发生了重定义
- 判断是否在指定了返回类型的函数中返回了空值
- 判断是否使用了错误的值类型进行赋值等
- 判断是否对声明为const的常量进行了重新赋值

#### 2.2.1 未定义行为

> **未定义行为（Undefined Behavior）**，简写为UB，指的是源代码中符合文法规则，但是在语义规则中没有相关规定的行为。通常来说，未定义行为不是编译错误，而如何理解未定义行为对应的程序操作（或决定它是不是编译错误），取决于编译器的实现者。

比如C语言中，访问没有初始化过的局部变量，是一种未定义行为：
```clike
int fun() {
    int a;
    printf("%d\n", a); // UB
}
```
C语言标准不要求局部变量被默认初始化为0，通常情况下编译器的实现者也不会，但上面的操作的确符合文法规则。在运行时，上面的程序的输出是不确定的。

由于不确定性的存在，要想编写健壮的程序，我们要尽量避免UB。而设计编程语言时，可以像Java那样给出非常严格的语义规则，也可以像C/C++那样把责任交给程序员。

#### 2.2.2 语法制导翻译

> **语法制导翻译**（syntax-directed translation）可以视为一边进行语法分析一边进行语义分析。

我们实验中采用的是：基于递归下降分析的语法制导翻译。

递归下降分析本身构建语法树的顺序就是之后在语法树上做遍历的顺序，父节点相对于子节点的生成树顺序决定了遍历的顺序（前序、后序等）。

如果让递归下降分析的父节点生成放在所有子节点生成后，并且同时进行语义分析，那么得到的动作指令序列（或中间代码）刚好满足逆波兰式。因为这个特点，这种分析方式和栈式指令集有很好的相性。由于动作序列（或中间代码）此时已经生成，因此在语义规则不太复杂的情况下，甚至可以省略语法树的构建。


### 2.3 符号表管理

> **符号表**是存储了已经被声明过的标识符的具体信息的数据结构。在语义分析阶段构造符号表，并根据分析的需要对符号表进行增加、删除、修改、查询，即是所谓的**符号表管理**。

对于编译型语言，符号表的生命周期往往和语义分析相同，最终得到的目标代码/可执行程序中，通常不会包含有关源代码中名字的信息。

符号表的形态往往取决于实际情况，在我们的mini实验中，符号表只是一个哈希表（[`std::unordered_map`](https://en.cppreference.com/w/cpp/header/unordered_map)）。而C0是拥有多级作用域的语言，如果其采用一趟扫描的编译，通常会采用栈式符号表；而对于多趟扫描的话，树形符号表会更实用一些。

### 2.4 错误处理

> **错误处理**是贯穿整个编译流程的一环，它主要负责对各编译子程序发现的错误进行记录、报告、甚至是纠正。

错误处理是一个可定制度很高的部分，比如下面的C程序：

```c
int main() {
    int a,b;
    int a;
    a = "1";
}
```

你可以简单地报错并终止编译：

```
error! line 3: identifier redeclaration 
```

也可以选择详细地输出错误的信息（位置、相关代码、上下文）并对查错和修改提出一些帮助或建议：

```
error: line 3 col 9: redeclaration of "a":
    int a;
        ^
note: previous declaration of "a" at line 2 col 9:
    int a,b;
        ^
```

还可以选择在报告错误后不终止编译，而是记录下来并跳读代码到可以继续正常编译的地方，以尽可能多地发现有效错误。甚至可以对一些未定义行为提出警告。

不夸张的说，一个优秀的错误处理程序，可以对程序员的错误处理提供极大的帮助。

### 2.5 代码优化

 > **代码优化**是为了提高诸如运行速度和内存占用等的程序性能，对编译的中间结果/目标代码进行一系列优化的行为。

如果你玩过一些策略型解密游戏，他们通常会设置一些挑战，当你采用的操作更少，但是收益更高时，就给你更高的星级评价。代码优化则是让你的程序可以取得更高评价的重要一环。

代码优化的可定制度极高，比如C程序：

```c++
int fun() {
    const int a = 1;
    int b = 2;
    return (a+b)*b+2*a*(a+b);
}
```

你可以进行下列优化：

- `a`是以字面量初始化的常量，你可以在编译期就将所有读取`a`的操作等价理解为读取`1`的；
- `a+b`在中间进行了两次求值，并且过程中`a`和`b`没有进行任何修改，你也可以将其优化为一次求值，该次求值的结果存储到一个中间变量，在第二次需要时直接访问中间变量；
- `fun`运行过程中，`a`和`b`的值都是编译期常量，因此`fun`的返回值也是编译期常量，可直接让`fun`返回`12`以消除求值的开销，甚至将所有对`fun`的调用替代为读取`12`以消除函数调用的开销。

代码优化往往需要结合目标运行平台进行特化。gcc目前是使用了一种语言无关、环境无关的中间语言，编译时gcc先将源代码转换成中间代码，对中间代码进行一系列优化之后再针对具体平台进行特殊的优化，以得到更优的可执行程序。

### 2.6 代码生成

编译的最终目的是将指定语言的源代码文件翻译成目标语言文件，通常来说目标语言是某种指令集。

我们的mini实验和C0实验均采用栈式虚拟机为目标平台。

#### 2.6.1 栈式虚拟机

> **栈式虚拟机**以栈为主要数据结构，执行指令时，操作主要发生在栈上。
>
> 栈式虚拟机的指令的操作数，通常来自栈顶，指令运行产生的数据会压到栈顶。

举一个简单的例子，对于加法运算`3+2+1`，x86可能会这么做：

```assembly
mov $3, %eax    # 将立即数3移入寄存器ax
add $2, %eax    # 将立即数2加到寄存器ax
add $1, %eax    
# 最终结果存储在寄存器ax

# 如果做了激进的优化
mov $6, %eax
```

而栈式虚拟机可能会：

```assembly
PUSH $3    # 将3压到栈顶
           # 此时栈从底到顶依次为： 3
PUSH $2    # 此时栈从底到顶依次为： 3 2
ADD        # 弹出栈顶作为右操作数，再弹出栈顶作为左操作数，两者相加得到的值再压到栈顶
           # 此时栈从底到顶依次为： 5
PUSH $1    # 此时栈从底到顶依次为： 5 1
ADD 
# 最终结果在栈顶，此时栈从底到顶依次为： 6

# 如果做了激进的优化
PUSH $6
```

从这个例子或许可以更直观的看出来：源代码的逆波兰表示和栈式虚拟机指令序列十分相似。

值得一提的是，Java的运行环境JVM也是一种栈式虚拟机，有兴趣的同学可以通过jdk提供的`javap`命令对.class字节码进行反汇编，或许可以给C0实验提供一些思路。

## 3. mini plc0 编译系统

mini plc0 是一门结合了 C0 和 PL0 的语言，当然它大部分灵感还是来源于 C0，后文中 mini 或者 plc0 均指 mini plc0。

这里我们先介绍 mini plc0 编译系统，然后再指导大家如何完成实验。

mini plc0 本身相当简单，但是它依然规定了程序最基本的一些功能：常量/变量定义、四则运算、赋值、输出。

通过本章节的实验，你将：

- ~~重新认识C\+\+~~
- 阅读 mini plc0 文法及其语义规则
- 了解编译器的基本结构
- 了解基于有限状态自动机的词法分析
- 了解递归下降分析以及基于它的代码生成
- 根据章内文档对 mini plc0 编译器的一种实现进行代码补全

实验用的源代码在[这里](https://github.com/BUAA-SE-Compiling/miniplc0-compiler)，可以先 clone 一份一遍看一遍阅读指导书。

另外再强调一次：任何抄袭抓到都是判 0 分，相信我们有这个决心，请不要碰这根红线。

### 3.1 编译系统整体结构

整个 mini plc0 编译系统包括一个编译器、一个汇编器、一个反汇编器、一个解释器和一个调试器。

这些工具共同的特点是：一旦发生错误，立即报错并停止运行。

助教会完整实现编译器、汇编器、解释器、反汇编器和调试器发送给学生；但是编译器在发送前，助教会删除部分核心逻辑并留下作为提示的代码注释。实验中，学生的任务是将编译器的逻辑补全，在用其他工具测试补全内容的正确性后提交。

#### 3.1.1 编译器

编译器可以对一份 mini plc0 源代码分别进行词法分析输出 tokens，或者语法分析输出 instructions。

输入语言是 mini plc0 ，输出的是为 mini plc0 编译系统虚拟机设计的指令序列，比如编译下面的 mini plc0 源文件

```
begin
    var a = 1;
    print(a);
end
```

其中一种可能的输出是

```
LIT 1
WRT
```

假设这个输出是 hello.plc0，我们马上还会见到这个文件。

具体的指令集建议先行阅读[miniplc0虚拟机标准](https://vm.buaasecompiling.cn)。

#### 3.1.2 虚拟机工具链

方便起见，把工具链放在一起说明，源代码在[这里](https://github.com/BUAA-SE-Compiling/miniplc0-toolchain)。

注意不同的系统下请使用对应的工具链，都可以在[这里](https://github.com/BUAA-SE-Compiling/miniplc0-toolchain/releases)找到，如果有新版本请及时更新，工具链的输出不作为评分依据。

然后首先看程序的 Usage：

```
A vm implementation for mini plc0.
Usage of miniplc0-windows-amd64.exe:
  -A, --assemble        Assemble a text file to an EPFv1 file.
  -d, --debug           Debug the file.
  -D, --decompile       Decompile without running.
  -h, --help            Show this message.
  -i, --input string    The input file. The default is os.Stdin. (default "-")
  -I, --interprete      Interprete the file.
  -o, --output string   The output file.
  -R, --run             Run the file.
```

这里 `-A` 表示汇编器，`-D` 表示反汇编器，`-d` 表示调试器，`-I` 表示解释器，`-R` 直接运行 EPF 文件。

使用的时候除了指定特定的 flag 以外还需要通过 `-i` 指定输入文件，部分选项还需要用 `-o` 指定输出文件。

汇编器可以将指令序列汇编成为一个 EPF(Executable PCode File) 文件，比如

```
$ cat hello.plc0
LIT 1
WRT
$ ./miniplc0.exe -A -i hello.plc0 -o hello.epf
$ ls
hello.epf*  hello.plc0  miniplc0-darwin-amd64*  miniplc0-linux-amd64*  miniplc0-windows-amd64.exe*
$
```

这里 `LIT 1` 表示入栈一个整数 1，`WRT` 表示输出栈顶的数字，所以这个文件就是我们 miniplc0 编译系统下的 hello world 了。

这里的 hello.epf 是我们虚拟机上的可执行文件，类似 Windows 下的 exe 文件， Linux 下的 ELF 文件，注意 EPF 文件是二进制文件。

然后尝试在虚拟机上直接运行我们刚刚编译的二进制文件。

```
$ ./miniplc0-windows-amd64.exe -R -i hello.epf
1
$
```

可以看到正常输出了 1。

我们也可以用解释器直接解释执行 plc0 文本文件，比如：

```
$ ./miniplc0-windows-amd64.exe -I -i hello.plc0
1
$
```

可以看到同样正常输出了 1。

如果我们想了解一个 EPF 文件的实际指令，可以使用反汇编器，比如：

```
$ ./miniplc0-windows-amd64.exe -D -i hello.epf -o -
LIT 1
WRT
$
```

可以看到反编译出了我们之前的 hello.plc0，注意这里 `-` 表示 stdout。

最后我们可以用调试器来单步执行，比如：

```
$ ./miniplc0-windows-amd64.exe -d -i hello.epf
Simple miniplc0 debugger.
You can use the abbreviation of a command.
[H]elp -- Show this message.
[N]ext -- Run a single instruction.
[L]ist n -- List n instructions.
[S]tack n -- Show n stack elemets.
[I]formation -- Show current information.
[Q]uit -- Quit the debugger.
>L
|       LIT 1   | <-- ip
|       WRT     | 1
>S
|       0       | <-- sp
>N
Next instruction: WRT
>S
|       0       | <-- sp
|       1       | 0
>L
|       LIT 1   | 0
|       WRT     | <-- ip
>N
1
Next instruction: ILL
>S
|       1       | <-- sp
>Q
$
```

可以看到利用调试器我们可以很方便的查看栈的变化。

如果上述工具链出现了任何问题，请及时联系助教。

### 3.2 mini plc0 文法与约束

mini plc0 的完整文法如下，如果存在理解困难，请参阅附录[EBNF](#ebnf)。

```
<字母> ::=
     'a'|'b'|'c'|'d'|'e'|'f'|'g'|'h'|'i'|'j'|'k'|'l'|'m'|'n'|'o'|'p'|'q'|'r'|'s'|'t'|'u'|'v'|'w'|'x'|'y'|'z'
    |'A'|'B'|'C'|'D'|'E'|'F'|'G'|'H'|'I'|'J'|'K'|'L'|'M'|'N'|'O'|'P'|'Q'|'R'|'S'|'T'|'U'|'V'|'W'|'X'|'Y'|'Z'
<数字> ::= '0'|'1'|'2'|'3'|'4'|'5'|'6'|'7'|'8'|'9'
<符号> ::= '+'|'-'

<无符号整数> ::= <数字>{<数字>}
<标识符> ::= <字母>{<字母>|<数字>}

<关键字> ::= 'begin' | 'end' | 'const' | 'var' | 'print'

<程序> ::= 'begin'<主过程>'end'
<主过程> ::= <常量声明><变量声明><语句序列>

<常量声明> ::= {<常量声明语句>}
<常量声明语句> ::= 'const'<标识符>'='<常表达式>';'
<常表达式> ::= [<符号>]<无符号整数>

<变量声明> ::= {<变量声明语句>}
<变量声明语句> ::= 'var'<标识符>['='<表达式>]';'

<语句序列> ::= {<语句>}
<语句> ::= <赋值语句>|<输出语句>|<空语句>
<赋值语句> ::= <标识符>'='<表达式>';'
<输出语句> ::= 'print' '(' <表达式> ')' ';'
<空语句> ::= ';'

<表达式> ::= <项>{<加法型运算符><项>}
<项> ::= <因子>{<乘法型运算符><因子>}
<因子> ::= [<符号>]( <标识符> | <无符号整数> | '('<表达式>')' )

<加法型运算符> ::= '+'|'-'
<乘法型运算符> ::= '*'|'/'
```

参考翻译表

| 中文名 | 英文名 | 中文名 | 英文名 | 中文名 | 英文名 |
| :-----: | :-----: | :-----: | :-----: | :-----: | :-----: |
| 数字 | digit | 字母 | alpha / letter | 符号 | sign |
| 无符号整数 | unsigned integer literal | 标识符 | identifier | 关键字 | keyword |
| 程序 | program | 主过程 | main process | 常量声明 | constant declarations |
| 常量声明语句 | constant declaration statement | 常表达式 | constant expression | 变量声明 | variable declarations | 
| 变量声明语句 | variable declaration statement | 语句序列 | statement sequence | 语句 | statement |
| 赋值语句 | assignment expression | 输出语句 | output statement | 空语句 | empty statement |
| 表达式 | expression | 项 | term | 因子 | factor |
| 加法型运算符 | additive operator | 乘法型运算符 | multiplicative operator|||

语义规则：

- 不能重定义：同一个标识符，不能被重复声明
- 不能使用没有声明过的标识符
- 不能给常量赋值：被声明为常量的标识符，不能出现在赋值语句的左侧
- 不能使用未初始化的变量：不能参与表达式的运算，也不能出现在赋值语句的右侧
- 无符号整数字面量的值必须在值域$\left[0, 2^{31}-1 \right]$以内


### 3.3 词法分析

从这里开始我们推荐配合 tokenizer/tokenizer.h 和 tokenizer/tokenizer.cpp 阅读。

#### 3.3.1 token

直接查看 tokenizer/token.h 中定义的 miniplc0 的 token 种类：

```
// tokenizer/token.h

namespace miniplc0 {

	enum TokenType {
		NULL_TOKEN, // 仅仅为了内部实现方便，不应该出现在分析过程
		UNSIGNED_INTEGER, // 无符号整数
		IDENTIFIER, // 标识符
		BEGIN, // 关键字 BEGIN
		END, // 关键字 END
		VAR, // 关键字 VAR
		CONST, // 关键字 CONST
		PRINT, // 关键字 PRINT
		PLUS_SIGN, // 符号 +
		MINUS_SIGN, // 符号 -
		MULTIPLICATION_SIGN, // 符号 *
		DIVISION_SIGN, // 符号 /
		EQUAL_SIGN, // 符号 =
		SEMICOLON, // 符号 ;
		LEFT_BRACKET, // 符号 (
		RIGHT_BRACKET // 符号 )
	};
    ...
}
```

注意在完成 miniplc0 的过程中你不应该添加、删除或者修改 TokenType。

#### 3.3.2 状态图

![mini-plc0 词法分析状态图](https://i.imgur.com/8EWWERh.png)

这里给出大家我们自动机的状态图。

从图中可以看出，各token的first集是完全不重合的，在具体实现时，可以通过判断第一个非空白符后调用对应的词法分析子程序，避免一个巨大的if/switch，提高可维护性。

#### 3.3.3 为什么状态不是类成员而是局部变量

我们在实现的过程中，故意把自动机的状态设计成了一个局部变量，也就是说每次分析的时候自动机一定是从开始状态开始的。

另外一种可能实现是把自动机的状态设计为类成员，也就是说每次开始一次新的分析，自动机的状态停留在上次结束的状态。

这两种实现都是完全可以的，对于第一种实现虽然会略微增加一些编码量，但是有两点优点

- 自动机状态转移稍微简洁直观一些
- 减少一定脑力负担

其中微妙的区别大家在设计 C0 编译器可能会有所感受。

#### 3.3.4 缓冲区设计

缓冲区设计其实是一个很复杂的话题，在 miniplc0 中为了大家理解方便，我们实现的非常简单粗暴：

- 一次读入文件所有内容
- 基于行的缓冲区，包含换行符，统一为 `\n`
- 一个指针指向下一个要读取的字符
- 和词法分析器完全耦合

因为我们在词法分析的时候经常需要预读，所以要么实现 peek 要么实现 unread，缓冲区这样实现无论是 peek 还是 unread 都非常方便。

当然这里有很多实现思想都是有问题的，比如不应该一次全部读入，但是 miniplc0 的重点不在这里，有关具体的细节可以参考 tokenizer.h 的注释。

### 3.4 语法分析

从这里开始我们推荐配合 analyser/analyser.h 和 analyser/analyser.cpp 阅读。

#### 3.4.1 缓冲区设计

这里首先要注意的一点是，对于语法分析器，输入不再是字符序列，而应该是 token 序列。

和词法分析器的缓冲区类似，我们一次读入全部 token 存起来，用一个指针指向下一个 token。

#### 3.4.2 符号表设计

在 miniplc0 中，为了简单起见，符号表是和语法分析器直接耦合的，在递归下降的过程中逐步填入符号表。

在 mini plc0 虚拟机中，变量寻址都是通过栈偏移完成的，`stack[0]` 就是第一个元素（栈底），`stack[1]` 就是第二个元素等等。

因此符号表的设计就是字符串到偏移的映射，这样在生成指令的时候我们就可以知道相应的操作数。

同时在实现的时候我们选择把不同类型的符号分开来存，而不是所有符号放在一张大表。

### 3.5 代码生成

miniplc0 在编译后会运行在一个我们特制的虚拟机上，这个虚拟机只有栈没有堆，有专门的代码区，寄存器只有 ip 和 sp。

由于课程本身不包含虚拟机设计，因此虚拟机实现不作要求，但是我们非常推荐能自己亲自实现一个简单的虚拟机用于测试，这也是我们给出具体标准的原因。

虚拟机的实现标准在[这里](https://vm.buaasecompiling.cn)，一个非常简单的实现在 tests/simple_vm.hpp 供参考。

### 3.6 错误处理

从这里开始我们推荐配合 error/error.h 阅读。

同样是为了简单，mini plc0 直接用 enum 代表不同的错误，同时在 fmts.hpp 我们通过一个巨大的 switch case 把它们转换为对应的字符串。

### 3.7 测试

最后提到的这部分不属于编译的过程，但是属于软件工程的过程，那就是测试。

一遍写出正确的代码几乎是不可能的，这对我们也是一样，为了获得更多的分数我们非常推荐学生写尽量多的测试。

在 miniplc0 项目中使用的框架是 catch2，在 tests/ 文件夹下我们给出了 simple_vm.hpp 方便测试，同时给出了一些单元测试的例子，更多的可以参考附录。

### 3.8 如何进行实验

到这里大部分文件应该都已经看过了，可以开始实验了。

#### 3.8.1 IDE选择

CMake 本身是跨平台的，因此 IDE 选择上就简单许多了。

##### 3.8.1.1 Visual Studio

为什么把这个放在第一个呢？因为这是助教用的IDE（溜

最大的优点就是无敌的 Intellisense，用过的都说好（溜x2

要注意的是 Visual Studio 默认是不支持 CMake 的，需要安装相关 package，可以参考[这里](https://docs.microsoft.com/en-us/cpp/build/cmake-projects-in-visual-studio?view=vs-2019)

优点

- Intellisense 太智能了
- 可视化 CMake build/rebuild/clean操作
- 可视化 test case 通过率
- 有些非标准写法也能编译通过

缺点

- 注意，有些 MSVC 编译通过的写法可能在测试环境不通过
- 体积比较大
- Windows only

##### 3.8.1.2 CLion

JetBrains 出品的 C/C++ IDE，只支持 CMake 管理的项目，整体体验尚可，在 Windows 上注意要配好 mingw64。

优点

- 稍轻量
- 代码提示还算不错

缺点

- Indexing 卡
- 因为 indexing 容易卡，代码提示容易卡

##### 3.8.1.3 Visual Studio Code

相对来说 VSC 的配置复杂一些，需要自己安装 CMake 和 C/C++ 插件配置，定制化程度较高。

优点

- 最轻量

缺点

- 代码提示稍弱

总之，用什么 IDE 都无所谓，选择一款趁手的就行。

#### 3.8.2 编译产物

虽然我们给出的编译器代码是残缺的，但是也是能正常编译的。

项目编译之后会生成一个 miniplc0 可执行文件，可以查看 Usage 有

```
$ ./miniplc0.exe --help
help called

Usage: miniplc0 [options] input

Positional arguments:
input           speicify the file to be compiled.

Optional arguments:
-h --help       show this help message and exit
-t              perform tokenization for the input file.
-l              perform syntactic analysis for the input file.
-o --output     specify the output file.[Required]
```

这里 -t 和 -l 表明是进行词法分析还是语法分析，-o 表示输出的文件，如果没有指定默认输出到 stdout，对于 hello.plc0 如果实现正确的话一种可能的输出是这样的

```
$ ./miniplc0.exe hello.plc0 -l
LIT 0
LIT 1
STO 0
LOD 0
WRT
$ ./miniplc0.exe hello.plc0 -t
Line: 0 Column: 0 Type: Begin Value: begin
Line: 1 Column: 1 Type: Var Value: var
Line: 1 Column: 5 Type: Identifier Value: a
Line: 1 Column: 7 Type: EqualSign Value: =
Line: 1 Column: 9 Type: UnsignedInteger Value: 1
Line: 1 Column: 10 Type: Semicolon Value: =
Line: 2 Column: 1 Type: Print Value: print
Line: 2 Column: 6 Type: LeftBracket Value: (
Line: 2 Column: 7 Type: Identifier Value: a
Line: 2 Column: 8 Type: RightBracket Value: )
Line: 2 Column: 9 Type: Semicolon Value: =
Line: 3 Column: 0 Type: End Value: end
$
```

注意上面生成的指令序列并不唯一。

#### 3.8.3 具体实验内容

具体实验分为两部分，词法分析器和语法分析器

##### 3.8.3.1 词法分析器

词法分析器的目标是对输入的字符序列分析得到 tokens，如前面所述采用自动机实现，因此学生需要补全的是 tokenizer/tokenizer.cpp 的 nextToken 函数。

需要注意的是 nextToken 并不会直接被调用，它是外部接口 NextToken 的核心实现，至于 NextToken 怎么使用可以参考 main.cpp。

##### 3.8.3.2 语法分析器

语法分析器的目标是对 token 序列分析后生成指令序列，如前面所述采用递归下降实现，学生需要补全的是 analyser/analyser.cpp 的系列函数。

##### 3.8.3.3 对修改的要求

为了方便测试，请务必注意在补全的时候是有一些限制的。

你可以

- 在 analyser.cpp 或者 tokenizer.cpp 的 miniplc0 命名空间下添加全局的辅助函数
- 完全重新实现 nextToken 和 analyse***** 函数
- 修改 ErrorCode 的可能值，但是注意如果你添加或者删除了 ErrorCode 请修改 fmts.hpp 中的 format 函数。
- 添加新的 include，但是仅限于标准库和项目除了 3rd_party 以外的文件。

你不能

- 修改 CMakeLists.txt。
- 添加和修改任何新的源代码文件。
- 修改除了 error.h, fmts.hpp, analyser.cpp, tokenizer.cpp, test_analyser.cpp, test_tokenizer.cpp 以外的任何文件。
- 修改 error.h 中除了 ErrorCode 取值以外的代码。
- 修改 fmts.hpp 中除了 ErrorCode 对应的 switch case 以外的代码。
- 修改 analyser.cpp 中 analyser***** 以外的函数。
- 修改 tokenizer.cpp 中 nextToken 以外的函数。
- 添加任何宏，包括但不限于 define, ifdef 等。
- 添加任何 using，比如 using namespace std。
- 修改任何已有函数的签名。

如果是设计上本身的缺陷需要修改不能修改的文件/函数，请尽快联系我们。

#### 3.8.4 我们怎么评测代码

我们会采用半自动的方式来评测代码。

首先我们不会依赖提交的实验中的 tests，因此即使你 tests 写错了也不会影响我们评测。

我们会用一个特殊的脚本把整个项目变形为 OJ 可接受的源码，然后补全输入输出调用源码中的 Tokenizer 和 Analyser 分析输入提交到 OJ 上自动化跑样例，有点类似 Special Judge。

同时我们也会手动去检查代码，比如抄袭问题等，并且避免 OJ 可能存在的误判。

### 3.9 提交方式和要求

最后编译环境以 [C0 编译环境](https://hub.docker.com/r/lazymio/compilers-env) 为准，请务必保证在这个镜像中编译通过，使用方法见附录 D。

提交的时候把 CMakeLists.txt 所在的目录完整打包即可，不需要打包输出，格式请使用zip。
压缩包命名统一 ***学号_姓名_mini.zip***，比如 *16211070_孔子乔_mini.zip*。

此外注意提交的时候压缩包根目录应该是项目根目录，也就是说应该是

```
- 根目录
| - 3rd_party
| - analyser
| ...
```
而不要是

```
- 根目录
| - miniplc0
  | - 3rd_party
  | - analyser
  | ...
```

也不要是

```
- 根目录
| - homework 
  | - 3rd_party
  | - analyser
  | ...
```

也不要是

```
- 根目录
| - 学号_姓名_mini 
  | - 3rd_party
  | - analyser
  | ...
```

## 附录A EBNF


Extended Backus-Naur Form（扩展巴科斯范式），是[ISO/IEC 14977](http://standards.iso.org/ittf/PubliclyAvailableStandards/s026153_ISO_IEC_14977_1996(E).zip)接受的一种元语法符号表示法。

由于当前课本采用的类似EBNF的表示法在某些情况下的描述能力很差，因此我们参考了Wikipedia，以[BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)为主，结合了一些[EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form)和[ABNF](https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_form)的语法糖，得出了一种更加严谨的类似EBNF的表示法。

### A.1 BNF

一套文法，由若干条规则组成。

#### A.1.1 规则与终结符


> **约定1.** 规则使用字符序列`::=`分割，在规则左侧（`::=`左侧）出现过的符号称为**非终结符**；**没有在规则左侧出现过、也不是用以描述规则的特殊符号**的其他字符，称为**终结符**。
> 
> 在描述一条规则时，通常我们用一对尖括号`<>`包围非终结符，而用一对引号（单引号`''`或双引号`""`皆可）包围连续的任意长的终结符串（或称为**字面量**）。而没有用尖括号或是引号包围的串，**都是元符号**。


例如下面的**布尔字面量文法**包含了三条规则：

```
<bool literal> ::= <true literal> | <false literal>
<true literal> ::= 'true'
<false literal> ::= 'false'
```

其中：
- `bool literal`、`true literal`和`false literal`在规则左侧出现过，因此是非终结符，可以看到规则中他们被尖括号`<>`包围。
- 字符串`true`到`false`是字面量，可以看到规则中他们被单引号`''`包围。
- 字符串`::=`分隔规则的两侧内容
- 字符`|`表示这条规则的右侧有多种候选项，详见后文

在本附录章节的后续内容中，除了完整写出一条规则的情况，其他情况下非终结符会用尖括号包围（如`<bool literal>`），而终结符串则会直接写出（如`true`）。

> **推导** 规则右侧的内容可以由左侧的非终结符通过推导得到。
> 
> 能够由文法的起始非终结符经过有限次推导得到的终结符串，是符合该文法的字符串。

比如非终结符`<bool literal>`可以推导得出`<true literal>`，也可以推导得出`<false literal>`；非终结符`<true literal>`可以推导得出字符串`true`。


> **约定2.** 规则左侧只能有一个符号，且必须是非终结符。

比如下列都不是一个合法的规则：

- `<A><B> ::= 'ab'`
- `'a' ::= 'b'`
- `<A>'b' ::= 'ab'`

> **约定3.** 规则中的不属于某个符号（非终结符或字面量）的任何空格或换行，都不具有实际含义，只是为了美观和可读性而人为添加的。

比如，规则`<A> ::= 'a'`和规则`<A>::='a'`本质上是完全相同的规则。

再比如，规则`<bool literal> ::= <true literal> | <false literal>`也可以像下面这么写：
```
<bool literal> ::= 
     <true literal> 
    |<false literal>
```

但是规则`<space> ::= ' '`和规则`<space> ::= ''`不是一样的，因为单引号包围的所有内容都被认为是字面量。事实上，使用引号包围字面量的一个原因是为了更直观的描述空格。

> **连接** 如果规则中存在两个相邻的符号A和B（各自可以是非终结符或终结符串），那么这两个符号之间的运算就是连接。
> 
> 在进行推导时，如果符号A能够推导出字符串x，符号B能够推导出字符串y，那么符号A和B的连接AB能够推导出字符串x和y的连接xy。
> 
> 因此规则中两个字面量进行连接，可以表述为一个新的字面量，这个字面量的内容是原来两个字面量的内容的连接。

比如：

- 规则`<A> ::= <A> <B>`中`<A>`和`<B>`是相邻的。
- `<A> ::= 'a' 'a'`等价于`<A> ::= 'aa'`
- `<A> ::= 'a' ' ' 'a'`等价于`<A> ::= 'a a'`

请不要忘记，文法中不出现在非终结符或字面量中的空格，只是为了美观而添加的。

> **选择** 选择符指的是规则右侧出现的字符`|`，规则左侧的非终结符可以推导出`|`两侧的任意选项。
> 
> 连接具有比选择更高的运算符优先级。

比如：

- 规则`<zerone> ::= '0' | '1'`表示`<zerone>`可以推导出字面量`0`或`1`。
- 规则`<A> ::= <A><A>|'a'`表示`<A>`可以推导出`<A><A>`或字面量`a`。

> **约定4** 当字面量需要描述双引号`"`时，该字面量必须用一对单引号`''`包围；需要描述单引号`'`时，必须用双引号包围`""`。
> 
> 多个不同种类的引号存在时，以第一个出现的引号为字面量的开始标记，第一个和开始标记同种类的引号为结束标记。

比如规则：
- `<single quote> ::= "'"`说明字面量`'`可以由`<single quote>`推导得出 
- `<double quote> ::= '"'`说明字面量`"`可以由`<double quote>`推导得出 
- `<zero integer literal> ::= '0'` （或`<zero integer literal> ::= "0"`）描述的是整数字面量`0`
- `<zero char literal> ::= "'0'"`描述的是C语言中的字符字面量`'0'`
- `<LF char literal> ::= "'\n'"`描述的是C语言中等值于换行符（line-feed）的字符字面量`'\n'`
- `<string literal> ::= '"' <characters> '"'`描述的是C语言中的形如`"123"`的字符串字面量

> **约定5.** 不出现在非终结符或字面量时，使用字符`ε`表示空串。

比如：

-  `<empty> ::= ε` 表示`<empty>`推导出空串。
-  `<optional A> ::= 'A' | ε`表示`<optional A>`可以推导出字面量`A`，也可以推导出空串
-  `<εAε> ::= εεεε'εaε'εεεε` 等价于`<εAε> ::= 'εaε'`，不等价于`<A> ::= 'a'`

### A.2 EBNF

#### A.2.1 组

> **组** 组指的是规则右侧由一对圆括号`()`包围的符号串，该符号串被视为一个整体

比如`<A> ::= (<A>|ε)'a'`等价于`<A> ::= <A>'a'|'a'`。

#### A.2.2 可选项

> **可选项** 可选项指的是规则右侧由一对方括号`[]`包围的符号串，`[<A>]`等价于`(<A>|ε)`。

比如`<A> ::= [<A>]'a'`等价于`<A> ::= (<A>|ε)'a'`等价于`<A> ::= <A>'a'|'a'`。

#### A.2.3 重复项

> **重复项** 重复项指的是规则右侧由一对花括号`{}`包围的符号串，`{<A>}`等价于无数个`[<A>]`的连接，即`[<A>][<A>][<A>][<A>]...`。

比如`<A> ::= {'a'}`可以推导出空串、`a`、`aa`、由任意多个`a`连接得到的字面量。

### A.3 ABNF

> **约定6.** 文法中使用格式为`%x??`的符号表示ASCII码值为`??`的字符，`%x??-!!`表示ASCII码值在闭区间（包含边界值）`??`到`!!`的任意字符
> 
> **约定6最终没有采用**

此约定目的是便于描述控制字符，或是便于描述可选值域过大的字面量。

比如：
- `<space> ::= ' '` 等价于 `<space> ::= %x20`
- `<zerone> ::= %x30-31` 等价于 `<zerone> ::= %x30 | %x31` 等价于 `<zerone> ::= '0' | '1'`
- `<alpha> ::= %x41-5A | %x61-7A` 表示 `<alpha>`可以推导出任意一个英文字母（无论大小写）
- `<LF> ::= %x0A` 表示 `<LF>`可以推导出值为`0x0A`的换行符；此规则和`<LF literal> ::= '\n'`完全不同，后者推导出的是两个字节长的字面量`\n`

由于不够直观因此对阅读存在一些阻碍（比如为了理解`<space> ::= %x20`，读者需要事先知道ASCII值等于`0x20`的字符是空格），**约定6最终没有采用**，因此实际情况下，可能会用如下表述：

```
<空白符> ::= ' ' | <LF> | <CR> | <HT>
<LF> 是 ASCII值等于0x0A的字符(换行符)
<CR> 是 ASCII值等于0x0D的字符(回车符)
<HT> 是 ASCII值等于0x09的字符(水平制表符)
```

```
<alpha> 可以是ASCII值满足如下任意一个条件的任意字符：
    大于等于0x41(A)且小于等于0x5A(Z)
    大于等于0x61(a)且小于等于0x7A(z)
```

## 附录B mini plc0 整体编码风格

~~你 lazymio 助教写C\+\+用optional和pair模仿golang搞个err这样的C\+\+你喜欢吗？~~

其实上面一句就可以概括了。

由于在设计实验的时候编码风格受 golang 的影响很重，因此不由自主的把 C\+\+ 写成了 golang 风格，其中主要有两点

- 绝大多数函数需要返回一个 err 表明有没有出问题
- Error 本身是 value

实际上这样并不太 modern C\+\+，一个更好的方式应该是配合 exception 来完成。

不过有意思的是 Google Chromium 默认关闭了 C\+\+ 的异常，如果一个函数需要抛出一个异常，必须这样写

```C++
void add(int a, int b, exception_context& ctx);
```

由调用者去检查异常的结果并处理，这样的好处是如果函数签名没有 ctx，那么一定不会抛出异常。

## 附录C 一些C++知识

### C.1 Why C++?

一开始思考用什么语言实现 mini plc0 编译器的时候我们也有很多考虑。

首先考虑哪些语言是1721已经学过的：

- C/C++
- Java

其实也就三个可选，C 虽然语法简单，但是很容易浪费精力在造轮子上，Java 似乎是一个很好的选择但是我们都不太想写（溜），所以最后就是 C\+\+ 了。

C++提供了功能多样且强大的[标准库]( https://en.cppreference.com/w/cpp/header )，能避免很多需要造轮子的场景，让精力更多放在实现编译器的逻辑上。

### C.2 知识清单

由于预见到可能的各种问题，下面列出一些你可能需要了解的知识。

如果有不会的地方首先可以考虑询问助教，所有语言相关的问题我们都会尽力解答。从自学的角度，建议先参考 [cppreference](https://en.cppreference.com)（[中文版](https://zh.cppreference.com)）。

- 语法
  - [作用域](https://en.cppreference.com/w/cpp/language/scope)
  - [`auto`]( https://en.cppreference.com/w/cpp/language/auto )与[`decltype`](https://en.cppreference.com/w/cpp/language/decltype) (C++11)： 将类型推导交给编译器
  - [lambda](https://en.cppreference.com/w/cpp/language/lambda) (C++11)： 匿名函数
  - [scoped enumerations](https://en.cppreference.com/w/cpp/language/enum#Scoped_enumerations) (C++11)：不污染符号表的枚举类
- 工具
  - [std::optional](https://en.cppreference.com/w/cpp/utility/optional) (C++17)： 提供 NULL 的语义。
  - [std::any](https://en.cppreference.com/w/cpp/utility/any) (C++17)： 提供类型安全的类似动态类型的功能。
  - [std::variant](https://en.cppreference.com/w/cpp/utility/variant) (C++17)： 提供类型安全的类似union的功能。
- [容器]( https://en.cppreference.com/w/cpp/container )
  - [std::string]( https://en.cppreference.com/w/cpp/string/basic_string )： 可修改的变长的字符串
  - [std::vector]( https://en.cppreference.com/w/cpp/container/vector )： 变长的容器
  - [std::pair](https://en.cppreference.com/w/cpp/utility/pair)： 存储一对值的容器
  - [std::map]( https://en.cppreference.com/w/cpp/container/map )： 以键排序的键值对容器
  - [std::unordered_map]( https://en.cppreference.com/w/cpp/container/unordered_map ) (C++11)： 基于hash的无序键值对容器

### C.3 CMake/Catch2

CMake 是一个跨平台的编译系统，它的工作方式很好理解：首先把不同编译器的通用选项抽象出来，然后根据不同的平台生成不同的编译系统，然后再用相应的编译系统去编译。

比如我们知道在 Linux 上一般用的是 MakeFile（当然还有一坨 autoxxx 来生成 MakeFile，这里不提），而 CMake 的工作方式就是按 CMakeLists.txt 生成一个带 MakeFile 的项目然后 MakeFile 会调用相应的编译器去编译生成目标。 

和传统的 configure->make->make install 三部曲类似，对于一个 CMake 项目如果没有特殊说明一般是

```shell=bash

mkdir build

cd build && cmake ..

make
```

其实 CMake 的工作方式就很好的体现了编译原理前后端的思想啦。

Catch2 是一个 C++ 单元测试框架，这里不多做介绍，实验中给出了一个例子可以参考，更多的可以查看官方教程 https://github.com/catchorg/Catch2/blob/master/docs/tutorial.md#top

### C.4 fmt/argparse

这里两个库是为了写代码方便用的。

fmt 是提供了类似 C# 中格式化字符串的能力，毕竟 iostream 太烂了。

argparse 用来格式化命令行参数。

## 附录D Build with docker

### D.1 Linux

首先我们从最新的 image 创建一个 container。

```bash
# pull latest image
docker pull lazymio/compilers-env
# -t --tty
# -d --detach
docker run -t -d --name mycontainer lazymio/compilers-env
# open a shell in the container
docker exec -it mycontainer /bin/bash
```

注意从这里开始我们是在 container 内执行指令。

接下来先编译。

```bash
cd ~
git clone https://github.com/BUAA-SE-Compiling/miniplc0-compiler
cd miniplc0-compiler
git submodule update --init --recursive
mkdir build
cd build
cmake ..
make
```

然后如果直接运行可以

```bash
./miniplc0 --help
```

想运行测试可以

```bash
make test
```

再次提醒：在这个 image 下的产物的输出作为最终判定的结果。

### D.2 FAQ

#### D.2.1 docker pull 好慢

换源，具体问 Google。

#### D.2.2 Windows/Mac 怎么办

我建议虚拟机，包括 mac。

#### D.2.3 代码都在 container 里怎么上 IDE 呀

docker volume / mount 请自行探索。

或者也可以先在本地写，提交前测试改好。
