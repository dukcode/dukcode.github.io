---
layout: single
title: "git-secret 사용해보기"
categories: git
tag: [git, git-secret]
toc: true
toc_sticky: true
author_profile: true
sidebar:
    nav: "docs"
header:
  teaser: "https://git-secret.io/images/git-secret-big.png"
# search: false
---

팀 프로젝트나 협업을 진행하다 보면 password나 credential, 서버 주소, port번호 등 민감한 정보가 프로젝트 파일에 존재하는 경우가 있다.

`.gitignore`파일에 해당 디렉토리나 파일을 포함시켜 stage에 올리지 않는 방법이 있지만 해당 방법을 사용하면, remote repository에 해당 정보가 올라가지 않기 때문에 팀원과의 협업에서 민감한 파일을 공유하는데 번거로움이 생긴다.

예를 들어, `.gitignore`처리된 파일을 직접 공유한다면 매번 민감정보가 담긴 파일이 업데이트 될 때마다 공유해야하는 번거로움이 생기고, 또 해당 파일의 버전관리가 전혀 되지 않는 문제점이 생긴다. 또한 자동 배포 시스템 구축 시에도 해당 민감 파일을 따로 업로드 해주어야 해서 자동화의 의미가 퇴색된다.

![](https://git-secret.io/images/git-secret-big.png){: .align-center}

**git-secret**은 이런 문제들의 해결을 도와주는 툴이다. **git-secret**을 이용하면 다음과 같은 장점이 있다.

- 민감 파일을 암호화해 `git` 레포지토리에 포함되게 함으로써, 해당 파일의 git을 통한 공유가 가능하다.
- 민감 파일을 암호화해 `git` 레포지토리에 포함되게 함으로써, 해당 파일의 버전관리가 가능하다.
- 자동 배포 시스템에서 암호화 된 파일을 복호화하는 스텝을 추가함으로써, 자동 배포가 가능하다.

그러면 직접 `git-secret`을 설치해보고 사용방법을 알아보자.

## git-secret 설치

**git-secret**은 두가지 프로그램에 dependency를 가지고 있다. 당연한 이야기 이지만 **git**이 필수로 깔려있어야 하고, 암호화를 위해 **gpg**가 깔려있어야 한다. 따라서 아래와 같은 명령어를 터미널에 입력한다.

```sh
$ brew install git-secret gpg git
```

**gpg**나 **git**이 이미 깔려 있다면 **git-secret**만 설치하면 된다.

## git-secret 사용 방법

새로운 임시 프로젝트를 생성해 예시를 통해 사용 방법을 알아보자.

### git-secret 초기화

당연한 이야기겠지만, 가장 먼저 프로젝트 폴더를 git 저장소로 만들어야 한다. `temp-project`폴더를 만들고 git 저장소로 초기화 한다.

```sh
$ mkdir test-project
$ cd test-project
$ git init
```

이제 해당 프로젝트를 **git-secret**저장소로 만들자.

```sh
$ git secret init
```

위의 명령어로 프로젝트 디렉토리를 **git-secret** 저장소로 만든다. 그러면 `.gitsecret`디렉토리가 프로젝트 안에 생성된다. `.gitsecret` 디렉토리 안에 암호화, 복호화를 위한 key와 암호화, 복호화 할 파일의 목록을 저장한다.

#### gpg key 생성 및 등록

이제 암호화 및 복호화를 위한 key를 생성하고 등록해보자. **gpg**를 이용해 키를 생성한다.

```sh
$ gpg --full-generate-key
```

위의 명령어를 입력하면 **gpg** key를 만들 수 있다. key의 종류나, 정보 등을 물어보게 된다. 해당 질문에 대해 답을 하지 않고 그냥 엔터를 누르면 괄호안의 default값이 적용된다.

```sh
Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 1
```

다양한 키의 종류를 선택할 수 있다. 나는 RSA 방식을 선택했다.

```sh
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072)
Requested keysize is 3072 bits
```

key의 길이를 정한다. default값인 3072를 선택했다.

```sh
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
```

key의 expire 기간을 정한다. 나는 expire기간을 따로 두지 않았다.

```sh
GnuPG needs to construct a user ID to identify your key.

Real name: test
Email address: test@email.com
Comment: 테스트용 gpg 키
You are using the 'utf-8' character set.
You selected this USER-ID:
    "test (테스트용 gpg 키) <test@email.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
```

가장 중요한 이름, 이메일을 정한다. **이메일은 git-secret에서 사용자를 추가할 때 사용하므로 실제 이메일을 주어 다른 팀원과 겹치지 않게 하자.**

코멘트는 있어도 되고 없어도 되지만 해당 키의 용도를 설명해놓으면 나중에 gpg 키 리스트를 볼 때 용도를 알 수 있다.

해당 과정을 마치면 **passphrase**를 정의하는데, 2차 비밀번호와 같은 성격이다. **passphrase**를 2번 입력해준다. 엔터를 계속 눌러주면 **passphrase**를 설정하지 않고 진행할 수도 있다. 하지만 보안상의 이유로 추천하지는 않는다.

이로서 git-secret에서 사용할 gpg key를 생성했다.

#### git-secret에 key 등록

key 등록은 아주 간단하다. 다음과 같은 명령어를 입력해주면 추가된 키로 파일의 암복호화를 진행할 수 있다.

```sh
$ git secret tell test@email.com
```

`test@email.com`을 이메일로 가지는 키를 **git-secret** 저장소에 등록시킨다. 위의 명령어를 입력하면 `.gitsecret`  디렉토리에 새로운 키의 내용이 업데이트 된다.

이로써 `test@email.com`의 키를 가지고 있는 사용자는 해당 프로젝트 디렉토리에서 파일의 암복호화를 진행할 수 있다.

### 암복호화 해보기

이제 암호화 할 파일을 만들고 해당 파일을 git-secret에 등록시키고 암복호화 과정을 진행해 보자.

```sh
$ touch password.txt
```

`password.txt` 라는 파일을 만들고 `vi` 에디터로 내용을 적어보자.

![](https://i.imgur.com/Aql3MTp.png){: .align-center}

필자는 위와 같은 내용을 적었다. 실제 프로젝트에서는 민감한 정보가 적혀있을 것이다.

#### 암호화

해당 파일을 `git-secret`이 관리하도록 다음과 같은 명령어를 통해 추가한다.

```sh
$ git secret add password.txt
```

그러면 아래와 같이 해당 파일이 `.gitignore`에 추가되고 **git-secret**이 관리하기 시작한다.

![](https://i.imgur.com/2xsjK9a.png){: .align-center}

이제 실제로 암호화를 해보자. 아주 간단한 명령어로 모든 **git-secret**이 관리하는 파일을 암호화 할 수 있다.

```sh
$ git secret hide
```

위의 명령어를 입력하면 암호화될 파일이 있는 위치에 `.secret`확장자가 붙은 암호화된 파일이 생성된다.

![](https://i.imgur.com/0CACCGY.png)

이 상태에서 commit과 push를 한다면 원본 파일은 `.gitignore`처리가 되어있기 때문에 remote repository에 올라가지 않고 `.secret`파일만 remote repository에 올라가게 된다.

#### 복호화

remote repository에서 clone이나 pull을 받은 상황을 가정하여 원본 파일을 삭제한다. 실제로 pull이나 clone을 받았을 경우에는 원본 파일이 존재하지 않을 것이기 때문이다.

```sh
$ rm password.txt
```

삭제 결과는 다음과 같다.

![](https://i.imgur.com/cbMDmyG.png){: .align-center}

이제 `.secret`파일을 복호화 해보자.

```sh
$ git secret reveal
```

위의 명령어를 입력하면 다음과 같이 `.secret`파일이 복호화 된다.

![](https://i.imgur.com/Qd8JJ3l.png)

![](https://i.imgur.com/Qn9woBO.png)

이로써, remote repository에서 받은 `.secret` 파일을 간단하게 복호화 할 수 있다.

### 팀원 추가하기

이제 팀원을 추가해 보자. 팀원의 컴퓨터에서 gpg를 생성하고 public 키를 받아 해당 public 키를 **git-secret**에 등록시켜 보자.

#### 팀원이 gpg key 생성하고 export 하기

팀원의 컴퓨터에서 다음 명령어를 실행해 gpg 키를 만들고 public key를 export한다. `[키 아이디]`는 gpg key를 만들 때 사용한 `name`을 말한다.

```sh
$ gpg --full-generate-key
$ gpg --export [키 아이디] > public.key
```

위의 명령어를 실행하면 `public.key`가 생기고 이 파일을 이메일이나 다른 툴(ex. 카카오톡)을 이용해 공유한다. public 키이기 때문에 공유해도 보안상의 문제가 없다. 팀원은 자신의 private키만 노출시키지 않는다면 안전한다.

#### 팀원의 gpg key import 하고 git-secret에 추가하기

팀원의 public key를 받아 git-secret에 추가해 보자. 간단한 명령어로 gpg key를 import 시킬 수 있다.

```sh
$ gpg --import public.key
```

그리고 프로젝트 폴더로 들어가 다음과 같이 입력한다.

```sh
$ git secret tell [팀원의 키 이메일]
```

위의 명령어를 입력하면 `.gitsecret` 디렉토리에 팀원의 키가 업데이트 된다.

이 상태에서 remote repository에 push를 해도 팀원이 암호화된 파일을 복호화 할 수 없다. **`.secret`파일들이 해당 팀원의 키가 추가된 상태에서 암호화 된 것이 아니기 때문이다.**

따라서 다음 명령어로 파일들을 다시 암호화 해준 뒤에 push하자.

```sh
$ git secret hide
```

위의 명령어를 입력하면 기존의 `.secret`파일이 새로 추가된 팀원이 복호화 할 수 있게 다시 암호화된다. 이제 팀원이 pull이나 clone을 받아서 `git secret reveal` 명령어를 사용하면 복호화가 가능하다!

## 주의점

팀원을 추가하는 과정에서 알 수 있듯이, **git-secret**은 암호화 파일을 자동으로 업데이트 시켜주지 않는다. 따라서 다음과 같은 문제가 생길 수 있다.

- remote repository에서 pull을 받고 나서 reveal을 하지 않으면, pull받은 `.secret`파일들이 복호화되지 않아 업데이트 된 비밀 파일을 사용하는 것이 아니라 이전의 파일을 사용하게 된다.
- 팀원을 추가하고 파일을 다시 암호화 하지 않으면 팀원은 `.secret` 파일을 복호화 할 수 없다. 해당 팀원의 키가 추가된 상태에서 암호화된 파일이 아니기 때문이다.
- 암호화 될 파일을 업데이트하고 commit 전에 `git secret hide`를 하지 않으면, 업데이트 내용이 반영되지 않는다.

따라서 결론은 다음과 같다.

> - pull이나 clone, checkout 등 git의 branch가 업데이트 되거나 옮겨다닐 경우 `git secret reveal`을 해주자
> - commit 전에 항상 `git secret hide`를 해주자

위의 두 규칙을 지킨다면 암호화 된 파일은 항상 해당 버전을 유지할 것이다.

**git hook**으로 해당 과정을 자동화 해보려고 했지만, `pre-commit`에 파일을 추가하면 stage에 반영되지 않는 문제가 있어 자동화를 하지 못했다. **git hook**으로 해당 과정을 자동화하는 방법이 있다면 새로운 포스트에 포스팅해보겠다.