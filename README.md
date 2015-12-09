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


void f()
{
  m_a = (int *)100;
  printf("A::f m_a == %p\n", m_a);
}

