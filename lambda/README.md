# 람다 아키텍처 실습
> `NoSQL`에 해당하는 엔진들은 `Update`에 취약(`Delete & Insert`)하기 때문에 태생적으로 Lambda 아키텍처 구조와 적합한 구조라고 말할 수 있습니다. 물론, 일반적인 동적인 서빙레이어(`MySQL, MongoDB`)를 사용하는것도 가능하지만, 일괄 색인 시에 엔진에 부하가 있을 수 있으므로 주의가 필요합니다


## I. 람다 아키텍처 스피드 레이어를 통한 Join & Aggregation 실습
> 람다 아키텍처를 통한 비즈니스 로직은 동일하지만 집계를 통한 스트리밍 지표 적재
* speed layer : fluentd -> kafka -> spark-stream-enrich -> druid-kafka-index -> druid -> druid-console 

### 1. `fluentd` 통한 예제 데이터 카프카로 저장
> `fluentd` 에이전트에 대한 설명 및 실습 애플리케이션에 대한 설명 
> 플루언트디 컨테이너 기동 `docker-compose up -d fluentd`
* 더미 에이전트를 통한 로그 생성
* 더미 로그를 표준 출력
* 시간 필드를 추가

> 카프카 컨테이너 기동 `docker-compose up -d kafka`
* 카프카 토픽으로 전송
```properties
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
* 카프카 토픽 확인
```bash
cd /opt/kafka
boot="--bootstrap-server localhost:9093"
bin/kafka-topics.sh $boot --list
bin/kafka-console-consumer.sh $boot --topic events
```

### 2. `kafka` 에 저장된 `events` 토픽을 `spark streaming` 집계 콘솔 출력
> `kafka` 클러스터에 대한 설명 및 콘솔 도구에 대한 설명
> `spark streaming` 애플리케이션에 대한 코드 설명 및 동작방식 설명
> 노트북 컨테이너 기동 `docker-compose up -d notebook`

### 3. `stream` 데이터를 집계하지 않고 `enrich` 만 하고 `kafka` 싱크로 지정하여 적재
> 싱크를 변경하여 적재하도록 변경
> 정적인 테이블 `names` 테이블을 로컬에서 json 파일로 읽어와서 left outer join 통해서 처리합니다
> 생성된 값을 다시 json 형태로 출력해야 합니다

### 4. `druid-kafka-index` 통해 `druid` 엔진에 저장합니다
> 노트북 컨테이너 기동 `docker-compose up -d druid`
> `druid-console` 통해서 적재된 지표에 대한 조회하되, 여기서 지표집계 및 모든 분석이 가능합니다


## II. 람다 아키텍처 배치 레이어를 통한 변환 적재 실습
> 모든 실시간 작업은 종료되었다고 가정하고, 익일이 되어서 이전 일자의 데이터를 일괄 처리하는 예제입니다
> 명확하게 색인 데이터를 삭제하고 전체 색인을 일괄 수행하는 예제를 통해서 배치 레이어 작업을 테스트합니다

> 실시간으로 적재된 데이터를 배치 레이어를 통해 전체 데이터를 중복제거, 지연로그를 포함하여 일괄 처리  
* batch layer : fluentd -> hdfs -> spark-batch-agg -> druid-hadoop-index -> druid -> druid-console

### 1. `fluentd` 통한 예제 데이터 하둡에 저장
> `fluentd` 하둡 저장 플러그인에 대한 설명 및 실습 애플리케이션에 대한 설명
> 로컬 경로에 저장하고 해당 경로가 노트북에서 읽어들일 수 있도록 설정하여 출력확인

### 2. `spark batch` 처리를 통해 데이터 정재 및 가공처리 후 콘솔 출력
> `spark batch` 애플리케이션에 대한 코드 설명 및 동작방식 설명
> 스파크 데이터를 읽고 정적인 데이터도 읽어서 동일한 결과 확인


### 3. `batch` 데이터를 적재하고 `kafka-hadoop-index` 통하여 배치색인
> 적재된 데이터를 `druid-local-index` 작업을 통해서 일괄 색인
> `events_batch` 테이블에 적재합니다

### 4. `druid-kafka-index` 통해 `druid` 엔진에 저장합니다
> `druid-console` 통해서 적재된 지표에 대한 조회


## III. 아키텍처 개선 및 확장 

### 1. `fluentd` 통하여 멀티 싱크를 통하여 파이프라인 간소화 
> 복사 플러그인을 통한 멀티 싱크 예제 설명

### 2. `airflow` 같은 파이프라인 서비스를 통해 의존성 및 스케줄링 통합
> 에어플로우 스케줄링 및 의존성 예제 설명

