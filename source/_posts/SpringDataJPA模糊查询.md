---
title: Spring Data Jpa模糊查询
categories: 正常的文章
date: 2019-11-15 17:24:39
tags: [Java,Spring,JPA]
---

## 实体类
```java User.java
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
    
    public void setId(int id) {
        this.id = id;
    }
    
    @Basic
    @Column(name = "uname")
    public String getUsername() {
        return username;
    }
    
    public void setUsername(String username) {
        this.username = username;
    }
    
    @Basic
    @Column(name = "upassword")
    public String getPassword() {
        return password;
    }
    
    public void setPassword(String password) {
        this.password = password;
    }
    
    @Basic
    @Column(name = "uemail")
    public String getEmail() {
        return email;
    }
    
    public void setEmail(String email) {
        this.email = email;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User that = (User) o;
        return id == that.id &&
                Objects.equals(username, that.username) &&
                Objects.equals(password, that.password) &&
                Objects.equals(email, that.email);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id, username, password, email);
    }
    
    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", username='" + username + '\'' +
                ", password='" + password + '\'' +
                ", email='" + email + '\'' +
                '}';
    }
}
```

## 不依赖数据库的实现方法

### 自定义查询语句（concat函数）
mysql:concat:用于将多个字符串连接成一个字符串，是最重要的mysql函数之一。
```java
    @Query("select u from User u " +
            " where (u.username like concat('%',?1,'%') or ?1 is null) " +
            " and u.email=?2 ")
    List<User> findByUsernameLikeAndEmail(String username, String email);
```
```java
    @Test
    void findByUsernameLikeAndEmail() {
        System.out.println(repository.findByUsernameLikeAndEmail("test", "test3@ttt"));
    }
```
输出：
[User{id=3, username='test3', password='test3', email='test3@ttt'}, User{id=5, username='test3', password='test3', email='test3@ttt'}, User{id=6, username='test3', password='test3', email='test3@ttt'}, User{id=7, username='test3', password='test3', email='test3@ttt'}]

### 自定义查询语句
```java
    @Query("select u from User u " +
            " where (u.username like ?1 or ?1 is null) " +
            " and u.email=?2")
    List<User> findByUsernameLikeAndEmail(String username, String email);
```
```java
    @Test
    void findByUsernameLikeAndEmail() {
        System.out.println(repository.findByUsernameLikeAndEmail("%test%", "test3@ttt"));
    }
```
输出：同方式一
注意：效果同方式一，调用传参的不同，相比方式一要自己传入包含模式的匹配字符串："te_t%"、"test%"等.

方式一和方式二调用：`repository.findByUsernameLikeAndEmail(null, "test8@ttt")`
输出：[User{id=8, username='null', password='8', email='test8@ttt'}]

> `u.username like %?1% or ?1 is null`和`u.username like concat('%',?1,'%') or ?1 is null`代表不存在则忽略模糊查询条件，存在则进行模糊查询。

### jpa自动提供实现的方式
```java
List<User> findByUsernameLikeAndEmail(String username,String email);
```
```java
    @Test
    void findByUsernameLikeAndEmail() {
        System.out.println(repository.findByUsernameLikeAndEmail("%test%", "test3@ttt"));
    }
```
输出：同方式一
```java
    @Test
    void findByUsernameLikeAndEmail() {
        System.out.println(repository.findByUsernameLikeAndEmail(null, "test8@ttt"));
    }
```
由于jpa自动提供实现，没有`or ?1 is null`逻辑，传参`null`进行调用抛出异常：
异常：`org.springframework.dao.InvalidDataAccessApiUsageException: Value must not be null!; nested exception is java.lang.IllegalArgumentException: Value must not be null!`

## 利用不同数据库中提供的函数的实现方法

### mysql:contains
MYSQL中可以用`CONTAINS`函数,在全文索引的的字段中使用：
```java
@Query(value = " select * from event e "
			+ " where (?1 is null or CONTAINS(e.event_title,'?1')) "
			+ " and (to_days(e.register_time)=to_days(?2) or ?2 is null) "
			+ " and e.status = '1' "
			+ " order by e.register_time desc limit ?3,?4 ",nativeQuery = true)
	List<Event> findAllList(String eventTitle,Timestamp registerTime,Integer pageNumber,Integer pageSize);
```
还可以用 `find_in_set()` 方法,例如:`select * FROM users WHERE find_in_set('aa@email.com', emails)`;

### oracle:instr
Oracle中提供了 `instr(strSource,strTarget)`函数，比使用 `LIKE %关键字%` 的模式效率高很多。

instr函数也有三种情况：

instr条件 | 相当于like什么
:----: | :----: 
instr(字段,‘关键字’)>0 | 字段like ‘%关键字%’
instr(字段,‘关键字’)=1 | 字段like ‘关键字%’
instr(字段,‘关键字’)=0 | 字段not like ‘%关键字%’

### pgsql:~~用法

 - `~` 表示匹配正则表达式，且区分大小写。
 - `~*` 表示匹配正则表达式，且不区分大小写。
可以通过这两个操作符来实现like和ilike一样的效果,^和$就是正则表达式里的用法，如下：

1. 匹配以“张”开头的字符串 : `select * from table where name ~ ‘^张’`;

2. 匹配以“小”结尾的字符串 : `select * from table where name ~ ‘小$’`;

 - `!~`是`~`的否定用法，表示不匹配正则表达式，且区分大小写。
 - `!~*`是~*的否定用法，表示不匹配正则表达式，且不区分大小写。
 - `~~`等效于`like`，`~~*`等效于`ilike`。
 - `!~~`等效于`not like`，`!~~*`等效于`not ilike`。

