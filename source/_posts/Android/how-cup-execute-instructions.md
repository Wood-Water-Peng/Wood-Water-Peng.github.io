---
layout: post
title: "CPU是怎样执行指令的"
excerpt: "基于CPU基本结构分析执行指令的流程"
tags: 
- CPU
- instruction
- Android
categories:
- Android
- CPU
comments: true
share: true
---

#### 执行指令的基本步骤

<!-- more --> 

<table>
    <tr>
        <td>抓取指令</td>
		<td>指令是什么？指令在哪里？CPU将抓取到的指令放在哪了？</td>
    </tr>
	<tr>
        <td>PC计数器增加</td>
		<td>什么是PC计数器？PC计数器保存的是什么？PC计数器自增后指向哪里？</td>
    </tr>
	<tr>
        <td>解码指令</td>
		<td>指令为什么需要被解码？指令是如何被解码的？</td>
    </tr>
	<tr>
        <td>抓取操作数</td>
		<td>操作数是什么?操作数在哪里？抓取后将他们放在哪里？</td>
    </tr>
	<tr>
        <td>执行操作</td>
		<td>CPU是如何执行操作的？</td>
    </tr>
	<tr>
        <td>保存结果</td>
		<td>结果是什么？从哪里得到？保存到哪里？</td>
    </tr>
	<tr>
       <td>重复上述步骤</td>
		<td>循环什么？从哪里开始循环?</td>
    </tr>
</table>


#### 下面以实际的代码片段来分析一下每个步骤的执行

基本的寄存器

* PC: Program Counter 程序计数器   保存CPU当前执行的指令的地址
* IR：Instruction Register 指令寄存器 保存CPU抓取的指令
* ALU：算术逻辑单元，接受两个或者一个输入，产生一个输出，可以执行简单的逻辑操作
* A,B,C：存放数据的寄存器

假设我们这里用c来写这样一行代码  

	float result=50.00*0.01
	
那么经过编译器后，这行代码会产生多句汇编代码，汇编器又会将汇编代码转化成二进制的机器代码，机器代码是可以直接被CPU执行的  

<figure class="half">
	<a href="/images/How-cpu-execute-instructions/cup_execute_instructions_01.png"><img src="/images/How-cpu-execute-instructions/cup_execute_instructions_01.png"></a>
	<figcaption>CPU和RAM</figcaption>
</figure>   

**为了便于理解，我们将RAM中保存的机器指令用符号表示**
  
	1.  在初始时，PC保存着指令100（指令的地址）  
	2.  CPU从指令内存中抓取指令100（LOAD A，2000），存放到指令寄存器IR中
	3.  PC计数器自增，指向下一条指令地址(104)   
	4.  CPU解析指令---将地址为2000的内存数据放到寄存器A中
	5.  CPU从指令内存中抓取指令104（LOAD B，2004），存放到指令寄存器IR中
	6.  PC计数器自增，指向下一条指令地址(108)
	7.  CPU解析指令---将地址为2004的内存数据放到寄存器B中
	8.  CPU从指令内存中抓取指令108（Multiply A,B,C），存放到指令寄存器中
	9.  PC计数器自增，指向下一条指令地址(112)
	10. CPU解析指令---将寄存器A和B的内容相乘，结果保存在寄存器C中
	11.将寄存器C中的值保存在内存地址为2008的数据块中

---

上面的分析是非常简化的，具体的执行流程肯定是要复杂的多，但是先了解基本概念再深入细节绝对是正确的学习姿势。在理解的过程中，把我两个抽象是很重要的

*   指令集的概念  
	指令集就好比是CPU能够看得懂的语言，如常见的Intel的IA32指令集。编译器将高级语言C，java等写成的代码编译成机制语言，也就是机器指令，对应的CPU能够读懂该指令，接着就是在寄存器，RAM之间不断的搬移数据，一般会将得到的结果写入RAM，显示器一刷新，就是我们看到的。
	
*   虚拟地址的概念    
    我们可以将RAM看成一大块字节数组，CPU所看到的地址都是虚拟地址，如100，真正得到硬件地址还需要操作系统的配合。





