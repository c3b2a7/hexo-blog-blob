---
title: EntityManagerFactory和EntityManager
categories: 正常的文章
date: 2019-11-12 16:53:19
tags: [Java,Spring,JPA,Hibernate]
---

## 常见异常及可能的解决办法

### `javax.persistence.TransactionRequiredException: no transaction is in progress`

可能的原因：

 - 没有开启事务
 - 开启事务没有生效

`@Transactional`注解是作用于`EntityManager`的操作的层级上
`EntityManagerFactory#createEntityManager`生成的`EntityManager`对于写操作手动管理事务（开启-操作-提交）时，需要注意的是：**对于一个操作中的所有子操作要在一个事务中完成**



### `java.lang.IllegalStateException: Transaction not successfully started`

`EntityManagerFactory#createEntityManager`每次都会返回一个新的实例，所以通过`EntityManagerFactory.createEntityManager().getTransaction()[.begin()|.commit()]`开启和提交事务是不行的，例如：

```java
    public void insertOne(User user) {
        emf.createEntityManager().getTransaction().begin();
        emf.createEntityManager().persist(user);
        emf.createEntityManager().getTransaction().commit();
    }
```
**正确方式：**
```java
    public void insertOne(User user) {
        EntityManager entityManager = emf.createEntityManager();
        entityManager.getTransaction().begin();
        entityManager.persist(user);
        entityManager.getTransaction().commit();
    }
```

### `org.springframework.dao.InvalidDataAccessApiUsageException: Removing a detached instance;nested exception is java.lang.IllegalArgumentException: Removing a detached instance`

```java
    public void deleteById(int id) {
        EntityManager entityManager = emf.createEntityManager();
        entityManager.getTransaction().begin();
        entityManager.remove(getById(id));
        entityManager.getTransaction().commit();
    }
    public User getById(int id) {
        return emf.createEntityManager().find(User.class, id);
    }
```
**可能的原因：**注意getById(int id)中的内容，`EntityManagerFactory#createEntityManager`()`每次返回一个新的实例，即在一个事务中操作两个不同的entityManager。
通常：在删除一个游离状态的实体时也会报这个异常。对于实体不同状态下的操作见：[JPA EntityManager的四个主要方法 ——persist,merge,refresh和remove](https://blog.csdn.net/u013456370/article/details/52672383)
**正确方式：**
```java
    public void deleteById(int id) {
        EntityManager entityManager = emf.createEntityManager();
        entityManager.getTransaction().begin();
        entityManager.remove(entityManager.find(User.class,id));
        entityManager.getTransaction().commit();
    }
```

### `java.lang.IllegalArgumentException: attempt to create delete event with null entity`

删除的实体不存在，检查需要删除的记录是否不存在于数据库的表中。

----------
## SimpleJpaRepository#saveAndFlush和SimpleJpaRepository#save方法

### save方法

 - 在没有主键时调用EntityManager.persist方法执行插入操作。
 - 在有主键时调用EntityManager.merge方法执行更新操作。

### saveAndFlush方法
调用save方法后调用flush方法，将PersistenceContext的信息刷新到数据库中，当触发flush这个动作的时候，所有的实体都将会被insert/update/remove到数据库中，数据库不会触发Commit的操作。

### 源码
```java
	@Transactional
	@Override
	public <S extends T> S save(S entity) {

		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}

	@Transactional
	@Override
	public <S extends T> S saveAndFlush(S entity) {

		S result = save(entity);
		flush();

		return result;
	}
```
## EntityManager#merge和EntityManager#persist方法

> persist(),是保存，跟save（）方法一样，知识jpa官方说叫persist比较好一些，更接近持久化的含义。而merge（）是合并的意思，就是当你保存的实体，根据主键id划分，如果已存在，那么就是更新操作，如果不存在，就是新增操作。persist会把传进去的实体放到持久化上下文中，此时如果持久化上下文中有了这个实体，就会抛出javax.persistence.EntityExistsException，没有的话事务提交的时候把那个对象加进数据库中，如果数据库中已经存在了那个对象（那一行），就会抛出com.mysql.jdbc.exceptions.jdbc4.MySQLIntegrityConstraintViolationException；而merge会在持久化上下文中生成传进去的实体的受管版本，如果已经有了受管版本，那也不会抛出异常，然后把那个受管的实体返回出来，事务提交的时候如果数据库中不存在那个对象（那一行），就把把那个受管的加进去，存在的话就替换掉原来的数据。merge是如果持久化上下文中有了受管版本，那就更新，没有就复制一份，返回受管的。再次总结persist（①，②-③，④-⑤）： （这里说的抛出的异常都是指对象（或者数据库中的行）重复的异常） ①如果persist的是一个受管实体（即已经在上下文中），就不会抛出异常。②如果persist的是一个游离实体（即上下文中没有它），而上下文中又没有它的受管版本，数据库中也没有，也不会抛出异常，而会把这个实体写进数据库中。③如果persist的是一个游离实体（即上下文中没有），而上下文中又没有它的受管版本，数据库却有这个实体，那么EntityManager在persist它的时候不会抛出异常，但是事务提交的时候就会抛出异常：Caused by: com.mysql.jdbc.exceptions.jdbc4.MySQLIntegrityConstraintViolationException: Duplicate entry '7' for key 1；④如果persist的是一个游离实体（即上下文中没有），而上下文中却有它的受管版本，数据库中又没有这个实体，那么还是不会抛出异常，而是把它的受管版本加进去（不是那个游离的，是那个受管的！）（即，这种情况persist和没persist是一样的！）。 ⑤如果persist的是一个游离实体（即上下文中没有），而上下文中却有它的受管版本，数据库中也有了这个实体，那么EntityManager在persist它的时候就会抛出异常：javax.persistence.EntityExistsException 而merge就不会抛出什么对象重复的异常的了。

事实上，经过测试，`EntityManager#merge`方法在没主键时会执行插入操作（可用于插入记录操作）。但是无法通过`EntityManager#merge`插入自定义主键的记录，而`EntityManager#persist`在有无主键时都会进行插入操作，对于主键的值会根据表主键的生成策略进行生成。

即：persist是直接保存，merge是根据id是否存在来判断是保存还是修改（id存在，则修改； id不存在，则添加）
