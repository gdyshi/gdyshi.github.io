---
layout: post
title:  "python调用第三方动态库"
categories: python
tags:  python 混合编程 动态库 C
---

* content
{:toc}

## 摘要
本文讲述python混合编程之调用动态库

## 引言
python因为良好的编码性和扩展库正被大规模的使用，但他有两个缺点：1、代码可见；2、执行效率低，于是在实际应用中经常会把高效和核心代码用C/C++实现，业务部分用python实现。这就需要进行混合编程，本文对python调用动态库的方法及注意事项进行记录

## 主题
python标准库函数中提供了调用动态库的包————[ctypes](https://docs.python.org/3/library/ctypes.html)

### 加载动态库
* 查找动态库`ctypes.util.find_library`
* 根据动态库调用方式的不同，可以分为`cdecl`和`stdcall`两种，这两种方式的主要区别见下表。后面的例子以cdecl调用方式为例，stdcall类同。

  | 调用标准  | 内存栈维护者                      | 函数名              |
  | :------------- | :------------- | :------------- |
  | cdecl | 调用者                 | 前面加下划线，后面加“@”符号和参数的字节数      |
  | stdcall |被调用者           | 在输出函数名前面加下划线      |

* ctypes加载动态库有两种方式。构造类对象`libc = CDLL("libtestlib.dll")`和实例化instance`libc = cdll.LoadLibrary("libtestlib.dll")`。这两种方式都会返回一个动态库操作的句柄，以供后面使用

### 类型
* 常规参数类型

C常规参数类型与ctypes、python类型对应关系表如下：

| C type                                 | ctypes type  | Python type              |
|:---------------------------------------|:-------------|:-------------------------|
| _Bool                                  | c_bool       | bool (1)                 |
| char                                   | c_char       | 1-character bytes object |
| wchar_t                                | c_wchar      | 1-character string       |
| char                                   | c_byte       | int                      |
| unsigned char                          | c_ubyte      | int                      |
| short                                  | c_short      | int                      |
| unsigned short                         | c_ushort     | int                      |
| int                                    | c_int        | int                      |
| unsigned int                           | c_uint       | int                      |
| long                                   | c_long       | int                      |
| unsigned long                          | c_ulong      | int                      |
| __int64 or long long                   | c_longlong   | int                      |
| unsigned __int64 or unsigned long long | c_ulonglong  | int                      |
| size_t                                 | c_size_t     | int                      |
| ssize_t or Py_ssize_t                  | c_ssize_t    | int                      |
| float                                  | c_float      | float                    |
| double                                 | c_double     | float                    |
| long double                            | c_longdouble | float                    |
| char * (NUL terminated)                | c_char_p     | bytes object or None     |
| wchar_t * (NUL terminated)             | c_wchar_p    | string or None           |
| void *                                 | c_void_p     | int or None              |

* 指针
  * 弱引用指针`byref()`，这种方式速度更快
  * 强指针`pointer()`
  * c_types定义的指针`c_char_p`、`c_wchar_p`、`c_void_p`不可修改，如果需要在C函数中被修改，需要使用函数`create_string_buffer()`创建可变内存
  * 用指针的`value`属性获取指针指向的内容

* 结构体/联合体
  * 通过构建类继承`Structure`/`Union`来实现结构体的定义，把结构体属性组合成元组数组放在类中的`_fields`属性中。关于结构体的其它特性（对齐、指针嵌套、位域等）请参照[官网](https://docs.python.org/3/library/ctypes.html)
  ```
  class POINT(Structure):
       _fields_ = [("x", c_int),
                   ("y", c_int)]
  class RECT(Structure):
       _fields_ = [("upperleft", POINT),
                   ("lowerright", POINT)]
  ```
* 数组直接使用`类型 * 元素个数`的方式来定义，如`array = c_int * 10`

* 函数指针（回调函数）`CFUNCTYPE`
  * 使用注释的方式一次定义
    ```
    @CFUNCTYPE(c_int, c_int)
    def py_cb_func(a):
        print("py_cb_func", str(a))
        return a + 1
    ```
  * 标准方式
    ```
    # 定义c_types类型
    PY_CB_FUNC = CFUNCTYPE(c_int, c_int)
    def py_cb_func(a):
        print("py_cb_func", str(a))
        return a + 1
    cb_func = PY_CB_FUNC(py_cmp_func)
    ```

### 函数
* 函数入参声明`libtest.parm_int.argtypes = [c_int]`，做了函数声明后，会在python调用时对参数格式进行检查
* 函数返回值类型声明`libtest.parm_int.restype = c_int`，做了函数声明后，会在python调用时对参数格式进行检查
* 函数调用`ret = libtest.parm_int(c_int(1))`

## 实例
我专门针对c_types调用动态库写了一个[实例](https://github.com/gdyshi/python_expreiment/tree/master/library_verification/c_types)。实例包括两个部分
- 用C实现的动态库代码。主要从入参类型、返回值类型、回调函数调用三个方面实现及提供接口
- 用python实现的对动态库调用代码。用于调用动态库的所有接口

## 附录


## 参考
---
- [python标准库](https://docs.python.org/3/library/index.html)
- [ctypes官方文档](https://docs.python.org/3/library/ctypes.html)
