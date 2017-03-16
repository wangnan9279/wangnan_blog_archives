---
title: Mybatis 使用<where>标签时遇到的一个问题与<trim>标签的使用
tags: [Mybatis, ORM框架]
date: 2017-03-16 16:11:20
categories: Mybatis
link_title: mybatis-where-trim
---
> 今天遇到一个场景需要写一个这样的查询语句：
用户对象userInfo包含下面几个字段： 
userName phone email qqId weiboId wxId
<!-- more -->

现在新注册用户，传过来一个注册userInfo对象，现在要到数据库中验证状态status=1 （表示激活的用户）的用户中，是否存在一个用户，只要它这些字段中至少有一个与新注册的对象对应的字段内容相同，那就说明重复注册。

翻译成sql语句表示一下的意思大概就是： 
select * from tablename where 
( 
userName=”xxx” 
or phone =”xxx” 
or … 
) 
and status=1

一开始我是这样写的，在mybatis中的代码就是这样：
```xml
<select id="selectBySelective" resultType="xxx.UserInfo">
   select 
    <include refid="Base_Column_List" />
    from uc_user 
    <where>
    (<if test="userName != null" >
        user_name = #{userName}
      </if>
      <if test="email != null" >
        or email = #{email}
      </if>
      <if test="phone != null" >
        or phone = #{phone}
      </if>
      <if test="weiboId != null" >
        or weibo_id = #{weiboId}
      </if>
      <if test="wxId != null" >
        or wx_id = #{wxId}
      </if>
      <if test="qqId != null" >
        or qq_id = #{qqId}
      </if>)
     </where>
     and status = 1
</select>
```
这样代码看似没有什么问题但是其实是有问题的。为什么呢？ 
如果userName 为空，后面某字段不为空，最后的sql语言会成为这样：
```sql
select * from uc_user where(or email = "xxx") and status = 1
```
使用mybatis < where > 标签就是为了防止这种情况，mybatis会在第一个 
userName 为空的情况下，帮我们去掉后面的语句的第一个”or”

但是我加了where标签中加入（）后，语句会报错。因为自动去掉”or”会失效。

查看了mybatis官方文档发现了另一个标签 < trim >可以通过自定义 trim 元素来定制我们想要的功能

trim标签包围的内容可以设置几个属性： 
prefix ：内容之前加的前缀 
suffix ：内容之后加的后缀 
prefixOverrides： 属性会忽略通过管道分隔的文本序列（注意此例中的空格也是必要的，多个忽略序列用“|”隔开）。它带来的结果就是所有在 prefixOverrides 属性中指定的内容将被移除。

所以我修改后的代码是：
```xml
<select id="selectBySelective" resultType="xxx.UserInfo">
   select 
    <include refid="Base_Column_List" />
    from uc_user 
    <trim prefix="WHERE ("  suffix=")" prefixOverrides="AND |OR "> 
    <if test="userName != null" >
        user_name = #{userName}
      </if>
      <if test="email != null" >
        or email = #{email}
      </if>
      <if test="phone != null" >
        or phone = #{phone}
      </if>
      <if test="weiboId != null" >
        or weibo_id = #{weiboId}
      </if>
      <if test="wxId != null" >
        or wx_id = #{wxId}
      </if>
      <if test="qqId != null" >
        or qq_id = #{qqId}
      </if>  
     </trim>
     and status = 1
</select>
```
