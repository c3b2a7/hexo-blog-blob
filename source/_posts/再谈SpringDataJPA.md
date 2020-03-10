---
title: 再谈SpringDataJPA
categories: 正常的文章
date: 2019-12-27 18:21:08
tags: [Java,Spring,JPA]
---

## 前言  
在 {% post_link 关于SpringBoot使用JPA进行更新操作 %} 这一篇文章中曾提到了关于jpa使用save方法更新记录时会出现当有些参数为null时，save操作会用null覆盖数据库中的字段的情况，通常我们的需求是动态的去更新记录，而不是全部覆盖，所以对比起Mybatis的动态sql，Jpa不太灵活的特性就暴露出来，事实上，对于动态更新虽然实现上麻烦了点，但还是能操作一下的。

## 以前的方法 
对于以前实现动态更新，我们无非是使用@Query注解来写原生sql,就像这样：
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
但往往我们每张表都会有动态更新的需求，按照这样的方法，都写一遍原生sql实在是又麻烦又易错，而且这种方法实际上并没有真正做到**动态**，而是我们自己把需要更新的部分写死了，当我们需要更新其他字段岂不是又要再写一个方法？总之，很是麻烦。

## 动态sql新姿势  
假设现在有好几张表：user、rank ...，对于所有的有动态更新需求的表都要进行实现的话，那么我们可以这样做：
先定义一个`BaseRepository`
```java
/**
 * @author lolicom
 */
@NoRepositoryBean
public interface BaseRepository<T, ID extends Serializable> extends JpaRepository<T, ID>, JpaSpecificationExecutor<T> {
    
    T dynamicUpdate(T entity);
    
}
```
然后定义一个`BaseRepositoryImpl`作为BaseRepository的默认实现
```java
public class BaseRepositoryImpl<T, ID extends Serializable> extends SimpleJpaRepository<T, ID> implements BaseRepository<T, ID> {
    
    private Class<T> entityClass;
    private EntityManager entityManager;
    private JpaEntityInformation<T, ?> entityInformation;
    
    public BaseRepositoryImpl(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {
        super(entityInformation, entityManager);
        this.entityClass = entityInformation.getJavaType();
        this.entityManager = entityManager;
        this.entityInformation = entityInformation;
    }
    
}
```
然后在配置类上加上注解`@EnableJpaRepositories(repositoryBaseClass = BaseRepositoryImpl.class)`，看到这里，应该有朋友会感觉有点熟悉。这正是我们为所有Repository自定义公共方法的实现方法，我们在BaseRepository中定义共有方法，然后在`BaseRepositoryImpl`中提供实现，后续所有继承`BaseRepository`这个接口的Repository都可以获得这个默认实现，因为我们在注解`@EnableJpaRepositories(repositoryBaseClass = BaseRepositoryImpl.class)`中指定了Repository的默认实现用BaseRepositoryImpl，而BaseRepositoryImpl继承`SimpleJpaRepository`这个jpa默认的实现，所以我们相当于扩展JpaRepository，好的，扯远...所以现在我们只要在BaseRepository中定义`T dynamicUpdate(T t);`这样一个方法,然后在BaseRepositoryImpl中提供实现，那么所有repository都可以获得这个实现，所以，说到最后，怎么实现这样一个方法呢？
```java
/**
 * @author lolicom
 */
@Transactional(readOnly = true)
public class BaseRepositoryImpl<T, ID extends Serializable> extends SimpleJpaRepository<T, ID> implements BaseRepository<T, ID> {
    private Class<T> entityClass;
    private EntityManager entityManager;
    private JpaEntityInformation<T, ?> entityInformation;
    
    public BaseRepositoryImpl(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {
        super(entityInformation, entityManager);
        this.entityClass = entityInformation.getJavaType();
        this.entityManager = entityManager;
        this.entityInformation = entityInformation;
    }
    
    @Override
    @Transactional
    public T dynamicUpdate(T entity) {
        BeanWrapper wrapper = new BeanWrapperImpl(entity);
        Object id = entityInformation.getRequiredId(entity); //获取id的值
        T old = entityManager.find(entityClass, id); //先根据id从数据库中查出记录
        Set<String> nullProperty = new HashSet<>();  //存放为null的属性名
        for (PropertyDescriptor propertyDescriptor : wrapper.getPropertyDescriptors()) {
            String propertyName = propertyDescriptor.getName(); //属性名
            Object propertyValue = wrapper.getPropertyValue(propertyName);  //属性值
            if (propertyValue == null) { //如果属性值为null就添加到set中
                nullProperty.add(propertyName);
            }
        }
        BeanUtils.copyProperties(entity, old, nullProperty.toArray(new String[0])); //将t中属性的值复制到old中，忽略nullProperty.toArray(new String[0])中的属性
        return entityManager.merge(old); //更新并返回更新后的结果
    }
}
```
思路注释里已经写得很清楚了。  
对于`BeanWrapperImpl`和`BeanUtils`这两个类，都是利用反射去获取Bean的各种信息以及进行操作的类。  
前者是Bean的包装类，后者是Bean工具类,两者有什么区别呢？  
看类名也能猜出来，前者包装bean后可以获得这个bean的一些信息，比如有那些属性，属性的值是什么，值可不可以修改，能不能读取等等（侧重于对传进去的bean对象进行操作）。  
后者提供了一些静态方法，用于实例化bean,检查bean的属性类型，复制属性值等（侧重于对类的信息和bean之间属性进行操作）。

## 测试
```java User.java
/**
 * @author lolicom
 */
@Entity
@Table(name = "user")
@DynamicUpdate
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

```java UserRepository.java
@Repository
public interface UserRepository extends CustomUserRepository, BaseRepository<User, Integer> {
    
}
```

测试方法：
```java
    @Test
    void dynamicUpdate() {
        User user = new User();
        user.setId(3);
        user.setUsername("test3");
        user = repository.dynamicUpdate(user);
        System.out.println(user);
    }
```
对数据库表中id为3的记录进行更新，将username设置为test3,原本的记录是
3&emsp;&emsp;test&emsp;&emsp;test3&emsp;&emsp;test3@ttt
执行方法，输出：

![结果](https://lolico.griouges.cn/images/NUKi.png) 
两条sql,先查询再更新，更新的sql中可以看到只对uname进行set，这是因为我们在User实体类上注解了`@DynamicUpdate`
**注意**这个注解并不是动态更新记录的意思，这个注解的意思是在进行merge更新时，会对比数据库中的记录，如果不一致就生成对应set的语句，所以可以看到我们测试输出的更新sql语句里只有对uname进行set的sql，因为其他字段的值与数据库记录是一致的，利用这个注解我们可以实现对于数据库中设置默认值比如`DateTime`，在使用`@DynamicInsert`后就不会插入`null`值，而是使用默认的值。讲的有点乱，还没理解的可以去看看这篇文章[关于@DynamicUpdate的误解](https://juejin.im/post/5c68099ef265da2de04aa82f)
现在看下数据库中id=3的记录是否更新了uname：

![更新后的记录](https://lolico.griouges.cn/images/N6Uj.png) 
与输出的一致，很完美！

## 总结 
对于动态sql，利用这个方法或许性能不太行（因为先查询了一下），但可行性上来说还是可以的，既然想体验Spring Data JPA自动生成实现的功能，自然就很难再保证Mybatis那种灵活的特性。
