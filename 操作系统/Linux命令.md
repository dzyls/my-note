### find

---

find命令是查找的命令。

`find . -iname "*.txt"`

`-iname`是忽略大小写的意思。

**OR**

如果想合并两个查询，可以使用`-o`，如

`find . -iname '*.txt' -o -iname '*.sh'` 就会打印出两个查询



**Path路径匹配**

`find /home -path '*/path-name/*' -print`

这样可以根据路径去匹配



**regex正则匹配**

`find . -iregex '.*\(\.py)$'`

可以使用正则来匹配



**否定**

查询还可以使用不符合条件的，如`find . ! -name '*.txt'`



**限制搜索的目录深度**

搜索时指定目录深度，可以减少查询范围：

`find . -maxdepth 2 -name 'f*'`



**根据文件类型搜索**

可以根据类型来搜索，常见的类型有 ：

- 目录 d
- 文件 f
- 链接 l

`find . -type f`



**根据时间搜索**

还可根据时间来搜索：

-  `-atime` ：上次访问时间
- `-mtime` ：上次修改时间
- `-ctime` ：权限上次改动时间



以上的时间单位都是天，可以用更小的如 ：

- `-amin`
- `-mmin`
- `-cmin`

如 ，`find . -type f -amin 60`，过去一小时访问的文件。





**基于文件大小搜索**

`-size`可以指定文件大小来搜索，如 ：

`find . -type f -size +2k` 搜索大于2k的文件



**find执行命令**

find命令可与`exec`命令结合。

如，将搜索到的多个文件合并 ：

`find . -type f -name '*.txt' -exec cat {} \; > new_append.txt`

其实用`xargs`也可实现 ：`find . -type f -name '*.txt' | xargs cat > new.txt`



`find .type f -name '*.txt' -exec chmod +x {} \;`

命令中的 `{}`代表查找到的文件对象列表。



### tr

---

tr命令是用于转换的命令。常见的如 ：

**大小写转换  ：**

`echo 'hello,world' | tr 'a-z' 'A-Z'`



**删除指定的字符**

`echo 'hello,world' | tr -d 'o'`

结果 ：`hell,wrld` 



**删除不在集合中的字符**

`echo 'hello,world' | tr -d -c 'lo'`

结果 ：`llool`



**去除重复的字符**

`echo 'hello     world' | tr -s ' '`

输出 ：`hello world`



**文件中的数字相加**

`cat sum.txt | echo $[$(tr '\n' '+') 0]`

将换行符换为`+`，最后再补个`0`。



### 哈希

---

计算文件的md5 ：

`md5sum filename`

可以有多个文件。

或者`sha1sum` 



### 加密

---

**crypt**





**gpg**

`gpg -c file_name` 会让输入密码，并生成file_name.gpg

`gpg file_name.gpg`这是解密的



**base64**

加密 ：`base64 file_name > base64.txt`

解密 ：`base64 -d base64.txt > new_file_name`



### 排序

---

sort

**对文件排序**

`ls | xargs sort`



**对数字排序**

`sort -n num.txt`



**逆序**

`sort -r file.txt`

如果是数字的话 ：`sort -nr num.txt`



**去重**

`sort -nr num1.txt num2.txt|uniq`

`uniq`只能用于管道。



**指定列排序**

`-k`可以指定对哪一列进行排序。

如 `sort -k 2`



### 临时文件名

---

```bash
filename=`mktemp`
echo $filename
#临时文件名
```

```bash
dirname=`mktemp -d`
echo $dirname
#临时文件夹
```

`-u`可以不创建文件和目录，如 ：

```bash
tmpfile=`mktemp -u`
echo $tmpfile
```



### 分割文件

---

早期的磁盘都很小，将文件分割分别存储在不同的磁盘中。

```bash
spilt -b 10k data.file
#分割为10k大小的小文件
```

如果想按行分的话 ：

```bash
spilt -l 10 data.file
#每个小文件就10行
```



### 获取文件的名称

---

```bash
#!/bin/bash
full_name=$1
name=${full_name%.*}
echo $name
```

`%.*`代表删除%右侧的通配符，**从右向左**进行匹配。

`%`非贪婪的，`%%`是贪婪的。比如，同样的入参 : `hack.test.txt.book`

`${VAR%.*}` 的结果为 ：`hack.test.txt`

而`${VAR%%.*}`的结果为 `hack`



还可以获取文件的后缀 ：

```bash
#!/bin/bash
full_name=$1
name=${full_name#*.}
```

#号则代表从左到右进行匹配。



### 批量重命名

---

```bash
#!/bin/bash
count=0
for count in $(seq 10)
do
	touch f-$count.txt
	let count++
done
```



```bash
#!/bin/bash
count=0
for file in `find -iname '*.txt' -o -iname '*.text' -type f -maxdepth 1`
do
	new=file-$count.${file##*.}
	mv "$file" "$new"
	let count++
done
```

注意 ：

- 使用`bash`执行
- 引用变量使用双引号

