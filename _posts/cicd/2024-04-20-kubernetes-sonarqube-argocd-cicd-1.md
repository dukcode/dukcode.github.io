---
layout: single
title: "[티어리스트] (1) Kubernets, ArgoCD, GitHub Action, Sonarqube를 통한 CICD 구축기"
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
# UTM에 Ubuntu 20.04 LTS 띄우고 설정하기

![](https://i.imgur.com/rD4aT7s.png)

M2 Mac에 UTM을 통해 Ubuntu 20.04 LTS를 띄우고 추가적인 설정까지 마쳐보자.

## UTM 설치

아래의 명령어로 UTM을 설치할 수 있다.

```sh
brew install --cask utm
```

## ubuntu 20.04 LTS 버전 설치

### ISO 파일 다운로드

[LINK](https://cdimage.ubuntu.com/releases/focal/release/)에서 설치할 수 있다. M2는 ARM 아키텍처를 사용하므로 ARM버전의 `ISO`파일을 다운받자.

![](https://i.imgur.com/RL7NyaK.png)

### UTM으로 ubuntu 설치

![](https://i.imgur.com/wXfjSrs.png)

ARM CPU위에서 ARM 버전의 Ubuntu를 설치할 것이므로 에뮬레이션대신 Virtualize를 선택한다.

![](https://i.imgur.com/IG4veGc.png)

리눅스를 선택한다.

![](https://i.imgur.com/fVAs6u6.png)

메모리, CPU, Storage 용량을 적절하게 설정하고 Boot ISO Image에서 다운받은 이미지를 불러온다.

공유폴더는 설정하지 않는다.

![](https://i.imgur.com/h9H9d1e.png)

적절한 이름을 생성한다. 저장 후 이미지를 재생하면 Ubuntu 설치 절차가 진행된다.

![](https://i.imgur.com/Iiwup4w.png)

Install Ubuntu Server를 선택한다.

![](https://i.imgur.com/JcvV6nu.png)

English를 선택한다.

![](https://i.imgur.com/n0OlalC.png)

Update to the new Installer를 선택하면 업데이트가 진행된다. 약 1분정도 시간 소요되었다.

![](https://i.imgur.com/m1Rlzjy.png)

별다른 설정 없이 다음으로 넘어간다.

![](https://i.imgur.com/7ZeaRAJ.png)

별다른 설정 없이 다음으로 넘어간다.


![](https://i.imgur.com/Na8c8eU.png)

별다른 설정 없이 다음으로 넘어간다.


![](https://i.imgur.com/snRwCQZ.png)

별다른 설정 없이 다음으로 넘어간다.


![](https://i.imgur.com/CeSL6wb.png)

별다른 설정 없이 다음으로 넘어간다.


![](https://i.imgur.com/0yeg3bX.png)

별다른 설정 없이 다음으로 넘어간다.


![](https://i.imgur.com/K4DPI9g.png)

별다른 설정 없이 다음으로 넘어간다.

![](https://i.imgur.com/NB7glXi.png)

Continue를 선택한다.

![](https://i.imgur.com/VBdErze.png)

name과 password를 설정해준다.

![](https://i.imgur.com/Rm8Sl5p.png)

별다른 설정 없이 다음으로 넘어간다.

![](https://i.imgur.com/oCPDD7V.png)

openssh-server는 수동으로 설치할 것이므로 넘어가준다.

![](https://i.imgur.com/wrfUTlF.png)

아무것도 선택하지 않고 넘어간다.

![](https://i.imgur.com/pVl6mfV.png)

설치 진행은 10분 정도 소요된다.

![](https://i.imgur.com/1EVoaaX.png)

설치가 마무리되면 왼쪽 위 전원버튼을 눌러 전원를 OFF한다.

![](https://i.imgur.com/j7o6Lw3.png)

CD/DVD 초기화 후 다시 실행해준다. 설치가 마무리 되었으므로 이미지는 더이상 필요하지 않다.

![](https://i.imgur.com/l0S7uyl.png)

설치 시 만들었던 이름과 password를 입력하면 우분투 서버를 시작할 수 있다.

### 추가 설정
#### GPU 드라이버 설정

GPU 드라이버 설정을 하지 않으면 화면 관련 오류가 뜰 수 있다. 먼저 VM을 종료한다.

![](https://i.imgur.com/9loA8E9.png)

다음과 같은 순서로 진행한다.

- VM 우클릭
- edit
- 디스플레이
- Emulated Display Card 
- virtio-ramfb-gl (GPU Supported) 선택 후 저장

위와 같이 설정하면 화면 관련 오류가 뜨지 않는다.
#### 네트워크 설정 (고정 IP)

해당 VM을 서버용으로 쓰려면 고정 IP설정은 필수이다.

가장먼저 VM 종료해준다. 우클릭 - edit을 통해 설정창으로 진입한다.

![](https://i.imgur.com/kh29bkH.png)

왼쪽의 추가버튼으로 네트워크를 추가한다. 

![](https://i.imgur.com/ldYs7IF.png)

두번째 네트워크를 Bridged (Advanced)로 변경하고 저장한다. 해당 MAC Address를 기억해두자. 이제 VM을 부팅한다.

```sh
sudo apt install net-tools
```

위의 명령어로 네트워크 관련 툴들을 설치한다.

```sh
ifconfig
```

위의 명령어를 통해 네트워크 인터페이스들을 확인할 수 있다.

![](https://i.imgur.com/Chs4NuV.png)

아까 기억해두었던 MAC Address와 일치하는 네트워크 인터페이스의 이름을 기억해두자.

```
sudo vi /etc/netplan/00-installer-config.yaml
```

위의 명령어로 vi에디터로 진입해 다음과 같이 수정한다.

```yml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s2:
      dhcp4: no
      addresses: [192.168.0.100/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8,8.8.4.4]
```

`gateway4`의 주소는 공유기의 주소로 설정한다. 또한 `addresses`는 공유기의 DHCP서버 설정을 확인하고 대여되지 않은 IP를 설정해주자.

아래의 명령어로 설정을 적용할 수 있다.

```sh
sudo netplan apply
sudo reoot
```

공유기 설정에서 `192.168.0.100`으로 IP가 잡힌걸 확인할 수 있다.

![](https://i.imgur.com/Bmb280W.png)

 추후에 IP에 충돌이 날 수도 있으니 공유기 설정에서 해당 주소와 MAC Address를 수동으로 등록해주자.

![](https://i.imgur.com/eRItksa.png)

마지막으로 VM에서 `curl www.naver.com`으로 외부 인터넷 접속이 잘되는것을 확인한다. 이제 VM을 재부팅하여도 고정된 IP를 가질 수 있게 되었다.

#### SSH 설정

`openssh-server`를 설치하자

```sh
sudo apt update
sudo apt upgrade -y
sudo apt install openssh-server -y
```

`sudo systemctl status ssh`명령어로 백그라운드에서 `ssh`가 잘 실행중인지 확인한다.

![](https://i.imgur.com/RKSbipQ.png)

`ssh`에 대해 방화벽 제외 설정을 다음과 같이 진행한다.

```sh
sudo ufw allow ssh
```

이제 로컬 컴퓨터에서 `ssh dukcode@192.168.0.100` 명령어로 접속할 수 있다.

![](https://i.imgur.com/yKIvd0d.png)

##### 공개키/비밀키를 통해 SSH 접속하기

매번 SSH 접속 시마다 비밀번호를 입력하는 것은 번거롭다. 다음과 같은 과정을 통해 공개키/비밀키를 발급해 접속할 수 있다.

접속하고자 하는 컴퓨터에서 다음과 같이 입력한다.

```sh
ssh-keygen -t rsa
```

생성 과정중 passphrase를 입력하는 부분이 나오는데 2차 비밀번호과 같은 성격이니 설정해준다.

`~/.ssh`로 이동해보면 `id_rsa`와 `id_rsa.pub` 파일이 생성되었을 것이다. 다음과 같이 퍼미션 설정을 하자. `authrozied_keys`는 없을 수도 있다.

```sh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 644 ~/.ssh/authorized_keys
chmod 644 ~/.ssh/known_hosts
```

이제 `id_rsa.pub` 파일을 리모트 서버의 `/.ssh/authorized_keys`파일에 추가해주어야 한다. 다음 명령어로 키를 전송해주자.

```sh
scp ~/.ssh/id_rsa.pub dukcode@192.168.0.100:id_rsa.pub
```

이제 서버에 접속해서 다음 명령어로 공개키를 추가해준다.

```sh
cat $HOME/id_rsa.pub >> $HOME/.ssh/authorized_keys
```

다시 local로 돌아와 `~/.ssh/config`를 만들어 다음과 같이 입력한다.

```
Host k8s
  HostName 192.168.0.100
  User dukcode
  IdentityFile ~/.ssh/id-rsa
```

이제 다음 명령어로 간단하게 접속할 수 있다.

```sh
ssh k8s
```

passphrase를 매번 물어보게되는데 다음 명령어를 통해 재부팅 전까지는 passphrase를 묻지 않도록 할 수 있다.

```sh
ssh-add ~/.ssh/id-rsa
```

만약 위의 명령어가 실행되지 않는다면 다음 명령어를 입력 후 시도해보자.

```sh
eval “$(ssh-agent -s)”
```

#### sleep 설정

원격에서 SSH를 통해 접속 시 끊김 및 재접속 불가 현상이 발생할 수 있다. 원인은 일정 시간 작업을 수행하지 않을 때 ubuntu에서 Suspend/Sleep Mode로 전환되기 때문이다.

다음 명령어로 sleep 관련 서비스들을 꺼주도록 하자.

```sh
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

아래의 명령어를 통해 모두 `dead`상태라면 sleep관련 서비스가 꺼진 것이다.

```sh
sudo systemctl status sleep.target suspend.target hibernate.target hybrid-sleep.target
```

#### 네트워크 끊김 시 해결방법

UTM의 문제인지 Ubuntu 20.04의 문제인지 간헐적으로 네트워크가 다운되는 현상이 있다. SSH로 접속할 수도 없으며 서버 내부에서도 네트워크가 작동하지 않는다.

정확한 이유는 모르겠지만 서버에 접속해 다음과 같은 shell script를 백그라운드에서 돌려서 문제를 해결할 수 있다.

네트워크를 15초마다 확인하고 네트워크 연결이 없을 때 해당 네트워크 인터페이스를 재시작하는 스크립트이다.

```sh
#!/bin/bash
if [ $(whoami) != root ]
then
        echo This command must be run as root >&2
        exit 1
fi

while true
do
        now=$(date +'%Y-%m-%dT%H:%M:%S')
        if curl --silent --show-error --max-time 5 https://cloudflare-ipfs.com/ipfs/bafkreihdwdcefgh4dqkjv67uzcmw7ojee6xedzdetojuzjevtenxquvyku
        then
                continue
        else
                echo "$now - Internet is not OK"
                ip link set enp0s2 down
                ip link set enp0s2 up
        fi
        sleep 15
done
```

위와 같이 `network-recovery.sh`를 작성하고 다음 명령어로 백그라운드에서 실행한다.

```sh
nohup ./network-recovery.sh &
```

만약 네트워크가 끊긴다면 `nohup.out`에 로그가 기록되고 네트워크가 재부팅될 것이다.

## 다음으로

Ubuntu를 가상머신으로 띄우고 서버로 이용하기 위한 기본 설정을 해주었다. 이제 Ubuntu에 쿠버네티스를 설치해보자.