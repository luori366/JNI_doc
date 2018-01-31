# JNI调试及其他

本文档主要介绍如何调试JNI，以及其他相关主题

## JNI调试

### 单步调试
参考[JNI单步调试](http://blog.linguofeng.com/archive/2013/04/18/eclipse-android-ndk-debug.html)

### 打印程序
#### 使用android_log_print
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

## JNI中C/C++本地代码的不同

### 引用类型的定义
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

### JNIEnv定义的不同
以GetArrayLength为例，C和C++中的定义：
```C
struct _JNIEnv;             //前置声明，后面将正式声明

#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;     //在C++中，JNIEnv是_JNIEnv的别名，本质是一个类
#else
typedef const struct JNINativeInterface * JNIEnv;    //在C中JNIEnv是JNINativeInterface结构的指针，本质是一个指针
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

JNIEnv的实现方式不同，影响了它的使用方式
```C
// JNIEnv通常是以JNIEnv *env的指针传进jni方法里的
jsize len = (*env)->GetArrayLength(env, array);  //C中的使用方式
jsize len =env->GetArrayLength(array);          //C++中的使用方式
```

## JNI中汉字的处理
java内部是使用的16bit的unicode编码（utf-16）来表示字符串的，无论英文还是中文都是2字节；  
C/C++使用的是原始数据，ascii就是一个字节，中文一般是GB2312编码，用2个字节表示一个汉字。

### java unicode字符串->C/C++ 汉字
java字符串映射到JNI层是jstring类型，
- 如果其中不包含汉字，可以调用GetStringUTFChars/GetStringChars将它转化成一个UTF-8/UTF-16字符串；
- 如果包含字符串，则需要使用java.lang.String的API先将它转化成一个jbytearray，然后将这个jarray对象
里的内容复制到char类型的内存中。

```C
// GetStringUnicodeChars: Unicode字符串->char *
// GetStringChars: Unicode字符串->jchar *
// GetStringUTFChars: Utf-8字符串->char *
char * GetStringUnicodeChars(JNIEnv* env, jstring jsrc, const char * encoding) {
	char* rtn = NULL;

	jclass clazz = (*env)->FindClass(env, "java/lang/String");
	jmethodID methodID = (*env)->GetMethodID(env, clazz, "getBytes", "(Ljava/lang/String;)[B");
	jstring jencoding = (*env)->NewStringUTF(env, encoding);
	jbyteArray jbyte_array = (jbyteArray)(*env)->CallObjectMethod(env, jsrc, methodID, jencoding);

	jsize byte_array_len = (*env)->GetArrayLength(env, jbyte_array);
	jbyte * jbyte_array_ptr = (*env)->GetByteArrayElements(env, jbyte_array, JNI_FALSE);
	if (byte_array_len > 0) {
		rtn = (char*) malloc(byte_array_len + 1);
		memcpy(rtn, jbyte_array_ptr, byte_array_len);
		rtn[byte_array_len] = 0;
	}
	(*env)->ReleaseByteArrayElements(env, jbyte_array, jbyte_array_ptr, 0);

	return rtn;
}
```
### C/C++ 汉字->Java unicode字符串
jni返回给java的字符串，c/c++首先应该负责把这个字符串变成UTF-8或者UTF-16格式，然后通过NewStringUTF或者
NewString来把它封装成jstring，返回给java就可以了。
- 如果字符串中不含中文字符，只是标准的ascii码，那么用NewString/NewStringUTF就可以构造返回结果；
- 如果字符串中有中文字符，那么需要用Java String的API来构造jstring，其本质是调用String的构造函数
public String (byte[] data, String charsetName)
```C
//NewStringUnicode:Unicode char * -> jstring
//NewString: jchar * -> jstring
//NewStringUTF: UTF char *-> jstring
jstring NewStringUnicode(JNIEnv * env, const char * src, const char * encoding) {
	jbyteArray byte_array = (*env)->NewByteArray(env, strlen(src));
	(*env)->SetByteArrayRegion(env, byte_array, 0, strlen(src), (jbyte*) src);
	jstring jencoding = (*env)->NewStringUTF(env, encoding);

	jclass clazz = (*env)->FindClass(env, "java/lang/String");
	jmethodID methodID = (*env)->GetMethodID(env, clazz, "<init>", "([BLjava/lang/String;)V");
	return (jstring)(*env)->NewObject(env, clazz, methodID, byte_array, jencoding);
}
```

## Eclipse 安装cdt插件
Eclipse里如果没有安装cdt插件，工程里的jni代码会有各种警告，并且不能进行变量或函数调跳转（Ctrl+Left）
###安装与eclipse版本对应的cdt插件
Eclipse 4.3 (Kepler): http://download.eclipse.org/tools/cdt/releases/kepler
Eclipse 4.2 (Juno): http://download.eclipse.org/tools/cdt/releases/juno
Eclipse 3.7 (Indigo): http://download.eclipse.org/tools/cdt/releases/indigo
Eclipse 3.6 (Helios): http://download.eclipse.org/tools/cdt/releases/helios
Eclipse 3.5 (Galileo): http://download.eclipse.org/tools/cdt/releases/galileo

//TODO
