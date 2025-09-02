### **Item 10: Prefer scoped enums to unscoped enums**

Enums declared like this:
	enum Color {black, white, red}; 
Bleed into the scope, not allowing for something like:
	auto white = false;  // This gives an error
so you need scoped enums:
	enum class Color {black, white, red };
	auto white = false; // this is fine

Scoped enums also called enum classes, reduces namespace pollution. 2nd compelling advantage: their enumerators are strongly typed.

There is no implicit conversion to ints when using scoped enums. If you want to do this, use a cast:

	if(static_cast<double>(c) < 14.5){  
	auto factors = primeFactors(static_cast<std::size_t>(c));
	..
	}

C++11's enums can be forward declared (don't need to specify enumerators):
	enum class Color;
And eliminates needing to to recompile the whole system:
	enum class Status; // forward declaration
	void continueProcessing(Status s); // use of fwd-declared enum
This needs no recompilation if Status's definition is revised. 

Can define underlying type for scoped enums (is in by default):
	enum class Status: std::uint32_t;
Can also do that for unscoped enum and can be forward declared:
	enum Color: std::uint8_t;
and can go on definition:
	enum class Status: std::uint32_t { good = 0,
	failed = 1,
	incomplete = 100,
	corrupt = 200,
	audited = 500,
	indeterminate = 0xFFFFFFFF
	};
Might want unscoped enum in case of when using tuple:
	using UserInfo = // type alias; see Item 9
	std::tuple<std::string, // name
	Item 10 | 71
	std::string, // email
	std::size_t> ; // reputation

	UserInfo uInfo; 
	auto val = std::get<1>(uInfo); // Might not remember what field is what

So instead we use an unscoped enum:
	enum UserInfoFields { uiName, uiEmail, uiReputation };
	
	UserInfo uInfo;
	auto val = std::get<uiEmail>(uInfo);

Things to remember:
-C++98 style enums are unscoped enums
-Scoped enum enumerators only visible within its scope, can be type cast
-Scoped enums have int underlying type by default
-Scoped enums can be forward declared. Unscoped can be only if their underlying type is declared. 


### **Item 11: Prefer deleted functions to private undefined ones**

Sometimes you want certain automatically declared functions (special member functions") inaccessible to a client. 
Old way around that was to declare the function as private, but not define it. 
Better way is to delete the function:

	template <class charT, class traits = char_traits<charT> > 
	class basic_ios : public ios_base {
	public:
		...
		basic_ios(const basic_ios& ) = delete;
		basic_ios& operator=(const basic_ios&) = delete;
		...
	};
Deleted functions cannot be used in any way, so even code in member and friend functions fail to compile. 

Deleted functions are declared public by convention.  

Any function may be deleted, while only member functions can be private. 

We can prevent certain overloads by doing:

```
bool isLucky(int number);
bool isLucky(char) = delete;
bool isLucky(bool) = delete;
bool isLucky(double) = delete; 
```

Can also use delete to prevent use of template instantiations that should be disabled:
```
template<typename T>
void processPointer(T* ptr);

template<>
void processPointer(void*) = delete;

template<>
void processPointer(char*) = delete;
```

Things to remember:
- Prefer deleted functions to private undefined ones
- Any functions may be deleted, including non-member functions and template instantiations

### **Item 12: Declare overriding functions override**

Virtual function - function in base class that is expected to be overwritten in derived class

For overriding to occur, several things must happen:
1. Base class function is declared virtual
2. Base and derived class function names but be the same (except for constructors and destructors)
3. Parameter types of base and derived functions must be identical
4. Constness of both must be identical 
5. Return types and exception specification of both must be compatible
A special case only for C++11:
6. Functions' reference qualifiers must be identical. 

Reference qualifiers: 
- Make it possible to limit use of a member function to only lvalues or rvalues (of *this)
```
class Widget {
public:
	 …
	 void doWork() &; // this version of doWork applies
	 // only when *this is an lvalue
	 void doWork() &&; // this version of doWork applies
}; // only when *this is an rvalue
…

Widget makeWidget(); // factory function (returns rvalue)
Widget w; // normal object (an lvalue)

…

w.doWork(); // calls Widget::doWork for lvalues
 // (i.e., Widget::doWork &)
 
makeWidget().doWork(); // calls Widget::doWork for rvalues
 // (i.e., Widget::doWork &&)

```


You should always call override functions override, so that you get compiler warnings if there are issues with the override function.  
```
class Base {
public:
 virtual void mf1() const;
 virtual void mf2(int x);
 virtual void mf3() &;
 virtual void mf4() const;
};

class Derived: public Base {
public:
 virtual void mf1() const override;
 virtual void mf2(int x) override;
 virtual void mf3() & override;
 void mf4() const override; // adding "virtual" is
```

You can use reference qualifiers to overload a member function for lvalue or rvalue this*:

```
class Widget {
public:
 using DataType = std::vector<double>;
 …
 DataType& data() & // for lvalue Widgets,
 { return values; } // return lvalue
 DataType data() && // for rvalue Widgets,
 { return std::move(values); } // return rvalue
 …
private:
 DataType values;
};
```

For if you are dealing with a situation where you need to make a new object that uses a copy or move construct depending on if the returned value is lvalue or rvalue. 

```
auto vals1 = w.data(); // calls lvalue overload for
 // Widget::data, copy-
 // constructs vals1
 
auto vals2 = makeWidget().data(); // calls rvalue overload for
 // Widget::data, move-
// constructs vals2

```

Things to remember:
- Declare overriding functions override
- Member function reference qualifiers make it possible to treat lvalue and rvalue (*this) differently. 
