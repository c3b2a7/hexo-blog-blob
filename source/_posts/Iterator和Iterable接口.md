---
title: Iterator和Iterable接口
categories: 正常的文章
date: 2018-11-14 16:30:32
tags: Java
---

通常情况下,我们使用集合框架时会希望能够遍历一个集合中的元素。例如，显示集合中的每个元素。
一般的，我们都是采用for循环或增强for（for-each），这两个方法也可以用在集合框架。但是在集合框架中还有一种方法是采用迭代器遍历集合框架，它是一个对象，实现了Iterator接口或ListIterator接口。

> 迭代器（Iterator），使你能够通过循环来得到或删除集合的元素。ListIterator，继承了Iterator以允许双向遍历列表和修改元素。

## 两个问题

### 什么是foreach循环？
for-each是增强型的for循环，是Java5的新特性之一。<br>foreach语法格式如下：
```java
    for(元素类型t 元素变量x : 遍历对象obj){
        引用了x的java语句
    }
```

### 什么时候可以使用foreach进行遍历？
可以被foreach遍历的对象分两种：
1. **数组**
java对于数组的foreach处理相对来说简单。在编译期将foreach还原成简单的for循环。
2. **实现了Iterable接口的对象**
实现了`Iterable`接口的对象，可以被foreach。并且java会通过迭代器的形式来遍历他。

----------

 - **Iterable接口：**故名思议，实现了这个接口的集合对象支持迭代，是可迭代的。
 - **Iterator接口：**迭代器，它就是提供迭代机制的对象，具体如何迭代，都是`Iterator`接口规范的。

## Iterable和Iterator

### Iterable
我们看看`Iterable`接口的用途是什么：
```doc
/**
 * Implementing this interface allows an object to be the target of
 * the "for-each loop" statement. See
 * <strong>
 * <a href="{@docRoot}/../technotes/guides/language/foreach.html">For-each Loop</a>
 * </strong>
 *
 * @param <T> the type of elements returned by the iterator
 *
 * @since 1.5
 * @jls 14.14.2 The enhanced for statement
 */
```
> 实现这个接口，允许一个对象成为for-each的目标

再来看看接口中定义了哪些需要实现的方法
```java Iterable.java
public interface Iterable<T> {
    /**
     * Returns an iterator over elements of type {@code T}.
     *
     * @return an Iterator.
     */
    Iterator<T> iterator();
}
```
上面我们说过了，实现了`Iterable`接口的对象，可以被foreach。并且java会通过**迭代器**的形式来遍历他。
`Iterable`接口中的`Iterator<T> iterator()`方法返回的就是一个迭代器（Iterator)

### Iterator
包含2个需要实现的方法：`hasNext()`、`next()` 和一个默认方法`remove()`。
我们以`ArrayList`为例：
```java
List<Object> list=new ArrayList<>();
for(Object obj:list);
```
编译后查看class文件:
```
List<Object> list = new ArrayList();

Object var3;
for(Iterator var2 = list.iterator(); var2.hasNext(); var3 = var2.next()) {
}
```
可以看到是使用了迭代器来遍历集合,首先获取集合的迭代器后调用`hasNext()`方法判断是否迭代到终点,
`next()`方法不仅要返回集合元素（下一个）,还要将迭代器的游标后移。

## 定义一个实现了Iterable接口的类
```java IterableTester.java
public class IterableTester<E> implements Iterable<E> {
	private Object[] elements = {}; //存放泛型对象的数组,取出时需要进行强制类型转换
	private int size = 0;
	
	IterableTester(int initialCapacity) {
		if (initialCapacity > 0) {
			elements = new Object[initialCapacity];
		}
	}
	
	public boolean add(E element) {
		if (size < elements.length) {
			elements[size] = element;
			size++;
			return true;
		} else {
			return false;
		}
	}
	
	public E get(int index) {
		//rangeCheck(index);
		return (E) elements[index]; //存放在Object类型的数组中，所以在取出的时候，需要进行强制类型转换。
	}
	
	@Override
	public Iterator<E> iterator() {
		return new itr(); //返回一个迭代器
	}
	
	//内部类,实现Iterator接口
	private class itr implements Iterator<E> {
		int cursor = 0;
		
		@Override
		public boolean hasNext() {
			//如果游标不等于元素个数,则还有下一个
			return cursor != size;
		}
		
		@Override
		public E next() {
			if (cursor >= size || cursor >= elements.length)
				throw new RuntimeException();
			return (E) elements[cursor++];
		}
	}
}
```
执行for-each的时候，会先调用`iterator()`方法返回一个迭代器。然后调用迭代器的`hasNext()`和`next()`方法来进行遍历。
### 测试样例1
```java
IterableTester<String> its = new IterableTester<>(5);
its.add(new String("i "));
its.add(new String("am "));
its.add(new String("a "));
its.add(new String("coder"));
for (String s : its)
    System.out.print(s);
```
#### 输出
`i am coder`

#### 需要注意的地方
**迭代出来的元素都是原集合元素的拷贝**
Java集合中保存的元素实质是*对象的引用*(可以理解为C中的指针)，而非对象本身。
迭代出的元素也就都是*引用的拷贝*，结果还是引用。那么，如果集合中保存的元素是可变类型的，我们就可以通过迭代出的元素修改原集合中的对象。而对于不可变类型，如`String`或者基本类型的包装类型`Integer`等则不会反应到原集合中。
![](https://i.loli.net/2020/03/09/qJawrHYBDdmz5G4.png)

### 测试样例2
```java
public class Main {
	public static void main(String[] args) {
		IterableTester<Person> itp = new IterableTester<>(5);
		ArrayList<String> als = new ArrayList<>();
		Person p = new Person("Tom");
		itp.add(p);
		als.add(new String("Tom"));
		for (Person p_ : itp)
			p_.setName("Jack"); //修改了原集合中的元素(Person对象)
		for (String s_ : als)
			s_ = "Jack"; //实际上是S_=new String("Jack"),并没有对原集合中的元素进行修改
		System.out.println(itp.get(0).getName());
		System.out.println(als.get(0));
	}
}

class Person {
	private String name;
	Person(String name) {
		this.name = name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public String getName() {
		return name;
	}
}
```
#### 输出
```
Jack
Tom
```
## 值得思考的问题
既然说:迭代出来的元素都是原集合元素的拷贝,如果集合中保存的元素是可变类型的，我们就可以通过迭代出的元素修改原集合中的对象。那么我们可不可以在将迭代元素修改时却不影响原集合元素呢?答案是肯定的,大概有两个实现的方法(这里只说一下大概的思路):

 - 将集合定义成不可变类,具体可看我的这篇博文：{% post_link Java中的可变类与不可变类 %}
 - 在实现`Iterator#next`方法时,返回集合元素的一个克隆,具体可看我的这篇博文：{% post_link Java中的对象克隆（复制） %}
