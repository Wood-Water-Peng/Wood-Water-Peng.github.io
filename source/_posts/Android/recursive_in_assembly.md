layout: post
title: "Recursive in assembly"
excerpt: "Recursive analysis using assembly"
tags: 
- recursive
- assembly
- Android
categories:
- Android
comments: true
share: true
---

###递归在汇编代码中的体现


c代码如下:

    `

		int rfact(int n){
			int result;
			if(n<1){
				result=1;				
			}else{
				result=n*rfact(n-1);
			}
			return result;
    	}

	`
<!-- more --> 

汇编代码:

%ebp 帧指针寄存器        
%esp 栈指针寄存器

*由于代码是手写的，在写的时候心里面要抽象出栈帧结构*

    `
		rfact:
			pushl  %ebp          save old %ebp
			movl   %esp,%ebp     set %ebp as frame pointer
			pushl  %ebx          save register %ebx
			subl   $4,%esp       allocate 4 bytes on stack 
			movl   8(%ebp),%ebx	 get n
			movl   $1,%eax       set result=1
	        comp   $1,%ebx       compare n:1
			jle    .L53
			leal   -1(%ebx),%eax compute n-1
			movl   %eax,(%esp)   store at top of stack
			call   rfact
			imull  %ebx,%eax     compute result=return value*n 
		
		.L53:
			addl   $4,%esp       deallocate 4 bytes from stack
			popl   %ebx          restore %ebx
			popl   %ebp          restore %ebp
			ret                  return result 
	
	`
现在假设由main函数调用rfact函数，那么rafct会在main的栈帧中获得参数n
每当我们创建出一个栈帧时，会有三个步骤：

1. 建立    初始化栈帧  
	如，保存之前的帧指针，确定新的帧指针，将需要保存的寄存器值压栈
2. 主体    执行过程的整体计算  
	如操作栈指针，给寄存器赋值，在寄存器之间搬移数据
3. 结束    恢复栈的状态，过程返回  
    如，恢复部分寄存器的值，调用ret返回过程
	
用该例来分析各个过程的体现

![](http://i.imgur.com/d5ollZI.png)

**初始化栈帧**

每一个函数的栈底由%ebp寄存器决定,栈顶由%esp寄存器决定,当%ebp确定后，一般不会再改变，而%esp会随着指令的执行而不断变化。

对于每一个函数而言，%ebp的值是不一样的，而%esp表示的是整个函数栈的深度，也表示了指令执行的位置。  
所以，每当开启一个新栈帧时，需要将上一个栈帧的起始位置保存起来，即将%ebp寄存器的值压栈。  
同时，有些寄存器是上一个过程后面需要的，如果该过程需要使用，那么就必须将那些寄存器的值保存起来，即压栈。

**执行过程**

当我们确定好帧指针，保存好必要的寄存器后，就可以开始执行指令了。
如，给寄存器赋值，将数据保存到栈顶，执行算术运算等

这里，我们假设n的初始值为**2**  
    
**执行前各寄存器的状态:**
  
%esp  保存着第一个rfact过程的帧指针  
%ebx  保存着n的值，即2
%eax  值为1  
%esp  栈顶,保存着n-1，即1

call rfact 指令执行

**执行后的寄存器状态:**  

将之前%ebp的值压栈  
将之前%ebx的值压栈  
将当前%esp的值赋值给%ebp,此时%ebp指向当前栈帧的栈顶  
拿到上一个栈帧栈顶的数据，保存到%ebx，即1  
将%eax的值设置为1  
此时，条件满足，跳转到.L53  
弹出栈顶值保存到%ebx,即2  
弹出栈顶值，赋值到%ebp  
跳到上一过程的返回地址，即call之后的那条指令的地址  
执行imull %ebx,%eax   
执行完后，%eax的值为2  
继续执行.L53  
基本流程和上面一样，恢复%ebx和%ebp的值，跳转到上一过程的返回地址继续执行



**恢复**

当这个函数的任务完成后，接下来就要做善后工作了。
如，将之前在栈上分配的空间回收，恢复之前保存的寄存器值(%esp)，让栈指针指向上一个过程的返回地址






