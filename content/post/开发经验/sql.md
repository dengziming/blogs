---
date: 2018-03-22
title: "常用sql写法"
author: "邓子明"
tags:
    - Sql
categories:
    - 开发经验
comment: true
---

## sql语法
每个语言的语法不一样，大致类似。

### 1. 调优经验
可以先explain，分析一下。然后再运行。

join可能会导致数据倾斜的情况：

join的key是不均匀的，
join的key有很多事空值，这时候可以分为两部分进行union


多表join：
一般是前面的stage一起执行，最后有一个stage。
如果发现很慢，数据也没有倾斜，可能是某些表的key重复，导致join会有笛卡尔积操作，得到很多数据所以慢，可以根据业务进行过滤。


## 常用sql

### 1.不等值join

```sql

cache table b as select * from location ;

select
    a.ip,
    b.city
from
    b
RIGHT JOIN
    fin_kg.kg_v2_v_ip a
ON
    a.ip >= b.ip_start
AND
    a.ip <= b.ip_end

```

和

```sql
CACHE TABLE chuanxiao AS SELECT id,name,regs FROM vdm_fin.chuanxiao_20180417;

insert overwrite table kg_v2_e_userid_gang
select
    b.id,
    b.name,
    a.phone,
    a.contact_phone,
    a.contact_name
from
     a
right join
    chuanxiao b
on
    a.contact_name regexp regexp_replace(b.regs,"%",".*");
```

修改null存储为'',
```sql
ROW FORMAT DELIMITED
      FIELDS TERMINATED BY ','
      LINES TERMINATED BY '\n'
      NULL DEFINED AS ''
     stored as textfile;
     
alter table fin_kg.kg_v2_e_phone_gang_bak SET SERDEPROPERTIES ('serialization.null.format'=''); 
```
### 2.分位点

percentile
percentile_approx
percentile_rank

### 3.保留小数
printf

### 4. 手动刷新impala元数据
invalidate metadata [tablename]

### 5.键值对
str_to_map(concat_ws(',',collect_set(concat_ws(':', ctime_month, cast(final_score as string) )))) as time2score

### 6.cache table 
