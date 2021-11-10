# 剑指 Offer 03. 数组中重复的数字

 [力扣题目链接](https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/)

## 思路

本题比较简单，方法有很多。例如：排序后查询重复数字，利用哈希表等等。介绍一个比较有意思的方法。

Krahets的图片

![Picture0.png](https://pic.leetcode-cn.com/1618146573-bOieFQ-Picture0.png)

如果没有重复数字，那么就有`nums[i]==i`,现在又重复数字，那么必然会再次出现`nums[i]==i`。

所以这里就有一个对应关系`nums[i] ->i` 但是这里并不使用map去实现，靠数组即可。

在索引为0对应的元素值nums[i]=2，好像并不对应，那么怎么找到呢？

0是索引，把当前元素值作为索引值去查找。那么一定会找到对应值。这是必然。

不断交换`nums[i]`和`i`索引对应元素值，找到就即可。

如果在找的途中，发现`nums[i]`与`i`对应的元素值好像是一致，那么代表出现了多余的对应关系。这就是你要找的重复元素值。

步骤：

* 遍历数组
  * 当`nums[i]==i` 说明找到一组对应关系，就跳过。
  * 当`nums[nums[i]]==nums[i]` 说明又出现一组对应关系，就返回重复元素。
  * 否则，就交换索引为`nums[i]`和`i`的值，直到找到对应关系。
* 遍历完成，未找到返回-1

## 代码

```go
func findRepeatNumber(nums []int) int {
    i:=0
    for i<len(nums){
        if nums[i]==i{
            i++
            continue
        }
        if nums[nums[i]]==nums[i]{
            return nums[i]
        }
        nums[i],nums[nums[i]]=nums[nums[i]],nums[i]
    }
    return -1
}
```

