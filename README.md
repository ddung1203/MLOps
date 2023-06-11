# MLOps

[Practitioners guide to MLOps](mlops-component.md)

MLOps는 Machine Learning Operations을 뜻하며, MLOps는 머신 러닝 모델을 productio으로 전환하는 프로세스를 간소화하고, 뒤이어 이를 유지관리하고 모니터링하는 데 주안점을 둔 기능이다. 이를 위해 ML 모델의 적절한 모니터링, 검증과 거버넌스를 포함해 지속적인 통합과 배포를 구현해야 한다.

## Setup Kubernetes

### 1. Install Docker

[Install Docker](https://github.com/ddung1203/TIL/blob/main/Docker/00_Docker_intro.md#docker-engine-%EC%84%A4%EC%B9%98)

### 2. Turn off Swap Memory

Swap 메모리의 경우 물리직으로 사용할 수 있는 메모리가 부족할 경우 일부를 디스크에 저장해놓고 메모리 블럭이 자리가 나면 다시 이동시키는 방법으로 데이터의 저장 위치를 교환한다. 만약 16Gi 램으로 17Gi 메모리가 필요한 연산을 진행할 수 있다.

그러나 Kubernetes에서 Swap Memory를 사용하게 되면 컨테이너의 속도가 변동적이게 된다. 왜냐하면 Swap Memory를 어떤 컨테이너가 쓰면서 느려지는지 예측할 수 없기 때문이다.

``` bash
sudo swapoff -a
```

### 3. Install Kubernetes - Kubeadm

``` bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> [cgroup driver 오류](https://github.com/ddung1203/TIL/blob/main/k8s/00_Kubeadm_k8s_install.md#cgroup-driver-%EC%98%A4%EB%A5%98)
> 
> 컨테이너 런타임과 kubelet의 cgroup 드라이버를 일치시켜야 하며, 그렇지 않은 경우 kubelet 프로세스에 오류가 발생한다. 상기의 링크를 참고하여 수정을 한다.

> 
> 
> 

### 4. Setup GPU

1. Install NVIDIA Driver

``` bash
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update && sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

2. Install NVIDIA-Docker

``` bash
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update
sudo apt-get install -y nvidia-docker2 &&
sudo systemctl restart docker
```

검증

``` bash
ubuntu@desktop > ~/git/MLOps > sudo docker run --rm --gpus all nvidia/cuda:12.1.1-base-ubuntu20.04 nvidia-smi
[sudo] password for ubuntu: 
Sun Jun 11 15:42:10 2023       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 530.41.03              Driver Version: 530.41.03    CUDA Version: 12.1     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                  Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf            Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce GTX 960          Off| 00000000:01:00.0  On |                  N/A |
|  5%   44C    P0               29W / 130W|    688MiB /  2048MiB |      3%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
+---------------------------------------------------------------------------------------+
```

3. NVIDIA-Docker - Default Container Runtime 설정

`/etc/docker/daemon.json`

``` json
{
  "default-runtime": "nvidia",
  "runtimes": {
      "nvidia": {
          "path": "nvidia-container-runtime",
          "runtimeArgs": []
  }
  }
}
```

``` bash
sudo systemctl daemon-reload
sudo service docker restart
```

검증

``` bash
ubuntu@desktop > ~/git/MLOps > sudo docker info | grep nvidia
 Runtimes: io.containerd.runc.v2 nvidia runc
 Default Runtime: nvidia
```

4. NVIDIA Device Plugin

Kubernets의 버전에 맞게 plugin을 설치한다.

[NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)

``` bash
wget https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml
```

현재 환경은 Control Plane에서 모든 것을 실행중이다. 따라서 하기의 설정을 추가한다.

``` yaml
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

배포

``` bash
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml
```

또한, Control Plane 노드에 Pod를 배포하기 위해선 taint를 제거해야 한다.

``` bash
kubectl taint nodes desktop node-role.kubernetes.io/control-plane-
```

검증

``` bash
ubuntu@desktop > ~/git/MLOps > kubectl get nodes "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
NAME      GPU
desktop   1
```

``` bash
ubuntu@desktop > ~/git/MLOps > cat <<EOF | kubectl apply -f -                           
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
    - name: cuda-container
      image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda10.2
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF
pod/gpu-pod created

ubuntu@desktop > ~/git/MLOps > kubectl logs gpu-pod                                                                              
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```