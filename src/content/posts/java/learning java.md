---
title: My Java security learning journey - part 1 - Java basic and java reflection
published: 2025-11-23
description: ''
image: ''
tags: [java]
category: 'java'
draft: true 
lang: ''
---

# Introduction
Đây là note của mình trong quá trình học Java security - một thứ theo mình là khá rộng lớn do có quá nhiều thứ trong Java ecosystem. Oke let go!

# Java OOP
Đã nhắc tới java thì ta phải nhắc tới OOP, khi viết code Java không ai mà không viết theo kiểu OOP cả. So với các ngôn ngữ dynamic type như Python hay Javascript thì các ngôn ngữ này thường ít dùng tới class hay kế thừa, hầu hết viết hàm là chính. Nếu bạn đã quen audit code python, javascript rồi thì khả năng cao lúc audit code Java sẽ bị ngợp do quá nhiều thứ liên quan đến OOP cũng như sự abstraction của các class khá là nhiều. Thì dưới đây mình sẽ tóm gọn một chút syntax OOP của java.

Ref:
1. https://www.w3schools.com/java

## Class, object bình thường
```java
public class Main {
  int x = 5;

  public Main() {
    x = 5;  // Set the initial value for the class attribute x
  }

  public void speed(int maxSpeed) {
    System.out.println("Max speed is: " + maxSpeed);
  }

  public static void main(String[] args) {
    Main myObj = new Main();
    System.out.println(myObj.x);
    Main.speed();
  }
}
```

## Modifier
Đối với `class`
- `public class Student{}`: class này có thể được access bởi tất cả class
- `class Student{}`: class này chỉ được access bởi class trong cùng một package
- `final`: class này không thể được kế thừa bởi class con nào

Đối với `attribute`,`method`,`constructor` thì:
- `public`: được access bởi mọi class
- `private`: chỉ được access bởi chính class đó
- `Không có modifier`: chỉ được access bởi class trong cùng một package
- `protected`: chỉ được access bởi class ở trong cùng một package hoặc class con
- `final`: giống constant, không thể thay đổi
- `static`: attribute hay method thuộc sở hữu của class. Mọi object đều sở hữu `static` attribute và có thể gọi `static` method mà không cần tạo object.
- `transient`: bị skip trong quá trình serialization
- `synchronized`: method chỉ được access bởi một luồng tại một thời điểm
- `volatile`: The value of an attribute is not cached thread-locally, and is always read from the "main memory"

## Kế thừa
```java
class Vehicle {
  protected String brand = "Ford";        // Vehicle attribute
  public void honk() {                    // Vehicle method
    System.out.println("Tuut, tuut!");
  }
}

class Car extends Vehicle {
  private String modelName = "Mustang";    // Car attribute
  public static void main(String[] args) {

    // Create a myCar object
    Car myCar = new Car();

    // Call the honk() method (from the Vehicle class) on the myCar object
    myCar.honk();

    // Display the value of the brand attribute (from the Vehicle class) and the value of the modelName from the Car class
    System.out.println(myCar.brand + " " + myCar.modelName);
  }
}
```

## super keyword
```java
class Animal {
  public void animalSound() {
    System.out.println("The animal makes a sound");
  }
}

class Dog extends Animal {
  public void animalSound() {
    super.animalSound(); // Call the parent method
    System.out.println("The dog says: bow wow");
  }
}

class Animal {
  String type = "Animal";
}

class Dog extends Animal {
  String type = "Dog";

  public void printType() {
    System.out.println(super.type); // Access parent attribute
  }
}

class Animal {
  Animal() {
    System.out.println("Animal is created");
  }
}

class Dog extends Animal {
  Dog() {
    super(); // Call parent constructor
    System.out.println("Dog is created");
  }
}
```

## Nested/Inner class
```java
class OuterClass {
  int x = 10;

  class InnerClass {
    int y = 5;
  }
}

public class Main {
  public static void main(String[] args) {
    OuterClass myOuter = new OuterClass();
    OuterClass.InnerClass myInner = myOuter.new InnerClass();
    System.out.println(myInner.y + myOuter.x);
  }
}
```

**Private Inner Class**
Unlike a "regular" class, an inner class can be private or protected. If you don't want outside objects to access the inner class, declare the class as private:

```java
class OuterClass {
  int x = 10;

  private class InnerClass {
    int y = 5;
  }
}
```

**Static Inner Class**

```java
class OuterClass {
  int x = 10;

  static class InnerClass {
    int y = 5;
  }
}

public class Main {
  public static void main(String[] args) {
    OuterClass.InnerClass myInner = new OuterClass.InnerClass();
    System.out.println(myInner.y);
  }
}
```

## Abstract class

=template with partial implementation.

```java
abstract class Animal {
  // Abstract method (does not have a body)
  public abstract void animalSound();
  // Regular method
  public void sleep() {
    System.out.println("Zzz");
  }
}

// Subclass (inherit from Animal)
class Pig extends Animal {
  public void animalSound() {
    // The body of animalSound() is provided here
    System.out.println("The pig says: wee wee");
  }
}

class Main {
  public static void main(String[] args) {
    Pig myPig = new Pig(); // Create a Pig object
    myPig.animalSound();
    myPig.sleep();
  }
}
```

## Interfaces
=promise to implement features.
```java
// Interface
interface Animal {
  public void animalSound(); // interface method (does not have a body)
  public void sleep(); // interface method (does not have a body)
}

// Pig "implements" the Animal interface
class Pig implements Animal {
  public void animalSound() {
    // The body of animalSound() is provided here
    System.out.println("The pig says: wee wee");
  }
  public void sleep() {
    // The body of sleep() is provided here
    System.out.println("Zzz");
  }
}

class Main {
  public static void main(String[] args) {
    Pig myPig = new Pig();  // Create a Pig object
    myPig.animalSound();
    myPig.sleep();
  }
}
```

Java does not support "multiple inheritance" (a class can only inherit from one superclass). However, it can be achieved with interfaces, because the class can implement multiple interfaces. Note: To implement multiple interfaces, separate them with a comma (see example below).

```java
interface FirstInterface {
  public void myMethod(); // interface method
}

interface SecondInterface {
  public void myOtherMethod(); // interface method
}

class DemoClass implements FirstInterface, SecondInterface {
  public void myMethod() {
    System.out.println("Some text..");
  }
  public void myOtherMethod() {
    System.out.println("Some other text...");
  }
}

class Main {
  public static void main(String[] args) {
    DemoClass myObj = new DemoClass();
    myObj.myMethod();
    myObj.myOtherMethod();
  }
}
```

## Anonymous class
```java
// Normal class
class Animal {
  public void makeSound() {
    System.out.println("Animal sound");
  }
}

public class Main {
  public static void main(String[] args) {
    // Anonymous class that overrides makeSound()
    Animal myAnimal = new Animal() {
      public void makeSound() {
        System.out.println("Woof woof");
      }
    }; // semicolon is required to end the line of code that creates the object

    myAnimal.makeSound();
  }
}

// Interface
interface Greeting {
  void sayHello();
}

public class Main {
  public static void main(String[] args) {
    // Anonymous class that implements Greeting
    Greeting greet = new Greeting() {
      public void sayHello() {
        System.out.println("Hello, World!");
      }
    };

    greet.sayHello();
  }
}
```

## Java enum
```java
enum Level {
  LOW,
  MEDIUM,
  HIGH
}

public class Main {
  public static void main(String[] args) {
    Level myVar = Level.MEDIUM;

    switch(myVar) {
      case LOW:
        System.out.println("Low level");
        break;
      case MEDIUM:
         System.out.println("Medium level");
        break;
      case HIGH:
        System.out.println("High level");
        break;
    }
  }
}

enum Level {
  // Enum constants (each has its own description)
  LOW("Low level"),
  MEDIUM("Medium level"),
  HIGH("High level");

  // Field (variable) to store the description text
  private String description;

  // Constructor (runs once for each constant above)
  private Level(String description) {
    this.description = description;
  }

  // Getter method to read the description
  public String getDescription() {
    return description;
  }
}

public class Main {
  public static void main(String[] args) {
    Level myVar = Level.MEDIUM; // Pick one enum constant
    System.out.println(myVar.getDescription()); // Prints "Medium level"
  }
}
```