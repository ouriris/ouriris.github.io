---
title: RecursiveDescent
date: 2017-11-23 17:47:09
tags: Java
---
Author: Oscar

### 目录
+ 递归算法
+ BNF范式
    + 定义
    + 终结符和非终结符
    + 语法
+ 递归下降求值法

# 递归算法
1. 递归算法是把问题转化为规模缩小了的同类问题的子问题，然后递归调用函数（或过程）来表示问题的解。子问题与原始问题为同样的事情，且更为简单<br>
2. 不能无限的调用本身，须有个出口。

# BNF范式
## 定义
> BNF范式也叫巴科斯范式，它奠定了现代计算机的基础。它以递归方式描述语言中的各种成分，凡遵守其规则的程序就可保证语法上的正确性。BNF由于其简洁、明了、科学而广泛接受，成为描述各种程序设计语言的常用工具。

## 终结符和非终结符
> 终结符：不能单独出现在推导式左边的符号，即不能再进行推导，是不可再拆分的最小元素。

> 非终结符：可理解为一个可拆分的元素

## 语法
> 公式： <符号> ::= <使用符号的表达式1>|<使用符号的表达式2> <br>
这里的<符号>是非终结符，即可以继续拆分，用<>括起来的是非终结符。::=和:=是同一个意思，都表示为定义。| 表示或，即可用表达式1或表达式2

### 示例
> <句子> ::= <主语> <谓语> <宾语> <br>
<谓语> ::= <动词> | <动词短语> <br>
<主语> ::= 你 | 我 | 他 <br>
<句子><谓语><主语><宾语>等等带<>的，都是非终结符，可以继续拆分，而你我他则是最小元素，不可拆分。

# 递归下降
> 传统上，编写语法分析器有两种方法，一种是自顶向下，一种是自底自上。自顶向下是从起始非终结符开始，不断地对非终结符进行分解，直到匹配输入的终结符；自顶向下方法就是我们所说的递归下降。

### 示例
> 使用BNF语法分析简单的四则运算表达式（一元多项式）<br>
如：1 + 3 * (4 + 2)
```java
<expr> ::= <expr> + <term>
         | <expr> - <term>
         | <term>
 
<term> ::= <term> * <factor>
         | <term> / <factor>
         | <factor>
 
<factor> ::= ( <expr> )
           | Num
```
> 有了BNF方法后，采用递归向下的方法来实现编译器是很直观的。
#### Java Code
```java
public class MyParser {
    public static int pos = 0;

    public static double expr(char[] s) {
        double t = term(s);
        while (s[pos] == '+' || s[pos] == '-') {
            if (s[pos] == '+'){
                pos++;
                t = t + term(s);
            }
            else if (s[pos] == '-'){
                pos++;
                t = t - term(s);
            }
        }
        return t;
    }

    public static double term(char[] s) {
        double t = factor(s);
        while (s[pos] == '*' || s[pos] == '/') {
            if (s[pos] == '*'){
                pos++;
                t = t * factor(s);
            }
            else if (s[pos] == '/'){
                pos++;
                t = t / factor(s);
            }
        }
        return t;
    }

    public static double factor(char[] s) {
        if (s[pos] == '('){
            pos++;
            double t = expr(s);
            if (s[pos] == ')'){
                pos++;
            }
            return t;
        }else{
            int t = 0;
            while('0' <= s[pos] && s[pos] <= '9'){
                t = t * 10 + s[pos] - '0';//t * 10 处理两位数以上;char[pos] - '0' =>'8'->int 8
                pos++;
            }
            return (double)t;
        }
    }

    public static void main(String[] args){
        String expression = "1+3*(4+2)#";
        double result = expr(expression.toCharArray());
        System.out.println("result : "+result);
    }

}
```