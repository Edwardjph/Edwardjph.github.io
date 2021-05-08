### 前提

**只设置根目录为Sources Root或者不设置Sources Root**

**导包报错的改为使用相对导包或者绝对导包**

### 打包步骤

#### 分发工具 setuptools

```
pip install setuptools
pip install wheel
```

#### 编写setup.py

```python
import os
import os.path as osp
import glob
import setuptools
from setuptools.extension import Extension
from Cython.Build import cythonize
from Cython.Distutils import build_ext
import sys
import shutil

build_dir = "build"


def get_extensions():
    this_dir = osp.dirname(osp.abspath(__file__))
    # get py list
    py_list = glob.glob(osp.join(this_dir, 'src/source/**', '*.py'), recursive=True)
    return py_list


def del_files(path):
    for i in os.listdir(path):
        if i.startswith('.'):
            continue
        # 如果是文件夹就递归下去
        if os.path.isdir(os.path.join(path, i)):
            del_files(os.path.join(path, i))
        # 删除.c文件
        ext = os.path.splitext(i)[1]
        if ext == ".c":
            os.remove(os.path.join(path, i))


BUILD_ONLY_SO = False
for arg in sys.argv[:]:
    if 'BUILD_ONLY_SO' in arg:
        BUILD_ONLY_SO = True
        sys.argv.remove(arg)

# exec(open('TI-Light/version.py').read())


if BUILD_ONLY_SO:
    setuptools.setup(
        name="TurboX Inspection",
        version="0.0.0",
        author="ThunderSoft",
        # author_email="tusson@163.com",
        description="TI Light system",
        cmdclass={'build_ext': build_ext},
        packages=['src.resources', 'src.source.interaction.utils.labelme.config',
                  'src.source.interaction.utils.labelme.icons'],
        package_data={
            '': ['*/*', '*'],
        },
        ext_modules=cythonize(get_extensions(),
                              build_dir="./build",
                              compiler_directives={'language_level': 3}),
        # long_description="DeepLearning model visual evaluation.",
        # long_description_content_type="text/markdown",
        # url="https://github.com/tusson/TI-Light",
        classifiers=[
            "Programming Language :: Python :: 3",
            "License :: OSI Approved :: MIT License",
            "Operating System :: OS Independent",
        ],
        zip_safe=False,
    )
    # 未确认打包成功之前，注释这句话同时不要删除打包生成的文件，避免二次打包时间过长
    del_files(osp.dirname(osp.abspath(__file__)))
else:
    setuptools.setup(
        name="TurboX Inspection",
        version="0.0.0",
        author="ThunderSoft",
        # author_email="tusson@163.com",
        description="TI Light system",
        # long_description="DeepLearning model visual evaluation.",
        # long_description_content_type="text/markdown",
        # url="https://github.com/tusson/TI-Light",
        packages=setuptools.find_packages(),
        package_data={
            'src': ['resources/*/*'],
            '': ['*.yaml']
        },
        classifiers=[
            "Programming Language :: Python :: 3",
            "License :: OSI Approved :: MIT License",
            "Operating System :: OS Independent",
        ],
        zip_safe=False,
    )

```

#### so 打包

```
python setup.py bdist_wheel BUILD_ONLY_SO
```

#### 源码 打包

```
 python setup.py bdist_wheel
```

### 踩坑记录

#### 一、error: Microsoft Visual C++ 14.0 is required

安装这个[Microsoft Visual C++ Build Tools 2015](http://go.microsoft.com/fwlink/?LinkId=691126)

#### 二、command 'C:\\Program Files (x86)\\Microsoft Visual Studio 14.0\\VC\\BIN\\x86_amd64\\link.exe' failed

- **首先**，进入到 C:\Program Files (x86)\Windows Kits\8.1\bin\x86
- **然后**，找到并复制这两个文件：rc.exe 和 rcdll.dll
- **最后**，粘贴到：C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC\bin

#### 三、LINK : error LNK2001: unresolved external symbol PyInit___init__

[通过Cython打包py文件，生成包含pyd的wheel(.whl) - 简书 (jianshu.com)](https://www.jianshu.com/p/ce39e39d7a51)

打开`安装目录\cython\Compiler\ModuleNode.py`，替换如下内容：

将原来的2315行

```python
        code.putln(header2)
        code.putln("#else") 
        code.putln("%s CYTHON_SMALL_CODE; /*proto*/" % header3)
```

替换为

```python
        if self.scope.is_package:
            code.putln("#if !defined(CYTHON_NO_PYINIT_EXPORT) && (defined(WIN32) || defined(MS_WINDOWS))")
            code.putln("__Pyx_PyMODINIT_FUNC init__init__(void) { init%s(); };" % env.module_name)
            code.putln("#endif")
        code.putln(header2)
        code.putln("#else")
        code.putln("%s CYTHON_SMALL_CODE; /*proto*/" % header3)
        if self.scope.is_package:
            code.putln("#if !defined(CYTHON_NO_PYINIT_EXPORT) && (defined(WIN32) || defined(MS_WINDOWS))")
            code.putln("__Pyx_PyMODINIT_FUNC PyInit___init__(void) { return %s(); };" % (
                self.mod_init_func_cname('PyInit', env)))
            code.putln("#endif")
```

#### 四、注意

- 如使用so打包，windows打包的只能在windows运行，linux打包的只能在linux运行
- 必须保证打包所用的python版本，与安装时使用的pip指向的python版本相同

