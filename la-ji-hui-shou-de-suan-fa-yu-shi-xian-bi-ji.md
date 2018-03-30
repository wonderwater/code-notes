# 垃圾回收的算法与实现笔记

---

## 1. Python的**引用计数**实现

python的字典对象内存分配：

| function | level |  |
| :--- | ---: | :--- |
| PyDict\_new\(\) | 3 |  |
| PyObject\_GC\_New\(\) | 2 |  |
| PyObject\_Malloc\(\) | 2 |  |
| new\_arena\(\) | 1 | 积蓄0层的空间 |
| malloc\(\) | 0 |  |

0层 - malloc\(\)  
调用时机：大于256byte

1层 -   
arena&gt;pool&gt;block  
以block返回给调用者

unused\_arena\_objects[]: 取可用的area_object
usable\_arenas[]: 取可用的pool。按照area_object.nfreepools降序排序

2层
usedpools: 紧凑的初始化过程

## A. C/C++回顾

数组指针：

```cpp
char (*pa)[4];
```

指针数组：

```cpp
char *arr[4] = {"hello", "world"};
```

地址对齐hack：根据cpu的不同，但在32位cpu上，glibc malloc总返回8byte对齐的地址，即二进制后三位总为0，这样我们可以利用后三位做标记用途，只是在寻址时将后三位置0

typedef:
typedef void (* Func)(void) 中，要被声明的“typedef名称”是Func，也就是说，Func与void (*变量名)(void) 同义。而void (*变量名)(void)是一种函数指针形式，那么Func也就是这种函数指针的同义了。
(https://www.zhihu.com/question/19894694/answer/13278250)


