title: "使用Distutils（setup.py）打包Python项目为rpm包简介"
date: 2017-2-21 20:19:28
tags: [Linux,python]

---

将python项目打包为rpm包有两种方法，一种是采用传统的rpmbuild工具构建rpm包， 这种方法需要自定义的配置文件SPEC文件，其缺点在于针对Python的项目的构建功能相对冗余，其中内置的工具使用复杂。本文将简要的介绍相对简单并切合Python实际使用场景的使用Distutils构建rpm包的方法。

<!--more-->

## 1.名词定义

1. 模块：python中的基本代码单元，一般为一个python文件。
2. 扩展模块：由其他语言（C++/JAVA）对Python进行扩展的模块。
3. 包：包含多个模块的包，也即是Python的包。
4. root 包：类似于包的文件夹，但是不是标准的python的包（没有\_\_init\_\_.py文件）。

## 2.打包流程
利用setup.py将python项目打包成rpm包主要包括以下步骤：

1.建立一个文件夹，将python项目代码拷贝到该目录下，并在该目录下建立setup.py文件，如下所示的就针对一个由多个模块组成的python项目打包所需的setup的代码。

```python
from distutils.core import setup  

setup(name='maerd',
      version='1.0',    
      description='maerd',    
      author='yunhuanran',    
      author_email='yunhuanran@gmail.com',    
      url='http://yunhuanran.github.io/',    
      packages=['maerd','maerd.cmd','maerd.common','maerd.maerd_agent'],
      data_files=[('/etc/maerd',['maerd.conf'])],
)
```

2.在该目录下执行`python setup.py bdist_rpm`生成rpm包。注意该命令需要系统支持rpm，也需要系统安装了rpmbuild。

## 3.脚本讲解
1. `name`：定义项目的名称。
2. `version`：定义项目的版本。
3. `description`：项目的描述。
4. `author`：项目的作者。
5. `author_email`:项目作者的邮箱。
6. `url`：项目的url。
7. `packages`：该字段为一个列表，其中每个字段表示包的名称，索引方式为`setup.py`文件所在的目录的相对路径。除了`.`以外也可以用`/`表示相对路径。
8. `py_modules`:该字段为一个列表，其中每个字段表示模块的名称，索引方式为`setup.py`文件所在的目录的相对路径。
9. `data_files`：该字段表明除了项目源码之外的文件，其中一个元组中的第一个值表示该文件在安装后需要安装到的路径，第二个值表示该文件相对`setup.py`所在的路径。
10. `ext_modules`:标记扩展模块。
