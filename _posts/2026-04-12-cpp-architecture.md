---
title: "从 Python 到现代 C++：一次 72 小时的架构思维突击与重塑"
date: 2026-04-12
permalink: /posts/2026/04/cpp-architecture/
tags:
  - C++
  - 架构设计
  - 学习笔记
  - 面试复盘
---

## **前言：带着敬畏心踏入 C++ 的世界**

作为一个习惯了 Python 灵活抽象（“鸭子类型”、自带垃圾回收）以及 C 语言底层直接（指针与内存操作）的开发者，我一直对现代 C++ 抱有敬畏之心。C++ 陡峭的学习曲线不仅在于其繁复的语法，更在于它要求开发者同时具备“系统级内存控制”与“高级面向对象抽象”的能力。

这篇博客记录了我近期的一段极限学习旅程：**在 72 小时内，通过解构一道“技术难题”的车辆调度系统架构面试题，系统性地学习现代 C++ (C++11/14/17) 的核心特性。**

我深知 72 小时远不足以精通 C++，这篇笔记更多是我作为一个 C++ 学习者的“认知升级记录”。希望能为同样处于语言转型期的朋友提供一些参考，也借此向各位前辈请教。

## **面试题背景：严格约束下的系统解耦**

一切学习始于一个具体的挑战：

**需求**：在车辆管理系统中，需要支持不同车辆之间的动作调度（如汽车 Car 触发自行车 Bike 响铃，未来还要扩展卡车 Truck 闪灯）。

**严格约束条件**：

1. 必须使用工厂模式。  
2. Car 绝对不能包含 Bike 的头文件（物理隔离）。  
3. 必须引入通用模板类实现动作分发。  
4. 底层类对上层模板类一无所知（依赖倒置）。  
5. 严禁使用任何静态变量或全局事件总线。

面对这个需求，我最初的本能是用 Python 的动态特性或者 C 的函数指针数组来暴力解决。但要在现代 C++ 的强类型和严格作用域下完成，我必须进行思维重塑。我的学习路径由此展开。

## **Day 1：跨越内存管理的深渊**

### **1\. 从 struct 到 class：理解封装的本质**

第一步是理解 C++ 的 class 与 C 语言 struct 的本质区别——**权限边界（封装）**。

在学习过程中，我构建了一个“银行金库”的心理模型：

* **private**：是绝对安全的保险箱（如车辆的 fuel\_ 油量数据），外部代码被编译器严禁直接访问，杜绝了脏数据的产生。  
* **public**：是受控的对外接口（如 refuel() 加油方法）。  
* **对比 Python**：不再依赖程序员的自觉（如 \_name 约定），而是依靠编译器的铁腕强权。

\#include \<iostream\>  
\#include \<string\>

using namespace std; // 为了方便，我们先开启 std 偷懒模式

class Vehicle {  
private:   
    string name\_;   
    int speed\_;  
    // 动作 1：新增一个私有属性，用来存油量（外部绝对不能直接改这个值，防止有人凭空变出油来）  
    int fuel\_; 

public:   
    // 动作 2：修改构造函数。既然图纸升级了，造车的时候就必须给一个初始油量  
    Vehicle(string name, int speed, int fuel) {  
        name\_ \= name;      
        speed\_ \= speed;    
        fuel\_ \= fuel;    // 将传进来的初始油量存入私有保险箱  
        cout \<\< "\[系统\] 创造了一辆 " \<\< name\_ \<\< "，速度: " \<\< speed\_   
             \<\< "km/h，初始油量: " \<\< fuel\_ \<\< "L" \<\< endl;  
    }

    \~Vehicle() {  
        cout \<\< "\[系统\] " \<\< name\_ \<\< " 被报废销毁了！" \<\< endl;  
    }

    void show() {  
        cout \<\< "\>\>\> 正在驾驶: " \<\< name\_ \<\< " (速度: " \<\< speed\_   
             \<\< "km/h，剩余油量: " \<\< fuel\_ \<\< "L)" \<\< endl;  
    }

    // 动作 3：新增加油方法。这是外部唯一能改变油量的合法途径  
    void refuel(int amount) {  
        fuel\_ \= fuel\_ \+ amount; // 把加的油累加到总油量里 (也可以简写为 fuel\_ \+= amount;)  
        cout \<\< "\>\>\> ⛽ 成功给 " \<\< name\_ \<\< " 加油 " \<\< amount   
             \<\< "L！当前总油量: " \<\< fuel\_ \<\< "L" \<\< endl;  
    }  
};

int main() {  
    cout \<\< "--- 程序开始 \---" \<\< endl;

    {  
        // 动作 4：造车的时候，记得传入第三个参数（初始油量，比如这里给 50L）  
        Vehicle myCar("保时捷", 120, 50);   
          
        myCar.show(); // 看看初始状态  
          
        // 调用我们刚才写的加油方法，加 30L 油  
        myCar.refuel(30);   
          
        myCar.show(); // 再次查看状态，验证油量是否真的变多了  
          
    } // myCar 在这里被销毁

    cout \<\< "--- 程序结束 \---" \<\< endl;  
    return 0;   
}

### **2\. 拥抱 RAII 与智能指针：告别野指针与内存泄漏**

这是我受震撼最大的一课。曾经写 C 语言时被 malloc/free 折磨的恐惧，在现代 C++ 中被彻底治愈。

我学习了 C++ 独有的 **RAII（资源获取即初始化）** 机制，并深入理解了 std::shared\_ptr。

* **大括号的魔法**：C++ 严格基于作用域 {} 管理对象生命周期。对象一旦离开作用域，必然触发析构函数 \~ClassName()。  
* **引用计数 (Reference Counting)**：智能指针就像一把带有芯片的车钥匙。当所有的钥匙（指针）都在作用域结束时被熔毁，引用计数归零，系统就会自动且精准地引爆并回收真正的车辆内存。

*学习感悟：现代 C++ 不是让你不用指针，而是让你用“有教养的指针”。*

\#include \<iostream\>  
\#include \<string\>  
\#include \<memory\> // 必须引入这个头文件才能使用智能指针

// 注意：这里去掉了 using namespace std;

class Vehicle {  
private:  
    std::string name\_; // 加上 std::  
public:  
    Vehicle(std::string name) {  
        name\_ \= name;  
        std::cout \<\< "\[系统\] 🛠️ 在堆内存中造出了一辆 " \<\< name\_ \<\< std::endl; // 加上 std::  
    }  
    \~Vehicle() {  
        // 当没有任何指针指向这个对象时，它会自动触发！  
        std::cout \<\< "\[系统\] 💥 " \<\< name\_ \<\< " 被彻底销毁，内存已自动回收！" \<\< std::endl;  
    }  
    void honk() {  
        std::cout \<\< "\>\>\> " \<\< name\_ \<\< " 发出喇叭声：滴滴！" \<\< std::endl;  
    }  
};

int main() {  
    std::cout \<\< "--- 任务开始 \---" \<\< std::endl;

    // 1\. 创建一个空的智能指针 ptr1  
    std::shared\_ptr\<Vehicle\> ptr1 \= nullptr; 

    std::cout \<\< "\\n\>\>\> \[第一阶段：进入局部隔离区\]" \<\< std::endl;  
    { // 大括号开始  
          
        // 2\. std::make\_shared 是现代 C++ 官方推荐的创建对象的方式   
        std::shared\_ptr\<Vehicle\> ptr2 \= std::make\_shared\<Vehicle\>("装甲车");  
          
        // use\_count() 可以查看当前有几个人在盯着这辆车  
        std::cout \<\< "   当前装甲车的引用计数: " \<\< ptr2.use\_count() \<\< std::endl;

        // 3\. 像普通指针一样使用它（注意：指针调用方法用箭头 \-\>）  
        ptr2-\>honk(); 

        // 4\. 将 ptr2 赋值给外面的 ptr1。此时两个指针同时盯着“装甲车”  
        ptr1 \= ptr2;  
        std::cout \<\< "   把车钥匙也给了外面的 ptr1 后，引用计数: " \<\< ptr2.use\_count() \<\< std::endl;

    } // ⚠️ 第一阶段大括号结束！局部的 ptr2 在这里死掉了。  
      
    std::cout \<\< "\\n\>\>\> \[第二阶段：离开局部隔离区\]" \<\< std::endl;  
      
    // 思考：ptr2 死了，装甲车会死吗？  
    std::cout \<\< "   局部的 ptr2 已死，但装甲车的引用计数变为: " \<\< ptr1.use\_count() \<\< std::endl;  
      
    ptr1-\>honk(); // 依然可以安全调用，绝对不会遇到 C 语言的野指针崩溃！

    std::cout \<\< "\\n\>\>\> \[程序准备结束...\]" \<\< std::endl;  
    // 马上要 return 0 了，main 函数结束，外面的 ptr1 也要死了。  
      
    return 0;  
}

## **Day 2：架构设计的艺术**

有了安全的内存基石，我开始挑战题目要求的“无耦合调度”。这要求我打破强类型语言的壁垒。

### **1\. 多态 (Polymorphism) 与接口抽象**

如何在 C++ 的 std::vector 中同时存放 Car 和 Bike？答案是继承同一个基类（接口），并使用虚函数。

* **virtual 的动态绑定**：这是打破 C++ “静态刻板”的钥匙。它告诉编译器在运行时通过对象背后的“虚表（vtable）”来决定调用哪个具体类的函数，而不是在编译时写死。  
* **虚析构函数的防坑指南**：我深刻记住了，包含虚函数的基类，其析构函数也必须是 virtual 的，否则在使用多态销毁对象时，子类的专属内存将会泄漏。

\#include \<iostream\>  
\#include \<string\>  
\#include \<vector\>  
\#include \<memory\>

// 1\. 基类（父类）：定义通用图纸  
class Vehicle {  
public:  
    // ⚠️ 核心魔法：virtual（虚函数）  
    // 加上 virtual，意味着告诉编译器：“不要现在写死调用哪个函数，等程序跑起来后，看它到底是什么车，再去调对应的声音！”  
    virtual void honk() {  
        std::cout \<\< "\[通用车辆\] 发出某种声音..." \<\< std::endl;  
    }

    // ⚠️ 面试必考点：只要类里有 virtual 函数，析构函数就必须是 virtual 的！  
    // 否则用智能指针销毁父类指针时，子类的内存会泄露。  
    virtual \~Vehicle() {  
        std::cout \<\< "\[系统\] 基础 Vehicle 被销毁" \<\< std::endl;  
    }  
};

// 2\. 派生类（子类）：汽车，继承自 Vehicle  
class Car : public Vehicle {  
public:  
    // override 关键字明确告诉编译器：我在这里重写了父类的虚函数！  
    void honk() override {  
        std::cout \<\< "\>\>\> 🚗 汽车鸣笛：滴滴滴！" \<\< std::endl;  
    }  
      
    \~Car() override {  
        std::cout \<\< "\[系统\] 🚗 Car 的专属零件被销毁" \<\< std::endl;  
    }  
};

// 3\. 派生类（子类）：自行车，继承自 Vehicle  
class Bike : public Vehicle {  
public:  
    void honk() override {  
        std::cout \<\< "\>\>\> 🚲 自行车响铃：叮铃铃！" \<\< std::endl;  
    }  
      
    \~Bike() override {  
        std::cout \<\< "\[系统\] 🚲 Bike 的专属零件被销毁" \<\< std::endl;  
    }  
};

int main() {  
    std::cout \<\< "--- 多态测试开始 \---" \<\< std::endl;

    // 🎯 见证奇迹的时刻：  
    // 我们建一个车库（vector 数组）。  
    // 这个车库规定只能停“指向 Vehicle 的智能指针”。  
    std::vector\<std::shared\_ptr\<Vehicle\>\> garage;

    // 但是！因为 Car 和 Bike 继承了 Vehicle，所以我们可以把它们塞进去！这叫“向上转型”  
    garage.push\_back(std::make\_shared\<Car\>());  
    garage.push\_back(std::make\_shared\<Bike\>());

    std::cout \<\< "\\n--- 统一调度开始 \---" \<\< std::endl;  
      
    // 遍历车库里的每一辆车  
    for (const std::shared\_ptr\<Vehicle\>& v : garage) {  
        // 编译器在这里只知道 v 是一辆 Vehicle，但因为 honk 是 virtual 的  
        // 它会聪明的“动态绑定”，汽车发汽车的声音，自行车发自行车的声音！  
        v-\>honk();   
    }

    std::cout \<\< "\\n--- 程序结束，准备清理内存 \---" \<\< std::endl;  
    return 0;  
}

### **2\. 泛型编程 (Templates) 与命令模式 (Command Pattern)**

为了满足“Car 不认识 Bike，却能调用 Bike 动作”的变态约束，我学习了如何使用 **接口 (ICommand)** 切断物理依赖，并利用 **模板类 (template \<typename T\>)** 实现**类型擦除 (Type Erasure)**。

我构建了一个“万能动作适配器”：

template \<typename Receiver\>    
class VehicleActionCommand : public ICommand {    
private:    
    std::shared\_ptr\<Receiver\> receiver\_;    
    using ActionFunc \= void (Receiver::\*)();     
    ActionFunc action\_;

public:    
    VehicleActionCommand(std::shared\_ptr\<Receiver\> receiver, ActionFunc action)    
        : receiver\_(std::move(receiver)), action\_(action) {}

    void execute() override {    
        if (receiver\_ && action\_) {    
            (receiver\_.get()-\>\*action\_)(); // 执行具体车辆的具体动作    
        }    
    }    
};

通过这种方式，底层 Car 的方向盘上只安装了一个无标签的 ICommand 按钮，而上层的装配层（main 函数）则通过依赖注入（DI）将具体的动作粘合进去。

## **最终落地：一份令自己满意的答卷**

通过两天的重塑，我最终完成了这套零全局变量、零直接耦合、支持运行时热插拔的现代 C++ 架构方案。

在这套方案中：

1. **接口层**：定义了 ICommand，充当法律契约。  
2. **实体层**：Car, Bike, Truck 各司其职，彼此代码文件完全隔离。  
3. **上层应用层**：利用模板将具体动作包装为抽象接口，实现类型擦除。  
4. **装配层**：在 main 函数中作为 IoC 容器，完成对象创建与依赖注入。

\#include \<iostream\>  
\#include \<memory\>  
\#include \<string\>

// \==========================================  
// \[Layer 0\] 接口层：定义核心抽象，切断物理耦合  
// \==========================================

// 通用动作调用接口 (解开 Car 和 Bike 的强耦合)  
class ICommand {  
public:  
    virtual \~ICommand() \= default;  
    virtual void execute() \= 0;  
};

// 车辆基础接口  
class IVehicle {  
public:  
    virtual \~IVehicle() \= default;  
    virtual void identify() const \= 0;  
};

// \==========================================  
// \[Layer 1\] 实体层：底层业务对象，彼此完全隔离  
// \==========================================

class Bike : public IVehicle {  
public:  
    void identify() const override { std::cout \<\< "\[Bike\] 是一辆自行车。" \<\< std::endl; }

    // Bike 的专属动作  
    void ringBell() {  
        std::cout \<\< "\>\>\> 🚲 自行车动作：叮铃铃！" \<\< std::endl;  
    }  
};

class Truck : public IVehicle {  
public:  
    void identify() const override { std::cout \<\< "\[Truck\] 是一辆卡车。" \<\< std::endl; }

    // Truck 的专属动作  
    void flashLights() {  
        std::cout \<\< "\>\>\> 🚚 卡车动作：闪烁大灯！" \<\< std::endl;  
    }  
};

class Car : public IVehicle {  
private:  
    // 核心考点：Car 只依赖 ICommand 抽象接口，对 Bike/Truck 一无所知  
    std::shared\_ptr\<ICommand\> triggerAction\_;

public:  
    void identify() const override { std::cout \<\< "\[Car\] 是一辆小汽车。" \<\< std::endl; }

    // 依赖注入：通过参数将具体的动作逻辑注入进来，而不是在内部 new  
    void setTriggerAction(std::shared\_ptr\<ICommand\> action) {  
        triggerAction\_ \= std::move(action);  
    }

    // Car 自身的触发逻辑  
    void honkAndTrigger() {  
        std::cout \<\< "\[Car\] 🚗 汽车按下了方向盘的综合调度按钮..." \<\< std::endl;  
        if (triggerAction\_) {  
            triggerAction\_-\>execute(); // 动态绑定，多态调用  
        }  
        else {  
            std::cout \<\< "\[Car\] 没有绑定任何调度动作。" \<\< std::endl;  
        }  
    }  
};

// \==========================================  
// \[Layer 1.5\] 工厂层：负责对象的创建  
// \==========================================

// 严格遵守约束 1 和 5：使用普通类和方法，杜绝全局变量和静态 (static) 方法  
class VehicleFactory {  
public:  
    std::shared\_ptr\<IVehicle\> createVehicle(const std::string& type) const {  
        if (type \== "Car") return std::make\_shared\<Car\>();  
        if (type \== "Bike") return std::make\_shared\<Bike\>();  
        if (type \== "Truck") return std::make\_shared\<Truck\>();  
        return nullptr;  
    }  
};

// \==========================================  
// \[Layer 2\] 上层应用层：泛型动作适配器  
// \==========================================

// 核心考点：利用模板将具体的类和成员函数“包装”成通用的 ICommand 接口  
template \<typename Receiver\>  
class VehicleActionCommand : public ICommand {  
private:  
    std::shared\_ptr\<Receiver\> receiver\_;  
    using ActionFunc \= void (Receiver::\*)(); // C++ 成员函数指针  
    ActionFunc action\_;

public:  
    VehicleActionCommand(std::shared\_ptr\<Receiver\> receiver, ActionFunc action)  
        : receiver\_(std::move(receiver)), action\_(action) {  
    }

    void execute() override {  
        if (receiver\_ && action\_) {  
            // 通过指针执行具体车辆的具体动作  
            (receiver\_.get()-\>\*action\_)();  
        }  
    }  
};

// \==========================================  
// \[Layer 3\] 装配层 (Main)：IoC 控制反转中心  
// \==========================================

int main() {  
    std::cout \<\< "=== 系统初始化与装配 \===" \<\< std::endl;

    // 1\. 创建非静态工厂实例  
    VehicleFactory factory;

    // 2\. 通过工厂创建车辆 (这里用 dynamic\_pointer\_cast 向下转型以便绑定专属动作)  
    std::shared\_ptr\<Car\> car \= std::dynamic\_pointer\_cast\<Car\>(factory.createVehicle("Car"));  
    std::shared\_ptr\<Bike\> bike \= std::dynamic\_pointer\_cast\<Bike\>(factory.createVehicle("Bike"));  
    std::shared\_ptr\<Truck\> truck \= std::dynamic\_pointer\_cast\<Truck\>(factory.createVehicle("Truck"));

    // 3\. 需求测试 1：Car 触发 Bike 的响铃  
    // 在上层将 Bike 和 ringBell 动作打包成一个 Command，并注入给 Car  
    std::shared\_ptr\<ICommand\> bikeCmd \= std::make\_shared\<VehicleActionCommand\<Bike\>\>(bike, \&Bike::ringBell);  
    car-\>setTriggerAction(bikeCmd);

    std::cout \<\< "\\n--- 场景 1：Car 调度 Bike \---" \<\< std::endl;  
    car-\>honkAndTrigger();

    // 4\. 需求测试 2 (扩展性)：Car 触发 Truck 的闪灯  
    // 验证架构威力：Car 类的代码不需要修改任何一行，即可调度全新的车辆动作！  
    std::shared\_ptr\<ICommand\> truckCmd \= std::make\_shared\<VehicleActionCommand\<Truck\>\>(truck, \&Truck::flashLights);  
    car-\>setTriggerAction(truckCmd);

    std::cout \<\< "\\n--- 场景 2：Car 调度 Truck \---" \<\< std::endl;  
    car-\>honkAndTrigger();

    std::cout \<\< "\\n=== 程序结束，智能指针开始自动回收内存 \===" \<\< std::endl;  
    return 0;  
}

## **写在最后：学无止境**

这两天的突击，让我深刻体会到 C++ 之父 Bjarne Stroustrup 的设计哲学：**“你不必为你不使用的东西付出代价，而你使用的东西，你将得到最好的性能。”**

从 Python 的快速开发，到 C 的底层控制，再到 C++ 的宏大架构，每一种语言都有其不可替代的魅力。我深知自己目前只是站在了 C++ 宏大世界的海岸线上，诸如右值引用、完美转发、多线程并发内存模型等深水区还在前方等待着我。

保持谦卑，持续构建。感谢这段 72 小时的旅程，期待在未来的工程实践中，写出更加健壮、优雅的 C++ 代码。

*(本文同步发布于我的个人博客 [ljlearning.xyz](https://ljlearning.xyz)，欢迎探讨交流。)*
