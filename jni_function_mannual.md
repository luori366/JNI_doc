##JNI函数手册
=====================
JNI所有函数可以在jni.h里找到原型，它们大多定义在JNINativeInterface和JNIInvokeInterface结构里；
前者的指针就是JNIEnv类型， 后者的指针是JavaVM类型。

```C
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;

struct JNINativeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;
    void*       reserved3;

    jint        (*GetVersion)(JNIEnv *);

    jclass      (*DefineClass)(JNIEnv*, const char*, jobject, const jbyte*,
                        jsize);
    jclass      (*FindClass)(JNIEnv*, const char*);
    ......
};

truct JNIInvokeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;

    jint        (*DestroyJavaVM)(JavaVM*);
    jint        (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);
    jint        (*DetachCurrentThread)(JavaVM*);
    jint        (*GetEnv)(JavaVM*, void**, jint);
    jint        (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*);
};
```
以下对这些函数进行简单分类，并介绍。
##版本信息
```java
/**
 * 获取JNI版本号
 *
 * @param env JNI interface指针
 * @return 返回一个十六进制整数，其中高16位表示主版本号，低16位标识表示次版本号，
 *  如：1.2, GetVersion()返回0x00010002, 1.4, GetVersion() returns 0x00010004.
 */
jint GetVersion(JNIEnv *env);
```
**后面再出现 JNIEnv *env 这样的参数不再注释**

###类操作
```java
    /**
     * 从原始类数据的缓冲区中加载类。
     *
     * @param loader 分派给所定义的类的类加载器
     * @param buf 包含 .class 文件数据的缓冲区 
     * @param buflen 缓冲区长度
     * @return 返回Java类对象。如果出错则返回NULL。
     *
     * @throw ClassFormatError  如果类数据指定的类无效
     *      ClassCircularityError  如果类或接口是自身的超类或超接口
     *      OutOfMemoryError  如果系统内存不足
     */
    jclass DefineClass (JNIEnv *env, jobject loader, const jbyte *buf , jsize bufLen); 
    
    /**
     * 该函数用于加载Java类。它将在CLASSPATH 环境变量所指定的目录和zip文件里搜索指定的类名。
     *
     * @param name  类全名 = (包名+‘/’+类名).replace('.', '/');
     * @return 类对象全名; 如果找不到该类，则返回 NULL。
     * @throw ClassFormatError      如果类数据指定的类无效
     *      ClassCircularityError   如果类或接口是自身的超类或超接口
     *      NoClassDefFoundError    如果找不到所请求的类或接口的定义
     *      OutOfMemoryError        如果系统内存不足
     */
    jclass FindClass(JNIEnv *env, const char *name);
    
    /**
     * 通过对象获取这个类。该函数比较简单，唯一注意的是对象不能为NULL，否则获取的class肯定返回也为NULL。
     *
     * @param obj Java 类对象实例
     */    
    jclass GetObjectClass (JNIEnv *env, jobject obj); 
    
    /**
     * 获取父类或者说超类 
     *
     * @param clazz Java类对象
     * @return 如果clazz代表一般类而非Object类，则该函数返回由clazz所指定的类的超类。 如果clazz 
           指向Object类或代表某个接口，则该函数返回NULL。
     */  
    jclass GetSuperclass (JNIEnv *env, jclass clazz); 
    
    /**
     * 确定 clazz1 的对象是否可安全地强制转换为clazz2
     *
     * @param clazz1 源类对象
     * @param clazz2 目标类对象
     * @return 以下三种情况返回JNI_TRUE, 否则返回JNI_FALSE
     *      1.第一及第二个类参数引用同一个 Java 类
     *      2.第一个类是第二个类的子类
     *      3.第二个类是第一个类的某个接口
     */    
    jboolean IsAssignableFrom (JNIEnv *env, jclass clazz1,  jclass clazz2); 
```
```java
    /**
     * 
     *
     * @param
     * @param
     * @param
     * @param
     * @return
     * @throw
     */
```
###异常操作
```java
    /**
     * 抛出 java.lang.Throwable 对象
     *
     * @param obj java.lang.Throwable 对象
     * @return 成功时返回 0，失败时返回负数
     * @throw java.lang.Throwable 对象 obj
     */
    jint  Throw(JNIEnv *env, jthrowable obj);
    
    /**
     * 利用指定类的消息（由 message 指定）构造异常对象并抛出该异常
     *
     * @param clazz java.lang.Throwable 的子类
     * @param message 用于构造java.lang.Throwable对象的消息
     * @return 成功时返回 0，失败时返回负数
     * @throw 新构造的 java.lang.Throwable 对象
     */    
    jint ThrowNew (JNIEnv *env ,  jclass clazz,  const char *message);

    /**
     * 确定某个异常是否正被抛出。在本地代码调用ExceptionClear()或Java代码处理该异常前，异常将始终保持
     *  抛出状态。
     *
     * @return 返回正被抛出的异常对象，如果当前无异常被抛出，则返回NULL
     */    
    jthrowable ExceptionOccurred (JNIEnv *env);

    /**
     * 将异常及堆栈的回溯输出到标准输出（例如 stderr）。该例程可便利调试操作。
     */    
    void ExceptionDescribe (JNIEnv *env);
    
    /*
     * 清除当前抛出的任何异常。如果当前无异常，则不产生任何效果。
     */
    void ExceptionClear (JNIEnv *env); 

    /*
     * 抛出致命错误并且不希望虚拟机进行修复。该函数无返回值
     * @param msg 错误消息
     */    
    void FatalError (JNIEnv *env, const char *msg);  
```

###全局及局部引用
###对象操作
###字符串操作
###数组操作
###访问对象的属性和方法
###注册本地方法
###
###
