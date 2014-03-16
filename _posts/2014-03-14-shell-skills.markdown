---
layout: post
title: Shell Skills
categories: shell linux
date: 2014-03-14 21:33:44 +0800
---

最近的一个测试项目经常要写Shell脚本，硬着头皮上啊。

其实脚本还是很简单的，而且非常实用。但是因为它比较奇怪的语法，`if ... fi`, `case ... esac`之类的，让我一直很排斥它。
后来想想，还有Scala、Lisp之流，也就释然了。以前接触过脚本，写的不多，都谷歌百度过来。这次则比较系统看了一本书，
用起来也算是得心应手啊。

这里就记录一点关于写脚本时学到一点知识，多是平常没有注意的细节。

<!--more-->

## 查看帮助

这是最重要的，也是我最后才发现的。

`man` 命令大家肯定不陌生，但是 shell 中用到的一些内置命令在 `man` 中是找不到的。

{% highlight bash %}
evans@master:~$ man jobs
No manual entry for jobs

# 查看内置命令
evans@master:~$ bash -c 'help jobs'
jobs: jobs [-lnprs] [jobspec ...] or jobs -x command [args]
    Display status of jobs.
    
    Lists the active jobs.  JOBSPEC restricts output to that job.
    Without options, the status of all active jobs is displayed.
    
    Options:
      -l    lists process IDs in addition to the normal information
      -n    list only processes that have changed status since the last
        notification
      -p    lists process IDs only
      -r    restrict output to running jobs
      -s    restrict output to stopped jobs
    
    If -x is supplied, COMMAND is run after all job specifications that
    appear in ARGS have been replaced with the process ID of that job's
    process group leader.
    
    Exit Status:
    Returns success unless an invalid option is given or an error occurs.
    If -x is used, returns the exit status of COMMAND.

{% endhighlight %}

## 数组的使用

{% highlight bash %}
# 申明数组：通过一个字符窜（空格分开）来创建一个数组
devices=(`adb devices | grep device | grep -v List | awk '{print $1}' | tr '\n' ' '`)

# 数组的常用操作

# length
dev_len=${#devices[@]}

# iterate on arrays

# for in
for d in "${devices[@]}" ; do
    echo $d
done

# C中的循环
for (( i=0; i<${#devices[@]}; i++ ))
do
    echo ${devices[$i]}
done

# 添加元素
devices+=("xyz")

{% endhighlight %}


## 调试脚本

下面的这个选项可以帮助展开变量和脚本中执行的命令。

{% highlight bash %}
# 打开调试
set -x

# 关闭调试
set +x

# 结合以上两点，将调试应用于局部
function for_test {
    set -x
    # do something here
    set +x
}
{% endhighlight %}


# 获取一个文件的绝对路径

{% highlight bash %}
# 注意一下 $() 的用法
echo $(cd $(dirname $0); pwd)/$(basename $0)
{% endhighlight %}

# 函数的使用

关于函数，这是每门语言中的一个基本项，Shell脚本也不例外。之所以拿出说，是因为它我和想像中的（之前遇见的）函数不一样，
或者说，很不一样。

以这个函数为例子：

{% highlight bash %}
function clean_dir {
  local target_dir=$(cd $(dirname $1); pwd)/$(basename $1)    # 第一个参数是目标目录
  find $target_dir -type f -name '*.tmp' -print0 | xargs -0r rm -fr
  local res=$?

  if [ "$res" = "0" ] ; then
    echo "succ"
  else
    echo "failed"
  fi

  return $res
}
{% endhighlight %}

首先参数的处理，不需要在函数定义明确声明到底接受多少参数，且参数的获取与脚本在获取命令行参数时是一样的。
`$1` 是第一个参数，`$2`是第二个，然后依次。但是，**`$0` 不是函数名，而是脚本名！**

函数的返回值，是最后一条命令的退出状态（如果没有显示地使用 `return` 语句的话）。如果你要返回一个字符窜什么，
那该怎么办？`echo "a string"` 可以用用，但是，不够优雅。好像还没有很好的方法。

{% highlight bash %}
clean_dir /some/dir/path
echo $?  # 这是clean_dir的返回值，即退出状态。似曾相识吧？！

res=`clean_dir /some/dir/path`  # 这样可以捕获函数的输出，也就是传递字符窜的方式
{% endhighlight %}

总之，在我看来，shell脚本中函数就是一段**完整的脚本的组合**。你甚至可以直接把它放到独立的文件，
通过`./a_func param1 param2`的方式来调用。你看它处理参数的方式，被调用的方式就明白了。

# 并发执行
在需要协调多个任务时，非常有用的。

{% highlight bash %}

failed=0

./task.sh a &
./task.sh b &
./task.sh c &

for t in `jobs -p`; do
    wait $t || let "failed+=1"
done

if [ "$failed" = "0" ] ; then
    echo "Got $failed failures"
fi

{% endhighlight %}

`task.sh` 可以是一个单独的脚本，也可以是当前脚本里面的一个函数。如果是一个函数，则：

{% highlight bash %}
task a &
task b &
task c &
{% endhighlight %}

## find & mv 

这里，发现了两个之前没有用到的参数：

{% highlight bash %}
find ./ -maxdepth 1 -name "*.tar.gz" -type d -print0 | xargs -0r mv -t x/
{% endhighlight %}

在`find`命令中使用`-print0`选项，输出的内容将会是以空字符，即'\0'，结尾。
在`xargs`中使用`-0`选项，则它会将前面管道中的数据按空字符来分隔，前后刚好匹配。另外，这里还一个`-r`选项，
表示如果前面的命令没有输出，就不要执行`xargs`中的命令了。

那么上面的效果就是，如果找到目标文件，就将它移到到指定的文件夹中，找不到就不做任务操作。

之前没有使用这两个选项，将是导致找不到文件 `mv` 就失败，导致脚本无法正常执行。

# 计算文件的大小

{% highlight bash %}
$ ls -l /usr/bin/vim
$ du -h /usr/bin/vim
$ stat -x /usr/bin/vim
{% endhighlight %}
