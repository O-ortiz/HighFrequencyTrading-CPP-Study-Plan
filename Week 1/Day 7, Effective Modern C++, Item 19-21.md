### **Item 19: Use std::shared_ptr for shared-ownership resource management.**

All objects that std::shared_ptr are shared by the pointers. Once all std::shared_ptr pointing to an object are destroyed, so too is the object it points too. 

Each std::shared_ptr points to the object and to a control block that contains the reference count, "weak count", and other data like custom deleter and allocator if those were specified. 

So there are a few implications:
- Std::shared_ptr is twice the size of a raw poiter
- Memory for the reference counts must be dynamically allocated. 
- Increments and decrements to the reference count must be atomic (for multithreaded purposes). 

There are times where the reference count is not increased, like for move constructors. 

When defining custom deleters, the deleter lamba function is not part of the std::shared_ptr type, like it is for std::unique_ptr:

```
auto loggingDel = [](Widget *pw) // custom deleter
 { // (as in Item 18)
	 makeLogEntry(pw);
	 delete pw;
 };
std::unique_ptr< // deleter type is
 Widget, decltype(loggingDel) // part of ptr type
 > upw(new Widget, loggingDel);

std::shared_ptr<Widget> // deleter type is not
 spw(new Widget, loggingDel); // part of ptr type

```

This allows std::shared_ptr to be put into containers like an std::vector without issue. 

Every time an std::shared_ptr is created, so too is a control block (when necesary), following these rule:
1. std::make_shared (see item 21) always creates a control block. 
2. A control block is created when std::shared_ptr is constructed from a unique-ownership ptr (ie std::unique_pre or std::auto_ptr)
3. When a std::shared_ptr constructor is called with a raw pointer, it creates a control bloc. 

There are a number of ways to cause undefined behavior when we create shared pointers, especially when doing so from raw pointers. This is because the pointed to object will have multiple control blocks, which means that at the end of the lifespan of the std::shared_ptr the object will be deleted multiple times. Which is very very bad behavior. 

```
auto pw = new Widget; // pw is raw ptr
…
std::shared_ptr<Widget> spw1(pw, loggingDel); // create control
 // block for *pw
…
std::shared_ptr<Widget> spw2(pw, loggingDel); // create 2nd
 // control block
// for *pw!

```

You want to have initially preferred smart pointers to begin with. Avoid passing raw pointers to std::shared_ptr. Alrernative is to use make_shared(), but that can't be done with a custom deleter. If you have to pass a raw pointer, do it directly from new without allocating it to a raw ptr:
```
Widget> spw1(new Widget, // direct use of new loggingDel);
```

Another way that this issue can be used it by returning "this" which is a raw pointer for the object. To return a std::unique_ptr type, you must inherit from std::enable_shared_from_this<> which is itself a templated type, that must be templated from the inheritor. 

This sounds weird and paradoxical but is apparently legal and part of an established design pattern called *The Curiously Recurring Template Pattern (CRTP).  

So you would declared Widgets for example as: 
```
class Widget: public std::enable_shared_from_this<Widget> {
public:
 …
 void process();
 …
};
```
And this way you can use shared_from_this() as a member function extended from std::enable_shared_from_this<> to return an std::shared_ptr. 

```
void Widget::process()
{
 // as before, process the Widget
 …
 // add std::shared_ptr to current object to processedWidgets
 processedWidgets.emplace_back(shared_from_this());
}
```

This would cause issues is called BEFORE there is any existence of an std::shared_ptr of that class, so classes inheriting from std::enable_shared_from_this often declare their constructors private and have clients create objects by calling factory functions that return std::shared_ptrs. 

```
class Widget: public std::enable_shared_from_this<Widget> {
public:
 // factory function that perfect-forwards args
 // to a private ctor
 template<typename... Ts>
 static std::shared_ptr<Widget> create(Ts&&... params);
 …
 void process(); // as before
 …
private:
 … // ctors
};

```

Std::shared_ptr is a reasonable cost for the functionality it provides. Under typical conditions:
-default deleter, default allocator, created by make_shared, the control block is only three words, and allocation is essentially free. 
-derefencing a shared_ptr is no more expensive than a raw ptr
-Operations that require changing reference count is done through one or two atomic operations, but are typically just single instructions even if a bit more expensive compared to non-atomic instructions

In exchange though, we get automatic lifetime management of dynamically allocated resources. 

Once you turn over a resouce to std::shared_ptr you cannot undo it or declare that object exclusive again. 

Std::shared_ptr do not handle dumb array but you don't need to when you already have better options in the standard library. 

Things to remember: 
- std::shared_ptr offers convenience approaching that of garbage collection for the lifetime management of arbitrary resources. 
- Compared to std::unique_ptr, std::shared_ptr objects are typically twice as big, incurring overhead for control blocks, and require atomic reference manipulations
- Default resource destructors via delete, but custom deleters are supported. The type of the deleter has no effect on the type of the std::shared_ptr. 
- Avoid creating std::shared_ptr from variables of raw pointer type. 

### **Item 20: Use std::weak_ptr for std::shared_ptr-like pointers that can dangle.**

There are very few use cases where this will be useful, but it's important to understand it for when it is useful. 

std::weak_ptr is typically created from std::shared_ptrs. They point to the same place as std::shared_ptr and can tell if that original object has been deleted. 

If std::weak_ptr dangles, then it is said to be expired. 

You can check if it is inspired using using expired() which returns a bool, or you can use .lock() which returns an std::shared_ptr object if it isn't expired, or null if it is. 

```
ared_ptr<Widget> spw1 = wpw.lock(); // if wpw's expired,
 // spw1 is null
auto spw2 = wpw.lock(); // same as above,
 // but uses auto
```

std::shared_ptr can also take a weak_ptr as a constructor argument, but will throw an exception if the std::weak_ptr is expired. 

```
std::shared_ptr<Widget> spw3(wpw); // if wpw's expired,
 // throw std::bad_weak_ptr
```

Things to Remember: 
- Use std::weak_ptr for std::shared_ptr-like pointers that can dangle. 
- Potential use cases for std::weak_ptr include caching, observer lists, and the prevention of std::shared_ptr cycles. 

### **Item 21: Prefer std::make_unique and std::make_shared to direct use of new.**

