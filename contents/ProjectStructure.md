# WebKit的项目结构

首先去下载一份最新的WebKit代码。

```shell
git clone https://github.com/WebKit/WebKit.git
```
根目录大概长这样
```shell
-rw-r--r-- 1 mbbill 197121   2033 Apr 27 19:20 CMakeLists.txt
-rw-r--r-- 1 mbbill 197121 279658 Apr 27 19:20 ChangeLog
-rw-r--r-- 1 mbbill 197121 794798 Mar 23  2016 ChangeLog-2012-05-22
-rw-r--r-- 1 mbbill 197121 782150 Mar 20  2019 ChangeLog-2018-01-01
-rw-r--r-- 1 mbbill 197121  75864 Apr 27 19:20 Introduction.md
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:21 JSTests
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:21 LayoutTests
-rw-r--r-- 1 mbbill 197121    607 Apr 27 19:22 Makefile
-rw-r--r-- 1 mbbill 197121   4511 Apr 27 19:22 Makefile.shared
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 ManualTests
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 PerformanceTests
-rw-r--r-- 1 mbbill 197121   6172 Apr 27 19:22 ReadMe.md
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 Source
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 Tools
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 WebDriverTests
drwxr-xr-x 1 mbbill 197121      0 Mar 20  2019 WebKit.xcworkspace
drwxr-xr-x 1 mbbill 197121      0 Oct  5  2020 WebKitBuild
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 WebKitLibraries
drwxr-xr-x 1 mbbill 197121      0 Mar 20  2019 Websites
drwxr-xr-x 1 mbbill 197121      0 Apr 27 19:22 resources
```

## WebKit 的组件和依赖关系

上面这份WebKit代码的根目录里面囊括了几乎所有的组件。很显然我们不可能直接入手一个个目录过一遍，所以先捋一下WebKit最重要的一些组件和他们之间的依赖关系，然后我们再来对照着这些组件整理代码。