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
