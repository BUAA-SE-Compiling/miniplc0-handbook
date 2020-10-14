# Miniplc0 Java 版指导书

## 0. 环境配置

运行 Java 版本的 Miniplc0 需要以下环境

- 支持至少 Java 11 的运行环境（推荐 OpenJDK）
- Gradle 6.0 以上

## 3. mini plc0 编译系统

mini plc0 是一门结合了 C0 和 PL0 的语言，当然它大部分灵感还是来源于 C0，后文中 mini 或者 plc0 均指 mini plc0。

这里我们先介绍 mini plc0 编译系统，然后再指导大家如何完成实验。

mini plc0 本身相当简单，但是它依然规定了程序最基本的一些功能：常量/变量定义、四则运算、赋值、输出。

通过本章节的实验，你将：

- ~~不会重新认识 Java~~
- 阅读 mini plc0 文法及其语义规则
- 了解编译器的基本结构
- 了解基于有限状态自动机的词法分析
- 了解递归下降分析以及基于它的代码生成
- 根据章内文档对 mini plc0 编译器的一种实现进行代码补全

实验用的源代码在 [这里](https://github.com/BUAA-SE-Compiling/miniplc0-java)，可以先 clone 一份然后一边看代码一边阅读指导书。

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

|    中文名    |             英文名             |    中文名    |         英文名          |  中文名  |        英文名         |
| :----------: | :----------------------------: | :----------: | :---------------------: | :------: | :-------------------: |
|     数字     |             digit              |     字母     |     alpha / letter      |   符号   |         sign          |
|  无符号整数  |    unsigned integer literal    |    标识符    |       identifier        |  关键字  |        keyword        |
|     程序     |            program             |    主过程    |      main process       | 常量声明 | constant declarations |
| 常量声明语句 | constant declaration statement |   常表达式   |   constant expression   | 变量声明 | variable declarations |
| 变量声明语句 | variable declaration statement |   语句序列   |   statement sequence    |   语句   |       statement       |
|   赋值语句   |     assignment expression      |   输出语句   |    output statement     |  空语句  |    empty statement    |
|    表达式    |           expression           |      项      |          term           |   因子   |        factor         |
| 加法型运算符 |       additive operator        | 乘法型运算符 | multiplicative operator |          |                       |

语义规则：

- 不能重定义：同一个标识符，不能被重复声明
- 不能使用没有声明过的标识符
- 不能给常量赋值：被声明为常量的标识符，不能出现在赋值语句的左侧
- 不能使用未初始化的变量：不能参与表达式的运算，也不能出现在赋值语句的右侧
- 无符号整数字面量的值必须在值域$\left[0, 2^{31}-1 \right]$以内


### 3.3 词法分析

从这里开始我们推荐配合 `src/main/java/miniplc0java/tokenizer` 文件夹下的文件阅读。

#### 3.3.1 token

直接查看 `src/main/java/miniplc0java/tokenizer/TokenType.java` 中定义的 miniplc0 的 token 种类：

```java
// tokenizer/TokenType.java
package miniplc0java.tokenizer;

public enum TokenType {
    None,        // 仅仅为了内部实现方便，不应该出现在分析过程
    Uint,        // 无符号整数
    Ident,       // 标识符
    Begin,       // 关键字 BEGIN
    End,         // 关键字 END
    Var,         // 关键字 VAR
    Const,       // 关键字 CONST
    Print,       // 关键字 PRINT
    Plus,        // 符号 +
    Minus,       // 符号 -
    Mult,        // 符号 *
    Div,         // 符号 /
    Equal,       // 符号 =
    Semicolon,   // 符号 ;
    LParen,      // 符号 (
    RParen,      // 符号 )
    EOF          // 文件结尾
}
```

注意在完成 miniplc0 的过程中你不应该添加、删除或者修改 TokenType。

#### 3.3.2 状态图

![mini-plc0 词法分析状态图](img/dfa.png)

这里给出大家我们自动机的状态图。

从图中可以看出，各token的first集是完全不重合的，在具体实现时，可以通过判断第一个非空白符后调用对应的词法分析子程序，避免一个巨大的if/switch，提高可维护性。

同时这里有一点必须说明的是，对于形如 `123abc` 这种 token，我们在读入到 a 时候实际实现会跳转到标识符，但是由于我们要求标识符不能以数字开头，因此会返回它是一个无效的标识符 `123abc` 而不是分析为 `123` 和 `abc`，这里出于文法的考虑没有画出这个跳转，在实现的时候请务必留意。

#### 3.3.3 我的状态机呢？

因为各个 token 的 first 集完全不重合，为了减少实现时的负担，Java 版本直接删除了 C++ 版本中的状态机，改为根据第一个非空白符调用相应的子程序。当然，如果你觉得 C++ 里的状态机也不错，依然可以抄过来用。

#### 3.3.4 缓冲区设计

缓冲区设计其实是一个很复杂的话题，在 miniplc0 中为了大家理解方便，我们实现的非常简单粗暴：

- 一次读入文件所有内容
- 基于行的缓冲区，包含换行符，统一为 `\n`
- 一个指针指向下一个要读取的字符
- 和词法分析器完全耦合

因为我们在词法分析的时候经常需要预读，所以要么实现 peek 要么实现 unread，缓冲区这样实现无论是 peek 还是 unread 都非常方便。

当然这里有很多实现思想都是有问题的，比如不应该一次全部读入，但是 miniplc0 的重点不在这里，有关具体的细节可以参考 tokenizer.h 的注释。

### 3.4 语法分析

从这里开始我们推荐配合 `src/main/java/miniplc0java/analyser` 文件夹阅读。

#### 3.4.1 缓冲区设计

这里首先要注意的一点是，对于语法分析器，输入不再是字符序列，而应该是 token 序列。

这里，我们采用的是迭代器模式，将词法分析器耦合了进来。每次在词法分析器中调用 `nextToken()` 函数时，词法分析器都会向前读入下一个 token 并把这个 token 返回给我们。

#### 3.4.2 辅助函数

为了减少实现负担，我们实现了一个可以预读单个 token 的预读系统以及相应的辅助函数。辅助函数一共有 5 个，分别对应编写时会遇到的不同场景。下面分别介绍这些辅助函数。

##### `Token next()`

输出并前进一个 token。

##### `Token peek()`

偷看下一个 token。多次调用 `peek()` 会返回同一个 token，调用 `peek` 后第一次调用 `next` 会返回与之前相同的 token。

##### `boolean check(TokenType tt)`

偷看下一个 token，如果类型与 tt 相同则返回 `true`，否则返回 `false`。

##### `Token nextIf(TokenType tt)`

偷看下一个 token，如果 token 的类型与 tt 相同则前进一个 token 并返回它，否则返回 `null`。相当于先调用 `check` 再调用 `next`。

##### `Token expect(TokenType tt)`

期待下一个 token 的类型与 tt 相同。如果相同则返回这个 token，否则直接抛出异常。相当于会抛异常版的 `nextIf`。

#### 3.4.3 符号表设计

在 miniplc0 中，为了简单起见，符号表是和语法分析器直接耦合的，在递归下降的过程中逐步填入符号表。

在 miniplc0 虚拟机中，变量寻址都是通过栈偏移完成的，`stack[0]` 就是第一个元素（栈底），`stack[1]` 就是第二个元素等等。

因此符号表的设计就是字符串到偏移的映射，这样在生成指令的时候我们就可以知道相应的操作数。

同时在实现的时候我们选择把不同类型的符号分开来存，而不是所有符号放在一张大表。

### 3.5 代码生成

miniplc0 在编译后会运行在一个我们特制的虚拟机上，这个虚拟机只有栈没有堆，有专门的代码区，寄存器只有 ip 和 sp。

由于课程本身不包含虚拟机设计，因此虚拟机实现不作要求，但是我们非常推荐能自己亲自实现一个简单的虚拟机用于测试，这也是我们给出具体标准的原因。

虚拟机的实现标准在[这里](https://vm.buaasecompiling.cn)，一个非常简单的实现在 tests/simple_vm.hpp 供参考。

### 3.6 错误处理

错误？抛异常就完事了。

### 3.7 测试

最后提到的这部分不属于编译的过程，但是属于软件工程的过程，那就是测试。

一遍写出正确的代码几乎是不可能的，这对我们也是一样，为了获得更多的分数我们非常推荐学生写尽量多的测试。

在 miniplc0-Java 项目中使用的框架是 JUnit。_TODO: 测试环节还没写完_

### 3.8 如何进行实验

到这里就可以开始实验了，首先请 **clone** 一份[miniplc0-java](https://github.com/BUAA-SE-Compiling/miniplc0-java)的源代码，不要直接 Download ZIP，其余 git 的使用这里不再赘述。

**注意：虽然我们会尽量保证在实验过程中 miniplc0-java 的 master 分支不再有任何更新，但是由于各种不可预见的意外，master 分支可能会有一些微小的修复。如果 master 分支有更新，我们会在论坛/实验课进行通知，请务必留意，同时为了避免评测问题，请在每次开始实验前先行 git pull。最后提交的时候如果是旧版代码库我们会酌情处理，但是由此带来的所有问题由学生承担。**

#### 3.8.1 IDE选择

Gradle 本身是跨平台的，因此 IDE 选择上就简单许多了。

注意虽然我们是在 Container 中评测，但是实际开发并不一定需要 Container，一般开发方式有以下几种：

- 配置远程调试，更多的请见[讨论](https://github.com/BUAA-SE-Compiling/miniplc0-handbook/issues/3)，优点是调试环境和最终测试环境一致。
- 利用本地的编译器完成实验，提交前在 Container 内测试好，优点是上手快一些，但是提交前请务必测试好。

第一种我们不在这里介绍，以下都是基于第二种介绍。当然它们很多也可以支持远程调试，请自行探索。

##### 3.8.1.1 IntelliJ IDEA

JB 公司出品的跨平台 Java IDE。

优点

- IDE 强强，开箱即用
- 用起来爽

缺点

- 助教没用过

##### 3.8.1.2 Visual Studio Code

轻量级代码编辑器，安装 Red Hat 出品的 Java 插件即可快乐编码。

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
Line: 1 Column: 10 Type: Semicolon Value: ;
Line: 2 Column: 1 Type: Print Value: print
Line: 2 Column: 6 Type: LeftBracket Value: (
Line: 2 Column: 7 Type: Identifier Value: a
Line: 2 Column: 8 Type: RightBracket Value: )
Line: 2 Column: 9 Type: Semicolon Value: ;
Line: 3 Column: 0 Type: End Value: end
$
```

注意上面生成的指令序列并不唯一。

#### 3.8.3 具体实验内容

具体实验分为两部分，词法分析器和语法分析器，**学生应该阅读编译器代码后补全编译器使其能正常分析 miniplc0 源代码并输出目标虚拟机指令。**

##### 3.8.3.1 词法分析器

词法分析器的目标是对输入的字符序列分析得到 tokens，如前面所述采用自动机实现，因此学生需要补全的是 tokenizer/tokenizer.cpp 的 nextToken 函数，在源代码中会有相应的提示，比如[这里](https://github.com/BUAA-SE-Compiling/miniplc0-compiler/blob/ebc238d8facf7514de3eb29e6a3393e2bc57e29f/tokenizer/tokenizer.cpp#L124)

```C++
			case UNSIGNED_INTEGER_STATE: {
				// 请填空：
				// 如果当前已经读到了文件尾，则解析已经读到的字符串为整数
				//     解析成功则返回无符号整数类型的token，否则返回编译错误
				// 如果读到的字符是数字，则存储读到的字符
				// 如果读到的是字母，则存储读到的字符，并切换状态到标识符
				// 如果读到的字符不是上述情况之一，则回退读到的字符，并解析已经读到的字符串为整数
				//     解析成功则返回无符号整数类型的token，否则返回编译错误
				break;
			}
```

学生可以按照注释填写，也可以完全重新实现 nextToken。

补全后编译器应该能对 miniplc0 进行正常的词法分析，学生可以通过编译器的输出来判断。

需要注意的是 nextToken 并不会直接被调用，它是外部接口 NextToken 的核心实现，至于 NextToken 怎么使用可以参考 main.cpp。

##### 3.8.3.2 语法分析器

语法分析器的目标是对 token 序列分析后生成指令序列，如前面所述采用递归下降实现，学生需要补全的是 analyser/analyser.cpp 的系列函数，同样在源代码中会有相应的提示，比如[这里](https://github.com/BUAA-SE-Compiling/miniplc0-compiler/blob/ebc238d8facf7514de3eb29e6a3393e2bc57e29f/analyser/analyser.cpp#L37)

```C++
	std::optional<CompilationError> Analyser::analyseMain() {
		// 完全可以参照 <程序> 编写

		// <常量声明>

		// <变量声明>

		// <语句序列>
		return {};
	}
```

学生可以按照注释填写，也可以完全重新实现。

补全后编译器应该能对 miniplc0 进行正常的语法分析，可以通过编译器输出的指令来判断。

##### 3.8.3.3 对修改的要求

你可以：

- 修改任何代码，但是要保证最终程序能运行

你不能：

- 抄作业

~~你可以~~

- ~~修改 analyser/analyser.cpp 的~~
    - ~~Analyser::analyseProgram~~
    - ~~Analyser::analyseMain~~
    - ~~Analyser::analyseVariableDeclaration~~
    - ~~Analyser::analyseStatementSequence~~
    - ~~Analyser::analyseConstantExpression~~
    - ~~Analyser::analyseExpression~~
    - ~~Analyser::analyseAssignmentStatement~~
    - ~~Analyser::analyseOutputStatement~~
    - ~~Analyser::analyseItem~~
    - ~~Analyser::analyseFactor~~
- ~~在下面这些类中添加新的私有函数和私有成员变量，注意要同时修改 .h 和 .cpp ~~
    - ~~Analyser~~
    - ~~Tokenizer~~
- ~~修改 tokenizer/tokenizer.cpp 的~~
    - ~~Tokenizer::nextToken~~
- ~~修改 tests 下的~~
    - ~~test_analyser.cpp~~
    - ~~test_tokenizer.cpp~~
- ~~修改 error/error.h 的~~
    - ~~ErrorCode 的可能值~~
- ~~修改 fmts.hpp 的~~
    - ~~fmt::formatter<miniplc0::ErrorCode>::format 函数中相应的 switch~~
- ~~修改 .gitignore~~
- ~~在 analyser.cpp 或者 tokenizer.cpp 的 miniplc0 命名空间下添加全局的辅助函数~~
- ~~完全重新实现 Tokenizer::nextToken 和 Analyser::analyse***** 函数~~
- ~~修改 ErrorCode 的可能值，但是注意如果你添加或者删除了 ErrorCode 必须修改 fmts.hpp 中的 format 函数。~~
- ~~添加新的 include，但是仅限于标准库和项目内除了 3rd_party 以外的文件。~~
- ~~在本地的 .git 上进行提交~~
- ~~同步到 GitHub，但是实验期间我们建议使用 private repo。~~

~~你不能~~

- ~~修改除了上面提到的可修改文件以外的任何文件。~~
- ~~添加和删除任何文件。~~
- ~~修改 error.h 中除了 ErrorCode 取值以外的代码。~~
- ~~修改 fmts.hpp 中除了 ErrorCode 对应的 switch case 以外的代码。~~
- ~~修改 analyser.cpp 中 analyse***** 以外的**已有**函数和成员变量。~~
- ~~修改 tokenizer.cpp 中 nextToken 以外的**已有**函数和成员变量。~~
- ~~修改 tests/test_main.cpp~~
- ~~修改 .gitmodules 以及 3rd_party 下 submodule 的版本~~
- ~~添加任何宏，包括但不限于 define, ifdef 等。~~
- ~~添加任何 using，比如 using namespace std。~~
- ~~修改任何已有函数的签名。~~

~~**在所有评测开始前，我们会先检测其余文件的完整性，方式是diff甚至是算hash，如果你修改了不应该修改的文件或者没有使用最新的代码库一定会影响成绩，建议提交前自己对着 master 分支的代码 diff 一遍。**~~

如果是设计上本身的缺陷需要修改不能修改的文件/函数，请尽快联系我们。

#### 3.8.4 我们怎么评测代码

我们会采用半自动的方式来评测代码。

首先我们不会依赖提交的实验中的 tests，因此即使你 tests 写错了也不会影响我们评测。

其次我们不会比较你的指令序列，比如对于

```
begin
    var a=0;
    a = 2*5*10;
    print(a);
end
```

你补全后的编译器输出

```
LIT 100
WRT
```

是完全没问题的。

我们会用类似 OJ 的形式去自动化跑样例，编译执行的步骤和附录D一致，同时我们也会手动去检查代码，比如抄袭问题等，并且避免可能存在的误判。


## 附录C 一些 Java 知识

### Gradle

Gradle 是一个适用于 Java 的工具链，可以自动安装依赖、编译、运行你的程序。Gradle 的官网是 <https://gradle.org>。

当然，这个实验中我们只会用到 gradle 的编译功能。编译和运行这个项目的操作如下：

```shell=bash
# 编译出来一个包含所有依赖的 Jar (Java 存档包) 文件
gradle fatjar

# 运行这个 Jar 文件
java -jar ./build/libs/miniplc0java.jar
```

我们不推荐使用 `gradle run` 来运行，因为它
