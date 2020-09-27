sed
===
功能：对每一行进行匹配后输出
---
语法
---
```
sed [-option] 'command' inputFile
```
option
---
* -n，匹配才输出，测了后有些有用有些没用
* -e，多个处理语句，可以用;隔开
* -i，输出回文件
* -f，从文件读入command
* -r，支持更有力的正则表达式

数字定址和正则定址
---
* 写在command的最前面
* n，对第n行处理
* n,m，对第n行到第m行处理
* n,+m，从第n行开始加m行
* n，~m，从第n行开始每个m的倍数行
* n~m，从n开始每隔m行
* n!，除了第n行
* 正则定址，将正则写在最前面

命令
---
* a xxx，下面插入xxx
* i xxx，上面插入xxx
* c xxx，行替换成xxx
* d，删除行
* =，每行上面打行号
* r xxx，xxx文件插入到下面
* s/pattern/replacement/flags，根据匹配替换