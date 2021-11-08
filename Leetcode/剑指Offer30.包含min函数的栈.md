# 剑指 Offer 30. 包含min函数的栈

[力扣题目链接](https://leetcode-cn.com/problems/bao-han-minhan-shu-de-zhan-lcof/)

## 思路

因为这个题目有要求`调用 min、push 及 pop 的时间复杂度都是 O(1)。` 所以你需要一个数据结构去维护最小值。选择肯定是**单调栈**。

需要一个单调栈起到维护一串元素中最小的值的辅助作用。

当元素入栈时候，你需要维护单调栈，判断是否能够形成一个递减的栈，如果可以那么就入栈。当弹出元素，如果弹出的元素和单调栈栈顶元素一致，单调栈的元素也要弹出。

步骤：

* 入栈
  * 判断辅助栈是否为空或者能形成递减，那么能的话也入辅助栈
* 出栈
  * 判断出栈元素是否与辅助栈的栈顶相等，那么相等也出栈

注意：这里的单调栈不是单调递减的。因为可以有重复元素，所以是递减的栈。

## 代码

```go
type MinStack struct {
    a []int//存储元素
    b []int //辅助栈 维护栈的最小值
}


/** initialize your data structure here. */
func Constructor() MinStack {
    return MinStack{
        []int{},
        []int{},
    }
}


func (this *MinStack) Push(x int)  {
    //当辅助栈为空或者能形成递减的，就入栈
    if len(this.b)==0||this.b[len(this.b)-1]>=x{
        this.b=append(this.b,x)
    }
    this.a=append(this.a,x)
}


func (this *MinStack) Pop()  {
    //当弹出元素与辅助栈栈顶元素相等
    if this.b[len(this.b)-1]==this.a[len(this.a)-1]{
        this.b=this.b[:len(this.b)-1]
    }
    this.a=this.a[:len(this.a)-1]
}


func (this *MinStack) Top() int {
    //栈顶元素
    return this.a[len(this.a)-1]
}


func (this *MinStack) Min() int {
    fmt.Println(this.b)
    //辅助栈的栈顶元素
    return this.b[len(this.b)-1]
}


/**
 * Your MinStack object will be instantiated and called as such:
 * obj := Constructor();
 * obj.Push(x);
 * obj.Pop();
 * param_3 := obj.Top();
 * param_4 := obj.Min();
 */
```

## 注意

>  如果辅助栈为空了，原始栈还有元素，那岂不是最小值拿不到了？

不可能会出现这个问题，因为辅助栈的第一个元素，就是原始栈的元素，除非原始栈也为空，否则辅助栈不会为空，会一直维护以第一个元素开头递减的栈。