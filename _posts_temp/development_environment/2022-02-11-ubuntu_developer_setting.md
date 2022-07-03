---
layout: single
title: "[Ubuntu] 초기 개발환경 설정"
categories: ubuntu
tag: [ubuntu, nvim]
toc: true
toc_sticky: true
author_profile: false
sidebar:
  nav: "docs"
search: true
---
# ubuntu 개발환경 세팅

## 기본 설정

### 1. 기본 환경설정

* 대기화면 전환 끄기

  [Setting] - [Privacy] - [Screen Lock] - [Blank Screen Delay] - [Never]

  [Setting] - [Privacy - [Screen Lock] - [Automatic Screen Lock] - [Off]

  

###  2. 터미널 설정

#### 미러서버 카카오로 변경

  ```sh
  $ sudo vi /etc/apt/sources.list
  ```

  vi창에서 
  ```
  :0,$/kr.archive.ubuntu.com/mirror.kakao.com/g
  
  or
  
  :0,$/us.archive.ubuntu.com/mirror.kakao.com/g
  ```

#### apt 설정
  ```sh
  $ sudo apt-get update		# 패키지 목록 최신으로 갱신
  $ sudo apt-get dist-upgrade # 의존성 체크 업그레이드
  ```
#### zsh 설치 및 설정
* zsh 설치 및 oh my zsh 설치
  ```sh
  $ sudo apt-get install zsh		# zsh 설치
  $ sudo apt-get install curl		# oh my zsh를 위한 curl 설치
  $ sudo apt-get install git		# oh my zsh를 위한 git 설치
  
  # oh my zsh 설치
  $ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
  ```



* zsh 테마 설정

  ```sh
  $ vi ~/.zshrc
  ```
  ZSH_THEME="robbyussell" 부분을 ZSH_THEME="agnoster"로 변경

  

* zsh 사용자 이름 출력 부분 수정  
  맨 아래에 위의 내용 추가
  

{% raw %}
  ```sh
  prompt_context() {
    if [[ "$USER" != "$DEFAULT_USER" || -n "$SSH_CLIENT" ]]; then
      prompt_segment black default "%(!.%{%F{yellow}%}.)$USER"
    fi
  }
  ```
{% endraw %}



* zsh plugin 설치
	syntax-highlighting : 타이핑 시 구문 강조 플러그인
	autosuggestions : 히스토리 기반 명령어 추천 플러그인
    ```sh
    $ git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
    $ git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
    $ vi ~/.zshrc
    ```
    ```sh
    plugins=(git)
    
    # 부분 다음으로 변경
    
    plugins=(
        git
        zsh-syntax-highlighting
        zsh-autosuggestions
    )
    ```
  
    터미널에서 아래와 같이 입력
    ```sh
    $ source ~/.zshrc
    ```



* D2Coding_Nerd 폰트 다운로드/설치 및 설정

  ```sh
  $ mkdir -p ~/util/fonts
  $ cd ~/util/fonts
  $ git clone https://github.com/kelvinks/D2Coding_Nerd.git
  $ sudo cp -rf D2Coding /usr/share/fonsts
  $ fc-cache -f -v
  ```

​		터미널 설정에서 글꼴 D2Coding Regular로 변경

### 3. 한글 설정 및 한국 시간대 설정

* 한글 입력 설정
  1. [Apps(왼쪽아래 점 9개)] - [Language Supprot] - [Install / Remove Languages...] - [Korean 체크] - [Apply]
  
  2. **목록에 한국어 안뜰 시** : [Korean 체크 해제] - [Apply] - [Korean 체크] - [Apply]
  
  3. 터미널 열고 아래 입력
  
     ```sh
     $ ibus-setup
     ```
     
  4. [Input Method] - [Add] - [...] - [Korean 검색] - [Hangul] - [Add]
  
  5. [Settings] - [Region & Language] - [+] - [Korean] - [Korean(Hangul)] - [Add]
  
  6. [Korean(Hangul) 톱니바퀴] - [Add] - [한/영키로 사용하고 싶은 키 입력] - [OK]
  
  7. [Apply] - [OK]
  
  7. [입력 소스] - [영어(미국) 삭제]
  
     
  
* 우분투 한글 설정(선택사항)

  1. [Settings] - [Region & Language] - [Language] - [한국어] - [Select] - [Restart...]

  2. **(중요!!!) [다시 묻지 않기 체크] - [예전 이름 유지]**

     

* 한국 시간대 설정

  1. [Settings] - [Region & Language] - [형식] - [대한민국] - [완료]

     

### 4. 키보드 설정

* 키보드 딜레이 설정

  터미널에서 아래와 같이 입력

  ```sh
  $ sudo xset r rate 175
  ```



### 5. openjdk JAVA 설치

* 터미널에서 아래와 같이 입력

  ```sh
  $ sudo apt-get install openjdk-17-jdk
  ```

* Java version 확인

  ```sh
  $ java --version
  ```

  

### 6. NeoVim 설치

* 터미널에서 아래와 같이 입력

  ```sh
  $ cd ~/util
  $ git clone https://github.com/neovim/neovim.git
  $ cd neovim
  $ sudo apt-get install ninja-build gettext libtool libtool-bin autoconf automake cmake g++ pkg-config unzip curl doxygen gcc make
  $ make
  $ sudo make install
  $ nvim ~/.zshrc
  ```

* `.zshrc`맨 아래에 아래의 내용 추가

  ```
  alias vi="nvim"
  alias vim="nvim"
  ```

* `.zshrc` 재로딩

  ```sh
  $ source ~/.zshrc
  ```

* 설정 파일 생성

  ```sh
  $ mkdir -p ~/.config/nvim && touch ~/.config/nvim/init.vim
  ```

* 플러그인 설치 프로그램 vim-plug 설치

  ```sh
  sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
         https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
  ```



### 7. NeoVim 추천 플러그인 설치 및 설정



### 8. 추천 설치 목록

* tree

  ```sh
  $ sudo apt-get install tree
  ```

  

