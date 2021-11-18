# 剑指 Offer 48. 最长不含重复字符的子字符串

 [剑指 Offer 48. 最长不含重复字符的子字符串](https://leetcode-cn.com/problems/zui-chang-bu-han-zhong-fu-zi-fu-de-zi-zi-fu-chuan-lcof/)

## 思路

先讲讲我错误的思路：

> 一开始打算从以 i 作为起始位置，去找与他相同的字符，找到后，那么索引 i-j 的字符串的不含重复字符字串为 j-i+1。但是有一点是存在问题的：首位字符相同的字串可能还存在另一对相同字符。

其实最大的问题是如何处理首位字符相同的字串存在另一对相同字符，从中获取最长的长度。

递推公式：

dp[j] 是以索引 j 为结束的字串，i 是寻找与索引为 j 的字符相同的字符 s[i]。

j-i 代表找到与 dp[i] 相同字符作为子串。

* 当 i<0 时候，代表没有找到与索引为 j 字符相同的字符。那么 dp[j]=dp[j-1]+1
* 当 dp[j-1]<j-i 时候代表此时 s[i]，在子串 dp[j-1] 外，**最长的长度为上一个子串中最长的部分，加上当前字符。**所以 dp[j]=dp[j-1]+1
* 当 dp[j-1]>j-i 时候代表此时 s[i] 在字串 dp[j-1] 里面，**当前的字串不适合利用上一个子串长度，应该获取新的长度**所以 sp[j]=j-1。

> 注：情况1和情况2可以合并，当 i < 0 时，由于 dp[j - 1]≤j 恒成立，因而 dp[j - 1] < j - i 恒成立，
>

## 代码

```go
func lengthOfLongestSubstring(s string) int {
    if len(s)==0{
        return 0
    }
    dp:=make([]int,len(s))
    dp[0]=1
    ans:=1
    for i:=1;i<len(s);i++{
        j:=i-1
        for j>=0&&s[i]!=s[j]{
            j--
        }
        if dp[i-1]<i-j{
            dp[i]=dp[i-1]+1
        }else {
            dp[i]=i-j
        }
        ans=max(ans,dp[i])
    }
    fmt.Println(dp)
    return ans
}
func max(a, b int)int{
    if a>b{
        return a
    }
    return b
}
```

优化

```go
func lengthOfLongestSubstring(s string) int {
    tmp:=0
    ans:=0
    for j:=0;j<len(s);j++{
        i:=j-1
        for i>=0&&s[i]!=s[j]{
            i--
        }
        if tmp<j-i{
            tmp++
        }else {
            tmp=j-i
        }
        ans=max(ans,tmp)
    }
    return ans
}
func max(a,b int)int{
    if a>b{
        return a
    }
    return b
}
```

