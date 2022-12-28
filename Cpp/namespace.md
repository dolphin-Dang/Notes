# 创建命名空间
在C++中引入命名空间的概念，来解决命名冲突的问题。

创建命名空间语法：

    namespace Number
    {
        int add(int a, int b)
        {
            return a + b;
        }
    }

使用命名空间中的函数：

	std::cout << Number::add(2, 4) << std::endl;

**注意：** 如果使用多文件，应当在头文件中仅声明函数，在cpp文件中实现函数。否则会报错。

# Defualt Global Namespace
默认全局命名空间。

如果要在其它命名空间中调用默认全局命名空间的函数，可以使用“::”。

如果在全局有一个函数

    add(int a, int b)
    {
        return a + b;
    }

那么在也有一个同名add函数的Number命名空间中调用全局的ad函数需要使用“::”

    namespace Number
    {
        add(int a, int b)
        {
            return a + b + 1;
        }
        add_int(int a, int b)
        {
            return ::add(a, b) + 2;
        }
    }

此时在main中调用：

    std::cout << add(2, 4) << std::endl;
    std::cout << Number::add(2, 4) << std::endl;
    std::cout << Number::add_int(2, 4) << std::endl;

输出结果为6、7、8。

# Using的用法
**强调**： 不要在头文件中使用using！！！
+ 只引入类
    
        using Number::A;
        //只引入了Number命名空间中的A类

+ 只引入函数

        using Number::add_int;
        //只引入了Number命名空间中的add_int函数

+ 引入命名空间

    要注意作用域问题：

        {
		    using namespace Number;
        }
        std::cout << add(2, 4) << std::endl;
        //这样会报错！！！

    如果出现命名空间函数名冲突的问题，使用“命名空间::”修饰即可。

# 嵌套命名空间与别名

嵌套的命名空间：

    namespace Number
    {
        int add(int a, int b)
        {
            return a + b;
        }
        namespace nextNumber
        {
            int add(int a, int b)
            {
                return a + b + 1;
            }
        }
    }

在main函数中调用nextNumber中的函数add：

	std::cout << Number::nextNumber::add(2, 4) << std::endl;

为nextNumber起一个别名data并限定在作用域中：

	{
		namespace data = Number::nextNumber;
		std::cout << data::add(2, 4) << std::endl;
	}
