# SwiftLogix 보안 감사 대비 발표 자료

> 본 문서는 SwiftLogix의 **네트워크 보안 / 데이터 보안 / 접근 제어 / 재해 복구 / SLA 대응 체계**를 정리한 보안 설계 문서입니다.  

## 1. 아키텍쳐 개요

### 전체 구성도

> 실제 프로젝트에서 구축한 아키텍처는 아래와 같이 3-Tier(웹–서비스–데이터) 구조로 구성되어 있으며,
> 자동 복구·고가용성·격리된 네트워크·보안 계층을 모두 고려해 설계되었습니다.

#### 핵심 구성 요소 요약

| 계층                             | 구성 요소                  | 역할                                   |
| -------------------------------- | -------------------------- | -------------------------------------- |
| 웹 계층 (Public Subnet)          | ALB, Bastion Host          | 외부 요청 수신, 운영자 SSH 진입 지점   |
| 서비스 계층 (Private App Subnet) | ECS Fargate(Task 최소 2개) | 주문 서비스 애플리케이션 컨테이너 실행 |
| 데이터 계층 (DB Subnet)          | Amazon RDS PostgreSQL      | 트랜잭션·데이터 저장소                 |

#### 핵심 특징

- 외부와 직접 통신하는 리소스는 **ALB와 Bastion만**

- ECS/Fargate, RDS는 **인터넷에서 직접 접근 불가**

- DB는 **100% Private Subnet에서만 통신 가능**

- 모든 장애 발생 시 **AWS 자동 복구 기능이 즉시 작동**
- *현재 ECS는 Multi-AZ 배포의 원칙을 충족하는 상태이지만, RDS는 예산 문제로 인해 Single-AZ 구성*



___



## 2. 네트워크 보안

### 2-1. Public / Private Subnet 격리

| Subnet 계층 | 목적              | 보안 효과                                     |
| ----------- | ----------------- | --------------------------------------------- |
| Public      | ALB, Bastion 배치 | 외부 노출 최소화(애플리케이션 직접 접근 차단) |
| Private App | ECS Fargate       | 내부 서비스만 동작, ALB 외 접근 불가          |
| DB Subnet   | RDS               | No Internet Route, 외부 접근 100% 차단        |

> Public → Private → DB 구조로 파고들기 어려운 다단계 심층 방어 구성.

### 2-2. Route Table 기반 네트워크 통제

| Route Table | Subnet      | 주요 규칙                         | 보안 목적                  |
| ----------- | ----------- | --------------------------------- | -------------------------- |
| rt-public   | Public      | 0.0.0.0/0 -> IGW                  | 외부 요청만 허용           |
| rt-private  | Private App | Outbound -> NAT                   | 패치 등 외부 나가기만 가능 |
| rt-db       | DB          | **Local Only(IGW/NAT 경로 없음)** | 인터넷 절대 차단           |

### 2-3. Security Group 기반 최소 권한 통제

#### 화이트리스트 기반 방화벽

| SG         | 허용 Inbound | 허용 대상          | 목적                   |
| ---------- | ------------ | ------------------ | ---------------------- |
| ALB-SG     | 80/443       | ALL                | 외부 요청 수신         |
| ECS-SG     | 19505        | ALB-SG             | ALB만 ECS 호출 가능    |
| RDS-SG     | 5432         | ECS-SG, Bastion-SG | App / 운영자만 DB 접근 |
| Bastion-SG | 22           | 사내 고정 IP       | 운영자 한정 SSH        |

#### 보안 효과

- 애플리케이션 컨테이너는 외부 접근 완전 차단
- DB는 오직 ECS와 Bastion만 접속 가능 -> 내부 침입 확률 최소화



---



## 3. 데이터 보안

### 3-1. 전송 중 암호화(HTTPS + SSL)

- ALB단에서 HTTPS 종단 처리
- Private Subnet 내부 통신도 AWS 내부 네트워크로 보호됨
- RDS는 SSL 연결을 지원(*현재는 SSL 강제 적용이 완성되지 않았으나, 향후 활성화 가능*)

### 3-2 저장 데이터 암호화

- RDS PostgreSQL: Storage-level AES-256 암호화 활성화
   (AWS KMS 기반 / 기본 구성)

### 3.3 백업 및 복구 전략

- 자동 백업(무료 구간) 유지
- Point-In-Time Recovery 지원
- 고객사 증가에 따른 규모 확장 시 백업 보관기간 연장 및 Cross-Region 백업 고려
  (*단, 현재는 예산 이슈로 리전 DR 전략 비적용*)



---



## 4. 접근 제어

### 4-1. IAM Role 기반 권한 관리

- ECS Task Execution Role
   → ECR Pull, CloudWatch Logs Push 권한만 포함
- Bastion Host IAM Role
   → 관리형 정책 최소화, SSM 연결 고려 가능

### 4-2. Bastion Host를 통한 운영자 제한 접근

- 운영자 SSH는 **회사 고정 IP /32**에서만 허용
- DB 접근도 Bastion → RDS만 허용 (직접 접근 차단)
- SSH Key 기반, 패스워드 로그인 금지

### 4-3. CloudWatch Logs 기반 모든 작업 추적

- ECS 애플리케이션 로그
- RDS 성능지표·오류 알람
- 필요 시 VPC Flow Logs를 활성화하여 네트워크 단위 접근 감사 가능
- DB Slow Query Log 역시 성능/보안 모니터링에 사용이 가능



---



## 5. 고가용성 및 재해 복구

### 5-1. Multi-AZ 고려

- ECS Fargate는 기본적으로 여러 Subnet에 배포되므로 AZ 장애 시 자동 우회
- RDS는 현재 Single-AZ 구성 -> 자동  Failover 없어 운영자 개입 필요한 상태

#### ECS Fargate

- Multi-AZ 배포 -> 한 AZ 장애 시 자동으로 다른 AZ Task가 서비스 유지
- Health Check 실패 시 자동으로 새 Task 생성

#### ALB Health Check

- `/actuator/health`
- 2회 연속 실패 시 즉시 차단

### 5-2. 자동 복구 시나리오 요약

| 장애           | 감지 방식                       | 자동 복구 방식          | 예상 복구 시간 |
| -------------- | ------------------------------- | ----------------------- | -------------- |
| OOM, App crash | ALB/ECS Health check            | 새 Task 자동 재생성     | 30~60초        |
| ECS Host 장애  | EC2 StatusCheckFail             | 다른 AZ에서 Task 재배포 | 1~2분          |
| DB 장애        | (Multi-AZ일 경우) 자동 Failover | Standby로 자동 승격     | 30~120초       |
| AZ 전체 장애   | ALB Unhealthy                   | 다른 AZ로 트래픽 우회   | 수 분 내       |

(※ RDS는 현재 **Single-AZ**, Failover 없음 -> 운영자 개입 필요)

------

# 6. Q&A(예상 질문에 대한 답변)

### 1) 데이터베이스가 인터넷에 노출되어 있나요?

 절대 불가능합니다.

- DB Subnet에는 IGW/NAT Route 없음
- 유일한 진입점: ECS-SG + Bastion-SG

### 2) 서버(Task)가 죽으면 어떻게 되나요?

- ECS는 Desired Count를 항상 유지
- Task crash → 30초 이내 자동 재시작
- ALB Health Check가 비정상 Task를 즉시 제외

### 3) 해커가 SSH로 침입하면?

- Bastion Host는 회사 고정 IP에서만 접속 가능합니다.
- SSH Key 기반 인증 + IAM 최소 권한
- CloudWatch로 모든 접속 기록 추적 가능

### 4) SLA(95th < 1s) 충족 근거는?

- JMeter 부하 테스트 결과:
   → **95th = 20ms**, SLA 기준의 1/50 수준
- 0.5 vCPU Task 1개가 안정적으로 20RPS 처리 가능
- Scale-out 기반 구조로 200RPS도 대응 가능

- **SLA 검증 데이터 요약**
  - Median: 8ms
  - 95th: 20ms
  - Max: 382ms
  - Error: 0%
