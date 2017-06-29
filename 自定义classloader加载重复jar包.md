# 工具类说明
    说明：这个类主要是解决加载多个jar包的时候防止jar包冲突的问题，就像jsp在web服务器运行时做修改，不需要重新启动，其实等于一个jsp有两个版本，被后一个版本覆盖了。
    
    
- **核心方法介绍**
- **loadJar**
- **loadClass（）**


-------------------

## loadJar

```
public void loadJar(String path) throws IOException, ClassNotFoundException {
		File root = new File(path);
		File[] files = root.listFiles();
		for (File file : files) {
			if (file.isDirectory() || !file.getAbsolutePath().endsWith(".jar"))
				continue;

			currentJar = new JarFile(file.getAbsolutePath());
			HashMap<String, ArrayList<String>> packages = new HashMap<String, ArrayList<String>>();
			Enumeration<JarEntry> entry = currentJar.entries();

			while (entry.hasMoreElements()) {

				JarEntry jarEntry = entry.nextElement();
				if (jarEntry.isDirectory() || !jarEntry.getName().endsWith(".class"))
					continue;
				String name = jarEntry.getName().replace(".class", "").replace("/", ".");
				int pos = name.lastIndexOf(".");
				String className = name.substring(pos + 1);
				String packageName = name.substring(0, pos);

				ArrayList<String> classes = packages.get(packageName);
				if (classes == null)
					classes = new ArrayList<String>();
				classes.add(className);
				packages.put(packageName, classes);
				loadClass(name,true );
			}
			jarinfo.put(file.getName(), packages);
		}
	}
```
    这个方法主要是遍历所有的jar里的class文件，并且交给loadClass加载，这个类是一个自定义类加载器，顾名思义要继承classLoader。
    
JarFile:是管理jar包的一个文件流。

## loadClass
方法：

```
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
		try {
			Class<?> c = null;
			c = findLoadedClass(name);
			if(c!=null){
				if(resolve){
					resolveClass(c);
					return c;
				}
			}
			String path = name.replace('.', '/').concat(".class");
			if (null != currentJar) {
				JarEntry entry = currentJar.getJarEntry(path);
				if (null != entry) {
					InputStream is = currentJar.getInputStream(entry);
					int availableLen = is.available();
					int len = 0;
					byte[] bt1 = new byte[availableLen];
					while (len < availableLen) {
						len += is.read(bt1, len, availableLen - len);
					}
					c = defineClass(name, bt1, 0, bt1.length);
					if (resolve) {
						resolveClass(c);
					}
				} else {
					if (parent != null) {
						return parent.loadClass(name);
					}
				}
			}
			return c;
		} catch (IOException e) {
			throw new ClassNotFoundException("Class " + name + " not found.");
		}
	}
```
    这个方法主要是解决类当类重复的时候，重新加载原来的类。
```
		if(c!=null){
			if(resolve){
				resolveClass(c);
				  return c;
				}
			}
```
     这个判断语句就是判断类是否已经被加载，如果是，重新加载。

```
				if (parent != null) {
						return parent.loadClass(name);
					}
```
这个判断语句主要解决是系统类的时候，比如说Runnable、Thread的时候，这些事jdk自带的类，所以交给ApplicationClassLoader进行双亲委派机制去加载。
