#### java原子更新器AtomicReferenceFieldUpdater的使用

##### 概要

* [摘自1](http://huangyunbin.iteye.com/blog/1944153)
* [Java多线程系列--“JUC原子类”04之 AtomicReference原子类](http://www.cnblogs.com/skywang12345/p/3514623.html)

##### 介绍
* AtomicReferenceFieldUpdater 一个基于反射的工具类，它能对指定类的指定的volatile引用字段进行原子更新。(注意这个字段不能是private的)
* 通过调用AtomicReferenceFieldUpdater的静态方法newUpdater就能创建它的实例，该方法要接收三个参数：
 + 包含该字段的对象的类
 + 将被更新的对象的类
 + 将被更新的字段的名称
