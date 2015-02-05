#Application.mk 语法总结
===========================
本文档用来描述Application.mk生成文件的语法。
Application.mk其实是一个小的GNU Makefile片段, 用来描述你的Android程序需要的原生模块.
它通过定义一些模块变量来指导本地代码编译

##APP_MODULES
这个变量是可选的。如果没有定义，NDK将会采用默认的方式，编译在Android.mk声明的所有modules，同时包含了
所有子目录下的Android.mk文件
##APP_PROJECT_PATH
这个变量必须给出你应用程序的工程根目录的绝对路径。
**注意$PROJECT/jni/Application.mk是可选的，但是对于$NDK/apps/<myapp>/Application.mk却是强制要求的**
##APP_OPTIM
这个可选的变量可定义在‘release’或者‘debug’中。当编译应用程序的modules的时候，这个用来改变优化级别.
默认是‘release’模式，并生成高级别的二进制文件。‘debug’模式会生成没有优化的二进制文件，这样更容易调试。
##APP_CPPFLAGS
当编译任何modules中的c或c++源文件的时候，会传递一个C编译标志的集合。这个可以用来改变应用程序需要的module的编译行为，
而不需要修改Android.mk文件本身.
只有在编译C++源代码的时候，传递的C++编译标志的集合。
##APP_BUILD_SCRIPT
默认情况下，NDK编译系统会查找$(APP_PROJECT_PATH)/jni目录下的Android.mk文件.如果你想改变这个默认行为，你
可以定义APP_BUILD_SCRIPT指向一个编译脚本。
##APP_ABI
默认情况下，NDK编译系统将会生成‘armeabi’ABI的机器码.
##APP_STL
默认情况下，NDK编译系统提供最小的C++运行时库（/system/lib/libstdc++.so）的头文件，这个最小的运行时库是由android系
统提供。但是，NDK自带的C++实现，让你能够使用或链接到你的运用程序中。定义APP_STL为下面的一个，例如：
```
APP_STL := stlport_static       #static STLport library        
APP_STL := stlport_shared       #shared STLport library        
APP_STL := system               #default C++ runtime library
```
##APP_GNUSTL_FORCE_CPP_FEATURES
##APP_SHORT_COMMANDS
##NDK_TOOLCHAIN_VERSION
定义这个变量为4.4.3或4.6来选择GCC编译器的版本。4.6是默认值。
##APP_PIE
