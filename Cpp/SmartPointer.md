# 前言
在C++11中引入了智能指针的概念，使得程序员不需要手动释放内存。

智能指针的种类：
+ std::unique_ptr
+ std::shared_ptr
+ std::weak_ptr

*注意：std::auto_ptr已经被废弃*

# 智能指针概述
C++的指针包括两种：
+ 原始指针（raw pointer）
+ 智能指针

并不是所有指针都能封装成智能指针，很多时候原始指针更加方便。weak_ptr是shared_ptr和unique_ptr的补充，使用场景较少。

智能指针只解决一部分问题，即独占/共享所有权指针的释放、传输。智能指针没有从根本上解决C++内存安全问题，不加以注意依然会造成内存安全问题。

*注：使用智能指针需要：*

    #include <memory>


# 独占指针： unique_ptr
+ 在任何给定的时刻，只能有一个指针管理内存
+ 当该指针超出作用域时，内存将自动释放
+ 该类型指针不可copy，只能move

有三种创建方式：
1. 通过已有的裸指针创建
2. 通过new创建
3. 通过std::make_unique创建（推荐）

例子：

类的构造：

    class Cat {
    public:
        Cat(std::string name) : name(name)
        {
            cout << "Constructor of Cat: " << name << endl;
        }
        Cat() = default;
        ~Cat() 
        {
            cout << "Destructor of Cat: " << name << endl;
        }
        //->
        void catInfo() const 
        {
            cout << "cat info name: " << name << endl;
        }
        std::string getName() const
        {
            return name;
        }
        void setName(std::string name)
        {
            this->name = name;
        }
    private:
        std::string name{ "Mimi" };
    };

unique_ptr的三种创建方式举例：
1. 通过原始指针创建

        Cat* pCat = new Cat("fan");
        std::unique_ptr<Cat> upCat{ pCat };

    此后原始指针pCat也可以继续使用。两者指向同一个对象，改变pCat也会影响upCat的内容，因此建议销毁pCat。

    	delete pCat;
	    pCat = nullptr;

2. 通过new创建

    	std::unique_ptr<Cat> upCat{ new Cat("fan")};

3. make_unique

    	std::unique_ptr<Cat> upCat = make_unique<Cat>("fan");

# unique_ptr与函数
## pass by value

+ 需要使用std::move来转移内存所有权
+ 如果参数直接传入std::make_unique语句，自动转换为move

例子：

    void showInfo(std::unique_ptr<Cat> p)
    {
        p->catInfo();
    }

    std::unique_ptr<Cat> upCat = make_unique<Cat>("fan");
    showInfo(std::move(upCat));
    upCat->catInfo();
    //因为内存所有权已经转移，所以这个调用会出错
    showInfo(std::unique_ptr<Cat>());
    //调用默认构造函数，并且自动move

## pass by ref

    void showInfo(std::unique_ptr<Cat> &p)
    {
        p->setName("drf");
        p->catInfo();
        p.reset();
    }
    
这里的[reset方法](https://blog.csdn.net/lzn211/article/details/109147985)会清空指针，并释放空间。

    std::unique_ptr<Cat> upCat = make_unique<Cat>("fan");
    showInfo(upCat);
    cout << "cat address: " << upCat.get() << endl;

[get()方法](https://blog.csdn.net/qq_41452267/article/details/108488864)可以获得指针指向的地址。

运行结果：
> Constructor of Cat: fan
>
> cat info name: drf
>
> Destructor of Cat: drf
>
> cat address: 0000000000000000

可以用函数返回一个智能指针（少用）：

    std::unique_ptr<Cat> getUniquePtr()
    {
        std::unique_ptr<Cat> p = std::make_unique<Cat>("local cat");
        return p;
    }

并用链式方法使用之：

    getUniquePtr()->catInfo();

**总结：**要尽量使用pass by reference而非pass by value。因为后者使用完之后所有权就转移了，之后没法再次使用这个指针。

# 计数指针： shared_ptr
shared_ptr计数指针又称共享指针。与unique_ptr不同之处在于它可以共享数据，也就是说计数指针是可以Copy的。

shared_ptr创建了一个计数器与类对象所指的内存相关联。copy则计数器加一，销毁则计数器减一。可以通过API：use_count()来获取计数器中的数字。

利用make_shared创建计数指针：

	std::shared_ptr<int> spInt = make_shared<int>(10);
	cout << spInt.use_count() << endl;
    //此处输出结果为1

copy计数指针：

	std::shared_ptr<int> spInt2 = spInt;
	cout << spInt.use_count() << endl;
	cout << spInt2.use_count() << endl;
    //此处两个输出皆为2

销毁一个指针：

	spInt2 = nullptr;
	cout << spInt.use_count() << endl;
	cout << spInt2.use_count() << endl;
    //此处输出为1和0

[为什么要用make_shared来创造计数指针？](https://blog.csdn.net/q2519008/article/details/100560402)
# shared_ptr与函数
## pass by value
创建函数：

    void catByVal(std::shared_ptr<Cat> cat)
    {
        cout << "func use count : " << cat.use_count() << endl;
    }

在main函数中：

	std::shared_ptr<Cat> c = make_shared<Cat>("fan");
	catByVal(c);
	cout << "main use count : " << c.use_count() << endl;

输出为：
>func use count : 2
>
>main use count : 1

**结论：** pass by value 传入函数之后，计数器会加一，但是函数外面不会变。因为函数结束之后局部的指针就会销毁，计数器会减一。**要注意多线程问题。**

## pass by ref
将函数改为：

    void catByRef(std::shared_ptr<Cat> &cat)
    {
        cout << "func use count : " << cat.use_count() << endl;
    }

在main函数中调用，输出为：

>func use count : 1
>
>main use count : 1

**结论：** pass by ref 传入函数之后，计数器不会变化。

# shared_ptr与unique_ptr的转化
+ 不能将shared_ptr转化为unique_ptr
+ unique_ptr可以转化为shared_ptr： 通过std::move

常见的设计：使用函数返回unique_ptr，这样可以提高代码的复用度，因为可以随时将其改变为shared_ptr。

转化代码：

	std::unique_ptr<Cat> upCat = std::make_unique<Cat>("drf");
	std::shared_ptr<Cat> spCat = std::move(upCat);

转化之后upCat因为已经被move，不再能被使用。而spCat的计数器为1。

也可以直接用shared_ptr来接收一个函数返回的unique_ptr：

	std::shared_ptr<Cat> upCatFromU = retUniquePtr();
	cout << "use count: " << upCatFromU.use_count() << endl;
    //此处输出为1

# weak_ptr： shared_ptr的补充
weak_ptr是对该内存的弱引用，并不拥有内存的所有权，因此不能调用->和解引用*。

## weak_ptr为什么会存在？
如果：
+ A类中有一个需求需要存储其它A类对象的信息
+ 如果使用shared_ptr，那么在销毁的时候会出现循环依赖的问题
+ 所以我们这里需要用一个不需要所有权的指针来标记该同类对象

## weak_ptr的创建
weak_ptr不能单独存在，必须依赖于shared_ptr。

	std::shared_ptr<Cat> spCat = std::make_shared<Cat>("drf");
	std::weak_ptr<Cat> wpCat(spCat);
	cout << wpCat.use_count() << endl;
    //此处输出为1

也就是说weak_ptr可以用“.”调用函数，但是计数器不会加一。

weak_ptr可以通过lock()函数来提升为shared_ptr：

	std::shared_ptr<Cat> spCatFromW = wpCat.lock();
	cout << spCat.use_count() << endl;
    //此处输出为2

weak_ptr可以通过expired()函数来查看指向的对象是否被销毁。

## weak_ptr的使用例子
给Cat类新增一个private数据：

	std::shared_ptr<Cat> catFriend;

再新增一个public成员函数：

	void setFriend(std::shared_ptr<Cat> c)
	{
		catFriend = c;
	}

然后在main函数中构造c1、c2两个对象，互相设为friend：

	std::shared_ptr<Cat> c1 = std::make_shared<Cat>("fan");
	std::shared_ptr<Cat> c2 = std::make_shared<Cat>("drf");
	c1->setFriend(c2);
	c2->setFriend(c1);

运行之后发现没有输出c1、c2的析构函数中的cout语句，因为形成了循环依赖问题。

只要将数据成员catFriend的类型改为weak_ptr就能解决问题：

	std::weak_ptr<Cat> catFriend;
