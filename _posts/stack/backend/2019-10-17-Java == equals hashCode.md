---
title: "Java == equals hashCode"
categories:
  - backend
tags:
  - java
classes: wide

excerpt: "==和equals的区别，equals和hashCode的联系"
---

# 1 ==
判断值是否相同

针对`Primitive Type`，就是比值

针对`Reference Type`，就是比Object在堆上的地址

```
int primitiveA = 123;
int primitiveB = 123;

System.out.println("== for Primitive Type: ");
System.out.println(primitiveA == primitiveB);

System.out.println("-----------------------");

Integer referenceA = new Integer(123);
Integer referenceB = new Integer(123);
System.out.printf("referenceA hex address: %x \n", System.identityHashCode(referenceA));
System.out.printf("referenceB hex address: %x \n", System.identityHashCode(referenceB));
System.out.println("== for Reference Type: ");
System.out.println(referenceA == referenceB);
```

```
== for Primitive Type: 
true
-----------------------
referenceA hex address: 38af3868 
referenceB hex address: 87aac27 
== for Reference Type: 
false
```

# 2 equals

java.lang.Object中的equals方法实现如下，如其他类没有重写equals方法，则就是比较两个对象地址是否相同
```
public boolean equals(Object obj) {
        return (this == obj);
    }
```

对于`Primitive Type`是没有equals方法的，但是对应的Wrapper类有，且重写了Object的equals方法，Integer重写equals方法如下，可以看到是比较值：
```
public boolean equals(Object obj) {
        if (obj instanceof Integer) {
            return value == ((Integer)obj).intValue();
        }
        return false;
    }
```

```
Integer intA = new Integer(123);
Integer intB = new Integer(123);
System.out.printf("intA hex address: %x \n", System.identityHashCode(intA));
System.out.printf("intB hex address: %x \n", System.identityHashCode(intB));
System.out.println("equals for Reference Type Integer: ");
System.out.println(intA.equals(intB));

System.out.println("-----------------------");

String strA = new String("123");
String strB = new String("123");
System.out.printf("strA hex address: %x \n", System.identityHashCode(strA));
System.out.printf("strB hex address: %x \n", System.identityHashCode(strB));
System.out.println("equals for Reference Type String: ");
System.out.println(strA.equals(strB));

System.out.println("-----------------------");

Object objA = new Object();
Object objB = new Object();
System.out.printf("objectA hex address: %x \n", System.identityHashCode(objA));
System.out.printf("objectB hex address: %x \n", System.identityHashCode(objB));
System.out.println("equals for Reference Type Object: ");
System.out.println(objA.equals(objB));
```
```
intA hex address: 38af3868 
intB hex address: 87aac27 
equals for Reference Type Integer: 
true
-----------------------
strA hex address: 3e3abc88 
strB hex address: 6ce253f1 
equals for Reference Type String: 
true
-----------------------
objectA hex address: 53d8d10a 
objectB hex address: e9e54c2 
equals for Reference Type Object: 
false
```

# 3 关于equals和hashCode
Java SE API中有关于hashCode方法有如下`contract`：  
The general contract of hashCode is:

- Whenever it is invoked on the same object more than once during an execution of a Java application, the hashCode method must consistently return the same integer, provided no information used in equals comparisons on the object is modified. This integer need not remain consistent from one execution of an application to another execution of the same application.
- If two objects are equal according to the equals(Object) method, then calling the hashCode method on each of the two objects must produce the same integer result.
- It is not required that if two objects are unequal according to the equals(java.lang.Object) method, then calling the hashCode method on each of the two objects must produce distinct integer results. However, the programmer should be aware that producing distinct integer results for unequal objects may improve the performance of hash tables.

简而言之就是如果两个对象的equlas返回为true，那么这两个对象的hashCode方法返回必须相同，反过来不做要求

hashCode的返回值主要用于HashMap中的key（会出现冲突的情况），冲突时则挨个比较是否equals

![](https://raw.githubusercontent.com/ZQQ1024/pictures/master/20191117103054.png)

```
Integer intA = new Integer(123);
Integer intB = new Integer(123);
System.out.println("equals for Reference Type Integer: ");
System.out.println(intA.equals(intB));

System.out.printf("intA hashCode: %x \n", intA.hashCode());
System.out.printf("intB hashCode: %x \n", intB.hashCode());
```
```
equals for Reference Type Integer: 
true
intA hashCode: 7b 
intB hashCode: 7b 
```

所以，为了HashMap能正常工作：
- 当两个对象的hashCode值相同的，equals不必相同
- 当两个对象equals时，hashCode要求相同

# 4 References
> [https://stackoverflow.com/questions/4360035/why-can-hashcode-return-the-same-value-for-different-objects-in-java](https://stackoverflow.com/questions/4360035/why-can-hashcode-return-the-same-value-for-different-objects-in-java)  
[https://docs.oracle.com/javase/8/docs/api/](https://docs.oracle.com/javase/8/docs/api/)