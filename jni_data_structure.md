#JNI数据结构
=================
本文档主要介绍JNI的数据结构，它与Java数据结构，C/C++数据结构的关系，以及它们之间的转换和传递。
**同一种JNI类型，在C与C++中很可能有不同的实现** 文档只以C的实现说明

Java和JNI之间**相对应**数据类型的转换，由JVM保证；但是JNI与本地代码相对应数据类型的转换，不得不由程序员  
自己完成，无疑，这是件复杂乏味并且容易出错的事情。

本文档里提到的函数请参考[JNI函数手册](/jni_function_mannual.md)

##基本类型
在jni.h中定义了jni基本类型，其本质是某种整型。每种类型与java中的一种基本类型对应
```C
typedef uint8_t         jboolean;       /* unsigned 8 bits，对应Java中的boolean*/
typedef int8_t          jbyte;          /* signed 8 bits，对应byte */
typedef uint16_t        jchar;          /* unsigned 16 bits，对应char */
typedef int16_t         jshort;         /* signed 16 bits，对应short */
typedef int32_t         jint;           /* signed 32 bits，对应int */
typedef int64_t         jlong;          /* signed 64 bits，对应long */
typedef float           jfloat;         /* 32-bit IEEE 754，对应float */
typedef double          jdouble;        /* 64-bit IEEE 754，对应double */

```

###jboolean
Java中有boolean类型，JNI中对应的是jboolean，它只有两个值: JNI_TRUE, JNI_FALSE；  
而C中是没有布尔类型的，但可以使用类似的宏定义，或者直接使用JNI的jboolean类型；  
C++中有bool类型，它与jboolean的映射可以用简单的if...else...完成。

###jbyte
在Java中，一个byte型整数在内存中占8位，也就是一个字节. 表数范围:-128 ~-127;  
在JNI中，jbyte底层是一个八位的整数；  
在C/C++中的signed char通常有相同底层实现，所以byte, jbyte, signed char之间可以直接转换

###jchar
Java中的char型，是表示unicode的一个两字节正整数;  
JNI中jchar的实现，正是两字节无符号整数；
但是在C中的char型，只是16位有符号整数，所以从java的char映射到C的char只有在0～127这个区间比较有效，
jchar其实与unsigned short更接近； 
C++中用w_char来支持unicode，它的底层实现整数unsigned short。

###jshort
Java中short取值范围是-32768~32767, 占两个字节，在JNI中int16_t，16位的整数实现；  
在C/C++中，可以用signed short映射。

###jint
在Java中int，JNI中的jint都是32位整数；  
在C/C++中，int长度可能和机器实现有关，但是大部分情况下是也是32位长度的。

###jlong
同jint

###float与double
Java， JNi， C/C++中可以互相通用。

##引用类型
引用类型中C的实现里只是一个void指针。
```C
typedef void*           jobject;            /*对应任何Java对象，通常对应一个类的实例*/
typedef jobject         jclass;             /*对应Java类对象，比如Student类*/
typedef jobject         jstring;            /*java.lang.String*/
typedef jobject         jarray;             /*数组*/
typedef jarray          jobjectArray;       /*任何对象的数组*/
typedef jarray          jbooleanArray;      /*boolean []*/
typedef jarray          jbyteArray;         /*byte []*/
typedef jarray          jcharArray;         /*char []*/
typedef jarray          jshortArray;        /*short []*/
typedef jarray          jintArray;          /*int []*/
typedef jarray          jlongArray;         /*long []*/
typedef jarray          jfloatArray;        /*float []*/
typedef jarray          jdoubleArray;       /*double []*/
typedef jobject         jthrowable;         /*java.lang.Throwable*/
typedef jobject         jweak;              /**/

```

###jobject与jclass
分别对应Java中的对象和类，本地代码对它们的操作需要通过一些列的JNI函数来完成，具体参考[JNI实现](jni_implementation.md)

###jstring
Java字符串java.lang.String对象映射到JNI，就是jstring；  
但是在C/C++中通常是用char *去操作数组的，有时候本地代码也要传递一个jstring给JNI。

####jchar *->jstring
NewString: 使用jchar指针创建新的jstring对象，此对象可以由JNI返回给Java层

####char *->jstring
NewStringUTF: 使用const char指针创建一个新的jstring对象，此脆性可以由JNI返回给Java

####jstring->jchar *
GetStringChars与ReleaseStringChars: 获取/释放unicode字符串的jchar指针
GetStringRegion：拷贝jstring里的内容到一个jachar类型的buf中，通常这个buf是个自动变量

####jstring->char *
GetStringUTFChars与ReleaseStringUTFChars：获取/是否utf-8字符串的C char指针
GetStringUTFRegion：拷贝jstring里的内容到一个achar类型的buf中，通常这个buf是个自动变量

###jarray(jintarray, jobjectarray...)
Java数组被JNI转换成jarray对象，对于JNI基本类型通常先要将jarray对象转换成NativeType[], 然后将具体的NativeType转换
成C/C++可操作的对象；对于引用类型有专门的方法来处理

####构造基本类型jarray
New<PrimitiveType>Array：用于构造基本类型数组对象

####构造引用类型的jarray
NewObjectArray：构造引用类型的数组对象，通常是某种java类

####操作基本类型jarray
Get<PrimitiveType>ArrayElements/Release<PrimitiveType>ArrayElements：获取/释放jarray对象的指针
Get<PrimitiveType>ArrayRegion: 拷贝jarray里的元素到一个NativeType类型的buf中，通常这个buf是个自动变量
Set<PrimitiveType>ArrayRegion：将NativeType类型的buf中拷贝一些元素到jarray里

####操作引用类型的jarray
GetObjectArrayElement/SetObjectArrayElement：根据索引获取/修改引用类型数组的元素

##类型签名
在Java和JNI经常会互相相调用方法，其间要保证参数和返回值之间的映射正确，因此二者需要一个协议来保证。
其实现就是在JNI中用一些符号来表示Java数据类型。  

|Type Signature         |Java　Type           |  
|----------------       |------               |  
|Z                      |boolean              |  
|B                      |byte                 |  
|C                      |S                    |  
|I                      |int                  |  
|J                      |long                 |  
|F                      |float                |  
|D                      |double               |  
|Lfully-qualified-class;|fully-qualified-class|  
|[type                  |type []              |  

如果需要同时指出参数和返回值的签名，用这样的格式: (arg-types)ret-type  
比如Java方法原型：long f(int n, String s, int[] arr); 对应的签名是： (ILJava/lang/String;[I)J

##其他常用类型
```C
typedef jint            jsize;

struct _jfieldID;                       /* opaque structure */
typedef struct _jfieldID* jfieldID;     /* field IDs */

struct _jmethodID;                      /* opaque structure */
typedef struct _jmethodID* jmethodID;   /* method IDs */

#define JNI_FALSE   0
#define JNI_TRUE    1

#define JNI_VERSION_1_1 0x00010001
#define JNI_VERSION_1_2 0x00010002
#define JNI_VERSION_1_4 0x00010004
#define JNI_VERSION_1_6 0x00010006

#define JNI_OK          (0)         /* no error */
#define JNI_ERR         (-1)        /* generic error */
#define JNI_EDETACHED   (-2)        /* thread detached from the VM */
#define JNI_EVERSION    (-3)        /* JNI version error */

#define JNI_COMMIT      1           /* copy content, do not free buffer */
#define JNI_ABORT       2           /* free buffer w/o copying back */

```
