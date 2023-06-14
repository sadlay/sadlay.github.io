---
layout: wiki
title: Conda
categories: Python
description: Conda安装和使用
keywords: Python, Conda
---

### 生成requirements.txt文件

用conda activate 你的环境名字，此时进入了你的环境中，然后使用代码：

```shell
pip freeze > requirements.txt
```

### 安装requirement.txt文件的扩展包

```shell
pip install -r requirements.txt
```

除了使用pip命令来生成及安装requirement.txt文件以外，也可以使用conda命令来安装。

```shell
conda install --yes --file requirements.txt
```

但是这里存在一个问题，如果requirements.txt中的包不可用，则会抛出“无包错误”。

使用下面这个命令可以解决这个问题

```shell
$ while read requirement; do conda install --yes $requirement; done < requirements.txt
```

如果想要在conda命令无效时使用pip命令来代替，那么使用如下命令：

```shell
$ while read requirement; do conda install --yes $requirement || pip install $requirement; done < requirements.txt
```

有时可以导出conda环境，导出格式为.yml文件

```shell
conda env export > requirements.yml
```

此时你的电脑需要这个conda环境，可以直接用这个yml文件在你的电脑上创造出一个同名字，同扩展包的环境，你只需要进入cmd，然后直接运行下面代码就可以了：

```shell
conda env create -f requirements.yml
```
