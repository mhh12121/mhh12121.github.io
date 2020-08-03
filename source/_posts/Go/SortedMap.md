---
title: Something should be noted in golang's map
date: 2019-06-19 10:36
tags: golang
---

There is **SortedMap** or **LinkedHashMap** in Java
Both can ensure the inserted order of elements, but f ** k myself, I found that Go **only supports basic HashMap**
which means you have to do SortedMap and LinkedHashMap by yourself.

                                    ----From a damn interview test
<!--more-->
I met a problem in an interview test:

Given a file containing URLs on every line,you must record all of their occurence times, then save it into a new file
format like:

    url1  2times
    url2  3times
    url3  4times

But it should be noticed that all urls in new file have the same sequence as those in source files;


其实你要解决三个问题：
1. url频次统计（map）
2. 要按顺序放入新文件？？？//todo
3. 文件太大怎么办（拆乘多个文件，进行映射，再统计，但这里又不知道怎么保证顺序？？！！难倒只有手动实现LinkedHashMap<del>红黑树</del>不一定是红黑树，但是LinkedHashMap是一定要实现的[这里跳到LinkedHashMap](LinkedHashMap实现)）



Go blog ![map in action](https://blog.golang.org/go-maps-in-action) says:

>>> When iterating over a map with a range loop, the iteration order is not specified and is not guaranteed to be the same from one iteration to the next. Since the release of Go 1.0, the runtime has randomized map iteration order.

### SortedMap

Someone has already implemented it 
From Go blog  ![map in action](https://blog.golang.org/go-maps-in-action)

```go
import "sort"

var m map[int]string
var keys []int
for k := range m {//extract all the damn key out
    keys = append(keys, k)
}
sort.Ints(keys)//sort them
for _, k := range keys {//then you can get a sorted map
    fmt.Println("Key:", k, "Value:", m[k])
}
```
If your **key** is not an int or string or you wanna sort it by yourself:

```go
//we assume a struct is like below:
type Obj struct {
	Title    string
	Sequence int
	Size     int
}
//
type By func(o1, o2 *Obj) bool

//ObjSorter joins a By function and a slice of Objs to be sorted.
type ObjSorter struct {//to wrap the obj and by function into this struct
	objs []Obj
	by   func(o1, o2 *Obj) bool
}

func (by By) Sort(objs []Obj) {
	os := &ObjSorter{
		objs: objs,
		by:   by,
	}
	sort.Sort(os)
}

func (s *ObjSorter) Less(i, j int) bool {
	return s.by(&s.objs[i], &s.objs[j])
}
func (s *ObjSorter) Swap(i, j int) {
	s.objs[i], s.objs[j] = s.objs[j], s.objs[i]
}
func (s *ObjSorter) Len() int {
	return len(s.objs)
}

```
Then to use it:

```go
func SortedMap(m map[Obj]int) {
	sortByTitle := func(o1, o2 *Obj) bool { //sort by Ttitle field
		return o1.Title < o2.Title
	}
	// sortBySequence := func(o1, o2 *Obj) bool { //sortBy Sequence field
	// 	return o1.Sequence < o2.Sequence
	// }
	tempObjs := make([]Obj, 0)
	for k := range m {
		tempObjs = append(tempObjs, k)
	}

	By(sortByTitle).Sort(tempObjs)//Sorted this
	for _, k := range tempObjs {//operate this 
		fmt.Printf("key:%v,value:%v \n", k, m[k])
	}

}

```
To test:

```go
func main(){
    obj1 := &Obj{Title: "4"}
	obj2 := &Obj{Title: "2"}
	obj3 := &Obj{Title: "1"}
	objs := make([]*Obj, 0)
	objs = append(objs, obj1, obj2, obj3)
	m := make(map[Obj]int)
	m[*obj1] = 1
	m[*obj3] = 1
	m[*obj2] = 1
	SortedMap(m)
}

```
Result:

```
key:{1 0 0},value:1 
key:{2 0 0},value:1 
key:{4 0 0},value:1 
```

Above example can be found in ![Golang slice doc examples](https://golang.org/pkg/sort/)
傻了，写到这里才发现，sortedMap有个鸡用，不符合这道题的要求啊！！！

### LinkedHashMap
To just ensure **inserted order**:
you can refer to ![Java LinkedHashMap]()//or arrayMap()

**But** 
here we won't achieve the LinkedHashMap, it's kind of troublesome...

We do a trade-off between time and space:


```go
//just create a slice to save sequence
func CalcUrls(){
    sequence:=make([]string,0)
    m:=make(map[string]int)
    //.....
    
    url:=fp.ReadLine()
    if _,exists:=m[url];exists{
        m[url]++
    }else{
        sequence=append(sequence,url)
    }
    //....
    for _,v:=range sequence{
        res:=fmt.Sprintf("url:%v,times:%v",v,m[v])
        Writer.write(res)//write into the new file
    }
    
    //....
}
```

**---------------------------------新增---------------------------------------------**
主要是红黑树是为了能便于删除和添加，而我遇到的面试题只考虑添加，所以没必要，
但是隔了一天，还是觉得不爽

自己尝试实现一下基于红黑树的map看看，就当做复习吧：

首先，红黑树也是AVL树，区别就只是颜色上的判断不同而已，所以AVL的rotate我这里略过，主要就是颜色的判断
要符合 
1. 每个节点是黑或者红
2. 根节点是黑色
3. 每个叶子节点是黑色（且为nil，添加的时候没用到，删除的时候会用）
4. 如果一个节点是红色的，它的子节点必须是黑色的
5. **从一个节点到该节点的所有叶子节点的路径上包含相同数目的黑节点** （这个一行代码可以搞定，但之后很可能会违反第4，所以要紧接着进行处理）

##### 节点插入时候，检查：
```go
//主要思路，先像AVL一样插入，之后再调整
func (brtree *BRTree) insertNode(pnode *BRNode) {
	tempRoot := brtree.root
	var temp *BRNode
	for tempRoot != nil { //已有根节点，就往下loop找到插入点
		temp = tempRoot //每次更新
		if pnode.val > tempRoot.val {
			tempRoot = tempRoot.right

		} else {
			tempRoot = tempRoot.left
		}
	}
	pnode.parent = temp //找到最后，把pnode的parent指向找到的最后一个点
	if temp != nil {    //不是根节点
		if temp.val < pnode.val { //判断放在左子树还是右子树
			temp.right = pnode
		} else {
			temp.left = pnode
		}
	} else { //根节点
		brtree.root = pnode
	}
	pnode.color = RED //直接把这个颜色先设置为红色！！！主要为了满足：
	// 从一个节点到该节点的所有叶子节点的路径上包含相同数目的黑节点（可以试想一下，最下方插入一个红色节点，因为目前红色节点的叶子节点肯定是黑色的nil，所以必会满足这个特性）
	brtree.insertCheck(pnode) //再检查，因为这个时候可能跟上面的第4点违背了，比如在红色的节点下面插一个红色的节点

}

```



```go
//插入时进行检查
func (brtree *BRTree) insertCheck(pnode *BRNode) {
	//1. 该节点没有父节点，即为root（与上面insertNode插入点的检查并不重复，因为接下来会进行递归，这个属于边界条件，所以这个检查root必不可少）
	if pnode.parent == nil {
		brtree.root = pnode
		brtree.root.color = BLACK
		return
	}
	//2.父节点是黑色直接添加(不用管)，红色接着处理
	if pnode.parent.color == RED {
		//2.1 父，叔叔节点不为空而且其颜色是红色 ,则将该父叔都改为黑色,将祖父改成红色
		/*
							  10,B
							 /     \
						    6,B	    15,B
						   /   \     /  \
						4,R   8,R  11,R  19,R
						/
					pnode,R

					==>父叔节点都变成黑色，祖父变成红色
							  10,B
							 /     \
						    6,R	    15,B
						   /   \     /  \
						4,B   8,B  11,R  19,R
			             /
					  pnode,R

					==>递归，从祖父（6）开始，因为此时父节点属于根节点，是黑色，所以完成

					           10,B
							 /     \
						    6,R	    15,B
						   /   \     /  \
						4,B   8,B  11,R  19,R
			             /
					  pnode,R
		*/
		if pnode.getUncle() != nil && pnode.getUncle().color == RED {
			pnode.parent.color = BLACK
			pnode.getUncle().color = BLACK
			pnode.getGrandParent().color = RED
			brtree.insertCheck(pnode.getGrandParent()) //对该树上的节点递归上去处理
		} else {
			//2.2 父节点为红色，叔叔节点不存在或者是黑色

			isLeft := pnode == pnode.parent.left
			isParentLeft := pnode.parent == pnode.getGrandParent().left

			if isLeft && isParentLeft {
				//2.2.1 是左子树，其父节点也是左子树
				/*       ...
						  /
				        8,B
					    /  \
					   7,R   /nil
					  /
					pnode,R
				*/
				//=》
				/*         /
					     7,B
				         /   \
				     pnode,R  8,R
				               \
				             /nil
				*/
				pnode.parent.color = BLACK
				pnode.getGrandParent().color = RED
				brtree.rotateRight(pnode.getGrandParent())

			} else if !isLeft && !isParentLeft {
				//2.2.2 不是左子树，父亲也不是左子树
				/*
						  7,B
					      /  \
					2,B/nil   8,R
					           \
					           10,pnode
				*/
				pnode.parent.color = BLACK
				pnode.getGrandParent().color = RED
				brtree.rotateLeft(pnode.getGrandParent())

			} else if isLeft && !isParentLeft {
				//2.2.3 是左子树，但父亲不是左子树
				/*
						8,B
					    /  \
					6,B/nil   10,R
					        /
					      pnode，R
				*/
				/*
						pnode,R
					    /  \
					6,B/nil  8,B
					            \
					            10,R
				*/
				brtree.rotateRight(pnode.parent)
				//实际上变成了2.2.2
				brtree.rotateLeft(pnode.parent) //pnode现在在原先pnode的父节点位置
				pnode.color = BLACK
				pnode.left.color = RED
				pnode.right.color = RED

			} else if !isLeft && isParentLeft {
				//2.2.4不是左子树，但父亲是左子树
				/*
							 \
					        8,B
						    /  \
						   7,R   10,B/nil
						      \
						      pnode，R
				*/
				//==>
				/*
						 8，B
					    /    \
					  pnode,R  10,B/nil
					  /
					7,R
				*/
				/*
						 pnode，R
					    /    \
					   8,B   10,B/nil
					  /
					7,R
				*/

				brtree.rotateLeft(pnode.parent)
				//实际上变成了2.2.1
				brtree.rotateRight(pnode.parent)
				pnode.color = BLACK
				pnode.left.color = RED
				pnode.right.color = RED

			}
		}

	}
}

```

##### 节点的删除
删除节点的时候也分几种情况
先把它当作普通二叉查找树处理：
1. 删除的节点没有子节点，直接删除
2. 被删除节点只有一个孩子，删除这个节点，并用其孩子代替这个节点
3. 被删除节点有两个孩子：找出后继节点，用后继节点内容复制给该节点，删除后继节点，再将后继节点删除;
就可以考虑后继节点了，被删除的节点（刚刚的后继节点）有两个非空子节点情况下，它的后继节点不可能两个子节点不是空的，
意思就是后继节点只有一个或零个儿子，就可以按前面的1,2进行处理


接着就是按照红黑树的颜色规则来改变和旋转了
//todo我实在写不动了
```go


```



那么把红黑树放在map上呢：

```go
//todo
```
#### LinkedHashMap实现

真正的实现，先讲一下：
1. 定义数据结构node，这个是链表上的每个节点，应该有prev,next,key，val这些
2. 定义整个map，应该是整个链表lmap，应该有指向头部，尾部的指针，map[key]node,length,

**---------------------------------end-----------------------------------------------**
The above code may use O(2n) space, but time complexity reduce to O(n)
**!!!Everything seems done!!!**

So what if we use a **large file?**



### Large File

Solution: Split the large file into several files, it depends on your MEM:
针对大文件，其实普通的思想就是"分而治之"，更多可以参考MapReduce

```go
func HashFile(url string) int{//pass a url
    seed:=131
    hash:=0
    for _,v:=range url{
        hash=hash*seed+v
    }
    return hash & 0x7FFFFFFF
}

func main(){
    //........
    //url:=fp.ReadLine()
    pos:=HashFile(url)
    file:=pos%10//we assume that it split into 10 files
    bufio.newWriter(file+".txt")
    Writer.Write(file+".txt")//write it into specific file
    //........
    urls:=make([]string,0)
    for i:=0;i<10;i++{
        urlPlusTimes:=LoopEachFile(files)//calculate each files' urls
        urls=append(urls,urlPlusTimes...)// Here comes to the question??? how to ensure sequence???
    }
}

```

### 最终解决办法
几天了，想不出来啊
1. 主要是拆开多个文件的话，如果每一条记录都是unqiue的，map也要O（n）复杂度，内存会爆

//todo




### Addition

#### unsafe
map in golang is **not thread-safe**;
So it involves with concurrency situation, you must add **sync.Mutex** or **sync.RWMutex**
About that, can refer to ![my previous blog](/Concurrency)
