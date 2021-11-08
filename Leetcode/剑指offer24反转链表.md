# 剑指 Offer 24. 反转链表

[力扣题目链接](https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof/)

## 思路

> 迭代

以`1->2->nil`举例

反转就是将当前节点指向自己的前一个，如果没有指向nil

步骤：

* 保存2的地址
* 将1的next指向上一个，因为这里1是头节点，所以指向nil
* 当前节点来到2处
* 保存上一个节点的地址

> 递归

当然了这题也可以用递归解决。

以`1->2->3->4->5->nil`举例

若`1->2->3->4<-5`4和5已经反转了，当前处于3

我们希望自己4的指向当前节点，所以就有`head.Next.Next=head`

同时当前节点的指向清除 `head.Next=nil`

当然了，上面的操作都是从后面开始的。所以你需要通过递归到达尾节点。

## 代码

迭代

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
    var tail *ListNode
    for head!=nil{
        node:=head.Next//保存下一个节点
        head.Next=tail //将当前节点指向上一个节点 如果是头节点指向nil
        tail=head //保存当前节点，
        head=node  //跳到下一个节点
    }
    return tail
}
```

递归

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
    if head==nil||head.Next==nil{
        return head
    }
    newhead:=reverseList(head.Next)//到达最后一个节点 
    head.Next.Next=head //下一个节点的指向当前节点
    head.Next=nil //删除当前节点的指向
    return newhead
}
```

