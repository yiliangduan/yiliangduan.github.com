---
title: Unity的MemoryPool模块
date: 2018-07-14 16:44:10
type: "tags"
tags: unity
comments: false

---

> unity 4.7.1f1

MemoryPool主要是对内存分配的碎片化和频繁分配和释放进行优化。

**结构**

MemoryPool的设计结构如下:

```c++
public class MemoryPool
{
    ...
  
private:
    struct Bubble
    {
        char data[1]; //只是一个占位符,其实这个成员变量加不加sizeof(Block)大小都是1个字节
    };

    typedef dynamic_array<Bubble*> Bubbles;

    Bubble m_Bubbles;
  
    void* m_HeadOfFreeList;
  
    ...
}
```

<!-- more --> 

这里解释下里面的结构布局:

![](/images/unity_memorypool/unity_memorypool_1.png)



这里有个巧妙的设计就是Bubble分配了一块内存，这块内存会被分成几个指定的Block。这些Block在没有分配给用户使用的时候每个Block的首部4字节会存储下一个Block的首部地址，对于第一个Block的首部地址就单独用一个4字节的成员变量保存，就像一个链表一样。这样就不需要另外在用其他变量去保存每个Block的地址。



**分配**

每次用户请求内存的时候，固定给用户返回一个Block，这里有一个限制就是用户不能一次请求大于Block大小的内存。具体如下:

```c++
Bubble * bubble = (Bubble*)malloc(BubbleSize);//BubbleSize大小在创建MemoryPool的时候固定设置

m_Bubbles.push_back(bubble); //所有分配的bubble都会存起来，等待用户清理，然后在循环利用

m_HeadOfFreeList = bubble-data;

void* oldHeadOfFreeList = m_HeadOfFreeList;
void **newBubble = (void**)m_HeadOfFreeList;

//这里是把新分配的Bubble这块内存，相当于分成若干个Block，每个Block的首地址内存存储下一个Block的首地址
for (int j=0; j<m_BlocksPerBubble-1; ++j)//m_BlocksPerBubble是每个Bubble有几个Block
{
  //当前地址存储下一个Block的首地址
  newBubble[0] = (char*)newBubble + m_BlockSize;//m_BlockSize是每个Block的大小。
  
  //newBubble移动到一下个Block地址
  newBubble = (void**)newBubble[0];
}

newBubble[0] = oldHeadOfFreeList; //保存到之前可用内存的首地址

```



通过上述操作就可以分配一块新的可用的内存，当用户申请内存的时候直接返回给用户一个Block的地址即可:

```
void * allocBlock = m_HeadOfFreeList; //allocBlock返回给用户

m_HeadOfFreeList = *((void**)m_HeadOfFreeList); //移动到下一个Block
```



分配内存给用户的时候会有两个问题。

* 分配内存给用户的时候必须限制用户最大分配Block大小的内存，否则返回NULL。
* 因为分配的这个Block是在一个大块的连续的Bubble内存中，如果Block不是Bubble中最后一块内存，那么用户越界的话不会产生错误，但是会把下一个Block的数据给覆盖的问题。

**释放**

```
//首先判断释放的这块内存是否是m_Bubbles分配的

//先把需要释放的这块内存存储下当前可用内存的地址
*((void**)mem_Block) = m_HeadOfFreeList; //mem_Block是要释放的内存

//再把当前可用内存的地址只想释放的这块内存
m_HeadOfFreeList = mem_Block;
```


