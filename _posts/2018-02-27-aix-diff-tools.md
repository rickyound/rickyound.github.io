---
layout: post
title:  "AIX 文件及目录对比工具"
description: "因diff和sdiff均不能直接符合要求，故编写该工具。"
date:   2018-02-27 14:08:00 +0800
categories: technology
---

我们有时候需要做版本比对工作，会将两个待比对的版本放到两个目录，在Windows下，有很多对比工具可以选择，但是在AIX下，我们只有`diff`命令和`sdiff`两个命令可选。

但是，`diff`命令只会对比文本文件，对于目录，只会下钻一层；`sdiff`命令则只是相当于`diff`的封装，展现的更好看一点而已。这不符合我们的要求，我们需要比对指定两个目录下所有的文件，包括子目录中的文件，所以需要自己编写脚本来处理。

## 设计

这个需求很简单，我们相当于再次封装一下`sdiff`命令即可，大致流程如下：

```
开始 --> 判断传入文件类型是否一致 --是--> 判断是文件还是目录 --目录--> 递归查询子目录下所有文件
                  |                           |                         |
                  否                         文件                        |
                  +--> 报错退出                +--> 直接使用sdiff对比   <--+
                                                          |
                                                          +--> 展示结果
```

## 实现

根据上述设计，我们一步一步来实现。

### 1、参数

参考`sdiff`的参数，我们也定义几个：

```
-s,        不显示匹配上的行
-l,        仅列出结果的文件名
-w width,  指定结果显示的总宽度
-h,        显示帮助页面
```

另外需要定义几个变量来存储各个配置以及传入参数，所以我们编写如下脚本：

```ksh
#!/usr/bin/ksh

# 文件及目录比对工具
# author by 杨小定定

SHELL_PATH=`cd $(dirname $0);pwd`
SHELL_NAME=`basename $0`

# 定义两个变量存储需要比对的文件或目录，默认写为None
source_var="None"
target_var="None"

# 默认结果显示宽度130，左右各65
show_width=130

# 显示匹配项标志，默认显示
show_identical="Y"

# 只列出文件名标志，默认否
only_list_name="N"

# 处理传入参数
while test $# -gt 0
do
    case $1 in 
        "-s")
            show_identical="N"
            ;;
        "-l")
            only_list_name="Y"
            ;;
        "-w")
            shift
            show_width=$1
            ;;
        "-h")
            echo "some help text."
            exit 0
            ;;
        *)
            if test "$source_var" = "None"
            then
                source_var=$1
            elif test "$target_var" = "None"
            then
                target_var=$1
            else
                echo "参数个数错误"
                exit 1
            fi
			;;
done
```

### 2、校验

处理好参数后，我们需要对这些参数做一些校验，比如参数类型是否正确；传入文件是否存在等等：

```ksh
# 定义一个半宽度变量，同时判断计算返回值，做简单的数值正确性校验
show_half_width=`expr show_width / 2`

if test $? -ne 0
then
    echo "宽度参数错误"
    exit 1
fi

# 对传入的待比对的两个参数进行校验
if test "$source_var" = "None" -o "$target_var" = "None"
then
    echo "参数个数错误"
    exit 1
fi
```

为了校验传入的参数是文件，还是目录，还是不存在，我们定义一个`iswhat`简单的函数来判断：

```ksh
function iswhat
{
    local var=$1
    if test -f $var
    then
        echo "F"  #代表普通文件
    elif test -d $var
    then
        echo "D"  #代表目录
    else
        echo "N"  #代表其他文件或不存在的
    fi
}
```

所以利用上述简单的函数，我们可以来判断传入的待比对文件类型是否一致：

```ksh
if test `iswhat $source_var` != `iswhat $target_var`
then
    echo "待比对文件类型不一致"
    exit 1
fi
```

至此，我们校验也做完了，就可以开始具体的比对了。

我们先定义`diff_file`和`diff_path`两个函数，分别用来做文件比对和做目录比对。

### 3、文件比对

以下是文件的对比`diff_file`函数：

```ksh
function diff_file
{
    local left_file=$1
    local right_file=$2

    # 定义一半的显示宽度，-3是因为我们要在中间添加一些符号
    local count=`expr ${show_half_width} - 3`

    # 先用diff测试一下，如果相符则不显示，直接返回
    if test `diff $left_file $right_file > /dev/null && echo $? || echo $?` -eq 0
    then
        return 0
    fi

    # 显示文件名
    printf "%-${count}s <-> %-${count}s\n" $left_file $right_file

    # 如果定义只显示文件名，则现在返回
    if test "$only_list_name" = "Y"
    then
        return 0
    fi

    # 进行到此处就显示sdiff对比结果，考虑是否添加-s参数
    if test "$show_identical" = "Y"
    then
        sdiff -w ${show_width} $left_file $right_file
    else
        sdiff -s -w ${show_width} $left_file $right_file
    fi
}
```

### 4、目录比对

下面是目录的比对，`diff_path`函数，其中需要递归调用：

```ksh
function diff_path
{
    # 记录头部文件
    local left_head=$1
    local right_head=$2

    # 使用ls命令分别列出其下的文件
    local left_files=`ls $left_head`
    local right_files=`ls $right_head`

    # 定义一半的显示宽度，-3是因为我们要在中间添加一些符号
    local count=`expr ${show_half_width} - 3`
    
    # 循环左边的文件列表
    for LINE in $left_files
    do
        # 先判断该子项在右边的列表中是否存在，如否，则直接输出，继续下一个
        if test `echo $right_files | grep -q $LINE && echo $? || echo $?` -ne 0
        then
            printf "%-${count}s <-!\n" $left_head/$LINE
        else
            # 定义一个临时变量，记录左右两边的类型
            local temp_type=`iswhat $left_head/$LINE``iswhat $right_head/$LINE`

            case $temp_type in
                "FF")      # 均为文件，调用文件比对函数
                     diff_file $left_head/$LINE $right_head/$LINE
                     ;;
                "FD"|"DF") # 左右两边不一致，提示
                     echo "$left_head/$LINE 与 $right_head/$LINE 类型不同"
                     ;;
                "DD")      # 均为目录，则递归调用本身，继续下一层
                     diff_path $left_head/$LINE $right_head/$LINE
                     ;;
                "NF"|"ND") # 左边文件不存在或不为普通文件，输出提示
                     printf "%-${count}s !-> %-${count}s\n" " " $right_head/$LINE
                     ;;
                "FN"|"DN") # 右边文件不存在或不为普通文件，输出提示
                     printf "%-${count}s <-!\n" $left_head/$LINE
                     ;;
                *)         # 否则输出提示，均不存在
                     echo "左右均不存在或不为普通文件或目录"
                     ;;
            esac
        fi
    done

    # 再循环右边的列表，只需将右边中独有的找出输出即可
    for LINE in $right_files
    do
        if test `echo $left_files | grep -q $LINE && echo $? || echo $?` -ne 0
        then
            printf "%-${count}s !-> %-${count}s\n" " " $right_head/$LINE
        fi
    done
}
```

### 5、整合调用

准备好上面两个函数后，我们就可以进行分情况调用了。

因之前我们已经对输入的两个参数类型是否一致做过校验了，故我们只需判断一个的类型，就能确定使用哪个函数了：

```ksh
case `iswhat $source_var` in 
    "F")
        diff_file $source_var $target_var
        ;;
    "D")
        diff_path $source_var $target_var
        ;;
    *)
        echo "输入参数类型不对"
        ;;
esac
```
