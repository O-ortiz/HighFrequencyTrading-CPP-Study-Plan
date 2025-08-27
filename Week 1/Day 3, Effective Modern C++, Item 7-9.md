## Chapter 3: Moving to Modern C++

### Item 7: Distinguish between () and {} when creating objects.

"braced initialization"

specifying initial contents of container is easy:

	std::vector<int> v{1, 3, 5};

Can be used to specify default initializations values for non-static data members. 

	class Widget{
	...
	int x{0}; // fine, x's default value is zero
	int y = 0; // also fine
	int z(0); // error!
	};
Uncopiable objects (like std::atomics) may be initialized using braces or parnatheses, but not "="

	std::atomic ai1{ 0 }; // fine 50
	std::atomic ai2(0); // fine 
	std::atomic ai3 = 0; // error!

This is why {} initialization is considered "uniform", only one that can be used everywhere. 

Prohibits implicit *narrow conversions*, code won't compile if value of expression in bared initializer isn't guaranteed expressible by the type of object being initialized:

double x, y , z;

int sum1{x  + y + z}; // Error! sum of doubles may not be expressible as int

Immunity to C++'s *most vexing parse*:
-Anything that can be parsed as a declaration must be interpreted as one
Widget  w1(10); // Widget ctor with arg. 10
Widget w2(); // most vexing parse! declares function named w2 that returns a Widget

This cannot happen with {}, so:
Widget w3{}; // calls Widget ctor with no args

Braced initialization can lead to surprising behavior, like when auto deduces std::initializer_list. 

Design class constructors so that the overload called isn't affected by the use of parentheses or braces like in std::vector. 

Things to remember:
-Braced initialization is most widely usable initialization syntax prevent narrow conversion, and is immune to *most vexing parse*
-During ctor overload resolutions, braced initializers are matched to std::initializer_list parameters is at all possible, even if other constructors are better suited 
-Example where difference between intializers is significant is in std::vector<numeric type>
-Choosing between parantheses and braces for object creation inside templates can be challenging

### **Item 8: Prefer nullptr to 0 and NULL**

Neither 0 nor NULL have a pointer type. 
Nullptr advantage: it doesn't have an integral type

Things to remember:
-Prefer nullptr to 0 and NULL
-Avoid overloading on intergral and pointer types


### Item 9: Prefer alias declarations to typedef

Typedef example:
typedef
std::unique_ptr<std::unordered_map<std::string, std::string>>
UPtrMapSS;

Creates an object of unique ptr of an unordered map that maps a string to a string called UPtrMapSS. 

Using alias declarations:

using UPtrMapSS =
std::unique_ptr<std::unordered_map<std::string, std::string>>;

Templates are why we should prefer alias declarations, since they can be templatized. 

In TMP, you can take some types and change them using type traits (C++11), and C++14 has an alias equivalent:
	std::remove_const<T>::type // C++11: const T → T
	std::remove_const_t<T> // C++14 equivalent
	std::remove_reference<T>::type // C++11: T&/T&& → T
	std::remove_reference_t<T> // C++14 equivalent
	std::add_lvalue_reference<T>::type // C++11: T → T&
	std::add_lvalue_reference_t<T> // C++14 equivalent

Things to remember:

-Typedefs don't support templatization, but alias declarations do
-Alias templates avoid the ::type suffix, and in templates, the "typename" prefix often required to refer to typedefs
-C++14 offers alias templates for all the C++11 type traits transformations