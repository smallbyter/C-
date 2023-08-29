# C++笔记 part 2

## 一、动态内存

程序中使用的对象都有严格定义的生存期

- 全局对象在程序启动时分配，程序结束时销毁
- 局部自动对象在进入定义其所在的程序块时被创建，离开时被销毁
- 局部static对象在第一次使用前分配，程序结束时销毁

C++支持动态分配对象，这种对象的生存期和它们在哪创建无关，只有显示被释放时，这些对象才会销毁

程序除了静态内存和栈内存，还拥有一个内存池，这部分内存叫作**自由空间(free store)或堆（heap）**

程序用堆来存储动态分配的对象，即在程序运行时分配的对象，它们的生存周期由程序控制

程序中使用的对象都有严格定义的生存期

- 全局对象在程序启动时分配，程序结束时销毁
- 局部自动对象在进入定义其所在的程序块时被创建，离开时被销毁
- 局部static对象在第一次使用前分配，程序结束时销毁

C++支持动态分配对象，这种对象的生存期和它们在哪创建无关，只有显示被释放时，这些对象才会销毁

程序除了静态内存和栈内存，还拥有一个内存池，这部分内存叫作**自由空间(free store)或堆（heap）**

程序用堆来存储动态分配的对象，即在程序运行时分配的对象，它们的生存周期由程序控制

### 1、动态内存与智能指针

动态内存管理通过一对运算符来完成：

- **new**  在动态内存中为对象分配空间并返回一个指向该对象的指针，我们可以选择对对象进行初始化
- **delete** 接受一个动态对象的指针，销毁该对象，并释放与之关联的内存

动态内存的使用很容易出问题，因为确保在正确的时间释放内存很困难。

忘记释放内存就会产生**内存泄漏（内存泄漏（Memory Leak）是指程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。）**

有时在尚有指针引用内存的情况下我们就释放了它，这会产生**引用非法内存的指针**

为了更安全地使用动态对象，**标准库定义了两种智能指针（smart pointer）类型来管理动态分配的对象**

智能指针的行为类似常规指针，**区别是它负责自动释放所指的对象**

**这两种智能指针的区别在于管理底层指针的方式：**

- `shared_ptr`允许多个指针指向同一对象
- `unique_ptr`则独占所指向的对象

标准库还定义了叫作`weak_ptr`的伴随类，它是一种弱引用，指向`shared_ptr`所管理的对象

**这三种类型都定义在memory头文件中**

#### `shared_ptr`类

类似vector，智能指针也是模板，因此创建一个智能指针时，必须提供额外的信息（指针可以指向的类型）

**默认智能指针中保存着空指针**

```c++
shared_ptr<string> p1; //指向string
shared_ptr<list<int>> p2; //指向int的list

//若p1不为空，检查它是否指向一个空string
if(p1 && p1->empty())
    *p1 = "hi";  //若p1指向一个空string，解引用p1，将新值赋予string
```

![](D:\office\word\计算机基础\C++\截图2\智能指针操作.jpg)

![](D:\office\word\计算机基础\C++\截图2\shared_ptr独有操作.jpg)

##### make_shared函数

定义在头文件memory中，它是最安全的分配和使用动态内存的方法

**它在动态内存中分配一个对象并初始化它，返回指向此对象的`shared_ptr`**

使用make_shared函数时，**必须指定想要创建的对象的类型，它用其参数来构造给定类型的对象，比如调用`make_shared<string>`时传递的参数必须与string的某个构造函数匹配，类似之前的emplace**

```c++
shared_ptr<int> p = make_shared<int>(42);

//p2指向值为10个9的string
shared_ptr<string> p2 = make_shared<string>(10,'9');

//指向一个值初始化的int，即值为0
shared_ptr<int> p3 = make_shared<int>();

auto p4 = make_shared<vector<string>>();
```

##### `shared_ptr`的拷贝和赋值

进行拷贝和赋值操作时，每个`shared_ptr`都会记录有多少个其他`shared_ptr`指向相同的对象

每个`shared_ptr`都有一个关联的计数器，通常叫作**引用计数**。

无论何时拷贝一个此对象时，计数器都会递增

当给一个`shared_ptr`赋予新值或它被销毁时，计数器会递减

```c++
auto p = make_shared<int>(42); //p指向的对象只有p一个引用着
auto q(p); //p和q指向相同对象，此对象有两个引用者
```

一个`shared_ptr`的计数器变为0，他它就会自动释放自己所管理的动态内存对象

```c++
auto r = make_shared<int> 42; //r指向的int只有一个引用者

//递增q指向的对象的引用计数，递减r原来指向的对象的引用计数
//此时r原来指向的对象已经没有引用者，会自动释放
r = q; //给r赋值，使他指向另一个地址

```

当指向一个对象的最后一个`shared_ptr`被销毁时，`shared_ptr`类会自动销毁此对象，它通过析构函数（destructor）完成销毁。析构函数每个类都有一个，控制着此类型的对象销毁时做什么操作，一般用来释放对象所分配的资源

析构函数会递减它所指向对象的引用计数，若变为0，则它会销毁对象并自动释放它占用的内存

```c++
shared_ptr<int> factory(int arg){
	//...进行处理
    return make_shared<int>(arg);
}

void use1(int arg){
    shared_ptr<int> p = factory(arg);
}  //p离开作用域，指向的内存会自动释放

shared_ptr<int> use2(int arg){
    shared_ptr<int> p = factory(arg);
    return p;
} //p离开作用域，指向的内存不会被释放，因为这里向此函数的调用者返回p的拷贝
```

##### 使用了动态生存期的资源的类

程序使用动态内存的原因：

- 程序不知道自己需要使用多少对象，比如容器类就因此使用动态内存
- 程序不知道所需对象的准确类型
- 程序需要在多个对象间共享数据

**例：定义一个StrBlob类，允许它在多个对象间共享数据**

之前类中分配的资源都与对应对象的生存期一致，而此类中分配的资源具有与原对象独立的生存期

```c++
class StrBlob{
  public:
    typedef vector<string>::size_type size_type;
    StrBlob();
    StrBlob(initializer_list<string> il); //在调用它传入实参时可以使用花括号列表
    size_type size const {return data->size();}
    bool empty() const{return data->empty();}
    
    //添加和删除元素
    void push_back(const string &t) {data->push_back(t);}
    void pop_back();
    
    //元素访问
    string& front();
    string& back();
  private:
    shared_ptr<vector<string>> data;
    //若data[i]不合法，则抛出异常
    void check(size_type i ,const string &msg) const;
    
};

//构造函数
StrBlob::StrBlob(): data(make_shared<vector<string>>()) {}
strBlob::StrBlob(initializer_list<string> il): 
				data(make_shared<vector<string>>(il)){}

//元素访问的成员函数
//访问之前必须检查元素是否存在
void StrBlob::check(size_type i,const string &msg) const{
    if(i >= data->size())
        throw out_of_range(msg);
}

string& StrBlob::front(){
    //若vector为空，抛出异常
    check(0,"front on empty StrBlob");
    return data->front();
}
string& Strblob::back(){
    //若vector为空，抛出异常
    check(0,"front on empty StrBlob");
    return data->back();
}
void StrBlob::pop_back(){
    //若vector为空，抛出异常
    check(0,"front on empty StrBlob");
    data->pop.back();
}

```

StrBlob使用默认版本的拷贝、赋值和析构成员函数。本类只有一个数据成员data，它是`shared_ptr`类型

#### 直接管理内存

**使用运算符new分配内存，delete释放new分配的内存**

在自由空间分配的内存是**无名的**，因此new无法为其分配的对象命名，而是**返回一个指向该对象的指针**：

```c++
int *pi = new int; //pi指向一个动态分配的、未初始化的无名对象
```

默认情况下，动态分配的对象是默认初始化的，**即内置类型或组合类型的对象是未定义的，类类型对象将用默认构造函数进行初始化**

对于依赖于编译器合成的默认构造函数的内置类型成员，若未在类内被初始化，则它们的值未定义

```c++
string *ps = new string; //初始化为空string
int *pi = new int; //pi指向一个未初始化的int
int *pi2 = new int();  //值初始化为0

int *pi = new int(1024); //pi指向的对象值为1024
string *ps = new string(3,'9'); //ps指向“999”
vector<int> *pv = new vector<int>{1,2,3,4}; //vector有4个int元素
```

若提供了一个**括号包围的初始化器**，就可以使用**auto**从它来推断我们想分配的对象的类型。**但是由于要根据初始化器的类型来判断要分配的类型，括号中只能有单一初始化器**

```c++
auto p = new auto(obj); //p指向一个和obj类型相同的对象，该对象用obj进行初始化

auto p2 = new auto{a,b,c}; //错误！！！括号中只能有单个初始化器
```

**动态分配的const对象**

由于分配的对象是const的，new返回的指针是一个指向const的指针

```c++
const int *p = new const int(10);
const string *p = new const string;//默认初始化一个const的空string
```

**内存耗尽**

自由空间是有可能被耗尽的，一旦无可用的内存，则new表达式会失败

**默认情况下若new不能分配所要求的内存空间，会抛出一个类型为`bad_alloc`的异常**

可以改变使用new的方式来阻止它抛出异常，这叫作**定位new**

定位new表达式允许向new传递额外的参数，这里传递给它一个标准库定义的nothrow对象，代表它不能抛出异常，此时不能分配内存则会返回空指针

**`bad_alloc和nothrow`都定义在头文件new中**

```c++
int *p = new int; //若分配失败，new抛出std::bad_alloc
int *p = new (nothrow) int; //若分配失败，new返回一个空指针。
```

**释放动态内存**

通过**delete表达式**将动态内存归还系统，它接收一个指针，指向需要释放的对象

delete表达式**会销毁给定的指针指向的对象，释放对应的内存**

**传递给p的指针必须指向动态分配的内存或是空指针，释放一块非new分配的内存或将相同的指针值释放多次都是未定义的，但是编译器不能分辨**

**const对象的值虽不能改变，但它本身是可以被销毁的**

```c++
delete p; //p指向一个动态分配的对象或是空指针
double *pd = new double(1.0), *pd2 = pd;
delete pd;
delete pd2; //未定义：pd2指向的内存已经释放了

const int *p = new const int(10);
delete p; 
```

**动态对象的生存期直到被释放为止**

```c++
Foo* factory(T arg);
{
    return new Foo(arg);
}

void use(T arg){
    Foo *p = factory(arg);
} //p被销毁，但是它指向的内存未被释放！！！

void use(T arg){
    Foo *p = factory(arg); //p是指向这块动态内存的唯一指针
    delete p; //释放内存                                                                                                                   
}
```

**动态内存管理容易出错的三个问题：**

- 忘记delete内存会导致内存泄漏，这种内存永远不可能被归还给自由空间了。查找内存泄漏十分困难，只有等内存耗尽时才能检测到这种错误

- 我们可能会使用已经释放掉的对象。通过释放内存后将指针置为空有时可以检测出这种错误
- 同一内存释放两次，当两个指针指向同一动态内存对象时，若通过一个指针delete之后，再用另一指针delete，自由空间可能会被破坏

**注：坚持使用智能指针，就可避免这些问题**

**delete之后重置指针值**

当delete一个指针后，指针值就变为无效了。但是很多机器上指针仍然保存着已经释放了的动态内存地址，这时指针就变成了**空悬指针，指向一块曾经保存数据对象但现在已经无效的内存**

**避免空悬指针：在指针即将离开其作用域之前释放它所关联的内存**。若要保留指针，可以在delete之后将nullptr赋予指针，这样就清除地指出指针不指向任何对象

动态内存的基本问题是可能有多个指针指向相同的内存，delete之后重置指针的方法只对这个指针有效

```c++
int *p(new int(42)); //p指向动态内存
auto q = p; //p和q指向相同的内存
delete p;  //p和q均变为无效
p = nullptr;  //指出p不再绑定到任何对象
```

#### `shared_ptr`和new结合使用

使用new返回的指针来初始化智能指针

**接收指针参数的智能指针构造函数是explicit的，因此不能将内置指针隐式转换为类类型的智能指针，必须使用直接初始化形式（不能使用=号，即不能使用拷贝初始化）**

```c++
shared_ptr<double> p; //空指针
shared_ptr<int> p(new int(42));  //p指向一个值为42的int

//必须使用直接舒适化形式
shared_ptr<int> p = new int(10); //错误！！不能使用拷贝初始化
shared_ptr<int> p(new int(10));

shared_ptr<int> clone(int p){
    return new int(p); //错误！！！不能隐式转换
}

shared_ptr<int> clone(int p){
    return shared_ptr<int>(new int(p));
}
```

**默认一个用来初始化智能指针的普通指针必须指向动态内存，因为智能指针默认使用delete释放它所关联的对象**

**可以将智能指针绑定到一个指向其他类型资源的指针上，这样就必须提供自己的操作来替代delete**

![](D:\office\word\计算机基础\C++\截图2\定义和改变shared_ptr的其他方法1.jpg)

![](D:\office\word\计算机基础\C++\截图2\定义和改变shared_ptr的其他方法2.jpg)

##### 不要混合使用普通指针和智能指针

`shared_ptr`可以协调对象的析构，但仅限于其自身的拷贝之间，因此推荐使用make_shared而不是new，能够在分配对象时就将`shared_ptr`与之绑定，**避免了无意中将同一块内存绑定到多个独立创建的shared_ptr上**

```c++
//这里是值传递，ptr是一个类类型
//实参被拷贝到ptr中，拷贝一个shared_ptr会增加引用计数，因此在process运行过程中，引用计数值至少为2
//process结束是时，ptr引用计数减1，但不会为0
void process(shared_ptr<int> ptr){
    //...使用ptr
} //ptr离开作用域，被销毁

shared_ptr<int> p(new int(42));//引用计数为1
process(p);//引用计数为2
int i = *p; //引用计数为1

int *x(new int(10));
process(x);//错误！！！
//这里shared_ptr是临时量，调用所在的表达式结束临时量就销毁了，引用计数为0
process(shared_ptr<int>(x));;//合法，但是内存会被释放
int j = *x; //未定义！！x是空悬指针，继续指向已经释放的内存

```

**当将`shared_ptr`绑定到普通指针时，就将内存管理交给了这个`shared_ptr`，这时就不应该再用内置指针来访问这块内存了，不知道何时该对象会销毁**

##### 不能使用get初始化另一个智能指针或为其赋值

智能指针类型定义了get函数，**返回一个内置指针，指向智能指针管理的对象**

**get的使用场景：**

需要向不能使用智能指针的代码传递一个内置指针，**使用get返回的指针的代码不能delete此指针**

将另一个智能指针也绑定到get返回的指针上是错误的，但编译器不会给出错误信息

```c++
shared_ptr<int> p(new int(42));
int *q = p.get();

//新程序块
{
    shared_ptr<int>(q); //未定义：两个独立的shared_ptr指向相同的内存
}//程序块结束，此临时量被销毁，指向的内存被释放
int foo = *p; //未定义：p指向的内存已释放了，成为空悬指针
//当p被销毁时，还会发生二次delete
```

**注：这里的两个shared_ptr是相互独立创建的，彼此不是对方的拷贝，因此引用计数分别都是1**

```c++
p.reset(new int(1024)); //p指向了这个新对象

//reset和unique一起使用
if(!p.unique()){  //若引用计数不是1
    p.reset(new string(*p))； 
}
```

#### 智能指针和异常

异常处理程序能在异常发生后令程序继续，这种程序需要确保在异常发生后资源能被正确释放。使用智能指针是确保资源被释放的简单办法

若使用智能指针，即使程序块过早结束，智能指针也能确保在内存不再需要时将其释放

函数的退出要么是正常处理结束，要么是发生了异常。无论哪种情况，局部对象都会被销毁

```c++
void f(){
    shared_ptr<int> sp(new int(42));
    //抛出异常，且在f中未被捕获
}//函数结束时shared_ptr自动释放内存


//在delete之前发生异常，则内存永远不会被释放
void f(){
    int *ip = new int(42); //动态分配一个新对象
    //...抛出异常，且在f中未被捕获
    delete ip;
}
```

**智能指针和哑类**

包括标准库类在内的很多C++类都定义了析构函数，负责清理对象使用的资源

不是所有类都是这样良好定义的，特别是为C和C++两种语言设计的类，通常要求用户显式释放所使用的任何资源

 那些分配了资源，但没定义析构函数来释放这些资源的类可能会遇到与使用动态内存相同的错误——程序员忘记释放资源

在资源分配和释放之间发生了异常，程序也会发生资源泄漏

```c++
struct destination;  //表示正在连接什么
struct connection;  //使用连接所需的信息
connection connect(destination*);  //打开连接
void disconnect(connection);   //关闭给定连接
void f(destination &d){
    //获得一个连接，使用完需要关闭
    connection c = connect(&d);
    //...使用连接
    //若在f退出前忘记调用disconnect，就无法关闭c了
}
```

这里的connection无析构函数，无法自动关闭连接。可以使用shared_ptr来保证它正确关闭

**使用我们自己的释放操作**

默认情况下，`shared_ptr`假定它们指向的是动态内存，因此当shared_ptr被销毁时，它默认对它管理的指针进行delete操作

我们可以定义一个**删除器（deleter）**，这个函数可以代替delete，能够释放`shared_ptr`中的指针

```c++
void end_connection(connection *p){
    disconnect(*p);
}

void f(destination &d){    
    connection c = connect(&d);
    shared_ptr<connection> p(&c,end_connection);
    //使用连接
    //f退出时（即使是异常而退出），connection会被正确关闭
}
```

p被销毁时，它不会对自己保存的指针执行delete，而是调用end_connection，接下来它会调用disconnect，从而确保连接被关闭。即使发生异常，p同样会被销毁，从而关闭连接

**正确使用智能指针的规范：**

- 不使用相同的内置指针值初始化或reset多个智能指针

- 不delete get函数返回的指针

- 使用get函数时，当最后一个对应的智能指针销毁后，指针就无效了

- 若使用智能指针管理的资源不是new分配的内存，记住传递给它一个删除器

#### unique_ptr

unique_ptr拥有它所指向的对象，某个时刻只能有一个unique_ptr指向一个给定对象

**当unique_ptr被销毁时，它所指向的对象也被销毁**

![](D:\office\word\计算机基础\C++\截图2\unique_ptr操作.jpg)

定义`unique_ptr`时，需将其绑定到一个new返回的指针上

由于一个unique_ptr拥有它指向的对象，因此它不支持普通的拷贝或赋值

可以调用release或reset将指针的所有权从一个非（const）unique_ptr转移给另一个unique

```c++
unique_ptr<double> p1;//可以指向一个double
unique_ptr<int> p2(new int(42));

unique_ptr<string> p(new string("aaa"));
unique_ptr<string> p2(p1); //错误！！！它不支持拷贝
unique_ptr<string> p3;
p3 = p2; //错误！！！unique_ptr不支持赋值

unique_ptr<string> p2(p1.release()); //release将p1置为空，放弃指针的控制权并返回该指针
unique_ptr<string> p3(new string("bbb"));
//将所有权从p3转移给p2
p2.reset(p3.release()); //reset释放了p2原来指向的内存
```

release成员返回unique_ptr当前保存的指针并将其置为空，它返回的指针通常用来初始化另一个智能指针或给它赋值

```c++
p2.release(); //错误！！！p2丢失了指针，不会释放内存
auto p = p2.release(); //正确，但是必须记得delete(p)
```

##### 传递unique_ptr参数和返回unique_ptr

**不能拷贝unique_ptr的规则有一个例外，我们可以拷贝或赋值一个将要被销毁的unique_ptr**

以下两段代码编译器都知道要返回的对象将要被销毁，在此情况下编译器执行一种特殊的拷贝

```c++
//返回一个unique_ptr
unique_ptr<int> clone(int p){
    return unique_ptr<int>(new int(p));
}

//返回一个局部对象的拷贝
unique_ptr<int> clone(int p){
    unique_ptr<int> ret(new int(p));
    //...
    return ret;
}
```

##### 向unique_ptr传递删除器

unique_ptr也是默认情况下用delete释放它指向的对象，我们可以重载一个默认的删除器，这会影响unique_ptr类型以及如何构造该类型的对象

**与重载关联容器的比较操作类似，我们必须在尖括号中提供删除器类型**。在创建或reset一个这种unique_ptr类型的对象时必须提供一个指定类型的可调用对象（删除器）

```c++
//p指向一个类型为objT的对象，并使用一个类型为delT的对象释放它
//它会调用一个名为fcn的delT类型对象
unique_ptr<objT,delT> p(new objT,fcn);


//重写之前的连接程序
void f(destination &d){
    connection c = connect(&d);
    unique_ptr<connection,decltype(end_connection)*> p(&c,end_connection);
    //使用连接。。。
    //f退出时（即使由于异常退出），connection会被正确关闭
}
```

#### weak_ptr

它是一种不控制所指向对象生存期的智能指针，它指向由一个shared_ptr管理的对象。将weak_ptr绑定到shared_ptr不会改变其引用计数

一旦最后一个指向对象的shared_ptr被销毁，对象就会被释放

![](D:\office\word\计算机基础\C++\截图2\weak_ptr.jpg)

创建一个weak_ptr时，需要用一个shared_ptr来初始化它

**由于对象可能不存在，我们不能使用weak_ptr直接访问对象，而必须调用lock，它检查指向的对象是否存在，若存在，lock返回一个指向共享对象的shared_ptr。只要此shared_ptr存在，它指向的底层对象也会存在**

```c++
auto p = make_shared<int> (42);
weak_ptr<int> wp(p); //wp弱共享p,wp和p指向相同的对象,wp指向的对象可能被释放

if(shared_ptr<int> np = wp.lock()){  //np不为空则条件成立
    //在if中，np和p共享对象
}
```

可以利用weak_ptr不会影响生存期的特性，阻止用户访问一个不再存在的对象

### 2、动态数组

new和delete运算符一次分配/释放一个对象，但很多情况下需要一次为很多对象分配内存的功能，如vector和string都是连续内存中保存元素，容器重新分配时必须一次性为很多元素分配内存

C++语言和标准库提供了两种一次分配一个对象数组的办法。但是大多数应用应使用标准库容器而不是动态分配的数组，容器更简单，不容易出现内存管理错误并可能有更好的性能

分配动态数组的类必须定义自己版本的操作，在拷贝、赋值和销毁对象时管理所关联的内存

#### new和数组

在类型名后面跟一对方括号，在其中指明要分配的对象的数目，**new分配成功后返回指向第一个对象的指针**

方括号中的大小必须是整型，但不必是常量

```c++
//调用get_size确定分配多少个int
int *pia = new int[get_size()]; //pia指向第一个int

//使用表示数组类型的类型别名来分配
typedef int arrT[42]; //arrT表示42个int的数组类型
int *p = new arrT; //等价于int *p = new int[42];
```

**分配一个数组会得到一个元素类型的指针**

当new分配一个数组时，我们并未得到一个数组类型的对象，**而是得到一个数组元素类型的指针**

由于分配的内存不是一个数组类型，因此不能对动态数组调用begin或end，也不能使用范围for来处理

**初始化动态分配对象的数组**

默认情况下，new分配的对象都是默认初始化的，不管是单个分配还是数组中的

```c++
int *pia = new int[10]; //10个未初始化的int
int *pia2 = new int[10](); //值初始化为10个0
string *psa = new string[10]; //10个空string
string *psa = new string[10](); //10个空string

int *pia3 = new int[10]{0,1,2,3,4};//前面的元素用给定的值初始化，后面值初始化为0
```

**和单个分配时不同，不能用空括号给出舒适化器，这意味着不能用auto分配数组**

**动态分配空数组**

可以使用任意表达式来确定要分配对象的数目

```c++
size_t n = get_size();//返回需要的元素数目
int* p = new int[n]; //分配数组保存元素
```

当n为0时，仍然可以正常工作。我们虽然不能创建一个大小为0的静态数组对象，但是此时调用new[n]是合法的，**new会返回一个合法的非空指针，保证和new返回的其他任何指针都不同，类似于尾后指针**

```c++
char arr[0]; //错误！！！不能定义长度为0的数组
char *p = new char[0]; //正确！但是p不能解引用
```

**释放动态数组**

我们在delete时需要在指针前面加上一个空方括号，代表此指针指向了一个对象数组的第一个元素

第二条语句会销毁pa指向的数组中的元素，并释放对应的内存。**数组中的元素按照逆序销毁，从最后一个开始销毁**

```c++
delete p; //p必须指向一个动态分配的对象或为空
delete []pa; //pa必须指向一个动态分配的数组或为空
```

**智能指针和动态数组**

可以使用unique_ptr的一个版本来管理new分配的数组，必须在对象类型后面跟一对空括号

当销毁unique_ptr管理的指针时会自动调用delete[ ]

```c++
unique_ptr<int[]> up(new int[10]);
up.release(); //这里只是交出指针控制权，将up置空，并未销毁
```

![](D:\office\word\计算机基础\C++\截图2\指向数组的unique_ptr.jpg)

**当它指向数组时，不能使用点和箭头运算符。**

可以使用下标运算符访问元素

**shared_ptr不支持直接管理数组，若想使用必须提供自己定义的删除器**

**shared_ptr没有定义下标运算符，不支持指针的算术运算，必须使用get获取内置指针**

```c++
shared_ptr<int> sp(new int[10],[](int *p) {delelte []p;});
sp.reset(); //使用我们提供的lambda释放数组，它使用delete[]

for(size_t i = 0; i != 10; ++i){
    *(sp.get()+i) = i; //使用get来获取内置指针
}
```

#### allocator类

new有一些灵活性上的局限，它将内存分配和对象构造组合在一起。当分配单个对象时通常希望这样

但当分配一大块内存时，我们通常计划在这块内存上按需构造对象，这时将内存分配和对象构造组合在一起可能导致浪费

```c++
string *const p = new string[n]; //构造n个空string
string s;
string *q = p;//q指向第一个string
while(cin >> s && q != p+n)
    *q++ = s;
const size_t size = q-p;
delete []p;
```

这里初始分配了n个string，但是可能用不到这么多，只需要少量的就够了

**标准库allocator类定义了在头文件memory中，它帮助我们将内存分配和对象构造分离开来。它分配的内存是原始的、未构造的**

![](D:\office\word\计算机基础\C++\截图2\allocator类及其算法.jpg)

allocator是一个模板，必须指明这个allocator可以分配的对象类型。

当一个allocator对象分配内存时，它会根据给定的对象类型来确定恰当的内存大小和对齐位置

**为了使用allocate返回的内存，必须用construct构造对象**

**allocate()函数分配的元素都是未初始化的，返回指向分配对象的指针**

```c++
allocator<string> alloc; //可以分配string的allocator对象
auto const p = alloc.allocate(n); //分配n个未初始化的string
```

**construct成员函数在给定位置进行构造，它接收一个指针和额外的参数**

```
auto q = p;
alloc.construct(q++); //*q为空字符串
alloc.construct(q++,10,'c'); //*q为10个c
alloc.construct(q++,"aaa"); //*q为aaa
```

**在还未构造对象的情况下使用原始内存是错误的**

```c++
cout << *p <<endl;//正确：已经构造
cout << *q < endl; //错误：此时q指向未构造的内存
```

**当使用完对象后，必须对每个构造的元素调用destroy来销毁它们，destroy接收一个指针，对指向的对象执行析构函数**

**我们只能对真正构造了的元素进行destroy操作**

```c++
while(q != p) //q指向的是最后构造的元素之后的位置
    alloc.destroy(--q);
```

**一旦元素被销毁后，就可以重新使用这部分内存来保存其他string，也可以将其归还给系统。释放内存通过调用deallocate来实现**

传递给deallocate的指针不能为空，必须指向由allocate分配的内存，传递给它的大小必须和调用allocate分配内存时提供的大小一样

```c++
alloc.deallocate(p,n);
```

##### 拷贝和填充未初始化内存的算法

标准库为allocator类定义了两个伴随算法，**可以在未初始化内存中创建对象，它们都定义在头文件memory中**

![](D:\office\word\计算机基础\C++\截图2\allocaotr算法.jpg)

```c++
vector<int> vi; //假定其中有n个元素了
//分配比vi大一倍的动态内存
auto p = alloc.allocate(vi.size()*2);
//拷贝vi中的元素来构造从p开始的元素
auto q = uninitialized_copy(vi.begin(),vi.end(),p);
//构造剩余部分，初始化为42
uninitialized_fill_n(q,vi.size(),42);
```

**uninitialized_copy在给定目标位置构造元素，返回递增后的位置的迭代器**

### 3、例子：使用标准库完成文本查询程序

程序允许用户在给定文件中查询单词，查询结果是单词在文件中出现的次数及其所在行的列表

若一个单词在一行中出现多次，此行指列出一次，行会按照升序输出

#### 文本查询程序设计

开始一个程序设计的好方法是列出程序的操作，从而可以分析出我们需要什么样的数据结构

**文本查询程序的操作：**

读取输入文件时，必须记住单词出现的每一行，因此程序需逐行读取输入文件，并将每一行分解为独立单词

程序输出时，它必须能提取每个单词所关联的行号，行号必须升序出现且无重复；它必须能打印给定行号的文本

set保存每个单词在文本中出现的行号；map将每个单词和行号关联起来

```c++
class QueryResult; //先声明QueryResult
class TextQuery{
  public:
    using line_no = vector<string>::size_type;
    TextQuery(ifstream&);
    QueryResult query(const string&) const;
  private:
    shared_ptr<vector<string>> file;
    //每个单词到它所在行号的集合的映射
    map<string,shared_ptr<set<line_no>>> wm;    
};

TextQuery::TextQuery(ifstream &is):file(new vector<string>){
    string text;
    while(getline(is,text)){
        file->push_back(text);
        int n = file->size()-1; //当前行号
        istringstream line(text); //将文本分解为单词
        string word;
        while(line >> word){
            auto &lines = wm[word]; //lines是一个shared_ptr的引用，改变lines也会改变wm中的元素
            if(!lines)
                lines.reset(new set<line_no>);
            lines->insert(n);//将此行号插入set
        }
    }
}

class QueryResult{
    friend ostream &print(ostream&,const QueryResult&);
    public:
    	QueryResult(string s,
                   shared_ptr<set<line_no>> p,
                   shared_ptr<vector<string>> f):
    			sought(s),lines(p),file(f){ }
    private:
    	string sought; //查询单词
        shared_ptr<set<line_no>> lines;
	    shared_ptr<vector<string>> file;
    
};

QueryResult TextQuery::query(const string &sought) const{
    static shared_ptr<set<line_no>> nodata(new set<line_no>);
    auto loc = wm.find(sought);
    if(loc == wm.end())
        return QueryResult(sought,nodata,file);//未找到
    else
        return QueryResult(sought,loc->second,file);
}

ostream &print(ostream &os,const QueryResult &qr){
    os << qr.sought << "occurs" << qr.lines->size() << " "
        << make_plural(qr.lines->size(),"times","s") << endl;
    for(auto num : *qr.lines)
        os << "\t(line" << num+1 << ")"
           << *(qr.file->begin() + num) << endl;
    return os;
}
```

## 二、拷贝控制

拷贝和移动构造函数定义了用同类型的另一对象初始化本对象时做什么

拷贝和移动赋值运算符定义了将一个对象赋予同类型的另一对象时做什么

析构函数定义了此类型对象销毁时做什么

若一个类没有定义所有这些拷贝控制成员，编译器会自动为它定义缺失的操作。但不能完全依赖这些默认定义，它们可能并非我们所想

### 1、拷贝、赋值和销毁

#### 拷贝构造函数

**若一个构造函数的第一个参数是自身类型的引用，且任何额外参数都有默认值，则它是拷贝构造函数**

**第一个参数几乎都是const的引用，但是它也能接受非const引用**

**拷贝构造函数会被隐式使用，因此不能是explicit的**

```c++
class Foo{
  public:
    Foo(); //默认构造函数
    Foo(const Foo&); //拷贝构造函数
};
```

**合成拷贝构造函数**

若我们未定义一个拷贝构造函数，编译器会为我们定义一个

**即使我们定义了其他构造函数，编译器也会合成一个拷贝构造函数**

一般合成的拷贝构造函数会将**其参数的成员逐个拷贝到正在创建的对象中**，编译器从给定对象中一次将每个非static成员拷贝到正在创建的对象中来

**注：某些类的合成拷贝构造函数用来阻止拷贝该类类型的对象**

每个成员的类型决定了它如何拷贝：

类类型成员会使用它的拷贝构造函数来拷贝

内置类型的成员则直接拷贝

合成拷贝构造函数会逐元素拷贝一个数组类型的成员

```c++
//直接初始化
string dots(10,'.');
string s(dots); 

//拷贝初始化
string s2 = dots;
```

**直接初始化时我们使用与提供的参数最匹配的构造函数来完成；**

**拷贝初始化则要求编译器将右侧的运算对象拷贝到正在创建的对象中，通常使用拷贝构造函数（或移动构造函数）完成**

**拷贝初始化发生的情况：**

- 使用=定义变量
- 将一个对象作为实参传递给一个非引用类型的形参
- 从一个返回类型为非引用类型的函数返回一个对象
- 用花括号列表初始化一个数组中的元素或聚合类中的成员
- 某些类类型会它们所分配的对象使用拷贝初始化，比如初始化标准库容器或调用其insert或push成员时，容器会对其元素进行拷贝初始化；使用emplace成员创建的元素是直接初始化

**由于拷贝构造函数被用来初始化非引用类类型参数，拷贝构造函数自己的参数必须是引用类型。若不是，则调用永远也不会成功，为了调用拷贝构造函数，必须拷贝它的实参，而为了拷贝实参，又需要调用拷贝构造函数，如此无限循环**

**编译器可以绕过拷贝构造函数**

编译器在拷贝初始化时可以跳过拷贝/移动构造函数，直接创建对象

#### 拷贝赋值运算符

类可以控制它的对象如何赋值。

**若类未定义自己的拷贝赋值运算符，则编译器会为它合成一个**

```c++
Sales_data trans,accum;
trans = accum;  //使用该类的拷贝赋值运算符
```

##### 重载赋值运算符

**重载运算符**本质上是函数，**它的名字由operator关键字后接表示要定义的运算符的符号组成**

因此赋值运算符就是一个叫做**operator=**的函数，运算符函数也有一个返回类型和参数列表

重载运算符的参数表示运算符的运算对象，**某些运算符比如赋值运算符，必须定义为成员函数，这时它左侧的运算对象就绑定到隐式的this参数**

对于二元运算符来说，还是比如赋值运算符，**它右侧的运算对象作为显式参数传递，拷贝赋值运算符接收一个与其所在类相同类型的参数**

为了和内置类型的赋值保持一致，赋值运算符一般**返回一个指向其左侧运算对象的引用**

```c++
class Foo{
  public:
    Foo& operator=(const Foo&); //赋值运算符
};
```

**注：保存在容器中的类型要具有赋值运算符，且其返回值是左侧运算对象的引用**

##### 合成拷贝赋值运算符

若类未定义自己的拷贝赋值运算符，则编译器会生成一个**合成拷贝赋值运算符**，它会将右侧运算对象的每个非static成员赋予左侧运算对象对应的成员。对于数组类型成员，逐个赋值数组元素

**注：某些类中合成拷贝赋值运算符用来禁止该类对象的赋值**

```c++
//等价于合成拷贝赋值运算符
Sales_data& Sales_data ::operator=(const Sales_data &rhs){
    bookNo = rhs.bookNo;
    units_sold = rhs.units_sold;
    revenue = ths.revenue;
    return *this;
}
```

#### 析构函数

析构函数执行和构造函数相反的操作：**它释放对象使用的资源，并销毁对象的非static数据成员**

**析构函数是一个类的成员函数，名字由波浪号接类名构成，它无返回值，也不接受参数：**

**由于它不接受参数，因此不能重载，一个给定类只有唯一一个析构函数**

```c++
class Foo{
    public:
    	~Foo();  //析构函数
}；
```

**析构函数有一个函数体和一个析构部分，执行完函数体之后销毁成员，成员按初始化顺序的逆序销毁**

对象最后一次使用之后，析构函数的函数体可以执行类设计者希望执行的任何收尾工作，一般释放对象在生存期分配的所有资源

**析构部分是隐式的，成员销毁时发生什么完全依赖于成员的类型。销毁类类型的成员需要执行它自己的析构函数**

**内置类型无析构函数，因此销毁内置类型什么也不需要做**

**隐式销毁一个内置指针类型的成员不会delete它所指向的对象**

**智能指针是类类型，因此具有析构函数，智能指针在析构阶段会被自动销毁**

##### 何时调用析构函数

无论何时一个对象**被销毁，就会自动调用它的析构函数：**

- 变量在离开其作用域时被销毁
- 当一个对象被销毁时，其成员被销毁
- 容器（包括标准库容器和数组）被销毁时，其元素被销毁
- 对于动态分配的对象，当对指向它的指针应用delete运算符时被销毁
- 对于临时对象，当创建它的完整表达式结束时被销毁

**当指向一个对象的引用或指针离开作用域时，析构函数不会执行**

##### 合成析构函数

当一个类未定义自己的析构函数时，编译器会为它定义一个合成析构函数。

**对于某些类，合成析构函数被用来阻止该类对象被销毁。若不是这种情况，合成析构函数的函数体为空**

**析构函数体自身并不直接销毁成员，成员是在析构函数体之后隐含的析构阶段中被销毁的**

```c++
//等价于合成析构函数
class Sales_data{
    public:
    ~Sales_data(){ }
} //执行完毕后成员会被自动销毁 
```

#### 三/五法则

在新标准下，一个类还定义一个移动构造函数和一个移动赋值运算符

C++不要求定义所有这些操作，可以只定义其中一两个。但这些操作常常被看作一个整体。通常只需要其中一个操作而不需要定义所有操作的情况很少见

##### 需要析构函数的类也需要拷贝和赋值操作

对析构函数的需求比对拷贝构造函数或赋值运算的需求更明显。**若一个类需要自定义一个析构函数，则它也需要自定义一个拷贝构造函数和一个拷贝赋值运算符**

因为若需要析构函数时，表示里面有动态分配的资源，那么拷贝和赋值时若不自己重新定义，会两个指向同一片内存，这样析构操作就会重复，比如造成两次delete

**需要自定义拷贝操作的类也需要自定义赋值操作，反之亦然。但它们不一定需要自定义析构函数**

#### 使用=default

**它可以显式地让编译器生成合成版本**

一个默认的成员只影响为这个成员而生成的代码，因此=default直到编译器生成代码时才需要

```c++
class Sales_data{
  public:
    Sales_data() = default;
    Sales_data(const Sales_data&) = default;
    Sales_data& operator=(const Sales_data &);
    ~Sales_data() = default;
    
};
//非内联版本
Sales_data& Sales_data::operator=(const Sales_data&) = default;

```

#### 阻止拷贝

某些类的拷贝构造函数和拷贝赋值运算没有合理的意义，定义类时必须采用某种机制阻止拷贝或赋值

##### 定义删除的函数

我们可以通过将拷贝构造函数和拷贝赋值运算符定义为**删除的函数**来阻止拷贝

我们虽然声明了它们，但是不能以任何形式使用它们。在函数的参数列表后面加上**=delete**即代表删除的函数

=delete表示不定义这些成员，**它必须出现在函数第一次声明的时候**

**=delete可以对任何函数指定**

```c++
struct NoCopy{
    NoCopy() = default;
    NoCopy(const NoCopy&) = delete;  //阻止拷贝
    NoCopy& operator=(const NoCopy&) = delete; //阻止赋值
    
}
```

**析构函数不能是删除的成员**

我们不能删除析构函数，若它被删除，则无法销毁此类型的对象了，编译器不允许定义该类型的变量或创建该类的临时对象

**若类型删除了析构函数，仍可以动态分配这种类型的对象，但是不能释放**

```c++
struct NoDtor{
  NoDtor() = default;
  ~NoDtor() = delete; //不能销毁该类型对象
};

NoDtor nd; //错误！！它的析构函数被删除
NoDtor *p = new NoDtor(); //可以
delete p; //错误！！不能释放
```

##### 合成的拷贝控制成员可能是删除的

若一个类有数据成员不能默认构造、拷贝赋值或销毁，则对应的成员函数将被定义为删除的

一个成员有删除或不可访问（private）的析构函数会导致合成的默认和拷贝构造函数被定义为删除的，因为会创建无法销毁的对象

对于具有引用成员或无法默认构造的const成员的类，编译器不会为其合成默认构造函数。同样，**若类有const成员，则不能使用合成的拷贝赋值运算符。将新值赋予一个const对象是不可能的**

将新值赋予一个引用只会改变引用指向对象的值，不是引用本身。**若为这样的类合成拷贝赋值运算符，赋值后左侧运算对象仍指向赋值前的对象，而不是右侧运算对象指向的对象。因此对于有引用成员的类，合成拷贝赋值运算符被定义为删除的**

##### private拷贝控制

在新标准之前，类通过将其拷贝构造函数和拷贝运算符声明为private来组织拷贝

析构函数是public，仍然可以定义该类型对象。但是拷贝构造函数和拷贝赋值运算符都是private，外部代码不能拷贝它

此时友元或成员函数仍能拷贝对象，这时**可以声明但不定义它们**

```c++
class PrivateCopy{
	//默认是private
    privateCopy(const PrivateCopy&);
    privateCopy &operator=(const PrivateCopy&);
public:
    PrivateCopy() = default;
    ~privateCopy();
};
```

### 2、拷贝控制和资源管理

一旦一个类需要自定义析构函数，那么它几乎肯定需要一个拷贝构造函数和一个拷贝赋值运算符

可以定义拷贝操作，使类的行为看起来像一个值或像一个指针

类的行为像一个值，意味着它应该也有自己的状态，副本和原对象是完全独立的，比如标准库容器和string类的行为像一个值

类的行为像指针则共享状态，拷贝这种类对象时，副本和原对象使用相同的底层数据，改变副本也会改变原对象，反之亦然，比如shared_ptr提供类似指针的行为

**IO类型和unique_ptr不允许拷贝或赋值，因此它们的行为既不像值也不像指针**

#### 行为像值的类

```c++
class HasPtr{
  public:
    HasPtr(const string &s = string()):ps(new string(s)),i(o){ }
    //对ps指向的string，每个HasPtr对象都有自己的拷贝
    HasPtr(const HasPtr &p) :ps(new string(*p.ps),i(p.i)){ }
    HasPtr& operator=(const HasPtr&);
    ~HasPtr() {delete ps; }
  private:
    string *ps;
    int i;
};

HasPtr& HasPtr::operator=(const HasPtr &rhs){
    auto newp = new string(*rhs.ps); //拷贝底层string
    delete ps;//释放旧内存
    ps = newp;
    i = rhs.i;
    return *this; //返回本对象
}
```

**赋值运算符通常组合了析构函数和构造函数的操作**，类似析构函数，赋值操作会销毁左侧运算对象的资源。类似拷贝构造函数，赋值操作会从右侧运算对象拷贝数据。

这些操作以正确顺序执行，**即使将一个对象赋予它自身，也保证正确**。赋值运算符如果可能，还应该是异常安全的，即使发生异常也可将左侧运算对象置于有意义的状态

**赋值运算符编写的操作：**

**一般先将右侧运算对象拷贝到一个局部临时对象中，当拷贝完成后再销毁左侧运算对象的现有成员就安全了。当左侧运算对象的资源被销毁，就只需将数据从临时对象拷贝到左侧运算对象的成员中**

```c++
//错误编写赋值运算符的方法
HasPtr& HasPtr::operator=(const HasPtr &rhs){
    delete ps;//释放内存
    //若rhs和*this指向同一对象，我们就会从已释放的内存中拷贝数据！！
    ps = new string(*rhs.ps);
    i = rhs.i;
    return *this; //返回本对象
}
```

#### 定义行为像指针的类

拷贝构造函数和拷贝赋值运算符需要**拷贝指针成员本身**而不是它指向的string，析构函数只有当最后一个指向string的HasPtr销毁时，它才可以释放string

另一个类展现类似指针的行为的最好方法是使用shared_ptr来管理类中的资源。但是有时我们希望**使用引用计数直接管理资源**

**引用计数不能直接作为成员数据存放，否则会无法正确更新。可以将它保存在动态内存中，创建一个对象时就分配一个计数器，拷贝或赋值对象时就拷贝指向计数器的指针**

```c++
class HasPtr{
public:
    //构造函数分配新的计数器并且置为1，分配新的string
    HasPtr(const string &s = string()):ps(new string(s)),i(0),use(new size_t(1)){ }
    //拷贝构造函数拷贝所有三个数据成员，并递增计数器
    HasPtr(const HasPptr &p): ps(p.ps),i(p.i),use(p.use){++*use;}
    HasPtr& operator=(const HasPtr&);
    ~HasPtr();
private:
    string *ps;
    int i;
    size_t *use; //记录多少个对象共享成员
};

HasPtr::~HasPtr(){
	if(--*use==0){ //引用计数变为0，则释放string和计数器内存
        delete ps;
        delete use;
    }
}

//拷贝赋值运算符必须递增右侧运算对象的引用计数，递减左侧运算对象的引用计数
//在必要时释放内存，它执行拷贝构造函数和析构函数的工作
HasPtr& HasPtr::operator=(const HasPtr &rhs){
    ++*rhs.use; //递增右侧运算对象的引用计数
    if(--*use == 0){  //递减本对象的引用计数
        delete ps; 
        delete use;
    }
    ps = rhs.ps;
    i = rhs.i;
    use = rhs.use;
    return *this;
    
}
```

赋值运算符必须处理自赋值，可以先递增rhs中的计数然后再递减左侧运算对象的计数。

### 3、交换操作

管理资源的类除了定义拷贝控制成员，通常还定义一个swap函数。对于与重排元素顺序的算法一起使用的类，定义swap非常重要，这类算法会调用swap

**若一个类定义了自己的swap，则这些算法使用自定义的。否则它们会使用标准库定义的swap**

```c++
//标准库swap类似如下形式：
HasPtr temp = v1;
v1 = v2;
v2 = temp;  
//拷贝一个类值的HasPtr会分配一个新string来帮助实现交换
```

我们希望swap交换指针，避免不必要的内存分配。自定义swap来进行重载:

```c++
class HasPtr{
  friend void swap(HasPtr&, HasPtr&)  
  //其他成员的定义...    
};

//swap是为了优化代码，因此定义为内联函数
inline void swap(HasPtr &lhs, HasPtr &rhs){
    //内置类型调用标准库的swap
    using std::swap;
    swap(lhs.ps, rhs.ps); //交换指针
    swap(lhs.i , rhs.i);
}

```

**类类型成员若重写了自己的swap，注意编写此类的swap时不要使用std::swap，要使用该类类型的swap**

```c++
//假设Foo类中有HasPtr成员
void swap(Foo &lhs, Foo &rhs){
    std::swap(lhs.h, rhs.h); //这里使用了标准库版本的swap
    //交换其他成员
}

void swap(Foo &lhs, Foo &rhs){
    using std::swap;
    swap(lhs.h, rhs.h); //使用了HasPtr版本的swap、
    //交换其他成员
}
```

**在赋值运算符中使用swap**

定义了swap的类通常用swap来定义他们的赋值运算符。这些赋值运算符使用了**拷贝并交换**的技术，**将左侧运算对象和右侧运算对象的一个副本进行交换**

**这个技术自动处理了自赋值情况，且异常安全，改变左侧运算对象之前先拷贝右侧运算对象**

```c++
//rhs是按值传递的,进行了拷贝构造
//将右侧运算对象中的string拷贝到rhs
HasPtr& HasPtr::operator=(HasPtr rhs){
    //交换左侧运算对象和局部变量rhs的内容
    swap(*this, rhs); //rhs指向本对象曾使用的内存
    return *this;  //rhs被销毁，delete了rhs中的指针
}
```

### 4、对象移动

新标准可以移动而不是拷贝对象，某些情况对象拷贝后就立即被销毁，移动而非拷贝对象会大幅提升性能。另一个原因是IO类或unique_ptr类不能被拷贝，但是可以移动

标准库容器、string、和shared_ptr类既支持移动也支持拷贝；IO类和unique_ptr只支持移动

#### 右值引用

右值引用就是**必须绑定到右值的引用**，通过**&&**而不是&来获得右值引用

**右值引用只能绑定到一个将要销毁的对象，可以自由地将一个右值引用的资源移动到另一个对象中**

**类似左值引用，右值引用也只是某个对象的别名。**

**右值引用可以绑定到要求转换的表达式、字面常量或返回右值的表达式**，但不能将其直接绑定到一个左值上

```c++
int i = 42;
int &r = i;
int &&rr = i; //错误！！右值引用不能绑定左值
int *r2 = i*42; //错误！！i*42是右值
const int &r3 = i*42; //正确：const引用可绑定右值
int &&rr2 = i*42; //正确
```

返回左值引用的函数，连同赋值、下标、解引用和前置递增/递减运算符，都是返回左值的表达式的例子

返回非引用类型的函数，连同算术、关系、位和后置递增/递减运算符，都生成右值

**左值持久；右值短暂**

左值有持久的状态，而右值要么是字面常量，要么是在表达式求值过程中创建的临时对象

**右值引用只能绑定到临时对象**，所引用的对象将要被销毁，该对象无其他用户。因此使用右值引用的代码可自由接管所引用对象的资源

**变量是左值**

变量可看作是只有运算对象而没有运算符的表达式，**变量表达式都是左值**

不能将右值引用绑定到右值引用类型的变量上，**变量是持久的，直到离开作用域才被销毁**

```c++
int &&rr1 = 42;
int &&rr2 = rr1; //错误！！！表达式r1是左值
```

#### 标准库move函数

可以显式地将一个左值转换为对应的右值引用类型。

还可以通过调用move这个新的标准库函数来获得**绑定到左值上的右值引用**，它定义在头文件utility中

**调用move后对于一个左值，会像右值一样处理它。除了对rr1赋值或销毁外，我们将不再使用它**

对move不提供using声明

```c++
int &&rr3 = std::move(rr1);//使用std::move，避免潜在的名字冲突
```

#### 移动构造函数和移动赋值运算符

##### 移动构造函数

移动构造函数的第一个参数是该类类型的**右值引用**，任何其他的参数必须有**默认实参**

移动构造函数必须确保**移后源对象进行销毁时是无害的**，一旦资源完成移动，源对象必须不再指向被移动的资源，**这些资源已经归属新创建的对象**

**移动构造函数不分配新内存，它接管给定StrVec的内存，然后将给定对象中的指针都置为空**

```c++
StrVec::StrVec(StrVec &&s) noexcept:  //移动操作不应该抛出任何异常
	elements(s.elements),first_free(s.first_free),caps(s.cap)
{
	s.elements = s.first_free = s.cap = nullptr; //令s运行析构函数时是安全的       
}        
```

##### 移动操作、标准库容器和异常

移动操作窃取资源，它通常不会分配任何资源。因此移动操作不会抛出任何异常，这时需要将此事通知标准库，若不通知则标准库会认为我们的类可能抛出异常，同时为了处理它而做一些额外的工作

**通知标准库的一种方式是在构造函数中指明noexcept，承诺一个函数不抛出异常。**

**在构造函数中，noexcept出现在参数列表和初始化列表的冒号之间；而普通函数则在函数的参数列表后指定noexcept**

**必须在类头文件的声明和定义中（若定义在内外）都指定noexcept**

```c++
class StrVec{
  public:
    StrVec(StrVec &&) noexcept;
};
StrVec::StrVec(StrVec &&s) noexcept : //...{ }
```

比如vector容器重新分配内存空间时，我们移动元素通常会改变它的值。若在移动构造了部分元素后发生了异常就会出现问题。为了避免这种问题，除非vector知道移动构造时没有异常，否则会使用拷贝构造。

因此必须显式告诉标准库我们的移动构造函数可以安全使用

##### 移动赋值运算符

它执行与析构函数和移动构造函数相同的工作，它也必须正确处理自赋值的情况

若资源相同，不能在使用右侧运算对象的资源前就释放左侧的资源

```c++
StrVec &StrVec::operator=(StrVec &&rhs) noexcept{
    //检测自赋值
    if(this != &rhs){
        free();  //释放已有元素
        elements = rhs.elements; //从rhs接管资源
        first_free = rhs.first_free;
        cap = rhs.cap;
        rhs.elements = rhs.first_free = rhs.cap = nullptr;
    }
    return *this;
}
```

**移后源对象必须可析构**

从一个对象移动数据并不会销毁此对象，但有时移动操作完成后源对象会被销毁。因此需确保移后源对象进入可析构的状态，保证它仍是有效的（可安全地赋予新值或可安全使用而不依赖当前值）

##### **合成的移动操作**

**只有当一个类没有定义任何自己版本的拷贝控制成员，且类的每个非static数据成员都可以移动时，编译器才会为它合成移动构造函数或移动赋值运算符。**

编译器可以移动内置类型的成员，类类型则需要它有对应的移动操作才能够移动

```c++
//编译器会为X和hasX生成移动构造函数
struct X{
  int i;
  string s; //string定义了自己的移动操作  
};

struct hasX{
  X men;  
};
X x,x2 = std::move(x);
hasX hx,hx2 = std::move(hx);
```

**移动操作永远不会隐式定义为删除的函数，但是若显示指明=default的移动操作时且编译器不能移动所有成员，这时会将移动操作定义为删除的函数**

当类成员定义了自己的拷贝构造函数且未定义移动构造函数，或类成员未定义自己的拷贝构造函数且编译器不能为它合成移动构造函数，移动构造函数会被定义为删除的，移动赋值运算符类似

其余将合成的移动操作定义为删除的函数的情况和之前定义的删除的合成拷贝操作原则类似

```c++
struct hasY{
  hasY() = default;
  hsaY(hasY&&) = default; // hasY会有一个删除的移动构造函数
  Y men;  //Y类定义了自己的拷贝构造函数但是没定义自己的移动构造函数
};
hasY hy, hy2 = std::move(hy);  //错误！！移动构造函数是删除的
```

**若类定义了一个移动构造函数和（或）移动赋值运算符，则它的合成拷贝构造函数和拷贝赋值运算符会被定义为删除的（即必须也定义自己的拷贝操作）**

##### 移动右值，拷贝左值

若类中既有移动构造函数，也有拷贝构造函数，则使用普通的函数匹配规则来确定使用哪个。赋值操作类似

```c++
StrVec v1 , v2;
v1 = v2; //v2是左值，使用拷贝赋值
StrVec getVec(istream &);  //getVec返回右值
v2 = getVec(cin); //使用移动赋值，getVec返回右值
```

**若不存在移动构造函数，则右值也会被拷贝，即使调用move也是如此**

用拷贝构造函数代替移动构造函数肯定是安全的（赋值操作类似）

```c++
class Foo{
  public:
    Foo() = default;
    Foo(const Foo&);//拷贝构造函数
    //未定义移动构造函数
};

Foo X;
Foo y(x);
//move会返回绑定到x的Foo&&，拷贝构造再将它转换为const Foo&
Foo z(std::move(x)); //拷贝构造函数，未定义移动构造函数
```

#### 右值引用和成员函数

成员函数也可以同时提供拷贝和移动版本。它接收一个指向const的引用或非const的右值引用

比如push_back这一标准容器库中的函数提供了两个版本：

```c++
void push_back(const X&); //拷贝
void push_back(X&&); //移动，只能传递非const的右值
```

##### **强制左侧运算对象（即this指向的对象）是左（右）值**

与定义const成员函数相同，在参数列表后放置一个**引用限定符，&代表this可指向左值，&&代表指向右值。它必须同时出现在声明和定义中，引用限定符也只能用于非static成员函数**

**若同时出现const和引用限定，引用限定必须跟在const之后**

```c++
//向右值赋值
s1 + s2 = "wow!";

class Foo{
  public:
    Foo &operator=(const Foo&) &; //只能向可修改的左值赋值
};

Foo &Foo::operator=(const Foo &rhs) &{
    //...
    return *this;
}

class Foo{
  public:
    Foo someMem() const &; //const在前
};
```

**对于&限定的函数，只能用于左值；对于&&限定的函数，只能用于右值**

```c++
Foo &retFoo(); //返回引用，调用它是左值
Foo retVal(); //返回一个值，调用它是右值
Foo i,j;
retFoo() = j; //正确：该函数返回左值
i = retVal(); //正确：右值可作为右侧运算对象
retVal() = j; //错误！！该函数返回右值


class Foo{
	public:
    	//重载版本
    	Foo sorted() &&;  //用于可改变的右值
		Foo sorted() const &; //用于任何类型的Foo
    private:
    	vector<int> data;
};

//本对象是右值，可以原址排序
Foo Foo::sorted() &&{
    sort(data.begin(),data.end());
    return *this;
}

//本对象是const或是左值，都不能原址排序
Foo Foo::sorted() const &{
    Foo ret(*this);  //拷贝副本
    sort(ret.data.begin(),ret.data.end()); //排序副本
    return ret; //返回副本
}
retVal().sorted(); //retVal()返回右值  调用Foo::sorted() &&
retFoo().sorted(); //retFoo()返回左值  调用FOO::sorted() const &
```

**引用限定符在函数具有相同参数列表进行重载时必须在它们后面都加上，要么就都不加（只限相同参数列表）**

## 三、重载运算和类型转换

### 1、基本概念

**重载的运算符**是**具有特殊名字的函数：关键字operator和它后面要定义的运算符号共同组成**

重载的运算符类似其他函数，包括**返回类型、参数列表和函数体，参数数量和它作用的运算对象数相同**

**一元运算符有一个参数，二元运算符有两个参数**（左侧运算对象传递给第一个，右侧运算对象传递给第二个）。注意成员函数第一个参数绑定到隐式的this上，因此少一个参数

除了重载的**函数调用运算符operator( )**之外，其他重载运算符**不能含有默认实参**

**运算符函数要么是类的成员，要么至少含有一个类类型的参数**

```c++
int operator+(int,int); //错误！！不符合要求
```

可以重载大多数运算符，但不能发明新的运算符号。（`+ - * &`）这四个既是一元也是二元，可以从参数数量来推断是哪种

**重载运算符的优先级和结合律与对应的内置类型一致**

![](D:\office\word\计算机基础\C++\截图2\重载运算符.jpg)

另外。`&&   ||   ,`  不建议被重载，因为他们指定了运算对象的求值顺序，前两个还有短路求值属性。重载之后无法保留特性。

C++语言已经定义了**逗号运算符和取地址运算符**作用于类类型对象的特殊含义，因此一般不应该被重载

#### **调用重载运算符**

```c++
//非成员运算符函数的等价调用
data1 + data2;
operator+(data1, data2); //函数形式的调用

//调用成员函数的方式
data1 += data2;
data1.operator+=(data2);
```

 若某些操作在逻辑上和运算符相关，则它们适合定义成重载的运算符：

**重载运算符的返回类型通常应该和内置版本的返回类型兼容。**逻辑运算符和关系运算符返回bool，算术运算符返回类类型的值，赋值运算符和复合赋值运算符返回左侧运算对象的引用

#### 选择作为成员或非成员

`赋值=、下标[]、调用( )、成员访问箭头->`运算符必须是成员

复合赋值运算符一般是成员，并非必须

改变对象状态的运算符或与给定类型相关的运算符比如递增、递减、解引用运算符一般是成员

具有对称性的运算符可能转换任意一端的运算对象，比如**算术、相等、关系和位运算符**等，它们一般是非成员函数

**运算符作为成员函数时，左侧运算对象必须是运算符所属类的对象**

```c++
string s = "world";
string t = s + "!";
string u = "hi" + s; //若+是string类的成员函数，则产生错误。
//实际上string将+定义为普通的非成员函数，只要求其中一个是string类类型就行，否则就不会调用string的重载+运算符函数
```

### 2、输入和输出运算符

IO 库定义了用>>和<<读写内置类型的版本，而类则需要自定义合适其对象的新版本以支持IO操作

**输入输出运算符必须是非成员函数，否则它的左侧运算对象将是类对象**

**IO运算符一般还需要读写类的非公有数据成员，所以它们一般被声明为友元**

#### 重载输出运算符<<

一般它的**第一个形参是非常量ostream对象的引用**，非常量的原因是向流写入内容会改变其状态，引用则是因为无法复制一个流对象

一般**第二个形参是一个常量的引用**，它是我们想打印的类类型。它是引用的原因是我们不想复制实参，是常量则因为一般打印对象不会改变对象的内容

operator<<为了和其他输出运算符保持一致，它一般返回ostream形参

```c++
ostream &operator<<(ostream &os, const Sales_data &item){
    os << item.isbn() << " " << item.units_sold << " "
        << item.revenue << " " << item.avg_price();
    return os;
}
```

输出运算符应该主要负责打印对象的内容而非控制格式，不应该打印换行符

#### 重载输入运算符>>

**第一个形参是运算符将要读取的流的引用，第二个形参是将要读入到的非常量对象的引用**，它是非常量是因为需要将数据读入到这个对象中

**输入运算符必须处理输入可能失败的情况，而输出运算符不需要**

if会检查读取操作是否成功，若发生了IO错误，则运算符将给定的对象重置为空Sales_data，这样可确保对象处于正确的状态

在程序中没有逐个检查每个读取操作，而是全部读取后赶在使用之前一次性检查。

若发生了错误，不需要在意哪部分输入失败，只要将一个新的默认初始化的Sales_data对象赋予item

```c++
istream &operator>>(istream &is , Sales_data &item){
	double price;
    is >> item.bookNo >> item.units_sold >> price;
    if(is)  //检查是否输入成功
        item.revenue = item.units_sold * price;
    else
        item = Sales_data(); //输入失败：对象被赋予默认的状态
    return is;    
}
```

**当流含有错误类型的数据时读取操作可能会失败；当读取操作到达文件末尾或遇到输入流的其他错误时也会失败**

一些输入运算符需要做更多数据验证的工作，需要设置流的状态来标示出失败信息。最好由IO标准库来标示这些错误

### 3、算术和关系运算符

它们一般定义成**非成员函数**从而允许对左侧或右侧的运算对象进行转换，它们一般不改变运算对象的状态，所以**形参都是常量的引用**

算术运算符一般会计算两个运算对象并得到一个新值，它会位于局部变量之内，操作完成返回该局部变量的副本

若类定义了算术运算符，则它一般也会定义一个对应的复合赋值运算符。**此时最有效的方式是使用复合赋值来定义算术运算符**

```c++
Sales_data operator+(const Sales_data &lhs , const Sales_data &rhs){
    Sales_data sum = lhs; //将lhs的数据成员拷贝给sum
    sum += rhs;
    return sum;
}
```

#### 相等运算符

类通过定义相等运算符检验两个对象是否相等。只有当所有对应的成员都相等时才认为两个对象相等

```c++
bool operator==(const Sales_data &lhs, const Sales_data &rhs){
    return lhs.isbn() == rhs.isbn() &&
           lhs.units_sold == rhs.units_sold &&
           lhs.revenue == rhs.revenue;
}
bool operator !=(const Sales_data &lhs, const Sales_data &rhs){  //不等于也需要同时定义，但内部应该利用对方
    return !(lhs == rhs);
}
```

#### 关系运算符

定义了相等运算符的类常常包含关系运算符，在关联容器和一些算法中要用到小于运算符，所以定义operator<会比较有用

**关系运算符应该定义顺序关系，令其与关联容器中对关键字的要求一致。**

**若类同时包含==运算符，则定义一种关系令它与==保持一致。特别是两个对象`！=`时，那么一个对象应该<另一个**

某些类不存在一种逻辑可靠的<定义，这时这个类不定义<运算符会更好。

若类存在唯一一种逻辑可靠的<定义，则可以给它定义<。此时若同时存在==，则当且仅当<的定义和==产生的结果一致时才定义<运算符

### 4、赋值运算符

之前已经接触过拷贝赋值和移动赋值运算符，它们可以把类的一个对象赋值给该类的另一个对象。**此外类还可以定义其他赋值运算符以使用别的类型作为右侧运算对象**

**赋值运算符不论形参的类型是什么，它都必须定义为成员函数**

比如vector类定义了第三种赋值运算符，它接受花括号内的元素列表作为参数。可以用如下的方式使用它：

**这个运算符不需要检查对象向自身的赋值，因为它的形参确保了和this不是同一个对象**

```c++
vector<string> v;
v = {"a","an","the"};

//在类中使用此种赋值运算符
class StrVec{
    public:
    	StrVec &operator=(initializer_list<string>);
}
//为了与内置类型的赋值运算符保持一致，它将返回其左侧运算对象的引用
StrVec &StrVec::operator=(initializer_list<string> il){
    //alloc_n_copy分配内存空间并从给定范围内拷贝元素
    auto data = alloc_n_copy(il.begin(),il.end());
    free(); //销毁当前对象中的元素并释放内存空间
    elements = data.first;
    first_free = cap = data.second;
    return *this;
}
```

#### 复合赋值运算符

**它不一定是类的成员，不过一般将它也定义在类的内部作为成员**

为了和内置类型的复合赋值保持一致，类中的复合赋值运算符也要返回其左侧运算对象的引用

```c++
//作为成员的二元运算符：左侧运算对象绑定到隐式的this指针
//假定两个对象表示的是同一本书
Sales_data& Sales_data::operator+=(const Sales_data &rhs){
    units_sold += rhs.units_sold;
    revenue += rhs.revenue;
    return *this;
}
```

### 5、下标运算符

表示容器的类通常可以通过元素在容器中的位置访问元素，它们一般会定义下标运算符operator[ ]

**下标运算符必须是成员函数**

为了和下标的原始定义兼容，**下标运算符通常以所访问元素的引用作为返回值**，好处是下标可以出现在赋值运算符的任意一端

我们最好同时定义下标运算符的常量版本和非常量版本。当作用于一个常量对象时，下标运算符返回常量引用来确保我们不会给返回的对象赋值

```c++
class StrVec{
  public:
    string& operator[](size_t n){
        return elements[n];
    }
    const string& operator[](size_t n) const{
        return elements[n];
    } 
 private:
    string *elements; //指向数组首元素的指针
};

const StrVec cvec = svec;//假设svec是一个StrVec对象
if(svec.size() && svec[0].empty()){
	svec[0] = "zero";
    cvec[0] = "zip"; //错误！！！对cvec取下标返回的是常量引用
}

```

### 6、递增和递减运算符

**一般设置为成员函数**，因为它们会改变所操作对象的状态。**定义递增递减运算符的类应该同时定义前置版本和后置版本

**定义前置递增/递减运算符**

和内置版本保持一致，前置运算符应该返回递增或递减后对象的引用

这里的check函数会检查该对象是否有效，若是则接着检查给定的索引是否有效。

```c++
//前置递增和递减运算符
class StrBlobPtr{
  public:
    StrBlobPtr& operator++();
    StrBlobPtr& operator--();
};

StrBlobPtr& StrBlobPtr::operator++(){
    //curr若已经指向了容器的尾后位置，则无法递增它
    check(curr,"increment past end of StrBlobPtr");
    ++curr;
    return *this;
}
StrBlobPtr& StrBlobPtr::operator--(){
    --curr;//若已经是0，则继续递减它产生一个无效的下标，此时无符号数会是一个非常大的无效的正数值
    check(curr,"decrement past begin of StrBlobPtr");
    return *this;
}
```

**区分前置和后置运算符**

前置和后置版本使用的是同一个符号，此时普通的重载形式无法区分

为了解决此问题，**后置版本接收一个额外的（不被使用）int类型的形参**。使用后置运算符时，编译器为它提供一个值为0的实参。它的唯一作用就是区分前置版本和后置版本的函数

和内置版本保持一致，后置运算符应该返回对象的原值，返回的形式是一个值而非引用

后置运算符可以调用它们已经重载的前置版本来完成实际工作

```c++
//后置运算符
class StrBlobPtr{
    public:
    	StrBlobPtr operator++(int);
    	StrBlobPtr operator--(int);
};

StrBlobPtr StrBlobPtr::operator++(int){ //用不到int形参，所以不需要显示命名
    //后置无须检查有效性
    StrBlobPtr ret = *this;
    ++*this;
    return ret;
}
StrBlobPtr StrBlobPtr::operator--(int){
    //后置无须检查有效性
    StrBlobPtr ret = *this;
    --*this;
    return ret;
}

//函数版本调用后置运算符时必须为它的整型参数传值
StrBlobPtr p(a1);//p指向a1中的vector
p.operator++(0); //调用后置版本
p.operator++(); //前置版本
```

### 7、成员访问运算符

在迭代器类和智能指针类中常常用到解引用（*）和箭头运算符（->）

**箭头运算符必须是类的成员，解引用运算符通常也是类的成员**

**重载的箭头运算符必须返回类的指针或自定义了箭头运算符的某个类的对象**

```c++
class StrBlobPtr{
  public:
    string& operator*() const{
        auto p = check(curr,"dereference past end");//检查curr是否在作用范围，是则返回curr所指元素的引用
        return (*p)[curr]; //(*p)是对象所指的vector
    }
    string* operator->() const{
        return& this->operator*(); //调用解引用运算符并返回解引用结果元素的地址
    }
};

StrBlob a1 = {"hi","bye","now"};
StrBlobPtr p(a1); //p指向a1中的vector
*p = "okay"; //给a1的首元素赋值
cout << p->size() << endl;  //打印4，这是a1首元素的大小。使用p.operator->()的结果来获取size()。
cout << (*p).size() << endl;  //等价于p->size()
```

**对箭头运算符返回值的限定**

**箭头运算符永远不能丢掉成员访问这个最基本的含义，可以改变的是箭头从哪个对象中获取成员，而箭头获取成员这一事实则永远不变**

形如`point->mem`的表达式，**point必须是指向类对象的指针或是一个重载了operator->的类对象。**

```c++
//point->mem分别等价于：
(*point).mem; //point是一个内置的指针类型
//point是类对象时
//若point.operator->()的结果是指针，则执行(*结果).mem;
//若point.operator->()的结果是含有箭头重载的类对象，则执行此类对象的箭头重载函数，再使用它的结果调用mem
```

### 8、函数调用运算符

若类重载了函数调用运算符，则可以像使用函数一样使用该类对象。这样的类同时也能够存储状态，所以比普通函数更加灵活

**函数调用运算符必须是成员函数，一个类可定义多个不同版本的调用运算符，相互之间应该在参数数量或类型上有所区别**

类定义了调用运算符后，则该类的对象叫作**函数对象**，这些对象的行为像函数一样

```c++
struct absInt{
  int opearator()(int val) const{
      return val < 0 ? -val : val;
  }  
};

int i = -42;
absInt absObj;  
int ui = absObj(i);
```

**含有状态的函数对象类**

函数对象类一般含有一些数据成员，它们被用于定制调用运算符中的操作

```c++
class PrintString{
  public:
	PrintString(ostream &o = cout, char c = ' '): os(o),sep(c){}
    void operator() (const string &s) const{os << s << sep;}
  private:
    ostream &os;
    char sep;
};

PrintString printer;
printer(s);  //使用默认值打印到cout
PrintStirng errors(cerr, '\n');
errors(s);  //在cerr中打印s

```

函数对象常常用于泛型算法的实参

```c++
for_each(vs.begin(),vs.end(),PrintString(cerr,'\n'));//第三个参数是PrintString的一个临时对象
```

#### lambda是函数对象

编写一个lambda后，编译器将该表达式翻译成一个未命名类的未命名对象。在lambda表达式产生的类中含有一个重载的函数调用运算符

```c++
stable_sort(words.begin(),words.end(),
           [](const string &a, const string &b){return a.size()<b.size();});

//类似下面这个类的一个未命名对象
class ShorterString{
  public:
    bool operator()(const string &s1, const string &s2) const{ //默认情况下lambda不能改变它捕获的变量，因此是const
        return s1.size() < s2.size();
    }
        
};
//使用此类重写stable_sort
stable_sort(words.begin(),words.end(),ShorterString());//每次比较就会调用该对象
```

**表示lambda及相应捕获行为的类**

通过引用捕获变量时由程序确保lambda执行时引用所引的对象确实存在，编译器可以直接使用该引用而无须在lambda产生的类中将其存储为数据成员

通过值捕获的变量被拷贝到lambda中，lambda产生的类必须为每个值捕获的变量建立对应的数据成员，同时创建构造函数，使用捕获的变量的值来初始化数据成员

```c++
auto wc = find_if(words.begin(),words.end(),
                 [sz](const string &a){return a.size() >= sz;});

//产生的类如下
class SizeComp{
    public:
  		SizeComp(size_t n):sz(n){}
    //调用运算符的返回类型、形参和函数体都和lambda一致
   	 	bool operator()(const string &s)const{return s.size() >= sz;}
    private:
    	size_t sz;  //对应着通过值捕获的变量
};

auto wc = find_if(words.begin(),words.end(),SizeComp(sz));
```

**lambda表达式产生的类不含默认构造函数、赋值运算符和默认析构函数**

#### 标准库定义的函数对象

标准库定义了一组表示算术、关系、逻辑运算符的类，它们分别定义了一个执行命名操作的调用运算符。它们都是模板

**标准库函数对象定义在functional头文件中**

可以为它们指定具体的应用类型，即调用运算符的形参类型

![](D:\office\word\计算机基础\C++\截图2\标准库函数对象.jpg)

表示运算符的函数对象类常用来替换算法中的默认运算符

```c++
sort(svec.begin(),svec.end(),greater<string>()); //传入一个临时的函数对象用于执行两个string对象的>比较运算
```

这些函数对象也同样适用于指针，而不使用它们时**直接比较两个无关指针会产生未定义的行为**

```c++
vector<string*> nameTable;
sort(nameTable.begin(),nameTable.end(),
    [](string *a, string *b){ return a < b;}); //错误！！！nameTable的指针彼此之间无关，所以将产生未定义行为

//正确写法：
sort(nameTable.begin(),nameTable.end(),less<string*>());
```

**关联容器中使用`less<key_type>`对元素排序，因此可以定义一个指针的set或在map中使用指针作为关键值而无须直接声明less**

#### 可调用对象和function

C++语言有几种可调用的对象：函数、函数指针、lambda表达式、bind创建的对象和重载了函数调用运算符的类

**可调用对象也有类型**，比如每个lambda有它自己唯一的未命名类类型

**两个不同类型的可调用对象却可能共享同一种调用形式，调用形式指明了调用返回的类型和传递给调用的实参类型**

```c++
int(int,int); //调用形式对应一个函数类型
```

几个可调用对象共享同一种调用形式时，我们有时会希望把它们看成具有相同的类型

```c++
int add(int i, int j){return i+j;}

auto mod = [](int i , int j){return i%j;}; //lambda产生一个未命名的函数对象类

struct divide{
  int operator()(int i, int j){ return i/j; }  
};

//它们共享同一种调用形式
int(int,int)
    
map<string, int(*)(int,int)> binops;
binops.insert({"+",add});
binops.insert({"%",mod});  //错误！！mod不是函数指针，同样divide也不是，所以不能加入
```

**标准库function类型**

可以使用一个叫作function的新的标准库类型来解决上述问题，它是模板，**需要指明该function类型能够表示的对象的调用形式**

![](D:\office\word\计算机基础\C++\截图2\function的操作.jpg)

```c++
function<int(int,int)> f1 = add;
function<int(int,int)> f2 = divide();
function<int(int,int)> f3 = [](int i, int j){ return i*j;};
cout << f1(4,2) << endl; //打印6
cout << f2(4,2) << endl; //打印2
cout << f3(4,2) << endl; //打印8

map<string,function<int(int,int)>> binops ={
    {"+",add},  //函数指针
    {"-",std::minus<int>()},  //标准库函数对象
    {"/",dividie()},
    {"*",[](int i , int j){return i*j;}},
    {"%",mod}
};

binops["+"](10,5);//调用add(10,5)
binops["-"](10,5); //使用minus<int>对象的调用运算符
```

索引map时获得关联值的引用，因此上例中会得到function对象的引用。**function类型重载了调用运算符，它接受自己的实参然后将其传递给存好的可调用对象**

**不能直接将重载函数的名字存入function类型的对象中**

```c++
int add(int i , int j){ return i + j;}
Sales_data add(const Sales_data &,const Sales_data&);
map<string,function<int(int,int)>> binops;
binops.insert({"+",add}); //错误！！不知道是哪个add

//解决这种二义性问题
//存储函数指针
int (*fp)(int,int) = add; //指针所指的add是接受两个int的版本
binops.insert({"+",fp});
//使用lambda消除二义性
binops.insert({"+",[](int a , int b){return add(a,b);}});//此调用只能匹配接受int的add
```

**注：新版本的function和旧版本的unary_function和binary_function无关，后两个类已经被bind所替代**

### 9、重载、类型转换和运算符

转换构造函数和类型转换运算符共同定义了**类类型转换，也叫作用户定义的类型转换**

#### 类型转换运算符

**类型转换运算符**是类的一种特殊成员函数，它**负责将一个类类型的值转换成其他类型，它是隐式执行的（无法传参）**。形式如下：

```c++
operator type() const;
```

type表示某种类型，类型转换运算符可以对除了void之外的任意类型进行定义，只要该类型能作为函数的返回类型。

**因此不允许转换成数组或函数类型，但允许转换成指针（包括数组指针和函数指针）或引用类型**

**类型转换运算符既没有显示的返回类型，也没有形参，而且必须定义成类的成员函数。它一般不改变待转换对象的内容，因此一般定义为const**

```c++
class SmallInt{
  public:
    SmallInt(int i = 0):val(i){
        if(i<0 || i>255)
            throw std::out_of_range("bad smallInt value");
    }
    operator int() const{ return val;}
  private:
    size_t val;
};
```

该类既定义了向类类型的转换，也定义了从类类型向其他类型的转换 。其中构造函数将算术类型的值转换成SmallInt对象，而类型转换运算符将该类对象转换成int

**编译器一次只能执行一个用户定义的类型转换，但是隐式的用户定义类型转换可以在内置类型转换前后一起使用**

```c++
SmallInt si;
si = 4; //将4隐式转换为SmallInt，然后调用赋值运算符
si + 3; //首先将si隐式转换成int，然后执行整数的加法

SmallInt si = 3.14;
si + 3.14;
```

虽然类型转换函数不负责指定返回类型，但实际上每个类型转换函数都会返回一个对应类型的值：

```c++
class SmallInt;
operator int(); //错误！！不是成员函数
class SmallInt{
  public:
    int operator int() const; //错误：指定了返回类型
    oerator int(int = 0) const; //错误：参数列表不为空
    operator int*() const{ return 42;} //错误：42不是指针
};
```

类型转换运算符使用时最好在类类型和转换类型之间存在明显的映射关系。

比如当istream含有向bool的类型转换时，下面的代码会通过：

```c++
int i =42;
//这里会将istream转换为bool，再提升为int，执行左移。实际上输入流是没定义输出符的
cin << i; //此时向bool的类型转换是隐式的，则合法
```

#### 显式的类型转换运算符

为了防止上述istream的异常情况发生，新标准引入了**显式的类型转换运算符**

```c++
class SmallInt{
  public:
    explicit operator int() const{ return val;} //不会自动执行这一类型转换
};
```

**但是表达式被用作条件时，则编译器会将显式的类型转换自动应用于它。即它也会隐式执行**

新标准中IO标准库通过定义一个向bool的显式类型转换实现了同样的目的，**在条件中使用流对象，都会转换为bool**

**因此一般operator bool都定义为explicit的，向bool类型的转换一般发生在条件部分**

```
while(cin >> value); //cin的条件状态是good，则返回真。否则返回假
```

#### 避免有二义性的类型转换

类中若包含一个或多个类型转换，则必须确保在类类型和目标类型之间只存在唯一的一种转换方式。

**需要避免的情况：**

1、两个类提供相同的类型转换：A类定义了接收B类对象的转换构造函数，而B类定义了转换目标是A类的类型转换运算符，它们都能将B类转换为A类，提供了相同的类型转换

```c++
struct B;
struct A{
  A() = default;
  A(const B&);  //将B转换成A  
};
struct B{
  operator A() const;  //也是将B转换成A
};
A f(const &A);
B b;
A a = f(b); //二义性错误：不知道调用哪一个方式进行转换

//只能显式调用其中一个
A a = f(A(b)); //使用A的构造函数
A a = f(b.operator A());
```

2、定义了和内置类型的多个类型转换，最好只定义最多一个和算术类型有关的转换规则

## 四、面向对象程序设计

### 1、OOP：概述

面向对象程序设计（object-oriented programming）的核心思想是**数据抽象、继承和动态绑定**

#### 继承

通过**继承**联系在一起的类构成一种层次关系，层次关系的根部会有一个**基类（base class）**，而其他类则直接或间接地从基类继承而来，它们叫作**派生类（derived class）**

基类负责定义在层次关系中所有类共同拥有的成员，每个派生类定义各自特有的成员

**基类将类型相关的函数和派生类不做改变直接继承的函数区分对待。**对于某些函数，基类希望它的派生类各自定义适合自己的版本，此时基类就将这些函数声明成**虚函数（virtual function）**。

```c++
//基类：按原价销售的书籍
class Quote{
  public:
    string isbn() const; //返回书籍的isbn号
    virtual double net_price(size_t n) const; //返回书籍的实际销售价格，前提是用户购买书的数量达到一定标准。它是类型相关的
};
```

**派生类必须通过类派生列表明确指出它从哪个（哪些）基类继承而来。类派生列表的形式是首先一个冒号，后面紧跟以逗号分隔的基类列表，每个基类前面可有访问说明符**

**派生列表中使用了public，则完全可以将Bulk_quote的对象当做Quote的对象来使用**

**派生类必须在内部对所有重新定义的虚函数进行声明**，可以在这样的函数之前加上virtual关键字，但不是必须。

**新标准允许派生类显式地注明它将使用哪个成员函数改写基类的虚函数，只需要在函数的形参列表后加上override关键字**

```c++
//派生类：可以打折销售的书籍
class Bulk_quote : public Quote{  //继承了Quote
  public:
    double net_price(size_t) const override;
};
```

#### 动态绑定

通过动态绑定（dynamic binding）可以用同一段代码分别处理Quote和Bulk_quote对象

形参是基类的引用，我们既能使用基类对象也能使用派生类的对象。实际传入的对象类型会决定执行net_price的哪一个版本

**函数的运行版本由实参决定，即在运行时选择函数的版本，动态绑定也叫作运行时绑定，使用基类的引用或指针调用一个虚函数时会发生动态绑定**

```c++
double print_total(ostream &os,const Quote &item,size_t n){
    double ret = item.net_price(n);
    os << "ISBN:" << item.isbn()
        << n << ret << endl;
    return ret;
}

//basic类型是Quote，bulk类型是Bulk_quote
print_total(cout, basic , 20);
print_total(cout, bulk , 20);
```

### 2、定义基类和派生类

#### 定义基类

**基类一般都应该定义一个虚析构函数，即使该函数不执行任何实际操作也是如此**

```c++
class Quote{
  public:
    Quote() = default;
    Quote(const string &book, double sales_price): bookNo(book),price(sales_price){}
    string isbn() const{ return bookNo;}
    virtual double nei_price(size_t n) const{ return n *price;}
    virtual ~Quote() = default; //对析构函数进行动态绑定
  private:
    string bookNo; //ISBN编号
  protected:  
    double price = 0.0; //代表普通状态下不打折的价格
};
```

##### 成员函数和继承

派生类可以继承基类的成员，**遇到与类型相关的操作时，派生类必须对其重新定义，派生类需要对这些操作提供自己的新定义从而覆盖（override）继承来的旧定义**

基类必须将成员函数进行区分：一种是基类希望其派生类进行覆盖的函数，基类通常将它定义为**虚函数**；另一种是基类希望派生类直接继承而不要改变的函数

**使用指针或引用调用虚函数时，该调用会动态绑定**

**任何构造函数之外的非静态函数都可以是虚函数，关键字virtual只能出现在类内部的声明语句之前，不能用于类外部的函数定义**

**基类中的虚函数在派生类中隐式地也是虚函数**

普通的成员函数解析过程发生在**编译时**，不是运行时

##### 访问控制和继承

派生类可以继承定义在基类中的成员，但是派生类的成员函数不一定有权访问从基类继承而来的成员

派生类能访问公有成员，不能访问私有成员。**protected则代表基类希望它的派生类可以访问，禁止其他用户访问，这是受保护的访问运算符**

#### 定义派生类

派生类的声明：

声明中不包含派生列表，声明语句的目的是让程序知晓某个名字的存在以及它是什么样的实体

```
class Bulk_quote : public Quote; //错误：派生列表不能出现
class Bulk_quote;
```

**派生类必须将其继承来的成员函数中需要覆盖的函数重新声明**，派生列表中访问说明符的作用是控制派生类从基类继承而来的成员是否对派生类的用户可见

若一个派生是公有的，**则基类的公有成员也是派生类接口的组成部分，还能将公有派生类型的对象绑定到基类的引用或指针上**

大多数类都只继承自一个类，即单继承

```c++
class Bulk_quote : public Quote{  
  public:
    Bulk_quote() = default;
    Bulk_quote(const string&, double, size_t ,double);
    double net_price(size_t) const override;
  private: 
    size_t min_qty = 0;  //使用折扣政策的最低购买数量
    double discount = 0.0; //折扣额
};
```

**派生类经常（不总是）覆盖它继承的虚函数。若没有覆盖，则此虚函数会类似其他普通成员，派生类会直接继承它在基类的版本**

##### **派生类对象和派生类向基类的类型转换**

派生类对象包含一个含有派生类自己定义的（非静态）成员的子对象和一个基类对应的子对象。若有多个基类，则这样的子对象也有多个。这是**继承**的关键

![](D:\office\word\计算机基础\C++\截图2\Bulk_quote对象.jpg)



由于派生类对象中含有与其基类对应的组成部分，所以**可以将派生类对象当成基类对象使用，也可以将基类的指针或引用绑定到派生类对象中的基类部分上**

这种转换叫作**派生类到基类的类型转换，编译器会隐式执行**。我们可以将派生类对象或它的引用、指针运用到需要基类引用、指针的地方

##### 派生类构造函数

虽然派生类继承了基类的成员，但是派生类不能直接初始化它们。**派生类必须使用基类的构造函数来初始化它的基类部分**

**每个类控制它自己的成员初始化过程**，派生类遵循基类的接口，通过调用基类的构造函数来初始化从基类继承而来的成员

派生类对象的基类部分和它自己的数据成员都是在构造函数的初始化阶段执行初始化操作的

```c++
Bulk_quote(const string &book, double p, size_t qty, double disc):
			Quote(book, p), min_qty(qty),discount(disc){ }
```

此函数将它的前两个参数传递给Quote的构造函数，由他初始化基类部分。

**除非特别指出，否则派生类对象的基类部分会像数据成员一样执行默认初始化**

##### 派生类使用基类的成员

派生类可以访问基类的公有成员和受保护的成员：

```c++
//达到了购买书籍的某个最低限量值，就可享受折扣价格了
double Bulk_quote::net_price(size_t cnt) const{
	if(cnt >= min_qty)
        return cnt*(1-discount)*price;
    else
        return cnt*price;
}
```

**注：派生类的作用域嵌套在基类的作用域之内**

##### **继承和静态成员**

基类中若定义了静态成员，**则在整个继承体系中只存在该成员的唯一定义。**不论有多少个派生类，每个静态成员都只存在唯一的实例

```c++
class Base{
  public:
    static void statmen();
    
};
class Derived : public Base{
  void f(const Derived&);  
};

//静态成员若可访问，则既可以通过基类使用它也可以通过派生类使用它
void Derived::f(const Derived &derived_obj){
    Base::statmen();
    Derived::statmen();
    derived_obj.statmen();   //通过Derived对象访问
    statmen(); //通过this对象访问
}
```

##### 被用作基类的类

**若想将某个类用作基类，则该类必须已经定义而非仅仅声明。**派生类需要知道继承而来的成员是什么，当然**一个类不能派生它本身**

一个类是基类，同时它也可以是派生类

```c++
class Base{ //...};
class D1 : public Base{ //...}; 
class D2 : public D1{ //...};    
```

Base是D1的直接基类，是D2的间接基类。直接基类出现在派生列表中

每个类都会继承直接基类的所有成员，最终的派生类会继承其直接基类的成员，以此类推直到继承链的顶端。**最终的派生类将包含它的直接基类的子对象和每个间接基类的子对象**

**防止继承的发生**

若不希望其他类继承它或不想考虑它是否适合作为基类，新标准提供了防止继承发生的方法，在类名后面跟一个final关键字

```c++
class NoDerived final { //...};  //不能作为基类
```

#### **类型转换和继承**

理解基类和派生类之间的类型转换是理解面向对象的关键

一般想把引用或指针绑定到一个对象上，则引用或指针的类型应该和对象的类型一致。

但是**存在继承关系的类是一个重要例外：我们可以将基类的指针或引用绑定到派生类对象上。当使用基类的引用（或指针）时，我们不知道它所绑定的对象的真实类型。可能是基类对象，也可能是派生类对象**

和内置指针一样，智能指针类也支持派生类向基类的类型转换。**可以将一个派生类对象的指针存储在一个基类的智能指针内**

##### 静态类型和动态类型

使用存在继承关系的类型时，**必须将一个变量或其他表达式的静态类型和该表达式表示对象的动态类型区分开来**。

表达式的静态类型在编译时总是已知的，它是变量声明时的类型或表达式生成的类型。

**动态类型则是变量或表达式表示的内存中的对象的内型，动态类型直到运行时才可知**

```c++
double ret = item.net_price(n);
```

item的静态类型是Quote&，它的动态类型则依赖于item绑定的实参。若传递Bulk_quote对象给它，则动态类型是Bulk_quote。动态类型直到运行时调用该函数才会知道。

**表达式不是引用或指针，则它的动态类型和静态类型永远一致**

##### **不存在从基类向派生类的隐式类型转换**

派生类可向基类转换是因为派生类对象都包含了基类部分，而基类的引用或指针可以绑定到该基类的部分上。

基类对象可独立存在，也可作为派生类对象的一部分，因此**不存在从基类向派生类的自动类型转换。即使基类指针或引用绑定在一个派生类对象上，也不可以**

编译器在编译时无法确定某个特定的转换是否安全，它只能检查指针或引用的静态类型来推断转换是否合法

**若基类中含有虚函数，可以使用dynamic_cast请求一个类型转换，此转换的安全检查会在运行时执行**。若已知某个基类向派生类的转换是安全的，则可使用static_cast来强制转换

```c++
Bulk_quote bulk;
Quote *itemP = &bulk;  //正确：动态类型是Bulk_quote
Bulk_quote *bulkP = itemP; //错误：不能将基类转换成派生类对象
```

##### 对象之间不存在类型转换

**派生类向基类的自动类型转换只对指针或引用类型有效**，在派生类类型和基类类型之间不存在这种转换

**当用一个派生类对象为基类对象初始化或赋值时，只有该派生类对象中的基类部分会被拷贝、移动或赋值，它的派生类部分会被忽略**

```c++
#include <iostream>

class Base {
public:
    int a;

    Base() : a(0) {}
    Base(int val) : a(val) {}
};

class Derived : public Base {
public:
    int b;

    Derived(int val1, int val2) : Base(val1), b(val2) {}
};

int main() {
    Derived derivedObj(10, 20);
    Base baseObj = derivedObj; // Object Slicing occurs here

    std::cout << "Base Object's a: " << baseObj.a << std::endl;
    // Output: Base Object's a: 10

    return 0;
}
```

`Derived`类对象`derivedObj`中的基类部分（即`Base`部分）被用来初始化`baseObj`，而`Derived`特有的部分（这里是`b`）被忽略了。这就是所谓的“对象切割”。

### 3、虚函数

我们**使用基类的引用或指针调用一个虚成员函数时会执行动态绑定**。**由于直到运行时才能知道到底调用了哪个版本的虚函数，所以所有虚函数都必须有定义**

**使用普通类型（非引用或指针）的表达式调用虚函数时，在编译时就会将调用的版本确定，对于非虚函数的调用则在编译时也进行绑定**

**C++的多态性（polymorphism）**

多态性的含义是多种形式，具有继承关系的多个类型叫做多态类型，可以使用它们的多种形式而无须在意它们的差异

#### 派生类中的虚函数

在派生类中覆盖了某个基类的虚函数时，可以再次使用virtual关键字指出它的性质，但不一定要做。**因为一旦某个函数被声明成虚函数，则在所有派生类它都是虚函数**

**派生类中覆盖了某个虚函数时，它的形参类型必须和基类中的完全一致，返回类型也需要和基类匹配（但是当返回类型是类本身的指针或引用时不需要满足）**。也就是说D由B派生而来，则基类的虚函数可以返回`B*`，而派生类的对应函数可返回`D*`，但要求从D到B的类型转换是可访问的

#### final和override说明符

派生类**若定义了一个函数和基类的虚函数名字相同，但是形参列表不同，则是相互独立的两个函数**。因此有时可能会弄错形参列表，导致没发生覆盖

新标准中可以使用**override关键字**说明派生类中的虚函数，**若添加了该关键字，但该函数没有覆盖已存在的虚函数，则编译器会报错**

```c++
struct B{
  virtual void f1(int) const;
  virtual void f2();
  void f3();  
};

struct D1 : B{
  void f1(int) const override; //正确
  void f2(int) override; //错误：多个形参
  void f3() override; //错误：f3不是虚函数
  void f4() override; //错误：B无f4的虚函数  
};
```

**函数也可以指定为final**，之后任何尝试覆盖它的操作都会发生错误：

```c++
struct D2 : B{
  void f1(int) const final;   //不允许被覆盖  
};
struct D3 : D2{
  void f2(); //正确：覆盖从间接基类B继承而来的f2
  void f1(int) const; //错误：D2中f1是final
};
```

**final和override出现在形参列表（包括任何const或引用修饰符）和尾置返回类型之后**

#### 虚函数和默认实参

虚函数同样可以拥有默认实参，**函数调用时若使用默认实参，则实参值由本次调用的静态类型决定**。即若通过基类的引用或指针调用哈数，则使用基类中定义的默认实参。

**虚函数若使用默认实参，则基类和派生类中定义的默认实参最好一致**

#### **回避虚函数的机制**

有时希望对虚函数的调用不进行动态绑定，而是强迫执行虚函数的某个特定版本。**使用作用域运算符可实现这个目的**

```c++
//强行调用基类中定义的函数版本，不管baseP的动态类型到底是什么，此时会在编译时完成解析
double undiscounted = baseP->Quote::net_price(42);
```

**当派生类的虚函数想要调用它覆盖的基类中的虚函数版本时，需要回避虚函数的默认机制**。基类的版本通常完成继承层次中所有类型都要做的共同任务，派生类的版本执行一些和派生类本身相关的操作

**若派生类的虚函数要调用基类版本，但未使用作用域运算符，则在运行时该调用会被解析成对派生类版本自己的调用，导致无限递归**

### 4、抽象基类

#### 纯虚函数（pure virtual）

**纯虚函数无须定义，通过在函数体的位置（在声明语句的分号之前）书写=0就可将虚函数说明为纯虚函数。=0只能出现在类内部的虚函数声明语句处**

纯虚函数也可以提供定义，但是函数体必须定义在类的外部

```c++
class Disc_quote : public Quote{
  public:
    Disc_quote = default;
    Disc_quote(const string &book, double price, size_t qty, double disc):
    	Quote(book,price), quantity(qty), discount(disc){ }
    virtual double net_price(size_t) const = 0;
  protected:
	size_t quantity = 0;  //折扣使用的购买量
    double discount = 0.0; //表示折扣的小数值
};
```

**含有纯虚函数的类是抽象基类**

**含有（或未经覆盖直接继承）纯虚函数的类是抽象基类，抽象基类负责定义接口，后续的其他类可以覆盖接口。**即抽象基类的派生类必须给出自己的纯虚函数定义，否则它们仍然是抽象基类

**我们不能直接创建一个抽象基类的对象**

```c++
Disc_quote discounted; //错误：不能定义Disc_quote对象
Bulk_quote bulk; //正确：Bulk_quote中没有纯虚函数
```

```c++
class Bulk_quote : public Disc_quote{
  public:
    Bulk_quote() = default;
    Bulk_quote(const string &book, double price, size_t qty, double disc):
    	Disc_quote(book, price, qty, disc){ }
    double net_price(size_t) const override;
};
```

每个类各自控制它对象的初始化过程，它的构造函数将实参传递给直接基类，然后直接基类的构造函数继续调用Quote的构造函数

**重构**

以上的行为是重构的典型示例，**重构负责重新设计类的体系以便将操作、数据从一个类移动到另一个类中**

### 5、访问控制和继承

每个类分别控制自己的成员初始化过程，还分别控制着其成员对于派生类来说是否可访问

**受保护的成员**

protected可以看作public和private中和之后的产物，**派生类的成员或友元只能通过派生类对象来访问基类的受保护成员**

派生类的成员和友元不能直接访问基类对象中的受保护成员

```c++
class Base{
  protected:
    int prot_mem;
};

class Sneaky : public Base{
  friend void clobber(Sneaky&);  //可访问Sneaky::prot_mem
  friend void clobber(Base&);  //不能访问Base::prot_mem
  int j; //默认是private
};
void clobber(Sneaky& s){ s.j = s.port_mem = 0;}; //正确：它能访问Sneaky对象的private和protected成员
void clobber(Base &b){ b.prot_mem = 0;}; //错误：它不能访问Base的protected成员
```

#### 派生类的派生列表中的访问说明符

它用来**控制派生类的用户（包括此派生类自己的派生类）对基类成员的访问权限，不影响此派生类的成员访问基类成员**

经派生说明符控制后从基类继承来的成员相当于是派生类自己的通过此说明符控制的成员

```c++
class Base{
  public:
    void pub_mem();
  protected:
    int prot_mem;
  private:
    char priv_mem;
    
};
struct Pub_Derv : public Base{
  int f() { return prot_mem; }
  char g() { return priv_mem;} //错误：private成员对于派生类来说不可访问   
};
struct Priv_Derv : private Base{  //这里的private不影响派生类的访问权限
	int f1() const{return prot_mem; } 
};

Pub_Derv d1;
Priv_Derv d2;
d1.pub_mem();
d2.pub_mem(); //错误：它的派生访问说明符是private，在Priv_Derv中此成员就是private，因此不能直接调用

//控制继承自Priv_Derv的新类的访问权限
struct Derived_from_Private : public Priv_Derv{
  int use_base(){ return prot_mem;} //错误！！prot_mem中是private  
};
```

#### **派生类向基类转换的可访问性**

假定D继承了B：

只有D公有地继承B，用户代码才能使用派生类向基类的转换，受保护或私有都不行

不管D怎么继承B，D的成员函数和友元都能使用派生类向基类的转换

若D继承B是public或protected，则D的派生类的成员和友元可使用D向B的类型转换

**若基类的公有成员是可访问的，则派生类向基类的类型转换也是可访问的，反之则不行**

类的普通用户使用类的对象，只能访问类的公有（接口）成员；类的实现者编写类的成员和友元，它们可访问类的私有（实现）部分

#### 友元和继承

**友元关系不能传递，同样也不能继承**

```c++
class Base{
  friend class Pal;  
};
class Pal{
  public:
    int f(Base b){ return b.prot_mem;}
    int f2(Sneaky s){ return s.j;}  //错误！！Pal不是Sneaky的友元
    //对基类的访问权限由基类本身控制，即使对派生类的基类部分也是如此
    int f3(Sneaky s){ return s.prot_mem;} //正确：Pal是Base的友元
};

```

#### **改变个别成员的可访问性**

有时需要改变派生类继承的某个名字的访问级别，**可以使用using声明，它只指定名字（若是函数不需要指定形参列表）**

using声明语句中名字的访问权限由**该using声明语句之前的访问说明符决定**，派生类只能为它可以访问的名字提供using声明

```c++
class Base{
  public:
    size_t size() const {return n;}
  protected:
    size_t n;
};
class Derived : private Base{  //私有继承
    //使用using声明语句改变可访问性
    public:
    	using Base::size; 
    protected:
    	using Base::n;
} 
```

size和n本来继承过来是私有的，但是using声明语句改变了可访问性

#### 默认的继承保护级别

默认class关键字定义的派生类是私有继承，struct关键字定义的派生类是公有继承

```c++
class Base{ };
struct D1 : Base{ }; //默认public继承
class D2 : Base{ };  //默认private继承
```

### 6、继承中的类作用域

**存在继承关系时，派生类的作用域嵌套在其基类的作用域之内**。若一个名字在派生类的作用域内无法解析，则会继续在外层的基类作用域中寻找这个名字的定义。**因此派生类才能像使用自己的成员一样使用基类的成员**

### 7、构造函数和拷贝控制

####  虚析构函数

**基类一般应该定义一个虚析构函数，可以确保delete基类指针时将运行正确的析构函数版本**

当delete一个动态分配的对象的指针时，若指针指向继承体系中的某个类型，可能出现指针的静态类型和被删除对象的动态类型不符的情况。比如我们delete一个Quote*类型的指针，则该指针可能实际指向了一个Bulk_quote类型的对象。**这时编译器必须清楚它应该执行Bulk_quote的析构函数**



**在基类中将析构函数定义成虚函数可以确保执行正确的析构函数版本：**

```c++
class Quote{
  public:
    //若删除的是一个指向派生类对象的基类指针，则需要虚析构函数
    virtual ~Quote() = default; //动态绑定析构函数
};
```

**虚析构函数的虚属性同样会被继承，因此无论派生类使用合成的析构函数还是自己定义的析构函数都会是虚析构函数**

```c++
Quote *itemP = new Quote;  //静态类型和动态类型一致
delete itemP;  //调用Quote的析构函数
itemP = new Bulk_quote;  //静态类型和动态类型不一致
delete itemP; //调用Bulk_quote的析构函数
```

**若一个类定义了虚析构函数，哪怕是使用=default的形式使用了合成版本，编译器也不会为这个类合成移动操作**

派生类的析构函数只负责销毁自己分配的资源，然后再是基类的析构函数执行。

**注：构造或析构函数调用了某个虚函数，则应该执行和构造函数或析构函数所属类型对应的虚函数版本！！！**



**派生类中删除的拷贝控制和基类的关系：**

- 和其他类的情况一样，基类或派生类也会出于同样的原因将其合成的默认构造函数或任何一个拷贝控制成员定义为被删除的函数

- 若基类中的默认、拷贝构造函数、拷贝赋值符或析构函数是被删除或不可访问的，则派生类中对应的成员将是被删除的。编译器不能使用基类成员来执行派生类对象基类部分的构造、赋值或销毁

- 若基类中有一个不可访问或删除的析构函数，则派生类中合成的默认和拷贝构造函数将会是被删除的，无法销毁派生类对象的基类部分
- 使用=default请求一个移动操作时，若基类中对应的操作是删除或不可访问，则派生类中该函数是被删除的

**默认基类不含有合成的移动操作，派生类中也没有**

当想使用移动操作时，首先应该在基类中进行定义。

#### 派生类的拷贝控制成员

**定义派生类的拷贝或移动构造函数**

**默认基类的默认构造函数初始化派生类对象的基类部分。因此想拷贝或移动基类部分，必须在派生类的构造函数初始值列表显式使用基类的拷贝或移动构造函数**

```c++
class Base{ };
class D : public Base{
  public:
    D(const D& d) : Base(d){ }
    D(D&& d) : Base(std::move(d)){ }
};
```

尽管Base可以包含一个参数类型为D的构造函数，但是一般不这么做。

**Base(d)会匹配Base的拷贝构造函数，D类型的对象d会被绑定到该函数的Base&形参**

**同样，派生类的赋值运算符也必须显式为它的基类部分赋值**

#### 继承的构造函数

新标准中派生类能重用其直接基类定义的构造函数，这些构造函数不是常规方式继承而来，但也可以叫做继承的

一个类只初始化它的直接基类，**它也只继承直接基类的构造函数，但不能继承默认、拷贝和移动构造函数，它们会自己在派生类中合成**

使用using语句可以继承基类的构造函数，对于基类的每个构造函数，编译器都生成一个与之对应的派生类构造函数。

这里的using和普通成员的using声明不同，它不会改变构造函数的访问级别

```c++
class Bulk_quote : public Disc_quote{
    public:
    	using Disc_quote::Disc_quote; //继承Disc_quote的构造函数
    	double net_price(size_t) const;
};
```

编译器生成的构造函数：`derived(parms) : base(args)`。

derived是派生类的名字，base是基类的名字。parms是构造函数的形参列表，而args将派生类构造函数的形参传递给基类的构造函数

继承的构造函数等价于如下形式：

```c++
Bulk_quote(const string& book , double price, size_t qty, double disc):
		Disc_quote(book, price, qty, disc){ };
```

**派生类自己的数据成员会被默认初始化**

using声明语句不能指定explicit或constexpr，若基类的构造函数是explicit或constexpr，则继承的构造函数也拥有相同的属性

一个基类构造函数有默认实参时，这些实参不会被继承。同时派生类会获得另外的多个继承的构造函数，其中每个构造函数分别省略掉一个含有默认实参的形参

派生类可以继承一部分构造函数，但是为其他构造函数定义自己的版本。定义在派生类中的构造函数和基类中的具有相同形参列表，则它不会被继承

### 8、容器和继承

**容器存放继承体系中的对象时，必须采用间接存储的方式。**因为不允许在容器中保存不同类型的元素，所以不能把具有继承关系的多种类型的对象直接放在容器中

```c++
vector<Quote> basket;
basket.push_back(Bulk_quote("201",50,10,.25));//正确：但是只能将Quote的部分拷贝给basket
```

**向此vector中添加Bulk_quote对象时，它的派生类部分会被忽略掉。容器和存在继承关系的类型无法兼容**

**在容器中放置（智能）指针而非对象**

向容器中存放具有继承关系的对象时，我们实际上存放的通常是基类的指针。（更好的选择是智能指针），这些指针所指对象的动态类型可能是基类或派生类型

```c++
vector<shared_ptr<Quote>> basket;
basket.push_back(make_shared<Quote>("021", 50));
basket.push_back(
	make_shared<Bulk_quote>("233",50,10,.25)); //可以将派生类的智能指针转化成基类的智能指针，和普通指针一样
//实际调用的net_price版本依赖于所指向对象的动态类型
cout << basket.back()->net_price(15) << endl;
```

#### 编写basket类

```c++
class Basket{
	public:
        //basket使用合成的默认构造函数和拷贝控制成员
    	void add_item(const shared_ptr<Quote> &sale){
            items.insert(sale);
        }
    	//打印每本书的总价和购物篮中所有书的总价
    	double total_receipt(ostream&) const;
    private:
    	//用于比较shared_ptr,multiset成员会用到它
    	static bool compare(const shared_ptr<Quote> &lhs,
                           const shared_ptr<Quote> &rhs){
            return lhs->isbn() < rhs->isbn();
        }
    	//multiset保存同一本书的多个报价，并且按照compare成员排序
    	multiset<shared_ptr<Quote>>, decltype(compare)*> items{compare}; //shared_ptr没有定义小于运算符，所以必须提供自己的函数进行比较
};
double Basket::total_receipt(ostream &os) const{
    double sum = 0.0; //保存实时计算出的总价格
    //iter指向ISBN相同的一批元素中的第一个
    //upper_bound返回一个迭代器，它指向这批相同元素的尾后位置，即跳过和当前关键字相同的所有元素，也就是下一本书或集合的末尾
    for(auto iter = items.cbegin(); iter != items.cend(); iter = items.upper_bound(*iter)){
        sum += print_total(os, **iter, items.count(*iter)); //打印每本书的细节
    }
    os << "total sale:" << sum << endl;//打印总价
    return sum;
}
```

## 五、模板和泛型编程

面向对象编程和泛型编程都能处理在编写程序时不知道类型的情况。OOP能处理类型在程序运行之前都未知的情况，而在泛型编程中，在编译时就能获知类型

### 1、定义模板

#### 函数模板

可以定义一个通用的**函数模板（function template）**，它就是一个公式，可生成针对特定类型的函数版本

模板定义用**关键字template**开始，后跟一个**模板参数列表（不能为空），它是使用逗号分隔的一个或多个模板参数的列表，放入<>之中**

模板参数列表类似于函数的参数列表，它表示在类或函数定义中用到的类型或值。使用模板时，可以隐式或显式指定**模板实参**，将它绑定到模板参数上

```c++
template <typename T> //T表示一个类型
int compare(const T &v1, const T &v2){
    if(v1<v2) return -1;
    if(v2<v1) return 1;
    return 0;
}
```

**实例化函数模板**

调用一个函数模板时，编译器用**函数实参来为我们推断模板实参，会将实参类型绑定到模板参数T的类型**

```c++
cout << compare(1,0) << endl; //此时T为int
```

编译器用推断出的模板参数来**实例化**一个特定版本的函数，这些编译器生成的版本叫作**模板的实例**

```c++
//实例化出int compare(const int&, const int&)
cout << compare(1,0) << endl;  

//实例化出int compare(const vector<int>&, const vector<int>&)
vector<int> vec1{1,2,3}, vec2{4,5,6};
cout << compare(vec1, vec2) << endl;  
```

**模板类型参数**

类型参数可以用来指定返回类型或函数的参数类型，以及在函数体内用于变量声明或类型转换

**类型参数前必须使用关键字class或typename，模板参数列表中这两个关键字含义相同，可以互换使用**

```c++
template <typename T> T foo(T* p){
    T tmp = *p; //tmp的类型将是指针p指向的类型
    //...
    return tmp;
}
template <typename T,U> T calc(const T&, const U&); //错误！！U之前必须加上typename或class
template <typename T,class U> T calc(const T&, const U&); 
```

**非类型模板参数**

可以在模板中定义**非类型参数**，一个非类型参数表示一个值而非一个类型。**通过一个特定的类型名而非关键字class或typename来指定非类型参数**

模板被实例化时，非类型参数被一个用户提供的或编译器推断出的值所代替，这些值必须是常量表达式，从而允许编译器在编译时实例化模板

```c++
template<unsigned N, unsigned M>
int compare(const char (&p1)[N], const char (&p2)[M]){
    return strcmp(p1, p2);
}

compare("hi", "mom");
//实例化出如下版本
int compare(const char (&p1)[3], const char (&p2)[4]);
```

调用这个compare时，编译器会使用字面常量的大小来代替N和M，从而实例化模板。注意：字符串字面常量的末尾插入一个空字符作为终结符

**非类型参数可以是整型、指向对象或函数类型的指针或（左值）引用**。

**绑定到非类型整型参数的实参必须是常量表达式，绑定到指针或引用非类型参数的实参必须具有静态的生存期（它与程序运行期相同，直到程序运行结束），不能使用普通的（非static）局部变量或者动态对象作为实参。指针参数也可以使用nullptr或值为0的常量表达式来实例化**

**注：模板非类型参数是一个常量值**

**inline和constexpr的函数模板**

函数模板可以声明为inline或constexpr的，这两个说明符**放在模板参数列表之后，返回类型之前**

```c++
template <typename T> inline T min(const T&, const T&);
inline template <typename T> T min(const T&, const T&); //错误！！！inline位置不正确
```

##### 编写类型无关的代码

**泛型编码的两个原则：**

- 模板中的函数参数是const的引用，保证了函数可以用于不能拷贝的类型，用于处理大对象时函数会运行得更快
- 函数体中的条件判断仅适用<比较运算，降低了函数对要处理类型的要求。这些类型必须支持<，可以不支持>

模板程序应该尽量减少对实参类型的要求

**模板编译**

编译器遇到模板定义时，不会生成代码。**只有当实例化出模板的一个特定版本时，编译器才会生成代码**。

我们将类定义和函数声明放在头文件中，普通函数和成员函数的定义放在源文件中。

**模板则不同：为了生成实例化版本，编译器要知道函数模板或类模板成员函数的定义**。因此**模板的头文件既包括声明也包括定义**

#### 类模板

**类模板**是用来生成类的。**编译器不能为类模板推断模板参数类型**

**定义类模板**

类模板也以关键字template开始，后跟模板参数列表。**模板参数用来代替使用模板时用户需提供的类型或值**

```c++
template <typename T> class Blob{
    public:
    	//value_type就和T一样，相当于是类型了
    	typedef T value_type;
    	//typename vector<T>::size_type就类似于typename T，编译器在模板中需要加上关键字typename才知道它是个类型，最后再重命名为size_type
    	typedef typename vector<T>::size_type size_type;
    	//构造函数
    	Blob();
    	Blob(initializer_list<T> il);
    	
    	size_type size() const{ return data->size();} //元素数目
    	bool empty() const { return data->empty();}
    	void push_back(const T &t){ data->push_back(t);}
    	void push_back(T &&t){ data->push_back(std::move(t));} //移动版本
    	void pop_back();
    	//元素访问
    	T& back();
    	T& operator[](size_type i);
    private:
    	shared_ptr<vector<T>> data;
    	void check(size_type i, const string &msg) const;
    
};
```

**实例化类模板**

我们需提供**显式模板实参列表**，它们被绑定到模板参数。

实例化时编译器会**重写Blob模板，将模板参数T的每个实例替换为给定的模板实参**，本例中就是int

**我们指定的每一种元素类型，编译器都会生成一个不同的类！！！**

类模板的名字不是类型名，它用来实例化类型。

```c++
Blob<int> ia;
Blob<int> ia2 = {0,1,2,3};

//从以上的定义编译器会实例化出和下面等价的类：
template <> class Blob<int> {
public:    
  typedef typename vector<int>::size_type size_type;
  Blob();
  Blob(initializer_list<int> il);
  //...
private:
    shared_ptr<vector<int>> data;
    //...
};
```

在模板作用域中使用模板类型时，一般将模板自己的参数当做此被使用模板的实参，如上面的data成员

**类模板的成员函数**

类模板每个实例都有自己版本的成员函数，这些成员函数具有和模板相同的模板参数，**因此定义在类模板外的成员函数必须以关键字template开始，后接类模板参数列表**

定义成员函数时，模板实参和模板形参相同，即下面的`Blob<T>`，**类名必须包含模板实参，类模板不是类名**

```c++
//类外定义成员：
template <typename T>
ret-type Blob<T>::member-name(parm-list)    

//check成员：
template <typename T>
void Blob<T>::check(size_type i, const string &msg) const{
    if(i >= data->size())
        throw std::out_of_range(msg);
}
//Blob构造函数
template <typename T>
Blob<T>::Blob() : data(make_shared<vector<T>>()) { } //分配一个空vector
template <typename T>
Blob<T>::Blob(initializer_list<T> il):
		data(make_shared<vector<T>>(il)) { }

//调用构造函数初始化
Blob<string> articles = {"a", "an","the"};
```

**注：一个类模板的成员函数只有使用到它时才会进行实例化，若没用到则不会。因此即使某种类型不能完全符合模板操作的要求，仍然能用该类型实例化类**

**在类代码内简化模板类名的使用**

一般使用类模板类型时必须提供模板实参，但是有一个例外：**在类模板自己的作用域中，可以直接使用模板名，不需要提供实参**

```c++
template <typename T> class BlobPtr{
    public:
    	BlobPtr() curr(0){ }
    	BlobPtr(Blob<T> &a, size_t sz = 0) ; 
    			wptr(a.data), curr(sz){ }
    	T& operator*() const{
            auto p = check(curr, "dereference past end");
            return (*p)[curr];
        }
    	//递增和递减
    	BlobPtr& operator++(); //前置
    	BlobPtr& operator--();
    private:
    	size_t curr; //数组中的当前位置
    	shared_ptr<vector<T>> check(size_t, const string&) const;
    	weak_ptr<vector<T>> wptr;
};

//类模板外使用类模板名，这时不在类的作用域，直到遇到类名才表示进入
template <typename T>
BlobPtr<T> BlobPtr<T>::operator++(int){  //后置递增/递减对象但是返回原值
    //不需要检查，调用前置递增时会进行检查
    BlobPtr ret = *this; //保存当前值
    ++*this; //推进一个元素
    return ret;
}

```

##### 类模板和友元

一个类包含友元声明时，类和友元各自是否是模板时相互无关的。

**若类模板包含一个非模板友元，则友元被授权可访问所有模板实例。若此友元也是模板，类可授权给所有友元模板实例或者只授权给特定实例**

**一对一友好关系**

建立对应实例和它的友元间的友好关系

**友元的声明用Blob的模板形参来作为它们自己的模板实参，因此友好关系被限定在用相同类型进行实例化**

```c++
//前置声明，在Blob中声明友元所需要的
template <typename> class BlobPtr;
template <typename> class Blob;//==中的参数所需要的
template <typename T> 
bool operator==(cosnt Blob<T>&, const Blob<T>&);

template <typename T> class Blob{
  friend class BlobPtr<T>;
  friend bool operator==<T>(const Blob<T>&, const Blob<T>&);
  //...  
};
```

**为了让所有实例变成友元，友元声明中必须使用和类模板本身不同的模板参数**

```c++
//前置声明，在将模板的一个特定实例声明成友元时要用到
template <typename T> class Pal;

template <typename T> Class C{
    friend class Pal<T>; //C的每个实例将相同实例化的Pal声明成友元
    
    //Pal2所有的实例都是C的每个实例的友元，不需要前置声明
    template <typename X> friend class Pal2;
    
    friend class Pal3; //它是非模板类，不需要前置声明，是C所有实例的友元
};
```

**模板自己的类型参数变成友元**

虽然友元一般是类或函数，但也可以使用内置类型来实例化Bar。

```c++
template <typename Type> class Bar{
    friend Type; //将访问权限授予用来实例化Bar的类型
};
```

##### 模板类型别名

由于模板不是类型，不能定义一个typedef来引用一个模板，即无法定义typedef引用`Blob<T>`

```c++
typedef Blob<string> StrBlob; //但是可以使用typedef来引用实例化的类
```

新标准允许为类模板定义类型别名，此模板类型别名是一族类的别名

```c++
template <typename T> using twin = pair<T,T>;
twin<string> authors; //authors是一个pair<string,string>
twin<int> area; //pair<int,int>

template <typename T> using partNo = pair<T,unsigned>;  //固定一个或多个模板参数
```

##### 类模板的static成员

每一个模板实例都会有自己的static成员实例，static数据成员也定义为模板

```c++
template <typename T> class Foo{
  public:
    static size_t count() { return ctr;}
  private:
    static size_t ctr;
};
template <typename T>
size_t Foo<T>::ctr = 0;
```

#### 模板参数

一般将它命名为T，但实际上可使用任何名字

在模板内部不能重用模板参数名

```c++
typedef double A;
template <typename A, typename B> void f(A a, B b){
    A tmp = a; //tmp是模板参数A的类型
    double B; // 错误！！重用了模板参数名B
}

template <typename V , typename V> //错误！！使用了两次V
```

和函数参数相同，声明中的模板参数名不必和定义中的相同，模板声明时必须包含模板参数：

```c++
//等价
template <typename T> T calc(const T&, const T&);
template <typename U> U calc(const U&, const U&);
```

##### **使用类的类型成员**

在模板中使用作用域运算符会存在问题，比如`T::mem`，编译器不知道mem是类型成员还是static数据成员，直到实例化时才知道

为了处理模板，编译器必须知道名字是不是一个类型

**默认情况下，假定通过作用域运算符访问的名字不是类型**。若希望使用一个模板类型参数的类型成员时，需要使用**typename来显式告诉编译器是一个类型**

```c++
template <typename T>
typename T::value_type top(const T& c){ //容器类型的实参
    if(!c.empty())
        return c.back();
    else
        return typename T::value_type();
}
```

**默认模板实参**

新标准中可以为函数和类模板**提供默认模板实参**

```c++
//compare有一个默认模板实参less<T>和一个默认函数实参F()，其中less<T>是标准库定义的可调用对象，使用小于运算符进行比较
template <typename T, typename F = less<T>> //F代表用T实例化的less对象类
int compare(const T &v1, const T &v2, F f = F()){ //f是类型F的默认初始化对象
    if(f(v1, v2)) return -1;
    if(f(v2, v1)) return 1;
    return 0;
}

//类模板中使用模板默认实参
template<class T = int> class Numbers{ //T默认是int
  public:
    Numbers(T v = 0) : val(v){ }
  private:
    T val;
};
Numbers<> average_precision; //尖括号为空代表使用默认类型
```

#### 成员模板

成员模板就是**类的成员函数是一个模板，成员模板不能是虚函数**

**非模板类的成员模板**

```c++
class DebugDelete{
  public:
    DebugDelete(ostream &s = cerr) : os(s){ }
    template <typename T> void operator()(T *p) const{
        os << "deleting unique_ptr" << endl;
        delete p;
    }
  private:
    ostream &os;
};

double *p = new double;
DebugDelete d;
d(p);
int *ip = new int;
DebugDelete()(ip); //在临时DebugDelete对象上调用operator()(int*)
```

**类模板的成员模板**

在类模板外定义一个成员模板时，必须同时为类模板和成员模板提供模板参数列表。**类模板的参数列表在前，后跟成员自己的模板参数列表**

```c++
template <typename T> class Blob{
  template <typename It> Blob(It b , It e);  
};

template <typename T> //类的类型参数
template <typename It> //构造函数的类型参数
	Blob<T>::Blob(It b, It e): data(make_shared<vector<T>>(b,e)) { }
```

#### 控制实例化

当模板被使用时才会进行实例化，因此相同的实例可能出现在多个对象文件中。多个源文件使用相同的模板并提供相同的模板参数时，每个文件中就都有一个该模板的实例

在多个文件中实例化相同模板的额外开销很大，因此可以**显式实例化**来避免这种开销。

declaration是一个类或函数声明，其中所有的模板参数都被替换为模板实参。**编译器遇到extern模板声明时，不会在本文件中生成实例化代码**

**将一个实例化声明为extern就表示承诺在程序其他位置有该实例化的一个非extern声明（定义），对每个实例化声明在程序中某个位置必须有它显式的实例化定义**

**一个给定的实例化版本，可有多个extern声明，但只能有一个定义**

```c++
extern template declaration; //实例化声明
template declaration; //实例化定义

extern template class Blob<string>; //声明
template int compare(const int&, const int &); //定义
```

实例化定义时会实例化所有成员，因此用来显式实例化一个类模板的类型必须能用于所有成员

### 2、模板实参推断

和之前一样，顶层const无论在形参还是实参中都会被忽略

**能用于函数模板中的类型转换：**

- const转换，可将一个非const对象的引用或指针传递给一个const的引用或指针
- 若函数形参不是引用类型，则可以对数组或函数类型的实参应用正常的指针转换，数组实参转化为指向首元素的指针，函数实参转化为函数类型的指针

```c++
template <typename T> T fobj(T,T);
template <typename T> T fref(const T& , const T&);
string s1("aaaa");
const string s2("bbbb");
fobj(s1, s2); // 调用fobj(string,string)，const会被忽略
fref(s1,s2); //调用fref(const string&, const string&)

int a[10], b[42];
fobj(a,b); //调用f(int* , int*)
fref(a,b); //错误！！数组类型不匹配，需要a和b类型相同，但是两个数组大小不同
```

**注：因为如果形参是一个引用，则数组名不会转换为指向首元素的指针，而是代表数组类型**

#### 函数模板的显式实参

```c++
template <typename T1, typename T2, typename T3> T1 sum(T2, T3)；
```

此时T1没有任何函数实参能推断T1的类型，**因此调用时必须为它提供显式模板实参**

```c++
auto val = sum<long>(12,10); //T1显式指定为long
```

显式指定时，**第一个模板实参对应第一个模板参数，第二个模板实参对应第二个，以此类推。**

**注：模板类型参数已经显式指定了的函数实参，就可以进行正常的类型转换**

#### 尾置返回类型和类型转换

下例中编译器在遇到函数的参数列表之前beg是不存在的，此时可以使用尾置返回类型，它可以使用函数的参数

解引用返回一个左值，因此`decltype`推断的类型为beg这个元素类型的引用

```c++
//尾置返回允许在参数列表之后声明返回类型
template <typename It>
auto fcn(It beg, It end) -> decltype(*beg){
	return *beg;
}
```

使用头文件type_traits中的**remove_reference模板进行类型转换**，此模板中的type成员表示被引用的类型。**type脱去引用，剩下元素类型本身**

**例如remove_reference<int&>，则type成员会是int**

```c++
remove_reference<decltype(*beg)>::type  //此时剩下元素类型本身

template <typename It>
auto fcn2(It beg, It end) -> 
    typename remove_reference<decltype(*beg)>::type{
	return *beg; //返回序列中元素的拷贝
}    
```

#### 引用折叠和右值引用参数

```c++
template <typename T> void f3(T&&);
int i =10;
```

**一般右值引用不能绑定到左值。但是有两个例外情况：**

- 当右值引用是函数模板的类型参数时，将左值传递给此参数时编译器会推断它是实参的左值引用类型。因此当调用`f3(i)`时，编译器推断T的类型是int &，这时f3的参数是一个int&类型的右值引用。**我们不能直接定义一个引用的引用，但是通过类型别名或通过模板类型参数间接定义是可以的**
- **我们间接创建了一个引用的引用时，则这些引用形成了折叠。一般引用会折叠成一个普通的左值引用类型。但是有一个特殊情况会折叠成右值引用，右值引用的右值引用**

```c++
f3(i); //实参是左值，模板参数T是int&， 此时函数参数为int& &&

//实例化时函数参数折叠为int&
void f3<int&>(int&);
```

因此若函数参数是一个指向模板参数类型的右值引用，则可以传递给它任意类型的实参

##### **理解std::move**

标准库对move的定义

```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& t){
    return static_cast<typename remove_reference<T>::type&&>(t);
}
string s1("hi"),s2;
//推断T是string，相当于实例化string&& move(string &&t)，t的类型已经是string&&，因此return语句中的类型转换什么都不做
s2 = std::move(string("aaa")); //传递右值

//推断T是string&，参数t是string&&&，会折叠为string&，相当于实例化string&& move(string &t),即代表可以将右值引用绑定到一个左值
s2 = std::move(s1); //传递左值
```

**注：static_cast可以显式地将一个左值转换为一个右值引用**

#### 转发

将一个函数的参数定义为指向模板类型参数的右值引用，可以保持对应实参的所有类型信息。

使用引用参数（无论是左值还是右值）使得我们可以保持const属性，因为在引用类型中的const是底层的。同时通过引用折叠就可以保持左值/右值属性

```c++
template <typename F, typename T1, typename T2>
void flip2(F f, T1 &&t1, T2 &&t2){
    f(t2, t1);
} 

//但是它不能用于接受右值引用参数的函数
void g(int &&i, int& j){
    cout << i << " " << j << endl;
}
//函数参数和其他任何变量一样，都是左值表达式，此时调用g会传递给g的右值引用参数i一个左值
//即使将右值42传递给flip2,传递给g的将会是一个左值参数t2,从而发生错误
flip2(g,i,42);
```

**使用std::forward**

forward定义在utility中，**它可以保持原始实参的类型**。forward必须通过显式模板实参来调用，**forward返回该显式实参的右值引用，即`forward<T>`的返回类型是T&&**

函数参数是模板类型参数时，一般使用forward传递右值引用

若实参是右值，则Type是普通的（非引用）类型，`forward<Type>`返回Type&&；

若实参是一个左值，则通过引用折叠后Type本身是一个左值引用类型，`forward<Type>`返回类型是指向左值引用的右值引用，它会再次折叠，最终返回一个左值引用

```c++
template <typename Type> func(Type &&arg){
    funcA(std::forward<Type>(arg));
}
```

```c++
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2){
	f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```

### 3、重载和模板

函数模板可以被另一个模板或非模板函数重载，名字相同的函数必须有不同数量或类型的参数

一个调用的候选函数包括所有模板实参推断成功的函数模板实例，**因此候选的函数模板总是可行的，因为模板实参推断时会排除不可行的模板**

**涉及到函数模板的函数匹配规则：**

- 若同样好的函数中只有一个是非模板函数，则选择此非模板的函数
- 若同样好的函数中只有多个函数模板，且其中一个模板比其他的更特例化，则选择此模板。否则此调用有歧义

### 4、可变参数模板

一个可变参数模板就是接受可变数目参数的模板函数或模板类，可变数目的参数叫作**参数包**，包括**模板参数包，表示零个或多个模板参数；函数参数包，表示零个或多个函数参数**

**使用省略号来指出一个模板参数或函数参数表示一个包**

class或typename指出接下来的参数表示零个或多个类型的列表

一个类型名后面跟省略号则表示零个或多个给定类型参数（此参数不是一个类型）的列表

在函数参数列表中，若一个参数的类型是一个模板参数包，则此参数也是一个函数参数包

编译器会推断包中参数的数目

```c++
template <typename T, typename... Args> //Args是模板参数包
void foo(const T &t, const Args&... rest); //rest是函数参数包
```

可以使用sizeof...运算符得到包中元素的数目

```c++
template <typename... Args> void g(Args... args){
    cout << sizeof...(Args) << endl;
    cout << sizeof...(args) << endl;
}
```

#### **编写可变参数函数模板**

使用initializer_list只能接受所有实参相同的类型（或可转换为同一个公共类型），而可变参数函数则不需要，它的实参类型可变

**可变参数函数一般是递归的**

```c++
//终止递给并打印最后一个元素的函数
//它必须在可变参数版本的print定义之前声明
template<typename T>
ostream &print(ostream &os, const T &t){
    return os << t;
}

//包中除了最后一个元素之外的其他元素都会调用这个版本的print
template <typename T, typename... Args>
ostream &print(ostream &os, const T &t, const Args&... rest){
    os << t << ","; //打印第一个实参
    return print(os, rest...);  //递归调用，打印其他实参
}

```

每次递归调用时rest中的第一个实参都会被绑定到t，剩余实参形成下一个print调用的参数包。最后一次递归调用时两个函数提供同样好的匹配，但是非可变参数模板比可变版本更加特例化，因此选择第一个。

#### **包扩展**

扩展一个包时我们需提供用于每个扩展元素的**模式，扩展中的模式会独立应用于包中每个元素**

扩展包就是将它分解为构成的元素，对每个元素应用模式，获得扩展后的列表。**在模式右边放一个省略号来触发扩展操作**

```c++
template <typename T, typename... Args>
ostream& print(ostream &os, const T &t, const Args&... rest) //扩展Args，将模式const Args&应用到其中的每个元素
{
    os << t << ",";
    return print(os, rest...); //扩展rest
}

//更复杂的扩展模式
template<typename... Args>
ostream &errorMsg(ostream &os, const Args&... rest){
    return print(os, debug_rep(rest)...);//对每个实参调用此函数后再调用print进行打印
    
    print(os,debug_rep(rest...)); //错误！！此调用无匹配函数
}
```

#### 转发参数包

```c++
template<typename...Args>
void fun(Args&&... args){
    //将fun的所有实参转发给work函数，可以保持它们的类型信息
    work(std::forward<Args>(args)...);
}
```

`std::forward<Args>(args)...`它既扩展了模板参数包Args，也扩展了函数参数包args。它们是对应起来一起扩展

### 5、模板特例化

某些情况通用模板的定义对特定类型不适合或者不能编写单一模板，这时可以定义类或函数模板的特例化版本

**定义函数模板特例化**

必须为原模板中的每个模板参数都提供实参，使用template<>代表正在实例化一个模板，将为原模板的所有模板参数提供实参

**一个特例化版本它在本质上是一个实例，而不是重载**

模板及其特例化版本应该声明在同一个头文件中，所有同名模板的声明应该放在前面，然后是特例化版本

```c++
template<typename T>int compare(const T&, const T&); //原模板

//特例化版本,T是const char *
template<>
int compare(const char* const &p1, const char* const &p2){
    return strcmp(p1, p2);
}
```

**类模板特例化**

例子：为hash模板定义一个特例化版本

我们必须在原模板定义所在的命名空间中特例化它。

```c++
//打开std命名空间，以便进行特例化hash
namespace std{
	//...  花括号之间的任何定义都将会成为命名空间std的一部分
    
}   //关闭std命名空间，注意：没有分号

namespace std{
    template <> //定义一个特例化版本，模板参数是Sales_data
    struct hash<Sales_data>{
        typedef size_t result_type;
        typedef Sales_data argument_type;
        size_t operator()(const Sales_data& s) const;
    };
    
}

//类模板可以部分特例化
template<class T> struct remove_reference{};
template<class T> struct remove_reference<T&>{};
template<class T> struct remove_reference<T&&>{};
```

**类模板还可以只特例化一个成员**

## 六、标准库特殊设施

### 1、tuple类型

标准库中有一个tuple头文件，tuple是类似pair的模板，但是tuple可以有任意数量的成员，它们的类型可以不相同。

每个确定的tuple类型的成员数目是固定的

![](D:\office\word\计算机基础\C++\截图2\tuple支持的操作.jpg)

#### 定义和初始化tuple

**tuple的默认构造函数会对每个成员进行值初始化**

tuple也可以为每个成员提供初始值，这个构造函数是explicit的，所以**只能使用直接初始化**

```c++
tuple<size_t, size_t, size_t> threeD; //三个成员都是0
tuple<string,vector<int>> someVal("aaa",{1,2,3});

tuple<int,int,int> threeD = {1,2,3}; //错误！！！
tuple<int,int,int> threeD{1,2,3}; //正确

//item是一个tuple，类型是tuple<const char*,int,double>
auto item = make_tuple("abcd", 3, 3.2);
```

**访问tuple的成员**

由于tuple的成员数目没有限制，因此tuple的成员都是未命名的。访问tuple成员需要使用**get这一标准库函数模板。它的显式模板实参代表访问第几个成员，最后会返回该成员的引用**

```c++
auto book = get<0>(item); //返回item的第一个成员，尖括号中的值是整型常量表达式
auto cnt = get<1>(item); //以此类推
```

**使用辅助类模板查询tuple成员的数量和类型**

```c++
typedef decltype(item) trans; //trans是item的类型
//返回trans类型对象中成员的数量
size_t sz = tuple_size<trans>::value;
//cnt的类型和item中第二个成员相同
tuple_element<1,trans>::type cnt = get<1>(item);
```

**关系和相等运算符**

**只有两个tuple具有相同数量的成员时，才可以比较它们。比较时tuple中的每对成员必须能使用==和<运算符**

由于tuple定义了<和==运算符，因此可以将tuple序列传递给算法，同时也可以在无序容器中将tuple作为关键字类型

```c++
tuple<string, string> duo("1","2");
tuple<size_t, size_t> two(1,2);
bool b = (duo == two); //错误！！不能比较size_t和string
tuple<size_t, size_t, size_t> three(1,2,3);
b = (two < three); //错误！！成员数量不同
tuple<size_t, size_t> origin(0,0);
b = (origin < two); //b为true
```

#### 使用tuple返回多个值

tuple可以从一个函数返回多个值

### 2、bitset类型

bitset类定义在头文件bitset中，使得位运算更加容易，并且能够处理超过最长整型大小的位集合

#### 定义和初始化bitset

bitset是类模板，具有固定的大小，定义一个bitset时需要声明它有多少二进制位

整型值来初始化bitset时，此值会自动转换成unsigned long long

```c++
bitset<32> bitvec(1U); //32位，低位是1，其他位是0
```

![](D:\office\word\计算机基础\C++\截图2\bitset初始化.jpg)

string中最右字符是低位

```c++
bitset<32> a("1100");  //第2、3位是1
```

![](D:\office\word\计算机基础\C++\截图2\bitset操作.jpg)

### 3、正则表达式

**正则表达式（regular expression）**是一种描述字符序列的方法，它是一种及其强大的计算工具。

正则表达式库**RE库定义在头文件regex中**，它包含多个组件

![](D:\office\word\计算机基础\C++\截图2\正则表达式组件.jpg)

regex类表示一个正则表达式，**默认情况下regex使用的正则表达式语言是ECMAScript**

![](D:\office\word\计算机基础\C++\截图2\regex中函数的参数.jpg)

**注：m若匹配成功，则函数将成功匹配的信息保存在此给定的对象**

#### 使用正则表达式库

```c++
string pattern("[^c]ei");//查找不在字符c之后的字符串ei
//包含pattern的整个单词
pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
//构造一个用于查找模式的regex
regex r(pattern);
smatch results;
string test = "receipt friend theif receive";
//用r在test中查找和pattern匹配的子串，只要找到第一个匹配的子串，就返回true。从而停止查找
if(regex_search(test,results,r))
    cout << results.str() << endl;  //输出friend
```

正则表达式`[^c]`表示希望匹配任意不是c的字符，`[^c]ei`则代表希望匹配这种字符后接ei的字符串，这里正好包含3个字符。

第二行代码中`[[:alpha:]]`表示匹配任意字母，符号+和*分别表示希望匹配一个或多个、0个或多个匹配。因此第二行中代表前后都匹配0个或多个字母

**指定regex对象的选项**

![](D:\office\word\计算机基础\C++\截图2\regex对象的选项.jpg)

```c++
//一个或多个字母或数字后面接.再接cpp或cxx或cc
regex r("[[:alnum:]]+\\.(cpp|cxx|cc)$", regex::icase);
smatch results;
string filename;
while(cin >> filename)
    if(regex_search(filename, results, r))
        cout << results.str() << endl;
```

由于`.`在正则表达式中表示任意字符，因此需要在它之前加上\，由于\在c++中也是特殊字符，需要再加上一个\

**指定或使用正则表达式时的错误**

正则表达式是**在运行时**，当一个regex对象被初始化或被赋予一个新模式时才被编译，它也会发生错误。

若正则表达式存在错误，则在运行时标准库会抛出类型为**`regex_error`**的异常

`regex_error`有一个what描述发生了什么错误，code成员返回某个错误类型对应的数值编码

![](D:\office\word\计算机基础\C++\截图2\正则表达式错误类型.jpg)

code就是上表中的错误编号，从0开始

正则表达式是在运行时编译的，它的编译非常慢。应该避免创建不必要的正则表达式

**正则表达式类和输入序列类型**

RE为不同的输入序列都定义了对应的类型，使用的RE库类型必须和输入序列类型匹配:

```c++
regex r("[[:alnum:]]+\\.(cpp|cxx|cc)$", regex::icase);
smatch results;
if(regex_search("myfile.cc", results, r))  //错误！！输入序列是char*
    cout << results.str() << endl;

//正确写法
cmatch results;
if(regex_search("myfile.cc", results, r))  
      cout << results.str() << endl;
```

| 输入序列对应的正则表达式类 |                                                  |
| -------------------------- | ------------------------------------------------ |
| 输入序列类型               | 正则表达式类                                     |
| string                     | `regex、smatch、ssub_match、sregex_iterator`     |
| `const char *`             | `regex、cmatch、csub_match、cregex_iterator`     |
| `wstring`                  | `wregex、wsmatch、wssub_match、wsregex_iterator` |
| `const wchar_t *`          | `wregex、wcmatch、wcsub_match、wcregex_iterator` |

#### 匹配和Regex迭代器类型

regex迭代器是一个迭代器适配器

![](D:\office\word\计算机基础\C++\截图2\sregex_iterator操作1.jpg)

![](D:\office\word\计算机基础\C++\截图2\sregex_iterator操作2.jpg)

将`sregex_iterator`绑定到一个string和一个regex对象时，迭代器自动定位到给定string中第一个匹配的位置，递增时它会到达第二个位置

```c++
string pattern("[^c]ei");
pattern = "[[:alpha:]]*" + pattern + "[[:alpha:]]*";
regex r(pattern, regex::icase); //忽略大小写
//end_it是一个空的sregex_iterator，起到尾后迭代器的作用
for(sregex_iterator it(file.begin(), file.end(), r), end_it;
   it != end_it; ++it)
    cout << it->str() << endl;
```

![](D:\office\word\计算机基础\C++\截图2\smatch操作.jpg)

### 4、随机数

新标准出现之前都依赖于C库函数rand来生成随机数，**它生成均匀分布的伪随机数，每个随机数的范围在0和一个系统相关的最大值（至少是32767）之间**

定义在头文件random中的随机数库包括**随机数引擎类**和**随机数分布类**，引擎类可以生成unsigned随机数序列，分布类使用一个引擎类生成指定类型的、给定范围的、服从特定概率分布的随机数

#### 随机数引擎和分布

随机数引擎是函数对象类，它定义了一个调用运算符，此运算符不接受参数并且返回一个随机的unsigned整数

```c++
default_random_engine e; //生成无符号随机数
for(size_t i = 0; i<10; ++i)
    cout << e() << " ";
```

随机数引擎的输出大部分情况下不能直接使用，因为生成的随机数的值范围一般和我们想要的不符。

![](D:\office\word\计算机基础\C++\截图2\随机数引擎操作.jpg)

```c++
cout << "min:" << e.min() << "max:" << e.max() << endl; //获得引擎的范围
```

**分布类型和引擎**

随机数引擎生成unsigned数，范围内的每个数被生成的概率都相同。但是程序会需要不同类型或不同分布的随机数。标准库定义了不同随机数分布对象来满足这两方面的要求，令分布对象和引擎协同工作

为了得到一个指定范围的数，我们使用一个分布类型的对象：

```c++
uniform_int_distribution<unsigned> u(0,9); //生成[0,9]之间均匀分布的随机数
default_random_engine e; //生成无符号随机整数
for(size_t i = 0; i<10; ++i)
    //将u作为随机数源
    //每个调用返回在指定范围内且服从均匀分布的值
    cout << u(e) << " ";
```

分布类型也是函数对象类，它定义了调用运算符，接受一个随机数引擎作为参数。分布对象使用它的引擎参数生成随机数，并映射到指定的分布

**随机数发生器**就是指分布对象和引擎对象的组合。**对于一个给定的发生器，每次运行程序它都会返回相同的数值序列。**序列不变这一特性在调试时非常有用，但是使用随机数发生器的程序也要考虑这一点

```c++
//每次调用这个函数都会生成相同的100个数
vector<unsigned> bad_rand(){
    default_random_engine e;
    uniform_int_distribution<unsigned> u(0,9);
    vector<unsigned> ret;
    for(size_t i = 0; i < 100; ++i)
        ret.push_back(u(e));
    return ret;
}

//正确方法
vector<unsigned> good_rand(){
    //希望引擎和分布对象保持状态，将它们定义为static，则每次调用都生成新的数
    static default_random_engine e;
    static uniform_int_distribution<unsigned> u(0,9);
    vector<unsigned> ret;
    for(size_t i = 0; i < 100; ++i)
        ret.push_back(u(e));
    return ret;
}
```

**设置随机数发生器种子**

种子（seed）用于每次运行程序时令程序生成不同的随机结果，种子就是一个数值，引擎可以利用它从序列中一个新位置重新开始生成随机数

**可以在创建引擎对象时提供种子，或者调用引擎对象的seed成员**

```c++
default_random_engine e1; //使用默认种子
default_random_engine e2(2147482646);//使用给定的种子
default_random_engine e3; //使用默认种子
e3.seed(32767); //设置新的种子值
default_random_engine e4(32767);

```

常用的种子选取方法是调用系统函数time，它定义在头文件ctime中，返回从一个特定时刻到现在经过了多少秒

time接收一个指针，指向用于写入时间的数据结构。若指针为空，则函数简单地返回时间

time返回以秒计的时间，因此这种方式只用于生成种子的间隔为秒级别或更长的应用

```c++
default_random_engine e(time(0));
```

若程序作为一个自动过程的一部分反复运行，将time作为种子的方式就无效了，它可能多次使用的都是相同的种子

#### 其他随机数分布

**生成随机实数**

程序经常用到0和1之间的随机数。常用方法是用rand()的结果除以RAND_MAX，这种方法不正确的原因是随机整数的精度通常低于随机浮点数，这样有些浮点值就永远不会被生成

新标准库可以定义一个uniform_real_distribution对象，让标准库来处理随机整数到随机浮点数的映射

```c++
default_random_engine e; //生成无符号随机整数
//0到1的均匀分布
uniform_real_distribution<double> u(0,1);
for(size_t i = 0; i<10; ++i)
    cout << u(e) << " ";
```

**分布类型有默认的模板实参，生成浮点值是double，生成整型值是int。空<>代表使用默认类型**

```c++
uniform_real_distribution<> u(0,1); //默认生成double
```

**生成非均匀分布的随机数**

normal_distribution是正态分布，它生成浮点值，使用头文件cmath中的lround函数可以将每个随机数舍入到最接近的整数

```c++
default_random_engine e;
normal_distribution<> n(4,1.5); //均值4，标准差1.5
vector<unsigned> vals(9);  //初始值为0的9个元素
for(size_t i = 0; i!=200; ++i){
    unsigned v = lround(n(e)); //舍入到最接近的整数
    if(v<vals.size())
        ++vals[v];
}
```

**`bernouolli_distrbution`类**

伯努利分布，此类不是模板，它是一个普通类，返回一个bool值。它返回true的概率是一个常数，默认是0.5。

```
bernoulli_distribution b; //默认是）0.5的概率返回true
bernoulli_distribution b(.55); //返回true的概率是0.55
```

### 5、IO库再探

#### 格式化输入和输出

每个iostream对象还维护一个格式状态来控制IO的格式化细节。

标准库定义了一组操纵符来修改流的格式状态，操纵符可以是一个函数或对象，会影响流的状态，它可以用作输入或输出运算符的对象，返回处理的流对象。

**控制布尔值的格式**

**默认情况下bool值打印1或0，使用boolalpha操纵符可覆盖这种格式，后续都会打印true或false**

**noboolalpha可以恢复默认状态**

```c++
cout << "default bool values:" << true << " " << false
     << "\nalpha bool values:" << boolalpha
     << true << " " << false << endl;

```

**指定整型值的进制**

默认是十进制，可以使用hex、oct、dec分别改为十六进制、八进制、改回十进制。浮点值不受它们影响

showbase可以在输出中指出进制，0x代表16进制，0代表八进制，无前导字符串代表十进制。noshowbase可以恢复

## 



