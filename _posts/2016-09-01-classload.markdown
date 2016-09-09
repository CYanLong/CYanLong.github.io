---
title: "jvm类加载机制"
date: "2016-09-01"
tags: [jvm]
---

在Class文件中描述的各种信息最终要加载到虚拟机中之后才能运行和使用。类中被加载到虚拟机内存到卸载出内存，它的整个生命周期包括：`加载(Loading)`， `验证(Verification)`，`准备（Preparation）`，`解析(Resolution)`，`初始化(Initialization)` `使用` 和 `卸载(reload)` 7个阶段。

其中， `加载`， `验证`，`准备`，`初始化`这4个步骤顺序是一致的，`符号解析` 可能在初始化前完成，也可能在运行时解析（由于虚方法的运行时绑定）。

Java虚拟机规范并没有进行强制约束 `在什么情况下开始类加载过程`，这点虚拟机实现可以自由把握。虚拟机只规定了5种 `有且仅有` 的情况下会进行类的初始化。（类初始化是类加载阶段的最后一步，就是执行 `<clinit>` 方法）。下面我们主要来看一看常见的三种情况：
	
	1.遇到 `new`, `getstatic`, `putstatic`, `invokestatic` 这4条指令时，如果类没有进行初始化，则需要先触发其初始化。
	
	2.当初始化一个类的时候，如果发现其父类还没有初始化，则需要先触发其父类的初始化。
	
	3.当虚拟机启动时，用户需要指定一个要执行的主类，虚拟机会先初始化这个主类。

下面是一个有趣的关于 `主动引用` 和 `被动引用` 的例子

	class SuperClass{
		public static int value = 123;
		
		static{
			System.out.println("SuperClass init");
		}
	}

	class SubClass extends SuperClass{
		static{
			System.out.println("SubClass init");
		}
	}

	public class NotInit{
		
		public static void main(String[] args){
			int i = SubClass.value; //通过子类类型去引用父类static字段
		} 
	}


运行后我们发现，只有 `SubClass init`，即父类并没有进行初始化，即使使用了父类的静态字段。

其实，我们看看字节码指令就很好理解了：
	
	Code:
      stack=1, locals=2, args_size=1
         0: getstatic     #2                  // Field SubClass.value:I
         3: istore_1
         4: return


我们看到 `getstatic` 后面的字段符号引用是 `SubClass`，所以实际上 `SuperClass` 并没有遇到 `getstatic` 指令，也就不会进行初始化。


另外： `java` 命令的 `-XX:+TraceClassLoading` 参数可以追踪到类的加载(laod) 情况，我们会发现，当运行这个程序时，虽然 `SuperClass` 没有完成初始化，但进行了加载。

---
<H4>加载</H4>
---

`加载` 是 `类加载(Class Loading)` 过程的第一个阶段，主要完成这三件事情：

1.通过一个类的全限定类名来获取定义此类的二进制字节流。

2.将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。

3.在内存中生成一个代表这个类的 `java.lang.Class` 对象，作为方法区这个类的各种数据的访问入口。

对于 `HotSpot` 虚拟机而言，Class 对象比较特殊，它虽然是对象，但是存放在方法区中。这个对象将作为程序访问方法区中的这些类型数据的外部接口。

---
<H4>类加载器</H4>

虚拟机设计团队把类加载阶段中的 "通过一个类的全限定名来获取描述此类的二进制字节流" 这个动作放到 Java 虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。 


---
<H3>类与类加载器</H3>
---
类加载器除了执行加载类的动作之外，还用来确立一个类名称空间。 对于任意一个类，都需要由加载它的类加载器和这个类本身一同
