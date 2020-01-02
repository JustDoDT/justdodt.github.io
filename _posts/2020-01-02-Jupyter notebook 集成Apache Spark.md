## Jupyter Notebook 集成Apache Spark

## 1.简介

>Jupyter Notebook是基于网页的用于交互计算的应用程序。其可被用于全过程计算：开发、文档编写、运行代码和展示结果。----[Jupyter Notebook官网介绍](<https://jupyter-notebook.readthedocs.io/en/stable/notebook.html>)

简而言之，Jupyter Notebook是以网页的形式打开，可以在网页页面中**直接**编写代码和运行代码，代码的运行结果也会直接在代码块下显示。如在编程过程中需要编写说明文档，可在同一个页面中直接编写，便于作及时的说明和解释。**`她包括两大部分：`**

#### 网页应用

网页应用即基于网页形式的、结合了编写说明文档、数学公式、交互计算和其他富媒体形式的工具。**简言之，网页应用是可以实现各种功能的工具。**

#### 文档

即Jupyter Notebook中所有交互计算、编写说明文档、数学公式、图片以及其他富媒体形式的输入和输出，都是以文档的形式体现的。

这些文档是保存为后缀名为`.ipynb`的`JSON`格式文件，不仅便于版本控制，也方便与他人共享。

此外，文档还可以导出为：HTML、LaTeX、PDF等格式。

### 2.安装Jupyter Notebook

**基本包安装：**

~~~shell
yum update -y
yum install python-pip -y
yum install bzip2 -y
yum groupinstall "Development Tools" -y
~~~

**安装完pip后，把pip源修改未国内源**

~~~shell
mkdir ~/.pip
cat > ~/.pip/pip.conf << EOF
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com
EOF
~~~

**用pip安装IPython，Jupyter Notebook**

~~~shell
pip install ipython jupyter notebook
~~~

#### 修改配置文件

**修改Jupyter Notebook的密码**

~~~shel&#39;l
[root@hadoop001 ~]# ipython
python 3.7.4 (default, Aug 13 2019, 20:35:49) 
Type 'copyright', 'credits' or 'license' for more information
IPython 7.8.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: from IPython.lib import passwd                                                                                                                    

In [2]: passwd()                                                                                                                                          
Enter password: 
Verify password: 
Out[2]: 'sha1:56782a0837b7:c8e3ac3857f6ee6d241c036b3423e7f3e75f55c5'

~~~

**生成配置文件并修改**

~~~shell
[root@hadoop001 ~]# jupyter notebook --generate-config
Writing default config to: /root/.jupyter/jupyter_notebook_config.py
#生成的config file在/root/.jupyter/jupyter_notebook_config.py
~~~

`vim /root/.jupyter/jupyter_notebook_config.py`   

**修改值如下：**

~~~shell
c.NotebookApp.ip = '192.168.100.111'
c.NotebookApp.allow_root = True #默认为False，但是用默认值并且用root启动的时候需要加上--allow-root
c.NotebookApp.open_browser = False
c.NotebookApp.port = 8888
c.NotebookApp.password = 'sha1:56782a0837b7:c8e3ac3857f6ee6d241c036b3423e7f3e75f55c5' #输入上面加密后得到的密文
c.ContentsManager.root_dir = '/root/.jupyter/'
~~~

 **重新启动**

~~~shell
[root@hadoop001 ~]# jupyter notebook  
~~~



![1577898126703](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1577898126703.png)

输入密码即可登陆

## 3.安装Anaconda（可选）

也可以直接安装Linux版的Anaconda，因为Anaconda里面自带了Jupyter Notebook,IPython。Anaconda广泛用于数据科学中。

[安装文档](<https://docs.anaconda.com/anaconda/install/linux/>)

下载地址：<https://www.anaconda.com/distribution/>

~~~shell
bash Anaconda3-2019.10-Linux-x86_64.sh
~~~

然后一路回车，Yes即可。安装完毕后也需要修改Jupyter Notebook的配置文件。

### 3.安装Apache Toree

#### 什么是Apache Toree

>Apache Toree的主要目标是为交互式应用程序提供连接和使用Apache Spark。该项目旨在为应用程序提供发送打包的jar和代码的功能。由于Apache Toree实现了最新的Jupyter消息协议，因此可以轻松地集成Jupyter生态系统，以进行快速，交互式的数据浏览。Apache Toree 自动为您创建一个SparkContext对象，可以通过sc变量直接访问。
>
>https://toree.apache.org/docs/current/user/installation/

#### 安装Apache Toree

~~~shell
pip install --upgrade toree
jupyter toree install --spark_opts='--master=spark://localhost:7077' --user --kernel_name=Spark2.3.2 --spark_home=/home/hadoop/app/spark-2.3.2-bin-hadoop2.7
~~~

#### 检查Jupyter

~~~shell
[root@hadoop001 ~]# jupyter kernelspec list
Available kernels:
  spark2.3.2_scala      /root/.local/share/jupyter/kernels/spark2.3.2_scala
  python3               /root/anaconda3/share/jupyter/kernels/python3
  apache_toree_scala    /usr/local/share/jupyter/kernels/apache_toree_scala
 
~~~



![1577944802305](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1577944802305.png)

###4.创建Spark项目



![1577945417909](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1577945417909.png)



#### 在Spark Web查看项目

![1577948322269](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1577948322269.png)





![1577948358591](C:\Users\HUAWEI\AppData\Roaming\Typora\typora-user-images\1577948358591.png)



### 参考文档

<https://toree.apache.org/>

<https://www.anaconda.com/>

https://jupyter-notebook.readthedocs.io/en/stable/notebook.html





















