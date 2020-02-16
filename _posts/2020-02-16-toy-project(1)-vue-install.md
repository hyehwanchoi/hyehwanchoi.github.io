---
layout: single
title:  "ToyProejct(1) Vue Install"
date:   2020-02-15 11:14:00
author: HyeHwan Choi
categories: toy-project
tags:   vue
---

## 들어가며
공부를 위해 토이 프로젝트를 진행하게 되었으며, 기록을 남겨두려고 합니다.  
그 첫번째로 frontend에 vue를 사용하기 위해 설치를 하려고 합니다.    

### vue 란?
vue는 Evan You가 만들었으며, 자바스크립트 프레임워크 입니다.  
MVVM 패턴을 기반으로 디자인 되었고, 러닝커브가 적다는 이점이 있어  
vue를 사용하려고 합니다.    

### 1. vue Install
다양한 방법의 vue 설치 방법이 있지만, 
저는 npm을 이용하여 설치할 것입니다.    

npm을 이용하기 위해선 node를 설치해야 합니다.(생략)    

먼저 npm을 이용하여 vue를 설치하겠습니다.  
vue cli의 패키지 이름이 2 버전에서 3 버전으로 넘어가며 변경됐다고 합니다.  
기존 설치된 2 버전의 vue cli가 있다면 삭제 후 설치를 진행하겠습니다.  

```
1. npm i -g @vue/cli        // 삭제
2. npm install r -g vue-cli // 설치
3. vue --version            // 버전확인
```

다음은 생성된 프로젝트 디렉토리에 vue를 설치하겠습니다.  
프로젝트 폴더에 src와 같은 레벨로 frontend라는 폴더로 vue를 생성할 것 입니다.  
cmd 창에서 프로젝트가 설치되어있는 경로로 이동합니다.  
이제 vue 프로젝트를 생성하면 됩니다!  

```
vue create frontend
```

![vue-installing](/assets/images/vue_installing.png)    

설치가 진행되어 완료가 되면 아래와 같은 명령어를 입력하라고 안내가 되며,  
안내된 명령어를 입력하면 서버가 구동되고 localhost:8080으로 접속하여  
아래와 같은 화면이 나온다면 vue 설치가 완료된 것입니다.
```
cd 프로젝트명
yarn serve 
```
![vue-localhost](/assets/images/vue_localhost.png)  