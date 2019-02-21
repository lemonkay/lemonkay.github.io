
## 关于java方法参数是  Pass By Value or Pass By Reference？

<br/>
#### 起因，当然show code啦 ： ##
```java
public class MethodTest {

    public static void main(String[] args) {
        Integer a = new Integer(1);
        addNum(a);
        System.out.println("a="+a);

    }

    public static void addNum(Integer b) {
        b = new Integer(2);
        Integer c = b + 3;
        System.out.println("b="+b);
        System.out.println("c="+c);

    }
}
```
* 这边执行结果是： b=2 c=5  a=1
* a是Integer 引用类型，在addNum() 重新赋值 2，但最后打印的结果还是为1？why，


 
### hotspot jvm 的引用类型图

![hotspot](/images/javaReference.png)


>  使用直接指针访问方式的最大好处就是速度更快，它节省了一次指针定位的时间开销，由于对象的访问在Java中非常频繁，因此这类开销积少成多后也是一项非常可观的执行成本。

### 1. **javap -verbose MethodTest**   javap生成的字节码


```java
 public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #2                  // class java/lang/Integer
         3: dup
         4: iconst_1
         5: invokespecial #3                  // Method java/lang/Integer."<init>":(I)V
         8: astore_1
         9: aload_1
        10: invokestatic  #4                  // Method addNum:(Ljava/lang/Integer;)V
        13: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
        16: new           #6                  // class java/lang/StringBuilder
        19: dup
        20: invokespecial #7                  // Method java/lang/StringBuilder."<init>":()V
        23: ldc           #8                  // String a=
        25: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        28: aload_1
        29: invokevirtual #10                 // Method java/lang/StringBuilder.append:(Ljava/lang/Object;)Ljava/lang/StringBuilder;
        32: invokevirtual #11                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        35: invokevirtual #12                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        38: return
      LineNumberTable:
        line 11: 0
        line 12: 9
        line 13: 13
        line 15: 38
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      39     0  args   [Ljava/lang/String;
            9      30     1     a   Ljava/lang/Integer;

  public static void addNum(java.lang.Integer);
    descriptor: (Ljava/lang/Integer;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #2                  // class java/lang/Integer
         3: dup
         4: iconst_2
         5: invokespecial #3                  // Method java/lang/Integer."<init>":(I)V
         8: astore_0
         9: aload_0
        10: invokevirtual #13                 // Method java/lang/Integer.intValue:()I
        13: iconst_3
        14: iadd
        15: invokestatic  #14                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        18: astore_1
        19: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
        22: new           #6                  // class java/lang/StringBuilder
        25: dup
        26: invokespecial #7                  // Method java/lang/StringBuilder."<init>":()V
        29: ldc           #15                 // String b=
        31: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        34: aload_0
        35: invokevirtual #10                 // Method java/lang/StringBuilder.append:(Ljava/lang/Object;)Ljava/lang/StringBuilder;
        38: invokevirtual #11                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        41: invokevirtual #12                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        44: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
        47: new           #6                  // class java/lang/StringBuilder
        50: dup
        51: invokespecial #7                  // Method java/lang/StringBuilder."<init>":()V
        54: ldc           #16                 // String c=
        56: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        59: aload_1
        60: invokevirtual #10                 // Method java/lang/StringBuilder.append:(Ljava/lang/Object;)Ljava/lang/StringBuilder;
        63: invokevirtual #11                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        66: invokevirtual #12                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        69: return
      LineNumberTable:
        line 18: 0
        line 19: 9
        line 20: 19
        line 21: 44
        line 23: 69
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      70     0     b   Ljava/lang/Integer;
           19      51     1     c   Ljava/lang/Integer;
```
 **简单分析一下**：
 
  *  ![hotspot](/images/jvm栈帧.png)
  *   >  栈 帧（ Stack Frame） 是 用于 支持 虚拟 机 进行 方法 调用 和 方法 执行 的 数据 结构， 它是 虚拟 机 运行时 数据区 中的 虚拟 机 栈（ Virtual Machine Stack）[ 1] 的 栈 元素。
  
  *  >  操 作数 栈（ Operand Stack） 也 常 称为 操作 栈， 它是 一个 后入 先出（ Last In First Out, LIFO） 栈。当 一个 方法 刚刚 开始 执行 的 时候， 这个 方法 的 操 作数 栈 是 空的， 在 方法 的 执行 过程中， 会有 各种 字节 码 指令 往 操 作数 栈 中 写入 和 提取 内容， 也就是 出 栈/ 入栈 操作。
  *  >  字节 码 指令：**整数 加法 的 字节 码 指令 iadd**： 在 运行 的 时候 操 作数 栈 中最 接近 栈 顶的 两个 元素 已经 存入 了 两个 int 型 的 数值， 当 执行 这个 指令 时， 会 将 这 两个 int 值 出 栈 并 相加， 然后 将 相加 的 结果 入栈。
     * istore指令
      ![istore](/images/istore.png)
     * istore指令
      ![istore](/images/iload.png)
 
 
 *  >  LocalVariableTable:局部 变 量表 是一 组 变量 值 存储 空间， 用于 存放 方法 参数 和 方法 内部 定义 的 局部 变量。


 * **站在巨人的肩膀上**：
  ```java	
  LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      70     0     b   Ljava/lang/Integer;
  ```
	方法addNum（） 的slot 的第一个变量b 是Integer引用类型
	
	```java
	 0: new           #2                  // class java/lang/Integer
         3: dup
         4: iconst_2
         5: invokespecial #3                  // Method java/lang/Integer."<init>":(I)V
         8: astore_0
         9: aload_0
	```
这是  *b = new Integer(2)* 这个代码的字节码过程，iconst_2 是将int 2推至栈顶，之后就是调Integer的<init>方法 new 一个Integer对象，将其地址赋给引用变量 b（astore_0 指令）。


#### 2. **idea反编译的class**

```java
public class MethodTest {
    public MethodTest() {
    }

    public static void main(String[] args) {
        Integer a = new Integer(1);
        addNum(a);
        System.out.println("a=" + a);
    }

    public static void addNum(Integer b) {
        b = new Integer(2);
        Integer c = Integer.valueOf(b.intValue() + 3);
        System.out.println("b=" + b);
        System.out.println("c=" + c);
    }
}
```

这里反编译 将 *Integer c = b + 3;* 优化成 *Integer c = Integer.valueOf(b.intValue() + 3);* ，通过字节码也能够看出

#### 3. **回到最初的问题**
1. main 方法中的变量 a的值为什么没变？
2. 因为在main方法等 局部变量表中的a  一直引用堆中 value（还是final）是1的对象。
而传给方法 addNum()

     ```java
     9: aload_1
     10: invokestatic  #4                  // Method addNum:(Ljava/lang/Integer;)V
     
     ```
      ![aload](/images/aload.png)  
    是将 main 栈帧中 局部变量表的第一个slot的a推至操作栈顶，执行addNum()栈帧时，看字节码 是new Integer(2) 后直接将栈顶引用值赋给b；
    
 ```java
         5: invokespecial #3                  // Method java/lang/Integer."<init>":(I)V
         8: astore_0
     
  ```
![astore](/images/astore.png)  


3. **java说到底还是 Pass By Value，只不过对于引用类型来说，value是堆中对象的地址；就是方法中的参数变量 是通过操作栈来传递的.**
