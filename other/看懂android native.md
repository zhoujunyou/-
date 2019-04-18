###目标 看懂android native

* sizeof 运算符查询对象或类型的大小 返回一个 [std::size_t](https://zh.cppreference.com/w/cpp/types/size_t) 类型的常量。

 * ~IOCanary() 构造函数前有～ 析构函数类似java的finlize方法 调用delete后调用

 * IOCanary kInstance  C++中这样不仅是声明而且用默认构造方法给kInstance赋值了

 * C++静态成员 和函数调用需要用<类名>::<静态成员名>  ，<类名>::<静态函数名> ，静态成员变量只存储一份供所有对象共用

 * static IOCanary kInstance; **静态局部变量，局部作用域（只在局部作用域中可见）程序运行期一直存在，全局数据区，只被初始化一次，多线程中需加锁保护**

 * [C++ 11 多线程--线程管理](https://www.cnblogs.com/wangguchangqing/p/6134635.html)

 *  [C++11 并发指南五(std::condition_variable 详解)](https://www.cnblogs.com/haippy/p/3252041.html)


