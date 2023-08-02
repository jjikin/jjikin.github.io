---
title: ELB AccessLog Partitioning 개선하기(2)
date: 2023-01-07 00:00:00 +09:00
categories: [AWS, ELB]
tags: [aws, elb, alb, accesslog, athena, partitoning, glue]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---

이전 포스트 [LoadBalancer AccessLog Partitioning 개선하기 (1)](https://jjikin.com/posts/ELB-AccessLog-Partitioning-개선하기(1))에서 이어지는 포스트입니다.

<br>

## 사용 간 겪었던 문제 (2)

1. **파티션 갱신 문제**

   기존 방식은 자동, 수동 매핑 상관없이 분석 전 `ALTER TABLE`  또는 `MSCK REPAIR TABLE` 쿼리 사용이 반드시 선행되어야 했습니다.

   ```sql
   -- Hive 스타일 형식(...year=2022/month=07/day=28)으로 파티셔닝 된 경우 자동 매핑 사용
   MSCK REPAIR TABLE alb_logs
   
   -- 수동 매핑 
   ALTER TABLE alb_logs
   ADD PARTITION (year='2022', month='07', day='28') 
   LOCATION 's3://...'
   ```

     

   AWS Athena는 기본적으로 Apache Hive를 사용하여 테이블을 정의하고 데이터베이스를 생성합니다. 

   Hive는 SQL과 유사한 방식으로 데이터 요약, 질의 및 분석 기능을 제공하는 **Hadoop** 기반의 데이터 웨어하우스 솔루션으로, 일반적인 관계형 데이터베이스처럼 미리 스키마를 정의하고 이에 맞게 데이터를 `Insert` 하는 것이 아니라 데이터를 먼저 저장하고 정의한 스키마 정보를 메타스토어에서 참조하여 가져온 후 그 데이터에 입히는 방식을 사용합니다.

     {: .prompt-info }

     > Hadoop : 분산된 환경에서 빅데이터를 저장하고 처리할 수 있는 자바 기반의 오픈 소스 프레임워크   

   다시 말해 스키마 정의와 데이터가 함께 저장되지 않으므로 S3 버킷에 저장된 데이터를 수정하지 않으며, Hive를 사용하여 정의된 테이블은 이 데이터를 가리키는 논리적인 개념입니다.  
   따라서 파티션을 추가한 후에는 메타데이터에 추가된 파티션과 S3 버킷 내 데이터와 비교한 후 갱신하는 과정인 `MSCK REPAIR TABLE` 쿼리 실행이 필요합니다. 그렇지 않으면 추가 전 메타스토어에 있는 파티션에 해당하는 파일만 쿼리합니다.  
   만약 Hive 스타일 형식(...year=2022/month=07/day=28)으로 파티셔닝 되어있지 않은 경우에는 `ALTER TABLE ADD PARTITIONS` 쿼리로 파티션 형식에 맞는 LOCATION을 수동으로 지정해줘야 합니다.

      

   이를 해결하기 위해 이전 포스트에서 아래와 같이 파티션을 자동 생성하도록 람다를 구성했지만, 미처 파악하지 못해 발생한 문제가 있었습니다.

   KST 기준 매일 00:05분에 람다가 실행되지만 ALB 엑세스 로그는 UTC 기준으로 저장되므로(전일 15:00) 이 일자를 맞춰주기 위해 아래 코드와 같이 1일을 더하여 생성하도록 작성했었습니다.

   ```python
   utc_now = datetime.utcnow()
   ...
   date_added = timedelta(days=1)
   day = (utc_now + date_added).strftime('%d')
   ...
   ```

   하지만 2023/01/01 일자의 파티션이 생성될 때 년도를 고려하지 않아 **2022**/01/01 일자의 파티션으로 덮어쓰는 문제가 발생했습니다.

   

   <br>

2. **생성된 파티션에 대한 관리 필요**

   AWS Athena에서는 일반적으로 쿼리를 처리할 때 내부적으로 AWS Glue Data Catalog에 대한 `GetPartitions` 호출을 수행합니다. 이 때 테이블에 많은 수의 파티션이 있는 경우 쿼리 성능에 부정적인 영향을 줄 수 있습니다. 

   또한 `MSCK REPAIR TABLE` 쿼리의 경우 대규모의 파티션이 생성된 상황에서 사용하는 경우 시간 초과 문제로 인해 정상적으로 업데이트가 이루어지지 않는 문제가 발생할 수 있습니다.
   
   ![Untitled](/assets/img/posts/image-20230107153547770.png)
   _SHOW PARTITIONS alb_logs; 쿼리 결과_

<br>

## 개선 사항 도출 및 구성

- **Partition Projection을 통한 파티션 갱신 자동화**

  프로젝션 파티션이란 S3에 저장된 객체 Key에 Placeholder(자리 표시자)를 사전에 구성해 테이블의 파티션을 생성하는 기능입니다.  
  다시 말해 Athena에서 쿼리를 사용할 때 사전에 추가된 파티션 metadata에 맞게 S3 파일을 찾아 읽는 과정 없이 다이렉트로 객체 Key의 Placeholder(자리표시자) 부분에 해당하는 조건절을 구문 분석하는 방식을 사용합니다.  
  따라서 파티션 프로젝션을 사용할 경우 위의 1번 문제처럼 파티션을 갱신할 필요가 없습니다. 또한 파티션 프로젝션 구성은 파티션 자체를 빌드하는 데 필요한 모든 정보를 Athena에 제공하므로 위의 2번 문제처럼 `GetPartitions`를 호출할 필요가 없어 쿼리 성능에 영향을 미치지 않습니다.

  요약하자면, 고도로 분할된 필요가 있는 데이터에 Partition Projection을 적용하면 테이블의 쿼리 처리 속도를 향상시키고 파티션 관리를 자동화할 수 있습니다.

  하지만 어떻게 사용하느냐에 따라 쿼리 성능이 떨어지거나 제약 사항이 발생할 수 있으므로 적용 전 [AWS 문서](https://docs.aws.amazon.com/ko_kr/athena/latest/ug/partition-projection.html)를 꼭 확인해야 합니다.

  <br>

- **Partition Projection 구성**

  ```sql
  CREATE EXTERNAL TABLE alb_logs (
              type string,
              time_utc string,
              elb string,
              ...
              )
              PARTITIONED BY
              (
               account_id STRING, region STRING, alb_name STRING, day STRING, hour STRING
              )
              ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
              WITH SERDEPROPERTIES (
              'serialization.format' = '1',
              'input.regex' = 
          '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) (.*) (- |[^ ]*)\" \"([^\"]*)\" ([A-Z0-9-_]+) ([A-Za-z0-9.-]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^\"]*)\" ([-.0-9]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^ ]*)\" \"([^\s]+?)\" \"([^\s]+)\" \"([^ ]*)\" \"([^ ]*)\"')
              LOCATION 's3://pub-alb-accesslog/AWSLogs/'
              TBLPROPERTIES (
                   "projection.enabled" = "true",
                   "projection.account_id.type"="integer",
                   "projection.account_id.range"="000000000001,999999999999",
                   "projection.region.type"="enum",
                   "projection.region.values"="af-south-1,ap-east-1,ap-northeast-1,ap-northeast-2,ap-northeast-3,ap-south-1,ap-southeast-1,ap-southeast-2,ap-southeast-3,ca-central-1,eu-central-1,eu-north-1,eu-south-1,eu-west-1,eu-west-2,eu-west-3,me-central-1,me-south-1,sa-east-1,us-east-1,us-east-2,us-gov-east-1,us-gov-west-1,us-west-1,us-west-2",
                   "projection.alb_name.type"="injected",
                   "projection.day.type" = "date",
                   "projection.day.range" = "NOW-3YEARS,NOW+9HOURS",
                   "projection.day.format" = "yyyy/MM/dd",
                   "projection.day.interval" = "1",
                   "projection.day.interval.unit" = "DAYS",
                   "projection.hour.type" = "integer",
                   "projection.hour.range" = "0,23",
                   "projection.hour.digits" = "2",
                   "storage.location.template" = "s3://0106-web-accesslog/AWSLogs/${account_id}/elasticloadbalancing/${region}/${alb_name}/${day}/${hour}"
              )
  ```

  - 테이블 정의는 아래 AWS 문서에서 제공하는 ALB 로그 테이블 생성 쿼리와 동일합니다.
    [https://docs.aws.amazon.com/ko_kr/athena/latest/ug/application-load-balancer-logs.html](https://docs.aws.amazon.com/ko_kr/athena/latest/ug/application-load-balancer-logs.html)
    단, 로그가 기록되는 시간 자체는 UTC이며 KST로 후처리가 사실상 어려우므로, 혼동을 막기위해 time 값을 time_utc로 변경하여 표기하였습니다.
  - LOCATION의 경우 파티션 프로젝션이 시작하기 전 Prefix까지 입력합니다.
  - TBLPROPERTIES
    - 테이블 속성의 경우 아래 AWS 문서를 참고하여 정의하였습니다.
      [https://docs.aws.amazon.com/athena/latest/ug/partition-projection-supported-types.html](https://docs.aws.amazon.com/athena/latest/ug/partition-projection-supported-types.html)
    - alb_name : 로드밸런서 이름의 경우 매번 달라지므로 열거형 형식(enum)을 사용할 수 없습니다. 따라서 삽입 형식(injected)을 사용하여 쿼리에서 alb_name을 입력합니다.
    - projection.day.range : 이전 포스트에서 객체 이름 및 분석 기준을 모두 KST로 변환하였으므로, 파티션 프로젝션 또한 날짜 범위 값에 9시간을 추가하도록 설정합니다.
    - storage.location.template : 파티셔닝할 로그가 Hive 스타일 형식으로 저장되지 않는 경우 사용자 지정 Amazon S3 파일 위치에 파티션 값을 매핑하기 위해 사용합니다.
  - 테이블 생성 및 속성 정의는 AWS Glue 서비스에서도 적용 가능합니다.

<br>

## 구성 간 발생했던 문제와 해결 방법

- 파티션 프로젝션이 지원하는 삽입 형식(injected) 사용
  테이블 생성 시 정의한 TBLPROPERTIES 내 injected 타입을 가지는 Column은 쿼리 실행 시 WHERE 절에 조건식이 제공되지 않으면 쿼리에 실패합니다.

  ![Untitled](/assets/img/posts/image-20230107153547771.png)
  
  
  
  따라서 아래와 같이 반드시 alb_name를 명시해야합니다.

![Untitled](/assets/img/posts/image-20230107153547772.png)
