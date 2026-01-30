# AWS 아키텍처 개요

---

## 1) 전체 구성 개요

- Terraform으로 VPC, EKS, 애드온(ALB), 앱/DB, 관측성까지 일괄 구성한다.
- 외부 트래픽은 ALB Ingress로 유입되고 `/`는 Frontend, `/api`는 Backend로 라우팅된다.
- Backend는 Postgres(StatefulSet + EBS PVC)와 연결된다.
- 모든 워크로드는 EKS 위에서 실행된다.

---

## 2) 아키텍처 워크플로우

### 다이어그램 (텍스트)

```
Internet
  |
  v
ALB (Public Subnet)
  |
  v
Ingress (EKS)
  |                |
  | /              | /api
  v                v
Frontend SVC     Backend SVC
  |                |
  v                v
Frontend Pods    Backend Pods
                   |
                   v
          postgres (ExternalName)
                   |
                   v
       Postgres SVC (Headless)
                   |
                   v
        Postgres StatefulSet
                   |
                   v
               EBS PVC
```

### 다이어그램 (네트워크/보안그룹)

```
VPC
├─ Public Subnets
│   └─ ALB (SG: 80 open to 0.0.0.0/0)
│        |
│        v
├─ Private Subnets
│   └─ EKS Nodes (SG: allow from ALB SG)
│        |
│        v
│     Pods (Frontend/Backend/Postgres)
│
└─ EBS (PVC gp2) attached to Postgres Pod
```

### 워크플로우 A: 외부 트래픽 → Frontend

1. 사용자 요청이 ALB(퍼블릭 서브넷)로 유입된다.
2. Ingress 규칙에 따라 `/` 경로는 Frontend 서비스로 라우팅된다.
3. Frontend 서비스가 Frontend Pod(Deployment)로 트래픽을 전달한다.
4. 브라우저는 정적 리소스를 제공받는다.

### 워크플로우 B: 외부 트래픽 → Backend API

1. 사용자 요청이 ALB로 유입된다.
2. Ingress 규칙에 따라 `/api` 경로는 Backend 서비스로 라우팅된다.
3. Backend 서비스가 Backend Pod(Deployment)로 트래픽을 전달한다.
4. Backend는 `POSTGRES_HOST=postgres`로 DB에 접근한다.
5. ExternalName Service가 `postgres.database.svc.cluster.local`로 연결한다.
6. Postgres StatefulSet이 요청을 처리한다.

### 워크플로우 C: 데이터 영속화

1. Postgres는 StatefulSet으로 배포된다.
2. PVC(5Gi, gp2)를 통해 EBS에 데이터가 저장된다.
3. Pod 재시작/재스케줄링 시에도 데이터가 유지된다.

### 워크플로우 D: 모니터링/관측성

1. kube-prometheus-stack이 클러스터 메트릭을 수집한다.
2. CloudWatch Observability가 로그/메트릭을 AWS로 전송한다.
3. Grafana는 내부 대시보드를 제공한다.

---

## 3) 설계 의도

- **모듈 분리**: 네트워크/클러스터/애드온/앱을 분리해 재사용성과 변경 용이성을 확보한다.
- **EKS 기반 운영**: 워크로드를 표준 Kubernetes 패턴(Deployment/StatefulSet/Ingress)으로 관리한다.
- **ALB Ingress**: AWS 네이티브 LB를 사용해 안정적인 외부 노출을 보장한다.
- **StatefulSet + PVC**: DB 데이터의 영속성을 보장한다.
- **관측성 내장**: 기본 Prometheus + CloudWatch를 통해 운영 가시성을 확보한다.

---

# 빠른 요약

- VPC → EKS → ALB → Frontend/Backend → Postgres 흐름으로 구성된다.
- `/`와 `/api` 경로 기반 라우팅을 사용한다.
- DB는 EBS에 저장되어 상태가 유지된다.
- 관측성 스택으로 운영 모니터링을 지원한다.

---
