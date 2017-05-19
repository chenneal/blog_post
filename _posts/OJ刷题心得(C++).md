---
title: OJ刷题心得(C++)
date: 2016-10-21 18:01:16

tags:
- ACM

categories:
- 算法

---

最近一直在刷题，分享一点用C++刷题的心得和一些个人规范，也方便自己查阅

<!-- more -->

## 实现包含好头文件和某些函数（经常用宏函数）（不断更新中）

本文并非面向群众向，非常个人向的内容，有些甚至基础的令人发指，请务必不要嘲笑 :)

下面是我自己的版本，以后会不断的补充：

``` C++
#include <iostream>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <string>
#include <climits>
#include <cctype>

#include <algorithm>
#include <vector>
#include <map>
#include <set>
#include <queue>
#include <string>

#define pb push_back
//#define FILE_JUDGE

using namespace std;

typedef long long ll;
typedef double D;
typedef vector<int> VI;
typedef map<int, int> MII;
typedef set<int> SI;

int main(int argc, char* argv[]) {
    #ifdef FILE_JUDGE
    freopen("file.in", "r", stdin);
    freopen("file.out", "w", stdout);
    #endif
    return 0;
}
```
