awk
===
格式化输出
---
```
awk [-F separator] ‘commands’ inputFiles
```
* 相当于在for循环里处理每一行
* -F定义分隔符，默认是空格
* -f从文件中读入commands
* commands是一段{}包裹的代码
* awk自带变量，$0表示一行内容，$n表示第n个分割字段内容
* awk自带内置环境变量，需要的话上网查吧

举例
---
```
ps -ef | grep '{print $2 $8}'
```