---
title: Golang Reflection
date: 2020-03-26 20:10
tags: golang
---

<!--more-->

## Reflect Package

### 1. reflect.Type

```go


```
### 2. reflect.Kind



### Interface{}
接下来说一说经常用到的 interface{} 结构体，
传参的时候如果接收参数是interface{}，其实就用到隐式的反射，但是这个interface{}本身的属性也比较特殊；

我们可以先看一个问题:
```go
type Animal interface{
    Walk()
}
type Cat struct{

}
func (c *Cat) Walk(){
    log.Println("walking! cat")
}
type i2 interface{

}
func 

func main(){
    var t1 i1
    var t2 i2
    if t1==t2{
        log.Println("equal!")
    }else{
        log.Println("nope!")
    }
}

```


除了反射其还可以当做原本的用途： 多态 （OOP）的特征之一，只是golang是ducktype类型，实现多态的几个要求：
1. 有interface接口和方法，有子类把那个接口的方法都实现了，编译器就会自动认为这个结构体就是使用了这个接口
2. 父类指针指向子类的具体对象

满足以上就可以实现**多态**

一般有两种写法：
```go
//方法内嵌
type doInterfaceWithMethod interface{
    Do1(string) string
}
//方法放在外面
type doInterfaceWithoutMethod interface{

}
func (m *doInterfaceWithoutMethod) Do1(s string) string{}

```
