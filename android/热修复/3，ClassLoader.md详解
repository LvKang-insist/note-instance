## Java 中的 ClassLoader 回顾

​	

- Bootstrap ClassLoader ：加载虚拟机指定的 class 文件
- Extension ClassLoader：加载虚拟机指定的 class 文件
- App ClassLoader ：加载应用中的class 文件
- Custom ClassLoader ：通过自定义 ClassLoader 加载自己指定的 clas

​			类加载过程：加载，验证，准备，解析，初始化

## Android 中的 ClassLoader 作用详解

###  Android 中 ClassLoader 的种类

- PathClassLoader：加载我们已经安装到系统中的 apk 中的class 文件
- DexClassLoader：加载指定目录中的字节码文件
- BootClassLoader：主要用来加载 framework 层的字节码文件，继承自 ClassLoader
- BaseDexClassLoader：是一个父类，前两个 ClassLoader 都是这个类的子类



我们可以通过一段代码来查找一个最简单的程序中用到了那几个 ClassLoader

```java
ClassLoader loader = getClassLoader();
        if (loader != null) {
            Log.e(TAG, "ClassLoader: " + loader.toString());
            //获取父 ClassLoader
            while (loader.getParent() != null) {
                loader = loader.getParent();
                Log.e(TAG, "Parent ClassLoader: " + loader.toString());
            }
        }
```

```java
2019-11-12 10:21:10.306 4046-4046/? E/MainActivity: ClassLoader: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.testdemo.www.classloader-mNctun4pxQMM04Hsv0AU2A==/base.apk"],nativeLibraryDirectories=[/data/app/com.testdemo.www.classloader-mNctun4pxQMM04Hsv0AU2A==/lib/arm64, /system/lib64, /system/vendor/lib64]]]
2019-11-12 10:21:10.306 4046-4046/? E/MainActivity: Parent ClassLoader: java.lang.BootClassLoader@eea1b40

```

​		从上面我们可以看出一共使用了 PathClassLoader 和 BootClassLoader。这两个也是我们程序运行必须要有的两个 ClassLoader

### Android 中ClassLoader 的特点

​		使用双亲代理模型的特定来加载 class

​		双亲代理模型的特点：在加载一个 class 的时候会询问当前的 ClassLoader 是否加载过该 calss ，如果已经加载过，就直接返回。如果没有加载过，就会寻找当前 ClassLoader 的 Parent 进行加载，相应的 parent 也会判断是否加载过该 calss，如果加载过则进行返回，否则继续向上寻找。如果整个继承路中都没有加载过该 class， 则就会从最高层开始调用findClass从指定的路径进行加载，如果路径中没有，则由子类继续执行此过程。这样的好处就是，一个 class 只要被加载过一次，那么在系统以后的整个生命周期中都不会加载这个 class，大大提高了加载类的效率

​		类加载的共享功能：如果 class 被加载过，后面如果需要用到这个 class，则不会在进行加载，而是可以直接进行使用。

​		类加载的隔离功能：不同继承路线上的 ClassLoader 加载的肯定不是同一个类。好处是，避免用户自己去些一些代码去冒充核心的类库来访问我们核心类库中的成员变量。举个例子：我们系统的类会在初始化的时候加载，比如 String等。这些类是在应用程序启动之前被我们系统加载好的。如果自定义一个String 类来将 String替换掉的话会存在很大的安全问题，现在他会使用自定义的 ClassLoader 来加载自定义的 String。但是他不会成功。 具体就是因为针对java.*开头的类，jvm的实现中已经保证了必须由bootstrp来加载。 

​		验证多个类是同一个类的条件

- 相同的 ClassName
- 相同的 packageName
- 被相同的 ClassLoader 加载

### ClassLoader 源码讲解

看一下重点方法

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
       return loadClass(name, false);
}

protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // 首先检查类是否被加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                       //如果没有加载，就从父类继续找
                       c = parent.loadClass(name, false);
                    } else {
                    	//到这里就说明这个类还没有被加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
				// 如果没有没有加载，就调用 findClass
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
 }
//查找类，这个方法没有做任何实现，很明显是要子类来实现的
protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
}
```

------

下面我们看一下它的实现类

```java
//这个 ClassLoader 用于加载 jar 包 或者 apk 中的 class 文件，可以加载并没有安装到系统的应用中的类
//所以才说 DexClassLoader 才是动态加载的核心
public class DexClassLoader extends BaseDexClassLoader {
	
    //dex 文件路径
    //解压的dex文件存储路径，这个路径必须是一个内部存储路径，一般情况下使用当前应用程序的私有路径：/data/data/<Package Name>/...
    //包含 C/C++ 库的路径集合，多个路径用文件分隔符分隔分割，可以为null。
    //父加载器
    public DexClassLoader(String dexPath, String optimizedDirectory,
           String librarySearchPath, ClassLoader parent) {
        // 可看到 optimizedDirectory 参数并没有被使用，这个参数在 26中被废弃了，
        super(dexPath, null, librarySearchPath, parent);
    }
}
```

DexClassLoader 继承自 BaseDexClassLoader ，方法实现都在 BaseDexClassLoader 中

```java
public class PathClassLoader extends BaseDexClassLoader {
  // dex路径，
  // 父加载器  
  public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
  }   
  
  public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
  }  
}    
```

既然 optimizedDirectory 参数并没有被使用，这个参数在 26中被废弃了，那这样 DexClassLoader 和 PathClassLoader 好像就没有多大区别的吧(个人感觉)

PathClassLoader 继承自 BaseDexClassLoader ,方法的实现都在 BaseDexClassLoader 中。



这两个类其实没有任何的作用，唯一一个的作用就是： BaseDexClassLoader  DexClassLoader可以加载dex、apk、jar、zip等格式的插件， PathClassLoader  只能加载已安装的APK ，**当然这是针对 26 以前而言的**，至于为什么这样说呢，等一会我们就会看到。

**他们正在的行为都是在父类 BaseDexClassLoader 中完成的。接着，我们就看一下 BaseDexClassLoader 是如何来查找 dex 的**。

------

```java
public class BaseDexClassLoader extends ClassLoader {
     // 重要成员变量，
	 private final DexPathList pathList;
    
     public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent, boolean isTrusted) {
        super(parent);
         //初始化 pathList ,传入当前对象，dex 文件路径
         //从构造参数中可以看到 optimizedDirectory 并没有被使用，这个参数在 API 26以后被弃用了
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, isTrusted);

        if (reporter != null) {
            reportClassLoaderChain();
        }
    }
    
    
    //核心方法 
	@Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException(
                    "Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }    
}
```

​	findClass 中的第三行，调用了 pathList 的 findClass，传入了要查找的类名，也就是说当前的方法也不是真正的查找方法，只是一个中转而已。

​	下面我们来看一下 pathList 的 findClass 是如何查找的

```java

final class DexPathList {
	
    //后缀名
	private static final String DEX_SUFFIX = ".dex";
	//从构造方法中传入的 ClassLoader
    private final ClassLoader definingContext;
	// DexPathList 的内部类，等一下在说他
    private Element[] dexElements;
    
    //构造方法
    DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
		......
            
        //初始化    
        this.definingContext = definingContext;
        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        //1, 将dexPath保存为BaseDexClassLoader
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext, isTrusted);
		
    }
}
//切割路径
private static List<File> splitDexPath(String path) {
        return splitPaths(path, false);
}
```

​	首先是 DEX_SUFFIX ，这是一个常量，保存的就是后缀名，然后接着是 dexElements 

​	我们首先看一个静态内部类, Element 就是 dexElements 对象数组存储的具体静态内部类 。

```java
static class Element {
        private final File path;
        private final DexFile dexFile;
        private ClassPathURLStreamHandler urlHandler;
        private boolean initialized;
        
        public Element(File dir, boolean isDirectory, File zip, DexFile dexFile) {
                this.path = dir;
                this.dexFile = null;
                this.path = zip;
                this.dexFile = dexFile;
            }
        } 
}
```

然后我们看一下1, makeDexElements ，这里才是真正的实现

```java
private static Element[] makeDexElements(List<File> files, File optimizedDirectory,
            List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
    
      Element[] elements = new Element[files.size()];
      //下标	
      int elementsPos = 0;
     	//遍历所有的 file 文件
      for (File file : files) {
          //判断是否为 目录
          if (file.isDirectory()) {
              //如果是 目录，则继续进行递归
              elements[elementsPos++] = new Element(file);
           //是否为 文件   
          } else if (file.isFile()) {
              String name = file.getName();
              DexFile dex = null;
              //是否为 .dex 文件
              if (name.endsWith(DEX_SUFFIX)) {
                  // Raw dex file (not inside a zip/jar).
                  try {
                      //如果是 dex 文件，则加载这个文件
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                      if (dex != null) {
                          //将dex 文件保存,注意第二个参数传的底 null 
                          elements[elementsPos++] = new Element(dex, null);
                      }
                  } catch (IOException suppressed) {
                      System.logE("Unable to load dex file: " + file, suppressed);
                      suppressedExceptions.add(suppressed);
                  }
              } else {
                  //如果不是目录且不是 .dex 结尾的，那么他可能是 jar。加载这个文件
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);          
                  if (dex == null) {
                      elements[elementsPos++] = new Element(file);
                  } else {
                      // 保存dex 文件 和 当前的 file，这种情况下可能是 jar，具体可以看这个构造方法
                      elements[elementsPos++] = new Element(dex, file);
                  }
              }
             
          }
      }
     ......
      return elements;
}
```

​	上面对 file 文件进行了遍历，1，首先判断了是否为目录，如果是，则将目录进行保存。2，判断如果是 dex 文件 则  如果是则通过 loadDexFile 加载，3，如果不是目录且不是 .dex 结尾的，则继续加载这个文件，接着判断这个文件是否为 dex，如果不是，保存 file ，如果是 保存 dex 和 file 。

​	下面 看一下 loadDexFile 方法：

```java
   private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader,
                                       Element[] elements)
            throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file, loader, elements);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
        }
}
```

​	如果 optimizedDirectory 为 null ，说明该 file 就是 dex 文件，那么直接创建 DexFile 对象即可。

所以说 makeDexElements 方法就是 通过传入的 files 获取真正的 dex 文件。makeDexElements 方法执行完后，干了什么呢？我们接着往下看：

从 BaseDexClassLoader 的构造方法中创建了 DexPathList对象。 在 DexPathList 的构造方法中 通过调用 makeDexElements 方法获取到 dex 文件，并转化成了 Element 数组，在 BaseDexClassLoader  的 findClass 中 调用了 pathList.findClass ,并且传入了要查找的 class 名字，那么我们来看一下 DexPathList 的 findClass 方法：

```java
//name：要加载的类的名字
public Class<?> findClass(String name, List<Throwable> suppressed) {
    //遍历 Element 数组
        for (Element element : dexElements) {
           // 调用 element 的 findClass 获取class 对象
            Class<?> clazz = element.findClass(name, definingContext, suppressed);
            //不为空直接返回
            if (clazz != null) {
                return clazz;
            }
        }
		......
        return null;
}
```

这里最终会调用到底层的一个方法：

```java
private static native Class defineClassNative(String name, ClassLoader loader, Object cookie,DexFile dexFile)
```

这个方法我们看不了，但是可以推断一下：dex 文件中包含了整个项目的 class 文件，所以在这里，他应该是通过我们传入的 name，在 dex 文件中查找相关的数据，然后拼装成一个 class 最后返回给我们。

o**ptimizedDirectory  不为空的情况**：通过查看了 DexClassLoader 和 PathClassLoader 后我们知道 optimizedDirectory 这个参数传的是 null。所以上面的 else 是不会执行的。但是我们还是看一下这里吧：

```java
 if (optimizedDirectory == null) {
            return new DexFile(file, loader, elements);
            
         //如果不为 null    
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
        }
```

​		DexFile.LoadDex

```java
static DexFile loadDex(String sourcePathName, String outputPathName,
        int flags, ClassLoader loader, DexPathList.Element[] elements) throws IOException {

        return new DexFile(sourcePathName, outputPathName, flags, loader, elements);
    }



private DexFile(String sourceName, String outputName, int flags, ClassLoader loader,
            DexPathList.Element[] elements) throws IOException {
        if (outputName != null) {
            try {
                String parent = new File(outputName).getParent();
                if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                    throw new IllegalArgumentException("Optimized data directory " + parent
                            + " is not owned by the current user. Shared storage cannot protect"
                            + " your application from code injection attacks.");
                }
            } catch (ErrnoException ignored) {
                // assume we'll fail with a more contextual error later
            }
       }

        mCookie = openDexFile(sourceName, outputName, flags, loader, elements);
        mInternalCookie = mCookie;
        mFileName = sourceName;
        //System.out.println("DEX FILE cookie is " + mCookie + " sourceName=" + sourceName + " outputName=" + outputName);
}
```

可以看到这里最终调用的是 openDexFile：

```java
 private static Object openDexFile(String sourceName, String outputName, int flags,
            ClassLoader loader, DexPathList.Element[] elements) throws IOException {
        // Use absolute paths to enable the use of relative paths when testing on host.
        return openDexFileNative(new File(sourceName).getAbsolutePath(),
                                 (outputName == null)
                                     ? null
                                     : new File(outputName).getAbsolutePath(),
                                 flags,
                                 loader,
                                 elements);
 }
  public static native boolean isDexOptNeeded(String fileName)
            throws FileNotFoundException, IOException;
```

​	到这里就结束了，当前有兴趣的看以看一下[这篇文章](https://www.jianshu.com/p/5cf119d7103b)，对 DexClassLoader 和 PathClassLoader 讲的比较详细，



- 有许多组件类需啊哟注册才能使用
- 资源的动态加载很难