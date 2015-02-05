# Android.mk语法总结
=========================================
本文档是关于Android.mk文件语法和写法的总结，主要包括变量，函数的使用

##一个简单的例子  
```
LOCAL_PATH := $(call my-dir) # my-dir是由编译系统提供的宏函数，用于返回Android.mk所在路径

include $(CLEAR_VARS)        # CLEAR_VARS是编译系统预定义变量，指向一个特殊的Makefile，这个Makefile用于清除LOCAL_XXX变量
                             # 的定义（除了LOCAL_PATH），以防止不同模块编译中同名变量的影响

LOCAL_MODULE    := hello-jni    # 定义模块名字，编译后自动驾驶前缀和后缀，如libhello-jni.so

LOCAL_SRC_FILES := hello-jni.c  # 定义源码文件集合

include $(BUILD_SHARED_LIBRARY) # BUILD_SHARED_LIBRARY预定义的变量，指向一个Makefile，编译后得到一个动态库
                                # 如果要编译成静态库，则可以换成BUILD_STATIC_LIBRARY
```
                                
                                
##变量命名规则  
不能用小写字母命名变量，不能用LOCAL_, PRIVATE_, NDK_, APP_开头命名变量 

###其他的预定义变量

####PREBUILT_SHARED_LIBRARY
指向一个已编译好的共享库，次数LOCAL_SRC_FILES不再指向源文件，而是指向这个预编译共享库文件如，libfoo.so.可以在其他模块中，  
通过使用LOCAL_PREBUILDS变量来引用这个预编译模块，具体用法可以参考：  
[预编译](http://blog.csdn.net/smfwuxiao/article/details/8523479)  
####PREBUILT_STATIC_LIBRARY
与PREBUILT_SHARED_LIBRARY相同，只不过这里是静态库  
####TARGET_ARCH
目标CPU架构的名字   
####TARGET_PLATFORM
目标Android平台的名字, 具体参考[Stable APIs](http://blog.csdn.net/smfwuxiao/article/details/6590723)  
####TARGET_ARCH_ABI
目标CPU和ABI组合的名字，目前只有2个值可以用：armeabi(ARMv5TE), armeabi-v7a  
####TARGET_ABI
目标平台和ABI的组合，定义为 $(TARGET_PLATFORM)-$(TARGET_ARCH_ABI)  

###模块描述变量

####LOCAL_PATH 
这个变量表示当前文件（一般是Android.mk）所在的路径，该变量很重要，必须定义
```
LOCAL_PATH := $(call my-dir)
```
####LOCAL_MODULE 
该变量定义当前模块的名字，名字必须唯一，不能有空格
####LOCAL_MODULE_FILENAME 
该变量可以用来重定义输出文件的名字, 当定义了LOCAL_MODULE_FILENAME之后，输出文件名就是这个变量指定的名字
####LOCAL_SRC_FILES 
该变量用来指定该模块对应的源文件
####LOCAL_CPP_EXTENSION 
用来定义C++代码文件的扩展名
```
LOCAL_CPP_EXTENSION := .cxx
```
####LOCAL_CPP_FEATURES 
该变量用来指定C++代码所依赖的特殊C++特性
```
LOCAL_CPP_FEATURES := rtti       #告诉编译器C++代码使用了RTTI（RunTime Type Information）
LOCAL_CPP_FEATURES := exceptions #指定C++代码使用了C++异常
```
####LOCAL_C_INCLUDES
一个路径的列表，当编译C/C++、汇编文件时，这些路径将被追加到头文件搜索路径列表中
####
####LOCAL_CFLAGS
指定当编译C/C++源码的时候，传给编译器的标志
####LOCAL_CPPFLAGS 
编译C++代码的时候传递给编译器的选项（编译C代码不会用这里的选项）
####LOCAL_WHOLE_STATIC_LIBRARIES
它是LOCAL_STATIC_LIBRARIES的变体，用来表示它对应的模块对于linker来说应该是一个“whole archive”.

##NDK预定义宏函数
下面是NDK预定义的“函数”宏，用法是  $(call <function>) ，返回的是文本信息

###all-subdir-makefiles
返回当前的my-dir目录下的所有子目录的Android.mk文件的列表  
###this-makefile
返回当前的Makefile的路径（即该函数调用时的位置）  
###parent-makefile
如果当前这个Makefile被另一个Makefile包含，则返回那个包含了自己的Makefile的路径（即parent）  
###grand-parent-makefile
...
###import-module
该函数用于按模块名查找另一个模块的Android.mk文件，并包含进来
```
 $(call import-module,<name>) #在 NDK_MODULE_PATH变量所指定的目录列表中寻找名为<name>的模块，找到之后将包含进来
```

                                
