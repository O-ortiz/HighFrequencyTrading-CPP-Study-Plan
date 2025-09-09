### **Item 16: Make const member functions thread safe.**

If for example, we have a polynomial class with an expensive function that calculates its roots, we can do it once and keep the results in a cache. Also the roots function is const. 

Declaration (.h):
```
class Polynomial {
public:
	 using RootsType = // data structure holding values
	 std::vector<double>; // where polynomial evals to zero
	 … // (see Item 9 for info on "using")
	 RootsType roots() const;
	 …
};

```
Implementation (.cpp)
```
class Polynomial {
public:
 using RootsType = std::vector<double>;
 RootsType roots() const
 {
	 if (!rootsAreValid) { // if cache not valid
	 … // compute roots,
	 // store them in rootVals
	 rootsAreValid = true;
 }
 return rootVals;
 }
private:
 mutable bool rootsAreValid{ false }; // see Item 7 for info
 mutable RootsType rootVals{}; // on initializers
};
```

(Conceptually) Roots doesn't change the polynomial object, but does need to update rootsAreValid and rootVals, which is why those are specifically indicated as mutable. 

This could cause problems when multiple threads try to run roots and one hasn't updated the cache yet and you spend unnecessary resources calculating the roots multiple times. 

For this, we can employ a mutex: 
```
class Polynomial {
public:
 using RootsType = std::vector<double>;
 RootsType roots() const
 {
	 std::lock_guard<std::mutex> g(m); // lock mutex
	 if (!rootsAreValid) { // if cache not valid
	 … // compute/store roots
	 rootsAreValid = true;
 }
 return rootVals;
 } // unlock mutex
private:
	 mutable std::mutex m;
	 mutable bool rootsAreValid{ false };
	 mutable RootsType rootVals{};
};
```
Mutex is a move-only type so will make Polynomial move only as well. 

We might also want to use std::atomic instead. Disalows other threads to see what its operating on while intermediary steps, ie only lets other threads see the previous or after of its process. 

You can use it to count calls:

class Point { // 2D point
public:
 …
 double distanceFromOrigin() const noexcept // see Item 14
 { // for noexcept
 ++callCount; // atomic increment
 return std::sqrt((x * x) + (y * y));
 }
private:
 mutable std::atomic<unsigned> callCount{ 0 };
 double x, y;
};

You might want to always rely on std::atomic, but this could cause much more work. Rule of thumb, if you need a single variable or memory location requiring synchronization, then an atomic will be best, but once you need two or more variable or memory locations that require manipulation, mutex is better. 

And this is how we would do it for Widget::magicValue:

```
class Widget {
public:
 …
 int magicValue() const
 {
	 std::lock_guard<std::mutex> guard(m); // lock m
	 if (cacheValid) return cachedValue;
	 else {
	 auto val1 = expensiveComputation1();
	 auto val2 = expensiveComputation2();
	 cachedValue = val1 + val2;
	 cacheValid = true;
	 return cachedValue;
 }
 } // unlock m
 …
private:
	 mutable std::mutex m;
	 mutable int cachedValue; // no longer atomic
	 mutable bool cacheValid{ false }; // no longer atomic
};
```
This specific Item is for const functions that are likely to be used in multiple threads. If you can gaurantee that the const funtion will NOT be used in multiple threads, then did does not apply. However, threading-free scenarios are increasingly uncommon. 

Things to remember:
- Make const functions thread safe unless you're absolutely certain that it will now be used in a concurrent context. 
- Use of std::atomic variables may offer better performance than a mutex, but they're suited for manipulation of only a single concurrent context. 

### **Item 17: Understand special member function generation.**

Special member functions - functions that the constructor automatically generates for you on startup - like default constructors, destructors, etc.

Rule of Three: if you declare any of a copy constructor, copy assignment operator, or destructor, you should declare all three.

Move operations are generated for classes (when needed) only when there three things are met:
1. No copy operations are declared in the class. 
2. No move operations are declared in the class. 
3. No destructor is declared in the class. 

If you want both move and copy operations, and they default versions you can do:

```
class Base {
public:
	 virtual ~Base() = default; // make dtor virtual
	 Base(Base&&) = default; // support moving
	 Base& operator=(Base&&) = default;
	 Base(const Base&) = default; // support copying
	 Base& operator=(const Base&) = default;
	 …
};
```
It may be good to adopt a policy of declaring them ourselves using "=default". It makes intention cleaner and we can avoid subtle bugs. 

| Special Member Function              | Rules in C++11                                                                                                                                            |
|--------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Default constructor**              | Same as C++98. Generated only if the class contains no user-declared constructors.                                                                          |
| **Destructor**                       | Same as C++98, but destructors are `noexcept` by default. Virtual only if a base class destructor is virtual.                                               |
| **Copy constructor**                 | Same runtime behavior as C++98 (memberwise copy). Generated only if the class lacks a user-declared copy constructor. Deleted if the class declares a move operation. Generation is deprecated if the class has a user-declared copy assignment operator or destructor. |
| **Copy assignment operator**         | Same runtime behavior as C++98 (memberwise copy assignment). Generated only if the class lacks a user-declared copy assignment operator. Deleted if the class declares a move operation. Generation is deprecated if the class has a user-declared copy constructor or destructor. |
| **Move constructor & move assignment** | Perform memberwise moving of non-static data members. Generated only if the class contains no user-declared copy operations, move operations, or destructor. |



Things to remember: 
- The special member functions are functions that the ctor automatically generate on their own: default constructor, destructor, copy operations, and move operations. 
- Move operations are generated only for classes when they lack explicitly declared move operations, copy operations, and a destructor. 
- The copy constructor is generated only for classes that lack and explicitly declared copy constructor, and it's deleted if a move operator is declared. The copy assignment operator is generated only for classes lacking an explicitly declared copy assignment operator, and it's deleted if a move operation is declared. Generation of the copy operations in classes with an explicitly declared destructor is deprecated.
- Member function templates never suppress generation of special member functions.  

## **CHAPTER 4: Smart Pointers**

There are four smart pointers: std::auto_ptr, std::unique_ptr, std::shared_ptr, std::weak_ptr. 

std::auto_ptr is from C++98 and can be completely replaced with std::unique_ptr without a second thought. 

### **Item 18: Use std::unique_ptr for exclusive-ownership resource management.**

Unique pointers embody *exclusive ownership* semantics. Meaning that it always owns what it points to. Moving std::unique_ptr tranfers ownership. A non null std::unique_ptr destroys its resource upon destruction. 

One common use is in factory functions (functions that create and return an object). 

It can reasonably be assumed that std::unique_ptr object are the same size as raw pointers, but maybe not when custom deleters enter the picture. This is why a lamba deleter may be prefered. 

```
delInvmt1 = [](Investment* pInvestment) // custom
 { // deleter
	 makeLogEntry(pInvestment); // as
	 delete pInvestment; // stateless
 }; // lambda
 
template<typename... Ts> // return type
std::unique_ptr<Investment, decltype(delInvmt1)> // has size of
makeInvestment(Ts&&... args); // Investment*

void delInvmt2(Investment* pInvestment) // custom
{ // deleter
	 makeLogEntry(pInvestment); // as function
	 delete pInvestment;
}
template<typename... Ts> // return type has
std::unique_ptr<Investment, // size of Investment*
 void (*)(Investment*)> // plus at least size
makeInvestment(Ts&&... params); // of function pointer!

```
They are also useful for the Pimpl Idiom: Pointer to IMPlementation, technique to minimize compilation dependencies and hide implementation details from clients. Separates interface of a class from its private implementation. 

std::unique pointer can be easily converted to shared pointer:

```
std::shared_ptr<Investment> sp = // converts std::unique_ptr
 makeInvestment( arguments ); // to std::shared_ptr
```



Things to remember: 
- std::unique_ptr is a small, fast, move-only smart pointer for managing resources with exclusive-ownership semantics. 
- By default, resource destruction takes place via deletion, but custom deleters can be specified. Stateful deleters and function pointers as deleters increase the size of std::unique_ptr objects. 
- Converting a std::unique_ptr to std::shared_ptr is easy. 