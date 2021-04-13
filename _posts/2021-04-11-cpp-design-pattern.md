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



#### Observer

observer.hpp

```cpp
#include <map>
#include <memory>

class NonCopyable
{
protected:
    NonCopyable() = default;
    ~NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
};

template<typename Func>
class Events : NonCopyable
{
private:
    int _observerId = 0;
    std::map<int, Func> _connections;

    template<typename F>
    int Assign(F&& f) {
        int k = _observerId++;
        _connections.emplace(k, std::forward<F>(f));

        return k;
    }

public:
    Events() {}
    ~Events() {}
    
    // Register observer
    int Connect(Func&& f) {
        return Assign(f);
    }

    int Connect(const Func& f) {
        return Assign(f);
    }
    
    void Disconnect(int key) {
        _connections.erase(key);
    }

    template<typename... Args>
    void Notify(Args&... args) {
        for (auto& it : _connections) {
            it.second(std::forward<Args>(args)...);
        }
    }
};
```

test.cpp 

```cpp

struct stA
{
    int a, b;
    void print(int a, int b) { cout << a << ", " << b << endl; }
};

void print(int a, int b) { cout << a << ", " << b << endl; }

void test2() {
    Events<std::function<void(int, int)>> myEvent;

    auto key = myEvent.Connect(print);
    stA t;
    auto lambdaKey = myEvent.Connect(
        [&t](int a, int b){ 
            t.a = a; 
            t.b = b; 
        }
    );
  
  	// 其实一般不太想用bind，直接用个lambda对它进行封装就好了
    std::function<void(int, int)> f = std::bind(&stA::print, &t, std::placeholders::_1, std::placeholders::_2);
    auto val3 = myEvent.Connect(f);
    int a = 1, b= 2;

    myEvent.Notify(a, b);

    myEvent.Disconnect(key);
    cout << endl;

    myEvent.Notify(a, b);
}
```

