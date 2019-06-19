# 记录一次resuleMap未给id的踩坑血泪史
接着上一篇，我已经成功按照原来的思路映射出我要的嵌套对象集合了。由于有一个列表查询条件是需要同时判断两个字段的多种情况，于是乎我就直接用case when解决，
给FormBean加了一个字段，对应的resultMap中也加入了对应的<result/>。自此跳入了血坑当中。下边我继续用上篇的例子重现这个问题！
## 
UserFormBean:
```
@Data
public class UserFormBean {
    private Department department;
    private User user;
    private String sex;
}
```
UserMapper.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.mapper.UserMapper">
    <resultMap id="FormResultMap" type="com.example.demo.bean.UserFormBean">
        <result column="sex" property="sex" javaType="java.lang.String" jdbcType="VARCHAR"/>
        <association property="user" javaType="com.example.demo.bean.User">
            <result column="name" property="name" javaType="java.lang.String" jdbcType="VARCHAR"/>
            <result column="en_name" property="enName" javaType="java.lang.String" jdbcType="VARCHAR"/>
            <result column="sex" javaType="java.lang.Integer" jdbcType="INTEGER"/>
        </association>
        <association property="department" javaType="com.example.demo.bean.Department">
            <result column="department" property="department" javaType="java.lang.String" jdbcType="VARCHAR"/>
        </association>
    </resultMap>
    <select id="select" resultMap="FormResultMap">
        select
            u.name,
            u.en_name,
            u.sex,
            d.department,
            case when u.sex = 0 then '男'
                when u.sex = 1 then '女' end sex
        from user u
        left join department d on u.department_id = d.id
    </select>
</mapper>
```
执行查询Log，可以看到总共查询出来3条：  　
![log](/images/mybatis/2.jpg) 
debut看查出来的List，发现只有2条！：   
![debug](/images/mybatis/3.jpg)  
## 解决
仔细看结果集合，发现是按照sex的类别去查的！mybatis把相同的sex当成同一条记录去处理了！为什么会这样呢？看看我写的resultMap!
里边除了2个association还有一个我刚加的sex字段！所以mybatis就根据这个字段去标识一条记录了！那么解决方案就很明确了，老老实实把标识字段加上！   
UserMapper.xml里的<resultMap>里加入<id>:
```
<id column="id" property="id" javaType="java.lang.String" jdbcType="VARCHAR"/>
```
UserFormBean增加字段id:
```
 private String id;
```
debug查看结果集，都查出来了！
![debug](/images/mybatis/4.jpg)  