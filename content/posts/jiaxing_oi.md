+++
title = "2023嘉兴高三基础检测卷信息技术压轴题剖析"
date = "2023-10-03"

[taxonomies]
categories = ["zh-cn"]
tags = [
    "Algorithm",
]
+++

## 前言

这道题很有意思，考察了数据结构树，并且以一个很经典的问题——最短路径问题为背景，这在算法竞赛中是喜闻乐见的。相较于以往各种包浆的用链表解决实际问题的题目，这道题要清爽多了。

这篇文章不放完整原题干，打字太烦。:)



## (1)题

该题考查学生的逻辑思维能力。

`init`函数需要返回所有顶点的**相邻顶点**。这里的**相邻顶点**的**定义**可以根据**第15题图b**推断得到，即在一个顶点的**上下左右**的顶点为该顶点的相邻顶点，在对角位置的顶点不算相邻顶点。

源代码如下：

```python
def init(m, n):
    tot = (m+1)*(n+1) # 顶点总数
    lst = [[] for i in range(tot)]
    for i in range(tot):
        if i > m:
            lst[i].append(i-m-1)
        if i < (m+1)*n:
            lst[i].append(i+m+1)
        if i%(m+1) != 0:
            lst[i].append(i-1)
        if i%(m+1) != m:
            ___▲___
    return lst
```

不难发现，这里确定相邻节点需要判断边界。

`i > m`时，即`i`不在第一行，则必有一个顶点在它头上。

`i < (m+1)*n`时，即`i`不在最后一行，则必有一个顶点在它底下。

`i%(m+1) != 0`时，即`i`不在最左边，则必有一个顶点在它左边。

`i%(m+1) != m`时，即`i`不在最右边，则必有一个顶点在它右边。

故该空填：`lst[i].append(i+1)`。

## (2)题

该题考查对树的认识，以及细心程度:)

估计很多考生对着给出的树就填了个`3`。实际不是。

题目很心机地没有给全顶点`7`所有的相邻顶点，细心的你如果仔细观察，会发现顶点`7`还有一个相邻顶点：`6`。

<div align="left"><img src="/jiaxing-oi/track.png" width="40%" alt="第15题图a" title="第15题图a"></div>

因此到达顶点`11`的路径还有一条：4-5-6-7-11。

因此答案为：4。

## (3)，(4)题

先看(3)题。

```python
'''(3)'''
def print_path(x, path, length): # x为起点编号，length为Path中有效元素的个数。
    cnt = 0
    for i in range(length):
        if path[i][0] == x:
            cnt += 1
            s = "最短路径" + str(cnt) + ":"
            v = path[i]
            while ___▲___:
                s = s + str(v[0]) + ","
                v = path[v[2]]
            s = s + str(v[0]) + "。"
            print(s)
```

题目规定：

---

`path[i][0]`保存顶点的编号，`path[i][1]`保存当前顶点到终点的距离，`path[i][2]`保存下一顶点在`path`中的位置，其值为`-1`表示当前顶点为终点。

---

这里说的最不清楚的，就是`path[i][2]`。`path[i][2]`到底指向哪个顶点？显然，这是出题者的诡计，他想在这里拷打我们。

这个问题暂时不谈。

题干又有一句话：

---

可采用链表结构保存路径数据。

---

而`path[i][2]`保存的就是地址。再看(3)题的代码，长得像不像遍历链表？

遍历到什么时候截止？当然是遍历到终点截止咯。

那就是`path[i][2] == -1`的时候吧？

所以该空填：`v[2] != -1`。



接着看(4)题。

```python
'''(4)'''
m = 3 # 横向正方形数量
n = 2 # 纵向正方形数量
mtx = init(m, n)
x = int(input("请输入起点："))
y = int(input("请输入终点："))
path = [[] for i in range(999)]
passed = [False] * len(mtx) # 保存顶点是否已途经
___①___
dis = 0
head = 0
tail = 0
path[tail] = [y, 0, -1]
tail += 1
passed[y] = True
while not found:
    dis += 1
    pass_dis = [False] * len(mtx)
    tmp = tail
    for i in range(head, tail):
        v = path[i]
        for d in mtx[v[0]]:
            if not passed[d]:
                path[tail] = ___②___
                tail += 1
                pass_dis[d] = True
            if d == x:
                found = True
    head = tmp
    for i in range(len(mtx)): # 标记已途经的顶点
        if ___③___:
            passed[i] = True
# 输出结果           
print_path(x, path, tail)
```

①是最好解决的，显然是一个初始化。看下来就`found`没有被声明。

因此①：`found = False`。

②，③显然有难度。这时候我们需要去推断出它的算法，而不是看着代码硬解。

### 大的要来了

#### 算法梳理

我们总结一下题目给出的数据结构和算法。

- 数据结构：
    - 链表
- 算法：
    - 枚举算法

这很容易看出来。

它要枚举什么？

- 枚举出所有路径，然后找到最短的那几条路径。

那么，它是怎么枚举的？

- 根据(2)题可知，题目是逆向思维，由终点去找起点。查找终点的所有相邻顶点，之后再由每个相邻顶点去找其相邻顶点，不断循环下去。直到找到起点。

这意味着该链表中的每个节点要指向多个节点，对吧？

可是一个顶点有多个相邻顶点啊！而且题目中规定的`path`中的节点也只指向一个节点啊！

那么每个节点只能指向一个节点，这个节点该指向谁呢？

不要急，我们再看一下(2)题给的图。（这里图画错了，`10`的子结点`8`应为`6`）

<div align="left"><img src="/jiaxing-oi/tree.png" width="100%" alt="第15题图c" title="第15题图c"></div>

我超！树！

一个节点要指向多个节点，原来，这是要用链表存储的树。

我懂了，一切都懂了。

我让子结点指向父结点，一切都结束了。

实际上，我们的`path`链表应该长这样：

<div align="left"><img src="/jiaxing-oi/reversed-tree.png" width="100%" alt="第15题图c-reversed" title="第15题图c-reversed"></div>

因此，用到的数据结构和算法为：

- 数据结构：
    - 树
    - 链表
- 算法：
    - 枚举算法

理顺了这些，我们再看代码。

#### 代码逻辑梳理

```python
'''(4)'''
m = 3 # 横向正方形数量
n = 2 # 纵向正方形数量
mtx = init(m, n)
x = int(input("请输入起点："))
y = int(input("请输入终点："))
path = [[] for i in range(999)]
passed = [False] * len(mtx) # 保存顶点是否已途经
found = False
dis = 0
head = 0
tail = 0
path[tail] = [y, 0, -1]
tail += 1
passed[y] = True # 标识已经过终点
while not found:
    dis += 1
    pass_dis = [False] * len(mtx)
    tmp = tail # 临时变量，之后的head就是现在的tail
    for i in range(head, tail):
        v = path[i] # 当前顶点
        for d in mtx[v[0]]: # 访问当前顶点的所有相邻顶点，d为相邻顶点编号
            if not passed[d]:
                path[tail] = ___②___ # 访问未途经的相邻顶点
                tail += 1 # 每往path里增加一个节点，tail就+1，所以tail可以表示path的长度
                pass_dis[d] = True
            if d == x:
                found = True # 找到了
    head = tmp
    for i in range(len(mtx)): # 标记已途经的顶点
        if ___③___:
            passed[i] = True
# 输出结果           
print_path(x, path, tail)
```

做好了这些注释之后，发现这里还是有个疑点：

- `passed_dis`是个什么JB？

貌似不要紧，我们先看下②处是不是可以做了。

我们模拟下程序执行过程看看。

我们结合一下方才推断出的算法，可以得到下图：

<div align="left"><img src="/jiaxing-oi/flow-one.png" width="40%" alt="flow-one" title="flow-one"></div>

每个新增的子结点统统指向父结点。

因此②处应填：`[d, dis, i]`。

#### What the fuck is `passed_dis`?

我们不妨顺着树走一遍程序看看。

假设现在没有`passed_dis`这个似乎多余的玩意，源代码应该长这样：

```python
'''(4)部分'''
while not found:
    dis += 1
    pass_dis = [False] * len(mtx)
    tmp = tail # 临时变量，之后的head就是现在的tail
    for i in range(head, tail):
        v = path[i] # 当前顶点
        for d in mtx[v[0]]: # 访问当前顶点的所有相邻顶点，d为相邻顶点编号
            if not passed[d]:
                path[tail] = ___②___ # 访问未途经的相邻顶点
                tail += 1 # 每往path里增加一个节点，tail就+1，所以tail可以表示path的长度
                passed[d] = True # 修改后的，原为passed_dis[d] = True
            if d == x:
                found = True # 找到了
    head = tmp
# 输出结果           
print_path(x, path, tail)
```

我们顺着树走一遍看看。

<div align="left"><img src="/jiaxing-oi/banned-tree.png" width="100%" alt="banned-tree" title="banned-tree"></div>

可以看到，很多原本应该要走到的顶点，因为已途经而不再走了。

我们发现这些暗掉的结点，**在同一层下，是可以被重复经过的；而在不同层下，不可以被重复经过。**

这也是`passed_dis`存在的原因。

它要记录当前层下途经过的结点。在当前层下，可以重复经过在当前层下经过的结点。直到下一层，它才会将途经过的结点加入到`passed`中，以防经过在上一层经过的结点。

讲到这里，答案也就显而易见了。③处应填：`passed_dis[i]`。

## 延伸

### 深搜与广搜

这题实际上用到了**树的广度优先搜索**，简称**广搜**。

维基百科这样定义：

---

**广度优先搜索算法**（英语：Breadth-first search，缩写：**BFS**），又译作**宽度优先搜索**，或**横向优先搜索**，是一种[图形搜索算法](https://zh.wikipedia.org/wiki/搜索算法)。简单的说，BFS是从[根节点](https://zh.wikipedia.org/wiki/树_(数据结构)#术语)开始，沿着树的宽度遍历树的[节点](https://zh.wikipedia.org/wiki/节点)。如果所有节点均被访问，则算法中止。广度优先搜索的实现一般采用open-closed表。

---

简单点讲，就是遍历一棵树(或图)的时候，优先遍历同一层结点，这样就按照树的宽度去遍历了，即越宽越好。

通常，我们会采用**队列**来实现BFS。这道题就是一个很典型的BFS实现。并且这里表面上是**树**，实际上是一个**图**。而且是一个**无向图**。

<div align="left"><img src="/jiaxing-oi/graph.png" width="80%" alt="graph" title="graph"></div>

之后对该图进行广搜，找到所有最短路径。



相对应地，这个世界上存在**树的深度优先搜索**，简称**深搜**。

维基百科这样定义：

---

**深度优先搜索算法**（英语：Depth-First-Search，DFS）是一种用于遍历或搜索[树](https://zh.wikipedia.org/wiki/树_(数据结构))或[图](https://zh.wikipedia.org/wiki/图_(数学))的[算法](https://zh.wikipedia.org/wiki/算法)。这个算法会尽可能深地搜索树的分支。当节点v的所在边都己被探寻过，搜索将回溯到发现节点v的那条边的起始节点。这一过程一直进行到已发现从源节点可达的所有节点为止。如果还存在未被发现的节点，则选择其中一个作为源节点并重复以上过程，整个进程反复进行直到所有节点都被访问为止。[[1\]](https://zh.wikipedia.org/zh-hans/深度优先搜索#cite_note-ItoA-1)（p. 603）这种算法不会根据图的结构等信息调整执行策略[[来源请求\]](https://zh.wikipedia.org/wiki/Wikipedia:列明来源)。

深度优先搜索是图论中的经典算法，利用深度优先搜索算法可以产生目标图的[拓扑排序](https://zh.wikipedia.org/wiki/拓扑排序)表[[1\]](https://zh.wikipedia.org/zh-hans/深度优先搜索#cite_note-ItoA-1)（p. 612），利用拓扑排序表可以方便的解决很多相关的[图论](https://zh.wikipedia.org/wiki/图论)问题，如无权最长路径问题等等。

---

就是越深越好。

通常，我们会采用**栈**来实现DFS。在此之中，我们也会维护一个`flag`数组用来标识已经过的节点。



这里其实可以延伸到很多知识。不过都是竞赛相关的了，因为最短路径问题本身就很经典。感兴趣的可以访问 [OI WIKI](https://oi-wiki.org/)。