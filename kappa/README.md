# 카파 아키텍처 실습

>  벌크 색인 대신에 모든 데이터를 스트리밍 형태로 유지하고, 스트림 파이프라인을 데이터 저장소로 인지하고 이를 통해 기본적인 데이터 처리 및 재처리를 전재하고 있기 때문에 서빙 레이어에서 `MySQL 혹은 MongoDB`같은 동적 업데이트를 지원하는 엔진이 적합합니다. 혹은 스트림 소스를 그대로 노출하고 컨슈머를 통해 도메인 별로 애플리케이션을 구성하는 `Data Mesh` 아키텍처로 접근하기도 합니다

## 1. 최신버전 업데이트

> 원격 터미널에 접속하여 관련 코드를 최신 버전으로 내려받고, 과거에 실행된 컨테이너가 없는지 확인하고 종료합니다  

### 1.1 최신 소스를 내려 받습니다

> 자주 사용하는 명령어는 `alias` 를 걸어둡니다  

```bash
# terminal
cd ~/work/ssm-seoul-data-engineer
git pull

# alias
alias d="docker-compose"
```

### 1.2 이전에 기동된 컨테이너가 있다면 강제 종료합니다

```bash
# terminal 
docker rm -f `docker ps -aq`
`docker ps -a` 명령으로 결과가 없다면 모든 컨테이너가 종료되었다고 보시면 됩니다
```

### 1.3 실습을 위한 이미지를 내려받고 컨테이너를 기동합니다

```bash
# 개별 컨테이너 기동 : docker-compose.yml 파일의 서비스 이름을 명시
# docker-compose up -d <service-name>

# 전체 컨테이너 기동 : 서비스 이름을 명시하지 않으면 모든 컨테이너 기동
cd ~/work/ssm-seoul-data-engineer/kappa
docker-compose pull
docker-compose up -d

# 기동된 컨테이너 확인
docker-compose ps
```



## 2. 스트리밍 데이터 스테이징 토픽 생성

>  카파 아키텍처의 기본 스트리밍 파이프라인을 구성하여, 람다 아키텍처의 원본 테이블의 역할을 수행하는 스테이징 토픽을 생성합니다. 이러한 스테이징 소스를 생성 시에는 정보가 유실되지 않도록 유의해야 하며, 누구나 사용할 수 있는 데이터 소스를 가정하여 작성해야 하며, 생성된 토픽의 스키마 정보 (컬럼 이름, 데이터 타입 등)도 문서화 되어야 합니다

![processing-layer](images/processing-layer.png)

### 2.1 `fluentd` 통한 더미 데이터 카프카로 저장

>  람다 실습과 동일하게 `dummy` 에이전트를 통해 `{id:number, time:timestamp}` 형식의 예제 데이터 생성 및 메시지 확인 과정을 수행합니다

#### 1. `fluentd` 컨테이너 기동 및 접속

```bash
# terminal
docker-compose up -d fluentd
docker-compose exec fluentd bash
```

> 편의상 개별 컨테이너를 하나씩 기동하면서 실습하는 예제로 작성 되었습니다

#### 2.  `match` 지시자를 이용하여 `kafka` 싱크 설정

```xml
# cat > /fluentd/config/kappa-v3.conf
# https://docs.fluentd.org/v/0.12/input/dummy
<source>
  @type dummy
  tag info
  auto_increment_key id
  dummy {"hello":"ssm-seoul"}
</source>

# https://docs.fluentd.org/v/0.12/filter/record_transformer
<filter info>
    @type record_transformer
    enable_ruby
    <record>
        time ${Time.at(time).strftime('%Y-%m-%d %H:%M:%S')}
    </record>
</filter>

# https://docs.fluentd.org/v/0.12/output/stdout
<match debug>
  @type stdout
</match>

# https://docs.fluentd.org/v/0.12/output/kafka
<match info>
  @type kafka_buffered

  # list of seed brokers
  brokers kafka:9093

  # buffer settings
  buffer_type file
  buffer_path /var/log/td-agent/buffer/tmp
  flush_interval 3s

  # topic settings
  default_topic events

  # data type settings
  output_data_type json
  compression_codec gzip

  # producer settings
  max_send_retries 1
  required_acks -1
</match>
```

> 위와 같이 설정 파일을 생성하고, 카프카 기동을 위해 새로운 터미널을 하나 더 열어 접속합니다

#### 3. 카프카 컨테이너 기동 및 메시지 전송

> 카프카로 메시지를 전송하기 전에 **별도의 터미널**에서 카프카 컨테이너를 기동합니다

```bash
# terminal
docker-compose up -d kafka
```

> 다시 `fluentd` 터미널에서 카프카로 메시지 전송을 수행합니다

```bash
# docker: fluentd
fluentd -c /fluentd/config/kappa-v3.conf
```

#### 4. 카프카 토픽 메시지 확인

> 이번에는 `kafka` 컨테이너에 접속하여 카프카 메시지를 직접 확인합니다

```bash
# terminal
docker-compose exec kafka bash
```

> 컨테이너 내부에서 토픽을 확인하고, `events` 토픽의 메시지를 확인합니다

```bash
# docker: kafka
cd /opt/kafka

# 토픽 목록 출력
boot="--bootstrap-server localhost:9093"
bin/kafka-topics.sh $boot --list

# 토픽 메시지 확인
# 옵션 : --from-beginning --max-messages <num>
bin/kafka-console-consumer.sh $boot --topic events
```

> 첫 번째 단계인 `fluentd` 에이전트를 통해 메시지를 `kafka` 클러스터의 `events` 토픽으로 전송이 완료 되었습니다



### 2.2  `events` 토픽을 읽고 `events_stages` 토픽에 저장

>      `Keynote: 데이터 레이크 2일차 6교시 - 아파치 스파크 스트리밍` 문서를 통해 다시 한 번 리마인드 하시고, `notebook` 컨테이너를 기동하여 아래의 조건에 맞게 가공 및 데이터 보정 이후에 `events_stages` 토픽으로 메시지를 다시 포워드 합니다.

#### 변환 규칙

>  스트리밍 데이터 토픽의 경우에 스트리밍 데이터가 데이터의 소스가 되기 때문에 스트리밍 애플리케이션 내에서 기본적인 검증, 데이터 보정 및 데이터를 보완(enrich) 하는 작업을 모두 수행할 필요가 있습니다. 아래의 예제는 가상의 데이터를 구성하기 위해서 생성한 임의의 규칙임을 감안해서 봐주시기 바랍니다. 특히 널값에 대한 정의나, 코드 값에 대한 규칙 혹은 정규화 등은 항상 수행되어져야 하기 때문에 데이터 ETL 작업 가운데 가장 먼저 수행되어져야만 합니다.

| 조건                                                         |  컬럼  |             경로             |
| :----------------------------------------------------------- | :----: | :--------------------------: |
| id 값을 10으로 나눈 나머지를 names.uid 와 조인하여 이름 추가 |  name  | /home/jovyan/work/data/names |
| id 값을 5로 나눈 나머지 값을 dept.uid 와 조인하여 부서명 추가 |  dept  | /home/jovyan/work/data/dept  |
| id 값이 홀수이면 여성 아니면 남성으로 성별 컬럼을 추가       | gender |          여성, 남성          |
| id 값을 100으로 나눈 나머지 + 1 로 계산하여 나이 컬럼을 추가 |  age   |           1 ~ 100            |
| name 컬럼이 null 인 경우 default 값 처리                     |        |        'Unknown Name'        |
| dept 컬럼이 null 인 경우 default 값 처리                     |        |        'Unkown Dept'         |

#### 1. `notebook` 컨테이너 기동 및 접속

```bash
# terminal
docker-compose up -d notebook
docker-compose logs notebook | grep 127
# [JupyterLab] http://127.0.0.1:8888/labtoken=d0ffa88b4ca509687f7a6502e4376f1bbf198462f83c2
```

> 위와 같이 `127.0.0.1` 로 시작하는 전체 주소를 복사하여 크롬 브라우저로 접속합니다. **반드시 labtoken 값을 포함하여** 접속해야 정상적인 접속을 할 수 있습니다.

#### 2. `spark session` 생성 및 `events` 토픽 출력하기

> 스파크 애플리케이션 기동을 위해 스파크 세션을 초기화 합니다. 모든 노트북 명령은 해당 셀을 선택한 상태에서 `Shift+Enter` 명령으로 해당 셀 실행이 가능합니다

```python
# 1.pyspark-stream-stage.ipynb
from pyspark.sql import *
from pyspark.sql.functions import *
from pyspark.sql.types import *
from IPython.display import display, display_pretty, clear_output, JSON

spark = (
    SparkSession
    .builder
    .config("spark.sql.session.timeZone", "Asia/Seoul")
    .getOrCreate()
)

# 노트북에서 테이블 형태로 데이터 프레임 출력을 위한 설정을 합니다
spark.conf.set("spark.sql.repl.eagerEval.enabled", True) # display enabled
spark.conf.set("spark.sql.repl.eagerEval.truncate", 100) # display output columns size

# 공통 데이터 위치
home_jovyan = "/home/jovyan"
work_data = f"{home_jovyan}/work/data"
work_dir=!pwd
work_dir = work_dir[0]

# 로컬 환경 최적화
spark.conf.set("spark.sql.shuffle.partitions", 5) # the number of partitions to use when shuffling data for joins or aggregations.
spark.conf.set("spark.sql.streaming.forceDeleteTempCheckpointLocation", "true")

# 현재 기동된 스파크 애플리케이션의 포트를 확인하기 위해 스파크 정보를 출력합니다
spark
```

>  노트북에서 스파크 스트리밍 상태 및 데이터 조회를 위한 함수 선언

```python
# 스트림 테이블을 주기적으로 조회하는 함수 (name: 이름, sql: Spark SQL, iterations: 반복횟수, sleep_secs: 인터벌)
def displayStream(name, sql, iterations, sleep_secs):
    from time import sleep
    i = 1
    for x in range(iterations):
        clear_output(wait=True)              # 출력 Cell 을 지웁니다
        display('[' + name + '] Iteration: '+str(i)+', Query: '+sql)
        display(spark.sql(sql))              # Spark SQL 을 수행합니다
        sleep(sleep_secs)                    # sleep_secs 초 만큼 대기합니다
        i += 1

# 스트림 쿼리의 상태를 주기적으로 조회하는 함수 (name: 이름, query: Streaming Query, iterations: 반복횟수, sleep_secs: 인터벌)
def displayStatus(name, query, iterations, sleep_secs):
    from time import sleep
    i = 1
    for x in range(iterations):
        clear_output(wait=True)      # Output Cell 의 내용을 지웁니다
        display('[' + name + '] Iteration: '+str(i)+', Status: '+query.status['message'])
        display(query.lastProgress)  # 마지막 수행된 쿼리의 상태를 출력합니다
        sleep(sleep_secs)            # 지정된 시간(초)을 대기합니다
        i += 1
```

> 카프카 토픽 `events` 으로부터 스트림 데이터를 읽어서 노트북에 출력하는 코드를 작성하되, 카프카의 시작 오프셋의 위치는 `latest` 옵션으로 지정하고, 카프카 서버  `kafka:9093` 의 `events` 토픽으로 부터 메시지를 읽어와서 데이터 변환을 수행합니다.

```python
# "kafka:9093" 카프카 서버의 events토픽을 가장 최근위치(latest)에서 읽어오는 리더를 작성합니다
kafkaReader = (
    spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "카프카_주소:카프카_포트")
  .option("subscribe", "카프카_토픽")
  .option("startingOffsets", "카프카_오프셋_위치")
  .load()
)
kafkaReader.printSchema()

# 로그 스키마 : {"hello":"ssm-seoul","id":0,"time":"2022-09-30 16:42:01"}
kafkaSchema = (
    StructType()
    .add(StructField("id", LongType()))
    .add(StructField("hello", StringType()))
    .add(StructField("time", StringType()))
)

# JSON 문서를 읽어오기 위해서 from_json 함수를 사용합니다
# events.id 값은 무한대로 증가하는 숫자이므로 names 테이블의 0~9값과 조인하기 위해 나머지 연산자(%)를 사용합니다
kafkaSelector = (
    kafkaReader
    .select(
        col("key").cast("string"),
        from_json(col("value").cast("string"), kafkaSchema).alias("events")
    )
    .selectExpr(
      "events.id % 10 as user_id"
      , "events.id % 5 as dept_id"
      , "case when events.id % 2 = 0 then '남성' else '여성' end as gender"
      , "(events.id % 100) + 1 as age"
      , "events.*"
    )
    .select("time", "user_id", "dept_id", "gender", "age", "hello", "id")
)
kafkaSelector.printSchema()
```

> 여기까지는 토픽 정보를 읽어와서 변환하는 과정을 구현했고, 이제는 화면으로 출력하여 디버깅을 해 봅니다.

```python
# 콘솔로 출력하기 위해서 memory 기반의 테이블로 생성합니다
queryName = "consoleSink"
kafkaWriter = (
    kafkaSelector
    .select("*")
    .writeStream
    .queryName(queryName)
    .format("memory")
    .outputMode("append")
)

# 언제까지 수행했는지 정보를 저장하기 위한 체크포인트 위치를 지정합니다
checkpointLocation = f"{work_dir}/tmp/{queryName}"
!rm -rf $checkpointLocation

# 얼마나 자주 스트리밍 작업을 수행할 지 인터벌을 지정합니다
kafkaTrigger = (
    kafkaWriter
    .trigger(processingTime="5 second")
    .option("checkpointLocation", checkpointLocation)
)

# 파이썬의 경우 콘솔 디버깅이 노트북 표준출력으로 나오기 때문에, 위에서 선언한 함수로 조회합니다
kafkaQuery = kafkaTrigger.start()
displayStream(queryName, f"select * from {queryName} order by id desc", 10, 6)
kafkaQuery.stop()
```

> 약 1분 정도 결과를 확인하면 자동으로 종료 됩니다.

#### 3.  `names` 테이블과 `outer_join` 하여 콘솔에 출력

> `/home/jovyan/work/data/names` 경로에 저장되어 있는 `names` 테이블을 읽어서 `events_stages` 라는 토픽에 저장합니다. 해당 토픽은 스파크가 자동으로 생성해 주기 때문에 별도로 생성할 필요는 없습니다

```python
# 대상 경로에 저장된 파일을 읽어서 스파크 데이터 프레임으로 생성합니다
namePath = "/home/jovyan/work/data/names"
nameStatic = (
    spark
    .read
    .option("inferSchema", "true")
    .json(namePath)
).withColumnRenamed("name", "user_name")

# 스키마와 데이터를 확인합니다
nameStatic.printSchema()
nameStatic.show(truncate=False)

# 대상 경로에 저장된 파일을 읽어서 스파크 데이터 프레임으로 생성합니다
deptPath = "/home/jovyan/work/data/depts"
deptStatic = (
    spark
    .read
    .option("inferSchema", "true")
    .json(deptPath)
).withColumnRenamed("name", "dept_name")

# 스키마와 데이터를 확인합니다
deptStatic.printSchema()
deptStatic.show(truncate=False)
```

> `events` 테이블과 `names` 테이블 조인을 위한 조건을 생성합니다

```python
# 앞서서 id 값의 나머지를 통해서 0~9 사이 값으로 생성된 user_id 컬럼과 names 테이블의 uid 컬럼으로 조인
nameJoin = (kafkaSelector.user_id == nameStatic.uid)
nameJoined = kafkaSelector.join(nameStatic, nameJoin, "leftOuter")

# 앞서서 id 값의 나머지를 통해서 0~4 사이 값으로 생성된 dept_id 컬럼과 depts 테이블의 did 컬럼으로 조인
deptJoin = (kafkaSelector.dept_id == deptStatic.did)
nameAndDeptJoined = nameJoined.join(deptStatic, deptJoin, "leftOuter")

# 필요한 컬럼만 출력
# events: "time", "user_id", "dept_id", "gender", "age", "hello", "id"
# names : "uid", "user_name"
# depts : "did", "dept_name"
targetSelector = (
  nameAndDeptJoined
  .drop("uid", "did", "dept_id")
  .select("time", "user_id", "user_name", "dept_name", "gender", "age", "hello", "id")
)

# 이름 및 부서명이 널 값인 경우 예외처리
dictFill = {"user_name":"Unknown Name", "dept_name":"Unknown Dept"}
staticSelector = (
  targetSelector.na.fill(dictFill)
)
```

> 조인 조건을 이용하여 위의 콘솔 예제와 마찬가지로 메모리 테이블에 저장하여 조회합니다

```python
# 디버깅을 위해 메모리 테이블에 저장하고, 다시 카프카로 전송을 위해 JSON 포맷으로 변환합니다
queryName = "memorySink"
staticWriter = (
    staticSelector
    .selectExpr("user_id as key", "to_json(struct(*)) as value")
    .writeStream
    .queryName(queryName)
    .format("memory")
    .outputMode("append")
)

# 언제까지 수행했는지 정보를 저장하기 위한 체크포인트 위치를 지정합니다
checkpointLocation = f"{work_dir}/tmp/{queryName}"
!rm -rf $checkpointLocation

# 얼마나 자주 스트리밍 작업을 수행할 지 인터벌을 지정합니다
staticTrigger = (
    staticWriter
    .trigger(processingTime="5 second")
    .option("checkpointLocation", checkpointLocation)
)
```

> 메모리 테이블을 조회하여 결과를 확인합니다

```python
# 트리거를 시작하는 순간 스파크 스트리밍 애플리케이션이 기동되고, 동적 쿼리를 수행할 수 있습니다
staticQuery = staticTrigger.start()
displayStream(queryName, f"select * from {queryName} order by key desc", 10, 6)
staticQuery.explain(True)
staticQuery.stop()
```

> 생성된 쿼리를 이용하여 6초 인터벌로 총 10회 조회합니다

#### 4. 검증 완료된 데이터를 `events_stages` 토픽에 저장

> `stream` 데이터를 집계하지 않고 원본 메시지에서 추가 필드만 `enrich` 하고 `kafka` 싱크로 `events_stages` 토픽을 지정하여 적재합니다. 카프카 브로커 주소는 `kafka:9093` 이며 출력 토픽은 `events_stages` 으로 지정합니다

```python
# 카프카 브로커 주소는 'kafka:9093' 이며 출력 토픽은 'events_stages' 입니다
queryName = "kafkaSink"
staticWriter = (
    staticSelector
    .selectExpr("cast(user_id as string) as key", "to_json(struct(*)) as value")
    .writeStream
    .queryName(queryName)
    .format("kafka")
    .option("kafka.bootstrap.servers", "카프카_주소:카프카_포트")
    .option("topic", "출력_카프카_토픽")
    .outputMode("append")
)

checkpointLocation = f"{work_dir}/tmp/{queryName}"
!rm -rf $checkpointLocation

staticTrigger = (
    staticWriter
    .trigger(processingTime="5 second")
    .option("checkpointLocation", checkpointLocation)
)

staticQuery = staticTrigger.start()
displayStatus(queryName, staticQuery, 600, 10)
staticQuery.stop()
```

#### 5. 생성된 토픽 메시지를 직접 확인

> 앞서 테스트 한대로 `kafka` 컨테이너로 접속하여 정상적으로 메시지가 저장되는 지 확인합니다

```bash
# terminal
docker-compose exec kafka bash
```

```bash
# docker: kafka
cd /opt/kafka

# 토픽 목록 출력
boot="--bootstrap-server localhost:9093"
bin/kafka-topics.sh $boot --list

# 토픽 메시지 확인
# 옵션 : --from-beginning --max-messages <num>
bin/kafka-console-consumer.sh $boot --topic events_stages
```

> 스파크 스트리밍을 통해 `events` 토픽의 `id` 와 `names` 테이블의 `id, id_name` 컬럼을 조인하여 정상적인 조회를 확인합니다. 이 토픽에는 원본 토픽을 보강`enrich`한 상태로 저장하되, 정보의 유실 혹은 분석에 문제가 될 수 있는(널 값 등) 상태로 저장하는 것이 일반적입니다



## 3. 스트리밍 집계 파이프라인 생성

>  `event_stages` 토픽은 24시간 수신되는 스트리밍 로그 토픽이며, 이를 활용하여 1차 가공 데이터를 생성하고 이를 `MySQL` 테이블에 저장하는 예제를 실습합니다

![processing-layer](images/processing-layer.png)

### 3.1 `events_stages` 토픽을 활용하여 집계 지표를 생성합니다

>   특정 도메인에 맞는 비즈니스 로직을 담는 애플리케이션을 개발할 것이며, 기존에 생성했던 `events` 토픽의 데이터 뿐만 아니라 `events_stages` 토픽에 데이터는 계속 생성되고 있어야만 하므로 별도의 스파크 노트북을 생성하여 세션 객체를 생성해야만 합니다

#### 1. 별도의 스파크 애플리케이션을 하나 더 생성합니다

```python
# 2.pyspark-stream-aggregate.ipynb
from pyspark.sql import *
from pyspark.sql.functions import *
from pyspark.sql.types import *
from IPython.display import display, display_pretty, clear_output, JSON

spark = (
    SparkSession
    .builder
    .config("spark.sql.session.timeZone", "Asia/Seoul")
    .getOrCreate()
)

# 노트북에서 테이블 형태로 데이터 프레임 출력을 위한 설정을 합니다
spark.conf.set("spark.sql.repl.eagerEval.enabled", True) # display enabled
spark.conf.set("spark.sql.repl.eagerEval.truncate", 100) # display output columns size

# 공통 데이터 위치
home_jovyan = "/home/jovyan"
work_data = f"{home_jovyan}/work/data"
work_dir=!pwd
work_dir = work_dir[0]

# 로컬 환경 최적화
spark.conf.set("spark.sql.shuffle.partitions", 5) # the number of partitions to use when shuffling data for joins or aggregations.
spark.conf.set("spark.sql.streaming.forceDeleteTempCheckpointLocation", "true")

# 현재 기동된 스파크 애플리케이션의 포트를 확인하기 위해 스파크 정보를 출력합니다
spark
```

>  노트북에서 스파크 스트리밍 상태 및 데이터 조회를 위한 함수 선언

```python
# 스트림 테이블을 주기적으로 조회하는 함수 (name: 이름, sql: Spark SQL, iterations: 반복횟수, sleep_secs: 인터벌)
def displayStream(name, sql, iterations, sleep_secs):
    from time import sleep
    i = 1
    for x in range(iterations):
        clear_output(wait=True)              # 출력 Cell 을 지웁니다
        display('[' + name + '] Iteration: '+str(i)+', Query: '+sql)
        display(spark.sql(sql))              # Spark SQL 을 수행합니다
        sleep(sleep_secs)                    # sleep_secs 초 만큼 대기합니다
        i += 1

# 스트림 쿼리의 상태를 주기적으로 조회하는 함수 (name: 이름, query: Streaming Query, iterations: 반복횟수, sleep_secs: 인터벌)
def displayStatus(name, query, iterations, sleep_secs):
    from time import sleep
    i = 1
    for x in range(iterations):
        clear_output(wait=True)      # Output Cell 의 내용을 지웁니다
        display('[' + name + '] Iteration: '+str(i)+', Status: '+query.status['message'])
        display(query.lastProgress)  # 마지막 수행된 쿼리의 상태를 출력합니다
        sleep(sleep_secs)            # 지정된 시간(초)을 대기합니다
        i += 1
```

#### 2. `events_stages` 토픽을 읽어서 콘솔에 출력합니다

>  카프카로부터 스트림 데이터를 읽어서 노트북에 출력하는 코드를 작성하되, 카프카의 시작 오프셋의 위치는 `latest` (최신 메시지만 수신) 옵션으로 지정하고, 카프카 서버  `kafka:9093` 의 `events_stages` 토픽으로 부터 메시지를 읽어와서 데이터 변환을 수행합니다.

```python
# 조건에 맞는 값을 작성하세요
source_brokers = "카프카_주소:카프카_포트"
source_topic = "카프카_토픽"
source_offset = "카프카_오프셋_위치"

kafkaReader = (
    spark
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", source_brokers)
  .option("subscribe", source_topic)
  .option("startingOffsets", source_offset)
  .load()
)
kafkaReader.printSchema()

# 수신되는 메시지의 스키마를 정의합니다
# { "time", "user_id", "user_name", "dept_name", "gender", "age", "hello", "id" }
kafkaSchema = (
    StructType()
    .add(StructField("time", StringType()))
    .add(StructField("user_id", LongType()))
    .add(StructField("user_name", StringType()))
    .add(StructField("dept_name", StringType()))
    .add(StructField("gender", StringType()))
    .add(StructField("age", LongType()))
    .add(StructField("hello", StringType()))
    .add(StructField("id", LongType()))
)

# 카프카의 메시지를 읽어옵니다
kafkaSelector = (
    kafkaReader
    .select(
        col("key").cast("string"),
        from_json(col("value").cast("string"), kafkaSchema).alias("stages")
    )
    .selectExpr("stages.*")
)
kafkaSelector.printSchema()
```

#### 3. `user_name` 컬럼을 기준으로 빈도를 디버깅하는 코드를 작성합니다

> 여기까지는 토픽 정보를 읽어와서 변환하는 과정을 구현했고, 이제는 화면으로 출력하여 디버깅을 해 봅니다.

```python
# 콘솔로 출력하기 위해서 memory 기반의 테이블로 생성하며 워터마크 적용을 위한 timestamp 컬럼을 생성합니다
query_name = "memory_sink"
memoryWriter = (
    kafkaSelector
    .selectExpr("to_timestamp(time) as timestamp", "user_name")
    .withWatermark("timestamp", "1 minute")
    .groupBy("user_name", window("timestamp", "1 minute", "1 minute")).count().alias("count")
    .writeStream
    .queryName(query_name)
    .format("memory")
    .outputMode("complete")
)

checkpointLocation = f"{work_dir}/tmp/{query_name}"
!rm -rf $checkpointLocation

memoryTrigger = (
    memoryWriter
    .trigger(processingTime="5 second")
    .option("checkpointLocation", checkpointLocation)
)
```

```python
memoryQuery = memoryTrigger.start()
displayStream(query_name, f"select * from {query_name} order by window asc", 10, 6)
# memoryQuery.explain(True)
memoryQuery.stop()
```

> 약 1분 정도 결과를 확인하면 자동으로 종료 됩니다.

#### 4. `MySQL` 에 집계된 데이터를 `events_users_v1` 테이블로 적재합니다

>  `MySQL` 과 같은 관계형 데이터베이스에는 일반적인 방식으로 적재가 어렵기 때문에 `foreachBatch` 함수를 이용하여 적재할 수 있으며, 매 마이크로 배치 수행마다 한 번씩 적재가 이루어집니다.

```python
# foreachBatch 호출 시에 수행될 함수를 구현합니다
sink_table = "events_users_v1"

def saveToMySql(dataFrame, batchId):
    options = "useUnicode=true&serverTimezone=Asia/Seoul"
    url = f"jdbc:mysql://scott:tiger@mysql:3306/default?{options}"
    writer = (
        dataFrame
        .write
        .format("jdbc")
        .option("url", url)
        .option("dbtable", sink_table)
        .mode("append")
    )
    writer.save()
```

>  스트리밍 테이블과 정적인 테이블 조인 후에 `name` 컬럼 기준으로 집계한 결과를 실시간으로 보여주는 `events` 라는 테이블에 적재합니다. 스트리밍 조인 및 집계 처리시에는 반드시 `Watermark` 를 적용하여 지연된 데이터에 대한 처리가 필요합니다. 즉, 무한정 집계를 위해 지연된 데이터를 기다릴 수 없기 때문에 제한된 시간 내에 도착한 데이터만 사용하고 그 이상 지연된 데이터를 사용하지 않겠다는 의미입니다

```python
# 이용자 이름을 기준으로 집계된 결과를 MySQL 의 events_users 테이블에 적재합니다
query_name = "jdbc_sink"
jdbcWriter = (
    kafkaSelector
    .selectExpr("to_timestamp(time) as timestamp", "user_name")
    .withWatermark("timestamp", "15 seconds")
    .groupBy(window("timestamp", "10 seconds", "10 seconds"), "user_name").count().alias("count")
    .select("window.start", "window.end", "user_name", "count")
    .writeStream
    .queryName(query_name)
    .foreachBatch(saveToMySql)
)

checkpointLocation = f"{work_dir}/tmp/{query_name}"
!rm -rf $checkpointLocation

jdbcTrigger = (
    jdbcWriter
    .trigger(processingTime="5 second")
    .option("checkpointLocation", checkpointLocation)
)
```

### 3.2 `MySQL` 에 저장된 데이터를 조회합니다

#### 1. `phpMyAdmin` 웹 페이지로 접속하여 검증합니다

> `phpmyadmin` 페이지를 통해서 실시간으로 조회하여 집계와 적재가 잘 되는지 확인합니다. 이 때에 생성된 테이블이 외부 서비스로 노출되는 실시간 애플리케이션의 하나가 될 수 있습니다.

```python
# 데이터베이스에 조회를 위해 충분히 긴 시간 동안 모니터링을 수행합니다
jdbcQuery = jdbcTrigger.start()
displayStatus(query_name, jdbcQuery, 100, 6)
jdbcQuery.stop()
```

>  웹 페이지에 `http://localhost:80` 페이지에 접속하여 서버: `mysql`, 사용자명: `scott`, 암호: `tiger` 로 `default` 데이터베이스로 접속합니다

### 3.3 기타 `age`, `dept` 등의 기준으로 집계 테이블을 생성합니다

>  실시간으로 추출 및 모니터링 하고 싶은 차원(`age`, `dept` 등)으로 집계한 결과를 개별 `events_ages` 혹은 `events_depts` 테이블로 적재해볼 수 있습니다



## 3. 스트리밍 복구 파이프라인 생성

>    앞서 설명 드린 스트리밍 집계 파이프라인에 버그 혹은 문제가 뒤늦게 발견되어 복구가 필요한 경우에 배치 애플리케이션을 통해 갱신하는 방법도 있지만, 스트리밍 애플리케이션을 그대로 활용하는 방법도 있습니다. 지연된 로그가 발견되어 조금 더 긴 `Watermark`를 적용하거나, 데이터의 크기가 커지는 것을 감안하여 `CPU, Memory` 등의 리소스를 더 설정하는 등의 운영이 가능합니다. 하지만 스파크 스트리밍과 같은 대용량 분산 처리의 프레임워크 자체가 데이터베이스와 같은 `update` 같은 연산 수행을 지원하지 않기 떄문 (결국 1건씩 갱신하는 연산을 하는 것은 성능에 큰 영향을 미치기 때문)에 데이터 처리 작업과 별도로 `delete` 작업 혹은 `truncate` 작업 이후에 복구를 하거나, 별도의 테이블을 생성하고 교체 혹은 같이 조회하는 방식을 취하게 됩니다. 

![recovering-layer](images/recovering-layer.png)

### 3.1 복구를 위한 스파크 애플리케이션을 생성합니다

>  기존에 `2.1`에서 생성했던 애플리케이션은 시나리오 상 하루가 지난 이후에 복구를 하는 것으로 잡았기 때문에 기존의 애플리케이션을 종료하고 해당 애플리케이션을 복사해서 활용하면 됩니다.

#### 1. 기존 노트북을 복사해서 노트북을 생성합니다

> 메뉴에서 `Duplicate` 선택 후 노트북 이름을 `3.pyspark-stream-recovery.ipynb` 로 변경합니다

![image-20221002174528925](images/duplicate-notebook.jpg)

#### 2.  `withWatermark` 시간을 늘리고, `events_users_v2` 테이블로 적재합니다

>   `Watermark` 시간을 조정하고, 출력 대상 테이블을 **반드시** `events_users_v2` 테이블로 변경하여 적재합니다. 이 때에 기존 테이블을 지정하는 경우 `append` 옵션인 경우 중복이 발생하며, `overwrite` 옵션인 경우 모든 데이터를 삭제되기 때문에 유의해야만 합니다.

```python
# 적재 대상 테이블은 반드시 events_v2 와 같이 다른 테이블로 적재되어야만 합니다
sink_table = "events_users_v2"

def saveToMySql(dataFrame, batchId):
    url = "jdbc:mysql://scott:tiger@mysql:3306/default?useUnicode=true&serverTimezone=Asia/Seoul"
    writer = (
        dataFrame
        .write
        .format("jdbc")
        .option("url", url)
        .option("dbtable", sink_table)
        .mode("append")
    )
    writer.save()

# withWatermark 구문의 시간을 조정할 수 있습니다
# 1 minute : 1분까지 지연로그 처리
# 60 minutes : 최대 1시간까지 지연된 로그를 고려하여 처리
watermark_time = "1 minute"

jdbcWriter = (
    kafkaSelector
    .selectExpr("to_timestamp(time) as timestamp", "user_name")
    .withWatermark("timestamp", watermark_time)
    .groupBy(window("timestamp", "1 minute", "1 minute"), "user_name").count().alias("count")
    .select("window.start", "window.end", "user_name", "count")
    .writeStream
    .queryName(query_name)
    .foreachBatch(saveToMySql)
)

checkpointLocation = f"{work_dir}/tmp/{query_name}"
!rm -rf $checkpointLocation

jdbcTrigger = (
    jdbcWriter
    .trigger(processingTime="5 second")
    .option("checkpointLocation", checkpointLocation)
)
```

>  실습에서는 1시간 이상 지연된 데이터 처리에 대한 테스트가 어렵기 때문에 해당 값을 변수로 치환하고 실행 시마다 변경 가능하도록 구현하여 실행하는 것으로 카파 아키텍처에서 동일한 바이너리 수행이 가능하다는 정도로 갈음하겠습니다

#### 3. 해당 일자의 지표가 정상적으로 수정이 되는지 확인합니다

>  `phpmyadmin` 웹 화면을 통해서 적재되는 지 확인할 수 있으며, 이 때에 적재가 정상적으로 `events_users_v1` 테이블과 동일한 상태로 `catch-up` 되었다면 `v1` 테이블을 바라보던 애플리케이션을 단계적으로 `v2` 를 바라보도록 설정을 변경하면 됩니다.

### 3.1 스트리밍 파이프라인의 확장

#### 1 유사한 방식으로 `mongodb` 에 집계 후 적재할 수 있습니다

> `mongo-express` 같은 도구를 통해 확인이 가능합니다

