# jvm是怎么样启动的

当输入在命令行输入 `java.exe [Java虚拟机参数] YourClassName [应用程序参数]` 时，编写的Java程序便会运行。从jdk8源码来说，会经历以下过程。

## 虚拟机外部

### java.c

java.exe文件是通过编译录src\java.base\share\native\libjli\java.c文件生成的可执行文件。

### JLI_Launch

在java.c文件当中存在JLI_Launch方法。其中JLI是Java Launcher InterfaceJLI_Launch 是 Java Launcher Interface（JLI）的启动函数的名称。

```c_cpp
JNIEXPORT int JNICALL
JLI_Launch(int argc, char **argv,                 /* 主程序命令行参数 */
           int jargc, const char** jargv,          /* Java 参数 */
           int appclassc, const char** appclassv,  /* 应用程序类路径 */
           const char* fullversion,                /* 完整版本定义 */
           const char* dotversion,                 /* 未使用的点版本定义 */
           const char* pname,                      /* 程序名 */
           const char* lname,                      /* 启动器名 */
           jboolean javaargs,                      /* JAVA_ARGS */
           jboolean cpwildcard,                    /* 类路径通配符 */
           jboolean javaw,                         /* 仅限 Windows 的 javaw */
           jint ergo                               /* 未使用 */
)
{
    // 初始化与jvm运行有关的变量
    ...
    // 解析命令行参数
    ...
  // 调用 JVMInit 函数，该函数在后续会初始化 Java 虚拟机，加载主运行类，获取 main 方法 ID，并最终调用 Java 的 main 方法。
    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

Java Launcher Interface 是 Java 虚拟机（JVM）提供的一组接口，用于启动 Java 应用程序。
这个接口允许 Java 虚拟机与底层系统进行交互，执行 Java 应用程序的启动和初始化操作。

在这里，JLI_Launch 函数是 JLI 接口的入口点，用于启动 Java 应用程序。该函数接受多个参数，包括命令行参数、Java 虚拟机参数、应用程序类路径等。
当你运行“java.exe YourClassName”这个命令时，调用的就是JLI_Launch这个方法。通过调用这个函数，Java 虚拟机开始执行 Java 应用程序。

### JVMInit

作为启动流程的最后一步，JVMInit 调用虚拟机的入口函数，开始 Java 程序的执行。

以下是src\java.base\windows\native\libjli\java_md.c#JVMInit的调用链:

```c_cpp
JVMInit(InvocationFunctions* ifn, jlong threadStackSize,int argc, char **argv,int mode, char *what, int ret){ 
  // 调用ContinueInNewThread
    return ContinueInNewThread(ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

```c_cpp
ContinueInNewThread(ifn, threadStackSize, argc, argv, mode, what, ret);{
    ...
    // 调用CallJavaMainInNewThread
    rslt = CallJavaMainInNewThread(threadStackSize, (void*)&args);
    return (ret != 0) ? ret : rslt;
}
```

```c_cpp
CallJavaMainInNewThread(threadStackSize, (void*)&args){
    ...
    if (thread_handle) {
        ...
    } else {
        // 调用JavaMain
        rslt = JavaMain(args);
    }
    ...
    return rslt;
}
```

以上函数的作用是在JVMInit中使用新线程调用javaMain

### JavaMain

java.c#JavaMain这个方法是如何调用main方法的:

```c_cpp
int JavaMain(void* _args)
{
    JavaMainArgs *args = (JavaMainArgs *)_args;
    ...
    //初始化Java虚拟机
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }
    ...
    //加载主运行类
    mainClass = LoadMainClass(env, mode, what);
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
    ...  
    //通过加载的主运行类，获取main方法
    mainID = (*env)->GetStaticMethodID(env, mainClass,"main","([Ljava/lang/String;)V");   
    CHECK_EXCEPTION_NULL_LEAVE(mainID);
    
    //通过JNI调用main函数
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);
    ...
    LEAVE();
}
```

JNI（Java Native Interface）是 Java 平台提供的一种编程框架，用于实现 Java 应用程序与本地（非 Java）应用程序之间的相互调用。JNI 允许 Java 代码调用本地代码（通常是用 C 或 C++ 编写的），反之亦然。这里涉及到jvm中的本地方法栈，后面会说明。

### LoadMainClass

java.c#LoadMainClass在给定的 JNIEnv（Java 环境）中加载主运行类。

```c_cpp
LoadMainClass(JNIEnv *env, int mode, char *name)
{
    ...
    // 获取 LauncherHelper 类  sun/launcher/LauncherHelper(java类)
    jclass cls = GetLauncherHelperClass(env);
    ...
    // 获取 LauncherHelper 的静态方法 checkAndLoadMain 方法 赋值给mid
    NULL_CHECK0(mid = (*env)->GetStaticMethodID(env, cls,
                "checkAndLoadMain",
                "(ZILjava/lang/String;)Ljava/lang/Class;"));
    ...
    // 调用 LauncherHelper 的静态方法 checkAndLoadMain 方法 即（mid方法）
    CHECK_JNI_RETURN_0(
        result = (*env)->CallStaticObjectMethod(
            env, cls, mid, USE_STDERR, mode, str));
    ...
    return (jclass)result;
}
```

**至此，Java虚拟机便完成了启动到调用`public static void main(String[] args) {...}`所需要的外部准备工作**

## 虚拟机内部

### LauncherHelper

当jvm通过jni调用 LauncherHelper 的静态方法 checkAndLoadMain 方法时。程序便从虚拟机外部跳转到虚拟机内部。

### checkAndLoadMain

checkAndLoadMain方法的作用是检查并获取Main方法所在的Java类即Class<?>。

sun/launcher/LauncherHelper#checkAndLoadMain

```java
 public static Class<?> checkAndLoadMain是获取main(boolean printToStderr,方法所在Java Class类
                                        int mode,
                                        String what) {
    ...
    switch (mode) {
        //从Class文件中获取主类
        case LM_CLASS:
            cn = what;
            break;
        //从jar包中获取主类
        case LM_JAR:
            cn = getMainClassFromJar(what);
            break;
        default:
            // should never happen
            throw new InternalError("" + mode + ": Unknown launch mode");
    }
    // 例如java.lang.String
    cn = cn.replace('/', '.');
    Class<?> mainClass = null;
    try {
        // LauncherHelper内部定义有静态变量scloader
        // private static final ClassLoader scloader = ClassLoader.getSystemClassLoader();
        // scloader是系统类加载器实例也就是应用类加载器(AppClassLoader)的实例。
        // 可知jvm使用应用类(AppClassLoader)加载器来加载main方法所在类。
        mainClass = scloader.loadClass(cn);
    } catch (NoClassDefFoundError | ClassNotFoundException cnfe) {
        // 在macOS上获取mainClass
    }
    ...
    // 检查是否用JavaFx启动mainClass
    ...
    validateMainClass(mainClass);
    return mainClass;
}
```

### getSystemClassLoader和initSystemClassLoader

java/lang/ClassLoader.java#getSystemClassLoader

系统类加载器的初始化:

```java
public static ClassLoader getSystemClassLoader() {
    initSystemClassLoader();
    // private static ClassLoader scl;
      // scl是系统类加载器
    if (scl == null) {
        return null;
    }
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkClassLoaderPermission(scl, Reflection.getCallerClass());
    }
    return scl;
}
```

java/lang/ClassLoader.java#initSystemClassLoader

```java
private static synchronized void initSystemClassLoader() {
    if (!sclSet) {
        if (scl != null)
            throw new IllegalStateException("recursive invocation");
        // 初始化sun.misc.Launcher类，为了得到Launcher类中的类加载器
        sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
        if (l != null) {
            Throwable oops = null; 
            // 获取类加载器，由下面的代码可知：此加载器即系统类加载器，也就是应用类加载器
            scl = l.getClassLoader();
            try {
                scl = AccessController.doPrivileged(
                  // 设置系统类加载器
                    new SystemClassLoaderAction(scl));
            } catch (PrivilegedActionException pae) {
        ...
            }
        ...
        }
        sclSet = true;
    }
}
```

### Launcher

sun/misc/Launcher.java

sun/misc/Launcher$ExtClassLoader

sun/misc/Launcher$AppClassLoader

这些类加载器的具体加载范围在双亲委派原则和类加载器中细讲。

sun.misc.Launcher.java，是Java虚拟机启动后的入口类，或者说它是由C++程序（其实主要就是BootstrapClassLoader）切入到Java程序的入口类，上面说的sun.launcher.LauncherHelper类也是为了引出这个类。

可以写一个简单的代码进行验证

```java
public class App {
    public static void main(String[] args) {
        ClassLoader classLoader = Launcher.class.getClassLoader();
    }
}
// 输出为null，说明Launcher确实是BootstrapClassLoader加载的
```

```java
个双重检查创建了一个线程安全的单例的ExtClassLoaderURLClassLoaderURLClassLoaderAppClassLoaderAppClassLoader
/**
 * This class is used by the system to launch the main application.
Launcher */
public class Launcher {
    private static URLStreamHandlerFactory factory = new Factory();
    private static Launcher launcher = new Launcher();
    private static String bootClassPath =
        System.getProperty("sun.boot.class.path");
 
    // 在上面initSystemClassLoader中调用
    public static Launcher getLauncher() {
        return launcher;
    }
 
    // main方法所在类的加载器
    private ClassLoader loader;
 
    // Launcher构造方法
    public Launcher() {
        // Create the extension class loader
        ClassLoader extcl;
        try {
            //获取扩展类加载器
            extcl = ExtClassLoader.getExtClassLoader();
        } catch (IOException e) {
            throw new InternalError(
                "Could not create extension class loader", e);
        }
 
        // Now create the class loader to use to launch the application
        try {
            // 获取应用类加载器，并将extcl即扩展类加载器作为父类加载器传参
            loader = AppClassLoader.getAppClassLoader(extcl);
        } catch (IOException e) {
            throw new InternalError(
                "Could not create application class loader", e);
        }
 
        // 设置线程上下文类加载器，为了给SPI使用
        Thread.currentThread().setContextClassLoader(loader);
        ...
    }
    
    // 在上面getSystemClassLoader()->initSystemClassLoader()中最终返回的用来加载main方法的类加载器
    public ClassLoader getClassLoader() {
        return loader;
    }

    /*
     * The class loader used for loading installed extensions.
     */
    static class ExtClassLoader extends URLClassLoader {
        static {
            ClassLoader.registerAsParallelCapable();
        }
        private static volatile ExtClassLoader instance = null;

        // 利用了一个双重检查创建了一个线程安全的单例的ExtClassLoader
        public static ExtClassLoader getExtClassLoader() throws IOException
        {
            if (instance == null) {
                synchronized(ExtClassLoader.class) {
                    if (instance == null) {
                        instance = createExtClassLoader();
                    }
                }
            }
            return instance;
        }
        
        // 构造方法，其中URLClassLoader的构造方法是public URLClassLoader(URL[] urls, ClassLoader parent, URLStreamHandlerFactory factory) 
        // super(getExtURLs(dirs), null, factory)导致ExtClassLoader的parent类类加载器为null
        public ExtClassLoader(File[] dirs) throws IOException {
            super(getExtURLs(dirs), null, factory);
            SharedSecrets.getJavaNetAccess().
                getURLClassPath(this).initLookupCache(this);
        }
    }
    
    static class AppClassLoader extends URLClassLoader {
        static {
            ClassLoader.registerAsParallelCapable();
        }
        // 在Luncher构造方法中调用AppClassLoader.getAppClassLoader(extcl)->return new AppClassLoader(urls, extcl)
        // 最终设置AppClassLoader的parent类加载器为extcl(ExtClassLoader)
        AppClassLoader(URL[] urls, ClassLoader parent) {
            super(urls, parent, factory);
            ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);
            ucp.initLookupCache(this);
        }

        // 获取AppClassLoader并设置AppClassLoader的parent为extcl
        public static ClassLoader getAppClassLoader(final ClassLoader extcl)
            throws IOException
        {
            final String s = System.getProperty("java.class.path");
            final File[] path = (s == null) ? new File[0] : getClassPath(s);
            return AccessController.doPrivileged(
                new PrivilegedAction<AppClassLoader>() {
                    public AppClassLoader run() {
                    URL[] urls =
                        (s == null) ? new URL[0] : pathToURLs(path);
                    // 返回AppClassLoader
                    return new AppClassLoader(urls, extcl);
                }
            });
        }
    }
}
```

以上代码看着可能有些眼花缭乱，其实逻辑很简单，总结下来，就是依次启动三类加载器：启动类加载器、扩展类加载器、应用类加载器，最后由应用类加载器来加载当前类，然后遵循双亲委派模型，依次委派父类加载器进行加载。主要步骤如下：

- 由C++代码（启动类加载器）加载sun/launcher/LauncherHelper类，然后执行这个类里面的checkAndLoadMain方法；
- 其实说白了，启动类加载器启动之后，首先会去加载主类，怎么去找到这个主类呢？那就是sun/launcher/LauncherHelper这个类要干的事，所以启动类加载器会去加载这个类，而在其checkAndLoadMain方法中，会去加载main方法所在的类；
- 既然要加载main类，那么由谁来加载呢？在checkAndLoadMain方法中已经明确说了：由系统类加载器来负责加载，可能你又要问：啥是系统类加载器？我个人理解因为它是ClassLoader类中的getSystemClassLoader()方法得到的类加载器，顾名思义，这个类也被称为“系统类加载器”，而在getSystemClassLoader()方法中会去创建一个Launcher对象，获取系统类加载器的主要逻辑就在Launcher的构造方法里；
- 在Launcher()方法里，才有了上面所说的几句关键代码，最终就是由 AppClassLoader.getAppClassLoader(extcl)方法所得到的加载器（应用类加载器）来加载main方法所在类。
- 在上面代码中，设置一个类加载器的parent并不是说明二者是继承关系。相反AppClassLoader和ExtClassLoader都继承自URLClassLoader。
- ExtClassLoader使用双重检查创建了一个线程安全的单例的ExtClassLoader。
- 在jdk9及以后的版本中ExtClassLoader由ClassLoaders$PlatformClassLoader，即平台类加载器替代  
  1. 在JDK8中的这个Extension ClassLoader，主要用于加载jre环境下的lib下的ext下的jar包。当想要扩展Java的功能的时候， 把jar包放到这个ext文件夹下。然而这样的做法并不安全，不提倡使用。  
  2. 这种扩展机制被JDK9开始加入的“模块化开发”的天然的扩展能力所取代。  
- Java应用运行时的初始线程的上下文类加载器时系统类加载器。在线程中运行的代码可以通过该类加载器来加载类与资源。
正常情况下，线程执行到某个类的时候，只能看到这个类对应加载器所加载的类。但是你可以为当前线程设置一个类加载器，然后可视范围就增加多一个类加载器加载的类。
调SPI接口的方法依赖外部JAR包用应用类加载器加载，父加载器访问不到子加载器的类。但是可以设置当前线程的上下文类加载器，把当前线程上下文类加载器加载的类一并纳入可视范围。当高层提供了统一的接口让低层去实现，同时又要在高层加载（实例化）底层的类时，就必须要通过线程上下文类加载器来帮助高层的ClassLoader找到并加载该类。
