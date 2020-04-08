---
layout: post
title: Git Diff(初识)--常用算法LCS和SES
subtitle:   "操作系统"
date:       2018-05-02 16:31:00
author:     "BCM"
header-img: "../../img/post-bg-unix-linux.jpg"
tags:
    - ACM学生时代
---

`原创文章转载请注明出处，谢谢`

---


## 前沿

**Git diff属于我们每天开发中都需要用到的命令，最近比较好奇Git diff的实现原理是什么？**

<mark>不同的选择对于Diff实现具有天差地别的效率区别，所以这篇文章打算分为三节，探讨我对于Diff算法的理解；</mark>

如果我们将diff问题进行分解，假使我们有AB两个字符串，那么diff问题就演变成比较两个字符串内容差异度的问题，即两者有多少字符相同或者有多少字符不同；

**即我们常说的求AB两个字符串的最长公共子序列(LCS)或者最短编辑距离(SES)**


## LCS和SES的关系

对于LCS和SES如果我们细心想一下就不难发现两者其实就是同一个问题；

> **∵ 存在字符串string1=ABCDE，string2=BCEGF；**
> 
> **∴ 则字符串的长度分别计作Length(string1)，Length(string2);**
> 
> **∴ 从LCS的角度来看，最长公共子序列就是BCE，长度计作Length(LCS);**
> 
> **∴ 从SES的角度来看，最短编辑路径就是A(-)D(-)G(+)F(+)，长度计作Length(SES);**
> 
> <mark>∴ Length(string1)+Length(string2)=Length(SES)+Length(LCS)*2；</mark>
> 

所以对于我们来说求解**LCS**的同时其实也是在求解**SES**，接下来后面我们主要会以**LCS**进行举例；

**备注：上述的推论都是基于我们对SES的定义来推断的；一般我们对于SES的操作应该是Add或者Delete；不过对于我们通常看到diff来说，更加常见的应该是Insert，Delete以及Replace；不过这种都是属于对于基础定义的升级，这点后面我们单独进行论述。**


## 思路方向

其实对于字符串比较这种问题，我们通常想到的方法应该就是通过**递归**将进行问题进行分解，也就是就是不断对子问题进行求解；

> **∵ 序列A长度为n，{A1,A2,A...AiA...An}；**
> 
> **∵ 序列B长度为m，{B1,B2,B...BjB...Bm}；**
> 
> **∵ 序列C长度为k，{C1,C2,C...Ck}；C为AB的LCS，则有L(C)=LCS(An,Bm)**
> 
> **∴ 如果An=Bm，那么L(C)=LCS(An-1,Bm-1)+1;**
> 
> **∴ 如果An!=Bm，那么L(C)=max(LCS(An,Bm-1), LCS(An-1,Bm));**
> 

简而言之就是我们通过分解不同情况下的子问题求解，从而遍历完所有的可能情况，从而获取最优解；
 
##### 具体的迭代过程如下：

```
char *text1;
char *text2;
int LCS(int i, int j) {
  if (i == strlen(text1) || j == strlen(text2)) 
    return 0;
  if (text1[i] == text2[j]) { 
    return LCS(i + 1, j + 1) + 1;
  } else {
    return max(LCS(i, j + 1), LCS(i + 1, j));
  }
}
```

归根到底这种**递归**思想和我们**DFS(深度优先搜索)**的原理是一样的，造成的结果就是虽然我们没有占用其他任何的空间，但是导致搜索的时间复杂度特别高，其中会出现大量的重复子问题；

如果把所有搜索路径当作**Tree**来看，结果就如下图所示；

```
以两个字符串ABCDE和BCEGF为例；

    A B C D E
--------------
B | a b c d e
C | f g h i j  
E | k l m n o
G | p q r s t
F | u v w x y

                      		      a
-------------------------------------------------------------------------- Deep=1
               f                  |                   b      
-------------------------------------------------------------------------- Deep=2            
      k        |         g         |        g         |         c
-------------------------------------------------------------------------- Deep=3
   p  |   l    |   l   |    h    |   l    |    h    |    h     |     d
-------------------------------------------------------------------------- Deep=4 
 u  q | q   m  |  q  m |  m   i  | q   m |   m    i |  m    i  |  i     e
-------------------------------------------------------------------------- Deep=5
.........................................................................  省略
```

**通过观察上面的搜索路径我们也能发现其中存在着大量的重复子问题，所以我们需要做的就是剪枝；**

<mark>备注：所谓剪枝无非就是省略那些不可能的分支路线，这种情况下就需要我们知道每一层Deep的最优解，但是基于DFS的特性（DFS是一个不断向下的搜索方法）我们无法得知；</mark>


## 转换思路

如果我们要进行剪枝，就需要将原来的**纵向搜索**问题改成**横向搜索**问题；即将**DFS**转变成为**BFS**进行处理；通过记录每个遍历过的节点，保证记录下当前的最优解；

```
typedef struct{
    int x, y;
} Node;

class Solution {
public:
    int LCS(string text1, string text2) {
        queue<Node> queue;
        int map[1000][1000];
        int flag[1000][1000];
        memset(map,0,sizeof(map));
        memset(flag,0,sizeof(flag));
        Node tmpNode;
        tmpNode.x = 1;
        tmpNode.y = 1;
        queue.push(tmpNode);
        while(!queue.empty()) {
            Node node = queue.front();
            queue.pop();
            if (flag[node.x][node.y] == 0) {
                flag[node.x][node.y] = 1;
                if (text1[node.x - 1] == text2[node.y - 1]) {
                    map[node.x][node.y] = map[node.x - 1][node.y - 1] + 1;
                } else {
                    map[node.x][node.y] = map[node.x - 1][node.y] > map[node.x][node.y - 1]? map[node.x - 1][node.y] : map[node.x][node.y - 1];
                }
                if (node.x + 1 <= text1.length()) {
                    Node moveXNode;
                    moveXNode.x = node.x + 1;
                    moveXNode.y = node.y;
                    queue.push(moveXNode);
                }
                
                if (node.y + 1 <= text2.length()) {
                    Node moveYNode;
                    moveYNode.x = node.x;
                    moveYNode.y = node.y + 1;
                    queue.push(moveYNode);
                }
            }
        }
        return map[text1.length()][text2.length()];
    }
};
```

到这里我们已经成功将**LCS**的字符串问题转换成为**图论**的问题；
相比之前的**递归**方法，我们的**BFS**成功将**时间复杂度**转换成为**空间复杂度**；


## DP状态转移

回过头来，再重新观察一下我们对**LCS**使用的**BFS**算法；

通过引入一个二维数组，用以保存我们每个节点的最优解，然后通过对每个节点进行flag标记，防止队列中相同的节点会重新进行计算，从而省略那些不必要的重复路径；

不过这里我们再进行思考一下：我们是否可以对BFS的空间复杂度再度进行优化呢；也就是去除用来flag标记当前节点是否已经遍历过的空间；

其实会造成这种情况的原因主要在于**BFS**的特性，即它的搜索每次都是以当前节点往四个方向（**LCS**中只有两个方向）的形式进行遍历，这就会造成相同节点被多次加入队列，所以才需要flag空间进行剪枝操作；

那么我们思考一下我们是否可以沿着同一个方向进行搜索，直到所有的节点都遍历完全，每个节点的结果都依赖于之前已经遍历过节点的最优解；

所以我们可以再对方案进行修改：

```
int LCS(char *text1, char *text2) {
    int map[1000][1000];
    int i = 0;
    int j = 0;
    
    if (strlen(text1) == 0 || strlen(text2) == 0) {
        return 0;
    }
    
    for (i = 1; i <= strlen(text1); i++)
        for (j = 1; j <= strlen(text2); j++) {
            if (text1[i - 1] == text2[j - 1]) {
                map[i][j] = map[i - 1][j - 1] + 1;
            } else {
                if (map[i - 1][j] > map[i][j - 1]) {
                    map[i][j] = map[i - 1][j];
                } else {
                    map[i][j] = map[i][j - 1];
                }
            }
        }

    return map[strlen(text1)][strlen(text2)];
}
```

于是流程就简化成了我们熟悉的**DP**方程；

> **dp[i,j]=0;** <mark>if i = 0 or j = 0;</mark>
> 
> **dp[i,j]=dp[i-1,j-1]+1;** <mark>if i > 0, j > 0 and Ai = Bj;</mark>
> 
> **dp[i,j]=max(dp[i,j-1],dp[i-1,j])**  <mark>if i > 0, j > 0 and Ai != Bj;</mark> 
>

这里其实还想说明的一点就是我们之前提到的SES的定义问题；传统的SES问题只有Add和Delete两个操作，那么对应的**DP**方程如下：

> **dp[i,j]=i;** <mark>if j = 0 and i != 0;</mark>
>  
> **dp[i,j]=j;** <mark>if i = 0 and j != 0;</mark>
> 
> **dp[i,j]=0;** <mark>if i = 0 and j = 0;</mark> 
>
> **dp[i,j]=dp[i-1][j-1];** <mark>if i > 0 and j > 0 and Ai = Bj;</mark>
> 
> **dp[i,j]=min(dp[i-1,j],dp[i,j-1]);** <mark>if i > 0 and j > 0 and Ai != Bj;</mark>
>

**我们之前说到的操作变更其实就是针对于Ai != Bj这种情况发生的，原来的Add和Delete会演变成Insert，Delete和Replace；其中原来在同一个位置进行的先Delete后Add的操作统一算作成了Replace操作，所以导致了操作计数的不同（这种情况下前面提到的LCS和SES的关系就会失效）；**

对于这种情况我们就需要在原来的**DP**方程基础上对这一情况就行再次分解：
>
> **dp[i,j]=min(dp[i-1,j-1],dp[i-1,j],dp[i,j-1]);** <mark>if i > 0 and j > 0 and Ai != Bj;</mark>
> 


## 总结
其实**递归**和**DP**是我们日常解决问题十分常用的思想，两者既有相似点也有不同点，我对于这两者的理解是：

* **相同点：两者都具有相同子问题进行求解，以及每个子问题都具有最优解；**
* **不同点：DP会记录每个子问题的最优解，对于同一个子问题不需要重新进行计算；而递归则需要不断对每个子问题进行求解；**