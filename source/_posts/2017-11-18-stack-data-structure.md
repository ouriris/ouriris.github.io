---
title: stack data structure
date: 2017-12-04 08:52:32
tags:
---
#### 一、什么是数据结构
&emsp;&emsp;数据结构是计算机存储、组织数据的方式。数据结构是指相互之间存在一种或多种特定关系的数据元素的集合。在初学阶段，你可以认为数据结构就是关于数据在计算机中如何组织的一门课程。
&emsp;&emsp;比如，我要往一个数组中，存1，3，2这三个整数，那么，我实际存的时候，是按照从小到大排着存呢，还是从大到小存，还是没有顺序随便存，这要根据实际的需求来决定。根据需求决定数据存储的方式。这就是数据结构要研究的内容.
#### 一、什么是栈
&emsp;&emsp;栈是一种数据结构。栈是一种特殊的线性表。其特殊性在于限定插入和删除数据元素的操作只能在线性表的一端进行，最大的特点就是先进后出。
&emsp;&emsp;想一想，生活中，有很多栈的例子。比如叠在一起的碗盘，叠的时候我们是从底往高处叠，但是取的时候，我们是从最上面的一个依次向下取。这也是一个典型的栈：后进先出，先进后出。
![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2017-11-30/stack.png)
#### 一、栈的实现
- **使用int数组**
```
public class Stack {
    private int[] data;
    private int size;
    private int top = 0; // 指向栈的顶部

    public Stack(int size) {
        this.size = size;
        this.data = new int[size];
    }   
}
```
- **使用泛型**
```
package cn.kelvin.stack;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Stack;

public class MyStack<T> {
	Object[] datas = {};
	int size;
	int top=0;
	
	public MyStack() {
	}
	
	public MyStack(int size){
		this.size = size;
		datas = new Object[size];
	}
	
	//压栈操作
	public void push(T t){
		ensureCapacity(top);
		datas[top++] = (Object) t;
	}
	
	//当栈中的元素超出最大容量时扩容
	private void ensureCapacity(int newSize) {
		if(newSize>=size){
			size = (int)(Math.round(size + size*1.5+1));
			datas = Arrays.copyOf(datas,size);
		}
	}

    //出栈操作
	public T pop(){
		if(top<=size && top>0){
			return (T) datas[--top];
		}
		return null;
	}
	
	//访问栈顶元素
	public T getTop(){
		int n = top - 1;
		if(n<size && n>=0){
			return (T) datas[n];
		}
		return null;
	}
	
	public int size(){
		return top;
	}
	
	public boolean isEmpty(){
		return top==0;
	}
}
```
&emsp;&emsp;也可以使用java中的list来作为stack的底层元素容器，读者可以自己尝试去实现。另外可以去看看jdk中stack的相关实现，读源代码是最好的学习方式。

#### 一、栈的应用
&emsp;&emsp;栈在计算机编程中可以说是最基础也是最重要的数据结构了。其功能之强大，可能出乎很多人的意料。
- **使用栈实现括号间的匹配验证**（使用上文中泛型栈）
```
public static boolean isMatch(String matchStr){
	MyStack<Integer> stack = new MyStack<>();
	char[] chs = matchStr.toCharArray();
	for(int i=0;i<chs.length;i++){
		if(chs[i]=='(' || chs[i]=='[' || chs[i]=='{'){
			stack.push(Integer.valueOf(chs[i]));
		}else {
			if(stack.isEmpty()){
				System.out.println("stack empty");
				return false;
			}else{
				if(')'==chs[i]){
					if((stack.getTop()+1)==Integer.valueOf(chs[i])){
						stack.pop();
					}else{
						System.out.println("not match");
						return false;
					}
				}else if(']'==chs[i] || '}'==chs[i]){
					if((stack.getTop()+2)==Integer.valueOf(chs[i])){
						stack.pop();
					}else{
						System.out.println("not match");
						return false;
					}
				}else {
					return false;
				}
			}
		}
	}
	if(stack.isEmpty()){
		return true;
	}
	return false;
}
```
该程序仅支持 ({[ 之间的匹配，对于输入包含其他字符的字符串，会直接return false，读者可以自行输入测试。

- **使用栈实现后缀表达式求值**
该程序的实现需要借助另一个工具类：TokenStream, 该类是对System.in的一个适配器类。TokenStream接口定义：
```
public interface TokenStream {
	public Token getToken()  throws IOException;
    public void consumeToken();
}

public class Token {
	public enum TokenType {
		LPAR, RPAR,
        PLUS,
        MINUS,
        MULT,
        DIV,
        INT,
        NONE,
	}
	
	TokenType tokenType;
	Object value;
	
	public Token(TokenType tokenType, Object value) {
		this.tokenType = tokenType;
		this.value = value;
	}

	@Override
	public String toString() {
		String val = String.valueOf(value);
		try {
			Integer.valueOf(val);
			return "{" + tokenType + ", INTEGER(" + value + ")}";
		} catch (Exception e) {
			return "{" + tokenType + ", " + '"'+val+'"' + "}";
		}
		
	}

	public TokenType getTokenType() {
		return tokenType;
	}

	public void setTokenType(TokenType tokenType) {
		this.tokenType = tokenType;
	}

	public Object getValue() {
		return value;
	}

	public void setValue(Object value) {
		this.value = value;
	}
	
}
```
- 我的一个实现类（实现的方法应该会有所差异，读者可以尝试去使用自己的方式实现）
```java
package cn.kelvin.adapter;

import java.io.IOException;
import java.io.InputStream;
import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

import cn.kelvin.adapter.Token.TokenType;

public class MyTokenStream implements TokenStream{
	private int tokenIndex=0;
	private List<Token> tokenCache = new ArrayList<>();
	InputStream in;

	@SuppressWarnings("resource")
	public MyTokenStream(InputStream in) {
		this.in = in;
		String input = new Scanner(in).nextLine();
		parseInput(input);
	}
	
	private void parseInput(String input) {
		if(null==input || "".equals(input)){
			return;
		}
		int i=0;
		while(i<input.length()){
			char ch = input.charAt(i);
			switch (ch) {
			case '(':
				tokenCache.add(new Token(Token.TokenType.LPAR, String.valueOf(ch)));
				break;
			case ')':
				tokenCache.add(new Token(Token.TokenType.RPAR, String.valueOf(ch)));			
				break;
			case '+':
				tokenCache.add(new Token(Token.TokenType.PLUS, String.valueOf(ch)));
				break;
			case '-':
				tokenCache.add(new Token(Token.TokenType.MINUS, String.valueOf(ch)));
				break;
			case '*':
				tokenCache.add(new Token(Token.TokenType.MULT, String.valueOf(ch)));
				break;
			case '/':
				tokenCache.add(new Token(Token.TokenType.DIV, String.valueOf(ch)));
				break;
			default:
				try {
					Integer num = Integer.valueOf(String.valueOf(ch));
					tokenCache.add(new Token(Token.TokenType.INT, num));
				} catch (Exception e) {
//					e.printStackTrace();
				}
				break;
			}
			i++;
		}
	}

	@Override
	public Token getToken() throws IOException {
		if(tokenIndex<tokenCache.size()){
			return tokenCache.get(tokenIndex);
		}
		return new Token(TokenType.NONE, null);
	}

	@Override
	public void consumeToken() {
		tokenIndex ++;
	}
}
```
- **java虚拟机的实现是基于栈的**</br>
这里举一个简单的例子来说明在java虚拟机中是如何使用栈。Java虚拟机规范里定义了Java所使用的字节码。我们知道Java文件要先编译成字节码文件，也就是class文件，请看example：
```
public class Main { // Main.java
    public static int add(int a, int b){ 
        return a + b;
    }   
}
```
- 执行javac Main.java（或者javap -c Main）后可以看到编译后的字节码：
```
public static int add(int, int);
    Code:
       0: iload_0
       1: iload_1
       2: iadd
       3: ireturn
```
add 函数被编成了四条字节码指令，这四条字节码是什么意思呢？</br>
你可以认为Java的每一个函数都有一个操作数栈，每条指令就是在对这个操作数栈进行操作。比如 iload_0，就代表把第一个参数 push 进操作数栈，iload_1代表把第二个参数 push 进操作数栈，而 iadd 代表，从栈上连续pop两次，取两个数，将其相加，再把结果送到栈上。ireturn 则表示把栈顶的值做为返回值传给调用者。Java字节码的整个执行过程都是在这样一个栈上的。如果用图来表示，这四个步骤就是:
- iload_0，使a先进栈

![](https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2017-11-30/java_stack_01.png)

- iload_1， 使b进栈

![](/uploads/https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2017-11-30/java_stack_02.png)

- iadd，做了三件事情，b 和 a 出栈，计算 a+b，将计算结果入栈：

![](/uploads/https://raw.githubusercontent.com/ouriris/ouriris.github.io/hexo/source/uploads/2017-11-30/java_stack_03.png)

- ireturn 将当前栈顶的值返回出去。