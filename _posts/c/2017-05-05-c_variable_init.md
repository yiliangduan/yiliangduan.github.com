---
title: 'C语言的变量初始化问题'
date: 2017-05-05 09:50:36
type: "tags"
tags: c
comments: false
---

**问题**

今天在写opengl shader的时候遇到了一个问题(使用C语言编写的)，出问题的代码如下(这里只列出出问题的部分代码):


```

...

void main()
{
    vec3 norm = normalize(Normal);
    vec3 viewDir = normalize(viewPos - FragPos);
    
    vec3 result;
    
    for (int i=0; i<2; ++i)
    {
        result += CalcPointLight(pointLights[i], norm, viewDir);
    }
    
    color = vec4(result, 1.0f);
}

...


```

这段代码的原本的功能是计算点光源。可是代码跑起来之后使用这个shader的模型颜色明显异常。效果如下:

![](/images/c_init_error_state.png)

最终查出来的错误的原因是main函数里面result未初始化的原因导致的。一开始我以为是写的光照函数CalcPointLight里面实现有错误,定位了很久才发现出问题的点。

正确的效果应该是这样的:

![](/images/c_init_right_state.png)

验证了一下result如果没有初始化直接使用的话，它会对应一个stack区的随机值。这样的话在累加光照值之后计算的color值肯定是错误的。(其实我想如果C语言会报错提示使用为初始化的局部变量会更加人性化)

**原理**

顺着这个思路想了一下，为什么C语言的全局变量(global)就算不赋值会被自动初始化位默认值，但是局部变量(local)不会呢? 学习了一下C语言的内存布局结构，然后自己验证了一下然后明白了这个原因。


首先我们得知道C语言的内存布局结构,这篇文章 [Memory layout of c program](http://cs-fundamentals.com/c-programming/memory-layout-of-c-program-code-data-segments.php) 讲的非常详细。这里列出我要用到的部分,首先看结构(图片来自这篇文章):

![](/images/code-data-segments.png)

从这篇文章里面我明白了两点:

* 全局变量是存放在data段内存的。data段分为uninitialized data(bss)段和initalized data，未初始化的变量是放在bss段的，这部分内存存放的变量是会被自动初始化的(这是C语言的特性)
* 局部变量是存放在stack段的。这部分内存是被runtime时期动态分配的。(其实局部变量在代码编译之后就是一个地址，接下来会演示出来)

明白了这两点，那我通过一个demo来验证一下是否在内存中真的是这个样子。

>测试环境: Ubuntu 16.02.2 i386

```
//test_c_variable_init.c

#include <stdio.h>

int global_var;

int main()
{
    int local_var;
    
    local_var = local_var + 3;
        
    printf("global_var: %d & local_var: %d \n", global_var, local_var);
    
    return 0;
}

```

利用demo里面声明的一个全局变量(global_var)和局部变量(local_var)来验证现在的问题。(对local_var加常量3的操作仅仅是为了能够快速定位local_var在反编译代码中的位置)

首先可以通过linux的size命令查看总的内存分布

编译test_c_variable_init.c文件

> gcc -o test_c_variable_init test_c_variable_init.c

得到可执行文件test_c_variable_init,现在可以查看到这个可执行文件的内存分布:

![](/images/test_c_variable_init_size_1.png)

然后我们在test_c_variable_init.c中增加一个全局变量:

> int global_var2;

按照之前的步骤我们再次查看重新编译的文件的内存分布:

![](/images/test_c_variable_init_size_2.png)

可以看到bss段增加了4kb，这个大小就是我们增加的global_var2变量的大小。按照同样的方式增加local_var2我们会发现data和bss段都不会增加。这样可依初步确定上面列出的两点结论了。

然后我们通过查看反编译代码直接来查看变量在编译之后的位置，这样就明确之前的两点结论。

> objdump -D test_c_variable_init > test_c_variable_init.d

我们把反编译之后的代码重定向到test_c_variable_init.d文件，然后我们查看下这个文件，其中有两个地方可以让我们得到想要的答案。

这是反编译之后的main函数:

![](/images/test_c_variable_init_main.png)

可以看到347行的 *$0xc* ，之前我列出demo的时候为local_var特地添加了一个加常量3的操作，通过这个操作我可以很快定位到了*$0xc* 就是对应的local_var变量了, *$0x3* 就是常量3(addl就是汇编里面的加运算)。这里证明了local_var是分配在stack区的。此外我们注意到 *0x804a020* 这个地址其实就是global_var变量了，跳转到这个地址

![](/images/global_var_section.png)

可以很清楚的看到 *0x804a020* 就是global_var并且存放在bss段的。

这样我们就验证了上面的两点结论了,也知道了为什么未初始化的全局变量会被自动初始化位默认值，局部变量不会被初始化了。







