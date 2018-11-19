---
title: "Qt .pro 文件(qmake) 入门和常用功能"
date: 2018-11-19T21:42:52+08:00
categories:
- Qt
tags:
- Qt
- qmake
- pro
- Makefile
---

# Qt `.pro` 文件(`qmake`) 入门和常用功能

## 简介

Qt 的工程是这样一个流程：

* 创建一个 `pro` 文件，使用 `qmake` 语法描述工程设置
* 运行 `qmake` 命令，这个命令的作用是把 `pro` 文件转换成一个 `Makefile` 文件
* 运行 `make` 命令，会用生成好的 `Makefile` 进行编译

可以在 Qt creator 的项目页面查看并编辑:

```sh
qmake /Users/***/git/myProject/myProject.pro -spec macx-clang CONFIG+=debug CONFIG+=x86_64 && /Library/Developer/CommandLineTools/usr/bin/make qmake_all
```

## qmake 的作用

* 生成 Makefile

  ```sh
  qmake -makefile
  ```

* 生成工程文件 (.pro)

  ```sh
  qmake -project
  ```

* 对 Qt 特殊机制的支持，自动加入构建 [moc](http://doc.qt.io/qt-5/moc.html) 和[uic](http://doc.qt.io/qt-5/uic.html)的构建规则
* 可以生成 Microsoft Visual studio / XCode 工程文件
  [参考: Platform Notes | qmake Manual](http://doc.qt.io/qt-5/qmake-platform-notes.html)

  ```sh
  # xcode
  qmake xx.pro -spec macx-xcode

  # vs project
  set QMAKESPEC=win32-msvc2008 # if needed
  qmake -tp vc mainprojectfile.pro

  # vs solution
  qmake -tp vc -r mainprojectfile.pro
  ```

## 基本语法

很简单不细说

* [qmake Language | qmake Manual](http://doc.qt.io/qt-5/qmake-language.html)
* [Creating Project Files | qmake Manual](http://doc.qt.io/qt-5/qmake-project-files.html)

## 常见功能用法

### 设定编译目标名称

默认名称为 `pro` 文件的名字，假如是一个 `exe` 工程，工程文件是 `hello.pro`，则默认编译为 `hello.exe`
可以用 `TARGET` 变量来指定名字.
目标文件名的后缀会根据目标操作系统自动添加

```qmake
TARGET = helloworld
```

### debug|release 分支

```qmake
CONFIG(release, debug|release): message("build on release mode")
CONFIG(debug, debug|release): message("build on debug mode")
```

### 多条件

```qmake
win32 {
    debug {
        CONFIG += console
    }
}

# 或者
win32:debug {
    CONFIG += console
}
```

### 设置应用程序图标

[参考: Setting the Application Icon | Qt 4.8](http://doc.qt.io/archives/qt-4.8/appicon.html)

```qmake
win32:RC_FILE += $${PWD}/src/resource/icon/appico.rc
macx:ICON = $${PWD}/src/resource/icon/appico.icns
```

### 使用预编译头文件

> 预编译头文件将一些项目中普遍使用的头文件内容的词法分析、语法分析等结果缓存在一个特定格式的二进制文件中
> 当然编译实质 `C`/`C++` 源文件时，就不必从头对这些头文件进行词法语法分析，而只需要利用那些已经完成词法 - 语法分析的结果就可以了。
> 对于大工程, 把基本上不变的内容放到预编译头文件，可以显著减少编译时间

```qmake
PRECOMPILED_HEADER = aaa.h
```

```cpp
/* aaa.h */

#ifdef __cplusplus
/* Add C++ includes here */

#include <QRect>
#include <QString>
#endif
```

### 添加宏定义

```qmake
DEFINES += MY_DEBUG
DEFINES += MY_LOG_LEVEL=3
```

### 添加 framework 依赖

[参考: linking mac framework to qt creator](https://stackoverflow.com/questions/3513907/linking-mac-framework-to-qt-creator)

```qmake
QMAKE_LFLAGS += -F/path/to/framework/directory/
LIBS += -framework TheFramework
```

### 添加 `lib`/`dll`/`dylib`

[参考: Third Party Libraries | Qt 5.10](http://doc.qt.io/qt-5/third-party-libraries.html)
[参考: How to link to a dll - Qt Wiki](https://wiki.qt.io/How_to_link_to_a_dll)

以 `protobuf-lite` 为例

```qmake
macx {
  LIB_PROTOBUF="$$_PRO_FILE_PWD_/libs/libprotobuf-lite.a"
  !exists($$LIB_PROTOBUF): error ("Not existing $$LIB_PROTOBUF")
  LIBS+= $$LIB_PROTOBUF
}
win32 {
  CONFIG(release, debug|release): LIB_PROTOBUF="$$_PRO_FILE_PWD_\libs\libprotobuf-lite.lib"
  CONFIG(debug, debug|release): LIB_PROTOBUF="$$_PRO_FILE_PWD_\libs\libprotobuf-lite-debug.lib"

  !exists($$LIB_PROTOBUF): error ("Not existing $$LIB_PROTOBUF")
  LIBS+= $$LIB_PROTOBUF
}
```

### 添加对 C++ 新标准的支持

添加 C++14 支持, C++11/17 同理 (注意: 需要编译器支持才能真正生效！)

```qmake
CONFIG += c++14
```

### 编译前 / 后执行自定义命令

```qmake
# 编译前执行 (例子里是拷贝命令)
QMAKE_PRE_LINK = cp - f [source] [destionation]

# 编译后执行 (例子里是拷贝命令)
QMAKE_POST_LINK = cp - f [source] [destination]
```

### 自动编译 ts 文件为 qm 文件

需要 lrelease 文件的目录已经加进了系统 PATH，而且需要有实际的编译动作才会执行。

```qmake
# (caution1: this requires dir of lrelease is in PATH)
# (caution2: this only run when there is something to compile, or this step will be omitted)
QMAKE_PRE_LINK += lrelease $$_PRO_FILE_
```

### 执行 qmake 时执行一个 shell 命令

```qmake
system("lrelease"$$_PRO_FILE_)
```

### 编译完成后拷贝一些文件到指定目录

`$${QMAKE_COPY_DIR}` 是一个平台无关的目录拷贝命令
`$${QMAKE_COPY}` 是一个平台无关的文件拷贝命令
`$${OUT_PWD}` 是编译输出目录

```qmake
QMAKE_POST_LINK += $${QMAKE_COPY_DIR} $${_PRO_FILE_PWD_}/doc/* $${OUT_PWD}/../otherdoc
```

多个命令可以用分号隔开

```qmake
QMAKE_POST_LINK += $${QMAKE_COPY_DIR} $${_PRO_FILE_PWD_}/util/* $${_PRO_FILE_PWD_}/../other; \
    $${QMAKE_COPY_DIR} $${_PRO_FILE_PWD_}/data/* $${_PRO_FILE_PWD_}/../other
```

类似的命令: [参考](https://stackoverflow.com/questions/20324061/where-are-variables-such-as-mkdir-and-copy-dir-defined)

```qmake
QMAKE_TAR               = tar -cf
QMAKE_GZIP              = gzip -9f

QMAKE_CD                = cd
QMAKE_COPY              = cp -f
QMAKE_COPY_FILE         = $$QMAKE_COPY
QMAKE_COPY_DIR          = $$QMAKE_COPY -R
QMAKE_MOVE              = mv -f
QMAKE_DEL_FILE          = rm -f
QMAKE_DEL_DIR           = rmdir
QMAKE_DEL_TREE          = rm -rf
QMAKE_CHK_EXISTS        = test -e %1 ||
QMAKE_CHK_DIR_EXISTS    = test -d    # legacy
QMAKE_MKDIR             = mkdir -p   # legacy
QMAKE_MKDIR_CMD         = test -d %1 || mkdir -p %1
QMAKE_STREAM_EDITOR     = sed
```

### 把文件打包进 mac 版本的 xx.app

```qmake
macx{
    MACX_RESOURCES.files = translations/test_zh.qm
    MACX_RESOURCES.path = Contents/Resources
    QMAKE_BUNDLE_DATA += MACX_RESOURCES
}
```

### 编译完成后拷贝文件的另一种方法 (不推荐)

```qmake
config.path    = $${DESTDIR}/config
config.files   = config/*
INSTALLS       += config
```

### 屏蔽编译警告 (clang)

先从输出面板看到编译警告信息 `xxx.cpp ...[Wunused-parameter]`, 然后在警告的大写 `W` 后面加个 `no-`

```qmake
contains(QMAKE_COMPILER, clang) {
    QMAKE_CXXFLAGS_WARN_ON += -Wno-unused-parameter \
        -Wno-unused-function \
        -Wno-inconsistent-missing-override \
        -Wno-reorder \
        -Wno-ignored-qualifiers \
        -Wno-undefined-var-template \
        -Wno-varargs
}
```

以上内容需要执行`make install` 才能起作用，而 Qt creator默认是执行 `make`
需要到项目页面，添加构建步骤 -> Make -> Make 参数: 设置为 `install`

这些构建步骤保存在 xxx.pro.user 文件里 [参考: Sharing Project Settings | Qt Creator Manual](http://doc.qt.io/qtcreator/creator-sharing-project-settings.html)

## Qt 构建系统的现状

* [Qt build systems at QtCon 2016 - Qt Wiki](https://wiki.qt.io/Qt_build_systems_at_QtCon_2016)

## 其他参考

* [qmake Manual](http://doc.qt.io/qt-5/qmake-manual.html)
* [Qt for Windows - Deployment | Qt 5.10](http://doc.qt.io/qt-5/windows-deployment.html)
* [Qt for macOS - Deployment | Qt 5.10](http://doc.qt.io/qt-5/osx-deployment.html)
