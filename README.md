# AWS Config

AWS 리소스 구성을 

## Basic Configuration

1. AWS Management Console 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 Config을 검색하거나 Management & Governance 밑에 있는 **[Config]** 를 선택 후 **[Get started]**

2. **Resource types to record** = :white_check_mark: Record all resources supported in this region, :white_check_mark: Include global resources (e.g., AWS IAM resources),\
**Amazon S3 bucket** = :radio_button: Create a bucket,\
**AWS Config role** = :radio_button: Use an existing AWS Config service-linked role\
&rightarrow; **[Next]**

3. *[Filter by rule name, label or description]* 에 IAM을 입력하고 **iam-user-mfa-enabled** 를 선택하고 **[Next]** &rightarrow; **[Confirm]**

    > [Trigger types](https://docs.aws.amazon.com/config/latest/developerguide/evaluate-config-rules.html#aws-config-rules-trigger-types)
    > * Configuration changes: 특정 리소스가 생성, 변경 또는 삭제될때 구성사항을 바로 평가
    > * Periodic: 일정한 간격을 기준으로 구성사항을 평가

4. AWS Config Dashboard에서 **[Rules]** 로 이동 후 **Rule name** 탭에 있는 **iam-user-mfa-enabled** 선택 &rightarrow; **Compliance status**를  *Compliant*, *Noncompliant*로 변경해보면서 어떤 Resource가 규칙을 준수 혹은 미준수 하는지 확인

## Custom Rules

```text
Hello,

We previously sent a communication in early October to update your RDS SSL/TLS certificates by October 31, 2019. We have extended the dates and now request that you act before February 5, 2020 to avoid interruption of your applications that use Secure Sockets Layer (SSL) or Transport Layer Security (TLS) to connect to your RDS and Aurora database instances....
......
```

1. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 Lambda를 검색하거나 **[Compute]** 밑에 있는 **[Lambda를]** 를 선택

2. Lambda Dashboard에서  **[Create function]** 클릭후, **Function name** = rds-ca-2019, **Runtime** = Python 3.8, **[Create function]** 클릭

3. 좌측 하단에 있는 **[Execution role]** 에서 **View the rds-ca-2019-role-xxxx** on the IAM console 를 선택

4. **[Add inline policy]** 선택 후,\
**Service** = RDS, **Actions** = DescribeDBInstances, **[Add additional permissions]** 클릭, **Service** = Config, **Actions** = PutEvaluations\
**[Review policy]**  &rightarrow; **Name** = rds-ca-2019 &rightarrow; **[Create policy]**

5. Lambda Console로 돌아와서 아래 코드블록을 Lambda에 복사 후, **[Save]** 클릭

    ```python
    import json
    import boto3
    from datetime import datetime

    def lambda_handler(event, context):

        reseultToken = event['resultToken']

        ## Constructs service objects
        config = boto3.client('config')
        rds = boto3.client('rds')

        evaluations = []

        dbinstances = rds.describe_db_instances()

        for dbinstance in dbinstances['DBInstances']:

            compliance_type = 'COMPLIANT' if dbinstance['CACertificateIdentifier'] == 'rds-ca-2019' else 'NON_COMPLIANT'
            evaluation = {
                'ComplianceResourceType': 'AWS::RDS::DBInstance',
                'ComplianceResourceId': dbinstance['DbiResourceId'],
                'ComplianceType': compliance_type,
                'OrderingTimestamp': datetime.now()
            }
            evaluations.append(evaluation)

        response = config.put_evaluations(
            Evaluations=evaluations,
            ResultToken=reseultToken
        )

    ```

6. AWS Config Dashboard에서 **[Rules]** 로 이동 후 **[:heavy_plus_sign: Add rule]** &rightarrow; **[:heavy_plus_sign: Add custom rule]** &rightarrow;\
**Name** = rds-ca-2019, **AWS Lambda function ARN** = 위에서 생성한 Lambda function ARN, **Trigger type** = :white_check_mark: Periodic, **Frequency** = 24 hours &rightarrow; **[Save]**

7. AWS Config Dashboard에서 위에서 생성한 Rule을 선택 &rightarrow; **Compliance status**를  *Compliant*, *Noncompliant*로 변경해보면서 어떤 Resource가 규칙을 준수 혹은 미준수 하는지 확인

## Auto Remediation

1. AWS Config Dashboard에서 **[Rules]** 로 이동 후 **[:heavy_plus_sign: Add rule]** &rightarrow; *[Filter by rule name, label or description]* 에 ssh를 입력하고 **restricted-ssh**를 선택 &rightarrow; **[Save]**

2. AWS Management Console에서 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 EC2를 검색하거나 **[Compute]** 바로 밑에 있는 **[EC2]** 를 선택

3. 왼쪽 패널에서 NETWORK & SECURITY 섹션 아래에 있는 **Security Groups** 를 선택 &rightarrow; **[Create Security Group]** &rightarrow; **Security group name** = unrestricted-ssh, **Description** = unrestricted-ssh 으로 지정하고 아래와 같은 Inbound rule 추가후 **[Create]**

    | Type | Protocol | Port Range | Source            | Description |
    |------|----------|------------|-------------------|-------------|
    | SSH  | TCP      | 22         | Custom, 0.0.0.0/0 |             |

4. AWS Config Dashboard에서 **restricted-ssh** rule을 선택 &rightarrow; **[Re-evaluate]** &rightarrow; **Compliance status**를  *Noncompliant*로 설정후 방금 만들 보안그룹이 추가 되었는지 확인

5. 우측 상단에 **[Edit]** 클릭  &rightarrow; **Remediation action** 선택 후 텍스트박스에 SecurityGroup 입력 후 **AWS-DisablePublicAccessForSecurityGroup** 선택 &rightarrow; **Resource ID parameter** = GroupId &rightarrow; **[Save]**

6. AWS Config Dashboard에서 **restricted-ssh** rule을 선택 &rightarrow; **[Re-evaluate]**

7. **Resource ID**에서 방금 생성한 보안그룹 ID를 체크하고 **[Remediate]** 클릭

8. 보안그룹에서 SSH, 0.0.0.0/0 Inbound rule이 삭제됐는지 확인 후, 삭제 됬을 경우 다시 추가

9. 5분 정보 기다린 후 방금 위에서 추가한 Inbound rule이 존재하는지 확인

## Aggregating Results

1. AWS Management Console에서 우측 상단에 있는 **[Support]** 선택 &rightarrow; **[Support Center]** &rightarrow; **Account number**를 클립보드에 복사

2. AWS Config Dashboard에서 **[Aggregators]** 로 이동 후 **[:heavy_plus_sign: Add aggregator]** &rightarrow; :white_check_mark: Allow AWS Config to replicate data from source account(s) into an aggregator account. You must select this checkbox to continue to add an aggregator.

3. **Aggregator name** = aggregator &rightarrow; :radio_button: Add individual account IDs &rightarrow; **[Add source accounts]** &rightarrow; :radio_button: Add AWS account IDs &rightarrow; 텍스트박스에 위에서 복사한 Account number 붙여넣기 &rightarrow; **[Add source accounts]**

4. **Regions**에 모든 리전 선택 &rightarrow; :white_check_mark: Allow AWS Config to aggregate data from all future AWS regions where AWS Config is enabled. &rightarrow; **[Save]**

5. 다른 리전에 AWS Config 설정\
**Resource types to record** = :white_check_mark: Record all resources supported in this region,\
**Amazon S3 bucket** = :radio_button: Choose a bucket from your account,\
**Bucket name** = 다른 리전에서 AWS Config 구성할때 생성된 Bucket,\
**AWS Config role** = :radio_button: Use an existing AWS Config service-linked role\
&rightarrow; **[Next]**

6. *[Filter by rule name, label or description]* 에 ssh을 입력하고 **restricted-ssh** 를 선택하고 **[Next]** &rightarrow; **[Confirm]**

7. 기존 리전으로 돌아와서 2. AWS Config Dashboard에서 **[Aggregated view]** 로 이동 후 다른 리전에서 생성한 restricted-ssh rule이 보이지는 확인
