# week4-assignment

### 4주차 AWS ECS 클라우드 배포 및 아키텍처 설계 과제

### 1. docs

   https://github.com/hanbin-and-the-crew/week4-assignment/tree/main/docs

[문서 종류]

  - ARCHITECTURE_DESIGN.md (AWS Pricing Calculator 스크린샷, 네트워크 다이어그램)
  - DEPLOYMENT_REPORT.md (12개 스크린샷 포함)
  - SECURITY_PRESENTATION.md

---
   
### 3. Dockerfile 위치
   
   https://github.com/hanbin-and-the-crew/rocket-delivery/blob/feature/event-listener/module-order/Dockerfile


---


### 5. API 호출 테스트 방식 

## 주문 생성 API 테스트 (POST /api/orders)

### 📌 요청 URL
POST http://swiftlogix-alb-699706001.ap-northeast-2.elb.amazonaws.com/api/orders

### 📌 Headers
Content-Type: application/json

### 📌 Request Body
```json
{
  "supplierId": "55e08400-e29b-41d4-a716-446655440000",
  "supplierCompanyId": "55e08400-e29b-41d4-a716-446655440001",
  "supplierHubId": "55e08400-e29b-41d4-a716-446655440002",
  "receiptCompanyId": "55e08400-e29b-41d4-a716-446655440003",
  "receiptHubId": "55e08400-e29b-41d4-a716-446655440004",
  "productId": "55e08400-e29b-41d4-a716-446655440005",
  "quantity": 10,
  "deliveryAddress": "서울특별시 강남구 테헤란로 123",
  "userName": "최원철",
  "userPhoneNumber": "010-1111-2222",
  "slackId": "12@1234.com"
}
```

### 📌 응답 예시 (성공)
```json
{
  "meta": {
    "result": "SUCCESS",
    "errorCode": null,
    "message": null
  },
  "data": {
    "orderId": "5eb6a72a-ffa5-4ea5-b185-a5c9430769f1",
    "status": "PLACED",
    "totalPrice": 100000,
    "createdAt": null
  }
}
```

응답 코드: **200 OK**


---

## 주문 목록 조회 API 테스트 (GET /api/orders)

### 📌 요청 URL
GET http://swiftlogix-alb-699706001.ap-northeast-2.elb.amazonaws.com/api/orders

### 📌 Headers
(설정 없음)

### 📌 응답 예시 (성공)
```json
[
  {
    "orderId": "10000000-0000-0000-0000-000000000002",
    "userName": "김철수",
    "status": "PLACED",
    "productName": "상품B",
    "quantity": 1,
    "totalPrice": 20000,
    "dueAt": "2025-11-22T04:58:33.199665",
    "createdAt": "2025-11-22T04:58:33.199665",
    "updatedAt": "2025-11-22T04:58:33.199665"
  },
  {
    "orderId": "10000000-0000-0000-0000-000000000001",
    "userName": "홍길동",
    "status": "PLACED",
    "productName": "상품A",
    "quantity": 3,
    "totalPrice": 30000,
    "dueAt": "2025-11-23T04:58:02.027351",
    "createdAt": "2025-11-19T04:58:02.027351",
    "updatedAt": "2025-11-19T04:58:02.027351"
  }
]
```

응답 코드: **200 OK**

]
