---
title: "方法调用"
date: "2016-08-31"
tags: [jvm]
---

方法调用并不等同于方法执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法）。我们知道，`Class` 文件的编译过程中不包含传统编译中的连接步骤，一切方法调用在 `Class` 文件里面储存的都只是符号引用，而不是方法在实际运行时的内存布局的入口地址。这使得在调用方法时，要等到类加载期间，甚至到运行期间才能确定目标方法的直接引用。

---
<H4>方法调用</h4>
---

首先，我们来看一看有关方法调用的字节码指令：

1.`invokestatic`：调用静态方法。

2.`invokespecial`：调用实例构造器<init>方法，私有方法和父类方法（`super关键字`）。

3.`invokevirtual`：调用所有的虚方法。

4.`invokeinterface`	：调用接口方法。

（注：还有一个较为特殊的 `invokedynamic` 指令。后面单独介绍）


上面的指令都是用来进行方法调用的指每个指令是相似的令，它们都有自己不同的应用场景。但，比如：每个指令后面都会跟一个指向常量池的 `Methodref_info` 类型的方法符号表，用来表明要调用的方法的信息（方法名和描述符）。

所有方法调用中的 `目标方法` 在Class文件里面都是一个常量池的符号引用，在类加载的解析阶段，会有一部分符号引用转化为直接引用，还有一部分会在程序运行时才进行解析得到目标方法的入口内存地址。

也就是说，不管怎么样，方法调用前必须根据符号引用找到真正的方法指令的内存地址，这个过程叫做符号解析。它们的最大的不同在于何时进行符号的解析。

---
<h4>invokestatic 和 invokespecial</h4>
---

 由于 `静态方法`，`私有方法`，`实例构造方法`，`父类方法` 这4类方法无法通过继承或别的方式重写，所以 `invokestatic` 和 `invokespecial` 的解析过程很适合在类加载阶段（类已完成装载，即所有的方法都以载入内存）就完成，即在我们没有真正的方法接收者（当程序执行 `invoke*` 指令时，操作数栈存放着方法的调用者）时就确定方法的入口地址。

下面是关于调用静态方法的一个例子：

	class Demo{
		public static void method(){
			System.out.println(Demo static method);
		}
	}

	public class StaticInvoke{
		public void invoke(){
			Demo.method();
		}
	}

关于 `invoke` 方法的字节码指令：
	
	Code:
	stack=0, locals=1, args_size=1
	0: invokestatic  #15  // Method com/cyl/jvm/bytecode/Demo.method:()V
	3: return 

我们看到，上面 `invokestatic` 指令在运行时并没有目标接受者，并且由于这个方法没有参数和返回值， 方法的操作数栈（`stack` 值） 为0。正是由于静态方法的调用不需要运行时方法接收对象，我们完全可以在类加载阶段当类装载后是就立刻进行解析。


关于 `invokespecial` 的说明：

	public class SpecialInvoke{
		
		private void method(){
			System.out.println("private method");
		}

		public void invoke(){
			SpecialInvoke si = new Son();
			si.method();
		}
	}
	
	class Son{
		public void method(){
		}	
	}


下面是 `invoke` 方法的执行过程：
	
	Code:
		stack=2, locals=2, args_size=1
		 0: new           #30                 // class com/cyl/jvm/bytecode/Son
         3: dup
         4: invokespecial #32                 // Method com/cyl/jvm/bytecode Son."<init>":()V
         7: astore_1
         8: aload_1  #将上面创建的Son对象的引用加载在操作数栈顶
         9: invokespecial #33                 // Method com/cyl/jvm/bytecode/SpecialMethod. method:()V
        12: return


`invokespecial` 指令首先从 `<method-spec>` 中的方法描述符解析得到此方法的参数个数 n个（可能是0个），然后从操作数栈中 `pop` n 个参数。接着 `pop` 出 `objectref`，`objectref` 必须是目标方法表中对应的那个类的实例或者任意一个子类。 **解释器会从目标方法表对应的那个类中寻找此方法。注意：这个的寻找对象不是基于运行时 `objectref` 类型，而是编译时 `invoke*` 指令后面的目标方法表对应的那个类的。** 所以，我们看到，`invokespecial` 也适合在类加载阶段就完成符号解析。


---
<H4>invokeinterface 和 invokevirtual</H4>
---

	public class InterfaceInvoke{
		public void test(List<String> list){
			list.add("abc");
		}	
	}
	
	//invoke method info:
	Code:
      stack=2, locals=2, args_size=2
         0: aload_1   	#push local variable 1(i.e. the list object)
         1: ldc           #18                 // String abc
         3: invokeinterface #20,  2           // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
         8: pop
         9: return

我们看到，在调用invokeinterface时，会将参数 `list` 加载到操作数栈，方法 `add` 具体的实现依赖于这个运行时的本地变量 `list`，如果这个对象中包含了名称和描述符都和调用接口方法一致的方法，那这个方法就会被调用，查找过程终止。否则，如果 这个对象有父类，顺序递归搜索直接父类，直到找到。如果还没有找到，抛出 `AbstractMethodError`异常。

另外：如果 `objectref` 所对应的对象未实现接口中所需的接口，那么 `invokeinterface` 指令将抛出 `IncompatibleClassChangeError`。


`invokevirtual` 指令和 `invokeinterface` 相似，都要依赖于运行时操作数栈的 `objectref`。
	
	invokevirtual retrieves the Java class for objectref， 
	and searches the list of methods defined by that class 
	and then its superclass, looking for a method called methodname, 
	whose descriptor is descriptor。


---
<H4>类型</H4>
---

静态语言中一个变量有两个阶段可以体现 `类型(type)`，一是在将 `.java` 源码编译成 `.class` 字节码时，具体的 `List list = new ArrayList()  list.add("abc")` 对应的字节码指令 `invokeinterface` 所指向的目标方法描述为： `java/util/List.add(Ljava/lang/Object)V`，**其中前面的 `java/util/List` 正是变量 `编译时（静态）类型` 的体现，并且`javac` 编译器在生成的字节码时当遇到符号引用时是根据变量的声明类型确定的。**   等到这条指令真正运行时，会有一个 `objectref` 对象在操作数栈上，一些指令（invokeinterface, invokevirtual）在执行时并不会依据 `符号引用` 中的类型（这个类型是javac确定的），而是依据那个只有在真正运行到这一步才会得到的位于操作数栈的那个对象，我们叫它运行时（动态）类型。

