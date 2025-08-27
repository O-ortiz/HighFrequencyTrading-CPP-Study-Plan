
## Introduction
Terminology and Conventions
-C++11's most important feature: move semantics
-can distinguish between *rvalues* and *lvalues*
-*rvalues*: indicate objects eligible for move operations, while lvalues don't
-In concept:  rvalues correspond to temporary objects returned from functions, while lvalues correspond to objects you can refer to
-heuristic: and expression is an lvalue if you can take its address
-rvalues and lvalues are independent of type, any type T can be either
-Important to remember when dealing with a parameter of rvalue reference type, because the parameter itself if an ltype  (ie the reference is only a temporary object, but the object it is referencing has an address), as seen in 

class Widget {
public: 
	Widget(Widget&& rhs); // rhs is an lvalue, though it has
	                   // an rvalue reference type
	... // means other code should go here
}

Conventions:
-Widget: arbitrary user defined classs
-Use of parameter name "rhs". Author's preferred parameter name for move operations (such as *move constructor* and *move assignment* operator) and *copy operations* (s/a copy constructor and copy assignment operator). Also used for right-hand parameter of binary operators:
	Matrix operator+(const Matrix& lhs, const Matrix& rhs);

-Object create from other object, that is said to be a *copy* of initializing object, even if created using move constructor. 
-No terminology to distinct between copy-constructed copy and one that's move-constructed:

	void someFunc(Widget w); // someFunc's parameter w is passed by value

	Widget wid; // wid is some Widget

	someFunc(wid); // in this call to someFunc, w is a copy of wid that's created via copy constructor
	someFunc(std::move(wid)); // in this call to someFunc, w is a copy of wid that's created via move construction

-copies of rvalues are generally move constructed, copies of lvalues are usually copy constructed
-Expression passed at the call site are the *arguments*. Arguments used to initialize the function's *parameters*. 
-parameters are lvalues, but the arguments used to initialize them may be rvalues or lvalues
-Well designed functions: exception safe, offer at least the basic exception safety guarantee. Assures that if exception arrives, program remains intact, and no resources are leaked. 
-strong exception safety: state of program remains as it was prior to call in case of exception arising
-*function object*: usually means an object of a type supporting an operator() member function. (things in C++ that can invoked using some kind of function-calling syntax)
-closures: function objects created through lambda exercises. 

Many things in C++ can be both declared and defined. Declarations introduce names and types without giving details (s/a where storage is located or their implementation):

	extern int x; // object declaration
	class Widget; // class declaration
	bool func(const Widget& w); // function declaration
	enum class Color; // scoped enum declaration
Definitions provide the storage locations or implementation details:
	int x; // object definition
	class Widget{ // class definition
	...
	};
	bool func(const Widget& w){
	return w.size() < 10; }              // function definition
	enum class Color
	{Yellow, Red, Blue};                    // scoped enum definition
	
-function's *signature*: part of its declaration that specifies parameter and return types. Function and parameter names are not part of the signature. 
-built in pointers, such as those returned from new, called raw pointers. Opposite is called a small pointer

---

# Chapter 1: Deducing Types
## Item 1: Understand template type deduction.

We can think of a function template as looking like:
	template<typename T>
	void f(ParamType param);
A call looks like:
	f(expr);       // call f with some expression 

Compilers use expr to deduce two types: one for T and one for ParamType. Often different, because ParamType can contain adornments (s/a const or reference qualifiers). If the template is declared like:

	template<typename T>
	void f(const T& param); // ParamType is const T&
and we have the call:
	int x = 0;
	f(x)  // call f with an int
T is deduced to be int, but ParamType is deduced to be const int&. 
Three cases:
1. ParamType is a pointer or reference type, but not a universal reference. 
2. ParamType is a universal reference. 
3. ParamType neither a pointer or a reference. 

Three deduction scenarios to examine: 

### **Case 1: ParamType is a Reference or Pointer, but not a Universal Reference**

Type deduction works like this:
1. If expr's type is a reference, ignore the reference part
2. Then pattern-match expr's type against ParamType to determine T

For example:

template<typename T>
void f(T& param);                  // param is a reference

int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int

The deduced types for param and T are:

f(x);  // T is int, param is int&
f(cx); // T is const int, param is const int&
f(rx); // T is const int, param is const int&

If we change the type of f's parameter from T& to const T&:

template<typename T>
void f(const T& param); // Param is now a ref-to-const

f(x);  // T is int, param's type is const int&
f(cx); // T is int, param is const int&
f(rx); // T is int, param is const int&

If a param were a pointer (or pointer to const):

template<typename T>
void f(T* param);

int x = 27;
const int *px = &x;  // px is a pointer to x as a const int
f(&x);                // T is int, param is int*
 f(px);               //  T is const int, param is const int* 

### **Case 2: ParamType is a Universal Reference**

Declared like rvalue refences (Type T, universal reference's declared type is T&&), but behave differently when lvalue arguments are passed in.

-If expr is an lvalue, both T and ParamType are deduced to be lvalue references. Only time where T is deduced to be a reference. ParamType declared using syntax for an rvalue reference, its deduced type is an lvalue reference. 
-If expr is an rvalue, the "normal" rules apply (case 1)

Example:
template<typename T>
void f(T&& param);     // param is now a universal reference

int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int

f(x); // x is lvalue, so T is int&, param is also int&
f(cx); // cx is lvalue, so T is const int&, param is also const int&
f(rx); // rx is a lvalue, so T is const int&, param is also const int&
f(27); // 27 is rvalue, so T is an int, and param is int&&

Point: When universal reference are in use, type deductions distinguishes between lvalue arguments and rvalue arguments. 

### **Case 3: ParamType is Neither a Pointer nor a Reference**

When neither pointer or reference, dealing with pass-by-value:
	template<typename T>
	void f(T param);
This makes param a copy of whatever is passed in, completely new object. 

1. If expr's type is a reference, ignore the reference part. 
2. If, after ignoring expr's reference-ness, expr is const ignore that too. If it's volatile, also ignore that. 

int x = 27; // x is an int
const int cx = x; // cx is a const int
const int& rx = x; // rx is a reference to x as a const int

f(x); // T and param are ints
f(cx); // T and param are ints
f(rx); // T and param are ints

Consider when expr is a const pointer to a const object, and expr is passed to a by-value param:

template<typename T>
void f(T param);

const char* const prt = // pointer itself is const, pointing to a const char
"Fun with pointers";

f(prt); // pass arg of type const char* const

param is of type const char *, since the pointer ptr was passed by reference, the const part of it is ignored since a *copy was made*

**Array Arguments**

const char name[] = "J. P. Briggs";
const char* prtToName = name; // array decays to pointer


Array to pointer decay rule, but what happens with a template?

	
	template<typename T>
	void f(T param);

	f(name); // what types are deduced for T and param?

Array parameter declarations are treated as if there were pointer parameters, so the type of an array that's passed to a template function by value is deduced to be a pointer type. 

	f(name); // name is array, but T is deduced as const char*

But, can declare parameters that are references to arrays, so:

	template<typename T>
	void f(T& param); // template with by-reference parameter
and we pass an array to it:
	f(name);
the type deduced is, T const chat[13], param is const char(&)[13]

**Function Arguments**
Function types can decay into function pointers, so everything for type deduction for arrays applied to type deduction from functions. 

Thing to Remember:
-During template type deduction, arguments that are references are treated as non-references (reference-ness is ignored)
-When deducing types for universal reference parameters, lvalue arguments get special treatment
-When deducing type for by-value parameters, const, and/or volatile arguments are ignored
-During template types, arguments that are array of function names decay to pointers, unless they're used to initialize references


---

## **Item 2: Understand auto type deduction.**

auto x = 27; // Here the type specifier for x is simple auto by itself
const auto cx = x; // type specifier is const auto
const auto& rx = x;  // type specifier is const& auto

To deduce type, compiler acts as if there were a template for each declaration as well as a call to that template with the corresponding initializing expression: 

	template<typename T> // conceptual template for deducing x's type
	void func_for_x(T param);

	func_for_x(27); // conceptual call: param's deduced type is x's type

	template<typename T> 
	void func_for_cx(const T param);

	func_for_cx(x); // param's deduced type is cx's type

	template<typename T>
	void func_for_rx(const T& param);

	func_for_rx(x); 
Deducing types for auto is the same as deducing types for templates. 

Declare a type int with value 27:
auto x1 = 27; 
auto x2(27);

declare a variable of type std::initializer_list<int> containing a single element with value 27: 
auto x3 = { 27 }; 
auto x4{ 27 };

C++14 permits auto to indicate that a function's return type should be deduced, and C++14 lambdas may use auto in parameter declarations. These uses of auto use template type deductions, not auto type deductions. 

---

## **Item 3: Understand decltype.**

decltype tells you the name's or the expression's type. 

	const int i = 0; // decltype(i) is const int
	bool f(const Widget& w); // decltype(w) is const Widget& , decltype(f) is bool(Widget&)
	
Primary use of decltype in C++11 is declaring function templates where the function's return type depends on its parameter types

operator[] typically returns a T&, but it depends on container. 

example (needs refining):
	template<typename Container, typename Index>
	auto authAndAcess(Container& c, Index i) -> decltype(c[i])
	{
		authenticateUser();
		return c[i];
	}
use of auto is to indicate that trailing return type syntax is being used (->), has the advantage that the function's parameters can be used in the specification of the return type

C++11 allows return types for single-statement lambdas to be deduced, and C++14 extends this to both lambdas and all functions (including those with multiple statements)

To get authAndAccess to work, we need to use decltype(auto) in C++14.
auto - specifies that the type is deduce
decltype - says that decltype rules should be used during the deduction

template<typename Container, typename Index>
decltype(auto)
authAndAccess(Container& c, Index i)
{
	authenticateUser();
	return c[i];
}

decltype(auto) can also be used for claring variables when you want to apply the decltype type deduction rules:

	Widget w;
	const Widget& cw;
	auto myWidget1 = cw;  // auto type declaration, type is Widget (ignores &)
	decltype(auto) myWidget2 = cw; // myWidget2's type is const Widget&

We can use a reference parameter that binds to both lvalues and rvalues (which is what universal references do):
	template<typename Constainer, typename Index> 
	decltype(auto) authAndAccess(Container&& x, index i); 
Pass by value may cause performance issues when copying, but we stick with pass-by-value for index values (following the example of operator[] in std::string, std::vector, and std::deque)

We want to employ std::forward to universal references (final C++14 version):

	template<typename Container, Index>
	decltype(auto)
	authAndAcess(Container&& c, Index i)
	{
		autheticateUser();
		return std::forward<Container>(c)[i];
	}
In C++11 we want to do since decltype(auto) cannot be use for functions in this version: 
	template<typename Container, Index>
	auto
	authAndAcess(Container&& c, Index i)
	-> decltype(std::forward<Container>(c)[i])
	{
		autheticateUser();
		return std::forward<Container>(c)[i];
	}

Curiously, wrapping a name in parantheses changes it to be more complex than a name, so:

int x = 0; decltype (x) will give int but "(x)" will give type int&
	
	dectype(auto) f1()
	{
		int x = 0;
		return x; // decltype(x) is int, so f1 returns int
	}
	
	dectype(auto) f2()
	{
		int x = 0;
		return (x);  // decltype((x)) is int&, so f2 returns int&
	}

Things to remember:
-decltype almost always yield the type of a variable or expression without modifications
-For lvalue expressions of type T other than names, decltype always reports a type T&
-C++14 supports decltype(auto), deduces a type from its initializer, but performs the type deduction using decltype rules