---
layout: single
title: "[티어리스트] (2) Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기"
categories: cicd
tag: [cicd, argocd, github-action, sonarqube,  kubernetes]
toc: true
toc_sticky: true
author_profile: true
sidebar:
  nav: "docs"
header:
  teaser: "https://i.imgur.com/rD4aT7s.png"
# search: false
---
# 쿠버네티스 클러스터 구성하기

VM에 Ubuntu 서버를 띄웠다. 이제 쿠버네티스를 설치하고 클러스터를 구성해보자.

## 컨테이너 런타임 설치

쿠버네티스는 컨테이너 오케스트레이션 플랫폼이다. 컨테이너를 관리하는 컨테이너 런타임을 먼저 설치해야한다.
쿠버네티스는 기존에 도커를 지원했지만 도커가 CRI 인터페이스 이전에 나온 기술이라 CRI 인터페이스를 준수하지 못해 도커 컨테이너 런타임 지원을 중단하게 되었다. 따라서 도커에서 분리된 컨테이너 런타임인 `containerd`를 설치해보자.

[ubuntu docker 설치 링크](https://docs.docker.com/engine/install/ubuntu/)를 참고하여 설치할 수 있다.

```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# install containerd.io
sudo apt install containerd.io
```

다음 명령어로 `containerd.io` 서비스가 실행중인지 확인할 수 있다.

```sh
sudo systemctl status containerd
```

![](https://i.imgur.com/5QbPNZM.png)

### kubeadm 설치

[kubeadm 설치 공식 문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)를 참고하여 설치했다.

```sh
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

위의 명령어로 `kubeadm`을 설치할 수 있다.
### swapoff

쿠버네티스를 이용하려면 메모리 스왑 기능을 꺼야한다고 한다. 최근 업데이트에서는 이를 지원한다고 하긴 하는데 혹시 모르니 꺼주자.

다음 명령어로 메모리 스왑을 비활성화 할 수 있다.

```sh
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab # swapoff 영구 설정
```

위와 같이 설정해도 영구적으로 설정되지 않을 수 있으니 추가적으로 다음과 같이 설정하자.

```sh
# 이름 기억하기
sudo systemctl list-unit-files --type swap

# 비활성화
sudo systemctl mask [이름]

# 확인
sudo systemctl list-unit-files --type swap
```

## Kubernetes 클러스터 초기화 및 설정

```sh
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 
# 쿠버네티스용 네트워크 플러그인 ip대역 (ex: 플라넬 cni 권장 ip 대역)
```

위의 명령어로 쿠버네티스 클러스터를 초기화 할 수 있다.

### 오류 해결 1 (container runtime is not running)

`kubeadm`을 통해 클러스터를 초기화 할 때 `container runtime is not running`이라는 오류가 날 수 있다.

다음과 같이 설정하자.

```sh
sudo vi /etc/containerd/config.toml
```

![](https://i.imgur.com/7LPSW42.png)

`disabled_plugins = ["cri"]` 라인 삭제

```sh
sudo systemctl restart containerd
```

### 오류 해결 2 (FileContent--proc-sys-net-bridge-bridge-nf-call-iptables)

위와 같은 오류가 생겼을 때 다음과 같이 입력한다.

```sh
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

sudo reboot
```

### 오류 해결 3 (FileContent--proc-sys-net-ipv4-ip_forward)

위와 같은 오류가 생겼을 때 다음과 같이 입력한다.

```sh
sudo vi /etc/sysctl.conf

# net.ipv4.ip_forward=1 주석 해제

sudo sysctl -p # 설정 적용
```

![](https://i.imgur.com/mEvpqFK.png)

### kubectl 설정 파일 생성 및 적용

```sh
mkdir ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chmod 644 ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
echo 'export KUBECONFIG="$HOME/.kube/config"' >> ~/.bashrc

chmod o-r ~/.kube/config
chmod g-r ~/.kube/config
```

### flannel CNI 설치

CNI는 클러스터 내의 애플리케이션간의 통신을 하게 해준다. 쿠버네티스의 `kubenet`이라는 CNI가 존재하지만 기능이 제한적이다. 따라서 서드파티 플랫폼을 많이 사용한다.

그중 많이 사용하는 Flannel을 설치해보자. 다음 명령어로 쉽게 설치할 수 있다.

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 싱글노드 클러스터로 구성하기

원칙적으로 마스터플레인에는 파드나 서비스를 배치할 수 없다. 하지만 현재 리소스의 문제로 서버 하나만 운영하고 있다. 따라서 컨트롤플레인에도 서비스를 배치할 수 있게 하여 싱글 노드 클러스터로 사용할 예정이다.

다음과 같은 명령어로 컨트롤 플레인에 서비스 배치할 수 있도록 설정할 수 있다. 기본적으로 컨트롤 플레인에는 서비스를 배치할 수 없도록 `taint`가 걸려있는데 이를 해지해주는 명령어이다.

```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

## 다음으로

쿠버네티스 설치와 클러스터 구성을 마쳤다. 이제 GitOps를 구현해주는 ArgoCD를 배포해보자.