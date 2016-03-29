---
layout: article
title: Foreach in Java
---

# Foreach in Java

### 概念
---

在Java中，Foreach即指代Java中下列语法形式

     for ( FormalParameter : Expression ) Statement

语法中的Expression支持两种格式，要么是一种实现了Iterable接口的类型，要是是数组类型T[]. 
      
本文仅讨论第一种，即实现了Iterable接口类型的Expression。  

对于此类型的Expression，Foreach语句会把上述语法转换成一下形式  
    
    for (I i = Expression.iterator(); i.hasNext(); ) {
        TargetType Identifier = (TargetType) i.next();
        Statement
    }

example:

     List<? extends Integer> l = ...
     for (float i : l) ...
      

将会被转换成

    for (Iterator<Integer> i = l.iterator(); i.hasNext(); ) {
        float i0 = (Integer)i.next();
        ...

由此可见，Foreach的实现是Iterable接口中hasNext()和next()的实现决定的。

### Example
---

遇到这个问题是因为自己在实现一个数据结构Bag的时候，发现测试没通过。  
Bag是一个容器，具体代码就不贴了，以下是测试用例。

     public static void main(String args[]){
        Bag<Integer> bag = new Bag<>();
        bag.add(1);
        bag.add(2);
        bag.add(4);
        bag.add(3);
        for(int i : bag){
            System.out.println(i);
        }
        System.out.println("length: "+bag.size());
    }

我们来看看这段代码的字节码（节选）

    45: aload_2
    46: invokeinterface #15,  1           // InterfaceMethod java/util/Iterator.hasNext:()Z
    51: ifeq          77
    54: aload_2
    55: invokeinterface #16,  1           // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
    60: checkcast     #17                 // class java/lang/Integer
    63: invokevirtual #18                 // Method java/lang/Integer.intValue:()I
    66: istore_3
    67: getstatic     #19                 // Field java/lang/System.out:Ljava/io/PrintStream;
    70: iload_3
    71: invokevirtual #20                 // Method java/io/PrintStream.println:(I)V
    74: goto          45

通过以上我们能很明显的看出的确是依赖hasNext()和next()实现的。

