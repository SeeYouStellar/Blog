### c++三个特性

1. 封装性：封装可以隐藏实现细节，使得代码模块化。封装是把过程和数据包围起来，对数据的访问只能通过已定义的成员函数。（与c不同点在于，c++

   struct类型的所有成员都是public权限，而c++引进了类，使得类内的成员可以有private权限，通常将想要隐藏的信息设为private权限）

2. 继承性：

3. 多态性：一个接口，多种方法。有两种多态：

   1. 编译时多态性（静态多态）：通过重载函数实现
   2. 运行时多态性（动态多态）：通过虚函数实现（基类指针指向子类对象时，调用基类和子类同名的函数时，调用的是子类的函数） 

### 智能指针

### 强转

1. static_cast<>:

   static_cast 用于进行比较“自然”和低风险的转换，如整型和浮点型、字符型之间的互相转换。另外，如果对象所属的类重载了强制类型转换运算符 T（如 T 是 int、int* 或其他类型名），则 static_cast 也能用来进行对象到 T 类型的转换。

   static_cast 不能用于在不同类型的指针之间互相转换，也不能用于整型和指针之间的互相转换，当然也不能用于不同类型的引用之间的转换。因为这些属于风险比较高的转换.

2. reinterpret_cast<>:

   reinterpret_cast 用于进行各种不同类型的指针之间、不同类型的引用之间以及指针和**能容纳指针**的整数类型之间的转换。注意这里的容量，只有容量能完整存下指针的类型才行。比如int*（8Byte）可转为long(8Byte)，不能转为int(4Byte)

3. const_cast<>：

   const_cast 运算符仅用于进行去除 const 属性的转换，它也是四个强制类型转换运算符中唯一能够去除 const 属性的运算符。将 const 引用转换为同类型的非 const 引用，将 const 指针转换为同类型的非 const 指针时可以使用 const_cast 运算符。例如：

   ```c++
   const string s = "Inception";
   string& p = const_cast <string&> (s);
   string* ps = const_cast <string*> (&s);  // &s 的类型是 const string*
   ```

4. dynamic_cast<>:

   dynamic_cast专门用于将多态基类的指针或引用强制转换为派生类的指针或引用，而且能够检查转换的安全性。

   ```c++
   #include <iostream>
   #include <string>
   using namespace std;
   class Base
   {  //有虚函数，因此是多态基类
   public:
       virtual ~Base() {}
   };
   class Derived : public Base { };
   int main()
   {
       Base b;
       Derived d;
       Derived* pd;
       pd = reinterpret_cast <Derived*> (&b);
       if (pd == NULL)
           //此处pd不会为 NULL。reinterpret_cast不检查安全性，总是进行转换
           cout << "unsafe reinterpret_cast" << endl; //不会执行
       pd = dynamic_cast <Derived*> (&b);
       if (pd == NULL)  //结果会是NULL，因为 &b 不指向派生类对象，此转换不安全
           cout << "unsafe dynamic_cast1" << endl;  //会执行
       pd = dynamic_cast <Derived*> (&d);  //安全的转换
       if (pd == NULL)  //此处 pd 不会为 NULL
           cout << "unsafe dynamic_cast2" << endl;  //不会执行
       return 0;
   }
   ```

   