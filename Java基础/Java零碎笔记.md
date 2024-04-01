# Java零碎笔记

------

## JVM相关

堆内存：存放对象

栈内存：方法调用及变量存放

栈顶的方法是目前正在执行的方法，比如a()方法调用b()方法，则b()方法会在a()方法的上面。



## 格式化

%  [argument number]  [表示符]  [最小字符数]  [.精确度]  类型

注：

1、%和类型是必要的，[ ]里面的是选择性项目

2、如果要格式化的参数超过一个以上，可以在argument number指定是哪一个

3、注意精确度前面有个小圆点

例：

%,d	代表以十进制整数带有逗号的方式来表示

%.2f	代表以小数点后两位的方式来格式化此浮点数

%,.2f	代表整数部分以有逗号的形式表示，小数部分以两位来格式化



## 异常

RuntimeException被称为不检查异常，你可以自己抛出与抓住他们，但是没有这个必要，编译器也不管。

为什么编译器不关那些运行期间的异常？答：大部分的RuntimeException都是因为程序逻辑的问题，而不是以你所无法预测或防止的方法出现的执行期失败状况，你无法保证文件一直都在。你无法保证服务器不会死机。但是你可以确保程序不会运行例如对只有5项元素的数组取第8个元素的值的这种不合理的逻辑。

try/catch是用来处理真正的异常，而不是你程序的逻辑错误，该块要做的事是尝试恢复，或者至少会优雅地列出错误信息。



## 类

内部类的作用：

### 1、内部类可以很好的隐藏实现；

由于外部类是不允许定义为private或者protected类型的，所以如果我们要隐藏一些我们实现细节，就可以通过内部类来实现，比如支付是比较核心的功能，需要尽可能的隐藏其实现细节，调用者只需要能够使用即可，如下：

```java
public interface IPay{  
    void pay();  
}  
  
public class Payer1 {  

    private class PayImpl implements IPay{  
      
        public void pay() {  
          System.out.println("paying...");
        }  
    }  
  
    public IPay startPay() {  
        return new PayImpl();  
    }  
}  
  

class TestPay {
    public static void main(String[] args) {  
        Payer1 pm = new Payer1();  
        IPay p = pm.startPay();  
        p.pay();  
    }  
}  
```

上面的PayImpl定义在内部类中，用private修饰符来控制访问权限。在后面的main方法中，直接通过IPay.pay()方法进行操作，外部调用者甚至连该实现类的名字都没有看见，这样就可以尽可能的隐藏实现细节，体现java的封装性。

### 2、内部类可以实现多重继承；

现实世界中是存在多重继承关系的，比如，孩子既继承父亲也继承母亲的基因，父亲会work，母亲会dancing，孩子继承了父母的所有这些基因。由于在java中一个类只支持单继承，多实现，但是对于这种场景，用接口来定义父亲和母亲似乎不太合理，所以这时候内部类就发挥其作用了：

```java
public class father {
  void work(){...}
}

public class mother{
  void dancing(){...}
}

public class Child {
    public static void main(String args[]){
       Child child=new Child ();
       child.work();
       child.dancing();
   }

   public void work(){
	InnerChild1 child1 = new InnerChild1 ();
	child1.work();
	}

   public void dancing(){
    InnerChild2 child2 = new InnerChild2 ();
    child2.dancing();
    }

    private class InnerChild1 extends father{
        void work() {
          super.work();
        }
    }

    private class InnerChild2 extends mother{
        void dancing() {
          super.dancing();
        }
    }
}
```

这样孩子便继承了父母的优秀基因，可谓才艺双馨。

### 3、内部类拥有外部类的所有访问权限

由于非静态内部类会持有外部类的引用，因此，非静态内部类可以访问外部类的所有属性及方法，这样可以为程序的设计带来极大的灵活性。对第一段代码稍作修改：

```java
public interface IPay{  
    void pay();  
}  
  
public class Payer1 {  
   private double money;

    private class PayImpl implements IPay{  
      
        public void pay() {  
          money --;
          System.out.println("paying...");
        }  
    }  
  
    public IPay startPay() {  
        return new PayImpl();  
    }  
}  
  

class TestPay {
    public static void main(String[] args) {  
        Payer1 pm = new Payer1 ();  
        IPay p = pm.startPay();  
        p.pay();  
    }  
}  
```

我们在外部类中新增了一个用private修饰的money字段，然后在内部类PayImpl 调用pay方法时，修改了该字段的值，这样可以直接通过内部类来访问外部类。

### 4、可以避免父类和接口同方法名时的覆盖问题

试想一下，如果你的类要继承一个类，还要实现一个接口，可是你发觉你继承的类和接口里面有两个同名的方法该怎么办？你怎么区分它们？你可能会说，我改成不同名的不就完事了吗，多大点事？如果这个父类和接口你都能改的话，确实可以这样做，但是有没有办法在不改的前提下解决这个问题呢，或者说你根本就没有机会改，比如，他们是你引入的第三方SDK呢，你该咋办？ 这时候就需要我们实现内部类了：

```java
public class Payer{  
    void pay(){
     System.out.println("Payer implement...");
  }
}  

public interface IPay{  
    void pay();  
}  

 //上面部分中Payer类和IPay接口有相同的方法pay()，然后我们有如下实现方法：

public class Payer1 extends Payer  implements IPay{  
        @override
        public void pay() {  
          System.out.println("paying...");
        }  
    
}  
//  想问一下大家pay()这个方法是属于覆盖Payer这里的方法呢？还是IPay这里的方法。我怎么能调到Payer这里的方法？显然这是不好区分的。而我们如果用内部类就很好解决这一问题了。看下面代码

public class Payer1 extends Payer {

 private class InnerPayer implements IPay {
      public void pay() {
        Payer1.this.pay();
      }
 }
   IPay startPay() {
      return new InnerPayer ();
  }
}

class TestPay {
    public static void main(String[] args) {  
        Payer1 pm = new Payer1 ();  
        IPay p = pm.startPay();  
        p.pay();  
    }  
}  
```

这样就完美的解决了同名方法覆盖的问题。



## I/O和序列化

如果你把某个对象序列化，transient的应引用实例变量会以null返回，而不管储存当时它的值是什么。



## 集合与泛型

对象怎样才算相等

引用相等性：

对象相等性：

hashCode()

equals()

1、如果两个对象相等，则hashCode必须也是相等的。

2、如果两个对象相等，对其中一个对象调用equals()必须返回true。也就是说若a.equals(b)则b.equals(a)。

3、如果两个对象有相同的hashCode值，它们也不一定是相等的。但若两个对象相等，则hashCode一定是相等的。

4、因此若equals()被覆盖过，则hashCode()也必须要被覆盖。

5、hashCode()的默认行为是对在heap上的对象产生独特的值。如果没有override过hashCode()，则该class的两个对象怎样都不会被认为是相同的。

6、equals()的默认行为是执行==的比较。也就是说会去测试两个引用是否对上heap上的同一个对象。如果equals()没有被覆盖过，两个对象永远都不会被视为相同的，因为不同的对象有不同的字节组合。

 为什么不同的对象会有相同的hashcode的可能？因为hashCode()所使用的算法也许刚好会让多个对象传回相同的值，越糟糕的算法越容易碰撞，但这也与数据值域分布的特性有关。如果HashSet发现在比对的时候同样的hashcode有多个对象，它会使用equals()来判断是否完全符合。也就是说hashcode是用来缩小寻找的成本，但最后还是要用equals()才能认定是否真的找到相同的项目。