---
layout: post
author: "杨小定定"
title:  "BAT 脚本遍历文件"
description: "主要类似于读取配置文件，在批量处理文件时很有用。"
date:   2018-06-25
categories: technology
---

> 经常会有这样的需求，有一个文件列表（文本文件），然后需要通过脚本读取来处理的，可参考以下知识。 

### 读取并作为变量值使用

假设现在有一个文本文件`name.txt`，内容是名称和年龄列表，如下：

```txt
Tom,18
Rick,20
Jack,22
...
```

> Ps. 在Windows下，该文件编码应为ANSI，若为UTF-8，则中文会有问题。

如下代码就是读取该文件，并在每个名称前打印`Hello`，并加上`Age is:`显示：

```bat
@echo OFF

for /f "tokens=1,2 delims=," %%a in (name.txt) do (
	echo Hello, %%a !
	echo Age is: %%b
)
```

运行，输出如下：

```
Hello, Tom !
Age is: 18
Hello, Rick !
Age is: 20
Hello, Jack !
Age is: 22
```

以上代码解释一下：

- `for /f`：for循环的格式，括号内可以写多条命令，`/f`表示文件读取，文件名为`(name.txt)`可以指定一个绝对路径
- `tokens=1,2`：表示取该文件的第1和第2个字段
- `delims=,`：表示字段之间用`,`区分
- `%%a`：设置的变量名，类似于Java中`for (i=0;i++;i<10)`中的`i`
- 此处只需指定一个`%%a`即可，不管前面取多少个字段，后续要使用的话，变量名是ASCCII码往后+1，即可，如本例中的`%%b`（如果指定`%%i`，则为`%%j`）

### 读取文件中路径的处理

还有的时候，我们需要批量处理文件，假设这些文件名记录在一个`file.txt`的文件中。

比如我们需要处理`D:\sample`下以及其子目录下的所有文件（排除目录），则可以通过如下命令生成对应的`file.txt`：

```bat
cd /d D:\sample
dir /s /b /a:-d * > file.txt
```

上述`dir`命令选项分别为：

- `/s`：表示显示指定目录和所有子目录，即递归显示
- `/b`：使用空格式，即没有标题信息或摘要
- `/a`：显示具有指定属性的文件，格式为`/a[[:]attributes]，属性可选如下
  - `D`：目录
  - `R`：只读文件
  - `H`：隐藏文件
  - `A`：准备存档的文件
  - `S`：系统文件
  - `I`：无内容索引文件
  - `L`：解析点
  - `-`：表示“否”的前缀，即非
- 后面的`*`，当需要获取指定后缀名结尾的文件时，也可以用`*.zip`这种

现在，`file.txt`中文件内容类似如下：

```txt
D:\sample\name.txt
D:\sample\test.txt
D:\sample\temp\a.txt
D:\sample\temp\test\temp.txt
```

然后我们来循环读取该文件，并显示分别的路径名称和文件名称：

```bat
for /f "tokens=1 delims= " %%a in (file.txt) do (
    echo %%a
    echo dirname is %%~dpa
    echo basename is %%~nxa
)
```

运行后，输出如下：

```
D:\sample\name.txt
dirname is D:\sample\
basename is name.txt
D:\sample\test.txt
dirname is D:\sample\
basename is test.txt
D:\sample\temp\a.txt
dirname is D:\sample\temp\
basename is a.txt
D:\sample\temp\test\temp.txt
dirname is D:\sample\temp\test\
basename is temp.txt
```

上述示例中，使用了带有修改程序的变量，具体说明如下：

- `%~A`：展开删除任何前后引号 ("") 的`%A`
- `%~fA`：将`%A`展开到完全合格的路径名
- `%~dA`：只将`%A`展开到驱动器号
- `%~pA`：只将`%A`展开到路径
- `%~nA`：只将`%A`展开到文件名
- `%~xA`：只将`%A`展开到文件扩展名
- `%~sA`：展开路径以只包含短名称
- `%~aA`：将`%A`展开到文件的文件属性
- `%~tA`：将`%A`展开到文件的日期和时间
- `%~zA`：将`%A`展开到文件大小