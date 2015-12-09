# The analysis of a Static Singleton Crash Issue

今天在公司看到下面这样的一段代码，问这样运行的结果会是怎样呢？




		#include <stdio.h>
		
		class A
		{
		public:
		  static A& getInstance()
		  {
		    static A a;
		    return a;
		  }
		
		  A():m_a(0)
		  {
		    m_a = new int;
		    printf("%s\n",__FUNCTION__);
		  }
		  
		  ~A()
		  {
		    delete m_a;
		    m_a = NULL;
		    printf("%s\n",__FUNCTION__);
		  }

		  void f()
		  {
		    *m_a = 100;
		  }
		private:
		    int* m_a;
		}; 
		
		class B
		{
		public:
		  static B& getInstance()
		  {
		    static B a;
		    A::getInstance();
		    return a;
		  }
		  
		  B()
		  {
		    printf("%s\n",__FUNCTION__);
		  }
		
		  ~B()
		  {
		    printf("%s\n",__FUNCTION__);
		    A::getInstance().f();
		  }
		};
		
		int main()
		{
		  B::getInstance();
		  return 0;
		}

运行的结果:

		> ./a.out 
		B
		A
		~A
		~B
		Segmentation fault (core dumped)

为什么会这样呢？不是都是Static的变量的吗？如果是static的话，里面的空间分配应该还是在才对的啊?

为了理解里面的行为，我用汇编来看一下里面到底发生了什么事！
main里面首先调用的是B::getInstance,我们直接看B::getInstance吧，在gdb里面反汇编的结果是这样的：

		Dump of assembler code for function B::getInstance():
		   0x00000000004009f7 <+0>: push   %rbp
		   0x00000000004009f8 <+1>: mov    %rsp,%rbp
		   0x00000000004009fb <+4>: push   %r12
		   0x00000000004009fd <+6>: push   %rbx
		   0x00000000004009fe <+7>: mov    $0x601090,%eax
		   0x0000000000400a03 <+12>:  movzbl (%rax),%eax
		   0x0000000000400a06 <+15>:  test   %al,%al
		   0x0000000000400a08 <+17>:  jne    0x400a4b <B::getInstance()+84>
		   0x0000000000400a0a <+19>:  mov    $0x601090,%edi
		   0x0000000000400a0f <+24>:  callq  0x400750 <__cxa_guard_acquire@plt>   // 0, make sure to construct
		   0x0000000000400a14 <+29>:  test   %eax,%eax                            //    B static variable only once
		   0x0000000000400a16 <+31>:  setne  %al
		   0x0000000000400a19 <+34>:  test   %al,%al
		   0x0000000000400a1b <+36>:  je     0x400a4b <B::getInstance()+84>   // 1, double check
		   0x0000000000400a1d <+38>:  mov    $0x0,%r12d
		   0x0000000000400a23 <+44>:  mov    $0x601098,%edi
		   0x0000000000400a28 <+49>:  callq  0x400a7a <B::B()>   // 2, constructor
		   0x0000000000400a2d <+54>:  mov    $0x601090,%edi
		   0x0000000000400a32 <+59>:  callq  0x4007b0 <__cxa_guard_release@plt>
		   0x0000000000400a37 <+64>:  mov    $0x601078,%edx
		   0x0000000000400a3c <+69>:  mov    $0x601098,%esi     //  3, B static variable address
		   0x0000000000400a41 <+74>:  mov    $0x400a92,%edi     //  4, B destrutor address
		   0x0000000000400a46 <+79>:  callq  0x4007a0 <__cxa_atexit@plt>  //  5, call __cxa_atexit
		   0x0000000000400a4b <+84>:  callq  0x4008fd <A::getInstance()>  //  6, call A::getInstance
		   0x0000000000400a50 <+89>:  mov    $0x601098,%eax
		   0x0000000000400a55 <+94>:  jmp    0x400a74 <B::getInstance()+125>
		   0x0000000000400a57 <+96>:  mov    %rax,%rbx
		   0x0000000000400a5a <+99>:  test   %r12b,%r12b
		   0x0000000000400a5d <+102>: jne    0x400a69 <B::getInstance()+114>
		   0x0000000000400a5f <+104>: mov    $0x601090,%edi
		   0x0000000000400a64 <+109>: callq  0x4007c0 <__cxa_guard_abort@plt>
		   0x0000000000400a69 <+114>: mov    %rbx,%rax
		   0x0000000000400a6c <+117>: mov    %rax,%rdi
		   0x0000000000400a6f <+120>: callq  0x4007f0 <_Unwind_Resume@plt>
		   0x0000000000400a74 <+125>: pop    %rbx
		   0x0000000000400a75 <+126>: pop    %r12
		   0x0000000000400a77 <+128>: pop    %rbp
		   0x0000000000400a78 <+129>: retq

从上面我标记的3,4,5，我们可以看出，static类的构造是在第一次使用时构造的，同时在构造完后将其析构函数放进了__cxa_atexit 链表里面，__cxa_atexit是什么，说白了就是 atexit.这个函数会在程序退出时自动调用，而且和stack一样，是后进先出的。类似地，A::getInstance与B::getInstance()是一样的：

		Dump of assembler code for function A::getInstance():
		   0x00000000004008fd <+0>: push   %rbp
		   0x00000000004008fe <+1>: mov    %rsp,%rbp
		   0x0000000000400901 <+4>: push   %r12
		   0x0000000000400903 <+6>: push   %rbx
		   0x0000000000400904 <+7>: mov    $0x601088,%eax
		   0x0000000000400909 <+12>:  movzbl (%rax),%eax
		   0x000000000040090c <+15>:  test   %al,%al
		   0x000000000040090e <+17>:  jne    0x400951 <A::getInstance()+84>
		   0x0000000000400910 <+19>:  mov    $0x601088,%edi
		   0x0000000000400915 <+24>:  callq  0x400750 <__cxa_guard_acquire@plt>
		   0x000000000040091a <+29>:  test   %eax,%eax
		   0x000000000040091c <+31>:  setne  %al
		   0x000000000040091f <+34>:  test   %al,%al
		   0x0000000000400921 <+36>:  je     0x400951 <A::getInstance()+84>
		   0x0000000000400923 <+38>:  mov    $0x0,%r12d
		   0x0000000000400929 <+44>:  mov    $0x6010a0,%edi
		   0x000000000040092e <+49>:  callq  0x40097a <A::A()>
		   0x0000000000400933 <+54>:  mov    $0x601088,%edi
		   0x0000000000400938 <+59>:  callq  0x4007b0 <__cxa_guard_release@plt>
		   0x000000000040093d <+64>:  mov    $0x601078,%edx
		   0x0000000000400942 <+69>:  mov    $0x6010a0,%esi      // 0, static variable A address
		   0x0000000000400947 <+74>:  mov    $0x4009ae,%edi      // 1, A destructor address
		   0x000000000040094c <+79>:  callq  0x4007a0 <__cxa_atexit@plt>  // 2, call __cxa_atexit
		   0x0000000000400951 <+84>:  mov    $0x6010a0,%eax
		   0x0000000000400956 <+89>:  jmp    0x400975 <A::getInstance()+120>
		   0x0000000000400958 <+91>:  mov    %rax,%rbx
		   0x000000000040095b <+94>:  test   %r12b,%r12b
		   0x000000000040095e <+97>:  jne    0x40096a <A::getInstance()+109>
		   0x0000000000400960 <+99>:  mov    $0x601088,%edi
		   0x0000000000400965 <+104>: callq  0x4007c0 <__cxa_guard_abort@plt>
		   0x000000000040096a <+109>: mov    %rbx,%rax
		   0x000000000040096d <+112>: mov    %rax,%rdi
		   0x0000000000400970 <+115>: callq  0x4007f0 <_Unwind_Resume@plt>
		   0x0000000000400975 <+120>: pop    %rbx
		   0x0000000000400976 <+121>: pop    %r12
		   0x0000000000400978 <+123>: pop    %rbp
		   0x0000000000400979 <+124>: retq   
		End of assembler dump.

看与面标记的0,1,2,还真的就是那样。也就是在析构时对于static变量，先调用A::~A(),再调用B::~B()。因为B::~B()中用到了A::f()，也就是说用到了一个析构了的类，那出问题也就不奇怪了。
那样是怎么才可以知道0x6010a0和0x4009ae是A相关的地址啊？好吧，看看下面：

		breakpoint 3, A::~A (this=0x6010a0 <A::getInstance()::a>, __in_chrg=<optimized out>) at main.cpp:24
		24           delete m_a;
		(gdb) disas
		Dump of assembler code for function A::~A():
		   0x00000000004009ae <+0>: push   %rbp
		   0x00000000004009af <+1>: mov    %rsp,%rbp
		   0x00000000004009b2 <+4>: sub    $0x10,%rsp
		   0x00000000004009b6 <+8>: mov    %rdi,-0x8(%rbp)
		=> 0x00000000004009ba <+12>:  mov    -0x8(%rbp),%rax
		   0x00000000004009be <+16>:  mov    (%rax),%rax
		   0x00000000004009c1 <+19>:  mov    %rax,%rdi
		   0x00000000004009c4 <+22>:  callq  0x400780 <_ZdlPv@plt>
		   0x00000000004009c9 <+27>:  mov    -0x8(%rbp),%rax
		   0x00000000004009cd <+31>:  movq   $0x0,(%rax)
		   0x00000000004009d4 <+38>:  mov    $0x400b69,%edi
		   0x00000000004009d9 <+43>:  callq  0x400770 <puts@plt>
		   0x00000000004009de <+48>:  leaveq 
		   0x00000000004009df <+49>:  retq   
		End of assembler dump.

从上面第一行可以看出A::a的地址是0x6010a0，A::~A()的地址是0x00000000004009ae。同理，B的也类似可以得到。
不妨继续看下去，看看在调用A和B析构函数时会发生什么事：

		//Dump of assembler code for function B::~B():
		   0x0000000000400a92 <+0>: push   %rbp
		   0x0000000000400a93 <+1>: mov    %rsp,%rbp
		   0x0000000000400a96 <+4>: sub    $0x10,%rsp
		   0x0000000000400a9a <+8>: mov    %rdi,-0x8(%rbp)
		   0x0000000000400a9e <+12>:  mov    $0x400b64,%edi
		   0x0000000000400aa3 <+17>:  callq  0x400770 <puts@plt>
		   0x0000000000400aa8 <+22>:  callq  0x4008fd <A::getInstance()>  // 0, call A::getInstance
		   0x0000000000400aad <+27>:  mov    %rax,%rdi                    // 1, put static A::a address to %rdi
		   0x0000000000400ab0 <+30>:  callq  0x4009e0 <A::f()>            // 2, call A::f()
		   0x0000000000400ab5 <+35>:  leaveq 
		   0x0000000000400ab6 <+36>:  retq   
		//End of assembler dump.

		Dump of assembler code for function A::f():
		   0x00000000004009e0 <+0>: push   %rbp
		   0x00000000004009e1 <+1>: mov    %rsp,%rbp
		   0x00000000004009e4 <+4>: mov    %rdi,-0x8(%rbp)    // 0, save &A::a to -0x8(%rbp)
		   0x00000000004009e8 <+8>: mov    -0x8(%rbp),%rax    // 1, save &A::a to %rax
		=> 0x00000000004009ec <+12>:  mov    (%rax),%rax      // 2, save *(&A::a) to %rax
		   0x00000000004009ef <+15>:  movl   $0x64,(%rax)     // 3, save 0x64 (==100) to *(&A::a) 
		   0x00000000004009f5 <+21>:  pop    %rbp
		   0x00000000004009f6 <+22>:  retq   
		End of assembler dump.
		(gdb) p $rax
		$1 = 6295712
		(gdb) stepi
		0x00000000004009ef  32       }
		(gdb) p $rax
		$2 = 0

如上面代码写的注释，第1步里面，将A::a的地址传给了rax寄存器。什么，这个A::a不是析构了吗？怎么空间还在的？我在0x00000000004009ec处做了一个断点，在rax还没有改变是打印出其值，即A::a的地址，结果是：6295712，是的，这货还在啊。当我们继续执行下一条指令，即上面所标的第2步时，是将A::a地址内的值传给rax寄存器，这时再打印出其值，我们发现，这次rax的值为0.再执行第3步就是往这个0地址内写入0x64,即100.再转为易于理解的Ｃ++代码就是:
// we know that:   A::a == NULL； but we want to execute:
*(A::a) = 100;
这样的程序能不core吗？

如果我们将A::f改为下面的样子:

		void f()
		{
		  m_a = (int *)100;
		  printf("A::f m_a == %p\n", m_a);
		}

输出结果:

		> ./a.out
		B
		A
		~A
		~B
		A::f m_a == 0x64

这下不core了啊。是的，static的内存空间一直在啊。只是其变量m_a指向的空间被释放掉了，所以core的。其实这个程序的static变量空间分配在程序的 bss section里面，并非动态分配的，程序存在时，其空间也是一直存在的。

最后再回来之前的问题来，如果是atexit导致的问题，那么只要我们首先使用Ａ::a，那么应该就不会core才对。换回程序的最初始版本,将main函数改为以下形式：

		int main()
		{
		  A::getInstance();
		  B::getInstance();
		  return 0;
		}

结果为：

		> ./a.out 
		A
		B
		~B
		~A

也是没有core的

最后，通过上面的分析，你大概应该也明白C++ class static variable是
1, 怎样分配空间
2, 怎样构造
3, 怎样析构了的吧？


