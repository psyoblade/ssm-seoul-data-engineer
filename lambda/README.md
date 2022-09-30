# 람다 아키텍처 실습
> `NoSQL`에 해당하는 엔진들은 `Update`에 취약(`Delete & Insert`)하기 때문에 태생적으로 Lambda 아키텍처 구조와 적합한 구조라고 말할 수 있습니다. 물론, 일반적인 동적인 서빙레이어(`MySQL, MongoDB`)를 사용하는것도 가능하지만, 일괄 색인 시에 엔진에 부하가 있을 수 있으므로 주의가 필요합니다


## I. 람다 아키텍처 스피드 레이어를 통한 Join & Aggregation 실습
> 람다 아키텍처를 통한 비즈니스 로직은 동일하지만 집계를 통한 스트리밍 지표 적재
* speed layer : fluentd -> kafka -> spark-stream-agg -> druid-kafka-index -> druid -> druid-console 

### 1. `fluentd` 통한 예제 데이터 카프카로 저장
> `fluentd` 에이전트에 대한 설명 및 실습 애플리케이션에 대한 설명

### 2. `kafka` 에 저장된 `events` 토픽을 `spark streaming` 집계 콘솔 출력
> `kafka` 클러스터에 대한 설명 및 콘솔 도구에 대한 설명
> `spark streaming` 애플리케이션에 대한 코드 설명 및 동작방식 설명

### 3. `stream` 데이터를 집계하지 않고 `enrich` 만 하고 `kafka` 싱크로 지정하여 적재
> 싱크를 변경하여 적재하도록 변경

### 4. `druid-kafka-index` 통해 `druid` 엔진에 저장합니다
> `druid-console` 통해서 적재된 지표에 대한 조회하되, 여기서 지표집계 및 모든 분석이 가능합니다


## II. 람다 아키텍처 배치 레이어를 통한 변환 적재 실습
> 실시간으로 적재된 데이터를 배치 레이어를 통해 전체 데이터를 중복제거, 지연로그를 포함하여 일괄 처리  
* batch layer : fluentd -> hdfs -> spark-batch-agg -> druid-hadoop-index -> druid -> druid-console

### 1. `fluentd` 통한 예제 데이터 하둡에 저장
> `fluentd` 하둡 저장 플러그인에 대한 설명 및 실습 애플리케이션에 대한 설명

### 2. `spark batch` 처리를 통해 데이터 정재 및 가공처리 후 콘솔 출력
> `spark batch` 애플리케이션에 대한 코드 설명 및 동작방식 설명

### 3. `batch` 데이터를 적재하고 `kafka-hadoop-index` 통하여 배치색인
> 적재된 데이터를 `kafka-hadoop-index` 작업을 통해서 일괄 색인

### 4. `druid-kafka-index` 통해 `druid` 엔진에 저장합니다
> `druid-console` 통해서 적재된 지표에 대한 조회

## III. 아키텍처 개선 및 확장 

### 1. `fluentd` 통하여 멀티 싱크를 통하여 파이프라인 간소화 
> 복사 플러그인을 통한 멀티 싱크 예제 설명

### 2. `airflow` 같은 파이프라인 서비스를 통해 의존성 및 스케줄링 통합
> 에어플로우 스케줄링 및 의존성 예제 설명

