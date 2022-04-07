# 도커 컨테이너 생성 실습

> 도커 및 컴포즈를 통한 기본 명령어를 학습합니다

- 범례
  * :green_book: 필수, :blue_book: : 선택

- 목차
  * [1. WSL2 혹은 터미널 환경에 접속](#1-WSL2-혹은-터미널-환경에-접속)
  * [2. Docker 명령어 실습](#2-Docker-명령어-실습)
  * [3. Docker Compose 명령어 실습](#3-Docker-Compose-명령어-실습)
  * [4. 참고 자료](#4-참고-자료)
<br>


## 1. WSL2 혹은 터미널 환경에 접속

> WSL2 혹은 terminal 을 이용하여 도커 명령어 수행이 가능한 환경에 접속합니다


### 1-1. 패키지 설치 여부를 확인합니다
```bash
docker --version
docker-compose --version
git --version
```

<details><summary>[실습] 출력 결과 확인</summary>

> 출력 결과가 오류가 발생하지 않고, 아래의 버전보다 상위 버전이라면 정상입니다

```text
Docker version 20.10.6, build 370c289
docker-compose version 1.29.1, build c34c88b2
git version 2.17.1
```

</details>

[목차로 돌아가기](#도커-컨테이너-생성-실습)

<br>
<br>



## 2. Docker 명령어 실습

> 컨테이너 관리를 위한 도커 명령어를 실습합니다

* 실습을 위해 기존 프로젝트를 삭제 후 다시 클론합니다

```bash
# terminal
mkdir -p ~/work ; cd ~/work
git clone https://github.com/psyoblade/helloworld.git
cd ~/work/helloworld
```
<br>

### Docker 기본 명령어

### 2-1. 컨테이너 생성관리

> 도커 이미지로 만들어져 있는 컨테이너를 생성, 실행 종료하는 명령어를 학습합니다

#### 2-1-1. `create` : 컨테이너를 생성합니다 

  - <kbd>--name <container_name></kbd> : 컨테이너 이름을 직접 지정합니다 (지정하지 않으면 임의의 이름이 명명됩니다)
  - 로컬에 이미지가 없다면 다운로드(pull) 후 컨테이너를 생성까지만 합니다 (반드시 -it 옵션이 필요합니다)
  - 생성된 컨테이너는 실행 중이 아니라면 `docker ps -a` 실행으로만 확인이 가능합니다

```bash
# docker create <image>:<tag>
docker create -it ubuntu:18.04
```
<br>

#### 2-1-2. `start` : 생성된 컨테이너를 기동합니다

* 아래 명령으로 현재 생성된 컨테이너의 이름을 확인합니다
```bash
docker ps -a
```

* 예제의 `busy_herschel` 는 자동으로 생성된 컨테이너 이름이며, 변수에 담아둡니다
```bash
# CONTAINER ID   IMAGE          COMMAND   CREATED         STATUS    PORTS     NAMES
# e8f66e162fdd   ubuntu:18.04   "bash"    2 seconds ago   Created             busy_herschel
container_name="<목록에서_출력된_NAME을_입력하세요>"
```

```bash
# docker start <container_name> 
docker start ${container_name}
```

```bash
# 해당 컨테이너의 우분투 버전을 확인합니다
docker exec -it ${container_name} bash
cat /etc/issue
exit
```
<br>


#### 2-1-3. `stop` : 컨테이너를 잠시 중지시킵니다
  - 해당 컨테이너가 삭제되는 것이 아니라 잠시 실행만 멈추게 됩니다
```bash
# docker stop <container_name>
docker stop ${container_name} 
```
<br>


#### 2-1-4. `rm` : 중단된 컨테이너를 삭제합니다
  - <kbd>-f, --force</kbd> : 실행 중인 컨테이너도 강제로 종료합니다 (실행 중인 컨테이너는 삭제되지 않습니다)
```bash
# docker rm <container_name>
docker rm ${container_name} 
```
<br>


#### 2-1-5. `run` : 컨테이너의 생성과 시작을 같이 합니다 (create + start)
  - <kbd>--rm</kbd> : 종료 시에 컨테이너까지 같이 삭제합니다
  - <kbd>-d, --detach</kbd> : 터미널을 붙지않고 데몬 처럼 백그라운드 실행이 되게 합니다
  - <kbd>-i, --interactive</kbd> : 인터액티브하게 표준 입출력을 키보드로 동작하게 합니다
  - <kbd>-t, --tty</kbd> : 텍스트 기반의 터미널을 에뮬레이션 하게 합니다
```bash
# docker run <options> <image>:<tag>
docker run --rm --name ubuntu20 -dit ubuntu:20.04
```
```bash
# 터미널에 접속하여 우분투 버전을 확인합니다
docker exec -it ubuntu20 bash
cat /etc/issue
```
<kbd><samp>Ctrl</samp>+<samp>D</samp></kbd> 명령으로 터미널에서 빠져나올 수 있습니다

<br>


#### 2-1-6. `kill` : 컨테이너를 종료합니다
```bash
# docker kill <container_name>
docker kill ubuntu20
```
<br>


### 2-2. 컨테이너 모니터링

#### 2-2-1. `ps` : 실행 중인 컨테이너를 확인합니다
  - <kbd>-a</kbd> : 실행 중이지 않은 컨테이너까지 출력합니다
```bash
docker ps
```

#### 2-2-2. `logs` : 컨테이너 로그를 표준 출력으로 보냅니다

  - <kbd>-f</kbd> : 로그를 지속적으로 tailing 합니다
  - <kbd>-p</kbd> : 호스트 PORT : 게스트 PORT 맵핑
```bash
docker run --rm -p 8888:80 --name nginx -dit nginx
```

```bash
# docker logs <container_name>
docker logs nginx
```

```bash
# terminal
curl localhost:8888
```
> 혹은 `http://localhost:8888` 브라우저로 접속하셔도 됩니다 
<br>


#### 2-2-3. `top` : 컨테이너에 떠 있는 프로세스를 확인합니다

* 실행 확인 후 종료합니다
```bash
# docker top <container_name> <ps options>
docker top nginx
docker rm -f nginx
```
<br>


### 2-3. 컨테이너 상호작용

#### 2-3-1. 실습을 위한 우분투 컨테이너를 기동합니다

```bash
docker run --rm --name ubuntu20 -dit ubuntu:20.04
```

#### 2-3-1. `cp` :  호스트에서 컨테이너로 혹은 반대로 파일을 복사합니다

```bash
# docker cp <container_name>:<path> <host_path> and vice-versa
docker cp ./helloworld.sh ubuntu20:/tmp
```
<br>

#### 2-3-3. `exec` : 컨테이너 내부에 명령을 실행합니다 
```bash
# docker exec <container_name> <args>
docker exec ubuntu20 /tmp/helloworld.sh
```
<br>

#### 2-3-4. 사용한 모든 컨테이너를 종료합니다

* 직접 도커로 실행한 작업은 도커 명령을 이용해 종료합니다
```bash
docker rm -f `docker ps -a | grep -v CONTAINER | awk '{ print $1 }'`
```

<details><summary> :green_book: 1. [필수] `nginx` 컨테이너를 시작 후에 프로세스 확인 및 로그를 출력하고 종료하세요 </summary>

> 출력 결과가 오류가 발생하지 않고, 아래와 유사하다면 성공입니다

```text
docker run --name nginx -dit nginx:latest
docker ps nginx
docker top nginx
docker logs nginx
docker stop nginx
docker rm nginx
```

</details>
<br>

[목차로 돌아가기](#도커-컨테이너-생성-실습)

<br>


### 2-4. 컨테이너 이미지 빌드

> 별도의 Dockerfile 을 생성하고 해당 이미지를 바탕으로 새로운 이미지를 생성할 수 있습니다

#### 2-4-1. `Dockerfile` 생성

* `Ubuntu:18.04` LTS 이미지를 한 번 빌드하기로 합니다
  - <kbd>FROM image:tag</kbd> : 기반이 되는 이미지와 태그를 명시합니다
  - <kbd>MAINTAINER email</kbd> : 컨테이너 이미지 관리자
  - <kbd>COPY path dst</kbd> : 호스트의 `path` 를 게스트의 `dst`로 복사합니다
  - <kbd>ADD src dst</kbd> : COPY 와 동일하지만 추가 기능 (untar archives 기능, http url 지원)이 있습니다
  - <kbd>RUN args</kbd> : 임의의 명령어를 수행합니다
  - <kbd>USER name</kbd> : 기본 이용자를 지정합니다 (ex_ root, ubuntu)
  - <kbd>WORKDIR path</kbd> : 워킹 디렉토리를 지정합니다
  - <kbd>ENTRYPOINT args</kbd> : 메인 프로그램을 지정합니다
  - <kbd>CMD args</kbd> : 메인 프로그램의 파라메터를 지정합니다
  - <kbd>ENV name value</kbd> : 환경변수를 지정합니다

* 아래와 같이 터미널에서 입력하고
```bash
cat > Dockerfile
```

* 아래의 내용을 복사해서 붙여넣은 다음 <kbd><samp>Ctrl</samp>+<samp>C</samp></kbd> 명령으로 나오면 파일이 생성됩니다
```bash
FROM ubuntu:18.04
LABEL maintainer="student@lg.com"

RUN apt-get update && apt-get install -y rsync tree

EXPOSE 22 873
CMD ["/bin/bash"]
```

#### 2-4-2. 이미지 빌드

> 위의 이미지를 통해서 베이스라인 우분투 이미지에 rsync 와 tree 가 설치된 새로운 이미지를 생성할 수 있습니다

* 도커 이미지를 빌드합니다
  - <kbd>-f, --file</kbd> : default 는 현재 경로의 Dockerfile 을 이용하지만 별도로 지정할 수 있습니다
  - <kbd>-t, --tag</kbd> : 도커 이미지의 이름과 태그를 지정합니다
  - <kbd>-q, --quiet</kbd> : 빌드 로그의 출력을 하지 않습니다
  - <kbd>.</kbd> : 현재 경로에서 빌드를 합니다 

```bash
# terminal
docker build -t ubuntu:local .
```

<details><summary> :green_book: 2. [필수] 도커 이미지를 빌드하고 `echo '<본인의 이름 혹은 아이디>'`를 출력해 보세요</summary>

> 출력 결과가 오류가 발생하지 않고, 아래와 유사하다면 성공입니다

```bash
Sending build context to Docker daemon  202.8kB
Step 1/5 : FROM ubuntu:18.04
18.04: Pulling from library/ubuntu
e7ae86ffe2df: Pull complete
Digest: sha256:3b8692dc4474d4f6043fae285676699361792ce1828e22b1b57367b5c05457e3
Status: Downloaded newer image for ubuntu:18.04
...
Step 5/5 : CMD ["/bin/bash"]
 ---> Running in 88f12333612b
Removing intermediate container 88f12333612b
 ---> da9a0e997fc0
Successfully built da9a0e997fc0
Successfully tagged ubuntu:local
```

> 아래와 같이 출력합니다
```bash
docker run --rm -it ubuntu:local echo 'psyoblade or park.suhyuk'
```

</details>
<br>


### 2-5. MySQL 로컬 장비에 설치하지 않고 사용하기

> 호스트 장비에 MySQL 설치하지 않고 사용해보기


#### 2-5-1. [MySQL 데이터베이스 기동](https://hub.docker.com/_/mysql)

* 아래와 같이 도커 명령어를 통해 MySQL 서버를 기동합니다
```bash
docker run --name mysql-volatile \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=testdb \
  -e MYSQL_USER=user \
  -e MYSQL_PASSWORD=pass \
  -d mysql
```
<br>

* 서버가 기동되면 해당 서버로 접속합니다
  - 너무 빨리 접속하면 서버가 기동되기전이라 접속이 실패할 수 있습니다
```bash
sleep 10
docker exec -it mysql-volatile mysql -uuser -ppass
```

#### 2-5-2. 테스트용 테이블을 생성해봅니다

```sql
# mysql>
use testdb;
create table foo (id int, name varchar(300));
insert into foo values (1, 'my name');
select * from foo;
```
> <kbd><samp>Ctrl</samp>+<samp>D</samp></kbd> 명령으로 빠져나옵니다


#### 2-5-3. 컨테이너를 강제로 종료합니다

```bash
docker rm -f mysql-volatile
docker volume ls
```

<details><summary> :blue_book: 3. [선택] mysql-volatile 컨테이너를 다시 생성하고 테이블을 확인해 보세요</summary>

> 테이블이 존재하지 않는다면 정답입니다.

* 볼륨을 마운트하지 않은 상태에서 생성된 MySQL 데이터베이스는 컨테이너의 임시 볼륨 저장소에 저장되므로, 컨테이너가 종료되면 더이상 사용할 수 없습니다
  - 사용한 컨테이너는 삭제 후 다시 생성하여 테이블이 존재하는지 확인합니다
```bash
docker rm -f mysql-volatile
```

</details>
<br>


### 2-6. 볼륨 마운트 통한 MySQL 서버 기동하기

> 이번에는 별도의 볼륨을 생성하여, 컨테이너가 예기치 않게 종료되었다가 다시 생성되더라도, 데이터를 보존할 수 있게 볼륨을 생성합니다. 


#### 2-6-1. 볼륨 마운트 생성

* 아래와 같이 볼륨의 이름만 명시하여 마운트하는 것을 [Volume Mount](https://docs.docker.com/storage/volumes/) 방식이라고 합니다
  - `docker volume ls` 명령을 통해 생성된 볼륨을 확인할 수 있습니다
  - 이렇게 생성된 별도의 격리된 볼륨으로 관리되며 컨테이너가 종료되더라도 유지됩니다
```bash
docker run --name mysql-persist \
  -p 3307:3306 \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=testdb \
  -e MYSQL_USER=user \
  -e MYSQL_PASSWORD=pass \
  -v mysql-volume:/var/lib/mysql \
  -d mysql

sleep 10
docker exec -it mysql-persist mysql --port=3307 -uuser -ppass
```

#### 2-6-2. 볼륨 확인 실습

<details><summary> :blue_book: 4. [선택] mysql-persist 컨테이너를 강제 종료하고, 동일한 설정으로 다시 생성하여 테이블이 존재하는지 확인해 보세요</summary>

> 테이블이 존재하고 데이터가 있다면 정답입니다

```sql
# mysql>
use testdb;
create table foo (id int, name varchar(300));
insert into foo values (1, 'my name');
select * from foo;
```

```bash
# terminal : 컨테이너를 삭제합니다
docker rm -f mysql-persist

# 볼륨이 존재하는지 확인합니다
docker volume ls
```

```bash
# terminal : 새로이 컨테이너를 생성하고 볼륨은 그대로 연결합니다
docker run --name mysql-persist \
  -p 3307:3306 \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=testdb \
  -e MYSQL_USER=user \
  -e MYSQL_PASSWORD=pass \
  -v mysql-volume:/var/lib/mysql \
  -d mysql

sleep 10
docker exec -it mysql-persist mysql --port=3307 -uuser -ppass
```

* 컨테이너와 무관하게 데이터가 존재하는지 확인합니다
```sql
# mysql>
use testdb;
select * from foo;
```

</details>
<br>


#### 2-6-3. 실습이 끝나면, 모든 컨테이너는 *반드시 종료해 주세요*

> 다음 실습 시에 포트가 충돌하거나 컨테이너 이름이 충돌하는 경우 기동에 실패할 수 있습니다
```bash
docker rm -f `docker ps -aq`
```

[목차로 돌아가기](#도커-컨테이너-생성-실습)

<br>



## 3. Docker Compose 명령어 실습

> 도커 컴포즈는 **도커의 명령어들을 반복적으로 수행되지 않도록 yml 파일로 저장해두고 활용**하기 위해 구성되었고, *여러개의 컴포넌트를 동시에 기동하여, 하나의 네트워크에서 동작하도록 구성*한 것이 특징입니다. 내부 서비스들 간에는 컨테이너 이름으로 통신할 수 있어 테스트 환경을 구성하기에 용이합니다. 
<br>

### 실습을 위한 기본 환경을 가져옵니다

```bash
# terminal
cd ~/work
git clone https://github.com/psyoblade/ssm-seoul-data-engineer.git
cd ~/work/ssm-seoul-data-engineer/docker-tutorial
```
<br>


### Docker Compose 기본 명령어

### 3-1. 컨테이너 관리

> 도커 컴포즈는 **컨테이너를 기동하고 작업의 실행, 종료 등의 명령어**를 주로 다룬다는 것을 알 수 있습니다. 아래에 명시한 커맨드 외에도 도커 수준의 명령어들(pull, create, start, stop, rm)이 존재하지만 잘 사용되지 않으며 일부 deprecated 되어 자주 사용하는 명령어 들로만 소개해 드립니다

* [The Compose Specification](https://github.com/psyoblade/compose-spec/blob/master/spec.md)
* [Deployment Support](https://github.com/psyoblade/compose-spec/blob/master/deploy.md)

<br>

#### 3-1-1. up : `docker-compose.yml` 파일을 이용하여 컨테이너를 이미지 다운로드(pull), 생성(create) 및 시작(start) 시킵니다
  - <kbd>-f [filename]</kbd> : 별도 yml 파일을 통해 기동시킵니다 (default: `-f docker-compose.yml`)
  - <kbd>-d, --detach <filename></kbd> : 서비스들을 백그라운드 모드에서 수행합니다
  - <kbd>-e, --env `KEY=VAL`</kbd> : 환경변수를 전달합니다
  - <kbd>--scale [service]=[num]</kbd> : 특정 서비스를 복제하여 기동합니다 (`container_name` 충돌나지 않도록 주의)
```bash
# docker-compose up <options> <services>
docker-compose up -d
```
<br>

#### 3-1-2. down : 컨테이너를 종료 시킵니다
  - <kbd>-t, --timeout [int] <filename></kbd> : 셧다운 타임아웃을 지정하여 무한정 대기(SIGTERM)하지 않고 종료(SIGKILL)합니다 (default: 10초)
```bash
# docker-compose down <options> <services>
docker-compose down
```
<br>


<details><summary> :blue_book: 5. [선택] 컴포즈 명령어(--scale)를 이용하여 우분투 컨테이너를 3개 띄워보세요  </summary>

> 아래와 유사하게 작성 및 실행했다면 정답입니다


* `docker-compose` 명령어 예제입니다 
  - 컴포즈 파일에 `container_name` 이 있으면 이름이 충돌나기 때문에 별도의 파일을 만듭니다
```bash
# terminal
cat docker-compose.yml | grep -v 'container_name: ubuntu' > ubuntu-no-container-name.yml
docker-compose -f ubuntu-no-container-name.yml up --scale ubuntu=2 -d ubuntu
docker-compose -f ubuntu-no-container-name.yml down
```

* `ubuntu-scale.yml` 예제입니다 
  - 우분투 이미지를 기본 이미지를 사용해도 무관합니다
```bash
# cat > ubuntu-replicated.yml
version: "3"

services:
  ubuntu:
    image: psyoblade/data-engineer-ubuntu:20.04
    restart: always
    tty: true
    deploy:
      mode: replicated
      replicas: 3
      endpoint_mode: vip

networks:
  default:
```

* 아래와 같이 기동/종료 합니다
```bash
docker-compose -f ubuntu-replicated.yml up -d ubuntu
docker-compose -f ubuntu-replicated.yml down
```

</details>
<br>


### 3-2. 기타 자주 사용되는 명령어

#### 3-2-1. exec : 컨테이너에 커맨드를 실행합니다
  - <kbd>-d, --detach</kbd> : 백그라운드 모드에서 실행합니다
  - <kbd>-e, --env `KEY=VAL`</kbd> : 환경변수를 전달합니다
  - <kbd>-u, --user [string]</kbd> : 이용자를 지정합니다
  - <kbd>-w, --workdir [string]</kbd> : 워킹 디렉토리를 지정합니다
```bash
# docker-compose exec [options] [-e KEY=VAL...] [--] SERVICE COMMAND [ARGS...]
docker-compose up -d
docker-compose exec ubuntu echo hello world
```
<br>

#### 3-2-2. logs : 컨테이너의 로그를 출력합니다
  - <kbd>-f, --follow</kbd> : 출력로그를 이어서 tailing 합니다
```bash
# terminal
docker-compose logs -f ubuntu
```
<br>

#### 3-2-3. pull : 컨테이너의 모든 이미지를 다운로드 받습니다
  - <kbd>-q, --quiet</kbd> : 다운로드 메시지를 출력하지 않습니다 
```bash
# terminal
docker-compose pull
```
<br>

#### 3-2-4. ps : 컨테이너 들의 상태를 확인합니다
  - <kbd>-a, --all</kbd> : 모든 서비스의 프로세스를 확인합니다
```bash
# terminal
docker-compose ps -a
```
<br>

#### 3-2-5. top : 컨테이너 내부에 실행되고 있는 프로세스를 출력합니다
```bash
# docker-compose top <services>
docker-compose top mysql
docker-compose top ubuntu
```

#### 3-2-6. 사용한 모든 컨테이너를 종료합니다

* 컴포즈를 통해 실행한 작업은 컴포즈를 이용해 종료합니다
```bash
docker-compose down
```

[목차로 돌아가기](#도커-컨테이너-생성-실습)

<br>


### 3-3. 도커 컴포즈 통한 여러 이미지 실행

> MySQL 과 phpMyAdmin 2가지 서비스를 기동하는 컴포즈를 생성합니다

* 기존의 컨테이너를 중단시킵니다 (삭제가 아닙니다)
```bash
# terminal
docker-compose down
```

#### 3-3-1. 도커 컴포즈를 통해 phpMyAdmin 추가 설치

* 기존의 컨테이너를 삭제하게 되면 볼륨 마운트가 없기 때문에 데이터를 확인할 수 없는 점 유의하시기 바랍니다
  - 아래는 기존의  `docker-compose.yml` 파일을 덮어쓰되 phpMyAdmin 만 추가합니다
```bash
# cat > docker-compose.yml
version: "3"

services:
  mysql:
    image: psyoblade/data-engineer-mysql:1.1
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: testdb
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
    volumes:
      - ./custom:/etc/mysql/conf.d
  php:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin
    links:
      - mysql
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      PMA_ARBITRARY: 1
    restart: always
    ports:
      - 80:80
```

```bash
# 변경한 컴포즈를 실행합니다
docker-compose up -d
```

> phpMyAdmin(http://`vm<number>.aiffelbiz.co.kr`) 사이트에 접속하여 서버: `mysql`, 사용자명: `user`, 암호: `pass` 로 접속합니다

<br>


### 3-4. MySQL 이 정상적으로 로딩된 이후에 접속하도록 설정합니다

> 테스트 헬스체크를 통해 MySQL 이 정상 기동되었을 때에 다른 어플리케이션을 띄웁니다

* 기존의 컨테이너를 중단시킵니다 (삭제가 아닙니다)
```bash
# terminal
docker-compose down
```

#### 3-4-1. MySQL 기동이 완료되기를 기다리는 컴포즈 파일을 구성합니다

* 아래와 같이 구성된 컴포즈 파일을 생성합니다
```bash
# cat docker-compose.yml
version: "3"

services:
  mysql:
    image: local/mysql:5.7
    container_name: mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: testdb
      MYSQL_USER: user
      MYSQL_PASSWORD: pass
    healthcheck:
      test: ["CMD", "mysqladmin" ,"ping", "-h", "localhost"]
      interval: 3s
      timeout: 1s
      retries: 3
    volumes:
      - ./custom:/etc/mysql/conf.d
  php:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin
    depends_on:
      - mysql
    links:
      - mysql
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      PMA_ARBITRARY: 1
    restart: always
    ports:
      - 80:80
```

#### 3-4-2. 실습이 완료되었으므로 모든 컨테이너를 종료합니다
```bash
# terminal
docker-compose down
```

[목차로 돌아가기](#도커-컨테이너-생성-실습)

<br>


## 4. 참고 자료
* [Docker Cheatsheet](https://dockerlabs.collabnix.com/docker/cheatsheet/)
* [Docker Compose Cheatsheet](https://devhints.io/docker-compose)
* [Compose Cheatsheet](https://buildvirtual.net/docker-compose-cheat-sheet/)
* [Install Docker Desktop on Windows](https://docs.docker.com/desktop/windows/install/)
> ***Docker Desktop 약관 업데이트*** : 대규모 조직(직원 250명 이상 또는 연간 매출 1천만 달러 이상)에서 Docker Desktop을 전문적으로 사용하려면 유료 Docker 구독이 필요합니다. 본 약관의 발효일은 2021년 8월 31일이지만 유료 구독이 필요한 경우 2022년 1월 31일까지 유예 기간이 있습니다. 자세한 내용은 블로그 Docker is Updating and Extending Our Product Subscriptions 및 Docker Desktop License Agreement를 참조하십시오.
* [Docker is Updating and Extending Our Product Subscariptions](https://www.docker.com/blog/updating-product-subscriptions/)

