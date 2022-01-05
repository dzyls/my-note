## 流程控制

### if-else

shell的if-else非常简单 ：

```bash
read -p 'enter your name :' name
read -p 'enter your age :' age
if [ $name == 'tom' ]
then
	echo your are tom
else
    echo your name is $name
fi
```

通过上面一段简单的程序，我们可以看到 ：

- read 可以读取用户在bash中输入的字符并放入变量中。当然不使用read直接 `name=$1`和`age=$2`也是可以的
- 在if判断的条件表达式中，变量必须要用`$`号来表明变量。
- if判断表达式中，使用`==`号判断；**变量前后要有空格**
- then、else、fi关键字
- 如果要使用else-if，那么else-if后也要用then



### switch

**shell的switch不使用switch，而是使用case**。如下 ：

```base
read -p 'enter command :' command
case $command in
	start)
		echo 'start the app'
	;;
	stop)
		echo 'stop the app'
	;;
	*)
		echo 'wrong command'
	;;
esac
```

上面一个简单的例子，可以看到 ：

- shell的switch使用的是`case $var in`
- 每个条件完毕后使用`;;`表示结束，类似`break`
- `*)`可以表示未匹配的，类似于`default`
- 最后结束使用`esac`





## 文件描述符

Linux中的文件描述符0、1、2是预留的 ：

- 0 ：标准输入
- 1 ：标准输出
- 2 ：标准错误

`sh append.sh > append.log 2>&1`的意思就是将标准输出放到append.log文件中，并将错误输出也放到标准输入中。

除此之外，也可使用`exec`创建自定义的命令描述符，如 ：

```bash
echo hello world > input.txt
exec 3 < input.txt
```

后面就可以用3这个命令描述符来获取input.txt的内容了 ：

```bash
cat<&3
```

> exec 执行完命令会将命令窗口关闭，如:
>
> exec echo hello world会在输出完成后，关闭窗口。



### 追加和截断

-  `>`是截断模式，会将文件中的内容清空再写入
- `>>`是追加模式，会继续在文件后面写入