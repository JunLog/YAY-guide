# 剑指 Offer 09. 用两个栈实现队列

[力扣题目链接](https://leetcode-cn.com/problems/yong-liang-ge-zhan-shi-xian-dui-lie-lcof/)

## 思路

栈：后进先出

队列：先进先出

一个栈当作队尾，一个栈作为队首。

当有元素进入队列时候，先进入队尾中。当需要弹出队首的元素，那么将队尾中的元素都放到队首中，那么弹出来的元素就是先进的元素。

当你模拟这个过程时候，你会发现两个栈的出口相反，栈底放一起，就形成了一个队列。

步骤：

* 加入队尾 ：将元素入栈
* 弹出队首 ：有三种情况
  * 栈为空，那么需要队尾的元素进入队首里面，然后再弹出
  * 如果还没有元素，那么返回-1
  * 如果有元素，弹出元素

## 代码

```go
type CQueue struct {
    stack1 []int //队尾的栈
    stack2 []int //队首的栈
}


func Constructor() CQueue {
    return CQueue{[]int{},[]int{}}
}


func (this *CQueue) AppendTail(value int)  {
    this.stack1=append( this.stack1,value) //元素进入队列，先进入队尾的栈里面
}


func (this *CQueue) DeleteHead() int {
    if len(this.stack2)==0{ //如果队首为空，那么就将队尾的元素入栈到队首中
        for len(this.stack1)!=0{ 
            x:=this.stack1[len(this.stack1)-1] //先进元素
            this.stack2=append(this.stack2,x) //入栈队首
            this.stack1=this.stack1[:len(this.stack1)-1]
        }
    }
    //代表队列中没有元素
    if len(this.stack2)==0{
        return -1
    }
    //t
    x:=this.stack2[len(this.stack2)-1]
    this.stack2=this.stack2[:len(this.stack2)-1]
    return x
}


/**
 * Your CQueue object will be instantiated and called as such:
 * obj := Constructor();
 * obj.AppendTail(value);
 * param_2 := obj.DeleteHead();
 */
```

