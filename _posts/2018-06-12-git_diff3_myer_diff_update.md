---
layout: post
title: Git Diff(优化)--Myer's Diff的改进
subtitle:   "算法"
date:       2018-06-12 09:31:00
author:     "BCM"
header-img: "../../img/post-bg-unix-linux.jpg"
tags:
    - ACM学生时代
---

`原创文章转载请注明出处，谢谢`

---

## 改进
前面介绍了关于**Myer Diff**的标准算法，论文的后半部分介绍关于这个算法的升级版本；升级版本主要是优化时间和空间复杂度，**Git Diff**真正使用的其实这个优化算法；说实话关于时间和空间复杂度的优化具体数值我目前并不是很理解，所以后续还会补充这块内容；这篇文章目前只会介绍这个优化算法的思路；


## 算法思路
首先回顾一下我们之前基础的**Myer Diff**算法，**其实就是从图的左上角(0,0)，开始向右下角不断搜索，通过不断叠加步数Depth，直至走到右下角(M,N)，找到一条路径**；

相反也就是说如果我们**从右下角(M,N)点，开始向左上角不断搜索，通过不断叠加步数Depth，直至走到左上角(0,0)，也能找到一条路径；**

我们还是以之前的字符串**CBABAC**和**ABCABBA**比较为例；**如果按照从左上角往右下角的方式，那么得到的LCS路径应该是CABA；如果从右下角往左上角的方式，那么得到的LCS路径应该是BABA**，会有这种结果的原因就是可能存在多个LCS，由于之前我们遵守先删除再增加的原则，所以搜索方向的不同就会造成这个结果，具体见下图；

   <p align="center">
<img src="../../../../img/technology/2018-06-12/myer_diff_02.png" alt="change_pic" title="change_pic"/>
</p>

论文中有一个结论：

```
如果存在一条从(0,0)到(M,N)的D-path，当且仅当存在一条从(0,0)到(x,y)的D/2-path, 及一条比(u,v)到(M,N)的
D/2-path, (x,y),(u,v)需满足以下条件：
    (可行性) u+v ≥ D/2 and x+y ≤ N+M-D/2 and
    (重叠性) x-y = u -v and x ≥ u
并且以上的两条D/2-path均需要是D-path的一部分。
```

我们结合上面这个结论进行分析一下，**如果我们分别从左上角和右下角同时开启搜索，那么一定存在一个区间使两者会互相重叠在一起，文章中将这一段重叠的部分称作Snake；这样我们就可以将图分成三部分：Left，Snake，Right，Snake就是两个字段的相同部分，而Left和Right两部分又可以以相同的方式进行拆分，再继续寻找各自的Snake，知道递归结束为止；**

以上其实就是这个改版的算法主要思想了，只能说看起来简单，但相对于原来版本的Myer Diff算法而言要处理的情况要很多；所以接下来的时间主要说明我用这个思想实现的LCS版本代码，细节我通过注释来进行解释；

## 算法实现

```
class Solution {
public:
    int longestCommonSubsequence(string text1, string text2) {
        int v1[1000]; // 临时过题固定大小
        int v2[1000]; // 临时过题固定大小

        int max_d = (int)(text1.length() + text2.length() + 1) / 2; // 搜索路径长度从中间区分两半

        int length = 2 * max_d;

        int v_offset = max_d;

        int delta = (int)(text1.length() - text2.length()); // 用来计算两个不同方向k的关系

        bool front = (delta % 2)? true : false; // 判断路径奇偶差

        memset(v1, -1, sizeof(v1));
        memset(v2, -1, sizeof(v2));

        v1[v_offset + 1] = 0;
        v2[v_offset + 1] = 0;

        for (int d = 0; d < max_d; d++) {

            for (int k1 = -d; k1 <= d; k1 += 2) {
                int k1_offset = k1 + v_offset;
                int x1, y1;
                if (k1 == -d || (k1 != d && v1[k1_offset - 1] < v1[k1_offset + 1])) {
                    x1 = v1[k1_offset + 1];
                } else {
                    x1 = v1[k1_offset - 1] + 1;
                }

                y1 = x1 - k1;

                while(x1 < text1.length() && y1 < text2.length() && text1[x1] == text2[y1]) {
                    x1 ++;
                    y1 ++;
                }

                v1[k1_offset] = x1;
                
                // 如果是奇数，说明Snake应该从left方向出现
                if (front) {
                    int k2_offset = v_offset + delta - k1; // k1通过delta来计算k2
                    if (k2_offset >= 0 && k2_offset < length && v2[k2_offset] != -1) {
                        int x2 = (int)text1.length() - v2[k2_offset];
                        // 判断两条路径是否重叠
                        if (x1 >= x2) {
                            return diff_bisectSplitOfOldString(text1, text2, x1, y1);
                        }
                    }
                }
            }

            for (int k2 = -d; k2 <= d; k2 += 2) {
                int k2_offset = k2 + v_offset;
                int x2, y2;
                if (k2 == -d || (k2 != d && v2[k2_offset - 1] < v2[k2_offset + 1])) {
                    x2 = v2[k2_offset + 1];
                } else {
                    x2 = v2[k2_offset - 1] + 1;
                }

                y2 = x2 - k2;

                while(x2 < text2.length() && y2 < text2.length() && text1[text1.length() - x2 - 1] == text2[text2.length() - y2 - 1]) {
                    x2 ++;
                    y2 ++;
                }

                v2[k2_offset] = x2;
                
                // 如果是偶数，说明Snake应该从right方向出现
                if (!front) {
                    int k1_offset = v_offset + delta - k2; // k2通过delta来计算k1
                    if (k1_offset >= 0 && k1_offset < length && v1[k1_offset] != -1) {
                        int x1 = v1[k1_offset];
                        int y1 = v_offset + x1 - k1_offset;
                        x2 = (int)text1.length() - x2;
                        // 判断两条路径是否重叠
                        if (x1 >= x2) {
                            return diff_bisectSplitOfOldString(text1, text2, x1, y1);
                        }
                    }
                }
            }
        }

        return 0;
    }

     int diff_bisectSplitOfOldString(string text1, string text2, int x, int y) {
         
         // 拆分递归的left和right
         string text1a = text1.substr(0, x);
         string text2a = text2.substr(0, y);
         string text1b = text1.substr(x, text1.length() - x);
         string text2b = text2.substr(y, text2.length() - y);
         
         int left = diff_mainOfString(text1a, text2a);
         int right = diff_mainOfString(text1b, text2b);

        return left + right;
     }

    int diff_mainOfString(string text1, string text2) {
        if (strcmp(text1.c_str(), text2.c_str()) == 0) {
            return (int)text1.length();
        }

        int prefixNum = diff_commonPrefix(text1, text2);
        if (prefixNum > 0) {
            text1 = text1.substr(prefixNum, text1.length() - prefixNum);
            text2 = text2.substr(prefixNum, text2.length() - prefixNum);
        }

        int suffixNum = diff_commonSuffix(text1, text2);
        if (suffixNum > 0) {
            text1 = text1.substr(0, text1.length() - suffixNum);
            text2 = text2.substr(0, text2.length() - suffixNum);
        }

        return prefixNum + suffixNum + diff_compute(text1, text2);
    }
    
    // 计算相同的头部
    int diff_commonPrefix(string text1, string text2) {
        int n = (int)min(text1.length(), text2.length());
        for (int i = 0; i < n; i++) {
            if (text1[i] != text2[i]) {
                return i;
            }
        }
        return n;
    }

    // 计算相同的尾部
    int diff_commonSuffix(string text1, string text2) {
        int text1_length = (int)text1.length();
        int text2_length = (int)text2.length();
        const int n = min(text1_length, text2_length);
        for (int i = 1; i <= n; i++) {
            if (text1[text1_length - i] != text2[text2_length - i]) {
                return i - 1;
            }
        }
        return n;
    }

    int diff_compute(string text1, string text2) {
        if (text1.length() == 0) {
            return 0;
        }

        if (text2.length() == 0) {
            return 0;
        }

        string longtext = text1.length() > text2.length() ? text1 : text2;
        string shorttext = text1.length() > text2.length() ? text2 : text1;
        string::size_type idx = longtext.find(shorttext);

        if (idx != string::npos) {
            return (int)shorttext.length();
        }

        if (shorttext.length() == 1) {
            return 0;
        }

        return longestCommonSubsequence(text1, text2);
    }
};
```

## 总结
当然Github上有一个各个语言完整版的代码实现；它是完美实现了一个Diff算法，用的就是改进版Myer Diff算法的思想；地址是<mark>https://github.com/google/diff-match-patch</mark>

   <p align="center">
<img src="../../../../img/technology/2018-06-12/diff_match_patch.png" alt="change_pic" title="change_pic"/>
</p>



