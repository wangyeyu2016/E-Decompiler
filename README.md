# IDA7.5支持中文函数命名的办法

https://www.52pojie.cn/thread-1414525-1-1.html

## 解除函数名称的限制

默认配置情况下我们是不能将函数名称修改为中文的，会提示bad character，意思就是包含非法字符，那么怎样让中文成为合法的字符呢？

通过在网上查找资料，这个合法字符的定义在ida.cfg文件中，我们定位到文件，搜索Block_CJK_Unified_Ideographs，如下图所示:

 ```
// the following characters are allowed in user-defined names:
NameChars =
        "$?@"           // asm specific character
        "_0123456789"
        "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
        "abcdefghijklmnopqrstuvwxyz",
        // This would enable common Chinese characters in identifiers:
        // Block_CJK_Unified_Ideographs,
        CURRENT_CULTURE;
```
我们去掉Block_CJK_Unified_Ideographs这一行前面的注释，就可以给函数名字起中文了。

解除IDA反汇编代码限制
虽然通过上述操作函数可以起名为中文，但是实际上使用F5功能的时候，得到的伪代码，中文函数名称却会变成下划线:


通过对IDA进行逆向得知，原来hexray在生成伪代码的时候会调用一个calc_c_cpp_name函数，该函数会试图针对C/C++的函数名称进行优化，结果却误伤中文字符，我们将此处代码给NOP掉，就可以了。

###
修改ida.dll中的calc_c_cpp_name
nop掉下面的`*v10 = '_';`
```
if ( !v8 || !strchr(" [](),*&", v11) )
      {
        if ( v10 - v9 < 8 || strncmp(v10 - 8, "operator", 8ui64) )
        {
LABEL_23:
          *v10 = '_';
          goto LABEL_24;
        }
        if ( strchr(" +-*/%&!^", v11) )
          goto LABEL_24;
        if ( v11 == 40 )
        {
          v13 = v10 + 1;
          if ( v10[1] != 41 )
            goto LABEL_23;
        }
```
Edit -> Patch Program -> Assemble
`mov     byte ptr [rbx], 5Fh` -> `nop` -> `nop` -> `nop`
Edit -> Patch Program -> Apply patches to input file...

# E-Decompiler

用来辅助分析易语言程序的IDA 7.5插件，实验性项目。

反编译思路一:

基于retdec项目，将代码转换成LLVM IR后，结合易语言函数特征编写LLVM PASS来修正LLVM IR。

反编译思路二:

基于Ghidra项目，将汇编代码转换成PCode后，结合易语言函数特征编写Action来修正PCode。

反编译思路三:

基于IDA SDK，通过修改函数声明和CTree结构，结合易语言函数特征来优化反编译代码。



目前在尝试的是思路三。

### 如何编译项目

开发环境为: Windows + Visual Studio 2019 + Qt 5.6.3.0 + IDA SDK75

- 因为IDA自身使用的QT版本为Qt 5.6.3.0，因此必须和它保持同步，目前有效的下载地址为:https://download.qt.io/new_archive/qt/5.6/5.6.3/qt-opensource-windows-x86-msvc2015_64-5.6.3.exe
- 使用了Qt Vs插件，目前有效的下载地址为:https://download.qt.io/official_releases/vsaddin/

配置选项:

- `C/C++` -> `附加包含目录` 需要添加IDA SDK的头文件目录，例如C:\MyLib\idasdk75\include
- `C/C++` -> `预处理器定义` 需要添加 `QT_NAMESPACE=QT`这一行
- `C/C++` -> `代码生成` -> `安全检查` 设置为`禁用安全检查(/GS-)`
- `链接器` -> `附加库目录` 需要添加IDA的Vc Lib文件目录和Qt Lib目录，例如C:\MyLib\idasdk75\lib\x64_win_vc_32和C:\MyLib\idasdk75\lib\x64_win_qt
- `链接器` -> `附加依赖项` 添加需要的lib文件，有ida.lib、Qt5Core.lib、Qt5Gui.lib、Qt5Widgets.lib、user32.lib...

### 使用说明

1. 在使用本插件之前，需要给IDA7.5做一个patch，使之支持中文函数。详细情况见[IDA7.5支持中文函数命名的办法](https://www.52pojie.cn/thread-1414525-1-1.html)
2. 将E-Decompiler.dll和esig文件夹放置于插件目录，例如D:\IDA 7.5 SP3\plugins
3. 运行IDA后，按Ctrl+3快捷键呼出插件菜单，运行插件即可。

### 项目进度

- [x] 使IDA支持中文函数命名。
- [x] 易语言控件的彻底解析。
- [x] 易语言库函数的特征码识别引擎移植。
- [ ] 易语言模块函数的特征码提取与识别。

反编译核心部分

- [x] 控件的交叉引用，控件事件的展示。
- [ ] 易语言基础特征函数的优化、易语言数组操作。
- [x] DLL命令调用函数的优化。
- [x] 错误回调函数的优化。

本项目完全是由个人利用业余时间开发，不定时更新。



### 关于模块函数特征识别

通过定制化开发一套轻量级的特征识别引擎，来实现模块函数特征识别。 



### 参考资料

https://chaotic.gd/ghidra-decompiler-docs-built/index.html

[Online Courses – Ghidra](https://ghidra.re/online-courses/)

[NationalSecurityAgency/ghidra: Ghidra is a software reverse engineering (SRE) framework](https://github.com/NationalSecurityAgency/ghidra)

[airbus-cert/Yagi: Yet Another Ghidra Integration for IDA](https://github.com/airbus-cert/Yagi)

[avast/retdec: RetDec is a retargetable machine-code decompiler based on LLVM. (github.com)](https://github.com/avast/retdec)

[RolfRolles/HexRaysDeob: Hex-Rays microcode API plugin for breaking an obfuscating compiler (github.com)](https://github.com/RolfRolles/HexRaysDeob)

[patois/HRDevHelper: Context-sensitive HexRays decompiler plugin that visualizes the ctree of decompiled functions. (github.com)](https://github.com/patois/HRDevHelper)

https://github.com/OpenEpl/TextECode

NjMzNzgzNjAy
