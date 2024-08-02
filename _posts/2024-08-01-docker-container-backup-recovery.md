---
title: mysql 도커 컨테이너 백업 및 복구하기 (feat. Windows)
date: 2024-08-02 20:30 +0900
categories: Tutorial
tags: docker container recovery backup windows
---
PC를 교체하면서 기존 PC에 저장되어 있던 mysql 도커 컨테이너를 옮겨야 할 상황이 생겼습니다. 구글링한 정보로 도커 컨테이너를 옮기는 데에는 성공했으나, 대부분의 자료들은 리눅스 환경 명령어 기준으로 작성되어 mysql DB 덤프를 복원하는데 실패하였습니다.

이에 도커 컨테이너의 이미지 백업/복구 절차와 동시에 Windows 환경에서 컨테이너에 mysql DB를 복원하는 절차에 대해 정리해 보고자 합니다.

컨테이너 외부 윈도우 커맨드에서 직접 명령을 실행하는 경우 명령어 문자가 정상 인식되지 않아 컨테이너 내부 쉘에 접속해서 직접 복구 명령을 실행하는 방법으로 진행했습니다.

## 컨테이너 이미지 백업
### 절차
1. 현재 컨테이너를 이미지로 저장한다.
```console
> docker commit -p [컨테이너ID] [저장할 이미지명]
```
2. 저장된 이미지를 확인한다.
```console
> docker images
```
3. 생성한 이미지를 tar 파일로 저장한다.
```console
> docker save -o [저장할 파일명].tar [이미지명]
```

### 결과
```console
> docker commit -p b9598c8c9391 mysql_backup
sha256:dc727040a6b465918a3bc836c68c5567d4c2e54af99e581ff7bbac67906dd8c5

> docker images
REPOSITORY     TAG       IMAGE ID       CREATED         SIZE
mysql_backup   latest    15e8c2e141aa   3 seconds ago   643MB

> docker save -o mysql_backup.tar mysql_backup
```

## DB 백업
### 절차
1. 컨테이너 쉘로 접속한다.
```console
> docker exec -it [컨테이너명] /bin/bash
```
2. dump 파일을 생성 후 쉘을 나온다. (모든 DB 백업)
```console
# mysqldump -u[계정ID] -p[계정PW] --all-databases=TRUE > [덤프 파일명]
# exit
```
3. 덤프 파일을 컨테이너 외부로 복사한다.
```console
> docker cp [컨테이너명]:[덤프 파일 경로] [외부에 저장할 파일명]
```

### 결과
```console
> docker exec -it mysql-test /bin/bash
bash-4.4# mysqldump -uroot -proot --all-databases=TRUE > db_backup.dmp
bash-4.4# exit
> docker cp edu-mysql:/db_backup.dmp ./db_backup.dmp
```

## 컨테이너 복원
### 절차
1. 백업한 tar 파일을 복원할 PC에 가져와서 load 명령으로 이미지를 불러온다.
```console
> docker load -i [파일명]
```

2. 복원된 이미지를 확인하고, 컨테이너를 생성 및 실행한다.
```console
> docker images
> docker run -d -p [호스트포트]:[컨테이너DB서버포트] --name [컨테이너명] [이미지ID]
```

### 결과
```console
> docker load -i mysql_backup.tar
39d129b028b4: Loading layer [==================================================>]  118.8MB/118.8MB
a9c98ad38ccb: Loading layer [==================================================>]  11.26kB/11.26kB
43e09af64193: Loading layer [==================================================>]  2.359MB/2.359MB
Loaded image ID: sha256:15e8c2e141aa835ae2f9e280b719819064683d14b864c4a5a105bba04d2ee119

> docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
<none>       <none>    15e8c2e141aa   1 weeks ago   632MB

> docker run -d -p 3306:3306 --name edu-mysql 15e8c2e141aa
04efda434ff06e0f60bf716aae053d85d71c2f48f67a3b13cefc0cb022fab73d

> docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED         STATUS         PORTS                               NAMES
04efda434ff0   15e8c2e141aa   "docker-entrypoint.s…"   7 minutes ago   Up 7 minutes   0.0.0.0:3306->3306/tcp, 33060/tcp   edu-mysql

```
생성 및 실행된 컨테이너를 확인할 수 있다.

## DB 복구
### 절차
덤프 파일을 컨테이너 내부로 복사 후 컨테이너 내부에 접속하여 디비 복구를 진행한다.

1. 덤프 파일을 컨테이너 내 적절한 위치로 복사한다.
```console
> docker cp [덤프 파일] [컨테이너명]:[복사 위치]
```

2. 컨테이너 쉘 접속 후 덤프 복원 명령을 수행한다.
```console
> docker exec -it [컨테이너명] /bin/bash
bash-4.4# mysql -u[계정ID] -p[계정PW] < [복사 위치]/[덤프 파일]
```

### 결과
```console
> docker cp ./db_backup.dmp edu-mysql:/home
Successfully copied 11MB to edu-mysql:/home

> docker exec -it edu-mysql /bin/bash
bash-4.4# cd home
bash-4.4# mysql -uroot -proot < db_backup.dmp
```



## References
- https://velog.io/@sherlockid8/docker-container-%EC%82%AD%EC%A0%9C%EC%99%80-%EB%B3%B5%EA%B5%AC-%EB%B0%A9%EB%B2%95-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88%EB%A5%BC-%EC%82%AD%EC%A0%9C%ED%95%98%EB%A9%B4-%EB%B3%B5%EA%B5%AC-%EB%B6%88%EA%B0%80%EB%8A%A5%EC%9D%B8%EA%B0%80
- https://velog.io/@kimdy0915/DB-MySQL-%EB%B0%B1%EC%97%85-%EB%B3%B5%EC%9B%90