# 说说 super 关键字的用法?⁠​

## 一句话说明（白话）


## 它解决什么问题 / 为什么重要


## 核心原理（一步步讲清楚）


##典型使用场景


## 简单例子 /伪代码


## 常见坑与误区


##题库要点（原始材料）
`super`关键字在 Java 中是一个指向**当前对象的直接父类**的引用。
它主要用于在子类中访问和调用那些被子类覆盖或隐藏了的父类成员（变量、方法和构造方法）。
其主要用法有以下三种：
1. **调用父类的构造方法**：在子类的构造方法中，必须首先调用父类的构造方法以确保父类部分被正确初始化。这是通过 `super()`或 `super(参数列表)`实现的，并且**这条语句必须位于子类构造方法的第一行**。如果子类构造方法中没有显式写出，编译器会自动加上一个 `super()`调用父类的无参构造方法。
```java
    class Animal {
        public Animal(String type) {
            System.out.println("Animal constructor: " + type);
        }
    }
    class Dog extends Animal {
        public Dog(String name) {
            super("Dog");// 调用父类的有参构造方法，必须放在第一行
            System.out.println("Dog name: " + name);
        }
    }
```
2. **调用父类中被重写的方法**：当子类重写了父类的方法后，如果需要在子类方法中再次使用父类该方法的原始实现，可以使用 `super.方法名()`进行调用。
```java
    class Parent {
        public void display() {
            System.out.println("Parent's display method");
        }
    }
    class Child extends Parent {
        @Override
        public void display() {
            super.display();// 先调用父类的display方法
            System.out.println("Child's display method");
        }
    }
```
3. **访问父类中被隐藏的成员变量**：如果子类中声明的成员变量与父类中的成员变量同名，则父类的变量会被"隐藏"。此时，可以使用 `super.变量名`来访问父类中的那个变量。
``` java
	class Parent {
	    String name = "Parent";
	}
	class Child extends Parent {
	    String name = "Child";
	    void printNames() {
	        System.out.println(super.name);// 输出 "Parent"
	        System.out.println(this.name);// 输出 "Child"
	    }
	}
```

##关联知识
- 

## 延伸阅读（后续补充）
- 
