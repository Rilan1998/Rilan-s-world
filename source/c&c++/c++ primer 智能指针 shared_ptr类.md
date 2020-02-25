c++primer 笔记心得 智能指针 shared_ptr类 
===================
一，智能指针引入背景-----两种内存管理不善的情况：
1，忘记释放，内存泄漏。
2，指针引用非法内存。

二，智能指针如何解决上述问题
智能指针能自动释放所指向的对象，通过计算引用计数，在其为0是释放内存，新手入门可类比linux中的硬链接理解。
note：智能指针是没末班，创建智能指针时需提供额外信息T---智能指针可以指向的类型。

三，shared_ptr类
允许多个指针指向同一对象的智能指针。
官方文档：http://www.cplusplus.com/reference/memory/shared_ptr/

操作列表1(shared_ptr和unique_ptr均支持的操作)：
shared_ptr<T> sp  //空智能指针，可以指向类型为T的对象。
unique_ptr<T> up

p  //用作表达式时作用为判断指针是为空。
*p //解引用，获取p指向的对象

p->mem	//等于(*p).mem，如果返回指针指向类对象中的成员变量或函数。

p.get()	//返回智能指针p中保存的普通指针,若智能指针释放了其对象，返回的指针会指向非法内存，使用禁忌：不要用get初始化获赋值另外的智能指针。

swap(p, q) 或 p.swap(q)	//交换智能指针p和q中的普通指针

操作列表2(shared_ptr独有操作)：
make_shared<T>(args)	//返回一个shared_ptr，它指向动态分配的T类型对象，并用args初始化

shared_ptr<T>p(q)	//使p作为q的拷贝，q的引用计数+1

p = q	//p的引用计数-1，q 的引用计数+1

p.unique()	//若p的引用计数=1，则返回true

p.use_count()	//返回与p共享对象的智能指针的数量


常规使用方法：make_shared函数（该用法为推荐且最安全用法，以下不再对shared_ptr和new结合做多余笔记）
该函数在动态内存分配中分配一个对象并初始化它，返回指向此对象的shared_ptr。传入的参数需与T的类型的构造函数相匹配，如不传参，对象将进行值初始化。
note：该函数定义在头文件memory中。

shared_ptr<int> p3 = make_shared<int>(42);// 指向一个值为42的int

shared_ptr<string> p4 = make_shared<string>(10, '9');// 指向一个值为"9999999999"的string

shared_ptr<int> p5 = make_shared<int>();// 指向一个值初始化(为0)的int

auto p6 = make_shared<vector<string>>();// 指向一个动态分配的vector<string>， p5操作的简化上位取代方式

其他shared_ptr操作
我们可以用reset将一个新的指针赋予一个shared_ptr，例如
p.reset(new int (1024))；   

reset成员常与unique一起使用，来控制多个shared_ptr共享的对象。在改变底层对象之前，我们检查自己是否是当前对象仅有的用户。如果不是，在改变之前要制作一份新的拷贝：
if(!p.unique())
　　p.reset(new string(*p));      // 我们不是唯一用户；分配新的拷贝
*p += newVal;      // 现在我知道自己是唯一用户，可以改变对象的值

