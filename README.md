## **C++面经总结**
### **C++路线图**
![图片](image/1.png)

### **内存模型**
首先让我们讲解下内存模型。
#### **内存分布**
下面是一张内存分布图
![图片](image/2.png)
* 从高地址到低地址，一个程序由**内核空间**、**栈区**、**堆区**、**BSS段**、**数据段（data）**、**代码区**组成。
* 可执行程序在运行时会多出两个区域：
  * **堆区**：动态申请内存用。堆从**低地址向高地址增长**。
  * **栈区**：存储局部变量、函数参数值。 栈从**高地址向低地址增长**
* 在堆栈之间有一个**共享区（文件映射区，跟动态库有关系）**。
* **BSS**段：一块存放程序中**未初始化**的**全局变量**和**静态变量**的内存区域。
* **数据段(data)**：一块存放程序中**已初始化**的**全局变量**和**静态变量**的内存区域。
* **代码段**：存放程序执行代码的一块内存区域。只读，不允许修改，代码段的头部还会包括一些**只读的常量**，如**字符串常量字面值**（注意：const变量虽然属于常量，但本质还是变量，不存储于代码段）。


#### **内存分配**
##### **栈和堆的区别**
* **申请方式及申请效率不同**
  * 栈是自动申请和释放，堆是程序手动申请和释放。
  * 栈申请的快，而堆申请的慢。
* **能够申请的空间不同**
  * 栈空间最多2M，超过会Overflow。
  * 堆的空间理论可以解决3G。
* **缓存方式不同**
  * 栈是一级缓存，而堆是二级缓存。
* **内存利用率**
  * 栈不会产生内存碎片，而堆是人为申请，容易产生内存碎片。

##### **new、malloc、free、delete关健字**
###### **new和delete**
* 必须成对出现，否则内存泄露，delete的作用是回收用new分配的内存（释放内存），不是new出来的内存是不能用delete删除滴~
* delete一块内存，只能delete一次，不能delete多次。delete后，这块内存就不能用了，空指针可以多次删除但是没有实际意义。 
###### **malloc和free**
* malloc仅仅是分配一块内存,malloc返回值是void*,同理free仅仅是释放一块内存
###### **区别**
* **实际操作**：**new/delete** 调用构造函数并初始化
调用析构函数并释放内存。**malloc和free**申请一段新空间
释放一段空间。
* **本质**：**new/delete**本质是关键字，而**malloc和free**本质是函数。
* **初始化**：**new**会初始化，**malloc**不会初始化。
* **返回值**：**new**返回对象的指针，**malloc**返回的(void *)，需要强制转化对应的指针。
* **限制**：**new**的内存必须**delete**来回收。**malloc**的内存必须
用**free**回收。
#### 常见问题
* **OOM**: OOM（**Out Of Memory**）就是我们常说的内存溢出，它是指需要的内存空间大于系统分配的内存空间。
* **Memory Leak**: 内存泄漏(**Memory Leak**)是指程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。
#### 常见的内存检测工具
这里只介绍一种常用检测内存的工具：valgrind。
##### **case 1：排查Invalid Write**
```C++
#include <cstdio>
#include <cstdlib>
int* create(){
    return (int*)malloc(sizeof(int));
}
int main(){
    int *p = create();
    *p = 100;
    printf("%d\n", *p);
    free(p);
    int a = *p;
    return 0;
}
```
* step1.先编译 (-g表示开启gdb调试)  --->这样valgrind可以看到代码行数
```
 g++ -o test2  test2.cpp -g
```
* step2.运行valgrind,--leak-check=no | summary | full：对发现的内存泄露给出的信息级别，只有memcheck可用。
```
valgrind --tool=memcheck --leak-check=full ./test2
```
* step3.查看结果
![图片](image/3.png)

##### **case 2：排查OOM**
```C++
#include <cstdio>
#include <cstdlib>

int main(){
    int* array = (int*)malloc(sizeof(int)*10);
    for(int i=0;i<=10;i++)
        array[i]=i;
    return 0;
}
```
![图片](image/4.png)

##### **识别valgrind的提示**
* **definitely lost**：确认丢失。程序中存在内存泄露，应尽快修复。当程序结束时如果一块动态分配的内存没有被释放且通过程序内的指针变量均无法访问这块内存则会报这个错误。
* **indirectly lost**：间接丢失。**当使用了含有指针成员的类或结构时可能会报这个错误**。这类错误无需直接修复，他们总是与”definitely lost”一起出现，只要修复”definitely lost”即可。
* **possibly lost**：**当程序结束时如果一块动态分配的内存没有被释放且通过程序内的指针变量均无法访问这块内存的起始地址，但可以访问其中的某一部分数据，则会报这个错误**。
* **still reachable**：可以访问，未丢失但也未释放。如果程序是正常结束的，那么它可能不会造成程序崩溃，但长时间运行有可能耗尽系统资源，因此笔者建议修复它。如果程序是崩溃（如访问非法的地址而崩溃）而非正常结束的，则应当暂时忽略它，先修复导致程序崩溃的错误，然后重新检测。
* **suppressed**：已被解决。
  
#### **memcpy实现**
来源一道阿里面试题，如何实现一个严谨的**memcpy**。
![图片](image/5.png)
```C++
void memcpy(char * dest, const char * src, int n){
    //Problem 1 重叠问题：dest在src的前面
    //Problem 2 重叠问题：dest在src的后面
    //Tips:不可移动dest的指针
    if((dest==NULL)||(src==NULL)||(n<=0)){
        return NULL;
    }
    char *p=(char*)dest;
    char *q=(char*)src;
    if((dest>src)&&((src+n)>dest)){
        p=p+n-1;
        q=q+n-1;
        while(n--){
            *p++=*q++;
        }
    }else{
        while(n--){
            *p++=*q++;
        }
    }
    return dest;
}
```

### **编译相关**
#### **GCC/G++的工作流程**
* **预处理**：也叫预编译。除了#pragma编译指令，其他的#号指令全部完成转换，比如#include的头文件要完成导入，#define的宏要展开，#if的条件编译，注释全部删除。这个过程不会检查错误，生成预处理文件main.ii。
  * **#ifdef** 和 **#endif**
    * **#ifdef**判断某个宏是否被定义，若已定义，执行随后的语句。
    * **#endif** 是 **#if**, **#ifdef**, **#ifndef**这些条件命令的结束标志。
    * 其他的类似的命令还有 **#ifndef**、**#if**等。
  * **#pragma**
    * **#pragma**是预处理指令，它的作用是设定编译器的状态或者是指示编译器完成一些特定的动作。
  * **#define**
    * 常说的宏定义。
* **编译**
  * 这一步是整个全流程的核心步骤。把预处理完的文件main.ii进行词法分析、语法分析、语义分析、优化，生成汇编代码文件main.s。这个过程会**检查语法错误**，也可以通过参数来**屏蔽某些编译警告**。
  * **模板**(**template**)和**内联**(**inline**)在大多数编译器中都是在编译阶段进行处理。template在编译阶段完成具现化。inline在编译阶段将函数调用替换为函数本体，从而减少函数调用的开销。
* **汇编**：该步骤比较简单，它只是把上一步生成的汇编文件main.s翻译成机器指令，输出二进制文件main.o。
* **链接**：链接过程就像堆积木，本质上就是把多个不同的目标文件粘在一起。
  * **动态链接**：动态链接就是把调⽤的函数所在⽂件模块和调⽤函数在⽂件中的 位置等信息链接进⽬标程序，程序运⾏的时候再从中寻找相应函数代码，因此需要相应⽂件的⽀持 。
  * **静态链接**：静态连接库就是把 (lib或者a) ⽂件中⽤到的函数代码 直接链接进⽬标程序 ，程序运⾏的时候不再需要其它的库⽂件。
  * **区别**：动态链接是有需要才会将其整合到代码上，静态链接是不管是否需要都会整合到代码上。


#### **extern C**
extern C的作用是告诉C++编译器用C规则编译指定的代码（除函数重载外，extern C不影响C++其他特性）。
#### **gdb调试**
* **step1**. 使用ulimit命令，使得C++/C程序在core drump的情况下，能够生成
```
ulimit -c unlimited
```
* **step2**. gcc -g编译，打开调试
```
gcc -g -o test01 test.c
```
* **step3**.只会会生成一个core drump的文件，然后用gdb test01 test01.core，会进入到gdb界面中。
* **常见命令**：
  * **bt**：bt命令的作用书打印程序的堆栈，gdb调试过程中输入bt可以清晰的看到函数的调用路径。where命令也有相同的作用。
  * **frame x**：进入x-th的frame。
  * **down x**：往栈顶方向下移x个frame。
  * **up x**：往栈底方向向上移动x个frame。
  * **info args**：打印出当前函数的参数名称和值。
  * **info locals**：打印出当前函数里所以的局部变量的名字和值。
  * **print x**：打印x变量的值。
  * **help**：帮助命令。
  * **source xx.gdb**：编译xx.gdb脚本，之后可以像函数一样使用这个脚本。

### 参考资料
* [字节跳动 提前批C++开发一面面经 →【鹿の面经解答】](https://www.nowcoder.com/discuss/981246)
* [Linux下内存问题检测神器：Valgrind](https://zhuanlan.zhihu.com/p/75328270)
* [memcpy函数的实现](https://blog.csdn.net/lswfcsdn/article/details/122066079)
* [C++编译过程](https://zhuanlan.zhihu.com/p/497491709)
* [条件编译#ifdef的详解](https://blog.csdn.net/weixin_52244492/article/details/123982193)
* [动态链接库.so和静态链接库.a的区别](https://blog.51cto.com/u_13097817/2047647)
* [linux gdb调试命令详解](https://blog.csdn.net/strut/article/details/120309034)
* [gdb 查看函数调用堆栈（frame概念）](http://t.zoukankan.com/xiaoshiwang-p-12893748.html)