# SSM Seoul - Practical Issues in Data Engineering
> 2022 Practical Issues in Data Engineering for DBA Course

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

# 포크한 URL 통하여 로컬에 레포지토리 클론
github_id="깃헙아이디"
git clone https://github.com/${github_id}/ssm-seoul-data-engineer.git
cd ~/work/ssm-seoul-data-engineer

# 우분투 컨테이너 기동 테스트
docker-compose up -d ubuntu
docker-compose exec ubuntu echo hello ssm seoul
```



## rename master to main

```bash
git branch -m master main
git fetch origin
git branch -u origin/main main
git remote set-head origin -a
```
