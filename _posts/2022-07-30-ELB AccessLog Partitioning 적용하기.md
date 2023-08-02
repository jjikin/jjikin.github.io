---
title: ELB AccessLog Partitioning 적용하기
date: 2022-07-30 00:00:00 +09:00
categories: [AWS, ELB]
tags: [aws, elb, alb, accesslog, athena, partitoning]
image: 
# ------------------------------------------------------------------
# 포스트 작성 시 참고 URL
# https://chirpy.cotes.page/posts/write-a-new-post/
# https://chirpy.cotes.page/posts/text-and-typography/
---



## 구성 배경

고객사 ALB 로그를 분석할 때, elb-log-anlayzer Tool, AWS Athena를 병행하여 사용하고 있었습니다.

- [elb-log-analyzer](https://github.com/ozantunca/elb-log-analyzer)

  간단한 로그 분석에는 elb-log-analyzer를 사용했으며 분석이 필요한 로그를 일괄적으로 다운받기 위해 Cyberduck과 같은 FTP Client을 사용했습니다.

  

- AWS Athena

  좀 더 자세한 로그 분석이 필요한 경우 AWS Athena를 사용했습니다.  
  로드밸런서 로그는 AccessLog 설정을 활성화하고 S3 버킷을 지정하면 자동으로 년월일 단위로 로그가 분리되어 저장됩니다.  
  로그가 저장된 버킷에서 쿼리를 하는 경우 모든 로그를 스캔하기 때문에 많은 비용이 발생하게 되므로, 이를 방지하기 위해 Athena 로그 결과를 저장하는 별도의 버킷을 생성하고 분석이 필요한 로그를 ALB 로그 버킷에서 복사한 후 분석하는 방법을 사용했습니다.
  
  <br>

하지만 고객사로부터 로그 분석 요청이 지속적으로 증가하고 있었고, 매번 로그를 복사한 후 분석하는 방법이 비효율적이므로 ALB 로그를 파티셔닝하기로 결정했습니다.

<br>

## 고려 사항

파티셔닝은 파티션 프로젝션 통한 자동 맵핑 방식과 수동 맵핑 방식이 있습니다.  
ALB 로그의 경우 기본적으로 자동 맵핑 형식에 맞지 않게 S3에 저장되어 파티셔닝된 Athena 테이블을 다시 만들고, 파티션을 수동으로 추가하는 방식을 사용했습니다.

<br>

## 구성 내용

- 파티션된 테이블 생성

  ```sql
  CREATE EXTERNAL TABLE IF NOT EXISTS alb_logs (
              type string,
              time string,
              elb string,
              client_ip string,
              client_port int,
              target_ip string,
              target_port int,
              request_processing_time double,
              target_processing_time double,
              response_processing_time double,
              elb_status_code string,
              target_status_code string,
              received_bytes bigint,
              sent_bytes bigint,
              request_verb string,
              request_url string,
              request_proto string,
              user_agent string,
              ssl_cipher string,
              ssl_protocol string,
              target_group_arn string,
              trace_id string,
              domain_name string,
              chosen_cert_arn string,
              matched_rule_priority string,
              request_creation_time string,
              actions_executed string,
              redirect_url string,
              lambda_error_reason string,
              target_port_list string,
              target_status_code_list string,
              classification string,
              classification_reason string
              )
  						**PARTITIONED BY (`year` string, `month` string, `day` string)**
              ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
              WITH SERDEPROPERTIES (
              'serialization.format' = '1',
              'input.regex' = 
  		        '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) (.*) (- |[^ ]*)\" \"([^\"]*)\" ([A-Z0-9-_]+) ([A-Za-z0-9.-]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^\"]*)\" ([-.0-9]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^ ]*)\" \"([^\s]+?)\" \"([^\s]+)\" \"([^ ]*)\" \"([^ ]*)\"')
              LOCATION 's3://test-alb-access-logs-s3/AWSLogs/111111111111/elasticloadbalancing/ap-northeast-2/'
  ```

- 파티션 추가 및 확인

  ```sql
  ALTER TABLE alb_logs
  ADD PARTITION (year='2022', month='07', day='28') 
  LOCATION 's3://test-alb-access-logs-s3/AWSLogs/111111111111/elasticloadbalancing/ap-northeast-2/2022/07/28/'
  
  SHOW PARTITIONS alb_logs;
  ---
  year=2022/month=07/day=27
  year=2022/month=07/day=28
  ```

<br>

위와 같이 구성한 경우 매일마다 파티션을 별도로 추가하거나 분석 시 파티션을 추가해줘야 하는데, 
아래와 같이 EventBridge + Lambda를 사용해서 매일 00:05분에 파티션을 추가하도록 코드를 작성한 후 설정했습니다.

- EventBridge, Lambda 구성

  - EventBridge

    ![Untitled](/assets/img/posts/image-202207305832817.png)

  - Lambda

    - 제한 시간 : 10초

    - 환경 변수

      ![Untitled 1](/assets/img/posts/image-202207305832818.png)

    - 권한

      ```json
      {
          "Version": "2012-10-17",
          "Statement": [
              {
                  "Effect": "Allow",
                  "Action": [
                      "athena:*"
                  ],
                  "Resource": [
                      "*"
                  ]
              },
              {
                  "Effect": "Allow",
                  "Action": [
                      "glue:GetDatabase",
                      "glue:GetDatabases",
                      "glue:UpdateDatabase",
                      "glue:UpdateTable",
                      "glue:GetTable",
                      "glue:GetTables",
                      "glue:BatchCreatePartition",
                      "glue:BatchGetPartition",
                      "glue:CreatePartition",
                      "glue:UpdatePartition",
                      "glue:GetPartition",
                      "glue:GetPartitions"
                  ],
                  "Resource": [
                      "*"
                  ]
              },
              {
                  "Effect": "Allow",
                  "Action": [
                      "s3:GetObject",
                      "s3:GetBucketLocation",
                      "s3:ListBucket",
                      "s3:PutObject"
                  ],
                  "Resource": [
                      "arn:aws:s3:::test-alb-access-logs-s3",
                      "arn:aws:s3:::test-alb-access-logs-s3/*"
                  ]
              },
      				{
                  "Effect": "Allow",
                  "Action": [
                      "s3:PutObject"
                  ],
                  "Resource": [
                      "arn:aws:s3:::test-alb-access-logs-s3/Athena/Result",
                      "arn:aws:s3:::test-alb-access-logs-s3/Athena/Result/*"
                  ]
              }
          ]
      }
      ```

    - 코드

      ```python
      import boto3
      import time
      import os
      from datetime import timedelta
      from datetime import datetime
      
      def lambda_handler(event, context):
          database = os.environ['DATABASE']
          table = os.environ['TABLE']
          log_path = os.environ['LOG_PATH']
          result_path = os.environ['RESULT_PATH']
          
          utc_now = datetime.utcnow()
          year = utc_now.strftime('%Y')
          month = utc_now.strftime('%m')
          date_added = timedelta(days=1)
          day = (utc_now + date_added).strftime('%d')
          
          query = (
              f"ALTER TABLE {database}.{table} "
              f"ADD IF NOT EXISTS PARTITION (year={year},month={month},day={day}) "
              f"LOCATION '{log_path}{year}/{month}/{day}/';"   #log_path 값 마지막에 /가 포함된 경우  utc_now.year 사이 / 중복 체크 필요.
          )
          print(query)
          client = boto3.client('athena')
          try:
              response = client.start_query_execution(
                  QueryString = query,
                  QueryExecutionContext = {
                      'Database': database
                  },
                  ResultConfiguration = {
                      'OutputLocation': result_path
                  }
              )
          except Exception as e:
              print(e)
      ```

<br>

## 테스트

- ALB 로그 분석용 쿼리 예시

  ```sql
  SELECT * FROM alb_logs
  WHERE (
  		year LIKE '2022'
  		AND month LIKE '07'
  		AND day LIKE '28'
  	  )
  	AND elb LIKE 'app/test-alb-access-logs-s3/51219687d121212e'
  	AND (
  		parse_datetime(time, 'yyyy-MM-dd''T''HH:mm:ss.SSSSSS''Z') 
  		BETWEEN parse_datetime('2022-07-28-00:30:00', 'yyyy-MM-dd-HH:mm:ss')
  		AND parse_datetime('2022-07-28-00:45:00', 'yyyy-MM-dd-HH:mm:ss')
  		)
      AND elb_status_code LIKE '5%';
  ```

- 쿼리 결과
  ![Untitled 2](/assets/img/posts/image-202207305832819.png)
