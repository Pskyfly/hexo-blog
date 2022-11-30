---
title: Cpp学习笔记
date: 2022-11-30 17:34:52
tags:
---

# Cpp学习笔记

## C++类的六大函数

[C++类的六大函数--构造、析构、拷贝构造、移动构造、拷贝赋值、移动赋值 - lincoding` - 博客园 (cnblogs.com)](https://www.cnblogs.com/lincz/p/10768607.html)

```c++
#include <iostream>
#include <cassert>

class Base {
public:
    virtual ~Base() = default;

public:
    void bar(void) const noexcept {
        std::cout << "Base::bar" << std::endl;
    }
};

class Derived : public Base {
public:
    virtual ~Derived() = default;
};

void foo(const Base &base) {
    base.bar();
}

// class SmartPtr定义&实现
template<typename T>
class SmartPtr {
private:
    T *t;
public:
    // 初始化
    explicit SmartPtr(T *t1 = nullptr) : t(t1) {}

    // 拷贝构造
    SmartPtr(const SmartPtr &s) = delete;

    // 赋值构造
    SmartPtr &operator=(const SmartPtr &s) = delete;

//    // 移动构造
//    SmartPtr(SmartPtr &&s) noexcept {
//        this->t = s.t;
//        s.t = nullptr;
//    }

    // 移动赋值
    SmartPtr &operator=(SmartPtr &&s) noexcept {
        std::swap(s.t, t);
        return *this;
    }

    // 重载*
    T &operator*() {
        assert(this->t != nullptr);
        return *(this->t);
    }

    // 重载->
    T *operator->() {
        assert(this->t != nullptr);
        return this->t;
    }

    // 重载 bool
    operator bool() {
        return t;
    }

    // 析构函数
    ~SmartPtr() {
        if (this->t) {
            delete t;
            t = nullptr;
        }
    }
    //支持子类向父类的转换，如果不用这个，编译器也会自动帮我们转换
    template<typename U>
    inline explicit SmartPtr(SmartPtr<U> &&other) noexcept {
        t = other.t;
        other.t = nullptr;
    }
};

int main() {
    SmartPtr<Base> ptr1{new Derived()};
    // SmartPtr<Base> ptr2{ptr1}; // 编译Error
    SmartPtr<Base> ptr3;
    // ptr3 = ptr1; // 编译Error

    ptr3 = std::move(ptr1); // ok
    SmartPtr<Base> ptr4{std::move(ptr3)}; // ok

    ptr4->bar(); // ok
    foo(*ptr4); // ok

    return 0;
}
```

## explicit消除等号的隐式转换

[C++ explicit关键字详解 - 矮油~ - 博客园 (cnblogs.com)](https://www.cnblogs.com/rednodel/p/9299251.html#:~:text=C%2B%2B explicit关键字详解 首先%2C C%2B%2B中的explicit关键字只能用于修饰只有一个参数的类构造函数%2C,它的作用是表明该构造函数是显示的%2C 而非隐式的%2C 跟它相对应的另一个关键字是implicit%2C 意思是隐藏的%2C类构造函数默认情况下即声明为implicit (隐式).)

## C++ const修饰方法

[(248条消息) C++类中const修饰的函数与重载_未来之大神的博客-CSDN博客_c++ const 重载](https://blog.csdn.net/a512745183/article/details/52590223)

const方法不能修改成员变量

const变量只能调用const方法

带有const的方法和不带有const的方法可以并存，调用时，const对象调用const方法，非const变量调用非const方法

## C++的左值右值，左右引用，移动语意及完美转发

[谈谈C++的左值右值，左右引用，移动语意及完美转发 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/402251966)

![img](pictures/]PQOUN4]Q$]R%F_J0[79J71-1669803907654-8.png)

一个对象有两个部分：灵魂和躯壳，创建对象的时候会为这个对象分配内存空间这片内存就是灵魂，存放这块内存地址的变量就是躯壳（符号表），正常情况下，灵魂和躯壳是在一起的。

左值：有名称的，可以获取到存储地址的变量就是左值，可以用&取到地址（有躯干的对象）

右值：可以获取到值的表达式都可以成为右值，左值也可以作为右值来使用（有灵魂的表达式，1000等无法寻址的字面量，可以理解为只有灵魂）

右值又可以分为纯右值和将亡值

纯右值：临时对象或字面量（只有灵魂）

将亡值：使用move移动构造后，剩下的值就是将亡值，它内部的变量已经被设置为空值，无法再被使用，只剩下了一个空壳，所以叫作将亡值（只有躯壳）

引用是变量的别名，必须初始化

左引用：对左值的引用就是左引用（&）

右引用：对右值的引用就是右引用（&&）

const T&可以引用右值

移动语义：将左值变成右值，将内存地址提取出来，将原来存放这片内存地址变量置为空，然后将这片内存的地址作为返回值返回（将灵魂从躯壳中抽离出来）

完美转发：左值还是左值，右值还是右值（原来是灵魂，现在还是灵魂，原来有躯壳，线程还有躯壳）

[聊聊C++中的完美转发 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/161039484)

```c++
#include <iostream>

template<typename T>
void print(T & t){
    std::cout << "左值" << std::endl;
}

//接受右值，但是t变量本身是会分配内存的，内配内存并接受右值后，变成了左值
template<typename T>
void print(T && t){
    std::cout << "右值" << std::endl;
}
//接受右值，但是t变量本身是会分配内存的，内配内存并接受右值后，变成了左值
//move 抽取灵魂，全部变成右值
//forward 利用引用折叠，原来是左值回来的还是左值原来的是右值回来的还是右值
template<typename T>
void testForward(T && v){
    print(v);
    print(std::forward<T>(v));
    print(std::move(v));
}

int main(int argc, char * argv[])
{
    testForward(1);

    std::cout << "======================" << std::endl;

    int x = 1;
    testFoward(x);
}
```

## C++内存模型

堆：new和malloc出来的对象存放在这里

栈：存放局部变量，函数参数，函数返回地址等

静态区：全局变量，静态变量，虚函数，全局常量指针

常量存储区：全局常量，函数指针

代码区：存放代码

## cin输入规则

```c++
#include <iostream>

using namespace std;

signed main() {
    int x;
    cin>>x;
    getchar();
    cin>>x;
    cout<<x<<endl;
}
```

对于输入整数来说，cin会一直读取，直到遇到第一个非整数字符（整数里包含的字符，也就只有数字）

例如上面的输入35.8 15.8，得到的是0，因为第一次cin后，x位35，因为遇到了小数点`.`，此时光标就停留在了小数点这里，第二次cin的时候，第一个遇到的就是小数点，所以一个字符都没有读取，得到的就是0，如果读到的值大于了int的最大值，则得到的是int的最大值，如果小于int的最小值，得到的就是int的最小值，然后后面的内容都不再读取

同理如果读取的是浮点数，那么会到第一个非浮点字符（数字和小数点）截止。如果这样得到的是正常的数字就返回，如果得到的是小数点开头，则会带上0，如果什么都没有读到就返回0

例如：

```c++
#include <iostream>

using namespace std;

signed main() {
    double x,y;
    cin>>x>>y;
    cout<<x<<" "<<y;
}
```

输入11.11.12

得到11.11 0.12

对于char类型，会直接读取一个字符，会跳过空格，回车，制表符

对于char数组和string类型，会一直读取，直到遇到第一个空格和回车，想读取空格可以使用getline(cin,ss)

如果读取的字符个数超过了char数组的容量，会超容量读取……

![image-20221011233123411](pictures/image-20221011233123411-1669803907655-9.png)

对于cin读取字符会跳过空格和回车的问题，可以使用getchar来读取，也可以使用cin.get

```c++
#include <iostream>

using namespace std;


signed main() {
    char c1, c2;
    c1 = cin.get();
    cin.get(c2);
    cout << c1 << " " << c2 << endl;
}
```

cin内部对get方法进行的重载，不带参数的cin.get()得到的是缓冲区下一个字符的ASCII码值，这里通过隐式类型转换变成了对应char类型的值，而带参数的get(char &)方法，也是读取一个字符，赋值给传入的char变量中

如果没有遇到文件尾EOF，也没有遇到任何错误，可以使用cin.fail()会返回false,cin.good会返回true，如果遇到了文件尾，cin.eof()会返回true。遇到eof后，再使用cin读入也没有用，在有些OS中，可以使用cin.clear()来清除上面这个不可读入的状态

cout.put()可以输出一个字符，putchar也可以输出一个字符，传入参数的是字符的ASCII码

cin.get(ch)以及cin>> 返回值都是cin对象，如果需要bool类型，则调用的是good方法（重载了bool方法）

## 输出到文件 ofstream istream

输出文件使用步骤：

1. 定义输出文件对象ofstream,istream
2. 调用这个对象的open函数，打开文件，设置输入模式ios:app表示添加，ios:trunc表示清空文件,ios::out表示输出,ios::in表示输出
3. 像cout和cin一样使用这个对象

[(528条消息) ofstream的使用方法--超级精细_Ψ大鹏的博客-CSDN博客_ofstream](https://blog.csdn.net/weixin_44139428/article/details/102813246)

cpp读写文件有两个指针：读文件指针指针和写文件指针，可以实现文件的随机读写

文件的基本输出

```c++
#include <iostream>
#include <fstream>

using namespace std;

signed main() {
    cout<<"你好";
    string output = "hello world ";
    ofstream fOut;
    fOut.open("D:\\CppProjects\\test\\引用测试\\out.txt", ios::app);
    fOut << output;
    fOut.close();
}
```

文件路径可以是字符串字面量，可以是字符串或者字符数组（要以'\0'结尾）变量

如果输出失败，检查一下中文乱码问题

[(528条消息) 【C语言】CLion中文乱码问题的解决方案_星拱北辰的博客-CSDN博客_clion中文乱码](https://blog.csdn.net/weixin_43896318/article/details/104700306)

c++源文件应当使用GBK编码

我使用的解决办法：使用管理员权限

![image-20221012003422515](pictures/image-20221012003422515-1669803907655-10.png)

使用相对路径：

![image-20221012004649017](pictures/image-20221012004649017-1669803907655-14.png)

可以在open里面使用相对路径，但是这个相对路径相对的是执行者所在的目录，直接使用编译器的运行键，文件会输出在编译器的目录下面，而不是项目路径下面。所以想要使用相对路径，可以在控制台使用g++编译，然后运行

使用ifstream进行输入

```c++
#include <iostream>
#include <fstream>

using namespace std;

signed main() {
    ifstream fin;
    fin.open("in.txt", ios::in);
    cout<<fin.is_open()<<endl; //判断文件是否打开
    cout<<fin.eof()<<endl;   //判断文件是否读到文件末尾
    string ss;
    fin >> ss;
    cout << ss;
}
```

还是使用g++编译运行

同时使用ifstream和ofstream

```c++
#include <iostream>
#include <fstream>

using namespace std;

signed main() {
    ifstream fin;
    ofstream fout;
    fin.open("in.txt", ios::in);
    fout.open("out.txt", ios::trunc);
    cout << fin.is_open() << endl;
    while (!fin.eof()) {
        string line;
        getline(fin, line);
        fout << line << endl;
    }
}
```

四舍五入：

```
1.引入头文件 #include<iomanip>

2.输出用固定格式  cout<<setiosflags(ios::fixed)<<setprecision(2)<<result<<endl;

​                                  //将result保留2位小数，四舍五入后输出。
```

```c++
#include <iostream>
#include <fstream>
#include <iomanip>

using namespace std;

signed main() {
    double x = 123.565656;
    ofstream f;
    f.open("out.txt", ios::trunc);
    f << setprecision(5) << x << endl; //保留5位有效数字
    f << setiosflags(ios::fixed) << setprecision(5) << x << endl; //保留5位小数
}
/*
123.57
123.56566
*/
```

## Cpp指针

指针就是内存的地址

```
int b;
int *a=&b;
```

int* a存储的是b的内存地址，&b存储的也是内存地址，两者等价

b和*a得到的都是对应的值，两者也等价

```c++
#include <iostream>
#include <fstream>
#include <iomanip>

using namespace std;

signed main() {
    int *a = new int;
    *a = 123;
    int *b = a;
    delete b;
    cout << *a;
}
```

如果两个指针指向同一片内存空间，然后其中一个delete了，另一个指针访问值的时候，得到的会是随机值

cpp中，指针和数组是等价的，数组数组变量本质上也是指针，所以都可以使用[]，来访问元素，因为[]的实现方式也就是让指针移动对应的偏移量然后再取值，所以本质上是一样的，比如a[0]和*a就是等价的。这对于访问数组元素也同样适用。指针+1，实际上是让指针移动等同于指向类型所占字节数的内存。两者唯一的区别就是指针是一个变量，可以修改它的值。而数组指针是一个常量，不能修改它的值，它永远都指向数组的第一个元素。等价的原因是，cpp解释数组的方式是使用指针算术，对于指针而言，也是使用相同的指针算术，所以两者使用的语法是共通的。对数组变量使用sizeof 得到的是数组元素的大小✖数组长度，而对指针变量使用sizeof得到的是指针变量的大小

```c++
#include <iostream>

using namespace std;

signed main() {
    int *a = new int;
    *a = 123;
    cout << a[0];
}
```

为数组动态分配内存可以使用new int[n]，返回的是数组的第一个元素的地址，删除的时候，要用delete[] 来删除，这样删除的就是整个数组所占的内存。使用new []和delete[] 匹配，new和delete匹配，如果两者混着用结果是不可预知的

```c++
#include <iostream>

using namespace std;

signed main() {
    int *a = new int[12];
    cout << a[0]; //未知值
    delete[] a;
}
```

## C++强制类型转换

可以使用cpp版的 类型()，括号里面是要转换的数据

也可以使用C语言版的（类型），括号后面跟上要转换的数据

```c++
#include <iostream>

using namespace std;


signed main() {
    cout << char(49) << endl;
    cout << int('1') << endl;
    cout << double(1) << endl;

    cout << (char) 49 << endl;
    cout << (int) '1' << endl;
    cout << (double) 1 << endl;
}
```

不能用char变量存储eof（-1），需要先用int接收，如果不为eof再转换为char类型

## C++逻辑运算符

```c++
#include <iostream>

using namespace std;


signed main() {
    if (1 == 1 and 2 == 2 or not 3 == 3) {
        cout << "hello world" << endl;
    }

}
```

可以使用&&，||，!来表示与或非，也可以使用and,or,not来表示，两种表示方法是完全等价的

## Cpp字符操作

cctype中有许多对单个字符的操作，比如判断是不是数字，字母，标点，空白字符，以及变成大写和变成小写

并且cpp内部对各种基本类型都有变成字符串的to_string方法，以后就不需要我们自己写了！

![image-20221012212231962](pictures/image-20221012212231962-1669803907655-11.png)

## C++数组作为函数的参数

```c++
#include <iostream>

void test(int nums[]) {
    using namespace std;
    cout << nums << endl;
}

void test(int *nums) { //报错
    using namespace std;
    cout << nums << endl;
}

int main() {
    std::cout << "Hello, World!" << std::endl;
    return 0;
}
```

将数组作为函数的参数，实际上传递的是数组首字母的指针（地址）给函数的参数，所以函数参数列表中，使用int[]来接收或者使用int* 实际上的等价的，所以上述代码中不能用这种方式进行方法重载，因为数组变量和指针变量都可以作为参数传入到这两个函数中

```c++
#include <iostream>

void test(int nums[]) {
    using namespace std;
    nums++;
    cout << nums << endl;
}


int main() {
    std::cout << "Hello, World!" << std::endl;
    test(new int [0]);
    return 0;
}
```

数组变量和指针变量的用法几乎完全相同，只是再sizeof和数组变量不能修改上有区别

函数列表上的int a[]和int* a完全等价，都是指针，都可以修改指向，使用sizeof得到的都是4字节的指针大小

对于二维数组传递参数，要指明第二维的个数，不然编译器怎么知道每行有多少，每列有多少

```c++
#include <iostream>

void test(int nums[][5]) {
    using namespace std;
    cout << nums[0][0] << endl;
}


int main() {
    int a[5][5];
    a[0][0] = 123;
    test(a);
    return 0;
}
```



## 指针和const

定义const的时候加上const 表示不能通过这个指针来修改他所指向的值，包括数组

const指针可以指向变量，可以指向常量，但是非const指针不能指向const变量（常量），否则就可以通过这个非const指针修改这个常量从而失去了const的意义

```c++
#include <iostream>

using namespace std;

int main() {
    int x1 = 1;
    const int x2 = 2;
//    int *p1 = &x2; //非法，不能让非const指针指向const变量
    const int *p2 = &x1;
    p2 = &x2; //const指针可以修改指向
//    *p2 = 3;//但是不能通过这个指针修改数据的值
}
```

const指针可以修改指向，但是不能修改指向数据的值

对于指向指针的指针，const指针只能可以指向const指针，非const指针只能指向非const指针，不能混着用

## Cpp泛型

定义泛型可以使用class也可以使用typename两者完全等价

对于运算后未知的类型（可以转化的类型），可以使用decltype，但其实直接使用auto更加方便

```cpp
template<class T1, class T2>
auto del(T1 a, T2 b) {
    decltype(a - b) c = a - b;
    auto d = a - b;
    return c;
}
```

对于返回值也是泛型运算后的结果，可以使用下面这种格式

```cpp
template<class T1, class T2>
auto add(T1 a, T2 b) -> decltype(a + b) {
    return a + b;
}
```

但是后面的` -> decltype(a + b)`可以省略，结果也是对的

```cpp
template<class T1, class T2>
auto add(T1 a, T2 b) {
    return a + b;
}
```

实际上，任何返回值都可以用auto，类型申明后也可以用auto，这样编译器都可以自动帮我们推断类型

如果我们想要为泛型构造一个特例，可以使用函数具象化的语法

```cpp
template<class T1, class T2>
auto add(T1 a, T2 b) {
    return a + b;
}

template<> auto add<int, int>(int a, int b) {
    return a + b;
}
```

在函数开头加上template<>，表示这是函数具象化的一种，然后需要在函数名后申明具象化之后的泛型

我们在调用泛型方法的时候，编译器会根据传入的参数自动将这个方法隐式实例化，我们也可以显式指明泛型的类型

```cpp
#include <iostream>

using namespace std;

template<class T1, class T2>
auto add(T1 a, T2 b) {
    return a + b;
}

int main() {
    cout << add<int, int>(1, 1.2) << endl;
}
```

显式指明类型后，double会向int进行类型转换（向下转换），如果不实例化方法，第二个会将1.2作为double类型的变量传入

对于没有参数的方法，不实例化无法进行调用，因为编译器不知道传入的泛型是啥

```cpp
#include <iostream>

using namespace std;

template<class T>
T read() {
    T x;
    cin >> x;
    return x;
}

int main() {
    cout << read<int>() << endl;
}
```

如上面这段代码，如果不加上`<int>`就会报错

下面这个具象化一个泛型方法的语法，书上有，但是实际上测试的时候会编译报错，可能这种写法已经被抛弃了

```
//这个语句被编译器抛弃了
template auto add<int, int>(int,int);
```

## Cpp的多文件编写

cpp的文件结构一般按照如下策略进行划分

- 头文件，包含结构申明和函数原型
- 源文件，包含函数原型的实现
- 源文件，调用函数的函数

不要在头文件里面定义方法的实现和创建变量，因为一旦这个变量被多个源文件引用后，创建变量的行为会发生冲突，编译器会告诉你变量或者方法被重复创建（如果允许的话，就可能会出现同名但是实现不同的方法，编译器就不知道要调用哪一个，但是只申明函数原型的话就不会产生冲突）

头文件里面的内容一般包含如下内容

- 函数原型
- const变量
- 内联函数
- 泛型申明
- 结构申明
- 类申明

申明泛型不会被编译，只会告诉编译器如何去生成代码，所以可以被重复包含

结构申明，类申明并不创建变量，所以可以申明

常量和内联函数有特殊的规则，所以可以包含

include<>会直接从cpp系统目录里面找，而include""会先从用户目录下面找，再从系统目录下面找

一个头文件只能被包含一次，为了防止一个头文件被重复包含（一个头文件可能包含了另一个头文件）可以使用下面这个技术

```cpp
#ifndef __test_define
#define __test_define
//要定义的头文件
#include "iostream"

#endif
```

#ifndef表示，如果define了后面那个标识符，就直接跳到#endif，如果没有被定义则不跳过，执行后面的代码

我们在创建头文件的时候，将头文件的内容都放在#ifndef和#endif之间，并#define一个能代表这个头文件的变量，这样就可以防止头文件被重复包含，这样就可以保证一个头文件被多次引用后，头文件不会被重复引入。

```cpp
//
// Created by 黎明终点x on 2022/10/16.
//

#ifndef LEARN_FILE1_H
#define LEARN_FILE1_H
#include <iostream>

struct A{
    int a;
    auto getA(){
        return a;
    }
};

void print(A a);

#endif //LEARN_FILE1_H
```

比如一个头文件可以像上面这样写

这样，在其他文件中，无论这个文件被引入多少次，拼接到文件里的只有一次

```cpp
#include "file1.h"
#include "file1.h"
#include "file1.h"
#include "file2.h"
#include "iostream"

using namespace std;

int main() {
    cout << "test start" << endl;
    print({1});
    print2({2});

}
```

如果不加

```
#ifndef LEARN_FILE1_H
#define LEARN_FILE1_H
……
#endif //LEARN_FILE1_H
```

这个技术等价于#pragma once

上面的代码就会报错，提示重复引入的头文件

不能重复引入头文件的原因是同一个头文件在编译之前会重复插入到最后的可执行文件里面，导致编译的时候出现各类重复定义的错误。而不是cpp编译会检测相同头文件的#include语句。通过#ifndef，保证了一个头文件编译的时候只会引入一次，从而防止了这个错误。但是即便使用上面这个技术，也不能在头文件里面直接申明函数的实现，只能申明函数的原型，也不能定义普通变量，这个可能是因为#ifndef对函数申明不生效，并且函数的原型可以重复定义，不会出问题，比如如下代码可以通过编译

```cpp
#include "file1.h"
#include "file1.h"
#include "file1.h"
#include "iostream"

using namespace std;

void read();
void read();
void read();

int main() {
    cout << "test start" << xxxx << endl;
    print({1});
}
```

但是变量和已经实现的方法重复定义后就会报错

但是结构体和类里面的函数可以直接实现，头文件可以使用static来定义全局变量，也可以使用inline来定义函数，这些都放在#ifndef和#endif里面，就可以解决上面这些问题

```cpp
//
// Created by 黎明终点x on 2022/10/16.
//
#ifndef LEARN_FILE1_H
#define LEARN_FILE1_H
#include <iostream>

struct A{
    int a;
    auto getA(){
        return a;
    }
};
static int xxxx;
inline void print(A a) {
    std::cout << a.getA();
}

#endif //LEARN_FILE1_H

```

出了入口cpp文件外，每个cpp源文件都要有一个同名的.h头文件来管理，头文件里面申明的函数原因，以及类方法的原型，只能在**同名**的cpp源文件里面实现，没有头文件的cpp文件是无法使用里面的方法的，头文件实际上是申明了这个源文件可以导出的，供其他文件使用的内容，如果没有头文件则可以理解为，其他文件不能使用这个cpp文件，也就没有了意义。没有头文件的那个cpp文件就是主文件，其他文件在编译的时候会先将链接起来，得到.o文件，然后再将各个.o文件连接起来编译，得到最后的可执行文件。

## muable

表示是可以修改的，用于解除const的限定

```cpp
#include "iostream"

using namespace std;

struct AB {
    int a;
    mutable int b;
};

int main() {
    const AB ab = {1, 2};
    ab.a = 1;//报错
    ab.b = 2;//不报错
}
```

加上mutable后，就表示这个成员变量是可以修改的，哪怕整体被设置成了const类型

## 内部连接性和外部连接性

### 变量

普通的成员变量定义后，外部文件可以通过extern来引用这个变量，这个变量全局只有一份，所有使用这个变量的文件共享同一块内存地址，这个特性叫外部连接性

想使用其他文件中定义的变量必须申明为extern 的来引用其他文件的这个变量，否则会报错重复定义

```
//file1.cpp文件中：
int xxx=5;
//main.cpp文件中
extern int xxx;
cout<<xxx<<endl; //得到5
```

而被申明为static的变量和被申明为const的变量，每个cpp文件都有一组，互不干扰，这个特性叫做内部连接性

比如下面这段代码，在file2.h中定义了一个`static int xxx;`显然这个变量初始值是0，main.cpp和file1.cpp都引入file2.h，这样在这两个cpp里面都能直接使用xxx变量，file1.cpp中的print函数修改了xxx，main.cpp直接输出，最后得到的结果是1 0，证明了这两个文件的static变量是相互独立，互不干扰的（底层实现可以理解为加上了文件名作为前缀）

```cpp
// file1.cpp
// Created by 黎明终点x on 2022/10/16.
//
#pragma once
#ifndef LEARN_FILE1_H
#define LEARN_FILE1_H
#include <iostream>

void print();

#endif //LEARN_FILE1_H

// file1.cpp
#include "file1.h"
#include "file2.h"

void print() {
    xxx++;
    std::cout << xxx << "\n";
}

//file2.h
//
// Created by 黎明终点x on 2022/10/16.
//
#ifndef LEARN_FILE2_H
#define LEARN_FILE2_H

#endif //LEARN_FILE2_H

static int xxx;

//main.cpp
#include "iostream"
#include "file2.h"
#include "file1.h"

using namespace std;

int main() {
    print();    //1
    cout << xxx;//0
}
```

如果其他文件申明了外部连接性的变量，自己又申明了同名的内部连接性变量，会使用内部连接性的变量

```
//file1.cpp文件中：
int xxx=5;
//main.cpp文件中
static int xxx;
cout<<xxx<<endl; //得到0
```

### 函数

总结一下，一个文件中想使用其他文件的变量，可以有以下两种方式

- 在头文件里面定义static变量或者const变量，cpp文件引入这个头文件就能使用这些变量，每个文件的作用域就是这个文件，文件之间互不干扰（内部连接性）
- 在cpp源文件定义变量，其他文件使用extern引入，这种变量全局只有一份，所有文件中的变量共享一块内存空间（外部连接性）

为什么其他cpp文件定义的变量我这个文件可以使用？

因为编译的时候这些文件都会先和头文件连接（引入头文件的变量可以使用），然后再彼此连接在一起（其他cpp文件定义的变量可以使用），编译成机器语言之前，这些变量都按照各自的规则进行转换，放到了同一个文件中，所以再理论上也是可以互联互通的

对于函数而言也有类似的特性，只是我们不允许在函数里面定义函数，所以函数定义出来都默认是外部连接特性的，于是我们引用其他文件的函数也有两种方式

- 使用前定义这个函数的原型，或者引入带有函数原型的头文件（函数原型可以重复定义，不用担心重复引入），这个函数的具体实现可以放在参与编译的任何cpp文件中，然后都可以正常使用这个函数（函数默认是外部连接性的，在函数原型前面加上extern或者不加都是可以的）

如果不希望这个函数被其他文件引用，可以使用static关键字申明，这样不同文件就可以定义同名函数，互不冲突，一个文件只能调用自己的static函数

## 创建struct变量的方式

使用大括号进行创建

```cpp
#include "iostream"
#include "file1.h"

using namespace std;

struct ABC {
    int a, b, c;
};

int main() {
    ABC abc1{1, 2, 3};
    ABC abc2{1, 2};
    ABC abc3{};
    ABC abc4=ABC();
    ABC *abc5 = new ABC{1, 2, 3};
    cout << abc4.a << endl;
}
```

使用大括号进行创建其实是按顺序指定结构体里面的各个成员变量的值，从前往后按定义的顺序依次赋值，后面没有被赋值的变量会被赋予零值

不使用大括号申明的变量（abc4），所有成员的值都是未定义状态

如果使用括号来创建变量实际上是创建了一个方法？反正不是创建变量！

想要调用构造方法来创建结构体变量，可以使用`ABC abc4=ABC();`这样的语法

也可以使用new运算符来创建，使用方式和上面一样，只是分配内存的位置由栈变成动态存储区

new运算符其实使用了一个语法糖的函数，使用typedef进行了简化

```
new int -> new(sizeof(int))
new int[40] -> new(40*sizeof(int))
```

new运算符还可以指定需要分配的内存地址，来自己进行内存管理，使用方法为

```
new(006E4AB0)int[20]
new(006E4AB0)int
new(内存地址)变量类型
```

## 命名空间

声明域：可以定义变量，函数的区域，局部变量的申明域是代码块，全局变量是申明域是函数的外面（申明一个引用外部文件的变量或者使用头文件里面的静态变量，也当成全局变量）

潜在作用域：从定义变量开始到声明域的结尾，这个范围内的变量可能会在某些区域内被其他同名变量覆盖（隐藏）

作用与：未被隐藏的潜在作用域

命名空间可以定义在全局或者其他命名空间里面，可以在里面定义任意多的变量，函数，类型等，命名空间之间不会相互干扰，定义后如果不使用命名空间，里面定义的内容就不会干扰我们定义其他变量，函数等（具体的实现可以理解名字为加上了命名空间的前缀）

全局内部的其他变量可以理解为在一个空命名空间里面，可以使用::来访问

`using namespace xxx;`后，会覆盖前面同名的全局（局部）变量，后面定义的全局（局部）变量也可以覆盖命名空间中引入的变量

放在全局表示命名空间中定义的内容全局可用，放在代码块里面表示这个代码块里面可用

而`using xxx::yyy`不会覆盖前面定义的同名内容，而是直接报错

命名空间可以嵌套，可以通过赋值起别名

未命名的命名空间不能被其他文件使用，可以实现类似static的功能

## 类的构造函数

创建类对象不能直接使用{}来创建，必须有对应的构造函数来能这么写

```cpp
#include "iostream"
#include "file1.h"

using namespace std;

class ABC {
public:
    ABC() {

    }

    ABC(int i, int i1) {

    }

    int a, b, c;

    ABC(int i, int i1, int i2) {

    }
};

int main() {
    ABC abc1{1, 2, 3};
    ABC abc2{1, 2};
    ABC abc3{};
    ABC abc4; //不一定是0值
    ABC *abc5 = new ABC{1, 2, 3};
    cout << abc4.a << endl;
}
```

也可以使用小括号来调用构造函数，两者完全等价，只是写法有所不同

```cpp
#include "iostream"
#include "file1.h"

using namespace std;

class ABC {
public:
    ABC() {

    }

    ABC(int i, int i1) {

    }

    int a, b, c;

    ABC(int i, int i1, int i2) {

    }
};

int main() {
    ABC abc1 = ABC(1, 2, 3);
    ABC abc2 = ABC(1, 2);
    ABC abc3=ABC();
    ABC abc4;
    ABC *abc5 = new ABC(1, 2, 3);
    cout << abc4.a << endl;
}
```

也就是说，创建同一个对象有下面这些写法

`ABC abc1 = ABC(1, 2, 3) <=>ABC abc1{1,2,3}<=>ABC abc1(1, 2, 3)<=> ABC abc1={1,2,3} `

像分配内存在公共存储区可以使用`ABC abc1 = new ABC(1, 2, 3) <=>ABC abc1 = new ABC{1, 2, 3} `

注意，不能使用`ABC abc1()`来调用无参构造函数，因为cpp会把它视为方法原型

一个对象申明为const表示这个对象的成员变量，不能被构造函数以外的函数修改，如果在函数后面加上const，表示这个函数不会修改成员变量。所以const对象不能调用非const方法，因为非const方法可能会修改成员变量，而非const对象可以调用任意方法

对于只有一个参数的构造函数，可以使用赋值号来调用构造函数创建对象，这种行为也叫做赋值构造

```cpp
#include "iostream"
#include "file1.h"

using namespace std;

class ABC {
private:
    int a;
public:
    ABC() {

    }

    ABC(int a) : a(a) {
        cout << "被调用1" << endl;
    }

    ABC(double a) {
        this->a = a;
        cout << "被调用2" << endl;
    }
};

int main() {
    ABC a = 1; //被调用1
    ABC b = 1.0; //被调用2
}
```

注意：只有一个参数的构造函数以及有多个参数但是其他参数有默认值的构造函数都可以使用赋值号调用构造函数创建对象，会根据赋值号右边的值来决定使用哪个构造函数

```cpp
    ABC(int a) : a(a) {
        cout << "被调用1" << endl;
    }
    ABC(int &a) : a(a) {
        cout << "被调用1" << endl;
    }
    ABC(int &&a) : a(a) {
        cout << "被调用1" << endl;
    }
```

对于上面三种构造函数，首先第一种写法不能和第二和第三种写法同时存在，否则调用构造函数时会有歧义（如果不调用的话，编译也不会出错）

## 对象数组

创建对象数组可以有两种方式：

```CPP
ABC abc[4];
ABC abc2[]={ABC(1)};
ABC abc3[]={ABC{1}};
```

- 使用类似`ABC abc[4];`的语法，调用的是类的无参构造函数（所以要保证有无参构造函数）
- 使用初始化列表的方式，其中每个元素都可以选择自己的构造函数来创建对象

## 类的作用域

```cpp
class ABC {
private:
    int a;
    const int b = 2;
    int xx[b]; //错误
}
```

在全局里面创建常量b，然后作为数组长度是可以的，但是对于类不行，因为类只是一个定义，在创建对象前不占用存储空间，所以数组不能将b替换成具体的数字，所以不能按照这种方式来定义

解决方式有两种

- 使用enum

```cpp
class ABC {
private:
    int a;
    enum {
        b = 5
    };
    int xx[b];
}
```

这么做后，编译器会在编译的时候，将b替换成5

- 使用static const

其实类的定义保存静态区里面，使用static后，这个成员对象就不保存在对象中，而是保存在静态区里面，因而编译器在创建对象之前，可以获知数组的长度，从而可以创建

## 类的继承

cpp允许基类的引用或者指针指向派生类，而调用方法时，使用的是基类方法还是派生类方法则有以下规则：

- 如果方法加上了virtual，则通过引用，指针，对象调用的方法都是对象类型的方法
- 如果没有加上virtual，则通过引用，指针调用的方法是引用，指针的类型，而通过对象调用的则是自己的方法

子类在调用父类方法时，要加上父类的类型::来限定使用哪个类的方法

析构函数必须是virtual的，因为析构函数是释放对象内容的一些行为，应该和实际的对象绑定在一起，而不是指针类型，对于同一种对象而言，任何时候都应该调用自己的析构函数

派生类如果定义和基类同名的方法，不会形成两个重载的函数，派生类的函数会隐藏基类的函数

返回类型协变：基类的返回值是基类或者基类的引用，派生类对于相同的方法返回改为了派生类或者派生类的引用，这种情况，基类方法不会被隐藏

### 三种继承方式

protect的成员变量可以被派生类访问，但是不能被外部访问

如果使用共有继承：`class Son : public Base {}`，基类的私有成员就还是私有，共有成员就还是共有

如果使用私有继承：`class Son : private Base`，基类的所有成员变量和方法都变成私有（默认就是私有）（使用private继承的派生类可以访问，因为是作为直接派生类的private成员，所以只有直接派生类可以访问，外部和间接子类都不能访问）

如果使用保护继承：`class Son : protected Base`，基类所有成员都作为保护成员（所有派生类可以访问，外部不能访问）

只有在共有继承的时候，基类指针才能指向子类

### 方法隐藏

```cpp
#include "iostream"
#include "vector"
#include "file1.h"

using namespace std;

class Base {
private:
    int x;
public:
    void test() {
        cout << "base test1" << endl;
    }

    void test(int x) {
        cout << "base test2" << endl;
    }
};

class Son : Base {
public:
    //    void test() {
    //        cout << "son test1" << endl;
    //    }
    //
//    void test(int x) {
//        cout << "son test2" << endl;
//    }

    void test(double x) {
        cout << "son test2" << endl;
    }

    void test2() {
        test();//被重载的test隐藏了，所以不能调用基类的test方法
    }
};


int main() {
    Son son;
    son.test2();
}
```

如果派生类中出现了和基类同名的方法，基类所有重载的同名方法都会被隐藏，派生类和后续的派生类，在使用同名的方法时，都无法直接使用基类的方法，想要使用可以这样：`Base::test()`，这样就可以指定访问哪个类的方法

外部成员可以这么访问`son.Base::test()`，前提是要有访问权限

### override和final

对于虚方法，我们可以加上override来表示重写了一个基类方法。对于非虚方法，是按照指针类型来调用，用的哪个方法很明显，不需要加上这个来提醒自己。一个类的方法加上virtual，表示这个作为指针的类型时，具体调用这个方式时，调用的是实际的类的方法

相反的，final可以声明一个方法不能被重写

```cpp
#include "iostream"
#include "vector"

using namespace std;

class Base {
private:
    int x;
public:
    virtual void test() = 0;

    virtual void test(int x) {
        cout << "base test2" << endl;
    }
};

class Son : public Base {
public:
    void test() {
        cout << "son test1" << endl;
    }

    void test(int x) {
        cout << "son test2" << endl;
    }

    //    void test(double x) {
    //        cout << "son test2" << endl;
    //    }
};


int main() {
    Base *base = new Son();
    base->test();
}
```



## Cpp函数调用

函数的返回值不放在栈中，一般会放在寄存器里面或者内存中的某块地址（但反正不是栈，如果是栈很多问题就无法解释）

函数执行完后的返回值是一个右值，这一行代码执行完就会被释放

## Cpp类的默认行为（移动构造，拷贝构造）

创建一个类A

```cpp
class A {
private:
    string name;
public:
    explicit A(const string &name) : name(name) {
        cout << "调用拷贝构造函数  " << name << endl;
    }

    explicit A(const string &&name){
        cout << "调用移动构造函数  " << name << endl;
    }

    A() {
        cout << "调用默认构造函数" << endl;
    }

    A &operator=(const A &abc) {
        cout << "调用拷贝构造函数" << name << endl;
        name = abc.name;
        return *this;
    }

    virtual ~A() {
        cout << "调用析构函数  " << name << endl;
    }

    void setName(const string &name) {
        A::name = name;
    }
};
```

我们想从一个方法中拿到一个对象

下面这个做法是错误的，因为变量a在函数结束后会退栈销毁掉，而引用本质上保存的是变量是地址，变量不存在了，地址也就没有意义，所以下面返回的是未定义的值

```cpp
A& getA(){
    A a;
    a.setName("初始参数");
    return a;
}
```

所以应该改成下面这种写法

```cpp
A getA() {
    A a;
    a.setName("初始参数");
    return a;
}
```

这么改后，因为返回值会单独占据内存中的一块区域，函数返回的时候，会先将a变量拷贝到那块临时区域，然后将地址返回到原来的函数，这样函数结束后，a变量会销毁，但是拷贝还在，所以外面函数拿到的其实是这个拷贝的对象。

```cpp
int main() {
    getA();
}
```

但是这样得到的对象也只是一个临时值，这一行代码结束后，这个临时的对象也就不再存在，这个临时的变量也是我们所说的右值。如果返回值不被使用也就不会被拷贝。

```cpp
#include "iostream"
#include "vector"
#include "file1.h"

using namespace std;

class A {
private:
    string name;
public:
    A(const A &&name) {
        cout << "调用移动构造函数  " << name.name << endl;
    }

    A() {
        cout << "调用默认构造函数" << endl;
    }

    A(const A &a) : name(a.name) {
        cout << "调用拷贝构造函数  " << name << endl;
    }


    A &operator=(const A &abc) {
        cout << "调用复制运算符" << name << endl;
        name = abc.name;
        return *this;
    }

    virtual ~A() {
        cout << "调用析构函数  " << name << endl;
    }

    void setName(const string &name) {
        A::name = name;
    }

    const string &getName() const {
        return name;
    }
};

//A &getA() {
//    A *a = new A();
//    a->setName("初始参数");
//    return *a;
//}

A getA() {
    A a;
    a.setName("xxx");
    return a;
}

A useA(const A &a) {
    cout << "使用A:" + a.getName() << endl;
    return a;
}

int main() {
    cout << "main start" << endl;
    useA(getA());
    cout << "main end" << endl;
}
```

如果返回的对象在函数结束后会销毁，就不拷贝，返回这个要销毁的对象
如果函数结束后不销毁，则调用拷贝构造创建一个对象返回

这个其实是编译器帮我们做的优化，这样我们就不用编写移动构造

（const A&不能作为A&返回，变量当成常量没有风险，反过来就不一定了）

## constexpr 申明常量表达式

申明变量的时候可以加上constexpr，表示计算这个变量只需要常量，这样这个值就可以在编译期计算出来

## 使用CLION创建项目的时候，不要有中文路径

## CMAKE语法

[cmake常用命令的一些整理 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/315768216)

## 高斯白噪声

```cpp
#include <iostream>
#include <iterator>
#include <random>

int main() {
    // Example data
    std::vector<double> data = {1., 2., 3., 4., 5., 6.};

    // Define random generator with Gaussian distribution
    const double mean = 0.0;//均值
    const double stddev = 0.1;//标准差
    std::default_random_engine generator;
    std::normal_distribution<double> dist(mean, stddev);

    // Add Gaussian noise
    for (auto& x : data) {
        x = x + dist(generator);
    }

    // Output the result, for demonstration purposes
    std::copy(begin(data), end(data), std::ostream_iterator<double>(std::cout, " "));
    std::cout << "\n";

    return 0;
}
```

## emplace_back和push_back的区别

emplace_back放入的是移动构造后得到的对象，接收右值，调用移动构造函数

push_back放入的是拷贝构造得到的对象，接收左值，调用拷贝构造函数

## deque支持随机访问

deque底层是`map<int,vector>`，使用这样的一个数据结果实现双端队列

[C++ STL deque容器底层实现原理（深度剖析） (biancheng.net)](http://c.biancheng.net/view/6908.html)

![image-20221025020644091](pictures/image-20221025020644091-1669803907655-13-1669804077621-23.png)

deque支持随机访问，但是不能随机插入，只能添加或者删除队头和队尾的元素，这些操作的复杂度都是O(1)，相当于将一块连续的数组切分成了多个长度固定的数组，start迭代器记录第一个元素的起始位置，第一个数组的起始地址，第一个数组的中止地址，第一个map指针的地址，finish迭代器，记录最后一个元素的位置，最后一个数组的起始地址，终止地址，存放这个地址的map结点

这样我们就可以知道第一个元素是啥，最后一个元素是啥了，删除后，迭代器里面存放的指针就往中间移动一个，添加元素后就在添加完成后，指针往外面移动一格，所以这些操作都是O(1) 的。如果第一个数组满了，就在前面再申请一个数组，继续存放元素，更新start迭代器的指向，在后面添加也是一样的，都是O(1)的复杂度。

对于随机访问，由于每个数组的长度都是固定的，很容易根据下标，确定要访问的元素在哪个数组的哪个位置，从而实现O(1)随机访问

## 智能指针

为什么要有智能指针？

使用普通的指针，一方面我们可能会忘记delete掉申请的内存，另一方面，如果使用指针的过程中出现异常，可能会导致delete的那条代码没有执行，导致内存泄露，所以可以使用智能指针来实现，指针指针是实现了指针功能的类对象，如果函数出现异常会调用对象的析构函数释放内存，把地址值赋值给指针指针，就不需要我们来释放内存，由编译器调用析构函数自己完成。智能指针在`<memory>`里面。智能指针不能用于非堆内存，只能传入`new`出来的内存块的地址。如果传入非堆内存，就会delete非堆地址，就会报错。

智能指针可以和其他指针一起正常使用，那么就可以像指针一样赋值给其他智能指针对象，这样的话就会有问题。如果智能指针只是单纯在析构函数里面delete掉内存的话，就会出现同一块内存被重复delete的问题，解决这个问题的方案有三种

- 赋值的时候进行深拷贝
- 不允许赋值 `unique_ptr`
- 赋值的时候，转移所有权 `auto_ptr`
- 采用引用计数 `share_ptr`

`unique_ptr`禁止了拷贝赋值，但是允许了移动赋值（可以接收函数的返回值）

`auto_ptr`是在拷贝赋值中，转移对象的所属权，原来的智能指针对象会变成“悬挂的指针”，里面原生的指针会变成空指针，从而不能解引用来获取值

`unique_ptr`在编译期禁止了赋值这种危险的行为，所以比`auto_ptr`更加安全。如果我们确实需要进行`unique_ptr`的赋值操作，转移内容的所属权，可以使用move()将左值变成右值即可

`share_ptr`则使用引用计数来共享内容的所属权，使用起来更加方便

`auto_ptr`被废弃

使用智能指针unique

```cpp
#include <memory>
#include "iostream"

using namespace std;

#define WEBRTC_POSIX

int main() {
    unique_ptr<string> sp(new string("123"));
    unique_ptr<string> sp2{new string("234")};
    cout << *sp << " " << *sp2 << endl; //123 234
    //    sp = sp2; 被禁止，编译报错
    sp = move(sp2); //允许，但是sp2就没有用了
    cout << *sp << endl; //234
}
```

`share_ptr`在进行赋值前，如果原来已经有指向的对象，会将原有指针指向的内存释放掉

![image-20221028105501647](pictures/image-20221028105501647-1669803907655-12.png)

```cpp
#include <memory>
#include "iostream"

using namespace std;

#define WEBRTC_POSIX

int main() {
    shared_ptr<string> sp(new string("123"));
    shared_ptr<string> sp2{new string("234")};
    cout << *sp << " " << *sp2 << endl; //123 234
    sp = sp2;
    cout << *sp << " " << *sp2 << endl;
    sp2 = shared_ptr<string>(new string("345"));
    cout << *sp << " " << *sp2 << endl;
}
```

### 手写unique_ptr

```cpp
#include "iostream"

using namespace std;

template<typename T>
class unique_ptr {
    T *ptr;
public:
    //普通构造
    unique_ptr(T *ptr) : ptr(ptr) {
    }
	//移动构造
    unique_ptr(unique_ptr &&raw) {
        ptr = raw.ptr;
        raw.ptr = nullptr;
    }
	//默认构造
    unique_ptr() = default;
	//禁用拷贝赋值
    unique_ptr &operator=(const unique_ptr &) noexcept = delete;
	//允许移动赋值
    unique_ptr &operator=(unique_ptr<T> &&raw) noexcept {
        swap(ptr, raw.ptr);
        return *this;
    }
	//取值
    T &operator*() const noexcept {
        return *ptr;
    }
	//析构函数释放内存
    virtual ~unique_ptr() {
        delete ptr;
    }
};

int main() {
    unique_ptr<string> sp(new string("123"));
    unique_ptr<string> sp2{new string("234")};
    cout << *sp << " " << *sp2 << endl; //123 234
    //    sp = sp2;// 被禁止，编译报错
    sp = move(sp2); //允许，但是sp2就没有用了
    cout << *sp << endl; //234
}
```

移动赋值为啥swap

```cpp
	//允许移动赋值
    unique_ptr &operator=(unique_ptr<T> &&raw) noexcept {
        swap(ptr, raw.ptr);
        return *this;
    }
```

移动赋值需要三步：

- 将原来的内存释放掉
- 将右值引用的ptr赋值给自己的ptr
- 将右值引用的ptr置为nullptr防止重复delete

这样显然有些麻烦，可以使用swap一步完成

右值引用的声明周期只有那一行代码，那一行结束后，就会调用右值对象的析构函数释放内存

互换指针，一方面自己的ptr得到对方ptr的值，完成了第一步。右值对象中ptr的指向变成了自己的原来的指向，完成了第三步，防止重复释放内存，同时右值对象在这一行结束后会销毁，会调用析构函数释放内存，完成了第二步。所以这一个swap就完成了上述的三个操作

## 手写share_ptr

```cpp
#include <unordered_map>
#include "iostream"

using namespace std;


unordered_map<void *, unsigned> share_count_map;

template<typename T>
class shared_ptr {
private:
    T *ptr;

public:
    shared_ptr() = default;

    explicit shared_ptr(T *ptr) : ptr(ptr) {
        if (ptr != nullptr) {
            share_count_map[ptr] = 1;
        }
    }

    shared_ptr(const shared_ptr<T> &raw) noexcept {
        ptr = raw.ptr;

    }

    shared_ptr(const shared_ptr &&raw) noexcept {
        ptr = raw.ptr;
        share_count_map[ptr]++;
    }

    shared_ptr<T> &operator=(const shared_ptr<T> &raw) noexcept {
        if (raw.ptr == ptr) {
            return *this;
        }
        share_count_map[ptr]--;
        if (share_count_map[ptr] == 0) {
            share_count_map.erase(ptr);
            delete ptr;
        }
        ptr = raw.ptr;
        share_count_map[ptr]++;
        return *this;
    }

    shared_ptr<T> &operator=(shared_ptr<T> &&raw) noexcept {
        swap(ptr, raw.ptr);
        return *this;
    }

    T &operator*() const noexcept {
        return *ptr;
    }

    virtual ~shared_ptr() {
        share_count_map[ptr]--;
        if (share_count_map[ptr] == 0) {
            share_count_map.erase(ptr);
            delete ptr;
        }
    }
};


int main() {
    shared_ptr<string> p;
    shared_ptr<string> p2 = p;
    shared_ptr<string> sp(new string("123"));
    shared_ptr<string> sp2{new string("234")};
    cout << *sp << " " << *sp2 << endl; //123 234
    sp = sp2;
    cout << *sp << " " << *sp2 << endl;
    sp2 = shared_ptr<string>(new string("345"));
    cout << *sp << " " << *sp2 << endl;
}
```

share_ptr需要用到引用计数，但是这个引用计数表示的是一个类型的数据被多少个智能指针管理这，所以这个引用计数器应当是一个全局变量，独立于每个对象，如果放在对象里面，每个对象都有一个自己的副本，显然不行。引用计数器是所有`share_ptr`的管理者，所以应当定义在全局。

```cpp
unordered_map<void *, unsigned> share_count_map;
```

又因为要支持泛型，所以这里采用数据的内存地址作为map的key

## `string::nops`其实是`usigned long long`的最大值，find没有找到就返回这个值

```cpp
#include "iostream"

using namespace std;

#define WEBRTC_POSIX

int main() {
    cout << string::npos << endl;
    string s = "123";
    cout << (s.find("456")) << endl;
    cout << UINT64_MAX << endl;
}
```

## cpp Default的用法

```cpp
#include "iostream"

using namespace std;

class Base {
public:
    Base &operator=(const Base &b) noexcept {
        cout << "赋值运算符" << endl;
        return *this;
    }
};

class Son : public Base {
public:
    Son &operator=(const Son &) noexcept = default;
};

int main() {
    Son son;
    Son son2;
    son = son2; //输出 赋值运算符
}
```

default表示使用编译器默认生成的函数，如果父类有自定义的，就使用父类的。仅限于特殊的函数（构造函数，析构函数，赋值运算符，拷贝构造，移动构造，移动赋值）

const 和 noexcept一起写的时候，const要放在前面

## swap的原理

```cpp
_Tp __tmp = _GLIBCXX_MOVE(__a);
__a = _GLIBCXX_MOVE(__b);
__b = _GLIBCXX_MOVE(__tmp);
```

和我们平时写的逻辑类似，只是用了move，move会得到右值引用

```cpp
      _Tp __tmp = _GLIBCXX_MOVE(__a); //将__a的右值引用赋值给__tmp，这里会调用移动构造
      __a = _GLIBCXX_MOVE(__b);  //将__b的右值引用赋值给__a，这里会调用移动赋值
      __b = _GLIBCXX_MOVE(__tmp); //将__tmp的右值引用赋值给__b，这里会调用移动赋值
```

使用这种方式就可以避免调用拷贝构造和拷贝赋值，提高效率

## sscanf

sscanf用于从一个字符串中格式化读入参数

例如下面的dtm就是被读入的字符串，后面是格式化参数，再后面是接受这些变量的值

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "iostream"

int main() {
    int day, year;
    char weekday[20], month[20], dtm[100];
	
    strcpy(dtm, "Saturday March 25 1989");
    sscanf(dtm, "%s %s %d  %d", weekday, month, &day, &year);

    printf("%s %d, %d = %s\n", month, day, year, weekday);
    std::cout << dtm << std::endl;

    return (0);
}
```

## Cpp 中的信号（Signal和Raise）

[(884条消息) C++ Signal(信号)_肥喵王得福_ฅ・ω・ฅ的博客-CSDN博客_c++ signal](https://blog.csdn.net/u013271656/article/details/114537411)

raise函数用来触发信号，signal用于监听信号并在收到信号的时候进行软中断，执行设置的处理函数

```
void (*signal(int sig, void (*func)(int)))(int);
```

第二个参数是一个函数，可以传入我们自定义的函数，也可以填一些系统默认值，比如`SIG_DFL`表示进行默认的行为，`SIG_IGN`表示忽略

关于触发信号，STD中的`std::abord,std::atexit,std::terminate`等函数都可以触发信号，等价于`std::signal(对应的信号)`

## Cpp 函数指针

```cpp
#include <csignal>
#include <iostream>
#include <cstdlib>

using namespace std;

typedef void (*Func)(int);

void test(Func f) {
    f(1);
}

void test(int x) {
    cout << x << endl;
}

int main() {
    test(test);
}
```

函数指针定义方式和一般变量不同，所以可以使用typedef来将其变成类型的形式

```cpp
//     返回值  类型名 参数类型        
typedef void (*Func)(int);
```

也可以直接使用

```cpp
#include <csignal>
#include <iostream>
#include <cstdlib>

using namespace std;

typedef void (*Func)(int);

void test(void (*f)(int)) {
    f(1);
}

void test(int x) {
    cout << x << endl;
}

int main() {
    void (*f)(int);
    f(1); //会卡死，因为函数指针没有指向
    test(test);
}
```

