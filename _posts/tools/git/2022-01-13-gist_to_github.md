---
layout: single
title: "gist에서 commit하면 잔디가 안쌓일 때"
categories: tools
tag: [github, gist, tip]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---

# gist에서 github repository로 복제

## 개요
gist는 좋은 툴이지만 gist에서 commit하고 push를 하면 github profile에 잔디가 쌓이지 않는 단점이 있다.:cry: 그럴땐 gist에 저장된 내용을 github에 복제해 잔디를 쌓을 수 있다.

**local에서 remote저장소를 추가해 github로 push할 수 있다.**

## 방법

```sh
> cd [gist local 저장소]
```
컴퓨터에 저장되어있는 gist의 local저장소로 이동한다.

```sh
> git remote -v


[result] :
origin https://gist.github.com/[xxxxxxxxxxxxxxxxxxxxxxxxxxx].git (fetch)
origin https://gist.github.com/[xxxxxxxxxxxxxxxxxxxxxxxxxxx].git (push)
```
위와 같이 입력하면  자신의 gist주소에 origin(또는 자신이 지정한 이름)이라는 이름으로 remote가 되어있는 것을 볼 수있다.  

새로운 github 레포지토리를 생성하고 주소를 복사한다.\
(외부에 공개하기 싫다면 private 레포지토리를 만들어라)


```sh
> git remote add github [자신의 깃허브 레포지토리 주소 붙여넣기]
> git remote -v


[result] :
github	https://github.com/dukcode/.[자신의 github 레포지토리 이름].git (fetch)
github	https://github.com/dukcode/.[자신의 github 레포지토리 이름].git (push)
origin	https://gist.github.com/[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx].git (fetch)
origin	https://gist.github.com/[xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx].git (push)
```

위와 같이 github라는 이름으로 github 레포지토리에 리모트 된것을 볼 수 있다.\
(github라는 이름말고 다른 이름을 써도 된다.)

```sh
> git push github master
```

위와 같이 입력하면 정상적으로 github 레포지토리에 push되고 잔디도 쌓이는 것을 볼 수 있다.:+1:
