---
layout: single
title:  "APM Scouter Set"
date:   2020-03-31 21:00:00
author: HyeHwan Choi
categories: job
tags:   Scouter
cover:  "/assets/instacode.png"
---

## 들어가며
오늘은 APM 중 Scouter 설정 방법에 대해 글을 작성하려고 합니다.  
  
APM에 관심을 갖게 된 이유는 시스템을 관리하면서 사용자들이 사용하다가  
에러가 발생할 때 전화, 쪽지를 받을 때면 마음이 촉박해지고 업무에 지장이 있었습니다. 물론 에러가 발생하지 않도록 시스템을 만들어야겠지만요 !  
  
그래서 에러가 발생했을 때 내가 미리 알 수 있거나 빠르게 처리할 수 있는 방법을 찾다가 
APM을 알게 되었습니다. 간단하게 Slack 이라는 툴을 통해 에러에 대해 알림을 받을 수도 있지만, 더 정확하게 에러를 파악하기 위해 APM을 알아보게 되었습니다.  
Scouter에서 발생한 alert을 slack과 연동하여 알림을 받을 수도 있습니다.  

### APM 이란 ?
APM이란 Application Perpomance Monitoring 이며, 애플리케이션의 성능을 모니터링하고 
통제할 수 있는 도구입니다.  
  
그 중 Scouter는 LG CNS에서 개발한 도구입니다. 오픈소스로 무료로 이용할 수 있으며, 
사용법이 간단하고 다양한 OS, was를 지원하여 Scouter를 사용하게 되었습니다.  

### 설치 방법
설치가 간단하지만 최대한 자세하게 설명하려고 합니다.  
[Scouter github](https://github.com/scouter-project/scouter/releases)  
위에 링크된 페이지로 접속하여 최신 릴리즈의 **scouter-all**을 다운받습니다.  
다운받은 뒤 압축을 해제하면 다양한 폴더가 있습니다.  
먼저 `agent.java\conf\scouter.conf` 파일을 열어보면 **obj_name**이 보이는데 scouter client에서 확인 될 이름입니다.  
  
그 다음 `agent.host\conf\scouter.conf` 파일을 확인합니다.  
scouter.conf 파일에서 **net_collector_ip**를 확인합니다.  
이 ip는 모니터링 할 PC의 IP를 입력하면 됩니다. 또 아래 보이는 것처럼 기본 PORT는 6100 입니다.  
  
이제 마지막 **VM arguments** 설정만 남았습니다. 먼저 이클립스에서 설정을 해보겠습니다.  
이클립스에서 **servers** 목록 중 설정 할 서버를 더블클릭하면 Overview가 나오는데  **General Information**에 *Open launch configuration*을 클릭합니다.  
  
보이는 탭 중 **Arguments** 를 클릭하고 *VM arguments*에 적당한 위치에 아래 경로를  절대경로로 추가한 후 저장을 하고 서버를 시작합니다.
```
-Dscouter.config=절대경로\agent.java\conf\scouter.conf
-javaagent:절대경로\agent.java\scouter.agent.jar
```
    
아래 그림처럼 서버가 시작한다면 설정은 완료 되었습니다 !  
![eclipse_start](/assets/images/eclipse_start.png)    
  
이제 Scouter 파일을 실행하고 모니터링 툴에 접속만 하면 됩니다.
*agent.host*에 **host.bat**과 *server* 에 **startup.bat** 파일을 각각 실행합니다.  
그 다음 모니터링하려고 하는 PC에서 scouter github 페이지에 접속하여 사양에 맞는 **scouter.client**를 다운받아 *scouter.exe* 파일을 실행하면 됩니다.  
  
실행하면 로그인을 해야 합니다. Server Address는 서버가 동작중인 IP를 입력해야하며, 초기 ID와 Password는 admin/admin 입니다.  
![scouter_login](/assets/images/scouter_login.png)    
  
접속을 하고 시스템이 동작하면 이와 같은 화면이 보이고 에러 등 다양한 것들을 모니터링할 수 있습니다 ! 참고로 저는 windows와 resin을 사용하기 때문에 scouter.client에서 Window -> preferences에서 설정을 변경해주었고, 모니터링이 필요한 건 Collector에서 추가하였습니다.
![scouter_client](/assets/images/scouter_client.png)    
  
이클립스가 아닌 실제 사용하는 서버에서 설정을 해야 한다면 톰캣 서버는 구글링을 하면 많은 정보가 검색이 됩니다.  
  
저는 resin을 사용하는데 **resin.xml** 파일에서 아래와 같이 추가를 하면 설정이 완료됩니다.
```
<server-default>
	<jvm-arg>-Dscouter.config=절대경로\agent.java\conf\scouter.conf</jvm-arg>
	<jvm-arg>-javaagent:절대경로\agent.java\scouter.agent.jar</jvm-arg>
</server-default>
```
