---
layout: post

title: 애드투페이퍼 서비스 구성 Stack
cover_image: blog-cover.png

excerpt: "애드투페이퍼의 서비스 스택을 소개합니다."

author:
  name: 이경찬
  bio: CTO
  image: lkc.jpg
  page: http://leekchan.com
---

안녕하세요, [애드투페이퍼](http://www.add2paper.com)에서 개발을 담당하고 있는 [이경찬](http://www.leekchan.com)입니다. 애드투페이퍼를 개발 및 운영하면서 경험하고 해결한 사항들을 공유하고자 블로그를 만들었습니다. 첫 번째 포스트에서는 애드투페이퍼 서비스 구성 Stack에 대해서 공유하고자 합니다.

<br/>

## Overview

<br/>

### 서버 구성

애드투페이퍼 서비스는 KT Ucloud 인스턴스 7대 위에서 동작하고 있습니다. 각 인스턴스의 역할은 다음과 같습니다.


<ol>
<li>로드밸런서 1대 (1vCore 1GB | Ubuntu 11.04 | HAProxy)</li>
<li>어플리케이션 서버 피크 타임 기준 3대 (4vCore 4GB | Ubuntu 11.04 | Gunicorn, Django, Supervisor) - 애드투페이퍼앱은 매일 아침 8시 30분에 수신 거부자를 제외한 전체 회원(약 30만 명)에게 무료 포인트 적립 푸시를 발송합니다.</li>
<li>DB 2대 (4vCore 4GB | CentOS 5.4 | MySQL Master-Slave Replication 구성)</li>
<li>캐시 + 워커 서버 1대 (4vCore 8GB | Ubuntu 11.04 | Memcached, Celery, Redis, Supervisor)</li>
</ol>


현재는 [SPOF](http://en.wikipedia.org/wiki/Single_point_of_failure) 가 많지만, 점진적으로 개선 중입니다. 궁극적으로는 무중단 인프라를 구축하는 것이 목표입니다. 현재 어플리케이션 서버와 DB는 Failover가 되고, HAProxy 이중화 작업을 최우선 과제로 진행 중입니다.

<br/>

### 서비스 구성
애드투페이퍼 서비스 구성은 다음과 같습니다.

1. 애드투페이퍼 웹서비스 &nbsp;
2. 애드투페이퍼 앱 API 백엔드 &nbsp;
3. 프린터 API 백엔드 &nbsp;
4. 광고주용 실시간 리포트 사이트 &nbsp;
5. 프린터관리자용 실시간 프린팅 및 정산 정보 확인 사이트 &nbsp;
6. 내부 관리용 사이트 &nbsp;


<br/>

## Application Server 구성
1. Python
2. Django
3. Nginx
3. Gunicorn
4. Supervisor

앞서 언급한 것처럼 애드투페이퍼 서비스 코드는 Python, Django로 작성됩니다. 리버스 프록시로는 Nginx, WSGI Container로는 Gunicorn을 사용하고 있습니다. Supervisor는 Gunicorn 프로세스 모니터링 용으로 사용 중입니다.

<br/>

## Cache 구성
1. Memcached
2. [Johnny Cache](http://pythonhosted.org/johnny-cache/)

애드투페이퍼 서비스에서는 모든 Database Read Query를 Memcached에 캐시하고 있습니다. 모든 상황에 대해 견고한 Cache Invalidation 로직을 만드는 작업은 매우 많은 시간이 소요되는 작업이므로, Johnny Cache라는 Cache Framework를 사용하고 있습니다. Johnny Cache는 기본적으로 MySQL의 Query Cache와 비슷하게 동작합니다. Johnny Cache Middleware를 등록해 놓으면 Monkey Patch를 통해 SELECT 쿼리가 발생할 때는 쿼리를 Key로 Memcached에 캐시하고, INSERT 또는 UPDATE 쿼리가 발생할 때는 해당 Table에 관계된 Cache들을 자동으로 Invalidation 해줍니다.

백엔드 코드를 작성할 때는 Memcached의 Cache hit ratio를 높이기 위해 SELECT 문의 WHERE절에는 특정한 상수가 들어가는 것을 피하고 있습니다. 또한 어플리케이션상에서 데이터를 가공할 수 있을 때에는 최대한 어플리케이션에서 코드상에서 처리해서 I/O Bound 되는 것을 피하고 있습니다.

<br/>

## Database / Storage
데이터베이스는 MySQL 5.5 를 사용하고 있고, Master-Slave 구성으로 2대를 사용하고 있습니다.

서비스상에서 업로드 되는 이미지 및 파일은 Amazon S3 (Tokyo Region)에 저장하고 Tcloud CDN을 붙여서 사용하고 있습니다. 광고 이미지를 리사이징 하는 등의 작업은 어플리케이션 서버에서 인메모리로 처리하고 S3에 저장합니다.

<br/>

## Worker 구성
1. Redis
2. Celery
3. Supervisor

시간이 걸리는 Task들은 모두 Task Queue를 이용해 비동기로 처리하고 있습니다. Task Queue는 Celery를 이용하고 있고 작업들을 저장하고 분배하는 Broker 용도로 Redis를 사용하고 있습니다. 그리고 Celery 프로세스의 모니터링 용도로 Supervisor를 사용하고 있습니다.

<br/>

## Deployment

어플리케이션 배포는 [Fabric](http://www.fabfile.org/)을 이용해 자동화하고 있습니다. 배포 시 각 서버에서 실행하는 커맨드는 다음과 같습니다.

<br/>

###어플리케이션 서버

<pre>
[Git 저장소에서 최신 코드 다운로드]
pip install -r ./requirements.txt
supervisorctl status gunicorn | sed "s/.*[pid ]\([0-9]\+\)\,.*/\1/" | xargs kill -HUP
</pre>
저장소에서 최신 코드를 받아온 뒤, 새로 추가된 모듈들을 설치한 후 Gunicorn에 -HUP signal을 보내서 graceful reload 기능(처리 중인 요청을 모두 처리하고 프로세스를 재시작함)을 실행시킵니다. supervisorctl reload 커맨드로 Gunicorn을 재시작하면 재시작하는 도중에는 요청을 받을 수 없는 상태가 되므로, pid를 읽어와서 -HUP 시그널을 보내는 방식으로 재시작시켜서 무중단으로 코드를 변경하고 있습니다.

<br/>

### 캐시 + 워커 서버


<pre>
[Git 저장소에서 최신 코드 다운로드]
pip install -r ./requirements.txt
python ./manage.py migrate
supervisorctl reload
</pre>

캐시 + 워커 서버에서는 [South](http://south.aeracode.org/)의 migrate 커맨드를 실행시켜서 데이터베이스를 마이그레이션하는 작업을 담당하고 있습니다. 모든 작업이 끝난 후에는 supervisorctl reload 명령을 실행시켜서 Celery 프로세스를 재시작 합니다.

<br/>

## Server Monitoring
서버 모니터링은 [datadog](http://www.datadoghq.com/) 서비스를 이용해서 하고 있습니다. 실시간 모니터링이 가능하고 대시보드를 만들어 각 서버의 CPU, Memory, Network 사용량 등을 원하는 대로 가공해서 모아서 볼 수 있어서 효과적입니다.

<br/>

## And more?
지금까지 저희 서비스 구성 Stack에 대해 공유 드렸는데요, 애드투페이퍼 앱에도 흥미로운 공유사항들이 많이 있습니다. 살짝 공유하자면 저희 앱은 HTML, Javascript로 제작되어서 단일 코드 베이스로 iOS / Android 플랫폼 양쪽에 배포하고 있는데요, 후속 포스팅에서 자세히 공유하도록 하겠습니다.

감사합니다.
