### **Item 13: Item 13: Prefer const_iterators to iterators.**

Using 'const_iterators' is trivial:

```
std::vector<int> values; // as before
…
auto it = // use cbegin
std::find(values.cbegin(),values.cend(), 1983); // and cend

values.insert(it, 1998);
```

We can generalize this into a templated function that will work for C++14 but not C++11:

```
template<typename C, typename V>
void findAndInsert(C& container, // in container, find
 const V& targetVal, // first occurrence
 const V& insertVal) // of targetVal, then
{ // insert insertVal
	 using std::cbegin; // there
	 using std::cend;
	 auto it = std::find(cbegin(container), // non-member cbegin
	 cend(container), // non-member cend
	 targetVal);
	 container.insert(it, insertVal);
}
```


Things to remember: 
- Prefer const_int to iterators
- In maximally generic code, prefer non-member versions of begin, end, rbegin, etc, over their member function counterparts.  
### **Item 14: Declare functions noexcept if they won’t emit exceptions.**

Using noexcept is an optimization over functions without it since it allows compilers to generate better object code. This is because with exception specification, the stack is only possibly unwound before program execution. 

```
RetType function(params) noexcept; // most optimizable
RetType function(params) throw(); // less optimizable
RetType function(params); // less optimizable
```

Generally most functions will be *exception neutral*, unless you can gaurantee that the function will no throw an exception and neither will any of the processes it calls. 

Functions with wide contacts generally are able to be called with none of few reconditions. This means that these are generally better to set as noexcept because those with narrow contacts might exhibit undefined behavior if they are noexcept but don't meet their preconditions. 

It is allowed for a function that calls functions that aren't noexcept to be noexcept, the compiler won't complain. It is up to the programmer to be sure. 

Things to remember:
- noexcept is part of a function's interface, so callers might depend on it
- noexcept functions are more optimizable than non-except functions
- noexcept is particularly viable for the move operations, swap, memory deallocation, and destructors
- Most functions are exception-neutral rather than noexcept

### **Item 15: Use constexpr whenever possible.**

 Constrexpr for object is a beefed up const, indicates when something is known at compilation. 

Values known at compilation time are privileged and can be placed in read-only memory. 

Integral values that are constant and known during compilation can be used in conctexts where C++ requires an integral constant expression, such as specifying size of std::array)

```
int sz; // non-constexpr variable
…
constexpr auto arraySize1 = sz; // error! sz's value not
 // known at compilation
 
std::array<int, sz> data1; // error! same problem

constexpr auto arraySize2 = 10; // fine, 10 is a
 // compile-time constant
 
std::array<int, arraySize2> data2; // fine, arraySize2
 // is constexpr

```

All constexpr are const, but not all const are constexpr. 

Constexpr functions: 
- can produce a compile-time constant when they are called with compile-time constats. 
- if they are called with values not known until runtime, they produce runtime values. 

This means you don't need two functions to perform the same operation if one of them needs to be able to return a compile-time constant. 

Restrictions are imposed on constexpr function since they must be able to return compile-time results:
- In C++11, constexpr functions may contain no more than a single executable statement: a return. Can use two trick, conditionals and recursion. 
- . In C++14, the restrictions on constexpr func‐ tions are substantially looser, so the following implementation becomes possible:
```
constexpr int pow(int base, int exp) noexcept // C++14
{
	 auto result = 1;
	 for (int i = 0; i < exp; ++i) result *= base;
	 return result;
}

```

Constexpr functions are limited to taking and returning literal types, basically types that can have values determined during compilation. 
In C++11 all built in types except void qualify.

```
class Point {
public:
 constexpr Point(double xVal = 0, double yVal = 0) noexcept
 : x(xVal), y(yVal)
 {}
 constexpr double xValue() const noexcept { return x; }
 constexpr double yValue() const noexcept { return y; }
 void setX(double newX) noexcept { x = newX; }
 void setY(double newY) noexcept { y = newY; }
private:
 double x, y;
};
```

In C++11, setX and setY cannot be constexpr, because:
- They modify the object they operate on, and in C++11, constexpr member functions are implicity const. 
- They have void return types, which isn't a literal type in C++11

Both restrictions lifted in C++14, so we can set Point's setter as constexpr:
```
class Point {
public:
 …
 constexpr void setX(double newX) noexcept // C++14
 { x = newX; }
 constexpr void setY(double newY) noexcept // C++14
 { y = newY; }
 …
};
```

constexpr is part of an object’s or function’s interface, it proclaims "“I can be used in a context where C++ requires a constant expression." Clients may use a constexpr in that context, if you decided to change that later it may cause issues or disallow compilation. 

Things to remember:
- constexpr objects are consts and are initialized with values known during compilation
- constexpr functions can produce compile-time results when called with arguments whose values are known during compilation
- constexpr objects and functions may be used in wider range of contexts than non-constexpr objects and functions 
- constexpr is part of an objects' or function's interface 

