c++primer bind函数
=======================================
学习多线程的时候遇到了，特地再复习记录一下。

一，为什么使用bind函数
“谓词是一个可调用的表达式，其返回结果是一个能用作条件的值，一般作为算法重载的参数使用。STL中所使用的谓词分为两类：一元谓词(只接受一个函数),二元谓词(可以接受两个参数)
根据算法接受一元谓词还是二元谓词，我们传递给算法的谓词必须严格按照接受一个或两个参数。但是，如果在一个算法使用中，需要更多的参数，超出了对谓词的限制。则可以考虑lambda表达式。”
而有了bind函数之后，我们现在可以用bind函数解决了。


二，介绍bind函数
可以将bind函数看做一个通用的函数适配器，接受一个可调用对象，生成一个新的可调用对象来适应原对象的参数列表。
bind函数定义在头文件functional中，调用bind函数的一般形式如下：
auto newCallable=bind(callable,arg_list);
其中newCallable是一个可调用对象，arg_list是一个逗号分隔的参数列表对应给定的callable参数。
arg_list的参数可能包含形如_n的名字，其中n是一个整数。这些参数是占位符，n表示生成的可调用对象中参数的位置。
在调用newCallable时，将会调用callable函数并传入arg_list参数。newCallable的参数个数取决于占位符_n的多少，第一个参数记为_1,第二个参数记为_2，以此类推。对于确定的参数直接填入arg_list中，相当于lambda中的捕获对象。(_n都定义在命名空间placeholder)

三，简单示例
//check6是一个可调用对象，接受一个string类型的参数，并用此string和值6来调用check_size
auto check=bind(check_size,_1,6)；
此bind调用只有一个占位符，表示check6只接受单一参数。占位符出现在arg_list的第一个位置，表示check6的此参数对应check_size的第一个参数。此参数是一个const string&因此调用check6必须传给一个string类型的参数。

四，使用placeholders名字
_1,_2等，是放在了命名空间placeholder中，所以要使用：
using namespace std::placeholders;


五，用bind重排参数顺序

auto g = bind(func, a, b, _2, c, _1);//func是有5个参数的函数
调用g(_1, _2)， 等于 func(a, b, __2, c, _1)


六，绑定引用参数
默认情况下，bind的非占位符参数是值传递，如果想要用引用传递，需要使用标准库函数ref或cref(前者普通引用后者const引用)。
示例：
 for_each(svec.begin(),svec.end(),bind(print, ref(os), _1, cref(c)));
