---
layout:     post
title:      MySql character set和collation
subtitle:   character set和collation的区别
date:       2019-07-28
author:     Yuanye.Wang
header-img: img/post-bg01-1115.jpg
catalog: false
tags:
    - MySQL
---
#### 1.character set即字符集，如Unicode、UTF-8、GB2312；
> 打个比方，你眼前有一个苹果，在英文里称之为apple，而在中文里称之为苹果。
苹果这个实体的概念就是Unicode，而UTF-8，GB2312可以认为就是不同语言对苹果的不同称谓，本质上都是在描述苹果这个物。
#### 2.collation即比对方法
##### 1. 用于指定数据集如何排序，以及字符串的比对规则。
##### 2. 同一个character set的不同collation的区别在于排序、字符串对比的准确度（相同两个字符在不同国家的语言中的排序规则可能是不同的）以及性能。 

##### 3. collation名字的规则可以归纳为这两类
```sql
    * <character set>_<language/other>_<ci/cs>
    * <character set>_bin
# ci 是case insensitive的缩写，cs 是case sensitive的缩写。即，指定大小写是否敏感
# utf8_bin是将字符串中的每一个字符用二进制数据存储，区分大小写。
```
##### 4. 当collation忽略大小写时，如何区分大小写
```sql
    select * from student where name = binary 'xixi’;（推荐）
    select * from student where binary name = 'xixi’;(索引失效)
```

#### 3.字符集查看与修改
```sql
# 1. 数据库
show create database hspms;  #  查看数据库字符集
ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE …]; #  修改字符集

# 2. 表
show create table tbl_name; #  查看表字符集
alter table tbl_name convert to character set utf8 collate utf8_general_ci; #  修改字符集

# 3. 字段
show full columns  from tbl_name;  #  查看表字段字符集
alter table tbl_name change col_name col_name varchar(100) character set utf8 collate utf8_general_ci; #  修改字符集
```


#### 4.注意
```sql
# 1. 当表的character set是latin1时，若字段类型为nvarchar，则字段的字符集自动变为utf8。可见database character set，table character set，field character set可逐级覆盖。
# varchar(n)：长度为n个字节的可变长度且非Unicode的字符数据。
# nvarchar(n)：包含n个字符的可变长度Unicode字符数据。
```
