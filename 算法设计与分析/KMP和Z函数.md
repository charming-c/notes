



# KMP 和 Z 函数（扩展 KMP）算法

## 1. KMP 算法

KMP 算法是一种字符串匹配算法，在 o(m + n) 时间内实现两个字付串的匹配。

### 最长公共前后缀 -- next 数组

快速构建next数组，是KMP算法的精髓所在，核心思想是“**P自己与自己做匹配**”。

- 定义 “k-前缀” 为一个字符串的前 k 个字符； “k-后缀” 为一个字符串的后 k 个字符。k 必须小于字符串长度。 
- next[x] 定义为： P[0]~P[x] 这一段字符串，使得**k-前缀恰等于k-后缀**的最大的k.

　　这个定义中，不知不觉地就包含了一个匹配——前缀和后缀相等。接下来，我们考虑采用递推的方式求出next数组。如果next[0], next[1], ... next[x-1]均已知，那么如何求出 next[x] 呢？

　　来分情况讨论。首先，已经知道了 next[x-1]（以下记为now），如果 P[x] 与 P[now] 一样，那最长相等前后缀的长度就可以扩展一位，很明显 next[x] = now + 1. 图示如下。



<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/v2-6d6a40331cd9e44bfccd27ac5a764618_1440w.webp" alt="next数组" style="zoom:50%;" />

刚刚解决了 P[x] = P[now] 的情况。那如果 P[x] 与 P[now] 不一样，又该怎么办？

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/v2-ce1d46a1e3603b07a13789b6ece6022f_1440w.webp" alt="img" style="zoom:50%;" />

如图。长度为 now 的子串 A 和子串 B 是 P[0]~P[x-1] 中最长的公共前后缀。可惜 A 右边的字符和 B 右边的那个字符不相等，next[x]不能改成 now+1 了。因此，我们应该**缩短这个now**，把它改成小一点的值，再来试试 P[x] 是否等于 P[now].
　　now该缩小到多少呢？显然，我们不想让now缩小太多。因此我们决定，在保持“P[0]~P[x-1]的now-前缀仍然等于 now-后缀”的前提下，让这个新的now尽可能大一点。 P[0]~P[x-1] 的公共前后缀，前缀一定落在串A里面、后缀一定落在串 B 里面。换句话讲：接下来now应该改成：使得 **A的k-前缀**等于**B的 k-后缀** 的最大的k.
　　您应该已经注意到了一个非常强的性质——**串A和串B是相同的**！B的后缀等于A的后缀！因此，使得A的k-前缀等于B的k-后缀的最大的k，其实就是串A的最长公共前后缀的长度 —— next[now-1]！

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/v2-c5ff4faaab9c3e13690deb86d8d17d71_1440w.webp" alt="img" style="zoom:50%;" />

来看上面的例子。当P[now]与P[x]不相等的时候，我们需要缩小now——把now变成next[now-1]，直到P[now]=P[x]为止。P[now]=P[x]时，就可以直接向右扩展了。代码实现如下：

```java
int now = 0;
int[] next = new int[m];
Arrays.fill(next, 0);
for (int i = 1; i < m; i++) {
  while(pre > 0 && pattern[i] != pattern[now]) {
    now = next[now - 1];
  }
  if(pattern[i] == pattern[now]) {
    now++;
  }
  next[i] = now;

}
```
完整的匹配出字符串的过程如下所示：

```java
int p = 0;
int cur = 0;
int ans = 0;
while (p < n - 1) {
  if (res[p] == pattern[cur]) {
    p++;
    cur++;
  }
  else if(cur > 0) {
    cur = next[cur - 1];
  }
  else p++;
  if(cur == m) {
    ans++;
    cur = next[cur - 1];
  }

}
```



## 2. 扩展 KMP 算法

