---
title: C++多线程获取返回值的方法
date: 2021-08-27 19:53:54
tags:
---

在许多时候，我们会有这样的需求——即我们想要得到线程返回的值。但是在C++11 多线程中我们注意到，std::thread对象会忽略顶层函数的返回值。

那问题来了，我们要怎么获得线程的返回值呢？

我们通过一个例子来说明如何实现这个需求。用多个线程计算（a+b）/ (x+y) 的值

有两种方法，分别是

# **1. 传统的方法：在线程间共享指针**

```
#include#include#include#includeusing namespace std;
 
void func2(int x, int y,int* ans) {
  *ans= x + y;
}
 
int main()
{
  //计算（a+b）/(x+y)
  //用三个线程，一个线程计算a+b，另一个线程计算x+y
  int a, b, x, y;
  a = 10, b = 8, x = 2, y = 4;
 
  int* sum1 = new int(0);
  int* sum2 = new int(0);
  thread t1(func2, a, b, sum1);
  t1.join();
  thread t2(func2, x, y, sum2);
  t2.join();
   
  cout << (*sum1) / (*sum2) << endl;
 
  delete sum1;
  delete sum2;
   
  system("pause");
  return 0;
}
```

# **2. C++11的方法：使用std::future和std::promise**

std::future和std::promise是封装好的两个类模板，这两个类需要配合使用，他们的头文件是#include

std::future，它表示存储着一个未来会被初始化的变量。这个变量可以通过std::future提供的成员函数std::future::get()来得到。如果在这个变量被赋值之前就有别的线程试图通过std::future::get()获取这个变量，那么这个线程将会被阻塞到这个变量可以获取为止。

std::promise同样也是一个类模板，这个对象承诺在未来一定会初始化一个变量(这个变量也就是std::future中的变量)。

每一个std::promise对象都有一个与之关联的std::future对象。当std::promise设置值的时候，这个值就会赋给std::future中的对象了。

```
#include#include#include#include#include//std::future std::promise
#include//std::ref模板传参的时候使用
 
void func2(int x, int y,std::promise&promiseObj) {
  promiseObj.set_value(x+y);
}
 
int main()
{
  //计算（a+b）/(x+y)
  //用三个线程，一个线程计算a+b，另一个线程计算x+y
  int a, b, x, y;
  a = 10, b = 8, x = 2, y = 4;
 
  int sum1, sum2;
  //声明一个类
  std::promisepromiseObj;
  //将future和promise关联
  std::futurefutureObj = promiseObj.get_future();
  //模板传参的时候使用ref，否则传参失败
  std::thread t1(func2, a, b, ref(promiseObj));
  t1.join();
  //获取值
  sum1 = futureObj.get();
  std::cout << "sum1=" << sum1 << std::endl;
 
  //不能直接复用上面的future和promise
  std::promisepromiseObj2;
  std::futurefutureObj2 = promiseObj2.get_future();
 
  std::thread t2(func2, x, y, ref(promiseObj2));
  t2.join();
  sum2 = futureObj2.get();
  std::cout << "sum2=" << sum2 << std::endl;
 
  std::cout << "sum1/sum2=" << sum1 / sum2 << std::endl;
   
  std::system("pause");
  return 0;
}
```

-EOF-

---

转载自：[ITPUB](http://blog.itpub.net/69955379/viewspace-2702825/)

作者：[大雄45](http://blog.itpub.net/69955379/)
