# Mybatis-结果集映射嵌套对象
最近在做项目的时候，遇到要联合查询两张表展示相关字段。查询逻辑很简单，但是发现这两张表要展示的字段太太太多了，光是copy到一个FormBean上我都嫌累，而且页面老长了。
于是乎想到做嵌套对象，即FormBean里的属性既是我要关联查询的两张表对象。那么如何让mybatis把查出来的结果集映射成我需要的嵌套对象？用resultMap里association！  
##
eg:关联查询user和department，把查询结果封装到嵌套对象user、department的userFormBean中
## 
user:
```
    @Data
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public class User extends BaseBean implements Serializable {

    private static final long serialVersionUID = 1L;

    private String name;

    private Integer sex;

    private String enName;

    private String departmentId;
}
```
department:
```
    @Data
    @Builder
    @AllArgsConstructor
    @NoArgsConstructor
    public class Department extends BaseBean implements Serializable {

    private static final long serialVersionUID = 1L;

    private String department;
}
```
UserFormBean:
```
@Data
public class UserFormBean {
    private Department department;
    private User user;
    private String sex;
}
```
userMapper.xml:
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.mapper.UserMapper">
    <resultMap id="FormResultMap" type="com.example.demo.bean.UserFormBean">
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
结果集合图示：  
![result](/images/mybatis/1.jpg)     