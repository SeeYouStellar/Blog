## 1. 使用接口作为返回类型，而不是用具体的实现接口的类
接口和实现接口的类：

![alt text](image-2.png)
![alt text](image-3.png)

使用接口作为返回类型的函数：
![alt text](image.png)

controller类的成员是接口类型
![alt text](image-4.png)
![alt text](image-6.png)
![alt text](image-5.png)

实际传递的都是类，但是在函数实现时都用接口，发现这种现象很普遍

可以思考下接口的这种使用方法的好处

