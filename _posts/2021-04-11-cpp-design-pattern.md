---
title: Cpp design pattern
date: 2021-04-11
permalink: /posts/2021/04/designpattern/
tags:
  - cpp
typora-root-url: ../../middaywords.github.io
---

### Design Pattern

#### singleton 

singleton.hpp

```cpp
#include <memory>

template <typename T>
class Singleton
{
private:
    static T *_instance;

    // 设为私有，不允许复制和赋值
    Singleton(void) = delete;
    // 析构函数为虚函数是为了防止，在析构时只析构基类而不析构派生类的状况发生。
    virtual ~Singleton(void);
    Singleton(const Singleton &);
    Singleton &operator=(const Singleton &);

public:
    template <typename... Args>
    static T* Instance(Args &&...args);

    static T* GetInstance();

    static void DestroyInstance();
};

# static成员初始化不需要加static修饰
template<class T>
T* Singleton<T>::_instance = nullptr;

# 这些内容不能另存到singleton.cpp中，否则会报错 undefined reference
# refer to https://itachi666.github.io/posts/bfee53d1/
template <typename T>
template <typename... Args>
T* Singleton<T>::Instance(Args &&...args)
{
    if (_instance == nullptr)
    {
        _instance = new T(std::forward<Args>(args)...);
    }

    return _instance;
}

template <typename T>
T* Singleton<T>::GetInstance()
{
    if (_instance == nullptr)
    {
        throw std::logic_error("The instance is not init.");
    }

    return _instance;
}

template<typename T>
void Singleton<T>::DestroyInstance()
{
    delete _instance;
    _instance = nullptr;
}

```

test.cpp 测试代码

```cpp
#include <iostream>
#include "singleton.hpp"

using namespace std;

struct A
{
    A() {
        cout << "A created\n";
    }
};

struct B
{
    B(int x) {
        cout << "B created\n";
    }
};

struct C
{
    C(int x, int y) {
        cout << "C created\n";
    }
};

void test1()
{
    Singleton<A>::Instance();
    Singleton<B>::Instance(2);
    Singleton<C>::Instance(3, 4);

    Singleton<A>::DestroyInstance();
    Singleton<B>::DestroyInstance();
    Singleton<C>::DestroyInstance();
}
```



