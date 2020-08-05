
| 该文档是鹏城实验室AAH团队的benchmark说明文档 |
| :-------------: |
# AAH Benchmark v1.0



## Benchmark结构设计
AAH Benchmark基于微软NNI开源框架，以自动化机器学习（AutoML）为负载，使用network morphism进行网络结构搜索和TPE进行超参搜索。

Master节点将模型历史及其达到的正确率发送至Slave节点。Slave节点根据模型历史及其正确率，搜索出一个新的模型，并进行训练。Slave节点将根据某种策略（如连续10个Epoch的测试集正确率没有提升）停止训练，并将此模型及其达到的正确率发送至Master节点。Master节点接收并更新模型历史及正确率。
现有NNI框架在模型搜索阶段在Master节点进行，该特性是的AutoML作为基准测试程序负载时成为了发挥集群计算能力的瓶颈。为提升集群设备的计算资源利用率，项目组需要从减少Master节点计算时间、提升Slave节点GPU有效计算时间的角度出发，对AutoML框架进行修改。主要分以下特性：
将网络结构搜索过程分散到Slave节点上进行，有效利用集群资源优势；

1. 将每个任务的模型生成与训练过程由串行方式改为异步并行方式进行，在网络结构搜索的同时使得GPU可以同时进行训练，减少GPU空闲时间；
2. 将模型搜索过程中进行结构独特性计算部分设置为多个网络结构并行计算，减少时间复杂度中网络结构个数（n）的影响，可以以并发个数线性降低时间负载度；
3. 为从根本上解决后期模型搜索时需要遍历所有历史网络结构计算编辑距离的问题，需要探索网络结构独特性评估的优化算法或搜索效率更高的NAS算法，将其作为NAS负载添加至Benchmark框架中。

为进一步提升设备的利用率、完善任务调度的稳定性，修改、完善了调度代码，将网络结构搜索算法分布到每个slave节点执行，并采用slurm分配资源、分发任务。

### Benchmark模块结构组成
源代码（AAH/src）：AAH主体模块为src模块，该模块包含了整个AAH主体框架

参数初始化（AAH/examples/trials/network_morphism/imagenet/config.yml）：在AAH运行之前对参数进行调整

日志&结果收集（AAH/scripts/reports）： 在AAH运行结束后将不同位置的日志和测试数据统一保存在同一目录下

数据分析（AAH/scripts/reports）： 对正在运行/结束的测试进行数据分析，得出某一时间点内该测试的Error、FLOPS、Score，并给出测试报告

资源监控： 监控测试过程中的硬件资源使用，有助于测试分析和发现瓶颈 

	1. (必需)自动化脚本资源监控（AAH/examples/trials/network_morphism/imagenet/resource_monitor.py）
 	2. (可选) 可视化资源监控（AAH/scripts/monitor）

### NOTE：
后续文档的主要內容由Benchmark环境配置、安装要求，测试规范，报告反馈要求以及必要的参数设置要求组成

## 1 Benchmark环境配置、安装要求
Benchmark运行环境由Master节点-Slaves节点组成，其中Mater节点不参与调度不需要配置GPU/加速卡，Slave节点可配置多块加速卡。

### 运行环境配置、安装要求
Benchmark运行时，需要先获取虚拟资源各节点信息（包括IP、环境变量等信息），根据各节点信息组建slurm调度环境，以master节点为slurm控制节点，各slave节点为slurm的计算节点。以用户的共享文件目录作为数据集、实验结果保存和中间结果缓存路径。
同时Master节点分别作为Benchmark框架和slurm的控制节点，根据实验配置文件中的最大任务数和slurm实际运行资源状态分配当前运行任务（trial）。每个trial分配至一个slave节点，trial的训练任务以节点中8GPU数据并行的方式执行训练。

本文默认在root用户下执行
每个节点部署直接安装或一个容器（容器推荐基于镜像nvidia/cuda:10.1-cudnn7-devel-ubuntu16.04）

#### 所有节点安装基础工具

```
apt update && apt install git vim cmake make openssh-client openssh-server wget curl sshpass -y
```
##### 配置ssh-server
开启ssh root登录权限,修改ssh配置文件 /etc/ssh/sshd_config

```
vim /etc/ssh/sshd_config
```
找到PermitRootLogin prohibit-password所在行，并修改为

```
#PermitRootLogin prohibit-password
PermitRootLogin yes
```
##### 重启ssh服务

```
service ssh restart
```
#### 配置共享文件系统

若集群环境中已有共享文件系统则跳过配置共享文件系统的步骤,若无共享文件系统，则需配置共享文件系统。
##### 搭建NFS
AAH使用NFS共享文件系统进行数据共享和存储
##### 安装NFS服务端
将NFS服务端部署在master节点

```
apt install nfs-kernel-server -y
```

###### 配置共享目录
创建共享目录/userhome，后面的所有数据共享将会在/userhome进行

```
mkdir /userhome
```

###### 修改权限

```
chome -R 777 /userhome
```
###### 配置NFS
打开NFS配置文件

```
vim /etc/exports
```
添加以下内容

```
/userhome   *(rw,sync,insecure,no_root_squash)
```

###### 重启NFS服务

```
service nfs-kernel-server restart
```
##### 安装NFS客户端
所有slave节点安装NFS客户端

```
apt install nfs-common -y
```
slave节点创建本地挂载点

```
mkdir /userhome
```

slave节点将NFS服务器的共享目录挂载到本地挂载点/userhome

```
mount NFS-server-ip:/userhome /userhome
```
如果使用的跨节点的docker，除了需要在物理机安装NFS外，还需要在创建容器时使用
```
-v /userhome:/userhome
```
将共享目录同步到容器中
#### 配置python运行环境
##### 所有节点安装python3.5

```
apt install --install-recommends python3 python3-dev python3-pip -y
```
##### 安装python环境库
###### 升级pip

```
pip3 install --upgrade pip
```

###### 安装环境库
```
pip3 install -r AAH/requirements.txt
```

### 编译&安装AAH

```
git clone http://github.com/pcl-ai-public/AAH.git
cd AAH
source install.sh
```

### 所有节点安装slurm
AAH的资源调度通slurm进行
#### 安装slurm、munge

```
apt install munge slurm-llnl -y
```
#### master节点创建munge秘钥

```
/usr/sbin/create-munge-key -r
```
秘钥路径为/etc/munge/munge.key,使用scp将master节点上的munge.key拷贝到所有节点的相同路径下

```
scp /etc/munge/munge.key root@192.168.116.10:/etc/munge
```
#### 配置slurm
以下操作在master节点进行

将AAH源码拷贝到共享目录/userhome下，因为脚本所生成的slurm.conf会被所有节点链接使用，进入AAH/script/autoconfig_slurm目录

```
cd /userhome/AAH/script/autoconfig_slurm
```

##### 进行ip地址配置
1. 将所有slave节点ip按行写入slaveiip.txt。
2. 将master节点ip写入masterip.txt。
3. 确保所有节点的ssh用户、密码、端口是一致的，并根据自身情况修改 slurm_autoconfig.sh脚本中的用户名和密码。
##### 运行自动配置脚本
```
bash slurm_autoconfig.sh
```
#### 运行后检查
执行命令查看所有节点状态
```
sinfo
```
如果所有节点STATE列为idle则slurm配置正确，运行正常。

如果STATE列为unk，等待一会再执行sinfo查看，如果都为idle，则slurm配置正确，运行正常。

### 目录调整
#### 创建必要的目录
mountdir 存放实验过程数据，nni存放实验过程日志
```
mkdir /userhome/mountdir
mkdir /userhome/nni
```
所有节点将共享目录下的相关目录链接到用户home目录下
```
ln -s /userhome/mountdir /root/mountdir
ln -s /userhome/nni /root/nni
```
#### 必要的路径及数据配置

将权重文件复制到共享目录/userhome中

```
cp /userhome/AAH/examples/trials/network_morphism/imagenet/weights/resnet50_weights_tf_dim_ordering_tf_kernels.h5  /userhome
```

将下载好的ImageNet ILSVRC2012数据集存放到/userhome中。

### 资源监控程序

#### resource_monitor(必须)
resource_monitor.py监控程序源码需跟用例源码放在同级目录(AAH/examples/trials/network_morphism/imagenet)，在启动AAH时自动在每个slave节点启动，并将测试过程中的cpu、内存、GPU的信息记录在 /userhome/mountdir/device_info
/experiments/experiment_ID目录下，请注意在后面进行运行参数配置修改AAH/examples/trials/network_morphism/imagenet/config.yml文件时，需要将command行的srun参数 --cpus-per-task 设置成当前可用cpu减1，slurm需要空出一个CPU运行resource_monitor.py监控程序。

#### prometheus&grafana(可选)

资源监控程序使用prometheus收集硬件资源信息，通过grafana将各节点收集到的信息图形化显示在web UI上，有助于在测试过程中对硬件资源使用的实时监控、发现性能瓶颈。

prometheus官方网站: https://prometheus.io

grafana官方网站: https://grafana.com

nvidia官方网站: https://www.nvidia.cn
##### 下载安装包
为保证监控程序运行正常，现提供我们支持的版本下载:

百度云：：https://pan.baidu.com/s/186bIuqaguoT9j31q-s10wg, 提取码：94be

所有监控依赖的安装包下载至路径脚本当前路径 AAH/scripts/monitor 的.monitortmp文件夹下，可以直接在浏览器下载好之后拷贝到当前路径

##### slave节点执行安装脚本
```
cd  AAH/scripts/monitor
bash monitor_slave_run.sh -i 安装路径 
```
##### master节点执行安装脚本

1）在master节点本地的prometheus.yml配置文件，增加每台slave节点的数据配置，包括ip:9100和ip:9400（将ip改为实际值)
```
  - job_name: 'node-exporter'
    static_configs:
    - targets: ['ip1:9100'，'ip2:9100']
  - job_name: 'GPU-exporter'
    static_configs:
    - targets: ['ip1:9400'，'ip2:9400']  
````

2）执行安装脚本
```
cd  AAH/scripts/monitor
bash monitor_master_run.sh -i 安装路径 
```
##### 访问grafana查看资源信息
打开浏览器访问 master_ip:3000,初始账号密码为admin/admin;

##### grafana增加数据源
在左侧菜单栏按顺序点击以下按钮

configuration ->Data Sources 

点击Prometheus,在URL框中填入master_ip:9090，点击 Save & Test 按钮
##### 导入模板文件
在左侧菜单栏按顺序点击以下按钮

Create -> Import 

点击 Upload .json file 导入 'AAH/scripts/monitor/monitor.json'

点击 load 即可看到监控的资源使用情况

##### 其他操作
###### 重启资源监控
当机器重启后监控服务会被关闭，需要手动启动
####### master重启服务
进入到先前安装指定的路径执行
```
/prometheus &
service grafana-server restart
```
###### slave重启服务
```
./node_exporter &
dcgm-exporter &
```

## 2 Benchmark测试规范

1. 经过多次8/16/32/64/128卡(tesla v100-32G )规模的测试， 在12小时后正确率会开始收敛， 因此建议测试运行时间应不少于12小时；

2. 测试用例的训练精度应不低于float16；

3. 测试用例初始的 “batch size” ，建议设置为 gpu显存*14 ，eg：32G的显存，batch_size = 32 * 14；

4. benchmark的算分机制在正确率大于等于65%才给出有效分数， 如果测试长时间达不到有效正确率(65%)，建议停止实验后调整训练参数(eg：batch size， learning rate，)重新测试 。

### 配置运行参数
#### 
 根据需求修改example/trials/network_morphism/imagenet/config.yml配置

|      |         可选参数         |                   说明                   |     默认值      |
| ---- | :----------------------: | :--------------------------------------: | :-------------: |
| 1    |     trialConcurrency     |            同时运行的trial数             |        1        |
| 2    |     maxExecDuration      |          设置测试时间(单位 ：h)          |       24        |
| 3    |   CUDA_VISIBLE_DEVICES   |        指定测试程序可用的gpu索引         | 0,1,2,3,4,5,6,7 |
| 4    | srun：--cpus-per-task=23 |         参数为当前可用的cpu核数减 1          |       23        |
| 5    |         --slave          |     跟 trialConcurrency参数保持一致      |        1        |
| 6    |           --ip           |               master节点ip               |    127.0.0.1    |
| 7    |       --batch_size       |                batch size                |       448       |
| 8    |         --epoch          |                 epoch数                  |       60        |
| 9   |       --initial_lr       |                初始学习率                |      2e-1       |
| 10   |        --final_lr        |                最低学习率                |      1e-5       |
| 11   |     --train_data_dir     |              训练数据集路径              |      None       |
| 12   |      --val_data_dir      |              验证数据集路径              |      None       |

可参照如下配置：

```
authorName: default
experimentName: example_imagenet-network-morphism-test
trialConcurrency: 1		# 1
maxExecDuration: 24h	# 2
maxTrialNum: 6000
trainingServicePlatform: local
useAnnotation: false
tuner:
 \#choice: TPE, Random, Anneal, Evolution, BatchTuner, NetworkMorphism
 \#SMAC (SMAC should be installed through nnictl)
 builtinTunerName: NetworkMorphism
 classArgs:
  optimize_mode: maximize
  task: cv
  input_width: 224
  input_channel: 3
  n_output_node: 1000
  
trial:
 command: CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7  \                                  # 3
       srun -N 1 -n 1 --ntasks-per-node=1 \
       --cpus-per-task=23 \	  # 4
       python3 imagenet_tfkeras_slurm_hpo.py \
       --slave 1 \								  # 5
       --ip 127.0.0.1 \							  # 6
       --batch_size 448 \						  # 7
       --epoch 60 \						          # 8
       --initial_lr 2e-1 \						  # 9
       --final_lr 1e-5 \						  # 10
       --train_data_dir /gdata/ILSVRC2012/ImageNet-Tensorflow/train_tfrecord/ \  # 11
       --val_data_dir /gdata/ILSVRC2012/ImageNet-Tensorflow/validation_tfrecord/ # 12

 codeDir: .
 gpuNum: 0
```
### 运行benchmark
在/userhome/AAH/example/trials/network_morphism/imagenet/目录下执行以下命令运行用例

```
nnictl create -c config.yml
```
#### 查看运行过程

执行以下命令查看正在运行的experiment的trial运行信息

```
nnictl top
```
当测试运行时间>=1h 后，运行以下程序会在终端打印experiment的Error、PFLOPS、Score等信息
```
python3 AAH/scripts/reports/reprot.py --id  experiment_ID  
```

#### 停止实验
停止expriments, 执行
```
nnictl stop
```
通过命令squeue查看slurm中是否还有未被停止的job，如果存在job且ST列为CG，请等待作业结束，实验才算完全停止。

##### 查看实验报告
当测试运行时间>=1h 后，运行以下程序会在终端打印experiment的Error、PFLOPS、Score等信息
```
python3 AAH/scripts/reports/reprot.py --id  experiment_ID  
```
同时会产生实验报告存放在experiment_ID的对应路径/root/mountdir/nni/experiments/experiment_ID/results目录下

实验成功时报告为 Report_Succeed.html

实验失败时报告为 Report_Failed.html

实验失败会报告失败原因，请查阅AI Benchmark测试规范分析失败原因

##### 保存日志&结果数据
运行以下程序可将测试产生的日志以及数据统一保存到/root/mountdir/nni/experiments/experiment_ID/results/logs中，便于实验分析
```
python3 AAH/scripts/reports/reprot.py --id  experiment_ID  --logs True
```
由于实验数据在复制过程中会导致额外的网络、内存、cpu等资源开销，建议在实验停止/结束后再执行日志保存操作。

# 报告反馈
当您将结果数据和日志保存下来后需要将 /root/mountdir/nni/experiments/experiment_ID目录打包、试验的训练的代码发送到我们的邮箱renzhx@pcl.ac.cn、yongheng.liu@pcl.ac.cn；



## 3 测试参数设置及安装要求
### 可变设置

1. slave计算节点-GPU卡数调整：用户可自定义规定每个trial运行的硬件要求，根据自身平台特性，可以通过数据并行方式将整个计算节点集群作为一个trial的计算节点，也可以将slave计算节点上单个GPU作为一个trial的计算节点。
2. 深度学习框架：建议使用keras+tensorflow，用户也可以根据测试平台特性，使用最适合的深度学习框架。
3. 数据集加载方式：建议将数据预处理乘TF格式，以加快数据加载的效率。用户也可以根据测试平台特性，调整数据加载策略。
4. 数据集存储方式：目前默认存储在网络共享存储器上，用户可以根据测试平台特性，调整存储路径。
5. 
6. 每个trial任务中的网络结构搜索次数：默认搜索次数为1次，用户可根据trial执行的耗时自定义网络结构搜索时间。
7. 超参搜索空间：目前搜索空间只有convkernel size、dropout rate，用户可根据自身情况，增加超参搜索空间，调加如optimizer等超参数。
8. 每个trial任务中网络结构的搜索次数：默认搜索次数30次，用户可根据测试平台特性，调整超参搜索次数。

### 安装要求
Benchmark 在Ubuntu16.04，CUDA10.1，python 64-bit >= 3.5，tensorflow2.2上进行了测试。

## 许可

基于 MIT license

