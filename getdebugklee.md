### klee-debug的编译过程

#### 安装LLVM等配套软件
详细见文档Building KLEE (LLVM 3.4)
以下为"Working with KLEE source code"中"Setting up a debug build of KLEE"一节,我在安装时遇到的问题和解决笔记.

#### 配置configure
1. configure的前后可以添加什么参数,其实是可以在configure文件中查到的
2. 执行configure配置命令
```
configure命令:
CXXFLAGS="-g -O0" CFLAGS="-g -O0" LDFLAGS="-ldl -pthread -ltinfo" ./configure with_llvm_build_mode="check" LLVM_BUILD_MODE="Debug"  --with-llvmsrc=../llvm-3.4-src --with-llvmobj="/home/ccc/program/llvm-3.4-build/" --with-stp=../stp   --with-llvmcxx=/home/ccc/program//llvm-3.4-install/bin/clang++ --with-llvmcc=/home/ccc/program/llvm-3.4-install/bin/clang --with-uclibc=/home/ccc/program/klee-uclibc --enable-posix-runtime --with-runtime=Debug+Asserts
```
出错 "configure: error: Could not autodetect build mode",在configure文件中搜索相应的字符串,找到如下代码
```
if test X${with_llvm_build_mode} = Xcheck ; then
  llvm_configs="`ls -1 $llvm_obj/*/bin/llvm-config 2>/dev/null | head -n 1`"
  if test -x "$llvm_configs" ; then
    llvm_build_mode="`$llvm_configs --build-mode`"
  else
    as_fn_error $? "Could not autodetect build mode" "$LINENO" 5
  fi
else
  llvm_configs="`echo $llvm_obj/*/bin/llvm-config`"
  if test -x "$llvm_obj/$with_llvm_build_mode/bin/llvm-config" ; then
    llvm_build_mode=$with_llvm_build_mode
  else
    as_fn_error $? "Invalid build mode: $llvm_build_mode" "$LINENO" 5
  fi
fi
```
这是由于"ls -1 $llvm_obj/\*/bin/llvm-config 2" 无法返回正确结果,修改为"ls -1 $llvm_obj/bin/llvm-config 2"
再次执行configure命令,出现错误"checking C LLVM Bitcode compiler works... configure: error: Could not find llvm-dis
",搜索该字符串,根据相关的if逻辑,找到代码:
```
if test -x "$llvm_obj/$llvm_build_mode/bin/llvm-dis" ; then
```
这是由于我机器上的$llvm_obj的目录机构中,bin文件夹上层没有$llvm_build_mode目录,为此替换所有的"$llvm_build_mode/bin/"为"bin/"
再次执行configure命令,无报错结束.

#### 编译连接过程make
发现生成的是Release+Asserts文件夹,而不是我期望的Debug+Asserts,并且出现"/home/ccc/program/klee-ccc/Makefile.rules:934: *** llvm-config --libs failed.  Stop.",去Makefile.rules的934行,根据逻辑上溯,发现
```
LLVMLibDir  := $(LLVM_OBJ_ROOT)/$(LLVM_BUILD_MODE)/lib
LLVMToolDir := $(LLVM_OBJ_ROOT)/$(LLVM_BUILD_MODE)/bin
LLVMExmplDir:= $(LLVM_OBJ_ROOT)/$(LLVM_BUILD_MODE)/examples
```
这还是和之前遇到找不到llvm-config一样的问题,修改为
```
LLVMLibDir  := $(LLVM_OBJ_ROOT)/lib
LLVMToolDir := $(LLVM_OBJ_ROOT)/bin
LLVMExmplDir:= $(LLVM_OBJ_ROOT)/examples
```
再次make,成功编译,但出现链接错误
```
...
make[2]: Entering directory `/home/ccc/program/klee-ccc/tools/klee'
llvm[2]: Compiling Debug.cpp for Release+Asserts build
llvm[2]: Compiling main.cpp for Release+Asserts build
llvm[2]: Linking Release+Asserts executable klee (without symbols)
/home/ccc/program/llvm-3.4-build/lib/libLLVMSupport.a(DynamicLibrary.cpp.o): In function `llvm::sys::DynamicLibrary::getPermanentLibrary(char const*, std::string*)':
/home/ccc/program/llvm-3.4-src/lib/Support/DynamicLibrary.cpp:60: undefined reference to `dlopen'
/home/ccc/program/llvm-3.4-src/lib/Support/DynamicLibrary.cpp:62: undefined reference to `dlerror'
/home/ccc/program/llvm-3.4-src/lib/Support/DynamicLibrary.cpp:79: undefined reference to `dlclose'
/home/ccc/program/llvm-3.4-build/lib/libLLVMSupport.a(DynamicLibrary.cpp.o): In function `llvm::sys::DynamicLibrary::getAddressOfSymbol(char const*)':
/home/ccc/program/llvm-3.4-src/lib/Support/DynamicLibrary.cpp:87: undefined reference to `dlsym'
/home/ccc/program/llvm-3.4-build/lib/libLLVMSupport.a(DynamicLibrary.cpp.o): In function `llvm::sys::DynamicLibrary::SearchForAddressOfSymbol(char const*)':
/home/ccc/program/llvm-3.4-src/lib/Support/DynamicLibrary.cpp:128: undefined reference to `dlsym'
/home/ccc/program/llvm-3.4-build/lib/libLLVMSupport.a(Process.cpp.o): In function `terminalHasColors':
/home/ccc/program/llvm-3.4-src/lib/Support/Unix/Process.inc:273: undefined reference to `setupterm'
/home/ccc/program/llvm-3.4-src/lib/Support/Unix/Process.inc:291: undefined reference to `tigetnum'
/home/ccc/program/llvm-3.4-src/lib/Support/Unix/Process.inc:295: undefined reference to `set_curterm'
/home/ccc/program/llvm-3.4-src/lib/Support/Unix/Process.inc:296: undefined reference to `del_curterm'
/home/ccc/program/llvm-3.4-build/lib/libLLVMSupport.a(Signals.cpp.o): In function `llvm::sys::PrintStackTrace(_IO_FILE*)':
/home/ccc/program/llvm-3.4-src/lib/Support/Unix/Signals.inc:278: undefined reference to `dladdr'
/home/ccc/program/llvm-3.4-src/lib/Support/Unix/Signals.inc:290: undefined reference to `dladdr'
collect2: error: ld returned 1 exit status
make[2]: *** [/home/ccc/program/klee-ccc/Release+Asserts/bin/klee] Error 1
make[2]: Leaving directory `/home/ccc/program/klee-ccc/tools/klee'
make[1]: *** [klee/.makeall] Error 2
make[1]: Leaving directory `/home/ccc/program/klee-ccc/tools'
make: *** [all] Error 1
```
这些链接的库其实再configure命令中其实已经赋值了,该命令会修改Makefile.config相应变量为"LDFLAGS := -ldl -pthread -ltinfo -g",为什么还有这个错误,说明makefile之间的连接存在问题(这是经过1天半时间不断的折腾才发现的),去往tools/klee/makefile,在文件中添加"LIBS += $(LDFLAGS)"
- 再次make,发现tools/klee连接聪哥了,但是tools/kleaver出现了同样的链接错误,去往tools/kleaver/makefile,在文件中添加"LIBS += $(LDFLAGS)",再次make,成功!,再次回到上面遗留的一个问题,为什么生成了完整的Release+Asserts文件夹,而Debug+Asserts文件夹中只有lib文件夹,lib文件夹中的文件还是残缺的,Release+Asserts中klee程序是release版还是debug版?

- Debug版在哪?
  - 首先这是链接过程配置问题,根据make输出中的"(without symbols)"信息,我们搜索此字符串,在Makefile.rules中发现代码段
  ```
  ifndef KEEP_SYMBOLS
    Strip := $(PLATFORMSTRIPOPTS)
    StripWarnMsg := "(without symbols)"
    Install.StripFlag += -s
  endif
  ```
  查看KEEP_SYMBOLS的继承关系,通过打印"$(info $(ENABLE_OPTIMIZED))",发现该值有时候为0,有时候为1,在Makefile.common中有"override ENABLE_OPTIMIZED := $(RUNTIME_ENABLE_OPTIMIZED)",在configure文件中会根据${with_runtime} 给其赋值
  ```
  # Check whether --with-runtime was given.
  if test "${with_runtime+set}" = set; then :
    withval=$with_runtime;
  else
    withval=default
  fi

  if test X"${withval}" = Xdefault; then
     with_runtime=Release+Asserts
  fi

  { $as_echo "$as_me:${as_lineno-$LINENO}: checking runtime configuration" >&5
  $as_echo_n "checking runtime configuration... " >&6; }
  if test X${with_runtime} = XRelease; then
      { $as_echo "$as_me:${as_lineno-$LINENO}: result: Release" >&5
  $as_echo "Release" >&6; }
      RUNTIME_ENABLE_OPTIMIZED=1

      RUNTIME_DISABLE_ASSERTIONS=1

      RUNTIME_DEBUG_SYMBOLS=

  elif test X${with_runtime} = XRelease+Asserts; then
      { $as_echo "$as_me:${as_lineno-$LINENO}: result: Release+Asserts" >&5
  $as_echo "Release+Asserts" >&6; }
      RUNTIME_ENABLE_OPTIMIZED=1

      RUNTIME_DISABLE_ASSERTIONS=0

      RUNTIME_DEBUG_SYMBOLS=

  elif test X${with_runtime} = XDebug; then
     { $as_echo "$as_me:${as_lineno-$LINENO}: result: Debug" >&5
  $as_echo "Debug" >&6; }
     RUNTIME_ENABLE_OPTIMIZED=0

     RUNTIME_DISABLE_ASSERTIONS=1

     RUNTIME_DEBUG_SYMBOLS=1

  elif test X${with_runtime} = XDebug+Asserts; then
     { $as_echo "$as_me:${as_lineno-$LINENO}: result: Debug+Asserts" >&5
  $as_echo "Debug+Asserts" >&6; }
     RUNTIME_ENABLE_OPTIMIZED=0

     RUNTIME_DISABLE_ASSERTIONS=0

     RUNTIME_DEBUG_SYMBOLS=1

  else
     as_fn_error $? "invalid configuration: ${with_runtime}" "$LINENO" 5
  fi
  ```
  
  但是打印$withval,结果是"Debug+Asserts",继续找RUNTIME_ENABLE_OPTIMIZED和ENABLE_OPTIMIZED的值定义,在Makefile.common中发现include $(LLVM_OBJ_ROOT)/Makefile.config,在该文件中发现
  ```
  # When ENABLE_OPTIMIZED is enabled, LLVM code is optimized and output is put
  # into the "Release" directories. Otherwise, LLVM code is not optimized and
  # output is put in the "Debug" directories.
  #ENABLE_OPTIMIZED = 1
  ENABLE_OPTIMIZED=1
  ```
  原来该llvmobj直接设置为了优化模式,这也是KLEE官方文档"Working with KLEE source code"中提到的
  ```
  Note that KLEE depends on LLVM and STP. If you need to debug KLEE’s calls to that code, then you will need to build LLVM/STP with debug support too
  ```
  我其实早注意这句,也重新编译过LLVM,并且在本llvmobj中bin目录下运行llvm-config --build-mode,返回的也是"Debgu"值,这2者为什么不一致?这个问题先放一放,将上面的ENABLE_OPTIMIZED=1修改为0,重新make,成功生成完整的Debug+Asserts.

#### 补充点知识
在configrue文件中打印语句:
```
echo ==============1=============
echo $变量
```
在makefile中打印
```
$(info $(VARNAME))
$(warning
$(error
$(echo  )
```
