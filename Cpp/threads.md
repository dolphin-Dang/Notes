<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

# 多线程

## 基本概念
并发、进程、线程的基本概念
1. 并发：多个独立的活动同时进行。单核CPU只能来回切换实现“并发假象”（上下文切换方式）。
2. 进程：计算机程序关于某一个数据集合的一次运行活动。
3. 线程：每一个进程都**有且只有**一个主线程。线程可以理解为运行代码的通道、路径。
4. 并发的实现：
      1. 多进程实现并发：主要解决进程间通信的问题
      2. **单个进程、多个线程实现并发**：一个主线程（main），多个子线程实现并发。*要注意共享资源的访问冲突问题*
---
## 线程的多种创建方式
C++线程创建的过程：
   1. 包含头文件。
   2. 调用thread类去创建一个线程对象。**注意：如果创建一个线程不做处理，会调用abort()异常终止函数。**
   3. join()函数：加入，汇合线程，阻塞主线程。等待子线程执行结束才会回到主线程中。**注意：多个线程连续join互不影响，运行时间重叠而非相加。**
   4. detach()函数：打破依赖关系。**注意：一旦主程序结束，子线程将停止运行；detach后就不能join。**
   5. joinable()函数：判断当前线程是否可以join，可以返回true，反之返回false。

因此可以用：

    if(th.joinable())
        th.join();

测试程序：

    #include <iostream>
    #include <thread>
    #include <fstream>
    #include <windows.h>

    using namespace std;

    void createFile() {
        Sleep(2000);
        fstream fc("myFile.txt");
    }

    int main()
    {
        thread th(createFile);
        th.detach();
        cout << "End!";
        return 0;
    }
    //试图利用子线程创建一个txt文件
    //但是发现并没有成功创建
---
## 其它创建线程的方法

1. 普通函数方式（上面讲过）
2. 通过类和对象
3. Lambda表达式创建线程
4. 带参的方式创建线程
5. 带智能指针的方式
6. 类的成员函数

### 通过类和对象创建线程
    #include <iostream>
    #include <thread>

    using namespace std;

    class A 
    {
    public:
        //类似仿函数
        void operator()()
        {
            cout << "child thread begin ..." << endl;
        }
    };


    int main()
    {
        //ordinary way: 对象充当线程处理函数
        A a;
        thread test1(a);
        test1.join();

        //better way: 无名对象
        thread test2((A()));
        test2.join();

        cout << "main thread here!" << endl;

        return 0;
    }

解析：

    thread test2((A()));  
    //如果A()不套括号会报错
    //解释器会错误地将test2解析为一个函数

用对象(ordinary way)创造线程的时候，会复制一个临时对象进入线程，因此会调用拷贝构造函数。这个子线程结束的时候，该被复制出来的对象也会调用析构函数，即释放了临时对象。

### 通过lambda表达式创建线程

    #include <iostream>
    #include <thread>

    using namespace std;

    int main()
    {
        //lambda表达式例子
        int (*pMax)(int, int) = nullptr;
        pMax = [](int a, int b) -> int {return a > b ? a : b; };
        cout << pMax(1, 2) << endl;

        thread test1([]() {
            cout << "child thread begin ... " << endl;
            });
        test1.join();

        cout << "main thread here!" << endl;
        return 0;
    }

### 带参的方式创建线程

    #include <iostream>
    #include <thread>

    using namespace std;

    void printInfo(int& num) 
    {
        cout << "child thread " << num << endl;
        num++;
    }

    int main()
    {
        int num = 0;
        //std::ref 用于包装引用传递值
        thread test1(printInfo, std::ref(num));
        test1.join();

        cout << "main thread here! num: " << num << endl;
        return 0;  
    }

解析：

    //std::ref 用于包装引用传递值
    thread test1(printInfo, std::ref(num));

### 带智能指针的创建方式

    #include <iostream>
    #include <thread>

    using namespace std;

    void print(unique_ptr<int> ptr)
    {
        cout << "child thread " << ptr.get() << endl;
    }

    int main()
    {
        unique_ptr<int> ptr(new int(1000));
        cout << "main thread: " << ptr.get() << endl;
        thread test1(print, move(ptr));
        test1.join();

	    cout << "main thread here!" << ptr.get() << endl;

        return 0;
    }

### 用类的成员函数

    #include <iostream>
    #include <thread>

    using namespace std;

    class A {
    public:
        void print(int& num)
        {
            num = 1001;
            cout << "child thread: " << this_thread::get_id() << endl;
        }
    };

    int main()
    {
        A a;
        int num = 1007;
        //需要告诉 是哪个对象
        thread test1(&A::print, a, ref(num));
        test1.join();

        cout << "main thread here! num = " << num << endl;
        return 0;
    }
---
## 传递临时对象作为线程参数
+ 即便传入的是引用类型，实际上也是复制了一份临时对象，即值传递
+ 指针不会复制，被释放之后会导致使用了它的子线程出现问题（或者说指针本身被复制了一遍，但是指向的内存地址依然是一样的）

但是依然**不建议**在子线程函数中将引用类型和指针作为参数传入。最好用值拷贝的方式传参。并且就算是引用，thread的构造函数依然会强制拷贝构造一个新的对象/变量。

代码示例：

    #include <iostream>
    #include <thread>

    using namespace std;

    void myPrint(int i, const string& pmybuf)
    {
        cout << i << endl;
        cout << pmybuf.c_str() << endl;
    }

    int main()
    {
        int myvar = 1;
        char mybuf[] = "this is a test";
        thread mytobj(myPrint, myvar, string(mybuf));
        mytobj.detach();

        cout << "main thread here!" << endl;
        return 0;
    }

解析：
    
    thread mytobj(myPrint, myvar, string(mybuf));

此处在创建子线程mytobj之前就使用mybuf创建了一个string对象给该线程，并且线程会再一次复制这个临时对象并储存起来（似乎类似于闭包）。这样可以防止mybuf被释放之后线程再去拷贝它，以防发生未定义的操作。

总结：
+ 如果传递int这种简单类型参数，建议使用值传递，不要用引用。防止节外生枝。
+ 如果传递对象，避免隐式类型转换。全部在创建线程的时候构建出临时对象，在子线程函数参数里用**const引用**临时变量来接收，否则程序还会再用传入的临时变量拷贝构造一个临时变量，造成资源浪费。
---

## 创建多个线程
### 多个线程的创建及运行

    #include <iostream>
    #include <thread>
    #include <vector>

    using namespace std;

    void myPrint(int i)
    {
        cout << "child thread begin ... id = " << i << endl;
        cout << "child thread end ... id = " << i << endl;
    }

    int main()
    {
        vector<thread> myThreads;
        for (int i = 0; i < 10; i++) {
            myThreads.push_back(thread(myPrint, i));
        }
        for (auto iter = myThreads.begin(); iter != myThreads.end(); iter++) {
            iter->join();
        }

        cout << "main thread here!" << endl;
        return 0;
    }

运行结果显示，多个线程运行的顺序是乱的；主线程会等待所有子线程运行结束之后再开始运行。

### 多个线程的数据共享
+ 只读数据是安全稳定的。
+ 有读有写容易出错、崩溃。

### 共享数据的保护案例代码
网络游戏服务器：两个自己创建的线程
+ 一个线程收集玩家命令，并把命令写入到一个队列中。（用一个数字代表玩家发来的命令，用list存储）
+ 另一个线程从队列中取出玩家发送来的命令。解析，然后执行。
---

## 互斥量和死锁
### 互斥量(mutex)的基本概念
互斥量是一个类对象。理解成一把锁，多个线程尝试用lock()成员函数来上锁。只有一个线程可以锁成功，成功标志是lock()返回了；如果没有锁成功就停在原地等待可以上锁的时候。

互斥量使用要小心：数据保护少了没有达到效果；多了影响代码效率甚至结果正确性。

### 互斥量的用法

    #include <iostream>
    #include <thread>
    #include <list>
    #include <mutex>

    using namespace std;

    class A
    {
    public:
        void inMsgList()
        {
            for (int i = 0; i < 1000; i++) {
                cout << "inMsgList()执行，插入一个元素" << i << endl;
                myMutex.lock();
                msgList.push_back(i);
                myMutex.unlock();
            }
        }
        void outMsgList()
        {
            for (int i = 0; i < 1000; i++) {
                if (!msgList.empty()) {
                    cout << "outMsgList()执行，处理的消息为" << msgList.front() << endl;
                    myMutex.lock();
                    msgList.pop_front();
                    myMutex.unlock();
                }
                else {
                    cout << "outMsgList()执行，但是消息队列中无信息" << endl;
                }
            }
        }

    private:
        list<int> msgList;
        mutex myMutex;
    };

    int main()
    {
        A a;
        thread outMsg(&A::outMsgList, &a);
        thread inMsg(&A::inMsgList, ref(a));

        outMsg.join();
        inMsg.join();

        cout << "main thread here!" << endl;
        return 0;
    }

解析：
1. 要引入头文件
   
        #include <mutex>
2. 对于包含了mutex对象的类的成员函数，创建线程时不能直接传入相应的对象，
   
        thread wrongMsg(&A::wrongMsgList,a);
    因为传入的类对象会触发拷贝动作，而mutex是不可拷贝对象
3. 而要用以下两种方式：

        thread outMsg(&A::outMsgList, &a);
        thread inMsg(&A::inMsgList, ref(a));

为了防止用户忘记unlock()，引入了std::lock_guard的类模板，可以直接取代lock()和unlock()。意味着使用了lock_guard之后不能使用lock()和unlock()了。

只需将之前的程序稍作修改：

    if (!msgList.empty()) {
        cout << "outMsgList()执行，处理的消息为" << msgList.front() << endl;
        lock_guard<mutex> myGuard(myMutex);
        //myMutex.lock();
        msgList.pop_front();
        //myMutex.unlock();
    }

其中在lock_guard构造函数中就执行了mutex::lock()，在lock_guard析构的时候就调用了mutex::unlock()。

**注意**：
+ 要确定myGuard的作用域，超出作用域时才会unlock()。
+ 如果将lock_guard放在一个for循环的第一行，由于很快就要开始下一次循环，很有可能会导致整个for循环一起被执行完才开始执行别的线程。
+ 可以灵活利用大括号{}限定lock_guard的生存周期。

### 死锁
两个线程1、2；两个互斥量A、B。
当线程1锁住了A，正在等待锁B，且线程2锁住了B，正在等待锁A时，两个线程就成为了死锁。

在下面的代码中就有可能出现死锁：

    class A
    {
    public:
        void inMsgList()
        {
            for (int i = 0; i < 10000; i++) {
                cout << "inMsgList()执行，插入一个元素" << i << endl;
                myMutex1.lock();
                myMutex2.lock();
                msgList.push_back(i);
                myMutex2.unlock();
                myMutex1.unlock();
            }
        }
        void outMsgList()
        {
            for (int i = 0; i < 10000; i++) {
                if (!msgList.empty()) {
                    cout << "outMsgList()执行，处理的消息为" << msgList.front() << endl;
                    myMutex2.lock();
                    myMutex1.lock();
                    msgList.pop_front();
                    myMutex1.unlock();
                    myMutex2.unlock();
                }
                else {
                    cout << "outMsgList()执行，但是消息队列中无信息" << endl;
                }
            }
        }

    private:
        list<int> msgList;
        mutex myMutex1;
        mutex myMutex2;
    };

死锁的一般解决方案：
+ 保证两个mutex的上锁顺序一致
+ std::lock()函数模板：

        std::lock(myMutex1, myMutex2);
        msgList.push_back(i);
        myMutex2.unlock();
        myMutex1.unlock();

    可以传入至少两个mutex互斥量，lock()会尝试锁住所有mutex。但是只要有一个没有成功锁住，它会将所有的全部unlock，等待可以锁住的时候。

    **注意**：需要手动unlock()

但是在一般的项目实践中，很少出现两个互斥量同时上锁的现象。基本上在这之间都会有其它代码段。

---

## unique_lock详解
### unique_lock取代lock_guard
+ unique_lock是一个类模板，工作中一般使用lock_guard。
+ unique_lock比lock_guard灵活，但是效率上差一些。
+ 如果只带第一个参数mutex对象，unique_lock和lock_guard效果完全一样。

### unique_lock的第二个参数
lock_guard可以带第二个参数，unique_lock支持lock_guard支持的所有第二个参数（标记），同时支持更多。unique_lock会将自己与mutex对象*绑定*。

+ std::adopt_lock: 表示这个互斥量已经被lock了。
+ std::try_to_lock: 尝试用mutex的lock()去锁定，但是如果没有锁定成功就会立即返回，不会阻塞。
+ std::defer_lock: 并没有给mutex加锁，但是将mutex与unique_lock对象绑定。手动lock()，但是不用自己unlock()。

        std::unique_lock<std::mutex> myLock(myMutex, std::defer_lock);
        myLock.lock();

    

### unique_lock的成员函数
+ lock()，配合std::defer_lock使用或者unlock()使用，灵活加锁解锁。
+ unlock()，可以手动解锁。
+ try_lock()，尝试给互斥量加锁。成功放回true，失败返回false，不会造成阻塞。
        
        if(myLock.try_lock){
            //cope with the data
        }else{
            //上锁失败...
        }
+ release()，返回它所管理的mutex对象指针并且释放所有权，也就是说这个unique_lock和mutex不再有关系。因此就需要在后续代码中自行解锁。

        std::unique_lock<mutex> myLock(myMutex);
        std::mutex *ptr = myLock.release();

效率问题：lock住的代码越少，执行越快，整个程序运行效率越高。

### unique_lock所有权的传递
所有权概念：

    std::unique_lock<mutex> myLock(myMutex);

这里myLock拥有myMutex的所有权。myLock可以把自己对myMutex的所有权转移给其它unique_lock对象。

因此unique_lock对象对mutex对象的所有权是属于可以转移但是不能复制的。类似于智能指针unique_ptr。

转移方式：

    unique_lock<mutex> myLock1(myMutex);
    unique_lock<mutex> myLock1(std::move(myMutex));

[移动语义](http://c.biancheng.net/view/7863.html)，相当于myLock2和myMutex绑定到了一起。现在myLock1指向myMutex，而myLock1指向空。

也可以用函数返回一个临时的unique_lock对象tmpLock：

    std::unique_lock<mutex> rtnTmpLock()
    {
        std::unique_lock<mutex> tmpLock(myMutex);
        return tmpLock;
    }

此处从函数返回一个对象是可以的。返回这种局部对象tmpLock会导致系统生成临时unique_lock对象，并调用unique_lock的[移动构造函数](http://c.biancheng.net/view/7847.html)。

---
## 设计模式
设计模式：代码的一些写法。程序灵活，维护方便，难以阅读。

### 单例设计模式
整个项目中，有某个或者某些特殊的类。属于该类的对象最多只能创建一个。

单例类MyCAS：

    class MyCAS {
    private:
        MyCAS(){}
        static MyCAS* Instance;

    public:
        static MyCAS* GetInstance() {
            if (Instance == NULL) {
                Instance = new MyCAS();
                static Clear cl;
            }
            return Instance;
        }

        class Clear {
        public:
            ~Clear() {
                if (MyCAS::Instance) {
                    delete MyCAS::Instance;
                    MyCAS::Instance = NULL;
                }
            }
        };

        void func() {
            cout << "test" << endl;
        }
    };
    MyCAS* MyCAS::Instance = NULL;

解析：
+ MyCAS对象只能通过GetInstance的方法来返回一个指针，且如果已经创造了指针，就只会返回原来那个，不会再次新建。
+ MyCAS构造函数为private，这样可以防止被通过其他方式构造对象。
+ 不能忘记最后的
  
        MyCAS* MyCAS::Instance = NULL;
+ 类中类Clear是为了在被析构的时候自动回收new出来的空间。Instance是一个指针，指针只有在delete的时候才会调用析构函数，因此在析构函数里面delete是不可行的。
+ 单例对象也是可以用静态对象创建的。唯一的区别是用静态指针是在第一次用到的时候构造；用静态变量是在程序一开始就构造。

### 单例设计模式共享数据问题
要在主线程中，创造所有子线程之前就调用单例类的GetInstance方法初始化一个单例对象，并调用所有必要的函数进行数据、文件的装载准备工作。

面临的问题：多个线程中需要调用GetInstance方法，同时new会出错。因此需要互斥。

在类外定义一个mutex对象myMutex，然后将GetInstance方法修改为：

	static MyCAS* GetInstance() {
		unique_lock<mutex> myLock(myMutex);
		if (Instance == NULL) {
			Instance = new MyCAS();
			static Clear cl;
		}
		return Instance;
	}
这样可以有效地防止同时new。但是同时大大降低了效率，因为实际上我们只需要防止第一次运行的时候两个线程同时new，而对于单例对象，之后每一次调用GetInstance方法都只是返回指针而已。故而可以将函数改为：

	static MyCAS* GetInstance() {
		if (Instance == NULL) {
			unique_lock<mutex> myLock(myMutex);
			if (Instance == NULL) {
				Instance = new MyCAS();
				static Clear cl;
			}
		}
		return Instance;
	}
通过double-check可以提高效率。

此处应该使用[volatile](https://www.runoob.com/w3cnote/c-volatile-keyword.html)关键字来修饰静态对象，以防止[内存reorder](https://blog.csdn.net/weixin_43239416/article/details/107441166)情况。如果是静态指针则没有问题，因为：

>reorder情况，在new时，通常先构造，然后再赋值给Instance变量。然而出现reorder情况先给Instance赋值，然后再执行new构造一个单例对象。如果后面线程进来得到Instance指针并非nullptr，直接返回Instance。但是Instance没有构造生成。

>有些变量是用 volatile 关键字声明的。当两个线程都要用到某一个变量且该变量的值会被改变时，应该用 volatile 声明，该关键字的作用是防止优化编译器把变量从内存装入 CPU 寄存器中。如果变量被装入寄存器，那么两个线程有可能一个使用内存中的变量，一个使用寄存器中的变量，这会造成程序的错误执行。volatile 的意思是让编译器每次操作该变量时一定要从内存中真正取出，而不是使用已经存在寄存器中的值。

### 函数模板std::call_once()
C++11引入的函数，第二个参数是一个函数名a()。call_once()的作用是保证函数a()只会被调用一次。因此可以用来解决单例对象初始化问题。效率上也比互斥量消耗的资源更少。

call_once()需要与一个标记std::once_flag（其实是一个结构）结合使用。call_once()通过这个标记来决定对应的函数a()是否执行。

    std::mutex myMutex;
    std::once_flag gFlag;

    class MyCAS {
        static void CreateInstance() {
            Instance = new MyCAS();
            static Clear cl;
        }
    private:
        MyCAS(){}
        static MyCAS* Instance;
    public:
        static MyCAS* GetInstance() {
            call_once(gFlag, CreateInstance);
            return Instance;
        }

        //...
    };
    MyCAS* MyCAS::Instance = NULL;


