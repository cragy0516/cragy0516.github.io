---
layout: post
title: ! 'Deploy Nginx And CTFd with Docker'
excerpt_separator: <!--more-->
tags:
  - Development
  - CTFd
---

오랜만에 개발 (?)

Nginx + CTFd를 Docker로 설치 해 본다.

<!--more-->

## 개요

현재 [CTFd](https://github.com/CTFd/CTFd) 플랫폼을 이용해서 CTF 페이지 개발하는거에 재미가 들려있는데.. 한창 개발하던 도중 `Docker`로 제공된 `CTFd Image`는 `HTTPS`연결을 설정할 수가 없었다...!!!

`CTFd Image`는 `gunicorn + flask`를 이용해서 pyuthon web 어플리케이션이 제작되어 있는데, `Docker`로 이미지를 설치해서 `run` 하면 해당 어플리케이션이 바로 실행되는 구조였다.

아마 `gsunicorn`이나 `flask`에서 제공하는 자체 내장 웹 서버를 사용하는 것으로 보였는데, `gunicorn`은 안 써봐서 모르겠고 하여튼간에 그냥 여기다가 인증서 넣고 대충 설정 끼적이면 `HTTPS` 연결이 되겠거니 했는데… **안됨 OTL**

아, 포기할 수는 없지않나. 그래서 해봤다 삽질.

일단 현재 `Docker Container`가 된 `MyCTFd` (가칭)는 계속 돌려놔야한다. 이거 `commit` 해서 `Docker image`로 떠 놓은 뒤 얘로 `docker-compose`를 통해서 `nginx + CTFd` 설치하면 깔끔하지 않느냐? 물론 맞는말이긴 한데, 이럼 db고 뭐고 싹다 날아가서 설정 처음부터 다시해야되고 문제도 처음부터 다시올려야되는데 db만 백업해서 써도 되긴하지만 귀찮지 않은가. ~~(의식의 흐름 무엇;)~~ `docker-compose`를 통해서 `nginx + CTFd`를 설치하는 방법은 pass. (지금 상황이 이래서 그렇지 보통은 이렇게 한다… 아직 개발을 시작 안한 분이라면 이 방법으로 하시길 ㅠ github 등지를 잘 찾아보면 나옴.)

그럼 이제 어떻게 해야하지? 정답은 `nginx docker image`를 이용해서 컨테이너를 만든 뒤 이거랑 `CTFd`를 통신시키기로 했다. 이게 뭔소리냐고? 그래서 그림을 준비했다.

![]({{ site.baseurl }}/images/KRater/2019-01-24-Deploy-Nginx-And-CTFd-with-Docker/pic01.png)

? 암튼 계속 해보자

## 큰일

났다. nginx 설정은 항상 `certbot`이 다 해줘서 난 손가락만 빨고 있었는데.. 이번엔 직접 해야하네? ㅎㅎ… 그것도 남들이 안하는 개!뻘!짓! 으로!

그래도 뭐 기본적인 틀은 똑같다. 먼저 하나의 `docker network`에 묶어주고, `nginx`만 호스트에다가 포트 열어준 뒤 `nginx` > `CTFd`로 포워딩 해 주도록 하면 된다. 그리고 `let's encrypt`에서 인증서도 받아주고.. 막상 이렇게 모아놓으니까 쉬운데 옵션 제대로 알려주는곳이 없어서 해맸다 ㅠ

## 도커 네트워크 맹글기

기본적으로 도커 네트워크가 다 묶여있는 친구들이긴 한데, 확장성을 고려해서 따로 묶어주기로 하였다. 물론 이런 소규모 프로젝트에 확장성이 왠말이냐고 하겠지만 뭐 연습하는셈치고 해보도록 하자.

일단 도커 네트워크를 만들어 줘야 한다.

`docker network create tmp_network`

그리고 네트워크 추가는 이렇게 한다

`docker network connect tmp_network app`

조와조와

## nginx 이미지 생성

`nginx` 이미지를 생성해야 하는데, 일단 결과물부터 놓고 보자. 아래와 같이 생성했다.

`docker run -d --name=nginx_live -v /etc/letsencrypt:/opt/cert -p 80:80 -p 443:443 nginx`

옵션이 좀 붙은게 많은데 별거 없다. 하나씩 보자.

`-d` : 백그라운드로 돌리려고..아니 이건 왜보고 있는거야 다음

`--name=nginx_live` : 컨테이너 이름. 더 이상 설명이 feel yo 한가?

`-v /etc/letsencrypt:/opt/cert` 이따 설명할 letsencrypt 인증서 관련된 경로다.

`-p 80:80 -p 443:443` : HTTP랑 HTTPS request를 받을 포트

`nginx` : 이미지 이름

별거 업다 80번 포트랑 443번 포트로 리퀘스트 받으면 nginx가 처리해주겟찌. 머 대충 이런거다

`/opt/cert`가 쫌 중요한데 `nginx`가 `ssl`통신할 때 인증서 받아오는 곳이기 때무니다

## nginx 설정 파일 수정

수정이라고 해놨는데 걍 만드는게 신상에 이롭다. 다음과같이 만든다

```bash
server {
    listen       80 default_server;
    listen      [::]:80 default_server;
    server_name  _;
    return      301 https://$host$request_uri;
#    location / {
#        proxy_pass http://172.18.0.3:8000;
#    }
}

server {
    listen      443;
    listen      [::]:443;
    ssl         on;

    server_name localhost;
    ssl_certificate     /opt/cert/live/your.domain.here/fullchain.pem;
    ssl_certificate_key /opt/cert/live/your.domain.here/privkey.pem;

    location / {
        proxy_pass http://app:8000;
    }
}
```

별거 업다

여기서 80번 포트로 접속하면 https로 돌려주고, 443으로 접속하면 `/opt/cert/` 아래 디렉토리 뒤져서 인증서 가지고 보안연결 한뒤 `aha-ctfd:8000`으로 걍 평문전송 하는거다. 왜 ? 이미 도커 내부니까 굳이 HTTPS 적용할 필요가 없어졌거든 ㅎ

`/opt/cert/`가 아까 `mount`한 인증서 있는 디렉토리란거 생각해두고, IP 주소 말고 컨테이너 이름으로 접속 가능한것도 알아두면 편하다. 그리고 `default_server `까먹지 말고. 안하면 오류남

암튼 세팅은 다 햇으니 인증서를 적용시켜보자

## 인증서ㅏ어어

`letsencrypt`하세요. 무료지, 갱신 무제한이지. 완죤 최애자너

현재 도커 외부 호스트가 `Ubuntu 16.04` 사용중이니까, `sudo apt-get install certbot`으로 `certbot`설치합시다. 인증서는 `sudo certbot certonly` 하고 `standalone`으로 설치 했다. 생각해보니 `nginx` 모듈 설치해도 됬을라나; 암튼 인증서를 호스트에 다운받을 작정이고 `nginx`는 도커 컨테이너중 하나일 뿐이니 걍 이렇게 햇다

암튼 인증서를 다운받고 위와같이 적용하면 장땡이긴한데 인증서 자동갱신도 추가해야되서 코드하나 만들었다. 루트권한 잊지마시고

```bash
#!/bin/bash

docker stop nginx_live
docker stop app

certbot renew

docker start app
docker start nginx_live
```

`certbot renew --dry-run` 하면 테스트 가능. 사실 저렇게해도 로그만 읽을줄 알면 별 무리없이 설치된다.

 수고스럽게 `crontab`도 수정 해 준다.

`0 12     * * *   root    /path/to/autorenew/autorenew.sh`

왜 정오냐고? 0시에는 문제풀어야지 문제풀다가 끊기면 빡치자너 > <

암틍갱에 요래하면

## 끝

![]({{ site.baseurl }}/images/KRater/2019-01-24-Deploy-Nginx-And-CTFd-with-Docker/pic02.png)

HTTPS` 적용된다 앗흥 > <

내일은 일해야지..하..
