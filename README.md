# SSM Seoul - Practical Issues in Data Engineering
> Practical Issues in Data Engineering for DBA Course

## 도커 컨테이너 정상 확인

```bash
# 설치 확인 사항
#* 도커 데스크탑 엔진 설치
#* 윈도우의 경우 WSL2
#* 깃 클라이언트 설치

# 도커 및 컴포즈 버전 확인
docker --version
docker-compose --version
git --version

# 워킹 디렉토리 생성
mkdir -p ~/work
cd ~/work

# [최초] 포크한 URL 통하여 로컬에 레포지토리 클론
github_id="깃헙아이디"
git clone "https://github.com/${github_id}/ssm-seoul-data-engineer.git"
cd ~/work/ssm-seoul-data-engineer

# [갱신] 기 클론한 레포지토리의 경우 업데이트
github_id="깃헙아이디"
cd ~/work/ssm-seoul-data-engineer
git pull

# 우분투 컨테이너 기동 테스트
docker-compose up -d ubuntu
docker-compose exec ubuntu echo hello ssm seoul

# 우분투 컨테이너 종료
docker-compose down

```



## Apache Sqoop Tutorial

### 컨테이너 기동

```bash
cd ~/work/ssm-seoul-data-engineer/sqoop
docker rm -f `docker ps -aq | awk '{print $1}'` # 이전에 사용된 컨테이너가 존재하는 경우 삭제
docker container prune # 이전에 사용된 캐시 컨테이너 삭제
docker network prune # 이전에 사용된 캐시 네트워크 삭제
docker-compose up -d

# 예제 테이블 생성위해 mysql 접속
docker-compose exec mysql mysql -uscott -ptiger
```

### 예제 테이블 생성

```sql
# 예제 테이블 생성위해 mysql 접속
docker-compose exec mysql mysql -uscott -ptiger

# 예제 테이블 생성 및 데이터 입력
USE default ;
CREATE TABLE student (
	no INT NOT NULL AUTO_INCREMENT
	, name VARCHAR(50)
	, email VARCHAR(50)
	, age INT
	, gender VARCHAR(10)
	, PRIMARY KEY (no)
) ; 
INSERT INTO student VALUES (1, 'suhyuk', 'suhyuk@gmail.com', 18, 'male') ;
INSERT INTO student VALUES (2, 'psyoblade', 'psyoblade@naver.com', 28, 'female') ;
```

### 예제 테이블 수집

```bash
# 예제 테이블 생성위해 sqoop 서버 접속
docker-compose exec sqoop bash

# 예제 테이블 수집 위한 명령어 실행
ask sqoop import -jt local -fs local -m 1 --connect jdbc:mysql://mysql:3306/default?serverTimezone=Asia/Seoul --username scott --password tiger --table student --target-dir /home/sqoop/target/student

# $ sqoop import -jt local -fs local -m 1 --connect jdbc:mysql://mysql:3306/default?serverTimezone=Asia/Seoul --username scott --password tiger --table student --target-dir /home/sqoop/target/student
# 위 명령을 실행 하시겠습니까? [y/n] y

# 수집 데이터 확인
cat ~/target/student/part-m-00000
1,suhyuk,suhyuk@gmai.com,18,male
2,psyoblade,psyoblade@naver.com,28,female
```



## TrasureData Fluentd Tutorial

### 실습을 위한 컨테이너 기동

```bash
cd ~/work/ssm-seoul-data-engineer/fluentd
docker rm -f `docker ps -aq | awk '{print $1}'` # 이전에 사용된 컨테이너가 존재하는 경우 삭제
docker container prune # 이전에 사용된 캐시 컨테이너 삭제
docker network prune # 이전에 사용된 캐시 네트워크 삭제
docker-compose up -d
```

### 초기화 및 파일 수집 컨테이너 기동

```bash
# 임의의 터미널에서 컨테이너 기동
docker-compose up -d
docker-compose exec fluentd bash

# 이전에 실행된 파일 삭제
rm /fluentd/source/accesslogs.pos
rm /fluentd/source/accesslogs.?
rm -rf /fluentd/target/*

# 파일 수집 컨테이너 설정 확인 및 실행
more /etc/fluentd/fluent.conf
./fluentd
```

### 예제 아파치 로그 생성

```bash
# 별도의 컨테이너 생성
docker-compose exec fluentd bash

# 예제 더미로그 생성 파이썬 실행
more flush_logs.py
```

### 파일 수집 적재 경로 확인

```bash
# 별도의 컨테이너 생성
docker-compose exec fluentd bash

# 적재 대상 경로에 파일이 잘 생성되는 지 확인
for x in $(seq 1 100); do tree -L 1 /fluentd/source; tree -L 2 /fluentd/target; sleep 10; done
```

