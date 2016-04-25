---
layout: article
title: Compile Everything on Max OS X
---

# Compile Everything on Max OS X

### LLVM and Clang
---

* 下载source code
    * 采用svn, 首先建立想存放LLVM代码的目录，然后运行`svn co`    
        `cd YourDir`    
        `svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm`
    * 等待下载完成。
    * 下载clang,放在 `llvm/tools/`下    
        `cd llvm/tools`     
        `svn co http://llvm.org/svn/llvm-project/cfe/trunk clang`
    * 等待下载完成
    * 下载compiler-RT,放在`llvm/projects/`下    
        `cd llvm/projects`    
        `svn co http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt`    
    * 等待下载完成
    * 还有一些可选的项目，可以参考上述方式下载。
* 准备工作
    * 需要用到cmake,若未安装cmake需先安装。若采用brew，则运行`brew install cmake`
    * 项目结构
        * bingdings/
        * cmake/
        * CMakeList.txt
        * CODE_OWNERS.TXT
        * configure
        * CREDITS.TXT
        * docs/
        * examples/
        * include/
        * lib/
        * LICENSE.TXT
        * llvm.spec.in
        * LLVMBuild.txt
        * projects/
        * README.txt
        * resources/
        * test/
        * tools/
        * unittests/
        * utils/
* `make`
    * 在当前目录下建立build目录
        * `mkdir build`
        * `cd build`
    * 执行`cmake -G "Unix Makefiles" ../`运行完成之后在build文件夹下会多出许多文件，如makefile等。
    * 执行`make`
    * 等待执行完成。
        * 最后一步类似于`[100%] Built target yaml2obj`
    * 此时build文件夹下存放的就是刚才编译好的llvm了。
* 完成    
    * 运行build/bin/下的`clang -v` 可以与系统自带的做对比。

### OpenJDK (Failed)
---

* 下载source code
    * 若采用mercurial，则运行如下命令。第一个命令只会下载readme,makefile等文件。脚本里的命令才真正clone项目的各个模块。    
        `hg clone http://hg.openjdk.java.net/jdk7u/jdk7u`         
        `cd jdk7u`     
        `sh get_source.sh`     
    * 若不采用mercurial，则去任何能下载到的地方下载source code.
* 准备工作
    * 项目结构
        * .(root)
            * contains common configure and makefile logic
        * Makefile
            - The top level Makefile is used to build the entire OpenJDK
        * Hotspot
            - contains source code and make files for building the OpenJDK Hotspot Virtual Machine
        * langtools
            - contains source code for the OpenJDK javac and language tools
        * jdk
            - contains source code and make files for building the OpenJDK runtime libraries and misc files
        * jaxp
            - contains source code for the OpenJDK JAXP functionality
        * jaxws
            - contains source code for the OpenJDK JAX-WS functionality
        * corba
            - contains source code for the OpenJDK Corba functionality
        * nashorn
            - contains source code for the OpenJDK JavaScript implementation
    * make sanity
        * 在编译之前可以先运行 `make sanity`
        * 该命令会对编译前环境进行检查
            - 如果出现`Sanity check passed` 则说明检查通过了。
            - 如果检查没通过，检查build文件夹下的sanityCheckErrors.txt,该文件会记载了导致检查失败的错误。build文件夹默认在当前文件夹中，也可以根据环境变量进行配置。
    * 环境变量
        * 编译之前有许多环境变量可以配置，部分是必须配置的，部分是可选的。具体可以参见`make sanity`检查的变量。
        * 建议可以将环境变量的配置写入脚本。
    * README-builds.html
        * 该文档记载了关于build的说明，可以参考。
    * ERRORS
        * 列举一些环境变量配置常见的错误。
        * ERROR: The Compiler version is undefined. 
            - 这个问题原因是Xcode5.0之后不再提供llvm-gcc与llvm-g++这两样东西，编译jdk是需要这两个所以一直出错。
            - 解决方案
                + 在Xcode的/usr/bin(/Applications/Xcode.app/Contents/Developer/usr/bin)下做一个ln -s的链接连到/usr/bin中的 llvm-g++ llvm-gcc中
                + `sudo ln -s /usr/bin/llvm-g++ /Applications/Xcode.app/Contents/Developer/usr/bin/llvm-g++`
                + `sudo ln -s /usr/bin/llvm-gcc /Applications/Xcode.app/Contents/Developer/usr/bin/llvm-gcc`
        * ERROR: Your JAVA_HOME environment variable is set.
            - 解决方案
                + 在make之前unset掉JAVA_HOME和CLASSPATH
        * ERROR: FreeType version  2.3.0  or higher is required.
            - 解决方案
                + 安装新版FreeType. 
                + 配置两个环境变量
                + `export ALT_FREETYPE_HEADERS_PATH=/usr/local/include/freetype2`
                + `export ALT_FREETYPE_LIB_PATH=/usr/local/Cellar/freetype/2.6.3/lib`
                + 此处是用brew安装的FreeType，具体路径根据安装的位置。
        * ERROR: You do not have access to valid Cups header files
            - 解决方案
                + 下载CUPS
                + 进入CUPS目录，执行`make`
                + 设置环境变量,设置为make完成之后的CUPS所在目录
                + `export ALT_CUPS_HEADERS_PATH=$YOURCUPSFOLDER`
* `make`（FAILED）
    * 进入项目所在目录，执行`make`
    * 报错(节选)
        * `/jdk7u/hotspot/src/share/vm/adlc/adlparse.cpp:3217:71: error: equality comparison with extraneous parentheses [-Werror,-Wparentheses-equality]`
        * `if ( ((primary = get_ident_or_literal_constant("primary opcode")) == NULL) ) {    
        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^~~~~~~ `
        * `make[8]: *** [../generated/adfiles/filebuff.o] Error 1`


