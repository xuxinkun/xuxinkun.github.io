---
layout:     post
title:      "tensorflow与kubernetes/docker结合使用实践"
subtitle:   "A practice used in combination with tensorflow and kubernetes."
date:       2016-10-08 8:00:00
author:     "XuXinkun"
header-img: "img/post-bg-tensorflow.jpg"
tags:
    - kubernetes
---

# tensorflow

[tensorflow](https://www.tensorflow.org/)是谷歌基于DistBelief进行研发的第二代人工智能学习系统，其命名来源于本身的运行原理。Tensor（张量）意味着N维数组，Flow（流）意味着基于数据流图的计算，TensorFlow为张量从图象的一端流动到另一端计算过程。TensorFlow是将复杂的数据结构传输至人工智能神经网中进行分析和处理过程的系统。

tensorflow可在小到一部智能手机、大到数千台数据中心服务器的各种设备上运行。本文主要探讨的是tensorflow在大规模容器上运行的一种方案。

tensorflow作为深度学习的框架，其对于数据的处理可以分为训练、验证、测试、服务几种。一般来说，训练是用指来训练模型，验证主要用以检验所训练出来的模型的正确性和是否过拟合。测试是计算黑盒数据对于训练的模型进行测试，从而评判模型的准确率。服务是指利用已经完成的训练模型提供服务。这里为了简化，将处理分为了训练和服务两种。

训练主要是指从给定训练的程序和训练数据集，用以生成训练的模型。训练完成的模型可以通过存储形成为checkpoints文件。

验证、测试、服务统统归一到服务，其主要流程是使用已有的模型，对于数据集进行处理。

# tensorflow训练 in kubernetes

对于tensorflow训练的支持，kubernetes可以通过创立多个pod来进行支持。tensorflow分布式可以通过制定parameters服务器(ps参数服务器)和worker服务器进行。

> 首先ps是整个训练集群的参数服务器，保存模型的Variable，worker是计算模型梯度的节点，得到的梯度向量会交付给ps更新模型。in-graph与between-graph对应，但两者都可以实现同步训练和异步训练，in-graph指整个集群由一个client来构建graph，并且由这个client来提交graph到集群中，其他worker只负责处理梯度计算的任务，而between-graph指的是一个集群中多个worker可以创建多个graph，但由于worker运行的代码相同因此构建的graph也相同，并且参数都保存到相同的ps中保证训练同一个模型，这样多个worker都可以构建graph和读取训练数据，适合大数据场景。同步训练和异步训练差异在于，同步训练每次更新梯度需要阻塞等待所有worker的结果，而异步训练不会有阻塞，训练的效率更高，在大数据和分布式的场景下一般使用异步训练。----[TensorFlow深度学习](http://blog.jobbole.com/105602/)

我使用rc创建多个ps和worker服务器。

> gcr.io/tensorflow/tensorflow:latest镜像是[tensorflow提供的官网镜像](https://www.tensorflow.org/versions/r0.11/get_started/os_setup.html#docker-installation)，使用CPU进行计算。使用GPU计算的版本下文再行介绍。

```
[root@A01-R06-I184-22 yaml]# cat ps.yaml 
apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-ps-rc
spec:
  replicas: 2
  selector:
    name: tensorflow-ps
  template:
    metadata:
      labels:
        name: tensorflow-ps
        role: ps
    spec:
      containers:
        - name: ps
          image: gcr.io/tensorflow/tensorflow:latest
          ports:
           - containerPort: 2222

[root@A01-R06-I184-22 yaml]# cat worker.yaml 
apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-worker-rc
spec:
  replicas: 2
  selector:
    name: tensorflow-worker
  template:
    metadata:
      labels:
        name: tensorflow-worker
        role: worker
    spec:
      containers:
        - name: worker
          image: gcr.io/tensorflow/tensorflow:latest
          ports:
           - containerPort: 2222
```

之后为ps和worker分别创建服务。

```
[root@A01-R06-I184-22 yaml]# cat ps-srv.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-ps
    role: service
  name: tensorflow-ps-service
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-ps
    
[root@A01-R06-I184-22 yaml]# cat worker-srv.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-worker
    role: service
  name: tensorflow-wk-service
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-worker
```

我们可以通过查看service来查看对应的容器的ip。

```
[root@A01-R06-I184-22 yaml]# kubectl describe service tensorflow-ps-service 
Name:			tensorflow-ps-service
Namespace:		default
Labels:			name=tensorflow-ps,role=service
Selector:		name=tensorflow-ps
Type:			ClusterIP
IP:			10.254.170.61
Port:			<unset>	2222/TCP
Endpoints:		4.0.84.3:2222,4.0.84.4:2222
Session Affinity:	None
No events.

[root@A01-R06-I184-22 yaml]# kubectl describe service tensorflow-wk-service 
Name:			tensorflow-wk-service
Namespace:		default
Labels:			name=tensorflow-worker,role=service
Selector:		name=tensorflow-worker
Type:			ClusterIP
IP:			10.254.70.9
Port:			<unset>	2222/TCP
Endpoints:		4.0.84.5:2222,4.0.84.6:2222
Session Affinity:	None
No events.
```

这里我使用[deep_recommend_system](https://github.com/tobegit3hub/deep_recommend_system)来进行分布式的实验。

在pod中先下载对应的deep_recommend_system的代码。

```
curl https://codeload.github.com/tobegit3hub/deep_recommend_system/zip/master -o drs.zip
unzip drs.zip
cd deep_recommend_system-master/distributed/
```

在ps的其中一个容器(4.0.84.3)中执行启动ps服务器的任务：

```
root@tensorflow-ps-rc-b5d6g:/notebooks/deep_recommend_system-master/distributed# nohup python cancer_classifier.py --ps_hosts=4.0.84.3:2222,4.0.84.3:2223 --worker_hosts=4.0.84.5:2222,4.0.84.6:2222 --job_name=ps --task_index=0 >log1 &
[1] 502
root@tensorflow-ps-rc-b5d6g:/notebooks/deep_recommend_system-master/distributed# nohup: ignoring input and redirecting stderr to stdout
 
root@tensorflow-ps-rc-b5d6g:/notebooks/deep_recommend_system-master/distributed# nohup python cancer_classifier.py --ps_hosts=4.0.84.3:2222,4.0.84.4:2223 --worker_hosts=4.0.84.5:2222,4.0.84.6:2222 --job_name=ps --task_index=1 >log2 &
[2] 603
root@tensorflow-ps-rc-b5d6g:/notebooks/deep_recommend_system-master/distributed# nohup: ignoring input and redirecting stderr to stdout
```

> 这里我尝试使用两个pod分别做ps服务器，但是总是报core dump的错误。官网也有[类似错误]((https://github.com/tensorflow/tensorflow/issues/3766))，未能解决，推测原因可能是复用了某个设备的缘故(两个pod都在同一个宿主机上)。使用一个pod作为两个ps服务器即无问题。

在worker两个容器中分别执行：

```
root@tensorflow-worker-rc-vznvt:/notebooks/deep_recommend_system-master/distributed# nohup python cancer_classifier.py --ps_hosts=4.0.84.3:2222,4.0.84.3:2223 --worker_hosts=4.0.84.5:2222,4.0.84.6:2222 --job_name=worker --task_index=0 >log &

***********************

root@tensorflow-worker-rc-cpnt7:/notebooks/deep_recommend_system-master/distributed# nohup python cancer_classifier.py --ps_hosts=4.0.84.3:2222,4.0.84.3:2223 --worker_hosts=4.0.84.5:2222,4.0.84.6:2222 --job_name=worker --task_index=1 >log &
```

之后在worker服务器上的checkpoint文件夹中可以查看计算模型的中间保存结果。

```
root@tensorflow-worker-rc-vznvt:/notebooks/deep_recommend_system-master/distributed# ll checkpoint/
total 840
drwxr-xr-x 2 root root   4096 Oct 10 15:45 ./
drwxr-xr-x 3 root root     76 Oct 10 15:18 ../
-rw-r--r-- 1 root root      0 Sep 23 14:27 .gitkeeper
-rw-r--r-- 1 root root    270 Oct 10 15:45 checkpoint
-rw-r--r-- 1 root root  86469 Oct 10 15:45 events.out.tfevents.1476113854.tensorflow-worker-rc-vznvt
-rw-r--r-- 1 root root 248875 Oct 10 15:37 graph.pbtxt
-rw-r--r-- 1 root root   2229 Oct 10 15:42 model.ckpt-1172
-rw-r--r-- 1 root root  94464 Oct 10 15:42 model.ckpt-1172.meta
-rw-r--r-- 1 root root   2229 Oct 10 15:43 model.ckpt-1422
-rw-r--r-- 1 root root  94464 Oct 10 15:43 model.ckpt-1422.meta
-rw-r--r-- 1 root root   2229 Oct 10 15:44 model.ckpt-1670
-rw-r--r-- 1 root root  94464 Oct 10 15:44 model.ckpt-1670.meta
-rw-r--r-- 1 root root   2229 Oct 10 15:45 model.ckpt-1921
-rw-r--r-- 1 root root  94464 Oct 10 15:45 model.ckpt-1921.meta
-rw-r--r-- 1 root root   2229 Oct 10 15:41 model.ckpt-921
-rw-r--r-- 1 root root  94464 Oct 10 15:41 model.ckpt-921.meta
```

## tensorflow gpu支持

### tensorflow gpu in docker

docker可以通过提供gpu设备到容器中。nvidia官方提供了[nvidia-docker](https://github.com/NVIDIA/nvidia-docker)的一种方式，其用nvidia-docker的命令行代替了docker的命令行来使用GPU。

```
nvidia-docker run -it -p 8888:8888 gcr.io/tensorflow/tensorflow:latest-gpu
```

这种方式对于docker侵入较多，因此nvidia还提供了一种[nvidia-docker-plugin](https://github.com/NVIDIA/nvidia-docker/blob/master/tools/src/nvidia-docker-plugin/main.go)的方式。其使用流程如下：

首先在宿主机启动nvidia-docker-plugin：

```
[root@A01-R06-I184-22 nvidia-docker]# ./nvidia-docker-plugin 
./nvidia-docker-plugin | 2016/10/10 00:01:12 Loading NVIDIA unified memory
./nvidia-docker-plugin | 2016/10/10 00:01:12 Loading NVIDIA management library
./nvidia-docker-plugin | 2016/10/10 00:01:17 Discovering GPU devices
./nvidia-docker-plugin | 2016/10/10 00:01:18 Provisioning volumes at /var/lib/nvidia-docker/volumes
./nvidia-docker-plugin | 2016/10/10 00:01:18 Serving plugin API at /run/docker/plugins
./nvidia-docker-plugin | 2016/10/10 00:01:18 Serving remote API at localhost:3476
```

可以看到nvidia-docker-plugin监听了3486端口。然后在宿主机上运行docker run -ti `curl -s http://localhost:3476/v1.0/docker/cli` -p 8890:8888 gcr.io/tensorflow/tensorflow:latest-gpu /bin/bash命令以创建tensorflow的GPU容器。并可以在容器中验证是否能正常import tensorflow。

```
[root@A01-R06-I184-22 ~]# docker run -ti `curl -s http://localhost:3476/v1.0/docker/cli` -p 8890:8888 gcr.io/tensorflow/tensorflow:latest-gpu /bin/bash
root@7087e1f99062:/notebooks# python
Python 2.7.6 (default, Jun 22 2015, 17:58:13) 
[GCC 4.8.2] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcublas.so locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcudnn.so locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcufft.so locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcuda.so.1 locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcurand.so locally
>>> 
```

可以看到tensorflow已经能够正确加载了。

这里同样使用deep_recommend_system进行测试。在pod中先下载对应的deep_recommend_system的代码。

```
curl https://codeload.github.com/tobegit3hub/deep_recommend_system/zip/master -o drs.zip
unzip drs.zip
cd deep_recommend_system-master/
```

然后使用GPU0和1进行计算。

```
root@7087e1f99062:/notebooks/deep_recommend_system-master# export CUDA_VISIBLE_DEVICES='0,1'    //用以指定使用的GPU的编号
root@7087e1f99062:/notebooks/deep_recommend_system-master# python cancer_classifier.py 
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcublas.so locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcudnn.so locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcufft.so locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcuda.so.1 locally
I tensorflow/stream_executor/dso_loader.cc:111] successfully opened CUDA library libcurand.so locally
Use the model: wide_and_deep
Use the optimizer: adagrad
Use the model: wide_and_deep
Use the model: wide_and_deep
I tensorflow/core/common_runtime/gpu/gpu_device.cc:951] Found device 0 with properties: 
name: Tesla K20c
major: 3 minor: 5 memoryClockRate (GHz) 0.7055
pciBusID 0000:02:00.0
Total memory: 4.69GiB
Free memory: 4.61GiB
W tensorflow/stream_executor/cuda/cuda_driver.cc:572] creating context when one is currently active; existing: 0x24402e0
I tensorflow/core/common_runtime/gpu/gpu_device.cc:951] Found device 1 with properties: 
name: Tesla K20c
major: 3 minor: 5 memoryClockRate (GHz) 0.7055
pciBusID 0000:04:00.0
Total memory: 4.69GiB
Free memory: 4.61GiB
I tensorflow/core/common_runtime/gpu/gpu_device.cc:972] DMA: 0 1 
I tensorflow/core/common_runtime/gpu/gpu_device.cc:982] 0:   Y Y 
I tensorflow/core/common_runtime/gpu/gpu_device.cc:982] 1:   Y Y 
I tensorflow/core/common_runtime/gpu/gpu_device.cc:1041] Creating TensorFlow device (/gpu:0) -> (device: 0, name: Tesla K20c, pci bus id: 0000:02:00.0)
I tensorflow/core/common_runtime/gpu/gpu_device.cc:1041] Creating TensorFlow device (/gpu:1) -> (device: 1, name: Tesla K20c, pci bus id: 0000:04:00.0)
[0:00:34.437041] Step: 100, loss: 2.97578692436, accuracy: 0.77734375, auc: 0.763736724854
[0:00:32.162310] Step: 200, loss: 1.81753754616, accuracy: 0.7890625, auc: 0.788772583008
[0:00:37.559177] Step: 300, loss: 1.26066374779, accuracy: 0.865234375, auc: 0.811861813068
[0:00:36.082163] Step: 400, loss: 0.920016527176, accuracy: 0.8359375, auc: 0.820605039597
```

同样我可以使用nvidia-smi查看GPU使用情况

```
[root@A01-R06-I184-22 ~]# nvidia-smi 
Tue Oct 11 00:10:28 2016       
+------------------------------------------------------+                       
| NVIDIA-SMI 352.39     Driver Version: 352.39         |                       
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla K20c          Off  | 0000:02:00.0     Off |                    0 |
| 30%   26C    P0    48W / 225W |   4540MiB /  4799MiB |      2%      Default |
+-------------------------------+----------------------+----------------------+
|   1  Tesla K20c          Off  | 0000:04:00.0     Off |                    0 |
| 30%   31C    P0    48W / 225W |   4499MiB /  4799MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   2  Tesla K20c          Off  | 0000:83:00.0     Off |                    0 |
| 30%   25C    P8    26W / 225W |     11MiB /  4799MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
|   3  Tesla K20c          Off  | 0000:84:00.0     Off |                    0 |
| 30%   24C    P8    25W / 225W |     11MiB /  4799MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID  Type  Process name                               Usage      |
|=============================================================================|
|    0    132460    C   python                                        4524MiB |
|    1    132460    C   python                                        4484MiB |
+-----------------------------------------------------------------------------+
```

nvidia-docker-plugin工作原理是是其提供了一个API

```
[root@A01-R06-I184-22 ~]# curl -s http://localhost:3476/v1.0/docker/cli
--volume-driver=nvidia-docker --volume=nvidia_driver_352.39:/usr/local/nvidia:ro --device=/dev/nvidiactl --device=/dev/nvidia-uvm --device=/dev/nvidia0 --device=/dev/nvidia1 --device=/dev/nvidia2 --device=/dev/nvidia3
```

可以看到`curl -s http://localhost:3476/v1.0/docker/cli`命令实际是提供了docker run时候的一些必要参数。其中包括把gpu设备映射进入容器中的部分(`--device=/dev/nvidiactl --device=/dev/nvidia-uvm --device=/dev/nvidia0 --device=/dev/nvidia1 --device=/dev/nvidia2 --device=/dev/nvidia3`)，还包括了将`nvidia_driver_352.39`存储映射进入容器的部分。

接下来我们对于`nvidia_driver_352.39`进行分析

```
[root@A01-R06-I184-22 ~]# docker volume ls
DRIVER              VOLUME NAME
nvidia-docker       nvidia_driver_352.39
[root@A01-R06-I184-22 ~]# docker volume inspect nvidia_driver_352.39 
[
    {
        "Name": "nvidia_driver_352.39",
        "Driver": "nvidia-docker",
        "Mountpoint": "/var/lib/nvidia-docker/volumes/nvidia_driver/352.39"
    }
]
```

可以看到该存储其实只是一个文件夹。对文件夹`/var/lib/nvidia-docker/volumes/nvidia_driver/352.39/`进行分析

```
[root@A01-R06-I184-22 ~]# tree -L 3 /var/lib/nvidia-docker/volumes/nvidia_driver/352.39/
/var/lib/nvidia-docker/volumes/nvidia_driver/352.39/
├── bin
│   ├── nvidia-cuda-mps-control
│   ├── nvidia-cuda-mps-server
│   ├── nvidia-debugdump
│   ├── nvidia-persistenced
│   └── nvidia-smi
├── lib
│   ├── libcuda.so -> libcuda.so.352.39
│   ├── libcuda.so.1 -> libcuda.so.352.39
│   ├── libcuda.so.352.39
│   ├── libGL.so.1 -> libGL.so.352.39
│   ├── libGL.so.352.39
│   ├── libnvcuvid.so.1 -> libnvcuvid.so.352.39
│   ├── libnvcuvid.so.352.39
│   ├── libnvidia-compiler.so.352.39
│   ├── libnvidia-eglcore.so.352.39
│   ├── libnvidia-encode.so.1 -> libnvidia-encode.so.352.39
│   ├── libnvidia-encode.so.352.39
│   ├── libnvidia-fbc.so.1 -> libnvidia-fbc.so.352.39
│   ├── libnvidia-fbc.so.352.39
│   ├── libnvidia-glcore.so.352.39
│   ├── libnvidia-glsi.so.352.39
│   ├── libnvidia-ifr.so.1 -> libnvidia-ifr.so.352.39
│   ├── libnvidia-ifr.so.352.39
│   ├── libnvidia-ml.so.1 -> libnvidia-ml.so.352.39
│   ├── libnvidia-ml.so.352.39
│   ├── libnvidia-opencl.so.1 -> libnvidia-opencl.so.352.39
│   ├── libnvidia-opencl.so.352.39
│   ├── libvdpau_nvidia.so.1 -> libvdpau_nvidia.so.352.39
│   └── libvdpau_nvidia.so.352.39
└── lib64
    ├── libcuda.so -> libcuda.so.352.39
    ├── libcuda.so.1 -> libcuda.so.352.39
    ├── libcuda.so.352.39
    ├── libGL.so.1 -> libGL.so.352.39
    ├── libGL.so.352.39
    ├── libnvcuvid.so.1 -> libnvcuvid.so.352.39
    ├── libnvcuvid.so.352.39
    ├── libnvidia-compiler.so.352.39
    ├── libnvidia-eglcore.so.352.39
    ├── libnvidia-encode.so.1 -> libnvidia-encode.so.352.39
    ├── libnvidia-encode.so.352.39
    ├── libnvidia-fbc.so.1 -> libnvidia-fbc.so.352.39
    ├── libnvidia-fbc.so.352.39
    ├── libnvidia-glcore.so.352.39
    ├── libnvidia-glsi.so.352.39
    ├── libnvidia-ifr.so.1 -> libnvidia-ifr.so.352.39
    ├── libnvidia-ifr.so.352.39
    ├── libnvidia-ml.so.1 -> libnvidia-ml.so.352.39
    ├── libnvidia-ml.so.352.39
    ├── libnvidia-opencl.so.1 -> libnvidia-opencl.so.352.39
    ├── libnvidia-opencl.so.352.39
    ├── libnvidia-tls.so.352.39
    ├── libvdpau_nvidia.so.1 -> libvdpau_nvidia.so.352.39
    └── libvdpau_nvidia.so.352.39

3 directories, 52 files
```

可以看到这个文件夹其实主要包含的是关于GPU显卡的一些库、包和一些必要的可执行文件。这些文件实际上也是从宿主机上由nvidia-docker-plugin收集拷贝到该文件夹中的，用以提供给容器，方便容器对于GPU的使用。

### kubernetes与GPU

kubernetes1.3已经引入了[GPU调度支持](https://github.com/kubernetes/kubernetes/pull/24836)，但是目前是实验性质。因为我们已经明白了nvidia-docker-plugin的工作原理，后面我们可以将其集成到kubelet中，从而在技术上支持pod使用GPU。

# tensorflow服务

[Serving Inception Model with TensorFlow Serving and Kubernetes](https://tensorflow.github.io/serving/serving_inception)中对于tensorflow服务与kubernetes结合使用的方式进行了介绍。

其基本的工作方式是首先根据已经训练好的模型，制作成可以对外提供服务的镜像`inception_serving`。而后使用该镜像创建rc，并对应建立service。

```
$ kubectl get rc
CONTROLLER             CONTAINER(S)          IMAGE(S)                              SELECTOR               REPLICAS   AGE
inception-controller   inception-container   gcr.io/tensorflow-serving/inception   worker=inception-pod   3          20s

$ kubectl get svc
NAME                CLUSTER_IP      EXTERNAL_IP      PORT(S)    SELECTOR               AGE
inception-service   10.15.242.244   146.148.88.232   9000/TCP   worker=inception-pod   3m

$ kubectl describe svc inception-service
Name:     inception-service
Namespace:    default
Labels:     <none>
Selector:   worker=inception-pod
Type:     LoadBalancer
IP:     10.15.242.244
LoadBalancer Ingress: 146.148.88.232
Port:     <unnamed> 9000/TCP
NodePort:   <unnamed> 32006/TCP
Endpoints:    10.12.2.4:9000,10.12.4.4:9000,10.12.4.5:9000
Session Affinity: None
Events:
  FirstSeen LastSeen  Count From      SubobjectPath Reason      Message
  ───────── ────────  ───── ────      ───────────── ──────      ───────
  4m    3m    2 {service-controller }     CreatingLoadBalancer  Creating load balancer
  3m    2m    2 {service-controller }     CreatedLoadBalancer   Created load balancer
```

用户请求直接通过EXTERNAL_IP(146.148.88.232:9000)进行服务访问。当用户有请求到来时，kubernetes将请求分发给`10.12.2.4:9000,10.12.4.4:9000,10.12.4.5:9000`之一的pod，然后由该pod上提供实际的服务，从而返回结果。

这一过程本质上来说同提供web服务(如tomcat的服务)等是没有多大区别的。kubernetes可以很好的支持。

# 参考资料

- [存储与恢复](https://www.tensorflow.org/versions/r0.11/how_tos/variables/index.html)
- [Serving Inception Model with TensorFlow Serving and Kubernetes](https://tensorflow.github.io/serving/serving_inception)
- [TensorFlow深度学习](http://blog.jobbole.com/105602/)
- [deep_recommend_system](https://github.com/tobegit3hub/deep_recommend_system)
