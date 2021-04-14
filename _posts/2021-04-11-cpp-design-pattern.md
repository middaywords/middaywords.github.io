---
title: Cpp design pattern
date: 2021-04-11
permalink: /posts/2021/04/designpattern/
tags:
  - cpp
typora-root-url: ../../middaywords.github.io
---

### Design Pattern

---

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
  	// 利用可变参数，消除重载
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



---

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

// 不需要多定义一个基类来抽象Observer，易于绑定
// NonCopyable使得其不可复制
// 其实一个Events类只能对应一种函数
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



---

#### visitor

visitor.hpp

```cpp
#include <iostream>
#include <memory>
using namespace std;

struct ConcreteElement1;
struct ConcreteElement2;

struct Visitor
{
    virtual ~Visitor() {}
    virtual void Visit(ConcreteElement1 *element) = 0;
    virtual void Visit(ConcreteElement2 *element) = 0;
};

struct Element
{
    virtual ~Element() {}
    virtual void Accept(Visitor &visitor) = 0;
};

struct ConcreteVisitor : public Visitor
{
    void Visit(ConcreteElement1 *elemet)
    {
        cout << "Visit ConcreteElemen1" << endl;
    }
    void Visit(ConcreteElement2 *elemet)
    {
        cout << "Visit ConcreteElemen2" << endl;
    }
};

struct ConcreteElement1 : public Element
{
    void Accept(Visitor &visitor)
    {
        visitor.Visit(this);
    }
};

struct ConcreteElement2 : public Element
{
    void Accept(Visitor &visitor)
    {
        visitor.Visit(this);
    }
};
```

Test.cpp

```cpp
#include "visitor.hpp"

void TestVisitor()
{
    ConcreteVisitor v;
    std::unique_ptr<Element> emt1(new ConcreteElement1());
    std::unique_ptr<Element> emt2(new ConcreteElement2());

    emt1->Accept(v);
    emt2->Accept(v);
}
```

这样实现的缺点是当需要加入新访问者的时候，需要在基类中再加入一个纯虚函数。



---

#### object pool

object_pool.hpp

```cpp
#include <string>
#include <functional>
#include <memory>
#include <map>

using namespace std;

const int MaxObjectNum = 10;

class NonCopyable
{
protected:
    NonCopyable() = default;
    ~NonCopyable() = default;
    NonCopyable(const NonCopyable &) = delete;
    NonCopyable &operator=(const NonCopyable &) = delete;
};

template <typename T>
class ObjectPool : NonCopyable
{
    template <typename... Args>
    using Constructor = std::function<std::shared_ptr<T>(Args...)>;

private:
    std::multimap<string, std::shared_ptr<T>> _objectMap;

public:
    //! \param num: Create `num` objects
    template <typename... Args>
    void Init(size_t num, Args &&...args);

    template <typename... Args>
    shared_ptr<T> GetObject();
};

template <typename T>
template <typename... Args>
void ObjectPool<T>::Init(size_t num, Args &&...args)
{
    if (num <= 0 || num > MaxObjectNum)
    {
        throw new std::logic_error("object num is out of range");
    }
    // 对于不同构造函数的对象池化
    auto constructName = typeid(Constructor<Args...>).name();
    for (size_t i = 0; i < num; i++)
    {
        _objectMap.emplace(constructName, shared_ptr<T>(
                                              new T(std::forward<Args>(args)...),
                                              [this, constructName](T *p) {
                                                  // 定义删除时，不直接删除对向，而是回收到对象池，以供下次使用
                                                  _objectMap.emplace(std::move(constructName), std::shared_ptr<T>(p));
                                              }));
    }
}

template <typename T>
template <typename... Args>
std::shared_ptr<T> ObjectPool<T>::GetObject()
{
    string constructName = typeid(Constructor<Args...>).name();

    auto range = _objectMap.equal_range(constructName);
    for (auto it = range.first; it != range.second; ++it)
    {
        auto ptr = it->second;
        _objectMap.erase(it);
        return ptr;
    }

    return nullptr;
}

```

Test.cpp

```cpp

void TestObjectPool() {
    ObjectPool<BigObject> pool;
    pool.Init(2);
    {
        auto p = pool.GetObject();
        Print(p, "p");
        auto p2 = pool.GetObject();
        Print(p2, "p2");
    }
    // 回收后可重新获取
    auto p = pool.GetObject();
    auto p2 = pool.GetObject();
    Print(p, "p");
    Print(p2, "p2");

    // 支持重载函数构造的对象
    pool.Init(2, 1);
    auto p4 = pool.GetObject<int>();
    Print(p4, "p4");
    pool.Init(2, 1, 2);
    auto p5 = pool.GetObject<int, int>();
    Print(p5, "p5");
}
```

