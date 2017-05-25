# 简单实现代码提交校验

现在很多面试页面中或者ACM中会有代码提交校验你的结果正确性，因此我也打算自己简单实现一下这样的效果。


    我简单的讲下我的思路，执行外部传过来的代表一个Java代码，然后通过  进行编译成class文件字节数组，通过JavaClassExecuter处理字节数组，并且把System.out替换成自定义的输出类。

    
- **简单的类介绍**
- **替换后比较**



-------------------

## 简单的类介绍

    先罗列出我的一些类 
    1、	自定义类加载器
    2、	自定义jdk编译器
    3、	自定义System替换java代码中的system.out
    4、	ByteUtils int<->byte string<->byte之间的转换
    5、	一个modifier（修改常量池中CONSTANT_Utf8_info常量的内容），从字节上做替换常量池指向的类
    6、	最后一个类做协调处理
![这里写图片描述](http://img.blog.csdn.net/20170525183824844?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzg3MjQyOTU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

首先看下Test类

```
public static void main(String[] args) throws Throwable {
		StringBuilder sb = new StringBuilder();
		sb.append("package Dongshuai;").append("\n");
		sb.append("public class Test1{").append("\n");
		sb.append("    public static void main(String[]args){").append("\n");
		sb.append("System.out.println(\"Hello Wolrd\");").append("\n");
		sb.append("    }").append("\n");
		sb.append("}").append("\n");
		//初始化编译器
		UpicJdkCompiler jdkCompiler = new UpicJdkCompiler();
		//自定义包名类名去加载，并返回加载好后的class字节数组
		byte[] clazzByte = jdkCompiler.doCompile("Dongshuai.Test1", sb.toString());
		//把字节数组交给处理类替换system.out
		String execute = JavaClassExecuter.execute(clazzByte);
		System.out.println(execute.equals("Hello Wolrd")?Boolean.TRUE:Boolean.FALSE);	}


```
 这个类主要是拼装一个java类的伪代码，然后交给写好的编译器编译。编译好后拿到字节数组，交给JavaClassExcuter作处理。最后拿到运行main方法所输出的值做对比，这样就类似与ACM中校验答案正确性（我这个只是非常简单的思路，也可以做成web应用，我这边就简便了下）。
 
2、看下JavaClassExecuter类
``` 
/**
	 * 执行外部传过来的代表一个Java类的byte数组
	 * 将输入类的byte数组中代表java.lang.System的CONSTANT_Utf8_info常量修改为劫持后的HackSystem类
	 * 执行方法为该类的static main(String[]args)方法,输出结果为该类向System.out/err输出的信息
	 * 
	 * @param classByte代表一个Java类的byte数组
	 * @return执行结果
	 */
	public static String execute(byte[]classByte) {
//		byte[]classByte=objectToBytes(clazzLoad);
		HackSystem.clearBuffer();
		ClassModifier cm = new ClassModifier(classByte);
		byte[] modiBytes = cm.modifyUTF8Constant("java/lang/System", "com/upic/System/HackSystem");
		UpicClassLoader loader = new UpicClassLoader();
		Class<?> clazz = loader.loadByte(modiBytes);
		try {
			//为了查看替换后的class文件
//			FileOutputStream f=new FileOutputStream("F:\\java\\Java tools\\java_基础\\newTest.class");
//			f.write(modiBytes);
//			f.close();
			//执行main方法
			Method method = clazz.getMethod("main", new Class[] { String[].class });
			method.invoke(null, new String[] { null });
		} catch (Throwable e) {
			e.printStackTrace(HackSystem.out);
		}
		return HackSystem.getBufferString();
	}

```
这个类就不详细解释了，上面注释应该写的比较清楚了。

3、我们看下HackSystem
``` 
public final static InputStream in = System.in;
	private static ByteArrayOutputStream buffer = new ByteArrayOutputStream();
	public final static PrintStream out = new PrintStream(buffer);
	public final static PrintStream err = out;

	public static String getBufferString() {
		return buffer.toString();
	}

	public static void clearBuffer() {
		buffer.reset();
	}
//.............和system类似
```
这里展示了部分代码，主要是把原来的System替换掉，把结果保存在ByteArrayOutputStream里边方便拿取。



4、UpicJdkCompiler
构造器 初始化
``` 
public UpicJdkCompiler() {
	compiler =ToolProvider.getSystemJavaCompiler();
	options = new ArrayList<String>()；
	final ClassLoader loader =     Thread.currentThread().getContextClassLoader();
	classLoader = new InnerUpicClassLoader(loader);
	StandardJavaFileManager=manager=compiler.getStandardFileManager(diagnosticCollector, null, null);
	
javaFileManager=newJavaFileManagerImpl(manager,classLoader);
	}

```
主要做初始化所需要用到的工具类

方法doCompile(….)
``` 
public byte[] doCompile(String name, String sourceCode) throws Throwable {
		int i = name.lastIndexOf('.');
		// 获得包名
		String packageName = i < 0 ? "" : name.substring(0, i);
		// 获得类名
		String className = i < 0 ? name : name.substring(i + 1);
		JavaFileObjectImpl javaFileObject = new JavaFileObjectImpl(className, sourceCode);
		javaFileManager.putFileForInput(StandardLocation.SOURCE_PATH, packageName, className + EXT_JAVA,
				javaFileObject);
				Boolean result = compiler
				.getTask(null, javaFileManager, diagnosticCollector, options, null, Arrays.asList(javaFileObject))
				.call();
		if (result == null || !result) {
			throw new IllegalStateException(
					"Compilation failed. class: " + name + ", diagnostics: " + diagnosticCollector);
		}
		return classes.get(name)!=null?((JavaFileObjectImpl)classes.get(name)).getByteCode():null;
	}
/**
……
**/

```
这个类主要把java伪代码编译成class文件并放入流中，并且返回字节数组,关于编译可以去参考javaAPI文档。

观察输出结果：true。

## 替换后比较

>我们现在就来做个测试观察下他的字节码的替换

这是原来的类
``` python
public class Test1 {
	public static void main(String[] args) {
		System.out.println("Hello World");
	}
}

``` 
我们查看他的常量池结构

```
 #1 = Methodref          #6.#15         // java/lang/Object."<init>":()V
   #2 = Fieldref           #16.#17        // java/lang/System.out:Ljava/io/Print
Stream;

```
这时候的常量池#2是指向java/lang/System.out:Ljava/io/PrintStream

然后我们再把下面代码去掉注释

```
//			FileOutputStream f=new FileOutputStream("F:\\java\\Java tools\\java_基础\\newTest.class");
//			f.write(modiBytes);
//			f.close();

```
再次运行程序
找到目录下的class文件，并且查看他的结构

```
 #1 = Methodref          #6.#15         // java/lang/Object."<init>":()V
   #2 = Fieldref           #16.#17        // com/upic/System/HackSystem.out:Ljav
a/io/PrintStream;

```
就会发现，原来的System已经被替换，所以能获取用户输出的结果。


个人对使用java原生态的编译器的意见：
tool.jar包中的大多数类和接口都定义了编译器（和一般工具）的 API，但最好不要在应用程序中使用接口 DiagnosticListener、JavaFileManager、FileObject 和 JavaFileObject。应该实现这些接口，用于为编译器提供自定义服务，从而定义编译器的 SPI。

**这是我个人的看法和一个简单的实现，如果大家有什么好的想法多多留言交流哈**

