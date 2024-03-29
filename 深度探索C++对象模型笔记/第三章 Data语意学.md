### 1、一个实例引起的思考

~~~C++
class X{};
class Y:virtual public X{};
class Z:virtual public X{};
class A:public Y, public Z{};
~~~

Lippman的一个法国读者的结果是：

```C++
sizeof X yielded 1                         
sizeof Y yielded 8                         
sizeof Z yielded 8                         
sizeof A yielded 12 
```

而我在VS2019，win10，64bit上以及Linux环境下的Ubuntu下输出都是

~~~C++
sizeof X yielded 1   
sizeof Y yielded 8  
sizeof Z yielded 8    
sizeof Z yielded 16   
~~~





### C++类的大小计算汇总，这篇文章讲的非常好：

https://www.cnblogs.com/fengyaoyao/p/10262312.html

先贴几个重要结论：

- **类大小的计算，遵循结构体的对齐原则；**
- **类的大小，与普通数据成员有关，与成员函数和静态成员无关。即普通成员函数、静态成员函数、静态数据成员、静态常量数据成员，均对类的大小无影响；**
- **虚函数对类的大小有影响，是因为虚函数表指针带来的影响；**
- **虚继承对类的大小有影响，是因为虚基表指针带来的影响；**
- **静态数据成员之所以不计算在类的对象大小内，是因为类的静态数据成员被该类所有的对象所共享，并不属于具体哪个对象，静态数据成员定义在内存的全局区；**

1、空类的大小

C++的空类是指这个类不带任何数据，即类中没有非静态(non-static)数据成员变量，没有虚函数(virtual function)，也没有虚基类(virtual base class)。

　　直观地看，空类对象不使用任何空间，因为没有任何隶属对象的数据需要存储。然而，C++标准规定，凡是一个独立的(非附属)对象都必须具有非零大小。换句话说，c++空类的大小不为0 。

计算结果1。

　　C++标准指出，不允许一个对象（当然包括类对象）的大小为0，不同的对象不能具有相同的地址。

　　这是由于：

- new需要分配不同的内存地址，不能分配内存大小为0的空间；
- 避免除以 sizeof(T)时得到除以0错误；

　　故使用一个字节来区分空类。

　　但是，有两种情况值得我们注意

　　第一种情况，空类的继承： 

　　当派生类继承空类后，派生类如果有自己的数据成员，而空基类的一个字节并不会加到派生类中去。sizeof(D)为4。

　　第二种情况，一个类包含一个空类对象数据成员：

sizeof(HoldsAnInt)为8。 

　　在这种情况下，空类的1字节是会被计算进去的。而又由于字节对齐的原则，所以结果为4+4=8。

　　继承空类的派生类，如果派生类也为空类，大小也都为1。

2、含有虚函数成员的类

　　虚函数（Virtual Function）是通过一张虚函数表（Virtual Table）来实现的。编译器必需要保证虚函数表的指针存在于对象实例中最前面的位置（这是为了保证正确取到虚函数的偏移量）。

　　每当创建一个包含有虚函数的类或从包含有虚函数的类派生一个类时，编译器就会为这个类创建一个虚函数表（VTABLE）保存该类所有虚函数的地址，其实这个VTABLE的作用就是保存自己类中所有虚函数的地址，可以把VTABLE形象地看成一个函数指针数组，这个数组的每个元素存放的就是虚函数的地址。在每个带有虚函数的类中，编译器秘密地置入一指针，称为vpointer（缩写为VPTR），指向这个对象的VTABLE。 当构造该派生类对象时，其成员VPTR被初始化指向该派生类的VTABLE。**所以可以认为VTABLE是该类的所有对象共有的，在定义该类时被初始化；而VPTR则是每个类对象都有独立一份的，且在该类对象被构造时被初始化。** 

3、基类含有虚函数的继承

1）虚函数按照其声明顺序放于表中。

2）基类的虚函数在派生类的虚函数前面。

1）覆盖的f()函数被放到了虚表中原来基类虚函数的位置；

2）没有被覆盖的函数依旧；

3）派生类的大小仍是基类和派生类的非静态数据成员的大小+一个vptr指针的大小；

~~~C++
#include<iostream>
using namespace std;
 
 
class A    
{    
};   
 
class B    
{ 
    char ch;    
    virtual void func0()  {  }  
};  
 
class C   
{ 
    char ch1; 
    char ch2; 
    virtual void func()  {  }   
    virtual void func1()  {  }  
}; 
 
class D: public A, public C 
{    
    int d;    
    virtual void func()  {  }  
    virtual void func1()  {  } 
};    
class E: public B, public C 
{    
    int e;    
    virtual void func0()  {  }  
    virtual void func1()  {  } 
}; 
 
int main(void) 
{ 
    cout<<"A="<<sizeof(A)<<endl;    //result=1 
    cout<<"B="<<sizeof(B)<<endl;    //result=16     
    cout<<"C="<<sizeof(C)<<endl;    //result=16 
    cout<<"D="<<sizeof(D)<<endl;    //result=16 
    cout<<"E="<<sizeof(E)<<endl;    //result=32 
    return 0; 
}
~~~

　　结果分析: 

　　1.A为空类，所以大小为1 ；

　　2.B的大小为char数据成员大小+vptr指针大小。由于字节对齐,大小为8+8=16 ；

　　3.C的大小为两个char数据成员大小+vptr指针大小。由于字节对齐，大小为8+8=16 ；

　　4.D为多继承派生类，由于D有数据成员，所以继承空类A时，空类A的大小1字节并没有计入当中，D继承C，此情况D只需要一个vptr指针，所以大小为数据成员加一个指针大小。由于字节对齐,大小为8+8=16 ；

　　5.E为多继承派生类，此情况为我们上面所讲的多重继承，含虚函数覆盖的情况。此时大小计算为数据成员的大小+2个基类虚函数表指针大小 ，考虑字节对齐，继承顺序B在先，B(8 + 1)，然后是C(8+1+1)，由于字节对齐，B得与C中最大值对齐，因此B+7变成16，再+C（10），得26，最后+E的其它成员+1，因为要整体对于最大值（8）对齐，因此补齐得32。（之前看到几篇博客这里解释得都有问题）

3.虚继承的情况

　　对虚继承层次的对象的内存布局，在不同编译器实现有所区别。 

　　在这里，只说一下在gcc编译器下，虚继承大小的计算。它在gcc下实现比较简单，不管是否虚继承，GCC都是将虚表指针在整个继承关系中共享的，不共享的是指向虚基类的指针。

~~~C++
class A {
 
    int a;　virtual void myfuncA(){}
};
 
class B:virtual public A{
 
    virtual void myfunB(){}
 
};
 
class C:virtual public A{
 
    virtual void myfunC(){}
 
};
 
class D:public B,public C{
 
    virtual void myfunD(){}
 
};
~~~

sizeof(A)=16，sizeof(B)=24，sizeof(C)=24，sizeof(D)=48；

　　A的大小为int大小加上虚表指针大小;

　　B，C中由于是虚继承，因此大小为int大小加指向虚基类的指针的大小。B,C虽然加入了自己的虚函数，但是虚表指针是和基类共享的，因此不会有自己的虚表指针，他们两个共用虚基类A的虚表指针。D由于B，C都是虚继承，其大小等于B+C）。





### 2、在VC中数据成员的布局顺序为：

1. vptr部分（如果基类有，则继承基类的）
2. vbptr （如果需要）
3. 基类成员（按声明顺序）
4. 自身数据成员
5. 虚基类数据成员（按声明顺序）

### 3、虚拟成员函数

如果function()是一个虚拟函数，那么用指针或引用进行的调用将发生一点特别的转换——一个中间层被引入进来。例如：

~~~C++
// p->function()   将转化为
(*p->vptr[1])(p);
~~~

- 其中vptr为指向虚函数表的指针，它由编译器产生。vptr也要进行名字处理，因为一个继承体系可能有多个vptr。

- 1是虚函数在虚函数表中的索引，通过它关联到虚函数function().

  

何时发生这种转换？**答案是在必需的时候**——一个再熟悉不过的答案。当通过指针调用的时候，要调用的函数实体无法在编译期决定，必需待到执行期才能获得，所以上面引入一个间接层的转换必不可少。但是当我们通过对象（不是引用，也不是指针）来调用的时候，进行上面的转换就显得多余了，因为在编译器要调用的函数实体已经被决定。此时调用发生的转换，与一个非静态成员函数(Nonstatic Member Functions)调用发生的转换一致。

### 4、静态成员函数

静态成员函数的一些特性：

1. 不能够直接存取其类中的非静态成员（nostatic members），包括不能调用非静态
   成员函数(Nonstatic Member Functions)。
2. 不能够声明为 const、voliatile或virtual。
3. 它不需经由对象调用，当然，通过对象调用也被允许。

除了缺乏一个this指针他与非静态成员函数没有太大的差别。在这里通过对象调用和通过指针或引用调用，将被转化为同样的调用代码。

需要注意的是通过一个表达式或函数对静态成员函数进行调用，被C++ Standard要求对表达式进行求值。如：(a+=b).static_fuc();

虽然省去对a+b求值对于static_fuc()的调用并没有影响，但是程序员肯定会认为表达式a+=b已经执行，一旦编译器为了效率省去了这一步，很难说会浪费多少程序员多少时间。这无疑是一个明智的规定。

