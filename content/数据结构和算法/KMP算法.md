```json
{  
    "date": "2021.05.22 14:03", 
    "tags": ["java","kmp"], 
    "description":"Knuth-Morris-Pratt 算法，简称KMP算法，是一个快速查找匹配串的算法，由Donald Knuth、James H. Morris 和 Vaughan Pratt三人于 1977 年联合发表。"
}
```

# 前缀表与next数组

next数组就可以是前缀表，但是很多实现都是把前缀表统一减一（右移一位，初始位置为-1）之后作为next数组。

# 时间复杂度

其中n为文本串长度，m为模式串长度，因为在匹配的过程中，根据前缀表不断调整匹配的位置，可以看出匹配的过程是O(n)，之前还要单独生成next数组，时间复杂度是O(m)。所以整个KMP算法的时间复杂度是O(n+m)的。

暴力的解法显而易见是O(n * m)，所以KMP在字符串匹配中极大的提高的搜索的效率。

以下文章统称haystack为文本串, needle为模式串。

# 构造next数组

我们定义一个函数getNext来构建next数组，函数参数为指向next数组的指针，和一个字符串。 代码如下：

```c++
void getNext(int* next, const string& s)
```

构造next数组其实就是计算模式串s，前缀表的过程。 主要有如下三步：

- 1.初始化
- 2.处理前后缀不相同的情况
- 3.处理前后缀相同的情况

接下来我们详解解释一下。

1.初始化：

定义两个指针i和j，j指向前缀终止位置（严格来说是终止位置减一的位置），i指向后缀终止位置（与j同理）。

然后还要对next数组进行初始化赋值，如下：

```c++
int j = -1;
next[0] = j;
```
j 为什么要初始化为 -1呢，因为之前说过 前缀表要统一减一的操作仅仅是其中的一种实现，我们这里选择j初始化为-1，下文我还会给出j不初始化为-1的实现代码。

next[i] 表示 i（包括i）之前最长相等的前后缀长度（其实就是j），所以初始化next[0] = j 。

2.处理前后缀不相同的情况

因为j初始化为-1，那么i就从1开始，进行s[i] 与 s[j+1]的比较。

所以遍历模式串s的循环下标i 要从 1开始，代码如下：
```c++
for(int i = 1; i < s.size(); i++) {
```
如果 s[i] 与 s[j+1]不相同，也就是遇到 前后缀末尾不相同的情况，就要向前回溯。 怎么回溯呢？ next[j]就是记录着j（包括j）之前的子串的相同前后缀的长度。

那么 s[i] 与 s[j+1] 不相同，就要找 j+1前一个元素在next数组里的值（就是next[j]）。 所以，处理前后缀不相同的情况代码如下：

```c++
while (j >= 0 && s[i] != s[j + 1]) { // 前后缀不相同了
    j = next[j]; // 向前回溯
}
```
3.处理前后缀相同的情况:

如果s[i] 与 s[j + 1] 相同，那么就同时向后移动i 和j 说明找到了相同的前后缀，同时还要将j（前缀的长度）赋给next[i], 因为next[i]要记录相同前后缀的长度。

代码如下：
```c++
if (s[i] == s[j + 1]) { // 找到相同的前后缀
    j++;
}
next[i] = j;
```
最后整体构建next数组的函数代码如下：
```c++
void getNext(int* next, const string& s){
    int j = -1;
    next[0] = j;
    for(int i = 1; i < s.size(); i++) { // 注意i从1开始
        while (j >= 0 && s[i] != s[j + 1]) { // 前后缀不相同了
            j = next[j]; // 向前回溯
        }
        if (s[i] == s[j + 1]) { // 找到相同的前后缀
            j++;
        }
        next[i] = j; // 将j（前缀的长度）赋给next[i]
    }
}
```

# 使用next数组来做匹配

在文本串s里 找是否出现过模式串t。 定义两个下标j 指向模式串起始位置，i指向文本串其实位置。 那么j初始值依然为-1，为什么呢？ 依然因为next数组里记录的起始位置为-1。

i就从0开始，遍历文本串，代码如下：

```c++
for (int i = 0; i < s.size(); i++) 
```
接下来就是 s[i] 与 t[j + 1] （因为j从-1开始的） 经行比较。如果 s[i] 与 t[j + 1] 不相同，j就要从next数组里寻找下一个匹配的位置。

代码如下：

```c++
while(j >= 0 && s[i] != t[j + 1]) {
    j = next[j];
}
```

如果 s[i] 与 t[j + 1] 相同，那么i 和 j 同时向后移动， 代码如下：
```c++
if (s[i] == t[j + 1]) {
    j++; // i的增加在for循环里
}
```
如何判断在文本串s里出现了模式串t呢，如果j指向了模式串t的末尾，那么就说明模式串t完全匹配文本串s里的某个子串了。

本题要在文本串字符串中找出模式串出现的第一个位置 (从0开始)，所以返回当前在文本串匹配模式串的位置i 减去 模式串的长度，就是文本串字符串中出现模式串的第一个位置。

代码如下：

```c++
if (j == (t.size() - 1) ) {
    return (i - t.size() + 1);
}

```

那么使用next数组，用模式串匹配文本串的整体代码如下：
```c++
int j = -1; // 因为next数组里记录的起始位置为-1
for (int i = 0; i < s.size(); i++) { // 注意i就从0开始
    while(j >= 0 && s[i] != t[j + 1]) { // 不匹配
         j = next[j]; // j 寻找之前匹配的位置
    }
    if (s[i] == t[j + 1]) { // 匹配，j和i同时向后移动
        j++; // i的增加在for循环里
    }
    if (j == (t.size() - 1) ) { // 文本串s里出现了模式串t
        return (i - t.size() + 1);
    }
}
```

Java版本

```java
// 方法一
class Solution {
    public void getNext(int[] next, String s){
        int j = -1;
        next[0] = j;
        for (int i = 1; i<s.length(); i++){
            while(j>=0 && s.charAt(i) != s.charAt(j+1)){
                j=next[j];
            }

            if(s.charAt(i)==s.charAt(j+1)){
                j++;
            }
            next[i] = j;
        }
    }
    public int strStr(String haystack, String needle) {
        if(needle.length()==0){
            return 0;
        }

        int[] next = new int[needle.length()];
        getNext(next, needle);
        int j = -1;
        for(int i = 0; i<haystack.length();i++){
            while(j>=0 && haystack.charAt(i) != needle.charAt(j+1)){
                j = next[j];
            }
            if(haystack.charAt(i)==needle.charAt(j+1)){
                j++;
            }
            if(j==needle.length()-1){
                return (i-needle.length()+1);
            }
        }

        return -1;
    }
}
```