---
layout: post
title: "JVM 之 String 常量池《一》"
date: 2014-05-27 20:30:20 +0800
comments: true
categories: jvm 
---
####1.Sting对象创建方式
JAVA里面创建字符串有两种方式：
1. 通过字符串常量方式:String a = "a";
2. 通过new 关键字创建:new String("a");

但是这两种创建字符串的方式有很大的不同。
1. 使用第一种方式，jvm首先会检查字符串池中是否存在了这个常量，如果存在，就返回池中的实例引用。如果不存在，就会实例化一个字符串并放到池中,然后返回引用。
2. 使用第二种方式，则直接分配到heap中，不指向字符串池中的任何对象，和常量池没有关系。

####2.下面来看一个例子: [本例子Gist地址](https://gist.github.com/zarue/25e0afedb33de86da650) 
```java
public class StringConstantTest {
	public static String str_0_static = "a";
	public String str_0 = "a";
	
	public static void main(String args[]){
		
		StringConstantTest sct = new StringConstantTest();
		//同package对象
		StringConstantTest1 sct1 = new StringConstantTest1();
		//不同package对象
		StringConstantTest2 sct2 = new StringConstantTest2();
		
		String str0 = "a";
		String str1 = "b";
		String str2 = "ab";
		String str3 = "a"+"b";
		String str4 = str0 + str1;
		String str5 = str0 + "b";
		String str6 = new String("ab");
		String str7 = new String("ab");
		String str8 = new String(str4);
		
		//局部变量和成员变量
		System.out.println("str0==sct.str_0: "+(str0 == sct.str_0));
		System.out.println("str0==sct1.str_0: "+(str0 == sct1.str_0));
		System.out.println("str0==sct2.str_0: "+(str0 == sct2.str_0));
		
		//局部变量和静态变量
		System.out.println("str0==str_0_static： "+(str0 == str_0_static));
		System.out.println("str0==StringConstantTest1.str_0_static: "+(str0 == StringConstantTest1.str_0_static));
		System.out.println("str0==StringConstantTest2.str_0_static: "+(str0 == StringConstantTest2.str_0_static));
		
		//局部变量
		System.out.println("str2==str3: "+(str2 == str3));
		System.out.println("str2==str4: "+(str2 == str4));
		System.out.println("str2==str5: "+(str2 == str5));
		
		//局部变量和对象
		System.out.println("str2==str6: "+(str2 == str6));
		System.out.println("str2==str7: "+(str2 == str7));
		System.out.println("str2==str8: "+(str2 == str8));
		
		//对象和对象 intern
		System.out.println("str2==str6.intern(): "+(str2 == str6.intern()));
		System.out.println("str7.intern()==str6.intern(): "+(str7.intern() == str6.intern()));
		System.out.println("str8.intern()==str6.intern(): "+(str8.intern() == str6.intern()));
		
		//对象和对象equals
		System.out.println("str6.equals(str7): "+str6.equals(str7));
		System.out.println("str2.equals(str6): "+str2.equals(str6));
	}
	
}
 
class StringConstantTest1 {
	public String str_0 = "a";
	public static String str_0_static = "a";
}
```
 
![结果](/images/blog/2014-05/20140528-string-pool-1.png)

####分析(下面所指行号均为源代码行号):
1. 24-31行的结果说明:  
只要是字符串常量方式创建的对象，无论是类变量，实例变量，还是局部变量，无论是不是位于同一个包中。都是共享字符串池中的同一个实例。  


2. 34行为true, 是因为:  
String str3 = "a"+"b"; 是因为“a” 和 “b” 都是常量，编译器在编译阶段会直接优化为String str3 = "ab"。  
可以通过javap 反编译class字节码来看一下:  
```
30: astore 5
32: ldc #35 // String ab
34: astore 6
36: ldc #35 // String ab 
```
上面第36行就是`String str3 = "a"+"b";`对应的字节码。这里可以看出"a"+"b"已经被优化为"ab"了。  

3. 第35行为false, 是因为:  
String str4 = str0 + str1; 是因为str0 和 str1 都是变量，需要运行期才能转换为对应的值，而且String 会把+操作，转换成StringBuilder的append操作,然后返回一个String对象。  
再看一下字节码文件,清晰可见了:   
 
```
40: new #37 // class java/lang/StringBuilder
43: dup 
44: aload 4
46: invokestatic #39 // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
49: invokespecial #45 // Method java/lang/StringBuilder."<init>":(Ljava/lang/String;)V
52: aload 5
54: invokevirtual #48 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
57: invokevirtual #52 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
60: astore 8
```
由上面的字节码可见`String str4 = str0 + str1;`  相当于:  
```
StringBuilder sb = new Stringbuilder(str0);  
sb.append(str1);  
sb.toString();  
```
再看一下StringBuilder的toString()方法:  
```
public String toString() {
// Create a copy, don't share the array
return new String(value, 0, count);
}
```
看代码toString()是new了一个String对象返回，所以是直接分配在heap上。因此结果为false。  

4. 39-41行为false: 是因为:  
str6,str7,str8 都是通过new 创建，数据分配到heap上面，不指向字符串池中的任何对象，所以三个对象均不同，引用自然不同，因此比较结果均为false。  
5. 44-46行为true: 是因为:  
当一个String实例str调用intern()方法时，Java查找常量池中是否有相同Unicode的字符串常量，如果有，则返回其的引用，如果没有，则在常量池中增加一个Unicode等于str的字符串并返回它的引用。所以上面均为true。  

6. 49-50行为true: 是因为:  
String的值是用char数组保存的，equals 是比较的两个String对象中的char数组值是否一致，所以两个结果为true。  
看一下String的equals方法:  
```
 public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String) anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                            return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```
