---
layout: post
title: Git Diff(渐知)--Myer's Diff出现
subtitle:   "算法"
date:       2018-05-05 13:31:00
author:     "BCM"
header-img: "../../img/post-bg-unix-linux.jpg"
tags:
    - ACM学生时代
---

`原创文章转载请注明出处，谢谢`

---

<mark>我好想吐槽一下好多人国人写博客真的是只会抄袭，感觉照搬的时候都没有加点自己的理解；一千个读者就有一千个哈姆雷特，这句话在这个方面真的完全不适用，感觉没几个哈姆雷特；<mark>

## Myer's Diff算法
Myer's Diff算法首先出现在一篇1986年一篇论文中”An O(ND) Difference Algorithm and Its Variations”中，关于论文详细内容这里不做过多的介绍，有兴趣的同学可以看[文章链接](http://xmailserver.org/diff2.pdf)然后去细读一番；
写这篇文章的目的只是想说明如何理解Myer's Diff算法的思路；


## 算法思路
通过前一篇文章我们知道即使我们使用了**DP**的方式，最后的时间复杂度还是维持在**o(n²)**；而**Myer's Diff**的方法可以有效的再此降低我们的时间复杂度。

这里回顾一下之前用**BFS**求解**LCS**时候的图论定义：

```
假设存在A[i],B[j]两个字符串，i∈[0,N),j∈[0,M)；
定义map[i][j],i∈[0,N),j∈[0,M), map[i][j]的每个节点表示A长度为i时，B长度为j时的LCS；

假设当A[i]!=B[j]时： 
Rightward，map[i-1][j] -> map[i][j];等价于删除A[i]节点操作；map[i][j] = map[i-1][j] + 1；
Downward，map[i][j-1] -> map[i][j];等价于增加B[j]节点操作；map[i][j] = map[i][j-1] + 1；

假设当A[i]=B[j]时：
斜向移动，map[i-1][j-1] -> map[i][j];等价于什么都不改变; map[i][j] = map[i-1][j-1]；

```
所以我们本质就是求**Map[n][m]**的最小值；同时先**Rightward**，再**Downward**（先删除，后新增）；

**Myer's Diff**算法使用的**图论**定义和**BFS**是一样的，只不过**BFS**的思路是标准的搜索思路我们通过求解每一步的最优解，然后直到运行到最后一个节点的时候才算出我们的最优解；


```
我们还是以论文中的两个字符串为例；
string1 = CBABAC;
string2 = ABCABBA;

         A        B       C        A        B         B        A
   o(0,0)--o(1,0)--o(2,0)--o(3,0)--o(4,0)--o(5,0)--o(6,0)--o(7,0)
   |       |       | \     |       |       |       |       |
C  |       |       |  \    |       |       |       |       |
   |       |       |   \   |       |       |       |       |
   |       |       |    \  |       |       |       |       |
   o(0,1)--o(1,1)--o(2,1)--o(3,1)--o(4,1)--o(5,1)--o(6,1)--o(7,1)
   |       | \     |       |       | \     | \     |       |
B  |       |  \    |       |       |  \    |  \    |       |
   |       |   \   |       |       |   \   |   \   |       |
   |       |     \ |       |       |     \ |     \ |       |
   o(0,2)--o(1,2)--o(2,2)--o(3,2)--o(4,2)--o(5,2)--o(6,2)--o(7,2)
   | \     |       |       | \     |       |       | \     |
A  |  \    |       |       |  \    |       |       |  \    |
   |   \   |       |       |   \   |       |       |   \   |
   |     \ |       |       |     \ |       |       |     \ |
   o(0,3)--o(1,3)--o(2,3)--o(3,3)--o(4,3)--o(5,3)--o(6,3)--o(7,3)
   |       | \     |       |       | \     | \     |       |
B  |       |  \    |       |       |  \    |  \    |       |
   |       |   \   |       |       |   \   |   \   |       |
   |       |     \ |       |       |     \ |    \  |       |
   o(0,4)--o(1,4)--o(2,4)--o(3,4)--o(4,4)--o(5,4)--o(6,4)--o(7,4)
   | \     |       |       | \     |       |       | \     |
A  |  \    |       |       |  \    |       |       |  \    |
   |   \   |       |       |   \   |       |       |   \   |
   |     \ |       |       |    \  |       |       |     \ |
   o(0,5)--o(1,5)--o(2,5)--o(3,5)--o(4,5)--o(5,5)--o(6,5)--o(7,5)
   |       |       | \     |       |       |       |       |
C  |       |       |  \    |       |       |       |       |
   |       |       |    \  |       |       |       |       |
   |       |       |     \ |       |       |       |       |
   o(0,6)--o(1,6)--o(2,6)--o(3,6)--o(4,6)--o(5,6)--o(6,6)--o(7,6)


```
想想我们能否把每一步的最优解坐标都先计算出来，然后通过不断增加步数的方式，直至找到某个步数正好等于最后的节点，这样我们就可以直接得到最优解；

首先我们需要保证数学上这个思路的时间复杂度是要优于**DP**，**DP**的时间复杂度是**O(NM)**，空间复杂度是**O(NM)**；所以我们必须要优于那这个算法才有探究的意义？


## 算法论证

> 我们定义**Node(x,y)**, **k = x - y**, **移动次数为d**;
> 
> if Node(x,y) to Rightward, Node(x+1,y), k - 1 = x - y;
> 
> if Node(x,y) to Downward, Node(x,y+1), k + 1 = x - y;
> 
> if **d = 1**; (0,0) => (1,0), **k = 1**;  (0,0) => (0,1), **k = -1**;
>
> if **d = 2**; (1,0) => (3,1), **k = 2**;  (1,0) => (2,2), **k = 0**;
>
> if **d = 2**; (0,1) => (2,2), **k = 0**;  (0,1) => (2,4), **k = -2**;
> 
> if **d = 3**; (3,1) => (5,2), **k = 3**;  (3,1) => (5,4), **k = 1**;
> 
> if **d = 3**; (2,2) => (5,4), **k = 1**;  (2,2) => (2,3), **k = -1**;
> 
> if **d = 3**; (2,4) => (4,5), **k = -1**; (2,4) => (3,6), **k = -3**;
> 
> .......
>
> **if d = 1, k = {-1,1}**;
> 
> **if d = 2, k = {-2, 0 , 2}**;
> 
> **if d = 3, k = {-3, -1, 1, 3}**;
> 

通过观察我们可以发现移动次数d和k对应的关系是**k∈[-d, -d + 2 ... d - 2, d]**;

所以就可以得到如下的对应关系：

```
for (int d = 0; d <= n + m; d++)
 for (int k = -d; k <= d; d += 2) {
	......
 }

最差的时间复杂度应该是:
o((N+M)D) = o(1 + 2 + ...+ (N + M + 1)) 
	  = o((N + M + 2) * (N + M) / 2) 
	  ≈ o((N + M)² / 2) 
	  ≈ o(N²) 

```

这与之前我们使用**DP**的时间复杂度是一样的；但问题就在于**DP**的时间复杂度是固定的**o(n²)**,而**o(n²)**只是**Myer's Diff**算法最差情况下的时间复杂度，也就是在实际的情况中**o((N+M)D)**要优于**o(N*M)**;


那么接下来就是考虑就是循环内部的算法判断;

首先我们需要通过一个空间记录所有k值对应的x最优解，令**V[]∈[-(n + m), (n + m)]**;

之后还需要知道哪些情况**Node**需要**Rightward**，哪些情况**Node**需要**Downward**；

```
if k = -d, 此时前一个Node一定是Downward, 即(x,y+1), x = V[k - 1];

if k = d, 此时前一个Node一定是Rightward, 即(x+1,y), x = V[k + 1] + 1;

if k∈(-d,d), 这种情况下我们需要考虑V[k - 1]和V[k + 1]的大小问题，

当V[k - 1] < V[k + 1]时，情况如下：
|-----------------------    
|      \  \  \ 
|       \  \  \
|        \  \  \
|         \  \  \
|          \  \  o(V[k + 1]) 
|           \  \  \
|  (V[k - 1])o  \  \
|             \  \  \
|              \  \V[k]
-----------------------

所以此时前一个Node一定是Downward, 即(x,y+1), x = V[k - 1];

当V[k - 1] > V[k + 1]时，情况如下：
|-----------------------    
|    \  \  o(V[k + 1]) 
|     \  \  \
|      \  \  \
|       \  \  \
|        \  \  \
|         \  \  \
|          \  \  \
|           \  \  \
|  (V[k - 1])o  \V[k]
-----------------------

所以此时前一个Node一定是Rightward, 即(x+1,y),x = V[k + 1] + 1;

当V[k - 1] = V[k + 1]时，情况如下：
|-----------------------    
|    \  \  \
|     \  \  \
|      \  \  o(V[k + 1]) 
|       \  \  \
|        \  \  \
|         \  \  \
|          \  \  \
| (V[k - 1])o  \  \
|            \  \V[k]
-----------------------

所以此时前一个Node Rightward和Downward都可以，但是我们遵守先删除后添加的原则, 即(x+1,y),x = V[k + 1] + 1;

我们知道了当前对应的x坐标，由于知道k，那么也可以求出y = x - k;

得知x,y的坐标以后，我只需要用一个循环再计算(x,y) == (x+1,y+1)这种情况，直至(x,y) != (x+1,y+1),然后更新V[k]对应的x; 最后再判断(x,y)节点是否已经到达终点即可；

```  

所以大致的思路代码如下所示:

```
int longestCommonSubsequence(char * text1, char * text2) {
    int sum = minDistance(text1, text2);
    return (strlen(text1) + strlen(text2) - sum) / 2;
}

int minDistance(char * text1, char * text2) {
    int map[10000];
    unsigned long allLength = (strlen(text1) + strlen(text2)) * 2;
    memset(map, 0, sizeof(map));
    
    for (int d = 0; d <= strlen(text1) + strlen(text2); d++)
        for (int k = -d; k <= d; k += 2) {
            int x, y;
            if (k == -d || (k != d && map[k - 1 + allLength] < map[k + 1 + allLength])) {
                x = map[k + allLength + 1];
            }  else {
                x = map[k + allLength - 1] + 1;
            }
            
            y = x - k;
            
            while (x < strlen(text1) && y < strlen(text2) && text1[x] == text2[y]) {
                x++;
                y++;
            }
            
            map[k + allLength] = x;
            if (x >= strlen(text1) && y >= strlen(text2)) {
                return d;
            }
        }
    return 0;
}
```
## 总结

到这里为止**Myer's Diff**算法的思路就已经说明完毕了，我们回过头来再看看时间都复杂度；即使增加**(x,y) == (x+1,y+1)**这种情况的内部循环，**Myer's Diff**的时间复杂度还是优于**DP**;

<mark>不过git diff最后使用的真实版本并不是上面这个，作者在上面的基础上又进行了思路的改进，主要原因就是上面这个方法要记录diff的路径空间复杂度会很高，这一点后面的文章在进行介绍；</mark>


## 引用

**https://imcuttle.github.io/o(nd)-difference-algorithm(译)**

**http://xmailserver.org/diff2.pdf(英)**

**https://blog.robertelder.org/diff-algorithm/**

**https://blog.jcoglan.com/2017/02/15/the-myers-diff-algorithm-part-2/**










