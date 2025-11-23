# SwiftLogix – Deployment Report

- 배포 방식: AWS ECS Fargate + Spring Boot + PostgreSQL + ALB
- 목적: 컨테이너 기반 B2B 물류 서비스 배포 자동화 및 운영환경 구축
- 기간: 2025.11.18 ~ 2025.11.24
- 주요 결과: 네트워크, 보안, DB, 컨테이너, 배포, 이벤트 전파까지 전체 흐름 E2E 성공

## 네트워크 구성

### VPC Resource Map

VPC 구성은 아래와 같이 6개의 Subnet, 5개의 Route Table, IGW 및 2개의 NAT Gateway로 구성된다.

<img width="2447" height="774" alt="image" src="https://github.com/user-attachments/assets/646a87ff-b1cf-4c30-a13a-09a5f392fdac" />


**[2025-11-23] — VPC Resource Map**
- swiftlogix-vpc 내부 Subnet, Route Table, IGW, NAT Gateway 전체 구성 확인

### Security Groups 설정 (3개 SG의 Inbound rules)

<img width="2485" height="704" alt="image" src="https://github.com/user-attachments/assets/dbb6d79f-13d8-461d-bce0-ff2d981b34e7" />


ALB-SG 인바운드 규칙

**[2025-11-23] Security Group – ALB-SG (Inbound Rules)
Inbound: HTTP 80 → 0.0.0.0/0**

[외부 요청이 ALB로 진입하는 공개 진입점 구성]
- 외부 클라이언트가 ALB로 접근할 수 있도록 공개 트래픽을 허용하는 규칙.

- 전체 요청은 반드시 ALB를 거쳐 내부 서비스로 전달된다.

- (보안 구조에서 유일한 Public Entry Point)

<img width="2460" height="687" alt="image" src="https://github.com/user-attachments/assets/4d5a54b9-c8d5-4dca-96a4-767601fef851" />


ECS-SG 인바운드 규칙

**[2025-11-23] Security Group – ECS-SG (Inbound Rules)
Inbound: TCP 19505 → ALB-SG**

[ALB에서만 애플리케이션 컨테이너 접근 허용]
- ALB에서 들어온 트래픽만 애플리케이션 컨테이너(Spring Boot 19505 포트)에 접근하도록 제한.

- 외부 direct 접근은 불가능하며, 실무 표준 서비스 보호 정책 적용됨.
  
<img width="2488" height="807" alt="image" src="https://github.com/user-attachments/assets/e9842f99-83f4-4659-bc90-d6bd93eb16f5" />

RDS-SG 인바운드 규칙

**[2025-11-23] Security Group – RDS-SG (Inbound Rules)**

[DB 접근 허용 대상을 Bastion 및 ECS로 제한 (강화된 보안 구조)]

**Inbound 1: TCP 5432 → Bastion-SG**Bastion Host에서 RDS로 직접 접속(운영/관리 목적)을 허용.

**Inbound 2: TCP 5432 → ECS-SG**

- 애플리케이션(ECS 컨테이너)에서 RDS PostgreSQL에 접근할 수 있도록 허용.

- DB는 오직 Bastion·ECS만 접근 가능하며 외부 접근은 차단된 상태.

### NAT Gateway Status "Available"

<img width="1169" height="273" alt="image" src="https://github.com/user-attachments/assets/28f7a474-09e2-43a3-9cff-c6f30447ea43" />


**[2025-11-23] NAT Gateway 상태 확인 (Available)**

- NAT Gateway **natgw-2a**, **natgw-2c** 모두 상태 **Available**
- Private Subnet(Fargate·ECS Task)가 인터넷 Outbound를 사용할 수 있음을 확인
- Public Subnet에 정상 배치, EIP 연결 완료

## **데이터베이스**

### RDS 인스턴스 상세

<img width="1951" height="1193" alt="image" src="https://github.com/user-attachments/assets/eca142bb-2dc7-4d7b-a877-ce83f7ecf208" />


**[2025-11-19] RDS 인스턴스 상세 구성 확인 (rocket-delivery-db)**

- PostgreSQL 16 / db.t3.micro(2 vCPU · RAM 1GB) 정상 구동
- 스토리지: gp3 20GB, 암호화 활성화
- Public Access 비활성화, DB Subnet Group 정상 사용
- 테스트 환경(Single-AZ) 기준으로 설정 최적화

### Bastion에서 MySQL 접속

<img width="979" height="321" alt="image" src="https://github.com/user-attachments/assets/f10cdc17-c1de-4c78-85f9-f8fb84ee6087" />

여기까지 들어오면 완료 SSH 접속 완료\

<img width="1639" height="669" alt="image" src="https://github.com/user-attachments/assets/7caebb44-fb26-41a4-a21e-bbd9ad4c4055" />


<img width="938" height="358" alt="image" src="https://github.com/user-attachments/assets/113c7fa0-c9a7-42e5-b871-69373d57af14" />


2025/11/19 _ Bastion에서 PostgreSQL 접속 성공

**[2025-11-19] Bastion에서 PostgreSQL 접속 성공 (쿼리 결과 확인)**

- Bastion Host(Amazon Linux 2023)에서 Private RDS에 정상 접속
- `SELECT NOW()` 실행 결과 정상 반환 → DB 시간 정상
- `SELECT current_database(), current_user, NOW()` 결과
    - 현재 DB: **postgres**
    - 현재 사용자: **app_user**
- RDS 보안 그룹에서 **Bastion-SG → 5432 허용** 규칙이 정상적으로 동작함 확인
- Private Subnet(DB Subnet)에 위치한 PostgreSQL 인스턴스 접근 성공

## **컨테이너 준비**

### 도커 파일

```
FROM eclipse-temurin:17-jre-alpine

ENV TZ=Asia/Seoul

ARG JAR_FILE=build/libs/*.jar

COPY ${JAR_FILE} app.jar

EXPOSE 19505

ENTRYPOINT ["java", "-Dserver.port=19505", "-jar", "/app.jar"]
```

### 레포지토리 정보

```
Repository Name: swiftlogix-order
Repository URI: 199844352575.dkr.ecr.ap-northeast-2.amazonaws.com/swiftlogix-order
Image Tag: 1.0.0
Region: ap-northeast-2 (서울)
ContainerPort = 19505
```

### 이미지 태그 전략

```
1.0.0
1.0.1
 ...
```

## 컨테이너 배포 확인

### ECR 이미지 목록

<img width="2516" height="276" alt="image" src="https://github.com/user-attachments/assets/c0bad9b9-92b1-4801-a871-710e8e622352" />
ECR

<img width="2225" height="297" alt="image" src="https://github.com/user-attachments/assets/b762a4db-c2a2-4138-8ce1-2bda66cc82bf" />
ECR 이미지 목록 (첫번째 풀링)

<img width="2489" height="450" alt="image" src="https://github.com/user-attachments/assets/014db5d1-fe4d-4342-b796-8004e1a5f65f" />
ECR 이미지 목록 (최종본 풀링)

**[2025-11-24] ECR 이미지 목록 확인 (swiftlogix-order)**

- 프라이빗 리포지토리 **swiftlogix-order** 정상 생성됨
- 이미지 태그 **1.0.3** 총 3개의 이미지 존재
- 빌드된 Docker 이미지가 ECR로 정상 Push 되었음을 확인
- ECS Task Definition에서 참조하는 **1.0.3 이미지가 모두 최신 상태로 업로드 완료**
- SHA256 Digest 확인 가능 → 이미지 무결성 유지

## **ECS 배포**

### ECS Services 목록

<img width="2474" height="900" alt="image" src="https://github.com/user-attachments/assets/9f57f37f-899d-407b-982e-626a030b1309" />

**[2025-11-24] ECS 서비스 상태 확인 (order-service 활성)**

- 클러스터 `swiftlogix-order` 내 서비스 `order-service` **활성(Active)**
- 원하는 Task 개수 2개 기준으로 **2/2 태스크 실행 중**
- Fargate 기반 단일 모놀리식 서비스가 정상적으로 기동됨을 확인

### CloudWatch Logs

<img width="2496" height="942" alt="image" src="https://github.com/user-attachments/assets/d0de7735-d55d-4bd4-8120-b2e10d25d207" />

**[2025-11-24] CloudWatch 로그 확인 – Spring Boot 정상 기동 확인**

- ECS Fargate Task의 `/ecs/order-service` 로그 그룹에서 **컨테이너 실행 로그 확인**
- Spring Boot v3.5.6 애플리케이션 정상 부팅 완료
- RepositoryConfigurationDelegate 로그를 통해 JPA 설정 정상 로드 검증
- 서비스가 정상적으로 기동되어 ALB → ECS → 컨테이너까지 **엔드투엔드 동작 보증**

## **통합 테스트**

### API 호출 결과

<img width="1973" height="394" alt="image" src="https://github.com/user-attachments/assets/79a0d042-0030-44a3-9c00-e6f1e0bbf58e" />

<img width="933" height="719" alt="image" src="https://github.com/user-attachments/assets/4c4c39a3-5f95-40fb-a508-aa30e1dc6ae4" />

<img width="1172" height="1223" alt="image" src="https://github.com/user-attachments/assets/1d3f1b8b-bd48-4d48-9eeb-dadfeab61385" />

<img width="2546" height="765" alt="image" src="https://github.com/user-attachments/assets/4c5aaede-e655-4135-90aa-23861ed1958d" />


**[2025-11-24] 주문 생성/조회 End-to-End 처리 검증 (ECS → RDS 정상 동작)**

- ALB 엔드포인트를 통해 **주문 생성/조회 요청(POST/GET /api/orders)** 정상 응답
- `status: PLACED`, `totalPrice`, `orderId`등 모든 필드 응답 확인
- ECS Task(2개), RDS 연결, VPC 라우팅, 보안 그룹 모두 정상 구성 되었음을 증명
- 실 서비스 수준의 **End-to-End API 처리 성공** 사례

### 이벤트 전파 확인 (3개 서비스 로그)

<img width="2462" height="566" alt="image" src="https://github.com/user-attachments/assets/e7f904e0-b020-4576-a8cd-0651c8f1ebab" />

<img width="1150" height="874" alt="image" src="https://github.com/user-attachments/assets/b023f9bd-b86e-425a-9306-154e892920da" />


**[2025-11-21] 주문 생성 후 로그 이벤트 확인**

### MySQL 데이터 확인 (3개 테이블 조회)

서비스가 ECS Fargate + RDS(PostgreSQL) 환경에서 정상적으로 초기화되었는지 확인하기 위해 생성한 초기 데이터가 RDS에 정상 반영되었는지 조회하였다.
아래는 Bastion Host → psql 접속 후 실행한 실제 검증 스크린샷이다.

<img width="1709" height="731" alt="image" src="https://github.com/user-attachments/assets/9bbeb3d9-c9c5-448e-9591-834934a25559" />


**[2025-11-21] RDS p_order_stocks 초기 재고 데이터 확인**

- 10개 상품 각각 재고(quantity)가 정상 저장
- 모든 재고 status = IN_STOCK 확인
- reserved_quantity = 0으로 초기 상태 정상
- hub_id 및 회사(company_id) 매핑 정상

<img width="2529" height="638" alt="image" src="https://github.com/user-attachments/assets/7f03396f-70a5-45fa-a8df-f06075e8dd82" />


**[2025-11-21] RDS p_orders 초기 샘플 주문 데이터 확인**

- 3개의 초기 주문(PLACED)이 정상 반영됨
- due_at, user 정보, 주소 스냅샷 등 도메인 요구사항 충족
- product_id / supplier_id / hub_id 데이터 정상
- canceled_xxx 필드 모두 null → 정상 상태

<img width="1707" height="1114" alt="image" src="https://github.com/user-attachments/assets/1bb0d344-bf73-4faa-a95f-30438ce443cb" />


**[2025-11-21] RDS p_order_products 초기 데이터 확인**

- 10개의 상품(Product)이 정상 삽입됨
- hub_id·company_id 등 참조키 데이터 정상
- is_active = true 로 활성화 상태 확인
- 가격(amount) 및 이름(product_name) 정상 반영

### 전체 아키텍처 체크리스트 완성본

**Phase 1: 네트워크 구성 (2-3시간)**

| No | 항목 | 검증 여부 |
| --- | --- | --- |
| 1 | VPC Resource Map에서 Public/Private Subnet, AZ 분리 확인 | ✔ 완료 |
| 2 | Security Group 3종(ALB-SG, ECS-SG, RDS-SG) Inbound Rule 구성 확인 | ✔ 완료 |
| 3 | NAT Gateway 2개 상태(Available) 확인 | ✔ 완료 |
- [x]  VPC 생성 (CIDR, DNS 설정)
- [x]  Internet Gateway 생성 및 연결
- [x]  Subnet 6개 생성 (Public 2, Private 2, DB 2)
- [x]  NAT Gateway 2개 생성 (각 AZ)
- [x]  Route Table 설정 (Public, Private, DB 각각)
- [x]  Security Group 4개 생성 (ALB, ECS, RDS, Bastion)

**Phase 2: 데이터베이스 (1-2시간)**

| No | 항목 | 검증 여부 |
| --- | --- | --- |
| 4 | RDS PostgreSQL 인스턴스 상세(엔드포인트·보안·서브넷) 확인 | ✔ 완료 |
| 5 | Bastion → RDS psql 접속 및 3개 테이블 조회 성공 | ✔ 완료 |
- [x]  RDS Subnet Group 생성 (DB Subnet 2개 포함)
- [x]  RDS MySQL 인스턴스 생성 (Multi-AZ 여부는 Mission 1 결정에 따라)
- [x]  Bastion Host 생성 (Public Subnet에 배치)
- [x]  Bastion에서 RDS 접속 테스트
- [x]  데이터베이스 스키마 생성 (3주차 과제 SQL 스크립트 활용)

**Phase 3: 컨테이너 준비 (2-3시간)**

| No | 항목 | 검증 여부 |
| --- | --- | --- |
| 6 | ECR 리포지토리 및 latest 이미지 3개 업로드 확인 | ✔ 완료 |
- [x]  Dockerfile 3개 작성 (각 서비스별)
- [x]  ECR 리포지토리 3개 생성
- [x]  Docker 이미지 빌드 및 ECR 푸시
- [x]  이미지 태그 전략 (latest, v1.0.0 등)

**Phase 4: ECS 배포 (3-4시간)**

| No | 항목 | 검증 여부 |
| --- | --- | --- |
| 7 | ECS Service 3개 모두 Active / Task 2 Running 확인 | ✔ 완료 |
| 8 | CloudWatch Logs에 ECS Task 로그 수집 정상 확인 | ✔ 완료 |
- [x]  ECS Cluster 생성 (Fargate 타입)
- [x]  Task Definition 3개 작성 (환경변수로 RDS 엔드포인트 설정)
- [x]  ALB 생성 및 Target Group 3개 설정
- [x]  Listener Rule 설정 (경로 기반 라우팅)
- [x]  ECS Service 3개 생성 (각 서비스당 Task 개수는 Mission 1 결정)
- [x]  Health Check 검증

**Phase 5: 통합 테스트 (1-2시간)**

| No | 항목 | 검증 여부 |
| --- | --- | --- |
| 9 | API 주문 생성 성공 (POST /api/orders) | ✔ 완료 |
| 10 | Kafka 이벤트 전파 확인 (reserved / failed / confirmed 등) | ✔ 완료 |
| 11 | 주문 생성 이후 RDS 내 Orders/Stocks/Products 데이터 반영 확인 | ✔ 완료 |
| 12 | 전체 아키텍처 체크리스트 문서 작성 및 검증 | ✔ 완료 |
- [x]  API 엔드포인트 테스트 (Postman 또는 curl)
- [x]  이벤트 전파 확인 (CloudWatch Logs에서 확인)
- [x]  데이터베이스 데이터 확인 (Bastion 통해 접속)
- [x]  CloudWatch Logs 확인 (에러 없는지)
