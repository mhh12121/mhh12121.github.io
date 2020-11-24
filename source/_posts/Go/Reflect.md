---
title: Golang Reflection
date: 2020-03-26 20:10
tags: golang
---
# Reflect Package

很多类型在`runtime`里是未导出的，比如`_type`,`typeAlg`,`interfaceType`,`eface`,`iface`等等，reflect包就是用于对应导出这些类型;
所以他们的结果在两个包中其实是一样的;

<!--more-->
空接口:
reflect包下:
```go
// emptyInterface is the header for an interface{} value.
type emptyInterface struct {
	typ  *rtype//指的是动态类型
	word unsafe.Pointer
}
// rtype is the common implementation of most values.
// It is embedded in other struct types.
//
// rtype must be kept in sync with ../runtime/type.go:/^type._type.
type rtype struct {
	size       uintptr
	ptrdata    uintptr // number of bytes in the type that can contain pointers
	hash       uint32  // hash of type; avoids computation in hash tables
	tflag      tflag   // extra type information flags
	align      uint8   // alignment of variable with this type
	fieldAlign uint8   // alignment of struct field with this type
	kind       uint8   // enumeration for C
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal     func(unsafe.Pointer, unsafe.Pointer) bool
	gcdata    *byte   // garbage collection data
	str       nameOff // string form
	ptrToThis typeOff // type for pointer to this type, may be zero
}
```

runtime包下:
```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```


非空接口:
reflect包下:
```go

// nonEmptyInterface is the header for an interface value with methods.
type nonEmptyInterface struct {
	// see ../runtime/iface.go:/Itab
	itab *struct {
		ityp *rtype // static interface type
		typ  *rtype // dynamic concrete type
		hash uint32 // copy of typ.hash
		_    [4]byte
		fun  [100000]unsafe.Pointer // method table
	}
	word unsafe.Pointer
}
```

runtime包下:

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
### 1. reflect.Type

`TypeOf`方法:
- 将控接口i变成`emptyInterface`类型,然后将其赋值给eface

注意`eface`在runtime包中
```go

```

`reflect`包中`emptyInterface`和`runtime.eface`是一样的
```go

// TypeOf returns the reflection Type that represents the dynamic type of i.
// If i is a nil interface value, TypeOf returns nil.
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}
```
`reflect.TypeOf`方法传入的值要是体现地址特性，我们一般直接使用传入变量（变量一般指的是copy值，但是编译器在使用到该方法时会有一个**隐式的复制值**，传入到`TypeOf`上）

```go
Package tt
type T1 struct{
    Name string
}
func (t T1) A(){
    println("A")
}
func (t T1) B(){
    println("B")
}


```

```go
func main(){
    a:=tt.T1{Name:"mhh"}
    //这里使用
    t:=reflect.TypeOf(a)
    println(t.Name(),t.NumMethod())
}
```

### 2. reflect.Value
上面主要是用反射读取信息，

使用反射修改信息用到`reflect.Value`:
- `typ`用作存储 反射变量的类型元数据 指针
- `ptr`存储数据的地址
- `flag`存储反射值的一些描述信息(是不是指针，是不是method，只读等等)
```go
type Value struct {
	// typ holds the type of the value represented by a Value.
	typ *rtype

	// Pointer-valued data or, if flagIndir is set, pointer to data.
	// Valid when either flagIndir is set or typ.pointers() is true.
	ptr unsafe.Pointer

	// flag holds metadata about the value.
	// The lowest bits are flag bits:
	//	- flagStickyRO: obtained via unexported not embedded field, so read-only
	//	- flagEmbedRO: obtained via unexported embedded field, so read-only
	//	- flagIndir: val holds a pointer to the data
	//	- flagAddr: v.CanAddr is true (implies flagIndir)
	//	- flagMethod: v is a method value.
	// The next five bits give the Kind of the value.
	// This repeats typ.Kind() except for method values.
	// The remaining 23+ bits give a method number for method values.
	// If flag.kind() != Func, code can assume that flagMethod is unset.
	// If ifaceIndir(typ), code can assume that flagIndir is set.
	flag

	// A method value represents a curried method invocation
	// like r.Read for some receiver r. The typ+val+flag bits describe
	// the receiver r, but the flag's Kind bits say Func (methods are
	// functions), and the top bits of the flag give the method number
	// in r's type's method table.
}

```
- 通常通过`reflect.ValueOf`拿到这个变量的value,注意传入的是空接口(同`reflect.TypeOf`处理方法一样)
- 另外还会显式的`escapes(i)`将指向的变量逃逸到堆上
```go
// ValueOf returns a new Value initialized to the concrete value
// stored in the interface i. ValueOf(nil) returns the zero Value.
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)

	return unpackEface(i)
}
```
关于逃逸:
同TypeOf函数一样，编译器会隐式创建一个copy值，但是因为逃逸，这个真正的数据会在堆上创建，而保留在栈上的是这个数据的地址，
但是因为这个是一个临时变量，毫无意义的修改不会影响到原来的`s`变量,所以会发生panic(use unaddressable value)

```go

s:="test"
t:=reflect.ValueOf(s)
t.SetString("test1") //panic
println(s)

```
更改后：
- 将s的地址传入
- 调用`Elem`方法，会将`t.ptr`指向的变量s包装成`reflect.Value`的返回值
```go
s:="test"
t:=reflect.ValueOf(&s)
t=t.Elem()
t.SetString("test1") //panic
println(s)
```
### 3. reflect.Kind



## Interface{}
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


## 额外

go会缓存itab的类型为一个k-v的表(底层有一个数组排列)，需要一个itab会首先从`itabTable`里面找

```go
const itabInitSize = 512
// Note: change the formula in the mallocgc call in itabAdd if you change these fields.
type itabTableType struct {
	size    uintptr             // length of entries array. Always a power of 2.
	count   uintptr             // current number of filled entries.
	entries [itabInitSize]*itab // really [size] large
}
```

如果能够找得到，就直接使用里面的itab值，否则会生成一个新的itab,存入`itabTableType.entries`字段中

key的hash 计算:
- 用接口类型的hash值XOR 动态类型的类型hash值

```go
func itabHashFunc(inter *interfacetype, typ *_type) uintptr {
	// compiler has provided some good hash codes for us.
	return uintptr(inter.typ.hash ^ typ.hash)
}
```