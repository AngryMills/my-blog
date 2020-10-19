# 从字节码看 finally关键字

大家好，我是烤鸭：

1.   认识finally

   finally 总是跟 try、catch一起出现，finally是执行方法结束一定要执行的代码，比如流关闭等等。

   finally是如何实现在异常捕捉之后保证执行 finally 代码块里的内容。

   其实不管是普通的代码，还是 try、catch ，JVM都是根据字节码文件中的指令来执行，

   也就是 finally的时候，字节码指令覆盖了这一种情况。

   而异常之后的操作指令是有专门的异常表来存储，在字节码指令之后（不是一定存在的，如果代码显示的try catch的话会有），结构如下图（图来自《深入理解java虚拟机》）

   ![1](E:\my\blog\字节码\finally\1.png)

2.   代码实践

   我们看一下 下面的代码，其实代码特别简单，输出3或4或者抛出非Exception的异常(finally会先执行)。

   ```
   public class TestCatchFinally {
       public static void main(String[] args) throws Throwable{
           int x = 0;
           try {
               x = 1;
               // throw new Throwable();
           } catch (Exception e) {
               x = 2;
           } finally {
               try {
                   x = 3;
               } catch (Exception e) {
                   x = 4;
               }
           }
           System.out.println(x);
       }
   }
   ```

   看一下生成的字节码指令

   ```
   javap -verbose TestCatchFinally.class
   ```

   ```
   Classfile / ... 无关内容省略
   Constant pool:
      // 常量池内容省略
   {
     public src.bytecode.TestCatchFinally();
       descriptor: ()V
       flags: ACC_PUBLIC
       Code:
         stack=1, locals=1, args_size=1
            0: aload_0
            1: invokespecial #1                  // Method java/lang/Object."<init>":()V
            4: return
         LineNumberTable:
           line 10: 0
         LocalVariableTable:
           Start  Length  Slot  Name   Signature
               0       5     0  this   Lsrc/bytecode/TestCatchFinally;
   
     public static void main(java.lang.String[]);
       descriptor: ([Ljava/lang/String;)V
       flags: ACC_PUBLIC, ACC_STATIC
        Code:
         stack=2, locals=5, args_size=1
            0: iconst_0
            1: istore_1
            2: iconst_1
            3: istore_1
            4: iconst_3
            5: istore_1
            6: goto          41
            9: astore_2
           10: iconst_4
           11: istore_1
           12: goto          41
           15: astore_2
           16: iconst_2
           17: istore_1
           18: iconst_3
           19: istore_1
           20: goto          41
           23: astore_2
           24: iconst_4
           25: istore_1
           26: goto          41
           29: astore_3
           30: iconst_3
           31: istore_1
           32: goto          39
           35: astore        4
           37: iconst_4
           38: istore_1
           39: aload_3
           40: athrow
           41: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
           44: iload_1
           45: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
           48: return
         Exception table:
            from    to  target type
                4     6     9   Class java/lang/Exception
                2     4    15   Class java/lang/Exception
               18    20    23   Class java/lang/Exception
                2     4    29   any
               15    18    29   any
               30    32    35   Class java/lang/Exception
         // ,,,
   }
   SourceFile: "TestCatchFinally.java"
   
   ```

   字节码指令比较简单了，没什么可说的，我们看下异常表。

   第1行指的是字节码指令 4-6行，对应的代码内容是 try {x = 3;}catch (Exception e) {x = 4;}。如果触发了这个异常，就会执行 9行指令，变量赋值为 4。

   第2行指的是字节码指令 2-4行，对应的代码内容是 try {x = 1;}catch (Exception e) {x = 2;}。如果触发了这个异常，就会执行 15行指令，变量赋值为 2。

   第3行指的是字节码指令 18-20行，对应的代码内容是 try {x = 3;}catch (Exception e) {x = 4;}。如果触发了这个异常，就会执行 23行指令，变量赋值为 4。

   关于指令重复，比如 iconst_3 执行了三次，第一次 是正常执行后执行 finally内容，第二次是 异常后执行finally内容，第三次是遇到 非 Exception异常 执行的finally

   两个any 指的是 catch不到的异常，比如 throwable ，如果触发，程序会执行finally后退出。

   

3.  总结

  字节码指令已经把显示异常和 finally 代码块都包含了，剩下的只是按指令执行而已了。

