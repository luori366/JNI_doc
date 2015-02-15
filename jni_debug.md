#JNI调试及其他
======================
本文档主要介绍如何调试JNI，以及其他相关主题

##JNI调试

###单步调试
参考[JNI单步调试](http://blog.linguofeng.com/archive/2013/04/18/eclipse-android-ndk-debug.html)

###打印程序
####使用android_log_print
这是android系统的本地log方法。
```C
typedef enum android_LogPriority {
    ANDROID_LOG_UNKNOWN = 0,
    ANDROID_LOG_DEFAULT,    /* only for SetMinPriority() */
    ANDROID_LOG_VERBOSE,
    ANDROID_LOG_DEBUG,
    ANDROID_LOG_INFO,
    ANDROID_LOG_WARN,
    ANDROID_LOG_ERROR,
    ANDROID_LOG_FATAL,
    ANDROID_LOG_SILENT,     /* only for SetMinPriority(); must be last */
} android_LogPriority;


/*
 * （中标准输出上）打印格式化的日志信息，用法和printf很类似
 * @param prio 优先级，可以是android_LogPriority中的某一个
 * @tag 输入的tag
 * @fmt 格式化字符串，与printf里的规则一样
 * @... 不定参数列表
 *
int __android_log_print(int prio, const char *tag,  const char *fmt, ...);
```
通常会对这个函数进行封装，比如定义一个头文件
```C
#ifndef NATIVE_LOG_
#define NATIVE_LOG_

#include <android/log.h>

#define TAG "4399SDK"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG,TAG ,__VA_ARGS__)
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,TAG ,__VA_ARGS__)
#define LOGW(...) __android_log_print(ANDROID_LOG_WARN,TAG ,__VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR,TAG ,__VA_ARGS__)
#define LOGF(...) __android_log_print(ANDROID_LOG_FATAL,TAG ,__VA_ARGS__)

#endif
```

##JNI中C/C++本地代码的不同

###引用类型的定义
在C中，引用类型的本质是void指针
```C
/*
 * Reference types, in C.
 */
typedef void*           jobject;
typedef jobject         jclass;
typedef jobject         jstring;
typedef jobject         jarray;
typedef jarray          jobjectArray;
typedef jarray          jbooleanArray;
typedef jarray          jbyteArray;
typedef jarray          jcharArray;
typedef jarray          jshortArray;
typedef jarray          jintArray;
typedef jarray          jlongArray;
typedef jarray          jfloatArray;
typedef jarray          jdoubleArray;
typedef jobject         jthrowable;
typedef jobject         jweak;

```
而在C++中，引用类型的本质是类的指针，并且这些类有一个共同的空基类
```C
/*
 * Reference types, in C++
 */
class _jobject {};
class _jclass : public _jobject {};
class _jstring : public _jobject {};
class _jarray : public _jobject {};
class _jobjectArray : public _jarray {};
class _jbooleanArray : public _jarray {};
class _jbyteArray : public _jarray {};
class _jcharArray : public _jarray {};
class _jshortArray : public _jarray {};
class _jintArray : public _jarray {};
class _jlongArray : public _jarray {};
class _jfloatArray : public _jarray {};
class _jdoubleArray : public _jarray {};
class _jthrowable : public _jobject {};

typedef _jobject*       jobject;
typedef _jclass*        jclass;
typedef _jstring*       jstring;
typedef _jarray*        jarray;
typedef _jobjectArray*  jobjectArray;
typedef _jbooleanArray* jbooleanArray;
typedef _jbyteArray*    jbyteArray;
typedef _jcharArray*    jcharArray;
typedef _jshortArray*   jshortArray;
typedef _jintArray*     jintArray;
typedef _jlongArray*    jlongArray;
typedef _jfloatArray*   jfloatArray;
typedef _jdoubleArray*  jdoubleArray;
typedef _jthrowable*    jthrowable;
typedef _jobject*       jweak;
```

###JNIEnv定义的不同
以GetArrayLength为例，C和C++中的定义：
```C
struct _JNIEnv;             //前置声明，后面将正式声明
struct _JavaVM;
typedef const struct JNINativeInterface* C_JNIEnv;

#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;     //在C++中，JNIEnv是_JNIEnv的别名，本质是一个类
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;    //在C中JNIEnv是JNINativeInterface结构的指针，本质是一个指针
typedef const struct JNIInvokeInterface* JavaVM;
#endif


struct JNINativeInterface {
    ......
    jsize       (*GetArrayLength)(JNIEnv*, jarray);
    ......
};

/*
 * C++ object wrapper.
 *
 * This is usually overlaid on a C struct whose first element is a
 * JNINativeInterface*.  We rely somewhat on compiler behavior.
 */
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;

#if defined(__cplusplus)
    ......
    jsize GetArrayLength(jarray array) 
    { return functions->GetArrayLength(this, array); }
    ......
    
#endif /*__cplusplus*/
};
```

因此调用JNI里的方法
```C
// JNIEnv通常是以JNIEnv *env的指针传进jni方法里的
jsize len = (*env)->GetArrayLength(env, array);  //C中的使用方式
jsize len =env->GetArrayLength(array);          //C++中的使用方式
```
