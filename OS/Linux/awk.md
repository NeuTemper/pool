# awk

## 1. Background
awk有三个不同的版本包括, awk、nawk和gawk，一般指gawk。gawk是AWK的GNU版本。
> Gawk  is  the GNU Project's implementation of the AWK programming language.  It conforms to the  definition of the language in the POSIX 1003.1 Standard.  This version in turn is based on  the description  in The AWK Programming Language, by Aho, Kernighan, and Weinberger.  Gawk provides the additional features found in the current version of UNIX awk and a number  of  GNU-specific extensions.

## 2. Options
`gawk`的options允许两种不同方式
- 传统`POSIX-style`的一个`-`短选项
- `GNU-style`的`--`长选项

每一个长选项都有对应的`POSIX`风格的短选项，在所有情况下都是可以进行互换的。关键字可以缩写，只要缩写允许选项被唯一标识即可。如果选项接受参数的话，关键字后面紧跟一个等号`=`和参数的值，或者关键字和参数的值由空格分割。当重复给出时，使用最后计算的值。

#### 常用选项

[详细options选项](https://www.gnu.org/software/gawk/manual/html_node/Options.html#Options)

##### -f program-file / --file program-file

直接从源文件而不是从第一个参数来读取awk程序源。此选项可以多次给出，awk程序包含每个指定源文件内容的串联。

##### -F fs / --field-separator fs
将输入值使用分隔符进行分割

##### -v var=val / --assign var=val
在程序执行开始前将变量var设置为值val。`-v`只能设置一个变量，但可以多次使用。

##### -e program-text / --source program-text
此选项允许在程序文本中提供程序源代码，可以在此使用命令行程序使用的库函数。`gawk`将每个字符串视为以换行符结束。其主要用于shell脚本中使用的中型到大型AWK程序。

##### -E file / --exec file
类似于`-f`，也是从一个文件作为源来输出到`awk`程序，其与`-f`的差别在于
- 此选项终止选项处理，命令行的任何内容都直接传递给awk程序
- 命令行变量赋值被禁止(-v)

## Start awk Programs
`awk`与其他编程语言的区别在于`awk`程序是`data driven`的(描述需要的数据以及做什么)，`awk`以行为处理单位，直到其到达输入文件的末尾。通常`awk`的`program`部分包含了一系列的规则，甚至函数。一个规则通常是由`pattern`与`action`两部分构成，用大括号与规则区分。`awk`中推荐使用单引号来进行闭合，`pattern`表示`awk`在数据中查找的内容，`action`是找到之后的一系列命令。
```bash
$ awk 'pattern { action }' input-file
```

#### 使用方式
有多种方式使用awk。

- 命令行方式

    在程序代码很短时，可以直接在指令中包含awk。下面的例子中，`program`是真正的awk命令， `-F`域分隔符是可选的，input-file是待处理的文件。在awk中文件每一行中，由域分隔符分开的每一项称为一个域。在不指明`-F`的情况下，默认的域分隔符是空格。

    ```bash
    $ awk [-F field-separator] 'program' input-file1 input-file2
    ```

- 单独awk文件

    将所有的awk命令插入到一个单独文件，然后调用。下面例子中，`-f`加载`program-file`的awk脚本。
    ```bash
    awk -f program-file input-file1 input-file2
    ``

- awk脚本

    可以通过`"#!"`的脚本来创造awk脚本。增加可执行权限，执行下面的`example`文件相当于执行`awk -f advice`
    ```bash
    #! /bin/awk -f

    BEGIN { print "Hello, Wolrd!" }
    ```

#### e.g

数据文件如下：
```bash
Amelia       555-5553     amelia.zodiacusque@gmail.com    F
Anthony      555-3412     anthony.asserturo@hotmail.com   A
Becky        555-7685     becky.algebrarum@gmail.com      A
Bill         555-1675     bill.drowning@hotmail.com       A
Broderick    555-0542     broderick.aliquotiens@yahoo.com R
Camilla      555-2912     camilla.infusarum@skynet.be     R
Fabius       555-1234     fabius.undevicesimus@ucb.edu    F
Julie        555-6699     julie.perscrutabor@skeeve.com   F
Martin       555-6480     martin.codicibus@hotmail.com    A
Samuel       555-3430     samuel.lanceolis@shu.edu        A
Jean-Paul    555-2127     jeanpaul.campanorum@nyu.edu     R
```

- 查找包含`li`的行
    ```bash
    $ awk '/li/ { print $0 }' data
    ```
- 查找大于80个字符的行
    ```bash
    $ awk 'length($0) > 80' data
    ```
- 输出最长行的长度
    下方示例包括`END`，表示其在所有输入读取之后执行。相反的`BEGIN`表示在所有输入执行之前执行。
    ```bash
    $ awk '{ if (length$(0) > max) max = length($0) } END { print max }' data
    ```
- 输出起码拥有一个域的行
    ```bash
    $ awk 'NF > 0' data
    ```
- 输出7个0-100的随机数
    ``` bash
    $ awk 'BEGIN { for (i=1; i<=7; i++) print int(101 * rand()) }'
    ```
- 输出所有用户名
    ```bash
    awk -F: '{ print $1 }' /etc/passwd | uniq | sort
    ```

## Regular expressions
正则表达式可以通过用斜杠括起来用作`parttern`，针对每条记录的整个文本测试正则表达式。

#### '~' 和 '!~'

正则表达过滤并不一定要求是整个行的输入。可以使用`'~'`或者`'!~'`来执行正则表达，或者使用`if`,`while`,`for`和`do`语句。

例如，输出当前目录下所有为目录的文件

```bash
$ ll -al | awk '$1 ~ /d/'
```

与之相反的，输出非目录文件
```bash
$ ll -al | awk '$1 !~ /d/'
```

#### 转义字符
某些字符不能直接包含在字符串常量或者regExp常量中。其应该用转义字符表示。其很典型的一个用途是在字符串常量中包含双引号字符。[转义字符列表](https://www.gnu.org/software/gawk/manual/html_node/Escape-Sequences.html#Escape-Sequences)列出了`awk`种使用的所有转义字符及其代表的内容。

#### 正则表达式操作符

可以通过组合特殊字符来进行正则表达，其中列出部分`awk`自带的[正则表达操作符](https://www.gnu.org/software/gawk/manual/html_node/Regexp-Operators.html#Regexp-Operators)。`gawk`在此基础上增加了特殊的[正则表达式操作符](https://www.gnu.org/software/gawk/manual/html_node/GNU-Regexp-Operators.html#GNU-Regexp-Operators)


- `^`: 匹配字符串开始， `^hello` 匹配以 `hello` 开头的字符串
- `$`: 匹配字符串结尾
- `.`: 匹配单个字符
- `|`: 交替运算符，指定备选方案，如`^P|[aeiouy]`匹配字符串以`P`开头或者包含`aeiouy`的字符串
- `{n, m}`: 匹配重复几个字符
