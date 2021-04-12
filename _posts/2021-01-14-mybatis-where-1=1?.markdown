---
layout: post
title: "mybatis where 中常出现的垃圾语句 1=1"
subtitle: "经常看到mapper的查询语句中，出现where 1=1,看起来没什么意思的话是用来做什么的呢？可以优化掉吗？"
date: 2021-01-14
author: liying
category: java
tags: [mybatis,java]
finished: true
---

* 目录
{:toc #markdown-toc}


## 1=1？

通常我们会在select语句中使用`1=1`的永真语句来避免我们使用`where`，`<if></if>`时出现`where`后没有查询条件问题。

如果我们直接这么写：

```xml
<select id="query" resultMap="BaseResultMap">
        SELECT * from t_table
        where
        <if test="type != null and '' != type ">
            type = #{type,jdbcType=VARCHAR}
            and
        </if>
        <if test="name != null and '' != name  ">
            name= #{name,jdbcType=VARCHAR}
            and
        </if>
 </select>
```

当if条件都不成立时，拼接出来的语句为:

```sql
SELECT * from t_table
        where
```

1=1就是为了防止前面两个条件都为否时，出现空where语句，从而导致的SQL语法问题。

```xml
<select id="query" resultMap="BaseResultMap">
        SELECT * from t_table
        where
        <if test="type != null and '' != type ">
            type = #{type,jdbcType=VARCHAR}
            and
        </if>
        <if test="name != null and '' != name  ">
            name= #{name,jdbcType=VARCHAR}
            and
        </if>
        1=1
 </select>

```

这是就算条件不成立，语句也不会出现语法问题：

```sql
SELECT * from t_table
        where 
        1=1
```



以为问题解决了？

不。

使用“1=1”会出现以下问题：

1. 多余语句：“1=1”看起来毫无意义（垃圾条件）
2. 性能损耗：加了“1=1”的过滤条件以后数据库就**无法使用索引**等查询优化策略，数据库系统将会被迫对每行数据进行扫描（也就是全表扫描）以比较此行是否满足过滤条件。



## 消除1=1

使用mybatis可以很方便地消除这个隐患。

#### 方法一:

使用`<where>`标签，mybatis会自动处理条件为空时的情况

```xml
SELECT * from t_table
<where>
            <if test="status != null and status !=''">
                AND t_table.status = #{status}
            </if>
            <if test="applicant != null and applicant !=''">
                AND t_table.applicant = #{applicant}
            </if>
</where>
```



#### 方法二：

动态追加标签。（网络搜索得到，未尝试）

```xml
 <statement id="getPersonsByAge" parameterClass=”java.Util.HashMap” resultClass="examples.domain.Person">

     <![CDATA[

          SELECT  *  FROM PERSON

              <dynamic prepend="WHERE">

                 <isNotNull prepend="AND" property="ageValue">

                    AGE > #ageValue#

                 </isNotNull>

             </dynamic>

     ]]>

 </statement>
```

`<dynamic>`元素划分出SQL语句的动态部分，非动态的语句不要写在dynamic标签里，默认把第一个条件成立的predend字段去掉。



参考：

1. [SQL中使用where 1=1 和 select * 的坏处](https://blog.csdn.net/Yao_yaowei/article/details/74953342)

2. [【mybatis】mybatis中避免where空条件后面添加1=1垃圾条件的 优化方法 标签](https://www.cnblogs.com/sxdcgaq8080/p/9412167.html)