---
layout: post
title: "中缀表达式转后缀表达式"
date: 2015-03-18 17:25
comments: true
author: devliubo
categories: c++
tag: "RPN"

---

第一篇，尝试一下，也熟悉下md：“X总”要一个四则运算类，还要写出详细实现说明-_-，根据“X总”的实现要求，这个是其中的中缀表达式转后缀表达式的函数。
<!-- more -->

**函数convertToRPN()：返回值为string类型，返回原始表达式(中缀表达式)转换后的后缀表达式。**   
   
1.首先规定运算符的优先级从低到高依次为： `'#' < ')' < '('< '+' = '-' < '*' = '/'`
{% codeblock lang:c %}
int Calculator::priority(const char a)
{
    switch (a)
    {
        case '#':
            return 1;
        case ')':
            return 9;
        case '(':
            return 10;
        case '+':
        case '-':
            return 20;
        case '\*':
        case '/':
            return 30;
        default:
            return 0;
    }
}
{% endcodeblock %}
2.转换过程构造两个stack，opStack用于临时保存运算符*(为方便做比较，先放入一个优先级最低的占位符#)*，rpnStack用于保存后缀表达式。
   
3.逐个字符读取原始表达式，过程如下：

*	3.1当数字出现，获取包含小数点的完整数字，直接入栈rpnStack
*	3.2当运算符出现：
* * 1)如果是左括号，压入栈opStack
* * 2)如果是右括号，将opStack中距离栈顶最近的左括号之间的运算符逐个出栈，并逐个压入栈rpnStack，最后丢弃左括号
* * 3)如果是除了括号之外的运算符:
* * * a.如果当前运算符的优先级大于opStack栈顶元素的优先级，则入栈opStack</br>
* * * b.如果当前运算符的优先级小于等于opStack栈顶元素的优先级，则将opStack的栈顶元素出栈，并压入栈rpnStack，直到当前运算符的优先级大于opStack栈顶元素的优先级，此时，再将当前运算符入栈opStack
   
4.逐个字符读取完成后，如果opStack不为空，将opStack中的元素逐个出栈并压入栈rpnStack，直到opStack栈顶元素为#为止。
   
5.将rpnStack中得元素**逆序输出(栈底元素为后缀表达式首元素)**，即可得到后缀表达式。

{% codeblock lang:c %}
string Calculator::convertToRPN()
{
    stack<char> opStack;
    stack<string> rpnStack;
    
    opStack.push('#');
    
    const char *p = originExpression.c_str();//originExpression为string类型
    for (; *p != '\0'; p++)
    {
        if (isOperator(*p))
        {
            if (*p == '(')
            {
                opStack.push(*p);
            }
            else if (*p == ')')
            {
                while (opStack.top() != '(')
                {
                    char ch = opStack.top();
                    opStack.pop();
                    rpnStack.push(string(1,ch));
                }
                opStack.pop();
            }
            else
            {
                if (priority(*p) > priority(opStack.top()))
                {
                    opStack.push(*p);
                }
                else
                {
                    while (priority(*p) <= priority(opStack.top()))
                    {
                        char ch = opStack.top();
                        opStack.pop();
                        rpnStack.push(string(1,ch));
                    }
                    opStack.push(*p);
                }
            }
        }
        else
        {
            string aNum = "";
            for (; *p != '\0' && isNumbers(*p); p++)
            {
                aNum += *p;
            }
            
            rpnStack.push(aNum);
            p--;
        }
    }
    
    while (!opStack.empty() && opStack.top() != '#')
    {
        char ch = opStack.top();
        opStack.pop();
        rpnStack.push(string(1,ch));
    }
    
    string rpnExpression;
    int length = (int)rpnStack.size();
    for (int i = length-1; i>= 0; i--)
    {
        rpnExpression = rpnStack.top() + " " + rpnExpression;
        rpnList.push_front(rpnStack.top());//逆序输出(栈底元素为后缀表达式首元素)
        rpnStack.pop();
    }
    
    return rpnExpression;
}
{% endcodeblock %}
