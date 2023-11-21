---
title: c语言笔记
date: 2023-05-23 17:14:57
categories:
- c
tags:
- c
- 笔记
- private
---


# const 用途：
- 定义常量
- 修饰函数参数
- 修饰函数返回值



# const的修饰规则:

- const在`*`的左边，则**指针指向的变量的值**不可直接通过指针改变（可以通过其他途径改变）；

- const在`*`的右边，则**指针的指向**不可变。简记为“**左定值，右定向**”。

  - 指针可以改变，指向的值不能改变

    ```c
    int a=10;
    const int *p1=&a;   //例如：允许 p1+=1 ，不允许 *p1=20
    int const *p2=&a;   //这两句效果一样
    ```

  - 指向的值可以改变，指针不可改变

    ```c
    int a=1;
    int b=2;
    int *const p3=&a;   //例如：允许 *p=3 ，不允许 p3=&b
    ```

  - 指针和指向的值都不能改变

    ```c
    int a=10;
    const int * const p4=&a;
    int const* const p4=&a;    //这两种写法一样
    ```

# signed/unsigned
C语言默认存储类型为 signed，所以 `int a = 1` 等价于 `signed int a = 1` 
- `signed` 意思为**有符号的**，其第一位代表正负，剩余的代表大小。当第一位为 0 时，表示正数。为 1 时，表示负数。所以 signed int a 的取值范围为：-128~127
- `unsigned` 意思为**无符号的**，所有的位都为大小，没有负数，例如：unsigned int a 的取值范围为：0~255

signed/unsigned 只能用于修饰整数变量，不能用来修饰float，double等类型的变量; 

在linux中，对 `byte` 的定义为**无符号** char，而 char 默认为有符号char; 在内存中，char 与 unsigned char 没有什么不同，都是一个字节，唯一的区别是，**char的最高位为符号位**，因此char能表示-127~127,unsigned char没有符号位，因此能表示0~255，8个bit，最多 256 种情况，因此无论如何都能表示256个数字; 

## 为什么 byte 用 unsigned char 表示
byte 没有什么符号位，更重要的是如果将byte的值赋给int，long等数据类型时，系统会做一些额外的工作。如果是char，那么系统认为最高位是符号位，而int可能是16或者32位，那么会对最高位进行扩展, 而如果是unsigned char，那么不会扩展。最高位若为0时，二者没有区别，若为1时，则有区别了. 



# static 用途：
1. 局部静态变量
2. 外部静态变量/函数
3. 静态数据成员/成员函数

- 作用主要有以下3个：
1. 扩展生存期；针对普通局部变量和static局部变量来说的。声明为static的局部变量的生存期不再是当前作用域，而是整个程序的生存期。
2. 限制作用域；static全局变量和函数，其作用域为当前cpp文件，其它的cpp文件不能访问该变量和函数。如果有两个cpp文件声明了同名的全局静态变量，那么他们实际上是独立的两个变量.
3. 唯一性；

# inline
函数被调用的语句就被替换成了函数本身，进行了一个预处理,免去进出函数的过程.



# volatile:
用最通俗易懂的话说就是禁止编译器优化，一个被声明为 `volatile` 的变量说明该变量**可能会被意想不到的改变**，所以每次使用的时候都会重新读取变量的值，不会使用保存在寄存器里的备份。

**经典例子：**
- 并行设备的硬件寄存器，如状态寄存器
- 一个中断服务子程序中会访问到的非自动变量
- 多线程应用中被几个任务共享的变量
- 使用DMA时希望读取的是物理内存的值而不是缓存数据

**衍生问题：**
【1】非自动变量: 
- 非自动变量：全局变量、静态变量；
- 自动变量：在函数内部定义和使用的变量；

【2】一个参数可以是const的同时也可以是volatile吗？
是的，只读状态寄存器，它是volatile因为可能被意想不到的改变，也是const因为程序不应该试图去修改。

【3】一个指针可以是volatile吗？
可以，一个例子是当一个中断服务子程序修改一个指向buffer的指针时。

【4】下面代码有什么错误？
```c
int square(volatile int *ptr)
{
         return ptr* *ptr;
}
```
这段代码目的是求取ptr指针指向的整型数的平方，但是该指针是volatile类型，编译器会解释为下述代码
```c
int square(volatile int *ptr)
{
        int a,b;
        a = *ptr;
        b = *ptr;
        return a * b;
}
```
因为该指针指向的值可能被意想不到的改变，所以a和b的值可能不同。
改进措施：
```c
long square(volatile int *ptr)
{
        int a;
        a = *ptr;
        return a * a;
}
```


# enum 巧用：

用于自动计算枚举个数

```c
enum {
    AA,
    BB,
    CC,
    DD,
    count // 用于自动计算枚举个数
};


int main() {
    int i = 0;
    printf("count = %d\n",count);
    for (i = 0; i < count; i++) {
        ...
    }
    return 0;
}
```

# 数组名的指针问题
在 `C` 中，数组名 `arr` 代表的是数组首元素的地址，相当于 `&arr[0]` 而 `&arr` 代表的是整个数组 `arr` 的首地址.

```c
int arr[5] = {1, 2, 3, 4, 5};

// arr 数组名代表数组首元素的地址
printf("arr -> %p\n", arr);		// arr -> 0061FF0C

// &arr[0] 代表数组首元素的地址
printf("&arr[0] -> %p\n", &arr[0]);	// &arr[0] -> 0061FF0C

// &arr 代表整个数组 arr 的首地址
printf("&arr -> %p\n", &arr);		// &arr -> 0061FF0C
```

运行程序可以发现 arr 和 &arr 的输出值是相同的，但是它们的意义完全不同。 首先数组名 arr 作为一个标识符，是 arr[0] 的地址，从 &arr[0] 的角度去看就是一个指向 int 类型的值的指针。 而 &arr 是一个指向 int[5] 类型的值的指针. 

```c
// 指针偏移
printf("arr+1 -> %p\n", arr + 1);	// arr+1 -> 0061FF10
printf("&arr+1 -> %p\n", &arr + 1);	// &arr+1 -> 0061FF20
```

`arr+1` : `arr` 是一个指向 `int` 类型的值的指针，因此偏移量为 `1*sizeof(int)`;

`&arr+1` : `&arr` 是一个指向 `int[5]` 的指针，它的偏移量为 `1*sizeof(int)*5`;

> 偏移量的知识：一个类型为 T 的指针的移动，是以 sizeof(T) 为移动单位的。


```c
int arr[5] = {1, 2, 3, 4, 5};

// ptr 是一个指针，为 arr 数组的第一个元素地址
int *ptr = arr;
printf("%p %d\n", ptr, *ptr);	// 0061FF08 1

// ptr 指针向高位移动一个单位，移向到 arr 数组第二个元素地址
ptr++;
printf("%p %d\n", ptr, *ptr);	// 0061FF0C 2

return (0);
```


# 指针传递
```c
// NO1
void GetMemory(char *p)
{
       p=(char *)malloc(100);
}
void Test()
{
  char * str=NULL;
  GetMemory(str);
  strcpy(str,"Hello world");
  printf(str);
}
// 实质：GetMemory（str）在调用时会生成一个_str与str指向同一个数，这是因为C语言中函数传递形参不改变实参的内容，但是指针指向的内容是相同的，因此可以用指针控制数据。题中的GetMemory(str),实质是对_str的操作，并没有对str操作，函数结束后_str撤销，因此不会产生新的内存空间，str仍然是一个空指针。


// NO2
char *GetMemory()
{
       char p[]="Hello World";
       return p;
}
void Test()
{
       char * str=NULL;
       str=GetMemory();
       printf(str);
}
// 实质：当一个函数调用结束后会释放内存空间，释放它所有变量所占用的空间，所以数组空间被释放掉了，也就是说str所指向的内容不确定是什么东西。但是返回的指针指向的地址是一定的。


// NO3
char *GetMemory()
{
       Return “hello world”;
}
void Test()
{
       char * str=NULL;
       str=GetMemory();
       printf(str);
}
// 实质：本例打印hello world，因为返回常量区，而且并没有修改过。在上一个例子中不一定能打印hello world，因为指向的是栈区。



// NO4
void GetMemory(char **p,int num)
{
       *p=(char *)malloc(num);
}
void Test()
{
       char * str=NULL;
       GetMemory(&str,100);
       strcpy(str,"Hello");
       printf(str);
}
// 可以正确的打印Hello但是内存泄露了，在GetMemory（）中使用了malloc申请内存，但是在最后却没有对申请的内存做任何处理，因此可能导致内存的泄露，非常危险。


// NO5
void Test()
{
       char *str=(char *)malloc(100);
       strcpy(str,"Hello");
       free(str);
       if (str!=NULL)
       {
              strcpy(str,"World");
              printf(str);
       }
}
// 申请空间，拷贝字符串，释放空间，前三步操作都没有问题，到了if语句里的判断条件开始出错了。因为一个指针被释放了之后其内容并不是NULL，而是一个不确定的值，所以if语句永远不能被执行，这也是著名的“野”指针问题。


// NO6
void GetMemory(void)
{
       char *str=(char *)malloc(100);
       strcpy(str,"hello");
       free(str);
       if (str !=NULL)
       {
              strcpy(str,"world");
              printf(str);
       }
}
// Str 为野指针，打印的结果不能确定。


// NO7 
char *GetMemory (char **p)
{
    *p = "hello word";
    return *p;
}

int main()
{
    char *str = NULL;
    if (NULL !=GetMemory (&str))
    {
        printf("  str = %s\n",str);
    }
    return 0;
}
// 可以输出hello word 没问题


// NO8  区别于NO2
char *GetStr(char *p)
{
    p = "hello word";
    return p;
}

int main()
{
    char *str = NULL;
    if (NULL != (str = GetStr(str)))
    {
        // hello word
        printf("  str = %s\n", str);
    }
    return 0;
}
```



# struct 内存占用
```c
#include<stdio.h>

/*
  结构体中的data[0]不占用任何空间，甚至是一个指针的空间都不占，data在这儿只是表示一个常量指针，这个特性是编译器来实现的，即在使用test1.data的时候，这个指针就是表示分配内存地址中的某块buffer，比如：

  _test1 *test1 = malloc(sizeof (_test1) + data_len);
  test1->len = data_len;
  memcpy(test1->data, buffer, data_len);
  
  定义的结构体指针可以指向任意长度的内存buffer，这个技巧在变长buffer中使用起来相当方便, 如果定义为一个指针，它需要占用4Bytes，并且在申请好内存后必须人为赋地址才可以使用
  _test2 test2 = malloc(sizeof (_test2));
  test2.len = data_len;
  test2->data = malloc(data_len);
  memcpy(test2->data, buffer, data_len);
  free(test2->data);
  free(test2)
/*
typedef struct a{
    int data;
    char a[0];     // 常量指针，不占用内存
} A;

typedef struct b {
    int data;
    char *a;
}B;

typedef struct c {
    int data;
    char a[];    // 可变长数组。不占用内存
}C;

union d{
    char a;
    int b;
    char *c;    // 共用体中最大的
}D;

union e{
    char a;
    int b;			// 共用体中最大的
//    char *c;
    char d[0];
}E;

/*
union f{
    char a;
    int b;
//    char *c;
    char d[]; //不能包含可变长
}F;
*/

typedef enum g {
    zero,
    one = 1,
    two,
    three
}G;



int main()
{
    printf("sizeof(a) = %d\n",sizeof(A)); //4
    printf("sizeof(b) = %d\n",sizeof(B)); //16
    printf("sizeof(c) = %d\n",sizeof(C)); //4
    printf("sizeof(union d) = %d\n",sizeof(D)); //8
    printf("sizeof(union e) = %d\n",sizeof(E)); //4
//    printf("sizeof(union f) = %d\n",sizeof(F)); //编译异常
    printf("sizeof(enum g) = %d\n",sizeof(G)); //4 G 是一个变量，跟内容无关
    
}
```



# Struct 字节对齐(位域)：

位域 (位段)：把一个字节中的二进位划分为几个不同的区域， 并说明每个区域的位数. 每个域有一个域名，允许在程序中按域名进行操作。 这样就可以把几个不同的对象用一个字节的二进制位域来表示。

```c
struct 位域结构名
{
	位域列表
};

```

其中位域列表的形式为：

**类型说明符** **位域名**：**位域长度**

```c
unsigned op_type:9; // 表示 op_type 是一个 9 位的整数变量
```
它声明了一个[位域](http://en.wikipedia.org/wiki/Bit_field)；冒号后面的数字以位为单位给出字段的长度（即，用多少位来表示它)

```c
struct A
{
    unsigned a: 4;    //  4 bits
    unsigned b: 4;    // +4 bits, same group, (4+4 is rounded to 8 bits)
    unsigned char c;  // +8 bits
};
// sizeof(A) = 2 (16 bits)


struct B
{
    unsigned a: 4;    //  4 bits
    unsigned b: 1;    // +1 bit, same group, (4+1 is rounded to 8 bits)
    unsigned char c;  // +8 bits
    unsigned d: 7;    // + 7 bits
};
// sizeof(B) = 3 (4+1 rounded to 8 + 8 + 7 = 23, rounded to 24)

struct bs
{
　　int a:8;
　　int b:2;
　　int c:6;
}data;
// 说明data为bs变量，共占两个字节。其中位域a占8位，位域b占2位，位域c占6位。
```

注意点：

1. **一个位域必须存储在同一个字节中，不能跨两个字节**。如一个字节所剩空间不够存放另一位域时，应从下一单元起存放该位域。也可以有意使某位域从下一单元开始。例如：

```c
struct bs
{
  	unsigned a:4
    unsigned b:5 /*从下一单元开始存放*/
    unsigned c:4
}
```

2. 由于位域不允许跨两个字节，因此**位域的长度不能大于一个字节的长度**。
3. **位域可以无位域名**，这时它只用来作填充或调整位置。无名的位域是不能使用的。例如：
```c
struct k
{
  	int a:1
    int :2 /*无位域名，该2位不能使用*/
    int b:3
    int c:2
};
```

```c
struct rb_node
{
    unsigned long  rb_parent_color;
#define    RB_RED        0
#define    RB_BLACK    1
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));

"__attribute__((aligned(sizeof(long))))" 的意思是把结构体的地址按“sizeof(long)”对齐,
```



# __attribute__ :

- 让编译器取消结构在编译过程中的优化对齐,按照实际占用字节数进行对齐, 是GCC特有的语法。这个功能是跟操作系统没关系，跟编译器有关，gcc编译器不是紧凑模式的

```c
在GCC下：struct my{ char ch; int a;} sizeof(int)=4;sizeof(my)=8;（非紧凑模式）
在GCC下：struct my{ char ch; int a;}__attrubte__ ((packed)) sizeof(int)=4;sizeof(my)=5
```

- 函数声明中的`__attribute__((noreturn))`，就是告诉编译器这个函数不会返回给调用者，以便编译器在优化时去掉不必要的函数返回代码。






# ++i 、i++、i=i+1、i+=1 哪个更快，为什么？

- ++i 最快

- i++ 次之，多用一个临时变量

- i+=1 第三，需要取地址

- i = i+1 最后，并多用一个临时变量



# #define与const区别

都可以修饰常量。

1. **const常量有数据类型，而#define（宏）常量没有数据类型**。**编译器可以对前者进行类型安全检查。而对后者只进行字符替换**，没有类型安全检查，并且在字符替换可能会产生意料不到的错误（边际效应）。

2. **const常量在堆栈分配了空间**，而#define（宏）常量只是把具体数值直接传递到目标变量罢了。或者说，const的常量是一个Run-Time的概念，他在程序中确确实实的存在并可以被调用、传递。而#define常量则是一个Compile-Time概念，它的生命周期止于编译期：在实际程序中他只是一个常数、一个命令中的参数，没有实际的存在。

3. **const常量存在于程序的数据段，#define常量存在于程序的代码段**。

4. 有些集成化的调试工具可以对const常量进行调试，但是不能对宏常量进行调试。



# #define和typedef的区别

使用方式：

```c
typedef   (int*)    pINT;

#define   pInt   int*
```

1. #define是预处理指令，在编译时不进行任何检查，只进行简单的替换但在编译替换展开后的程序可能会出现副作用。所以**多加()**

2. typedef是在编译时进行处理，我们也可以将其认为是给某个类型起了一个**别名**。 但它并**不分配实际内存空间**

3. `typedef char *string_a` 与 `#define string_b char *` 的差异

      pINT a,b;的效果同int * a; int * b; 表示定义了两个整型指针变量a和b;

      pInt a,b;的效果同int * a; int b; 表示定义一个整型指针变量a和一个整形变量b;

4. typedef是语句（ 以'；'结尾），而#define不是语句（ 不以'；'结尾）

- define 不能以分号结束，注意括号的使用

```c
#define dPS struct s *
typedef struct s * tPS;

dPS p1,p2;    // p1 为一个指向结构的指针，p2 为一个实际的结构
tPS p3,p4;    // 定义了 p3 和 p4 两个指针
```



# strlen 和 sizeof 比较:
```c
#include<stdio.h>

int main()
{
	char ss[] = "0123456789";
	int aa[] = {0,1,2,3,4,5,6,7,8,9};
	char buf[100] = "0123456789";


	printf("sizeof(ss) == %d\n",sizeof(ss)); //11 多一个\0
	printf("strlen(ss) == %d\n",strlen(ss)); //10

	printf("sizeof(aa) == %d\n",sizeof(aa)); //40 整形数组 sizeof(int) * size
	printf("strlen(aa) == %d\n",strlen(aa)); //不兼容，输出错误

	printf("sizeof(buf) == %d\n",sizeof(buf)); //100
	printf("strlen(buf) == %d\n",strlen(buf)); //10
	return 0;
}
```


# 内存分布：

**1. 栈区（stack）**— 由编译器自动分配释放 ，存放函数的参数值，局部变量的值等。其操作方式类似于[数据结构](http://lib.csdn.net/base/datastructure%20/t%20_blank%20/o%20%CB%E3%B7%A8%D3%EB%CA%FD%BE%DD%BD%E1%B9%B9%D6%AA%CA%B6%BF%E2)中的栈。

**2. 堆区（heap）** — 一般由程序员分配释放， 若程序员不释放，程序结束时可能由OS 。注意它与数据结构中的堆是两回事，分配方式倒是类似于链表，呵呵。

**3. 全局区（静态区）（static）**—，全局变量和静态变量的存储是放在一块的，初始化的全局变量和静态变量在一块区域，未初始化的全局变量和未初始化的静态变量在相邻的另一块区域。 - 程序结束后有系统释放

**4. 文字常量区** —常量字符串就是放在这里的。 程序结束后由系统释放

**5. 程序代码区**—存放函数体的二进制代码。

```c
int a = 0; // 全局初始化区
char *p1;  // 全局未初始化区
int main()
{
  int b; // 栈
  char s[] = "abc"; // 栈
  char *p2; // 栈
  char *p3 = "123456"; // 123456在常量区，p3在栈上。
  static int c =0； // 全局（静态）初始化区(数据段)
    
  // 分配得来得10和20字节的区域就在堆区。  
  p1 = (char *)malloc(10);
  p2 = (char *)malloc(20);
  
  // 123456放在常量区，编译器可能会将它与p3所指向的"123456"优化成一个地方
  strcpy(p1, "123456"); 
}
```



# 线程函数

```c
#include<stdio.h>  
#include<string.h>  
#include<stdlib.h>  
#include<unistd.h>  
#include<pthread.h>  
 
  
void* task1(void*);  
void* task2(void*);  
void usr();  
int p1,p2;  
  
int main()  
{  
    usr();  
    getchar();  
    return 1;  
}  
  
void usr()  
{  
    pthread_t pid1, pid2;  
    pthread_attr_t attr;  
    void *p;  
    int ret=0;  
	  //初始化线程属性结构  
    pthread_attr_init(&attr);      
  	//设置attr结构分离  
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);  
	  //创建线程，返回线程号给pid1,线程属性设置为attr的属性，线程函数入口为task1，参数为NULL  
    pthread_create(&pid1, &attr, task1, NULL);         
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);  
    pthread_create(&pid2, &attr, task2, NULL);  
    //前台工作  
		//等待pid2返回，返回值赋给p  
    ret=pthread_join(pid2, &p);         
    printf("after pthread2:ret=%d,p=%d/n", ret,(int)p);            
  
}  
  
void* task1(void *arg1)  
{  
    printf("task1/n");  
    //艰苦而无法预料的工作，设置为分离线程，任其自生自灭  
    pthread_exit( (void *)1);  

}  
  
void* task2(void *arg2)  
{  
    int i=0;  
    printf("thread2 begin./n");  
    //继续送外卖的工作  
    pthread_exit((void *)2);  
}  
```

## 分离线程：

pthread_attr_setdetachstate：





# 位运算

- 表达式`t = a<<2 + a;`，对于`a<<2`这个位操作，优先级要比加法要低，所以这个表达式就成了`t = a << (2+a)`

- 清零：a & ~(1 << 28 )
- 置一：将第28位置一： a | 1<<28
- 判断奇偶：((a +1) & 1) ^ 1;
- 使特定位翻转： 要使哪几位翻转就将与其进行∧运算的该几位置为１即可。
- 判断int型变量a是奇数还是偶数: a&1  = 0 偶数 ; a&1 =  1 奇数
- 取int型变量a的第k位(k=0,1,2……sizeof(int))，即a>>k&1
- 将int型变量a的第k位清0，即a=a&~(1<<k)
- 将int型变量a的第k位置1， 即a=a|(1<<k)
- 与０相∧，保留原值
- 判断一个整数是不是2的幂,对于一个数 x >= 0，判断他是不是2的幂

```c
boolean power2(int x)
{
  return ((x&(x-1))==0)&&(x!=0)；
}
```

- 交换两个值，不用临时变量：

```c
void swap(int a , int b)
{
  a = a^b;     // a^a=0；
  b = a^b;     // (a^b)^b = a;
  a = a^b;     // (a^b)^(a^b)^b = a^a^b^b^b = b;
}
```

- 计算绝对值

```c
int abs( int x )
{
  int y ;
  y = x >> 31 ;
  return (x^y)-y ;    //or: (x+y)^y
}
```



# 逗号表达式

- 在初始化和变量声明时，逗号并不是逗号表达式的意义;

```c
#include <stdio.h>
 
int main()
{
    int a = 1,2;
//	int a = (1,2);  // 应该扩起来使用
    printf("a : %d\n",a);
    return 0;
}
```



# printf输出注意点：

- printf返回值是输出的字符个数。

```c
#include <stdio.h>
int main()
{
    int i=43;
  	// 4321
    printf("%d\n",printf("%d",printf("%d",i)));
    return 0;
}
```

- %s遇到'\0'就停止输出;**但是之后的字符还是存在的**

```c
#include<stdio.h>
#include<string.h>
int main()
{
    char s[10]="Love mm";
    s[1]=0; //  s[1]=0;//等价于 s[1]='\0';等价于s[1]=NULL;
  	// L 
    printf("s = %s\n",s); // 遇到\0停止输出
  	// v
	  printf("s[2] = %c\n",s[2]); // 后面的内容依旧存在
    return 0;
}
```



# 类型转换注意点：

```c
#include <stdio.h>
int main() 
{
    float a = 12.5;
    printf("%d\n", a); 					// 0
    printf("%d\n", (int)a);			// 12
    printf("%d\n", *(int *)&a); // 1095237632
    return 0; 
}
```

浮点数是4个字节，12.5f 转成二进制是：01000001010010000000000000000000，十六进制是：0x41480000，十进制是：1095237632。所以，第二和第三个输出相信大家也知道是为什么了。而对于第一个，为什么会输出0，我们需要了解一下float和double的内存布局，如下：

- **float**: 1位符号位(s)、8位指数(e)，23位尾数(m,共32位)
- **double**: 1位符号位(s)、11位指数(e)，52位尾数(m,共64位)

然后，我们还需要了解一下printf由于类型不匹配，所以，会把float直接转成double，注意，12.5的float和double的内存二进制完全不一样。别忘了在x86芯片下使用是的反字节序，高位字节和低位字位要反过来。所以：

- **float版**：0x41480000 (在内存中是：00 00 48 41)
- **double版**：0x4029000000000000 (在内存中是：00 00 00 00 00 00 29 40)

而我们的%d要求是一个4字节的int，对于double的内存布局，我们可以看到前四个字节是00，所以输出自然是0了。

这个示例向我们说明printf并不是类型安全的，这就是为什么C++要引如cout的原因了。



# 交叉编译数组的引用：

1.c

```c
int arr[80];
```

2.c

```c
extern int *arr;
// extern int arr[] 正确用法
int main() 
{     
    arr[1] = 100;
    printf("%d\n", arr[1]);
    return 0; 
}
```

编译通过，但运行时会出错；用 extern int *arr来外部声明一个数组并不能得到实际的期望值，因为他们的类型并不匹配。所以导致指针实际并没有指向那个数组。注意：**一个指向数组的指针，并不等于一个数组**。修改：extern int arr[]。



# switch注意点：

- switch-case语句会把变量b的初始化直接就跳过
- default 位置得影响：先比较非default 选项，没有符合的在进入default ，处理完如果没有break ，依次向下执行
- 
```c
#include <stdio.h>
int main() 
{     
    int a=1;     
    switch(a)     
    {  
        // switch-case语句会把变量b的初始化直接就跳过，程序会输出一个随机的内存值。
        int b=20;         
        case 1:
            printf("b is %d\n",b);
            break;
        default:
            printf("b is %d\n",b);
            break;
    }
    return 0;
}
```


# sizeof注意点：

- sizeof不是一个函数，是一个操作符.

```c
#include <stdio.h>
int main() 
{
    int i;
    i = 10;
    printf("i : %d\n",i);
  	// sizeof(i++)直接就被4给取代了，在运行时也就不会有了i++这个表达式。
    printf("sizeof(i++) is: %d\n",sizeof(i++));
    printf("i : %d\n",i);
    return 0;
}
// 10 4 10
```



# 运算法优先级注意点：

`y = y/*p`没有加入空格和括号，结果`y/*p`中的`/*` 被解释成了注释的开始

```c
#include
#define PrintInt(expr) printf("%s : %dn",#expr,(expr))

int main()
{
  int y = 100;
  int *p;
  p = malloc(sizeof(int));
  *p = 10;
  y = y/*p; /*dividing y by *p */;
  // 100
  PrintInt(y);
  return 0;
}
```



# ++i 和 i++ 注意点：

- `++i` 表示取 i 的地址，增加它的内容，然后把值放在寄存器中；
- `i++` 表示取 i 的地址，把它的值装入寄存器中，然后增加内存中i的值；

```c
#include <stdio.h>
int main() 
{
    int i = 6;
    // 对于（条件1 && 条件2），如果“条件1”是false，那“条件2”的表达式会被忽略了。对于（条件1 || 条件2），如果“条件1”为true，而“条件2”的表达式则被忽略了
    // ++i< 7 失败跳过后续，进入++i <=9 为真，所以自加两次
    if( ((++i < 7) && ( i++/6)) || (++i <= 9) );
 
    // 8
    printf("%d\n",i);
    return 0;
}
```

- `++i` 在printf中，首先按自右至左的顺序**求参数的值**；然后输出.
- `i++` 首先按自右至左的顺序求出参数值，求出一个输出一个.
- 涉及到`++i`的运算，需待所有运算结束后，再输出。而`i++`的运算则是先输出，再进行运算
```c
#include <stdio.h>
int main() 
{
    int i=3;
    printf("%d %d %d\n",i,++i,++i); //输出是 5 5 5
    
    i = 3;
    printf("%d %d %d\n",i,i++,i++); //输出是 5 4 3
    
    i = 3;
    printf("%d %d %d\n",i,i++,++i); //输出是 5 4 5
    
    return 0;
}
```

- `-` 和 `++` 为同一优先级，结合方向为自右向左，所有`-i++`相当于`-(i++)`; 

> 此处不是减号运算符，减号运算符优先级比++要低，此处是负号运算符。

```c
	int i = 5;
	// j = 15 是因为i++是先运算之后然后才进行++，因此它的结果是15
	int j = (i++)+(i++)+(i++);  // 15（vc6.0环境 5+5+5）或 18(gcc环境 5+6+7)

	i = 5;
	// 因为++i是先++，然后才运算，但是并不是等所有的加完之后才运算，也就是说并不是24，而是利用贪吃法先进行 k = ((++i) + (++i)) + (++i) 也就是加法从左到右运算,先算前两个加数的和，再和第三个相加的过程当中，他会先把前两个括号内的东西执行完成后，再执行括号外的加法，也就是（7+7）+8=22
	int k = (++i)+(++i)+(++i);  // 22 不是24 （7+7+8）
	printf( "j: %d, k: %d\r\n", j, k );
```


# char *p 与char p[] 比较:
- 所有的字符窜常量都被放在静态内存区；因为字符串常量很少需要修改，放在静态内存区会提高效率
```c
char str1[] = "abc";
char str2[] = "abc";

const char str3[] = "abc";
const char str4[] = "abc";

const char *str5 = "abc";
const char *str6 = "abc";

char *str7 = "abc";
char *str8 = "abc";

// 0
( str1 == str2 )
// 0
( str3 == str4 ) 
// 1
( str5 == str6 )
// 1
( str7 == str8 ) 

// str1,str2,str3,str4是数组变量，它们有各自的内存空间；而str5,str6,str7,str8是指针，它们指向相同的常量区域。
```

- `char *a`和 `char a[]` 得区别：
  1. `char *c = "chengxu"`: "chengxu"的类型是const char *，编译器在编译时会在静态数据区为"chengxu"分配空间存储这个字符串，然后将字符串的首地址赋给字符指针char *c 。对于const char *赋给char *是因为在C语言时代这已成为一种习惯写法。所以，当使用指针c试图改变"chengxu"时，如：c[0]='a'，编译不会出错，但运行时会出错
  2. `char c[]="chengxu"`: 先在内存开辟一连续存储单元，然后将字符串存到存储单元。

- 函数调用：
```c
#include <stdio.h>
char *returnStr()
{
    // 字符串常量，存放在静态数据区，把该字符串常量存放的静态数据区的首地址赋值给了指针，所以returnStr函数退出时，该该字符串常量所在内存不会被回收，故能够通过指针顺利无误的访问。
    char *p  = "hello world!";	// 正常输出
    
    // 字符串常量，存放在静态数据区，没错，但是把一个字符串常量赋值给了一个局部变量(char []型数组)，该局部变量存放在栈中，这样就有两块内容一样的内存，也就是说“char p[]="hello world!";”这条语句让“hello world!”这个字符串在内存中有两份拷贝，一份在动态分配的栈中，另一份在静态存储区。这是与前者最本质的区别，当returnStr函数退出时，栈要清空，局部变量的内存也被清空了，所以这时的函数返回的是一个已被释放的内存地址，所以打印出来的是乱码。
    char p[] = "hello world!";  // 乱码输出
    
    // 针对char a[]修改， 如果函数的返回值非要是一个局部变量的地址，那么该局部变量一定要申明为static类型
    static char p[]="hello world!";
    
    return p;
}

int main()
{
    char *str=NULL;//一定要初始化，好习惯
     str=returnStr();
    printf("%s\n", str);
   
    return 0;
}
```



# void (*p)() 、 void *p()和void *（*p）(void)的区别:
- `void（*p）()` 是一个指向函数的**指针**，表示一个指向函数入口的**指针变量**，该函数的返回类型是void类型。
- `void *p()` 是一个指针型**函数**，它的函数名为p,返回一个指针，因为是void,这个指针没有定义类型，所以返回的是一个通用型指针。
- `void *（*p）(void)` 函数返回值是`void *`指针类型。`*p`则是一个函数指针。括号中的void表示函数指针指向的函数参数为void。


# 引用和指针得区别：
- 引用就是某一变量（目标）的一个别名，对引用的操作与对变量直接操作完全一样。 　　
- 引用的声明方法：
	类型标识符 &引用名=目标变量名；
	例如：
	`int a; int &ra=a;` //定义引用ra,它是变量a的引用，即别名 　　

1. `&`在此不是求地址运算，而是起标识作用。
2. 类型标识符是指目标变量的类型。
3. 声明引用时，必须同时对其进行初始化。
4. 引用声明完毕后，相当于目标变量名有两个名称，即该目标原名称和引用名，且不能再把该引用名作为其他变量名的别名。`ra=1`; 等价于 `a=1`;
5. 声明一个引用，不是新定义了一个变量，它只表示该引用名是目标变量名的一个别名，它本身不是一种数据类型，因此引用本身不占存储单元，系统也不给引用分配存储单元。故：对引用求地址，就是对目标变量求地址。&ra与&a相等。
6. 不能建立数组的引用。因为数组是一个由若干个元素所组成的集合，所以无法建立一个数组的别名.
7. 不能建立引用的引用，不能建立指向引用的指针。因为引用不是一种数据类型，所以没有引用的引用，没有引用的指针。
```c
　　int n；
　　int &&r=n；//错误，编译系统把"int &"看成一体，把"&r"看成一体，即建立了引用的引用，引用的对象应当是某种数据类型的变量
　　int &*p=n; //错误，编译系统把"int &"看成一体，把" *p "看成一体，即建立了指向引用的指针，指针只能指向某种数据类型的变量
```
8. 值得一提的是，可以建立指针的引用
```c
　　int *p;
　　int *&q=p; //正确，编译系统把" int * "看成一体，把"&q"看成一体，即建立指针p的引用，亦即给指针p起别名q。
```

- 指针和引用的比较:
1. 相同点：都是地址的概念:
   - 指针指向一块内存，它的内容是所指内存的地址；
   - 引用是某块内存的别名。
2. 不同点：指针是一个实体，而引用仅仅是个别名；
   - 引用中是没有const，指针有const ，const的指针不可改变
   - 引用不能为空，指针可以为空
   - 引用只能在定义时被初始化一次，之后不可变；指针是可以改变的，引用从一而终；


# 数组和指针区别：
- 修改内容：数组名对应着（而不是指向）一块内存，其地址与容量在生命期内保持不变，只有数组的内容可以改变。
```c
	char a[] = "hello";
	a[0] = 'X';

	char *p = "world";     // 注意p指向常量字符串, 不能修改
	p[0] = 'X';            // 编译器不能发现该错误
	
	char a[] = "hello";
	char *p = a;
	p[0] = 'x';
	// printf("a==%s\n",a);//xello
	// printf("a==%s\n",p+1);//ello
	// printf("a==%s\n",*p+1);//段错误
	
```
- 内容复制与比较: 不能对数组名进行直接复制与比较
> 若想把数组a的内容复制给数组b，不能用语句 b = a ，否则将产生编译错误。应该用标准库函数strcpy进行复制。同理，比较b和a的内容是否相同，不能用if(b==a) 来判断，应该用标准库函数strcmp进行比较。
```c
	// 数组…
	char a[] = "hello";
	char b[10];
	strcpy(b, a);           // 不能用 b = a 直接赋值
	if(strcmp(b, a) == 0)   // 不能用 if (b == a) 直接比较
	
	// 指针…
	int len = strlen(a);
	char *p = (char *)malloc(sizeof(char)*(len+1)); // 地址为空 要先申请空间
	strcpy(p,a);            // 不要用 p = a;
	if(strcmp(p, a) == 0)   // 不要用 if (p == a)
```
- 计算内存容量：
用运算符sizeof可以计算出数组的容量（字节数）`sizeof(a)`（注意别忘了`\0`）。指针p指向a，但是sizeof(p)的值却是4。这是因为sizeof(p)得到的是一个指针变量的字节数，相当于`sizeof(char*)`，而不是p所指的内存容量。C++/C语言没有办法知道指针所指的内存容量，除非在申请内存时记住它。

```c
char a[] = "hello world";
char *p  = a;
sizeof(a);   // 12字节
sizeof(p);   // 4字节 指针

void Func(char a[100])
{
	sizeof(a);   // 4字节而不是100 字节 (指针)
}
```

> 注意当数组作为函数的参数进行传递时，该数组自动退化为同类型的指针


# 指针数组和数组指针的区别:
- 指针数组：它实际上是一个**数组**，数组的每个元素存放的是一个指针类型的元素。

> 元素表示：*a[i]   *(a[i])是一样的，因为[]优先级高于*

```c
	//优先级问题：[]的优先级比*高
	//说明arr是一个数组，而int*是数组里面的内容
	//这句话的意思就是：arr是一个含有5个int*的数组
	int* arr[5];

	// 举例
	char a[3][8]={"gain","much","strong"}; //二维数组，由三个一维数组够成
	char *n[3]={"gain","much","strong"};   //指针数组，由三个一维数组够成
	
	
```

- 数组指针：它实际上是一个**指针**，该指针指向一个数组; 这个指针存放着一个数组的首地址，或者说这个指针指向一个数组的首地址.

> 元素表示：(*a)[i]  

```c
	//由于[]的优先级比*高，因此在写数组指针的时候必须将*arr用括号括起来
	//arr先和*结合，说明arr是一个指针变量
	//这句话的意思就是：指针arr指向一个大小为5个整型的数组。
	int (*arr)[5];
```



测试用例：

```c
#include <stdio.h>
#include <string.h>

#define DOMIAN_MAX_COUNT 100 // 最大截取数量
#define SR ","               // 截取标识

/*
 domian: 待截取串
 ptr： 截取后的结果集
 返回截取后的个数
*/
int split_domain_list(char *domain, char **ptr)
{
    char *str2, *saveptr2;
  
//  局部变量使用后 ptr 外部使用会产生内存越界，因为函数内部变量已经释放，所以在外层拷贝备份
//  char query_log[512];
//  strcpy(query_log, domain)    

    int i, ptr_count;
    for (i = 0; i< DOMIAN_MAX_COUNT; i++) // 限制单次下发最多域名个数
        ptr[i] = NULL;
    for (i = 0 ,str2 = domain/*query_log*/; i < DOMIAN_MAX_COUNT ; i++, str2 = NULL)
    {
        ptr[i] = strtok_r(str2, SR, &saveptr2);
        if (ptr[i] == NULL) //不能使用strlen判断，此处过滤掉空
            break;
        printf("ptr[%d] = %s\n",i,ptr[i]);
    }
    ptr_count = i;
    printf("ptr_count = %d; domain = %s\n",ptr_count,domain);

    return ptr_count;
}


int test_1(const char *name)
{
#if 1 // 正确操作
    // 针对二级指针求长度， 可以使用NULL 判断； 不可使用strlen
    if(name == NULL) {
        printf("name is NULL\n");
        return -1;
    }
#else // 错误操作
    // 针对二级指针求长度，会引起内存越界
    if(strlen(name) == 0)
    {
        return -1;
    }
#endif

    printf("1  %s\n",name);
    return 0;
}


int test_2(char **domain)
{
    if(*domain == NULL) {
        return -1;
    }
    printf("2  %s\n",*domain);
//    printf("%u\n",strlen(*domain)); // 内存越界 如果
    return 0;
}


int main()
{
    char domain[] = "www.1.com,,www.3.com,www.4.com,www.5.com";
    char *str0 = "c.biancheng.net";
    char *str[3] = {
        "aaaaa",
        "bbbbb",
        "ccccc"
    };  

    printf("str0 = %u---%s\n",strlen(str0),str0);
    printf("str[1] = %u---%s\n",strlen(str[1]),str[1]);

    // 防止源串被破坏
    char query_log[512];
    strcpy(query_log, domain);

    int i = 0;
  	// 指针数组，数组里面存的指针
    char *ptr[128];
  	// int split_domain_list(char *domain, char **ptr)
    int count = split_domain_list(query_log,ptr);// 不需要对ptr在取地址
    for(i=0;i<count;i++) {
        printf("ptr[%d] = %s\n",i,ptr[i]);
        // 两种方法
        test_1(ptr[i]);
        test_2(&ptr[i]);
    }

    return 0;
}
```





# 指针移位:
```c
int main()
{
    int arr[]={6,7,8,9,10};
    int *p = arr;
    *(p++)+=123; //先指针进行运算，然后指针后移, 此时指向元素1 ，6 + 123，后指针移位
    //同 *p+++=123;
    // 129 7
    printf("%d-%d\n", arr[0], *p);
    
    // 8__8  先指针自加后移，然后取值
    printf("%d-%d\n",*p,*(++p));
    
    // 8__7__129
    printf("%d____%d___%d\n",*p,*(p-1),*(p-2));
}
```



# 函数指针:

```c
//  定义方法一
typedef void* (*func_array_1)(void *arg);
// 方法一初始化
//func_array_1 fun[4] = {aa,bb,cc,dd};

// 定义方法二
void *(*func_array_2[4])(void *) = {aa,bb,cc,dd};
// 方法二初始化
//func_array_2[i](buf[i]);


void *aa(void *arg)
{
    printf("func %s\n",arg);
    return NULL;
}

char *buf[10] = {
    "aa",
    "bb",
    "cc",
    "dd"
};

int main() {
    func_array_1 fun[4] = {aa,bb,cc,dd};
    for (i = 0; i < count; i++) {
        fun[i](buf[i]);
    }

    for (i = 0; i < count; i++) {
        func_array_2[i](buf[i]);
    }
    return 0;
}

```



# 结构体传参：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <unistd.h>
#include <time.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <arpa/inet.h>
#include <pthread.h>
#include <errno.h>
#include <regex.h>
#include <pthread.h>

typedef struct udp{
    char buf[20];
    int fd;
    struct sockaddr_in serveraddr;
}Udp;

int test(struct udp *server)
{
    int fd = (server->fd);
    char *buf = server->buf;
//    struct sockaddr_in *serveraddr = &server->serveraddr;
    printf("%d  %s\n",fd,buf);
  
    server->fd = 4;
}
/*
* 因为在外部已经是指针，取地址传递为 指针传递 改变
*
* */
int init(struct udp *server)
{
    server->fd = 2;
    strcpy(server->buf,"aaaaaa");
}


/*
* 只有函数内部改变 值传递
*
* */
int init_test(struct udp *server)
{
    server = (struct udp*)malloc(sizeof(struct udp));
    server->fd = 3;
    strcpy(server->buf,"bbbbbbb");
    printf("fd = %d;buf = %s\n",server->fd,server->buf);
    return 0;
}
/*
* 参数 外部使用依然未改变
* 返回值 外部使用 可以改变
* 举例: node = init_test_2(node);
* */
struct udp* init_test_2(struct udp *server)
{
    server = (struct udp*)malloc(sizeof(struct udp));
    server->fd = 3;
    strcpy(server->buf,"bbbbbbb");
    printf("fd = %d;buf = %s\n",server->fd,server->buf);
    return server;
}
/*
* 二级指针传递
* 赋值时 (*server)->成员
* 不能使用 *server->成员 或 *(server)->成员
* */
int init_test_3(struct udp **server)
{
    *server = (struct udp*)malloc(sizeof(struct udp));
    (*server)->fd = 3;
    strcpy((*server)->buf,"bbbbbbb");
//    printf("fd = %d\tbuf = %s\n",(*server)->fd,(*server)->buf);
    return 0;
}

int main()
{
    Udp server;
// 	不能这么使用 应该用struct udp *node = NULL;（和 Udp 的区别）
//  udp *node = NULL;  
    Udp *node = NULL;
    memset(&server,0x0,sizeof(struct udp));
    init(&server);
  	// server.fd = 2   buf= aaaaaa
    printf("server.fd = %d  buf= %s\n",server.fd,server.buf);

//  init_test(node);
//  printf("server.fd = %d  buf= %s\n",server.fd,server.buf);
//    node = init_test_2(node);
//  printf("server.fd = %d  buf= %s\n",server.fd,server.buf);
  
    init_test_3(&node);
    if(node)
      	// node->fd = 3    buf= bbbbbbb
        printf("node->fd = %d  buf= %s\n",node->fd,node->buf);
    else
        printf("node is NULL\n");

    test(&server);
		// server.fd = 4   buf= aaaaaa
    printf("server.fd = %d  buf= %s\n",server.fd, server.buf);
}
```





# 自定义回调函数：

```c
#include <stdio.h>

int Print()
{
	int a = 100;
	printf("print is %d\n",a);
	return 0;
}

int Add(int a,int b)
{
	int sum;
	sum = a + b;
	printf("sum is %d\n",sum);
	return 0;
}

void callback_Print(int (*ptr)())
{
	(*ptr)();
}

void callback_Add(int (*ptr)(int x,int y),int a,int b)
{
	(*ptr)(a,b);
}

int main()
{
	callback_Print(Print);
	callback_Add(Add,10,20);  
	return 0;
}
```

# select、poll、epoll 区别

## select
定义内核的数组大小 `1024`，通过一个 `bit` 代表一个 `fd` 描述 `fd` 集合. 在使用过程中存在 `fd_set` 的拷贝，同时它是轮询监听这些 `fd`.

```c
#undef __FD_SETSIZE
#define __FD_SETSIZE  1024

typedef struct {
  unsigned long fds_bits[__FD_SETSIZE / (8 * sizeof(long))];
} __kernel_fd_set;
typedef __kernel_fd_set    fd_set;
```

### 注册监听事件
监听的 `fd` 为 3，将 3 添加到 `fd_set` 集合时，就会将 `fd_set` 置为 `0000,0100` ，如果再将为 5 的 `fd` 也添加到集合，集合会置为 `0001,0100` 

### 触发监听事件
比如 `fd` 为 5 的事件触发了，此时 `select` 会将 `fd_set` 集合返回用户态该值被置为 `0001,0000`，只有 `fd` 为 5 的位被保留了，其它没有事件触发的位被清零了。这也就是为什么每次 `select` 都要将关心的 `fd` 重新添加到集合。


开始时将 `fd_set` 集合中感兴趣 `fd` 对应的第 `fd` 个 `bit` 位置1，然后将该 `fd_set` 集合交给内核监听，一旦有事件触发，就保留对应的 `bit` 位，其没有事件触发的 `bit` 位清零，然后返回用户态进行事件处理，这里要注意，处理完毕后要再次将感兴趣 `fd` 对应的 `bit` 位重新置位，因为之前没有事件触发的bit位已经被清零了。

> 会先将需要监听的 `fd_set` 集合拷贝到对应的栈上，将需要返回用户空间的 `fds` 集合空间清零; 然后将 `fds` 结果返回用户空间触发监听事件，再经过一系列对 `fd_set` 判断回填。


## poll
通过链表解决了select的1024个文件描述符的限制，缺点依然是需要在内核和用户态间来回拷贝文件描述符集合和pollfd集合，这对监听大量fd时会造成性能和效率的严重损耗


## epoll

### 触发方式
- ET(edge-triggered) 边缘触发： 当有读写等事件到达后epoll会通知**一次**，如果用户态不进行处理，就只能等到该fd上下次再有读写事件时才会触发该fd上的读写事件。
- LT(level-triggered) 水平触发： 当有读写等事件到达后epoll会通知用户态，如果用户不进行处理，epoll还会继续通知用户该fd上有事件发生，直到用户将fd上的事件处理完。

> 对于读写的文件描述符在`ET`模式下一定要设置成非阻塞模式而且一定要一次性要可读写的数据处理完毕，否则可能会对下一次的事件读写产生影响。对于监听的`fd`最好设置成`LT`模式，如果设置成`ET`模式在高并发情况下极有可能会有连接`fd`连接不成功

```c
//调用epoll_ctl时就会创建epitem,用来存放io的事件
struct epitem {
  //红黑树的节点
  union {
    struct rb_node rbn;
    struct rcu_head rcu;
  };
  struct list_head rdllink;//有事件触发时就将对应的epitem挂到该链表
  struct epitem *next;
  struct epoll_filefd ffd;//item涉及到的文件描述符信息
  int nwait;//poll操作时监听等待队列的数量
  struct list_head pwqlist;//poll等待队列
  struct eventpoll *ep;//ep节点
  struct list_head fllink;
  struct wakeup_source __rcu *ws;
  struct epoll_event event;
};
```

数据结构优化 ：
- 利用红黑树链表降低全量拷贝

用户态、内核态切换：
- 用户空间过来的监听fd和事件我们进行了一次拷贝，之后就不再拷贝
- 只有就绪链表不为空则会发生向用户空间的数据拷贝


epoll是不受限于1024个文件描述符的，且在epoll_wait返回后无需再次将监听fd添加到内核，在返回就绪事件时epoll是通过拷贝就绪链表来完成的，这样返回到用户态的数据只有就绪事件。

epoll适用于连接量非常多，但是活跃的比较少
- epoll在内核态维护一颗红黑树和非常多的等待队列、链表，这在资源消耗上是比较大的，如果监听的数量不多，那对于底层是消耗比较大的
- epoll中注册了回调，一旦有事件触发就会调用回调并将该fd结构挂载到就绪链表，那么如果活跃的数量太多了，那这一层层的回调，将入就绪链表，这对服务器的消耗还是比较大的

以通过ET模式来解决epoll_wait的惊群


## 三者对比：
- select有1024个描述符的限制，同时select和poll又都是将用户态监听的文件描述符集合拷贝到内核态，内核态将监听的文件描述符状态修改后再拷贝回用户态进行处理，处理完后，用户程序要重新将需要监听的文件描述符再次加入select或poll
- poll自身的设计是在栈上分配32个描述符监听，如果再增加监听描述符，poll会从堆上进行分配，这样虽然避免了1024的限制，但是内存分配会非常损耗性能，但是采用select也有一定好处就是开发简单，跨平台容易，windows、linux、mac都是支持select的。
- epoll不需要遍历所有描述符，通过红黑树在内核管理监听的文件描述符集合，而且只用添加一次，不需要来回拷贝所有的文件描述符，内核只把就绪链表上的节点拷贝到用户态

所以，当连接量相对小的时候我们可以选用select来处理，开发难度低，性能损耗内存占用也不是很大，如果连接量比较大，我们最好选用epoll来解决，如果此时活跃量也很大，epoll可能就不是很好的解决方式了



























