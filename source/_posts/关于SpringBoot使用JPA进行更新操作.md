---
title: 关于SpringBoot使用JPA进行更新操作
categories: 正常的文章
date: 2019-11-14 17:08:44
tags: [Java,Spring,SpringBoot,JPA]
---

使用`SimpleJpaRepository#save`（JpaRepository的默认实现，更新操作本质上是调用`EntityManager#merge`方法）进行更新操作时会发现：在传入的对象只有部分参数时，更新后数据库中该记录的其他字段为`null`

**解决：**
```java
    @Transactional
    @Modifying
    @Query("update User u set u.email=:#{#user.email} where u.id=:#{#user.id}")
    void dynamicUpdateEmailById(@Param("user") User user);
    
    @Transactional
    @Modifying
    @Query("update  User u set u.email=?2 where u.id=?1")
    void dynamicUpdateEmailById(int id, String email);
```

----------

**对于调用`SimpJpaReposiory#sava`插入新纪录后get不到id(主键)的问题**

添加`@GeneratedValue`注解修改主键生成策略为`GenerationType.IDENTITY`，默认是`GenerationType.AUTO`
**异常：**
`org.springframework.dao.DataIntegrityViolationException: could not execute statement; SQL [n/a]; constraint [PRIMARY]; nested exception is org.hibernate.exception.ConstraintViolationException: could not execute statement`
**解决：**
使用`GenerationType.IDENTITY`
```java
@Entity
@Table(name = "user")
public class User {
    private int id;
    private String username;
    private String password;
    private String email;
    
    @Id
    @Column(name = "id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    public int getId() {
        return id;
    }
    ...
}
```
或
```java
@Entity
@Table(name = "user")
public class User {
    @Id
    @Column(name = "id")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;
    private String username;
    private String password;
    private String email;
    
    public int getId() {
        return id;
    }
    ...
}
```
**注意：**
测试将`@GeneratedValue(strategy = GenerationType.IDENTITY)`标注在属性（`@Id`、`@Column`等还是标注于getter方法)时不生效，原因可能是PPB在解析属性和表字段映射时要么从getter方法进行解析绑定，要么从属性解析绑定，而不是解析属性和对应getter方法进行合并。
