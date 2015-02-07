#JNI的作用与工作原理
===========================
本文主要翻译自Java JNI文档。
用接口的方式联接不同的编程语言并非什么新观念。例如，C语言通常可以调用FORTRAN和汇编语言编写的函数。同样的，
Lisp和Smalltalk的实现也支持很多其他语言的接口。  
JNI要解决的问题和其他支持协作机制的语言要解决的问题一样。但是，二者间有个重要的区别：JNI并非为特定的JVM实
现而设计，而是为所有JVM实现都支持的本地接口而设计。我们会在描述JNI设计目标的过程中详细说明这点。

##设计目标
JNI设计的最重要目标，就是支持针对不同本地环境的各种JVM实现之间的二进制兼容。  
**本地环境(Host Environment)**
- 又称为系统环境，有自己的本地库和CPU指令集
- 本地程序使用C/C++这样的本地语言来编写，编写的程序链接本地库，并在本地环境中运行
- 本地程序和本地库都依赖一个特定的本地系统环境  

JNI设计的第二个目标是高效。有时为了满足第一个目标，可能会牺牲一点小利，因此需要在平台无关和效率之间做一些
权衡。

最后，JNI必须是一个完整的体系。它必须公开足够多的JVM功能，以使本地程序完成一些有意义的任务。

##加载本地库
在程序能够调用本地接口之前，虚拟机必须定位和加载那些实现了这些接口的本地库。

###类加载器
本地库是通过类加载器来定位的。  
**类加载器(Class Loader)**
- Java类加载器基于三个机制：委托、可见性和单一性
- 委托机制是指将加载一个类的请求交给父类加载器，如果这个父类加载器不能够找到或者加载这个类，那么再加载它。
- 可见性的原理是子类的加载器可以看见所有的父类加载器加载的类，而父类加载器看不到子类加载器加载的类。
- 单一性原理是指仅加载一个类一次，这是由委托机制确保子类加载器不会再次加载父类加载器加载过的类。
- 有三种默认使用的类加载器：Bootstrap类加载器、Extension类加载器和System类加载器（或者叫作Application类加载器）。
每种类加载器都有设定好从哪里加载类。[查看更详细的机制](http://www.importnew.com/6581.html)

###类加载器与本地库
假设两个同名类C里面都有一个f方法（签名不一样），虚拟机用同一个名字C_f来对两个C.f方法的本地实现来定位。为了保证每个
类链接到正确的本地函数，每个类加载器必须维护一个与自己相关联的本地库的集合（相当于所有的本地方法定义在同一个文件里，
那么两个签名不同的方法自然而然可以区分开）。  
这样，只要类拥有相同的类加载器，开发者就可以使用单一的本地库库来存储在任意类里面使用的本地方法。  
当类被加载器回收时，本地库也会被JVM自动unload。  

###定位本地库
本地库通过System.loadLibrary方法加载。在下面的例子中，类Cls静态初始化时加载了一个本地库。
```java
package pkg; 

class Cls {
    native double f(int i, String s);
    static {
      System.loadLibrary("mypkg");
    }
 }
```
JVM会根据当前系统环境的不同，把库的名字转换成相应的本地库名字。例如，Solaris下，mypkg会被转化成libmypkg.so，而Win32
环境下，被转化成mypkg.dll。  
JVM在启动的时候，会生成一个本地库的目录列表，这个列表的具体内容依赖于当前的系统环境，比如Win32下，这个列表中会包含
Windows系统目录、当前工作目录、PATH环境变量里面的目录。  
System.loadLibrary在加载相应本地库失败时，会抛出UnsatisfiedLinkError错误。如果相应的库已经加载过，这个方法不做任何
事情。如果底层操作系统不支持动态链接，那么所有的本地方法必须被prelink到VM上，这样的话，VM中调用System.loadLibrary
时实际上没有加载任何库。  

JVM内部为每一个类加载器都维护了一个已经加载的本地库的列表。它通过三步来决定一个新加载的本地库应该和哪个类加载器关联。
- 确定System.loadLibrary的调用者。
- 确定定义调用者的类。
- 确定类的加载器。
所以上述例子中，JVM会把mypkg与Cls的加载器关联起来  

###类型安全约束
VM不允许一个本地库被多个类加载器加载。当一个JNI本地库已经被第一个类加载器加载后，第二个类加载器再加载时，会报
UnsatisfiedLinkError。这样规定的目的是为了确保基于类加载器的命名空间分隔机制在本地库中同样有效。如果不这样的话，通
过本地方法进行操作JVM时，很容易造成属于不同类加载器的类和接口的混乱。  
下面代码中，本地方法Foo.f中缓存了一个全局引用，指向类Foo：
```C
JNIEXPORT void JNICALL

Java_Foo_f(JNIEnv *env, jobject self)
{
  static jclass cachedFooClass; /* cached class Foo */
  if (cachedFooClass == NULL) {
    jclass fooClass = (*env)->FindClass(env, "Foo");
    if (fooClass == NULL) {
      return; /* error */
    }
    cachedFooClass = (*env)->NewGlobalRef(env, fooClass);
    if (cachedFooClass == NULL) {
      return; /* error */
    }
  }

  assert((*env)->IsInstanceOf(env, self, cachedFooClass));
  ... /* use cachedFooClass */
 }
```
因为Foo.f是一个实例方法，而self指向一个Foo的实例对象，所以，我们认为最后那个assertion会执行成功。但是，如果L1和L2
分别加载了两个不同的Foo类，而这两个Foo类都被链接到Foo.f的实现上的话，assertion可能会执行失败。因为，哪个Foo类的f方
法首先被调用，全局引用cachedFooClass指向的就是哪个Foo类。  

###Unloading 本地库
一旦JVM回收类加载器，与这个类加载器关联的本地库就会被unload。因为类指向它自己的加载器，所以，这意味着，VM也会被这
个类unload。  

##链接本地方法
VM会在第一次使用一个本地方法的时候链接它。假设调用了方法g，而在g的方法体中出现了对方法f的调用，那么本地方法f就会被
链接。VM不应该过早地链接本地方法，因为这时候实现这些本地方法的本地库可能还没有被load，从而导致链接错误。  
链接一个本地方法需要下面这几个步骤：
- 确定定义了本地方法的类的加载器。
- 在加载器所关联的本地库列表中搜索实现了本地方法的本地函数。
- 建立内部数据结构，使得之后的对本地方法的调用能直指本地函数    

VM通过下面这几步，同本地方法的名字生成与之对应的本地函数的名字：
- 前缀“Java_”。
- 类的全名。
- 下划线分隔符“_”。
- 方法名字。
- 有方法重载的情况时，还会有两个下划线（“__”），后面跟着参数描述符。  

虚拟机遍历类定义加载器的所有本地库，以搜索指定名字的本地函数。对每一个库进行搜索时，VM会先搜索短名字（short name），即没有参数描述符的名字。然后搜索长名字（long name），即有参数描述符的名字。当两个本地方法重载时，程序
员需要使用长名字来搜索。但如果一个本地方法和一个非本地方法重载时，就不会使用长名字。  
在下面的例子里，本地方法g不会用长名字链接，因为另一个g并非本地方法：
```java
Class Cls {
  int g(int i) { ... }
  native int g(double d);
}

```
如果多个本地库中都存在与一个编码后的本地方法名字相匹配的本地函数，哪个本地库首先被加载，则它里面的本地函数就与
这个本地方法链接。如果没有哪个函数与给定的本地方法相匹配，则UnsatisfiedLinkError被抛出。  
程序员还可以调用JNI函数RegisterNatives来注册与一个类关联的本地方法。这个JNI函数对静态链接函数非常有用。

##调用协议
调用协议决定了一个本地函数如何接收参数和返回结果。目前没有一个标准，主要取决于编译器和本地语言的不同。JNI要求同
一个系统环境下，调用协议必须相同。例如，JNI在UNIX下使用C调用协议，而在Win32下使用stdcall调用协议。  
如果程序员需要调用的函数遵循不同的调用协议，那么最好写一个转换层来解决这个问题。  

##JNIEnv接口指针
本地代码通过JNIEnv接口指针里暴露的方法来使用虚拟机的功能。

###JNIEnv接口指针的组织结构
JNIEnv是一个指向本地线程数据的接口指针，这个指针里面包含了一个指向函数表的指针。每一个接口函数在这表中都有一个
预定义的偏移位置。JNIEnv很像一个C++虚函数表或者Microsoft COM接口。
![JNIEnv interface pointer](/jnienv.png)  
- 如果一个函数实现了一个本地方法，那么这个函数的第一个参数就是一个JNIEnv接口指针。
- 从同一个线程中调用的本地方法，传入的JNIEnv指针是相同的。
- 本地方法可能被不同的线程调用，这时，传入的JNIEnv指针是不同的。但JNIEnv间接指向的函数表在多个线程间是共享的。
- JNIEnv指针是本地线程的，本地代码决不能跨线程使用JNIEnv。  

###使用接口指针的好处
比起写死一个函数入口来说，使用接口指针可以有以下几个优点：
- JNI函数表是作为参数传递给每一个本地方法的，这样的话，本地库就不必与特定的JVM关联起来。这使得JNI可以在不同的JVM
间通用。
- JVM可以提供几个不同的函数表，用于不同的场合。比如，JVM可以提供两个版本的JNI函数表，一个做较多的错误检查，用于
调试时；另外一个做较少的错误检查，更高效，用于发布时。

##数据传递
像int、char等这样的基本数据类型，在本地代码和JVM之间进行复制传递，而对象是引用传递的。每一个引用都包含一个指向JVM
中相应的对象的指针，但本地代码不能直接使用这个指针，必须通过引用来间接使用。  
比起传递直接指针来说，传递引用可以让VM更灵活地管理对象。

###局部引用与全局引用
JNI可以为本地代码创建两种对象引用：局部引用和全局引用。局部引用的有效期是本地方法的调用期间，调用完成后，局部引用
会被JVM自动铲除。而全局引用，除非显示释放它，否则将一直存在。  
JVM中的对象作为参数传递给本地方法时，用的是局部引用。大部分的JNI函数返回局部引用。JNI允许程序员从局部引用创建一个
全局引用。接受对象作为参数的JNI函数既支持全局引用也支持局部引用。本地方法执行完毕后，向JVM返回结果时，它可能向JVM
返回局部引用，也可能返回全局引用。  
局部引用只在创建它的线程内部有效。本地代码不能跨线程传递和使用局部引用。  
JNI中的NULL引用指向JVM中的null对象。对一个全局引用或者局部引用来说，只要它的值不是NULL，它就不会指向一个null对象。

###局部引用的实现
一个对象从JVM传递给本地方法时，就把控制权移交了过去，JVM会为每一个对象的传递创建一条记录，一条记录就是一个本地代码
中的引用和JVM中的对象的一个映射。记录中的对象不会被GC回收。所有传递到本地代码中的对象和从JNI函数返回的对象都被自动
地添加到映射表中。当本地方法返回时，VM会删除这些映射，允许GC回收记录中的数据。

###弱引用
弱引用所指向的对象允许JVM回收，当对象被回收以后，弱引用也会被清除。

##访问对象
JNI提供了很多函数来操作对象。这意味着，本地方法的实现不需要关心虚拟机内部如何表示对象。这项关键的设计决定是JNI不必
关心VM的内部实现。  
使用JNI函数来通过引用间接操作对象比使用指针直接操作C中的对象要慢。但是，我们认为这很值得。

###访问基本类型数组
访问数组时，如果用JNI函数重复调用访问其中的每一个元素，那么消耗是相当大的。
一个解决方案是引入一种“pinning”机制，这样JVM就不会再移走数组内容。本地方法接受一个指向这些元素的直接指针。但这有两个
影响：
- JVM的GC必须支持“pinning”。“pinning”机制在JVM中并不一定被实现，因为它会使GC的算法更复杂，并有可能导致内存碎片。
- JVM必须在内存中连续地存放数组。虽然这是大部分基本类型数组的默认实现方式，但是boolean数组是比较特殊的一个。Boolean数
组有两种方式，packed和unpacked。用packed实现方式时，每个元素用一个bit来存放一个元素，而unpacked使用一个字节来存放一个
元素。因此，依赖于boolean数组特定存放方式的本地代码将是不可移植的。  

JNI采用了一个折衷方案来解决上面这两个问题：
- JNI提供了一系列函数（例如，GetIntArrayRegion、SetIntArrayRegion）把基本类型数组复制到本地的内存缓存。如果本地代码
需要访问数组当中的少量元素，或者必须要复制一份的话，请使用这些函数。
- 程序可以使用另外一组函数（例如，GetIntArrayElement）来获取数组被pinning后的直接指针。如果VM不支持pin，这组函数会返
回数组的复本。这组函数是否会复制数组，取决于下面两点：
- 如果GC支持pin，并且数组的布局和本地相同类型的数组布局一样，就不会发生复制。
- 否则的话，数组被复制到一个不可变的内存块儿中（例如，C的heap上面）并做一些格式转换。并把复制品的指针返回。  
- 当数组使用完后，本地代码会调用另外一组函数（例如，ReleaseInt-ArrayElement）来通知JVM。这时，JVM会unpin数组或者把对复
制后的数组的改变反映到原数组上然后释放复制后的数组。

这种方式提供了很大的灵活性。GC算法可以自由决定是复制数组，或者pin数组，还是复制小数组，pinning大数组。
JNI函数必须确保不同线程的本地方法可以同步访问相同的数组。例如，JNI可能会为每一个被pinning的数组保持一个计数器，如果数
组被两个线程pin的话，其中一个unpin不会影响另一个线程。

###字段和方法
JNI允许本地代码通过名字和类型描述符来访问JAVA中的字段或调用JAVA中的方法。
例如，为了读取类cls中的一个int实例字段：
```C
//本地方法首先要获取字段ID
jfieldID fid = env->GetFieldID(env, cls, "i", "I"); 
//然后可以多次使用这个ID，不需要再次查找，除非JVM把定义这个字段和方法的类或者接口unload，字段ID和方法ID会一直有效。
jint value = env->GetIntField(env, obj, fid); 
```
字段和方法可以来自定个类或接口，也可以来自它们的父类或间接父类。JVM规范规定：如果两个类或者接口定义了相同的字段和方法
，那么它们返回的字段ID和方法ID也一定会相同。例如，如果类B定义了字段fld，类C从B继承了字段fld，那么程序从这两个类上获
取到的名字为“fld”的字段的字段ID是相同的。  
JNI不会规定字段ID和方法ID在JVM内部如何实现。  
通过JNI，程序只能访问那些已经知道名字和类型的字段和方法。而使用Java CoreReflection机制提供的API，程序员不用知道具体的
信息就可以访问字段或者调用方法。有时在本地代码中调用反射机制也很有用。所以，JDK提供了一组API来在JNI字段ID和
java.lang.reflect.Field 类的实例之间转换，另外一组在JNI方法ID和java.lang.reflect.Method类实例之间转换。

##错误和异常
JNI编程时的错误通常是JNI函数的误用导致的。比如，向GetFieldID方法传递一个对象引用而不是类引用等。

###不检查编程错误
JNI函数不对编程错误进行检查。向JNI函数传递非法参数会导致未知的行为。原因如下：
- 强制JNI函数检查所有可能的错误会减慢所有本地方法的执行效率。
- 大部分情况下，运行时没有足够的类型信息来做错误检查。

###异常检查
有两种方式可以检查异常：
- 多数JNI函数以返回一个非正常的值来表示有一个错误发生了。
- 当使用一个不能由返回值断定发生了错误的JNI函数时，native代码必须依赖提出的异常，来进行错误检查。

```java
class CatchThrow {
    private native void doit() throws IllegalArgumentException;
    private void callback() throws NullPointerException {
        throw new NullPointerException("CatchThrow.callback");
    }

    public static void main(String args[]) {
        CatchThrow c = new CatchThrow();
        try {
            c.doit();
        } catch (Exception e) {
            System.out.println("In Java:\n\t" + e);
        }
    }
    static {
        System.loadLibrary("CatchThrow");
    }
}
```
```C
JNIEXPORT void JNICALL Java_CatchThrow_doit(JNIEnv *env, jobject obj)
{
    jthrowable exc;
    jclass cls = (*env)->GetObjectClass(env, obj);
    jmethodID mid = (*env)->GetMethodID(env, cls, "callback", "()V");
    if (mid == NULL) {
        return;
    }
    (*env)->CallVoidMethod(env, obj, mid);          //调用Java方法
    exc = (*env)->ExceptionOccurred(env);           //使用ExceptionOccurred检查异常      
    if (exc) {
        /* We don't do much with the exception, except that
           we print a debug message for it, clear it, and
           throw a new exception. */
        jclass newExcCls;
        (*env)->ExceptionDescribe(env);            //通过调用ExceptionDescribe函数来输出描述信息
        (*env)->ExceptionClear(env);               //调用ExceptionClear来清除该异常
        newExcCls = (*env)->FindClass(env, "java/lang/IllegalArgumentException");
        if (newExcCls == NULL) {
            /* Unable to find the exception class, give up. */
            return;
        }
        (*env)->ThrowNew(env, newExcCls, "thrown from C code");  //向外抛出一个IlleagalArgumentException来代替
    }
}

/* Native code that calls Fraction.floor. Assume method ID
   MID_Fraction_floor has been initialized elsewhere. */
void f(JNIEnv *env, jobject fraction)
{
    jint floor = (*env)->CallIntMethod(env, fraction,
                                       MID_Fraction_floor);
    /* important: check if an exception was raised */
    if ((*env)->ExceptionCheck(env)) {            // 检查异常
        return;
    }
    ... /* use floor */
}
```
一个有JNI导致的异常，不会马上终止native方法的执行。这一点和java语言的不同，当java编程语言抛出了一个异常，VM会在自动
转换控制流到最近的满足异常类型的try/catch语句，然后清除附加的异常，并且执行这个异常处理。与之对比，JNI程序员，在一个
异常发生时必须显示地实现控制流。
