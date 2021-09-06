## sed

### sed基本语法和操作原理

#### 1. 语法描述

sed 命令行的基本格式如下：

```js
sed [option] 'script' file1 file2 ...
sed [option] -f scriptfile file1 file2 ..
```

发现这个和awk的命令一模一样，现在理解起来也比较容易，sed命令常见的参数如下:

- `-n` 默认情况下，模式空间中的内容在处理完成后将会打印到标准输出，该选项可以让其不打印，相当于静默模式；
- `-e` 指定要执行的命令，使用该参数，我们可以指定多个命令
- `-f` 指定包含要执行的命令的脚本文件

#### 2. 执行流程

1. 读取文件的每一行
2. 执行SED命令，默认在**模式空间**顺序执行，除非指定了行的地址，否则SED命令将会在所有行上一次执行
3. 发送修改后的内容到输出流，在发送数据之后，模式空间将会被清空

sed处理的文件既可以由标准输入重定向得到，也可以当命令行参数传入，命令行参数可以一次传入多个文件，sed会依次处理，编辑命令的基础格式其实和awk很像，依然是由 *pattern* 和 *action* 组成，其中 *pattern* 来匹配，*action* 来进行编辑操作。

```
sed [option] '/pattern/action'
```

> 注意：命令需要用单引号或者双引号引起来号； 注意：当你的命令中字符需要用到单引号时，是无法通过 '\' 来转义的，此时使用命令用双引号引起来即可。 注意：用到正则时需要用 '/' 进行标记

**举个例子**

```js
# 文件内容来自网上一个教程
shell> cat sed.txt
This is my cat
  my cat's name is betty
This is my dog
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam

shell> sed 's/This/That/g' sed.txt 
That is my cat
  my cat's name is betty
That is my dog
  my dog's name is frank
That is my fish
  my fish's name is george
That is my goat
  my goat's name is adam
```

首先命令模块就是 `s/This/That/g` , 用过 *vim*替换的一定会感觉到很熟悉，也大致会猜到将 **以行为单位处理，将文本中每行出现的 “This” 换成 “That”**，我们先拆分下命令格式，先熟悉命令格式，记住就好，至于为什么，后面会有阐述。

- `s` 表示文本操作命令 - 替换，诸如此命令的还有好多，下面会说到
- `/This/` 就是正则匹配了，表示该行匹配到的才进行后面的 *action*，记住一定要在 '/' 符号之间，当然你可以有多个正则匹配，后面也会说到用法；
- `That/g` 将前面全词匹配到的替换成 ‘That’, 另外 `/g` 代表的是所有，后面会介绍到使用其它命令来控制替换哪些。
- 上面的命令处理后输出到终端，并没有改变实际文件。

### 从一个简单的替换开始

**命令格式**

```
[address1[,address2]]s/pattern/replacement/[flags]
```

- sed在匹配前可以指定针对哪些行，这些行的指定你可以直接使用数字，也可以通过匹配得到，address 就是来指定行范围；
- pattern 模式匹配，也就是核心的正则表达式；后面业务中发现大部分时间都是在这里纠结。
- flags 替换时的功能选项

#### 1. address 控制范围的行寻址

行范围控制通常有两种，一种通过最直接的数字，另外一种通过匹配命令。先看例子:(为了更清晰的看到行寻址的结果，下面的例子将替换换成将行寻址的内容打印出来)

```js
shell> cat line.txt
1 line
2 line
3 line
4 line
5 line
6 line
7 line
8 line
shell> sed -n '3p' line.txt
3 line
```

- `3p` 打印第三行，`p` 功能为打印
- `-n` 表示静默模式，一般sed都有把所有读到的行打印出来，如果不加这个参数，它将一行行打印读到的，并且由于 `3p` 会重复打印第三行；

使用 `$` 符号来表示最后一行

```js
# 打印最后一行
shell> sed -n '$p' line.txt
8 line

# 打印从某行开始到最后一行
 sed -n '6,$p' line.txt
6 line
7 line
8 line
```

使用 `+` 操作符来间接寻址，表示从某行开始向下多少行

```js
# 打印第三行及下面2行
shell>  sed -n '3,+2p' line.txt
3 line
4 line
5 line
```

使用 `~` 指定地址范围，它使用M~N的形式，它告诉SED应该处理M行开始的每N行。例如，50~5匹配行号50，55，60，65等。

```js
# 打印奇数行
shell> sed -n '1~2 p' line.txt  
1 line
3 line
5 line
7 line
```

使用正则表达式匹配指定的行，注意必须用正斜杠将正则表达式封起来 

```js
shell> sed -n '/2/p' line.txt
2 line
```

正则匹配指定行可以和 **数字，+** 组合使用

```js
# 和数字使用
shell> sed -n '/2/,3p' line.txt
2 line
3 line

# 和 + 号使用
shell> sed -n '/2/,+3p' line.txt
2 line
3 line
4 line
5 line 
```

可以指定两个正则匹配来确定行范围，两个正则之间用逗号分隔，它表示选定两个匹配之间的行

```js
shell>  sed -n '/2/,/5/p' line.txt
2 line
3 line
4 line
5 line
```

#### 2. pattern 核心的正则匹配

sed的核心就是在于怎么玩正则表达式，这里列出一些简单的使用方法。

- `^` 表示一行的开头。如：`/^#/` 以#开头的匹配。
- `$` 表示一行的结尾。如：`/}$/` 以}结尾的匹配。
- `\<` 表示词首。 如：`\<abc` 表示以 abc 为首的詞。
- `\>` 表示词尾。 如：`abc\>` 表示以 abc 結尾的詞。
- `.` 表示任何单个字符。
- `*` 表示某个字符出现了0次或多次。
- `[ ]` 字符集合。 如：`[abc]` 表示匹配a或b或c，还有 `[a-zA-Z]` 表示匹配所有的26个字符。如果其中有^表示反，如 `[^a]` 表示非a的字符

举个例子，经常用的去掉html的tags

```js
shell> cat html.txt
<b>This</b> is what <span style="text-decoration: underline;">I</span> meant. Understand?
shell> sed 's/<[^>]*>//g' html.txt
This is what I meant. Understand?
```

- `s` 表示替换
- `/<[^>]*>/` 为正则，表示匹配 < 开始，后面出现0个或者多个非 >字符后，再以 > 结尾，把这匹配到的内容替换成空。 

#### 3. flag 帮助我们更好的针对行处理时的功能选项

这里所说的就是上面命令格式的 *flag* 部分，上面很多例子发现匹配的后面都有个 `/g` 表示改行出现的所有匹配内容都进行替换。如果不指定 *flag* 将默认只对改行匹配到的第一个做更改。

除此之外se还提供了其它几个命令

- 数字n，表示只更改该行匹配到的第n个
- `p` 只输出匹配到的行，再行寻址里面已经用过
- `w` 存储改变的行到文件，比如`sed -n 's/i/I/w junk.txt' books.txt`
- `i` 匹配时忽略大小写

### 除了替换还有哪些功能

其实sed能做的事很专注，它的强大依赖于你的正则，你可以通过各种正则技巧解决你各种奇葩问题。

上面例子看到一个 `s` 命令，这些命令也对应了sed的各个功能。

#### 1. 插入 `i`

命令格式：`[address1[,address2]]i Insert text`

例如再第一行前插入一行 "This is test file!"

```js
shell> sed '1i This is test file!' sed.txt  
This is test file!
This is my cat
  my cat's name is betty
This is my dog
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam
```

- 上面的 `1i` 就是说明再第一行之前

想想前面指定行范围，这里命令行的 address 是不是也可以像上面一样的功能。例如下面的例子就是匹配到 fish 字符后下面三行，每行前面增加 ------------!

```js
sed '/fish/,+3i ------------!' sed.txt
This is my cat
  my cat's name is betty
This is my dog
  my dog's name is frank
------------!
This is my fish
------------!
  my fish's name is george
------------!
This is my goat
------------!
  my goat's name is adam
```

- 上面发现sed就是这么灵活，很多前面看到的东西可以拿来自由组合
- 注意：当匹配到第一个 fish 就在下面3行每行之前增加；然后接下来处理的是3行之后的，如果后面还有匹配到的继续执行同样的过程。

这里再举个例子会更清楚，首先我们改了例子的文件内容

```js
shell> cat sed.txt 
This is my cat
  my cat's name is betty
This is my dog
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam
This is my cat
  my goat's name is adam
This is my goat
This is my fish
  my fish's name is george

sed '/cat/,+2i ------------!' sed.txt
------------!
This is my cat
------------!
  my cat's name is betty
------------!
This is my dog
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam
------------!
This is my cat
------------!
  my goat's name is adam
------------!
This is my goat
This is my fish
  my fish's name is george
```

- 这里要说明的是匹配到第一个cat之后，再+2总共三行之前需要插入，其实你发现匹配到cat后的一行也有cat，但并没有继续+2插入，而是相当于匹配的index+2之后继续匹配，往下又匹配到了cat，再执行同样的过程，有点绕，仔细看看例子你就会发现端倪。

#### 2. 追加 `a`

和插入功能一样，只是再匹配的行后面追加（并不是再本行追加，而是下一行）

```js
shell> sed '/cat/,+2a ------------!' sed.txt
This is my cat
------------!
  my cat's name is betty
------------!
This is my dog
------------!
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam
```

#### 3. 删除 `d`

由于sed命令是基于行为单位处理的，所以这里也是删除行，而且删除的是模式空间的缓存，只会影响输出，不会影响原来文件，格式如下：

命令格式：`[address1[,address2]]d`

例如删除匹配到cat的行和其后的2行

```js
shell> sed '/cat/,+2d' sed.txt 
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam
```

#### 4. 行替换 `c`

命令格式：`[address1[,address2]]c Replace text`

需要注意的是这里指定的行范围将会被一起替换成一行，而不是每行每行的替换，仔细观察下面的例子，将cat出现的行及后两行全部替换成一行 "replace test"

```js
shell> sed '/cat/,+2c replace test' sed.txt
replace test
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam
```

#### 5. 文件写入命令 `w`

`w` 指定是写命令，后面指定文件名，当提供了文件名但是文件不存在的时候它会自动创建，如果已经存在的话则会覆盖原文件的内容。

命令格式 `[address1[,address2]]w file`

```js
shell> sed '/cat/,+2w w.txt' sed.txt
This is my cat
  my cat's name is betty
This is my dog
  my dog's name is frank
This is my fish
  my fish's name is george
This is my goat
  my goat's name is adam
 shell> cat w.txt 
This is my cat
  my cat's name is betty
This is my dog
```

- 上面例子即使是写入，也不会影响终端输出，依然全文本输出，这也是由于模式空间的缓存都会被输出出来的原因
- 只将匹配到的内容写入新的文件

### sed的多行处理功能

前面所看到的sed编辑器命令都是针对单行数据执行操作的，在sed编辑器读取数据流时，它会基于换行符的位置将数据分成行，让后再每行中重复的执行脚本命令。除此之外sed也提供了三种可以多行处理的功能；

#### 1. 加载下一行处理 `N`

```js
shell> sed 'N; s/\n/,/g' sed.txt
This is my cat,  my cat's name is betty
This is my dog,  my dog's name is frank
This is my fish,  my fish's name is george
This is my goat,  my goat's name is adam
```

首先该例子将两行变一行，并且用逗号分隔，**我感觉这种处理模式更像是读两行放到模式匹配的缓存里，然后再使用命令处理**。下面我举个例子来佐证

```js
shell> sed 'N; s/i/I/1' sed.txt
ThIs is my cat
  my cat's name is betty
ThIs is my dog
  my dog's name is frank
ThIs is my fish
  my fish's name is george
ThIs is my goat
  my goat's name is adam
```

上面的例子是将第一个出现 i 字符换成 I，这里发现第二行出现的i并没有被替换，所以可以理解是将两行读到一起来处理命令的，或者说读了一行什么都不处理，模式空间也不清空，再读一行一起处理，最后处理完清空。

#### 2. 输出多行中的第一行 `P`

P命令用于输出N命令创建的多行文本的模式空间中的第一行,也就是说读进来两行，仅输出第一行。

```js
shell> sed -n 'N; P' sed.txt
This is my cat
This is my dog
This is my fish
This is my goat
```

### 常用命令积累

#### c++删除注释

```js
$ cat hello.cpp
#include <iostream> 
using namespace std; 
int main(void) 
{ 
   // Displays message on stdout. 
   cout >> "Hello, World !!!" >> endl;  
   return 0; // Return success. 
}
```

执行下面的命令可以移除注释

```js
$ sed 's|//.*||g' hello.cpp
#include <iostream>
using namespace std;
int main(void)
{

   cout >> "Hello, World !!!" >> endl;
   return 0;
}
```

#### 模拟grep命令

```js
$ echo -e "Line #1\nLine #2\nLine #3" | grep 'Line #1'
Line #1
$ echo -e "Line #1\nLine #2\nLine #3" | sed -n '/Line #1/p'
Line #1
```

#### 模拟grep -v命令

```js
$ echo -e "Line #1\nLine #2\nLine #3" | grep -v 'Line #1'
Line #2
Line #3
$ echo -e "Line #1\nLine #2\nLine #3" | sed -n '/Line #1/!p'
Line #2
Line #3
```

#### 过滤所有的html标签

```js
$ cat html.txt
<html>
<head>
    <title>This is the page title</title>
</head>
<body>
    <p> This is the <b>first</b> line in the Web page.
    This should provide some <i>useful</i> information to use in our sed script.
</body>
</html>                                                                                  
$ sed 's/<[^>]*>//g ; /^$/d' html.txt
    This is the page title
     This is the first line in the Web page.
    This should provide some useful information to use in our sed script.
```

#### 移除空行

```js
$ echo -e "Line #1\n\n\nLine #2" | sed '/^$/d'
Line #1
Line #2

（欢迎各位对文章中的错误指正，另外如果小伙伴们有好的实例来分享，谢谢各位
```