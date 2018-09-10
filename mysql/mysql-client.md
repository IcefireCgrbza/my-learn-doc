mysql-client
====
* alter column
	+ add
		```
		ALTER TABLE `xxx` ADD COLUMN xxx {type}  NOT NULL DEFAULT 'xxx';	
		```
	+ change
		```
		ALTER TABLE `XXX` CHANGE COLUMN XXX XXX {type} NOT NULL DEFAULT '';
		```
	+ drop
		```
		ALTER TABLE `XXX` DROP COLUMN XXX {TYPE} NOT NULL DEFAULT '';
		```
* alter index
	+ add
		```
		ALTER TABLE `XXX` ADD {XXX} INDEX XXX(`AAA`, `BBB`);
		```
	+ drop
		```
		ALTER TABLE `XXX` DROP INDEX XXX; 
		```
	+ 索引类型：
		- UNIQUE INDEX：唯一索引，重复时报错
		- INDEX：普通索引
		- FOREIGN KEY：外键
			* add
			```
			A
			```
	+ show
		```
		SHOW INDEX FORM XXX;
		```
	+ key与index区别：key具有约束和索引两重语义，而index只有索引语义