---
layout:     post
title:      "安装并使用jupyter notebook"
subtitle:   " \"Learning python with jupyter notebook\""
date:       2017-07-22 18:00:00
author:     "hxwhou"
header-img: "img/post-bg-jupyter-notebook.png"
header-mask: 0.2
wechat-qrcode: "img/wechat-qrcode.jpg"
catalog: true
tags:
    - python
    - jupyter
    - notebook
    - tools
---

Jupyter Notebook, 以前又称为IPython notebook,是一个交互式笔记本, 支持运行40+种编程语言. 可以用来编写漂亮的交互式文档.

Linux/Mac下, Jupyter Notebook的安装过程可以参考[Jupyter官方网站](https://jupyter.readthedocs.io/en/latest/install.html#id4), 可能只需要下面两步就能搞定:
> First, ensure that you have the latest pip; older versions may have trouble with some dependencies:
> ```
> pip install --upgrade pip
> ```
> Then install the Jupyter Notebook using:
> ```
> pip install jupyter
> ```

安装完之后, 在终端运行`jupyter notebook`即可打开jupyter notebook，如下图所示；
![img](/img/in-post/20170722/01.png)

此时，我们就可以使用其来学习python了。速速写一个hello world压压惊🤔

![img](/img/in-post/20170722/02.png)

可以看到在notebook中我们就可以完成交互式展现。是否觉得美好了很多，其实这只是jupyter强大功能的冰山一角。更多介绍请[参考](http://python.jobbole.com/87527/?repeat=w3tc)

从上图可以看到，此时的jupyter环境是Python2，即这里是Python2的Kernel。
那么当你想学习Python3的时候，显然是不能满足需求的，这就需要为Jupyter安装Python3的Kernel。
首先，确保已安装Python3
接着，执行如下命令安装ipykernel
```
sudo python3 -m pip install ipykernel
sudo python3 -m ipykernel install --user
```
安装完成后，重启jupyter，会发现Python3的Kernel，如下图

![img](/img/in-post/20170722/04.png)

If you’re running Jupyter on Python 3, you can set up a Python 2 kernel like this:
```
python2 -m pip install ipykernel
python2 -m ipykernel install --user
```

##Q&A:
使用jupyter的过程中出现的一些问题与解决方案：
Q1.运行`jupyter notebook`报错：Error executing Jupyter command 'notebook': [Errno 2] No such file or direct

A:执行下面的命令从新安装jupyter
```
sudo pip install --upgrade --force-reinstall --no-cache-dir jupyter
```