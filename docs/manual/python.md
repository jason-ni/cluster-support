# Python

Python 是一种简单易用的编程语言，现被广泛应用在网络编程、操作系统运维、科学计算和人工智能等领域。

[Anaconda][1] 是一个用于科学计算的Python发行版，支持 Linux, Mac, Windows系统以及 Python、R等科学计算语言，提供了包管理与环境管理的功能，可以很方便地解决多版本python并存、切换以及各种第三方包安装问题。Anaconda 利用 `conda` 命令来进行package和environment的管理，并且已经包含了Python和相关的配套工具。在集群上，我们建议用户使用 Anaconda 来管理和使用Python。我们已经在集群的 Anaconda 中安装好了常用的科学计算库，包括 jupyter、mpi4py、numpy、pandas、scikit-learn、xgboost等。

Anaconda环境需要几个G的存储空间，每个用户都自己安装一套Anaconda，将造成存储空间的浪费。用户自己编译安装的Python很有可能出现依赖错误，GCC版本过低等各种问题，且所编译的numpy等科学计算软件没有mkl加速，速度将会很慢。因此，这里推荐大家使用我们统一安装的Anaconda版的Python。如有其他个性化需求，请联系我们获取更多信息。

Python 入门教程请参考：

* [廖雪峰的Python教程][3]
* [菜鸟教程 Python2][4] / [菜鸟教程 Python3][5]

!!! tip "深度学习用户请使用容器"
        对于使用深度学习框架的用户，由于大多数框架对 Linux 操作系统版本要求高，目前无法直接在集群的操作系统上安装 TensorFlow 或 PyTorch 框架。网络上提供的修改GLIBC的方法也非常不安全，有可能导致你的环境崩溃，这里非常不建议。我们建议使用Singularity容器来运行你的计算任务。你可以忽略下文的教程，直接跳转到[Singularity](singularity.md)的页面。

加载 Anaconda：
```bash
module load anaconda/5.3.0
```

执行完此命令后，你已经可以使用我们提供的Python 3.7基础环境，此环境也安装有R 3.5和Julia 1.03。

## 在调度系统中提交 Python 作业

准备好一个 Python 程序，程序使用 xgboost 算法对 iris 数据集进行分类预测，输出准确度：

```python
from sklearn.datasets import load_iris
import xgboost as xgb
from xgboost import plot_importance
from matplotlib import pyplot as plt
from sklearn.model_selection import train_test_split

# read in the iris data
iris = load_iris()

X = iris.data
y = iris.target

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=1234565)

params = {
    'booster': 'gbtree',
    'objective': 'multi:softmax',
    'num_class': 3,
    'gamma': 0.1,
    'max_depth': 6,
    'lambda': 2,
    'subsample': 0.7,
    'colsample_bytree': 0.7,
    'min_child_weight': 3,
    'silent': 1,
    'eta': 0.1,
    'seed': 1000,
    'nthread': 4,
}

plst = params.items()
dtrain = xgb.DMatrix(X_train, y_train)
num_rounds = 500
model = xgb.train(plst, dtrain, num_rounds)

# 对测试集进行预测
dtest = xgb.DMatrix(X_test)
ans = model.predict(dtest)

# 计算准确率
cnt1 = 0
cnt2 = 0
for i in range(len(y_test)):
    if ans[i] == y_test[i]:
        cnt1 += 1
    else:
        cnt2 += 1

print("Accuracy: %.2f %% " % (100 * cnt1 / (cnt1 + cnt2)))

# 显示重要特征
#plot_importance(model)
#plt.show()
```

<font color=red >在你的程序所在的文件夹下</font>，编写 PBS 脚本如下，保存成名为`submit_python.sh`的文件：

```bash
#!/bin/bash

#PBS -N python_example
### use one node for this job
#PBS -l nodes=1:ppn=1
#PBS -q default

cd $PBS_O_WORKDIR
### set python in anaconda 
module load anaconda/5.3.0

python3 iris_with_xgb.py
```

上面这个脚本中`#PBS -l nodes=1:ppn=1`这行设置了申请多少CPU计算资源。申请过多的资源不但不会加速你的程序，而且还会导致自己和他人的作业排队；申请过少的资源，会导致你自己的作业运行较慢。Python语言默认只能使用多核处理器的单个核心，如果没有其他优化的话，ppn应该设置为1。如果你不确定自己应该申请多少计算资源，可以参考我们提供的[资源申请指南](../resources.md)。

提交这个作业：

```bash
qsub submit_python.sh
```

程序的结果将写入同文件夹的 python_example.oXXXX 文件中，报错信息将写入 python_example.eXXXX 文件中，其中 XXXX 为作业的id。你可以将这两个文件同步到你的个人电脑上查看，也可以在集群上使用 `cat python_example.oXXXX` 命令查看输出结果。

在上面所示的PBS脚本中用 `#PBS -N python_example` 来修改作业名，相应的输出文件名也会随之改变。

## 并行加速

在未做任何优化情况下，Python只能使用服务器众多核心的单个核心，即只能利用服务器 1/20 的计算能力。即使在PBS脚本中申请多个核心或者多个节点，也并不能加速你的程序。如果想利用服务器的多核，需要做一些优化，例如调用 multiprocessing 等包，才能充分利用多个CPU核心。

## 已安装依赖包

我们推荐用户使用我们在Anaconda中已经安装好的基础环境（base）。加载 Anaconda 后，默认使用的就是基础环境。基础环境中一些常用科学计算相关包如下所示：

| 包             | 版本    | 包             | 版本    |
|------------   |--------   |-------------- |--------   |
| gensim        | 3.7.1     | nltk          | 3.3.0     |
| gsl           | 2.4       | numpy         | 1.15.4    |
| hdf5          | 1.10.2    | openblas      | 0.3.3     |
| jieba         | 0.39      | pandas        | 0.23.4    |
| jupyter       | 1.0.0     | python        | 3.7.0     |
| matplotlib    | 3.0.2     | scikit-learn  | 0.20.2    |
| mkl           | 2019.1    | scipy         | 1.2.0     |
| mpi4py        | 3.0.0     | seaborn       | 0.9.0     |
| networkx      | 2.1       | xgboost       | 0.81      |

*表 base环境常用科学计算包*

可以使用 `conda list -n base` 查看基础环境已经安装的包。

此外，我们已经创建了 python36 和 python27 环境，环境中已经安装了常用的科学计算包。使用下面的代码激活该环境：

```bash
source activate python36
```

## 安装自己所需包

普通用户可以使用`pip`命令来安装包，因为不是管理员权限，所以注意要使用`--user`参数，将安装包安装到自己的家目录中。

```base
pip install --user jieba -i https://pypi.douban.com/simple
```

!!! tip "国内源"
    pip默认使用的官方源服务器在国外，相对比较慢。`-i https://pypi.douban.com/simple` 使用了豆瓣提供的国内Python安装源，可以大大加快安装速度。

用户可以将常用包告知我们，我们统一安装，也可以根据需要创建自己安装。一些新开发的包可能依赖更高版本的GLIBC，使用时会遇到 `version 'GLIBC_2.xx' not found` 错误，这时请使用Singularity容器，使用容器来运行你的计算任务，请参考我们的[Singularity](singularity.md)的页面。

一些包会依赖其他底层包，pip安装包时，并不会检查所依赖是否符合要求，因此会出现虽然pip安装成功，但是不能使用的问题，我们建议使用 conda 来安装包。

## 使用 conda 管理 Python 环境

如基础环境无法满足用户需求，也可以根据我们下面的教程来创建自己的环境。这里要使用 `conda` 来管理环境。

接下来我们用 `conda` 来安装和管理一个针对自己的Python环境：

```
conda create -n test_env python=3.6
```

conda 会在用户个人目录 `/home/~your-cluster-username~/.conda/envs/test_env` 里创建 Python 3.6 的环境，安装Python 3.6解释器、pip等软件。

使用 conda 继续安装所需软件包 `numpy`，-n 表示安装到刚刚创建的test_env的Python环境。

```bash
conda install -n test_env numpy
```

激活这个环境，激活后行首会出现`(your_env_name)`，表示激活成功：

```bash
[linpack@rmdx-cluster ~]$ source activate test_env
(test_env) [linpack@rmdx-cluster ~]$
```

在这个环境中也可以使用 `pip` 来安装自己想要安装的软件：

```bash
pip3 install jieba -i https://pypi.douban.com/simple
```

由于这个环境已经安装到了你自己的目录，可以不用 `--user` 参数。

conda 和 pip 都可以在你的环境中安装软件，这里推荐优先使用 conda 来安装，conda 会帮你安装所依赖的一些包，而 pip 却不会。

退出该环境：

```bash
source deactivate
```

!!! note "为什么要创建虚拟环境"
    conda 的虚拟环境提供了环境隔离。不同用户可以使用不同的包，同一用户也可以使用不同版本的包。对于 Python 这样一个发展非常快的开源社区，某个包很可能在短时间内有较大更新。用户编写的代码很可能是基于历史上某个特定版本的包，为保证用户代码正确执行，最好也要保证环境中所使用的包版本一致。虚拟环境为用户提供了一个可以解决上述问题的方案。虚拟环境隔离的功能也可以保证用户之间、环境和环境之间互相不冲突。

直接将base环境克隆，并起名为 test_env_2 ：


```bash
conda create --name test_env_2 --clone base
```

删除环境：

```bash
conda remove --name test_env_2 --all
```

!!! warning "注意！"
    如不激活任何环境，系统使用默认的基础环境（base）。有时会因为忘记执行激活用户环境的命令，导致程序因依赖包版本不对而报错，这时应该检查自己是否激活正确的环境。

## 使用Jupyter

Jupyter的使用请参考 [Jupyter使用方法](jupyter.md)。

[1]: https://www.anaconda.com/

[3]: https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000

[4]: http://www.runoob.com/python/python-tutorial.html

[5]: http://www.runoob.com/python3/python3-tutorial.html
