# 剑指 Offer 35. 复杂链表的复制

[力扣题目链接](https://leetcode-cn.com/problems/fu-za-lian-biao-de-fu-zhi-lcof/)

## 思路

> 哈希表

步骤：

* 利用map存储每一个节点的地址。
* 然后遍历每一个节点，将节点的next与random的指向对应的map节点
* 返回头节点

## 代码

```go
/**
 * Definition for a Node.
 * type Node struct {
 *     Val int
 *     Next *Node
 *     Random *Node
 * }
 */

func copyRandomList(head *Node) *Node {
    if head==nil{
        return head
    }
    cur:=head
    copy:=map[*Node]*Node{}
    //map存储每一个节点
    for cur!=nil{
        //创建节点
        node:=new(Node)
        node.Val=cur.Val
        copy[cur]=node
        cur=cur.Next
    }
    cur=head
    //遍历链表
    for cur!=nil{
        //将每一个节点的指向对应的map节点
        copy[cur].Next=copy[cur.Next]
        copy[cur].Random=copy[cur.Random]
        cur=cur.Next
    }
    //返回头节点
    return copy[head]
}
```

