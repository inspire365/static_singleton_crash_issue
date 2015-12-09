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

		void f()
		{
		  m_a = (int *)100;
		  printf("A::f m_a == %p\n", m_a);
		}

