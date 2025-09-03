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

