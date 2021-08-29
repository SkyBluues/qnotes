## awk introduction

### awk语法说明

#### 语法形式

```shell
awk [options] 'script' var=value files(s)
awk [options] -f scriptfile var=value file(s)
```

#### 常用选项

- -F fs fs指定输入分隔符，fs可以是正则表达式或字符串，如-F:
- -v var=value 赋值一个用户定义变量，将外部变量传递给awk
- -f scriptfile 从脚本文件中读取awk命令

#### 语法结构

awk是由pattern和action组成，pattern表示awk在数据中查找的内容，而action是在找到匹配内容时所执行的一系列命令。

```shell
awk '{pattern + action}' {filenames}
```

pattern可以是如下几种或者什么都没有（全部匹配）：

- /正则表达式/：使用通配符的扩展集。
- 关系表达式：使用运算符进行操作，可以是字符串或数字的比较测试。
- 模式匹配表达式：用运算符~（匹配）和~！（不匹配）。
- BEGIN语句块、pattern语句块、END语句块

action就是awk的代码，支持函数、表达式，之间由换行符或分号隔开，并位于大括号内：

- 变量或数组赋值
- 输出命令
- 内置函数
- 控制流语句

### awk常见应用和工作原理

下面列出一个最常用的awk命令结构，借此分析原理

```shell
awk 'BEGIN{commands} pattern{commands} END{commands}'
```

- 首先执行**BEGIN{commands}**内的语句块，注意这只会执行一次，经常用于变量初始化，头行打印一些表头信息，只会执行一次，在通过stdin读入数据前被执行；
- 从文件内容中读取一行，循环使用**pattern{commands}**处理文件的每一行；
- 最后执行**END{commands}**，也只执行一次。

第一个例子，获得/etc/passwd文件中每一行的第1个和第7个数据，以逗号分隔，并在第一行和最后一行打印一串文字。

```shell
shell> cat /etc/passwd | awk -F':' 'BEGIN{print "name,shell"} {print $1","$7} END{print "blue,/bin/nosh"}'
name,shell
root,/bin/bash
daemon,/usr/sbin/nologin
...
blue,/bin/nosh
```

#### 书写注意事项

- awk后的命令需要用单引号括起来
- 最后用'{}'括起来每个部分，便于阅读
- 每个'{}'可以有多个命令，用;号隔开

### 输出更友好

awk的输出主要靠**print**，**printf**指令，这两个指令的用法和c语言一样。awk处理每行时以列为每个域，列的分隔符默认为空格，但是可以通过**-F**指定，例如**-F;**表示指定分割符为分号，**-F[;,]**表示指定分隔符为分号和逗号。

通过**printf**可以控制输出的格式，它的控制语法和c语言类似：

```shell
shell> awk '{printf "%-8s %-8s %-8s %-18s\n",NR, $1,$2,$12}' top.txt
1        PID      USER     COMMAND           
2        2304     york     cinnamon          
3        12489    york     top               
4        1        root     systemd           
5        2        root     kthreadd          
6        4        root     kworker/0:0H      
7        6        root     ksoftirqd/0       
8        7        root     rcu_sched         
9        34       root     khungtaskd        
10       35       root     oom_reaper        
11       36       root     writeback         
12       37       root     kcompactd0
```

awk内置了一些变量：

| 变量        | 说明                                       |
| :---------- | :----------------------------------------- |
| ARGC        | 命令行参数的数目                           |
| ARGIND      | 命令行中当前文件的位置（从0开始算）        |
| ARGV        | 包含命令行参数的数组                       |
| CONVFMT     | 数字转换格式（默认值为%.6g）               |
| ENVIRON     | 环境变量关联数组                           |
| ERRNO       | 最后一个系统错误的描述                     |
| FIELDWIDTHS | 字段宽度列表（用空格键分隔）               |
| FILENAME    | 当前输入文件的名                           |
| FNR         | 同NR，但相对于当前文件                     |
| FS          | 字段分隔符（默认是任何空格）               |
| IGNORECASE  | 如果为真，则进行忽略大小写的匹配           |
| NF          | 表示字段数，在执行过程中对应于当前的字段数 |
| NR          | 表示记录数，在执行过程中对应于当前的行号   |
| OFMT        | 数字的输出格式（默认值是%.6g）             |
| OFS         | 输出字段分隔符（默认值是一个空格）         |
| ORS         | 输出记录分隔符（默认值是一个换行符）       |
| RS          | 记录分隔符（默认是一个换行符）             |
| RSTART      | 由match函数所匹配的字符串的第一个位置      |
| RLENGTH     | 由match函数所匹配的字符串的长度            |
| SUBSEP      | 数组下标分隔符（默认值是34）               |

#### 在处理过程中进行计算

统计出所有进程总共占了多少cpu：

```js
shell> awk 'BEGIN {sum=0} {printf "%-8s %-8s %-18s\n", $1, $9, $11; sum+=$9} END {print "cpu sum:"sum}' top.txt
PID      %CPU     TIME+             
2304     6.2      4:56.97           
12489    6.2      0:00.02           
1        0.0      0:04.35           
2        0.0      0:00.05           
4        0.0      0:00.00           
6        0.0      0:00.06           
7        0.0      0:14.82           
34       0.0      0:00.04           
35       0.0      0:00.00           
36       0.0      0:00.00           
37       0.0      0:00.00           
cpu sum:12.4
```

又看到几个新的东西，变量初始化及awk的一些基本运算

- `sum=0` 一般都在 `BEGIN` 里面初始化一个变量，如果不需要初始化可以直接进行对变量的赋值，这很像脚本语言中的自动推断，除了提供基本的运算以外（有哪些？c有哪些这就基本有哪些），还提供了一些高级计算函数,例如数学运算`log、sqr、cos、sin`, 字符运算 `length、substr`
- 引入 **awk变量** 除了能在内部使用，还可以从外部引入，使用 `-v` 参数指定，例如下面是打印环境变量的例子，这个在编写脚本中很常见。 `awk -v var=$PATH 'BEGIN {print var}' top.txt`

上面的例子引入了 awk 的运算，知道可以定义变量运算，除此之外还支持很多运算符，算术运算符，逻辑运算符，甚至一些内部提供的函数。

#### 对信息进行筛选

前面有说到awk是由 *pattern* 和 *action* 组成，其中 *pattern* 部分就是能帮我们匹配或者过滤掉一些信息，过滤方式有很多，比如条件判断，正则匹配，甚至还可以和c语言一样写 `if else`, 到此就不要把 awk 当命令了，它就是一门语言，后面还有更高级的用法。

去掉第一行，并且只输出cpu消耗大于0的行：

```js
shell> awk 'NR>1 && $9>0 {printf "%-8s %-8s %-18s\n",$1,$9,$12}' top.txt 
2304     6.2      cinnamon          
12489    6.2      top 
```

首先按照上面所介绍的 awk 执行流程来介绍，单引号里面的 *模式+命令* 在每读到一行时就会执行，判断条件也是如此 `NR>1 && $9>0` 这种写法和c语言没有两样，只是少了判断 `if` 而已，每读到一行时都执行这个判断条件来确定是否过滤；下面转换成高级语言的代码。

```js
for l in lines {
	if ( NR>1 && $9 >0 ) {
		printf ("%-8s %-8s %-18s\n",$1,$9,$12);
	}
}
```

保留表头，只输出特定用户名的进程：

```js
awk 'NR==1 || /york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt 
PID      USER     %CPU     COMMAND           
2304     york     6.2      cinnamon          
12489    york     6.2      top               
```

上面 *pattern* 出现了一个新语法 `/york/` ,这个就是正则匹配，面对一些字符串匹配来进行过滤，通过运算符显的很无力，这在处理大量log时尤为突出，awk 也想到这点，支持正则匹配来精准筛选；正则过滤有好几种运用方法，但主要格式都是 **在双斜杠内写上你的正则表达式**；例如上面的例子就是 **该行只要出现 ‘york’ 字符即可满足过滤条件**

#### 针对域精确匹配

上面例子表述的是 **该行只要出现** 即可匹配到，万一不是 *USER* 字段也有 ‘york’ 字符就会出现错误；

```js
shell> awk 'NR==1 || $2~/york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt 
PID      USER     %CPU     COMMAND           
2304     york     6.2      cinnamon          
12489    york     6.2      top 
```

上面引入了 `~` 运算符，该运算符和 `!~` 成对立关系，类似 `=`和`！=`的关系，前者表示匹配到的输出，后者表示匹配到的过滤。假设过滤 ‘york’ 的输出，可以这样写：

```js
shell> awk 'NR==1 || $2!~/york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt 
PID      USER     %CPU     COMMAND           
1        root     0.0      systemd           
2        root     0.0      kthreadd          
4        root     0.0      kworker/0:0H      
6        root     0.0      ksoftirqd/0       
7        root     0.0      rcu_sched         
34       root     0.0      khungtaskd        
35       root     0.0      oom_reaper        
36       root     0.0      writeback         
37       root     0.0      kcompactd0
```

另外要注意的就是针对域匹配 `$2~/york/`, 前面有说过 `$0` 代表整个域，所以 `$0~/york/` 和 `/york/` 是等价了，说这个的原因就是当我们需要 **针对一行做正则过滤的时候可以这样写** **`$0!~/york/`**, 这个在过滤日志的时候非常重要。下面增加几个例子

```shell
# 输出打印一行中出现 k 字符的行
awk 'NR==1 || /k/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt
awk 'NR==1 || $0~/k/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt

# 过滤掉一行中出现york字符的行
awk 'NR==1 || $0！~/york/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt

# 输出某个域字符以 k 开头的行
awk 'NR==1 || $12~/^k/ {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}' top.txt
```

### 根据条件定制输出流程

信息过滤针对的是 **pattern** 做文章，那么后面的 **action** 是不是也可以定制化控制，答案肯定可以，**action** 可以做流程定制，就是说我可以在里面使用 `if else while` 等流程控制语句。这和上面的条件判断不一样，因为他们针对的是不同部分，前面用于信息过滤，后面用于流程控制。

#### 过滤cpu大于0的，我还可以这样写

```js
awk '{if($9>0){printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12}}' top.txt
2304     york     6.2      cinnamon          
12489    york     6.2      top    
```

条条大路通罗马不是吗？这个例子就是流程控制的一个简单运用，利用到的是awk的流程控制，下面是流程控制的语法结构，了解下：

```js
if (expression) {
    statement;
    statement;
    ... ...
}

if (expression) {
    statement;
} else {
    statement2;
}

if (expression) {
    statement1;
} else if (expression1) {
    statement2;
} else {
    statement3;
}
```

上面的例子在扩展下，在结尾并统计下cpu的总和

```js
awk 'BEGIN {sum=0} {if($9>0){printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12; sum+=$9}} END {printf "cpu total usaged: %s\n", sum}' top.txt
2304     york     6.2      cinnamon          
12489    york     6.2      top               
cpu total usaged: 12.4
```

####  统计计算，输出各个USER的进程数

```js
shell> awk 'NR!=1{a[$2]++;} END {for (i in a) print i ", " a[i];}' top.txt
york, 2
root, 9
```

这个例子用一个数组统计不同用户的进程个数，并在最后用循环打印出来，这里有两个新的概念，一个是另外一种流程控制**循环**，另一个是数组的使用。关于循环的控制语法如下，和其它高级语言都类似。

```js
while(表达式)
  {语句}
for(变量 in 数组)
  {语句}
for(变量;条件;表达式)
  {语句}
do
{语句} while(条件)
```

除此之外，流程控制还有 `break`, `continue`, `exit` 等关键字，这些关键字含义和c语言一毛一样。

上面例子中 `a[$2]` 是典型的一种数组使用方法，用编程语言来看，这个叫数组似乎不大妥当，理解成 *map* 更合适，更像是 *key-value* 的存储结构。

#### 怎样使用数组

上面看到了数组的基本使用，其实 awk 给数组赋予了很多功能，和很多高级脚本语言一样，提供了相关的函数获取长度，排序等，另外存储是 *key-value* 结构，能像map一样判断key是否存在。

*获取长度 length* 

```js
shell> awk 'BEGIN{info="it is a test";lens=split(info,tA," ");print length(tA),lens;}'
4 4 
```

> 上面 split 函数用于字符串分割，和c语言的又是一毛一样

*循环输出*

```js
shell> awk 'BEGIN{info="it is a test";split(info,tA," ");for(k in tA){print k,tA[k];}}'
4 test
1 it
2 is
3 a 
```

> 由于是 *key-value* 的存储结构，所以使用这样的for循环输出结果是无序的，

*有序输出*

```js
shell> awk 'BEGIN{info="it is a test";tlen=split(info,tA," ");for(k=1;k<=tlen;k++){print k,tA[k];}}'
1 it
2 is
3 a
4 test
```

> 注意：数组下标从1开始的 注意：这种输出方法仅适用于把数组真正当作 ‘数组’ 使用，key值就是自然递增的数，而不是当map

*判断是否存在 key in array*

```js
shell> awk 'BEGIN{tB["a"]="a1";tB["b"]="b1";if( "c" in tB){print "ok";};for(k in tB){print k,tB[k];}}'
a a1
b b1
```

> 注意：判断语法是 `key in array` 不能直接写 `array[key] != value` 形式，如果这样写它会默认创建一个，使用过高级脚本语言的都知道；

*删除 deletekey*

```js
shell> awk 'BEGIN{tB["a"]="a1";tB["b"]="b1";delete tB["a"];for(k in tB){print k,tB[k];}}'                     
b b1
```

### 定制化输出到文件

这里引入怎么将处理后的文本保存下来，awk 提供了重定向的的功能； 

将上面例子中cpu大于0的保存到cpu.txt文件：

```js
shell> awk 'BEGIN {sum=0} {if($9>0){printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12 > "cpu.txt"; sum+=$9}} END {printf "cpu total usaged: %s\n", sum > "cpu.txt"}' top.txt

shell> cat cpu.txt
2304     york     6.2      cinnamon
12489    york     6.2      top
cpu total usaged: 12.4
```

上面引入的 `>` 就是重定向符，使用方法很简单，在你想要输出到文件的 `print` 命令后加上 `> "filename"` 即可。

拆分文件存储，按USER拆分：

```js
shell> awk 'NR>1 {printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12 > $2}' top.txt
shell> cat york
2304     york     6.2      cinnamon          
12489    york     6.2      top  
shell> cat root
1        root     0.0      systemd           
2        root     0.0      kthreadd          
4        root     0.0      kworker/0:0H      
6        root     0.0      ksoftirqd/0       
7        root     0.0      rcu_sched         
34       root     0.0      khungtaskd        
35       root     0.0      oom_reaper        
36       root     0.0      writeback         
37       root     0.0      kcompactd0 
```

上面按照 ‘USER’ 简单做了文件拆分，将输出内容拆分到 ‘york’和‘root’ 两个文件中，这个技巧在后面数据归类或者日志归类中使用非常频繁。

根据字符匹配来确定文件拆分 （结合if-else语句）

```js
shell> awk 'NR>1 {if($0~/york/){printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12 > "1.txt"}else if($0~/root/){printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12 > "2.txt"}else{printf "%-8s %-8s %-8s %-18s\n",$1,$2,$9,$12 > "3.txt"}}' top.txt 
```

上面将生成3个文件，其中USER=york的在‘1.txt’，USER=root在‘2.txt’，其它在‘3.txt’, 上面的例子就是前面所介绍的一种结合，正则匹配其实也可以用在 *action* 中做流程判断。

### awk与其他命令的结合

最后了解原理和过程后，发现一切都是水到渠成，其实不然，配合其它命令（比如管道，排序等）可以实现很多不可思议的工作，这里专门积累一些平时用到的。

```js
#按连接数查看客户端IP
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr
```