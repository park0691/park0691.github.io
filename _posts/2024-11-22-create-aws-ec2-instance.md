---
title: AWS EC2 인스턴스 생성하기
date: 2024-11-22 12:00 +0900
categories: AWS
tags: aws ec2
---
프로젝트용 서버 구축을 위한 EC2 인스턴스 생성 과정에 대해 기록하고자 한다.

## EC2, Elastic Computing Cloud
AWS에서 제공하는 클라우드 컴퓨팅 서비스로 아마존이 사용자에게 컴퓨팅 자원을 임대해주는 서비스다.

### 특징
- 초 단위 온디맨드 가격 모델
	- 초 단위 컴퓨팅 파워로 측정된 가격을 지불
	- 서비스 요금을 미리 약정하거나 선입금할 필요 없음
- 빠른 구축 속도와 확장성
	- 몇 분이면 전 세계에 인스턴스 수백여 대 구축 가능
- 다양한 구성 방법 지원
	- 머신 러닝, 웹 서버, 게임 서버, 이미지 처리 등 다양한 용도에 최적화된 서버 구성 가능
	- 다양한 과금 모델
- 여러 AWS 서비스와 연동
	- Auto Scaling, Elastic Load Balancer(ELB), CloudWatch

### 인스턴스 생성
1. EC2 대시보드 > 인스턴스 시작 버튼
    ![images](https://1drv.ms/i/s!AmQWleBW71GSeb6zwrWRUhFwGoM?embed=1&width=1276&height=715)

2. 인스턴스 이름 입력

    ![images](https://1drv.ms/i/s!AmQWleBW71GSekZJ9_9SIuV8Ibo?embed=1&width=587&height=167)

    인스턴스의 이름을 적절히 입력한다.

3. 애플리케이션 및 OS 이미지 선택
    ![images](https://1drv.ms/i/s!AmQWleBW71GSe_jeMBmSOgOQpc8?embed=1&width=783&height=806)

    사용할 OS, 적절한 이미지 버전을 선택한다.
    ![images](https://1drv.ms/i/s!AmQWleBW71GSfPum1XXRduiG0h8?embed=1&width=789&height=290)
	인스턴스는 프리 티어로 사용하기 적절한 t2.micro를 선택한다.
	
	> AMI, Amazon Machine Image<br/>
	> 인스턴스를 시작하는데 필요한 정보를 제공하는 이미지를 말한다.
	{: .prompt-info }


3. 키 페어
    ![images](https://1drv.ms/i/s!AmQWleBW71GSfVgacZf96xd_QY4?embed=1&width=791&height=235)
    키 페어의 이름을 입력 후 '새 키 페어 생성' 버튼
    ![images](https://1drv.ms/i/s!AmQWleBW71GSfmbZBkNVEvkRbCk?embed=1&width=608&height=615)
    
    키 페어 유형은 RSA를 기본으로 사용하고, 프라이빗 키 파일 형식은 PuTTY를 사용할 예정이므로 .ppk 를 선택한다. → `키 페어 생성` 버튼

	> 다운로드 받은 키 파일은 다시 받을 수 없기 때문에 잃어버리지 않게 잘 보관해야 하며, 제 3자에게 노출시키면 안 된다.
	{: .prompt-warning}

4. 스토리지 구성
    ![images](https://1drv.ms/i/s!AmQWleBW71GSf2ifyp2hih9DYwM?embed=1&width=787&height=487)
    스토리지 용량은 프리 티어가 지원하는 최대 용량인 30GB로 입력한다.

5. 인스턴스 시작<br/>
    ![images](https://1drv.ms/i/s!AmQWleBW71GSgQCyfKg8fseNjbbf?embed=1&width=386&height=785)
    
    화면 우측의 요약 영역에 `인스턴스 시작` 버튼 클릭시 EC2 인스턴스가 생성된다.
    
    ![images](https://1drv.ms/i/s!AmQWleBW71GSgQE-xlYmB5c1uYJJ?embed=1&width=629&height=192)
    
    ![images](https://1drv.ms/i/s!AmQWleBW71GSc2iwUPOak9DromA?embed=1&width=1441&height=142)
    인스턴스가 시작된 것을 확인할 수 있다.
    

### 탄력적 IP 할당
인스턴스에 고정 IP를 부여하기 위한 기능이다.
![images](https://1drv.ms/i/s!AmQWleBW71GScTgtplfQiJw-gic?embed=1&width=1208&height=170)

기본적으로 EC2 인스턴스는 고정 IP가 할당되어 있지 않으므로 인스턴스를 재시작하는 경우 IP 주소가 변경될 수 있으므로 탄력적 IP를 사용하는 것이 좋다.

> 탄력적 IP를 생성해놓고 사용하지 않으면 과금이 발생하므로 반드시 인스턴스와 연결된 상태이어야 한다. 만약 EC2 인스턴스가 제거되어 탄력적 IP와 연결이 끊겼따면 탄력적 IP 또한 제거해야 한다.
{: .prompt-warning}

1. `탄력적 IP 주소 할당` 버튼 선택
![images](https://1drv.ms/i/s!AmQWleBW71GSckjLePDS-7JWrbo?embed=1&width=1486&height=564)

2. 디폴트로 따로 건드리지 않고 `할당` 버튼 선택
![images](https://1drv.ms/i/s!AmQWleBW71GSdOoXqeALEZzpurk?embed=1&width=815&height=938)

3. 생성된 IP 주소를 확인한다. → 확인한 IP를 체크하고 `탄력적 IP 주소 연결` 버튼 선택
![images](https://1drv.ms/i/s!AmQWleBW71GSdpycAna4Pbj22_c?embed=1&width=1189&height=307)

4. 인스턴스 선택 후, `연결` 버튼
![images](https://1drv.ms/i/s!AmQWleBW71GSdS7dSaFOkOjsx7Y?embed=1&width=813&height=688)

5. 확인
    ![images](https://1drv.ms/i/s!AmQWleBW71GSd_idQtcwoHcb5m4?embed=1&width=1162&height=237)
    
    ![images](https://1drv.ms/i/s!AmQWleBW71GSeCHcqCSxNXgD9xg?embed=1&width=998&height=158)
    
    탄력적 IP가 할당된 것을 확인할 수 있다.

### 보안 설정
생성된 EC2 인스턴스에 보안 정책에 맞는 트래픽만 인입될 수 있다. 디폴트 보안 규칙은 다음과 같다.
![images](https://1drv.ms/i/s!AmQWleBW71GSgQKm7V2tGFJkdAbQ?embed=1&width=1102&height=636)

인바운드 규칙으로 현재 모든 IP에서 SSH(Port 22) 접속만 허용되어 있으며, 아웃바운드 규칙은 모든 IP로 모든 Port, 프로토콜의 트래픽이 나갈 수 있도록 되어 있다.

추후 API 또는 웹 서버를 EC2 인스턴스에 올리는 경우 인바운드 규칙을 추가하여 해당 프로토콜의 포트를 오픈해주어야 한다.

예를 들어 HTTP 서버(Port 80)를 오픈하려면,<br/>
좌측 메뉴 '보안 그룹' > 보안 그룹 선택 > `인바운드 규칙 편집` 버튼

![images](https://1drv.ms/i/s!AmQWleBW71GSgQMbdlDJUwvH7OTi?embed=1&width=1266&height=507)

`규칙 추가` 버튼을 선택하고 다음과 같이 설정할 수 있다.

필자는 외부에서 SSH 접속을 차단하기 위해 모든 IP에 열려있는 인바운드 규칙을 본인 PC IP로 제한하였다.

![images](https://1drv.ms/i/s!AmQWleBW71GSgQROyBXx7TwrCIsF?embed=1&width=1106&height=275)

## EC2 인스턴스 접속하기
### PuTTY 콘솔 활용
1. PuTTY 실행 후<br>
    Connection > SSH > Auth > Credentials 카테고리 선택
    
    ![images](https://1drv.ms/i/s!AmQWleBW71GSgQbGmcVNLDoSHtcb?embed=1&width=447&height=436)
    
    Private key file for authentication > Browse > 다운 받은 키 파일 (`*.ppk`) 선택
    
2. Session 카테고리로 돌아온 후<br/>
    
    ![images](https://1drv.ms/i/s!AmQWleBW71GSgQU_Pius8apJTJGU?embed=1&width=450&height=440)
    
    인스턴스의 퍼블릭 IPv4 주소 입력, 세션 설정을 저장하기 위해 세션의 이름을 지정 후 Save 버튼 → Open 버튼

3. 호스트 키 신뢰 여부 묻는 화면이 나오면 `Accept` 버튼
4. 콘솔 접속 화면이 다음과 같이 뜬다.<br/>
    ![images](https://1drv.ms/i/s!AmQWleBW71GSgQeTPOf9gqH0ZMWR?embed=1&width=741&height=585)
    
    AWS EC2 우분투 유저 계정명은 `ubuntu` 이므로 해당 계정 입력 후 엔터하면<br/>
    콘솔에 정상 접속된 것을 확인할 수 있다.

### AWS 콘솔 활용
EC2 인스턴스 화면에 들어오면
![images](https://1drv.ms/i/s!AmQWleBW71GSgQiX3dDfjDlg3CK0?embed=1&width=856&height=188)
인스턴스 체크 → 상단 `연결` 버튼

![images](https://1drv.ms/i/s!AmQWleBW71GSgQlPEnk_KhavqY5F?embed=1&width=816&height=710)
선택된 EC2 인스턴스 정보를 확인하고 사용자명은 디폴트로 `ubuntu` 입력된 것을 확인할 수 있다.<br/>
`연결` 버튼 선택한다.

![images](https://1drv.ms/i/s!AmQWleBW71GSgQp8XtvvXloO2ZUM?embed=1&width=1126&height=853)
브라우저에서 우분투 콘솔에 바로 접속된 것을 확인할 수 있다.


## References
- https://youtu.be/rdlHszMujnw?si=dIFgZjqy5MN8Fd9r
- https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/LaunchingAndUsingInstances.html