对于软件开发领域来说，哪些大语言模型是最合适的？

## 6、5 处理乏味的任务

### 6、5、1 正则表达式

正则表达式由各种符号和命令组成，可能会让人觉得非常棘手。

1、根据需求生成正则表达式



2、解读正则表达式



AI工具能帮助开发者从枯燥乏味的编程任务中解脱出来，做更具乐趣和创造性的部分。



# 第八章 编写代码

深入挖掘函数和类的潜力；

代码重构：将杂乱无章的代码，重构为流畅、高效的代码结构；

自动生成前端界面的AI工具





想通过当前 AI 工具学习编程的人应该设定两个目标：

一是学会足够的编程知识以理解工具生成的内容；

二是扮演‘代码审核员’的角色，学会识别和改进劣质代码。

如何识别劣质代码呢？



首先，使用AI工具生成项目文件结构，

先总体使用AI工具将项目的轮廓大致构建出来，然后再深入到代码区一点点去完善每个模块和细节。



# 8、8 重构

8、8、1 忍者代码



https://baoyu.io/translations/ai/if-you-use-ai-to-teach-you-how-to-code-remember-you-still-need-to-think-for-yourself



## 8、10 面向对象编程

帮助开发者设计类的初始结构。以下是一些设计类的提示范例：

```
使用C++语言，设计一个包含员工姓名(employeeName)、员工ID(employeeID)和部门等属性的Employee类。请实现一个方法来显示员工的详细信息，并包含一个构造函数来初始化这些属性。
```

AI工具生成的代码内容如下：

```
#include <iostream>
#include <string>
using namespace std;

class Employee {
private:
    string m_employeeName;  // 员工姓名
    int m_employeeID;       // 员工ID
    string m_department;    // 部门名称

public:
    // 构造函数，用于初始化成员变量
    Employee(const string& name, int id, const string& dept)
        : m_employeeName(name), m_employeeID(id), m_department(dept) {}

    // 方法：显示员工详细信息
    void displayDetails() const {
        cout << "Employee Details:" << endl;
        cout << "Name: " << m_employeeName << endl;  // 使用 m_ 命名成员变量
        cout << "ID: " << m_employeeID << endl;
        cout << "Department: " << m_department << endl;
    }
};

int main() {
    // 创建 Employee 对象并初始化
    Employee emp("John Doe", 1001, "Human Resources");

    // 显示员工详细信息
    emp.displayDetails();

    return 0;
}
```

## 8、11 框架和库

框架和库如果更新比较频繁的话，那么当AI工具的训练数据集并没有那么新时，可能会导致一些问题。



## 8、12 生成样本数据