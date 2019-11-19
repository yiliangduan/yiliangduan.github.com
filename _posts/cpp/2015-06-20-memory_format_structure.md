---
title: C++虚函数的内存结构浅析
date: 2015-06-20 16:44:10
type: "tags"
tags: cplusplus
comments: false
---

总结一个下c++虚函数的内存结构问题



```cpp
class A{
public:
    A(){print1();} //在构造函数里面调用virtual函数的做法本身不对，这里只为测试

    virtual void print1(){
        std::cout << "A print1" << std::endl;
    }
};

class B : public A{
public:
    virtual void print1(){
        std::cout << "B print1" << std::endl;
    }
};

class C{
public:
    char ch;
    virtual void print2(){
        std::cout <<"C print2"<<std::endl;
    }
};

class D : public A, public C
{
public:
    int cd;
    virtual void print1(){
        std::cout << "D print1" << std::endl;
    }

    virtual void print2(){
        std::cout << "D print2" << std::endl;
    }
};


int main()
{
    //Q1
    B b; // 输出 A print1

    //Q2
    std::cout << sizeof(D) <<std::endl; //16
    return 0;
}
```

//A1:
输出A print1的原因是在class A的构造方法里面，对象b的构造方法内部还没有执行，
所以对象b还没有初始化也就没有虚函数表，在A的构造方法里面this指针就是A类型的，
所以是调用A的print1

//A2

```cpp
//A的内存布局为
----------
__vfptr = 0x00c0ae48 const A::`vftable'{for `A'}

----------

//C的内存布局
----------
__vfptr = 0x00c0abd8 const C::`vftable'{for `C'}
ch = 0xcccccccc
----------

//D的内存布局
---------
A = {__vfptr = 0x00c0ae48 const D::`vftable'{for `A'} }
B = {__vfptr = 0x00c0abd8 const D::`vftable'{for `C'}
     ch = 0xcccccccc
    }
cd = 0xcccccccc
---------
```

所以 sizeof(D) = 4 + 8 + 4 （C的大小为8是因为内存还要对其）
