# PetCareLog (team5-config)

## 📑 프로젝트 소개
PetCareLog GitOps Configuration은 서비스의 모든 인프라 환경(dev, prod)과 관제 시스템을 쿠버네티스 매니페스트를 통해 유기적으로 제어하고 자동화하는 저장소입니다.  
애플리케이션 배포와 통합 모니터링을 Argo CD와 연동하여 서버 환경의 일관성을 유지하고 안정적인 배포 파이프라인을 유지하는 것을 목표로 합니다.
현재 프로젝트는 다중 클러스터 동기화, 무중단 배포 제어, 실시간 메트릭 관제를 중심으로 구성되어 있습니다.  
애플리케이션 사양 관리는 `apps/petcarelog/base/overlays`, 배포 정책 제어는 `argocd/projects/applicationsets`, 통합 모니터링 및 실시간 관제는 `monitoring/kube-prometheus-stack` 도메인으로 명확히 분리되어 있습니다.

## 🔔 목표
- **인프라 환경의 일관성 보장:** Kustomize를 통해 개발(dev)과 운영(prod) 환경의 설정 차이를 최소화하고 표준화된 배포 템플릿 유지
- **선언적 GitOps 배포 자동화:** 코드 변경 사항이 Argo CD를 통해 실시간으로 클러스터에 반영되는 무중단 배포 체계 선언
- **안전한 다중 클러스터 권한 분리:** AppProject 설정을 기반으로 환경별 리소스 접근 제어 및 클러스터 간 보안 경계 확립
- **애플리케이션 구동 현황 관제:** 프로메테우스와 그라파나를 연동하여 실시간 서버 자원 및 Spring Boot 비즈니스 메트릭 정밀 추적
- **리스크 최소화를 위한 선행 배포 준수:** Prometheus Operator CRD와 관제 본체를 분리 배포하여 인프라 생성 및 확장 시 발생할 수 있는 종속성 오류 원천 차단

---

## 📂 레포지토리 역할
`team5-config`는 PetCareLog 서비스의 Kubernetes 매니페스트와 Argo CD 설정을 제어하는 **단일 진실 공급원 (Single Source of Truth)** 역할을 하는 GitOps 저장소입니다.

### 🛠️ 주요 관리 대상
* **Application Manifests:** Kustomize 기반의 인프라 환경별(dev/prod) Overlay 매니페스트 관리
* **Argo CD Core Architecture:** 멀티 클러스터 제어를 위한 AppProject 및 자동 가동용 ApplicationSet
* **Observability (모니터링):** `kube-prometheus-stack` 기반 Metric 수집 엔진, Prometheus Operator CRD 및 Spring Boot 모니터링용 ServiceMonitor

---

## 🔄 전체 배포 흐름 (GitOps Pipeline)
PetCareLog는 애플리케이션 소스 코드, 인프라 코드, 배포 설정을 각각 다른 레포지토리로 분리하여 관리합니다.

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/6ae45393-f9d9-49fc-9b5f-3f2cf51aa3f7" />

Argo CD는 management EKS에 설치되어 있으며, dev/prod EKS 클러스터를 등록하여 원격으로 배포를 관리합니다.

```text
management EKS
└── Argo CD
    ├── petcarelog-dev 배포
    ├── petcarelog-prod 배포
    ├── monitoring-crds-dev 배포
    ├── monitoring-crds-prod 배포
    ├── monitoring-dev 배포
    └── monitoring-prod 배포
```

---

## 🗂️ 디렉토리 구조

```text
team5-config
├── apps
│   └── petcarelog
│       ├── base
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── ingress.yaml
│       │   ├── configmap.yaml
│       │   ├── serviceaccount.yaml
│       │   ├── servicemonitor.yaml
│       │   └── kustomization.yaml
│       │
│       └── overlays
│           ├── dev
│           │   ├── kustomization.yaml
│           │   ├── patch-deployment.yaml
│           │   ├── patch-ingress.yaml
│           │   └── patch-servicemonitor.yaml
│           │
│           └── prod
│               ├── kustomization.yaml
│               ├── patch-deployment.yaml
│               ├── patch-ingress.yaml
│               └── patch-servicemonitor.yaml
│
├── argocd
│   ├── projects
│   │   ├── petcarelog-dev-project.yaml
│   │   ├── petcarelog-prod-project.yaml
│   │   └── monitoring-project.yaml
│   │
│   └── applicationsets
│       ├── petcarelog-applicationset.yaml
│       ├── monitoring-crds-applicationset.yaml
│       └── monitoring-applicationset.yaml
│
└── monitoring
    └── kube-prometheus-stack
        ├── values-dev.yaml
        └── values-prod.yaml
```

---

## ☸️ 가동 인프라 리소스 명세

| 쿠버네티스 리소스 | 기술 스택 / 활용 목적 |
| :--- | :--- |
| **Namespace** | `petcarelog-dev` / `petcarelog-prod` / `monitoring` 공간 분리 |
| **Deployment** | Spring Boot 애플리케이션 가동 및 배포 Pod 자동 스케일링 제어 |
| **Service** | 클러스터 내부 Pod 간 통신을 위한 로컬 로드밸런싱 경로 제공 |
| **Ingress** | AWS Load Balancer Controller 연동을 통한 인프라 외부 ALB 생성 및 트래픽 라우팅 |
| **ConfigMap** | Spring 프로필 세팅 등 비민감성 환경 설정 변수 분리 관리 |
| **Secret** | DB 패스워드, Redis 토큰 등 보안이 필수적인 자격 증명 관리 (**Git 추적 금지**) |
| **ServiceAccount** | IRSA(IAM Roles for Service Accounts) 기술을 활용해 Pod에 AWS S3 접근 권한 주입 |
| **ServiceMonitor** | Spring Boot Actuator 메트릭 주소를 프로메테우스 엔진에 자동 타겟팅 연동 |

---

## 🛠️ Kustomize 구조

공통 리소스는 `base`에 정의하고, 환경별 차이는 `overlays/dev`, `overlays/prod`에서 patch합니다.

```text
base
→ 공통 Deployment, Service, Ingress, ConfigMap, ServiceAccount, ServiceMonitor

overlays/dev
→ dev 이미지, dev RDS, dev Redis, dev S3, dev ALB, dev ServiceMonitor label 설정

overlays/prod
→ prod 이미지, prod RDS, prod Redis, prod S3, prod ALB, prod ServiceMonitor label 설정
```

### 🔹 dev overlay

dev 환경은 `apps/petcarelog/overlays/dev` 경로를 사용합니다.

- 주요 설정: dev ECR 이미지, dev RDS endpoint, dev ElastiCache Redis endpoint, dev S3 bucket, dev Ingress / ALB 설정, dev ServiceMonitor label

### 🔸 prod overlay

prod 환경은 `apps/petcarelog/overlays/prod` 경로를 사용합니다.

- 주요 설정: prod ECR 이미지, prod RDS endpoint, prod ElastiCache Redis endpoint, prod S3 bucket, prod Ingress / ALB 설정, prod ServiceMonitor label, prod replica 수

---

## 🐙 Argo CD 구성 및 배포 구조

### AppProject

AppProject는 Argo CD Application이 접근 가능한 Git repository, cluster, namespace, resource 범위를 제한합니다.

- 관리 대상: petcarelog-dev, petcarelog-prod, monitoring

### ApplicationSet

ApplicationSet은 환경별 Application을 자동 생성합니다.

- 관리 대상: petcarelog-dev, petcarelog-prod, monitoring-crds-dev, monitoring-crds-prod, monitoring-dev, monitoring-prod

### PetCareLog 배포 구조

`argocd/applicationsets/petcarelog-applicationset.yaml`은 dev/prod 애플리케이션을 생성합니다.

```text
petcarelog-applicationset
├── petcarelog-dev
└── petcarelog-prod
```

각 Application은 다음 경로를 참조합니다.

- apps/petcarelog/overlays/dev
- apps/petcarelog/overlays/prod

---

## 📊 모니터링 및 Grafana 대시보드 구조

모니터링은 kube-prometheus-stack을 사용합니다. 리소스 생성을 위한 오퍼레이터 충돌을 방지하기 위해 CRD 패키지와 본체를 분리하여 순차 배포합니다.
- 구성 요소: Prometheus, Grafana, Alertmanager, Prometheus Operator, Node Exporter, kube-state-metrics

```text
monitoring-crds-applicationset
├── monitoring-crds-dev
└── monitoring-crds-prod

monitoring-applicationset
├── monitoring-dev
└── monitoring-prod
```

> **💡 CRD 선배포 사유:** Prometheus, Alertmanager, ServiceMonitor, PrometheusRule, PodMonitor 등의 리소스들은 Kubernetes 기본 리소스가 아니라 Prometheus Operator CRD가 선행 설치되어야 인식이 가능하기 때문입니다.

### 📈 Grafana 환경별 대시보드 구성

각 환경은 별도의 Grafana를 사용합니다.

- Dev Grafana: Dev Node 모니터링, Dev Kubernetes 모니터링, PetCareLog Dev 애플리케이션 모니터링
- Prod Grafana: Prod Node 모니터링, Prod Kubernetes 모니터링, PetCareLog Prod 애플리케이션 모니터링


**Node 대시보드**

- Grafana Dashboard Import ID: 19230 - Node Exporter Full with Node Name

**Kubernetes 대시보드**

- kube-prometheus-stack 기본 대시보드 활용:
  - Kubernetes / Compute Resources / Cluster
  - Kubernetes / Compute Resources / Namespace
  - Kubernetes / Compute Resources / Pod
  - Kubernetes / Compute Resources / Workload

**Application 대시보드**

PetCareLog 애플리케이션 전용 대시보드는 주요 지표 패널(앱 실행 상태, JVM 메모리, HTTP 요청 수/응답 시간, 활성 DB 커넥션, Pod 자원)을 기준으로 직접 생성합니다.

- 주요 관제 패널: 앱 실행 상태, JVM 메모리 사용량, 실시간 HTTP 요청 수 및 응답 시간, DB 활성 커넥션 수
- 대표 PromQL:

```promql
# 애플리케이션 구동 상태 확인 (Up/Down)
up{namespace="petcarelog-dev"}
```

```promql
# JVM 힙 메모리 실시간 사용량 추적
jvm_memory_used_bytes{namespace="petcarelog-dev"}
```

```promql
# HikariCP 활성 데이터베이스 커넥션 갯수 관제
hikaricp_connections_active{namespace="petcarelog-dev"}
```

```promql
# 톰캣 서버가 처리 중인 실시간 HTTP 총 요청 수
http_server_requests_seconds_count{namespace="petcarelog-dev"}
```

(※prod 환경의 경우 namespace="petcarelog-prod"로 변경하여 사용)

---

## 🚀 사전 인프라 준비 사항
이 저장소의 설정을 적용하기 전, team5-infra의 테라폼 배포가 완벽하게 완료되어 아래 인프라 요소(management/dev/prod EKS, ECR, RDS, Redis, S3, IAM/IRSA 등)가 AWS 상에 가동 중이어야 합니다.

### 1. 타겟 클러스터 및 로컬 kubeconfig Context 동기화
팀 공용 자격 증명(--profile project2) 프로필을 장착한 후, 로컬 컴퓨터에 3개의 클러스터 컨텍스트를 동기화합니다
```bash
# 1. Management 클러스터 연결
aws eks update-kubeconfig --region ap-northeast-2 --name team5-petcarelog-management-eks --profile project2

# 2. Dev 클러스터 연결
aws eks update-kubeconfig --region ap-northeast-2 --name team5-petcarelog-dev-eks --profile project2

# 3. Prod 클러스터 연결
aws eks update-kubeconfig --region ap-northeast-2 --name team5-petcarelog-prod-eks --profile project2
```

### 2. Management 클러스터에 Argo CD 핵심 사령탑 가동 및 로그인
```bash
# 콘텍스트를 중앙 매니지먼트로 고정
kubectl config use-context arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-management-eks

# Argo CD 전용 네임스페이스 생성 및 엔진 가동
kubectl create namespace argocd
kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)

# Argo CD 서버 제어 웹 접근을 위한 로컬 포트포워딩 터널 가동 (터미널 대기 유지)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
새로운 터미널 창을 열어 관리자 초기 비밀번호 확인 후 CLI 로그인을 마칩니다.

```bash
# 초기 비밀번호 디코딩 확인
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d

# Argo CD CLI 로그인 수행
argocd login localhost:8080 --insecure
# Username: admin / Password: 위에서 확인한 암호 입력
```
### 3. Argo CD 원격 클러스터(dev/prod) 가입 등록
```bash
# 원격 타겟 배포 클러스터 최종 등록 연동
argocd cluster add arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-dev-eks
argocd cluster add arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-prod-eks
```

---

## 🔑보안 관리 및 로컬 Secret 수동 생성 가이드
> **⚠️ 보안 수칙 :**
> 데이터베이스 암호 및 그라파나 마스터 패스워드가 포함된 실제 Secret 매니페스트 파일은 절대로 GitHub 저장소에 push하지 않습니다.  
> 배포 전 각 클러스터 타겟팅 콘텍스트로 이동하여 수동 인라인 명령어로 선배포를 완료해야 합니다.

### 1. PetCareLog 애플리케이션 연동 데이터베이스 암호 주입
```bash
# DEV EKS 진입 후 수동 주입
kubectl config use-context arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-dev-eks
kubectl create namespace petcarelog-dev
kubectl create secret generic petcarelog-secret -n petcarelog-dev \
  --from-literal=DB_USERNAME='petcarelog' \
  --from-literal=DB_PASSWORD='실제_DEV_DB_PASSWORD' \
  --from-literal=REDIS_USERNAME='' \
  --from-literal=REDIS_PASSWORD=''

# PROD EKS 진입 후 수동 주입
kubectl config use-context arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-prod-eks
kubectl create namespace petcarelog-prod
kubectl create secret generic petcarelog-secret -n petcarelog-prod \
  --from-literal=DB_USERNAME='petcarelog' \
  --from-literal=DB_PASSWORD='실제_PROD_DB_PASSWORD' \
  --from-literal=REDIS_USERNAME='' \
  --from-literal=REDIS_PASSWORD=''
```

### 2. Grafana 마스터 계정 암호 주입
```bash
# DEV 모니터링 전용 Secret 주입
kubectl config use-context arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-dev-eks
kubectl create namespace monitoring
kubectl create secret generic grafana-admin-secret -n monitoring \
  --from-literal=admin-user='admin' \
  --from-literal=admin-password='실제_DEV_GRAFANA_PASSWORD'

# PROD 모니터링 전용 Secret 주입
kubectl config use-context arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-prod-eks
kubectl create namespace monitoring
kubectl create secret generic grafana-admin-secret -n monitoring \
  --from-literal=admin-user='admin' \
  --from-literal=admin-password='실제_PROD_GRAFANA_PASSWORD'
```

---

## ⚡ 최종 인프라 적용 순서
모든 선언적 자동화 제어 명령은 중앙 Management EKS 콘텍스트 환경에서 일괄 실행합니다.

```bash
# 마스터 콘텍스트 고정 복귀
kubectl config use-context arn:aws:eks:ap-northeast-2:<ACCOUNT_ID>:cluster/team5-petcarelog-management-eks

# 1단계: 접근 제어용 AppProject 적용 규칙 가동
kubectl apply -f argocd/projects/

# 2단계: PetCareLog core 코어 애플리케이션셋 배포 가동
kubectl apply -f argocd/applicationsets/petcarelog-applicationset.yaml

# 3단계: [선행 작업] 프로메테우스 오퍼레이터 CRD 인프라 선배포 및 동기화
kubectl apply -f argocd/applicationsets/monitoring-crds-applicationset.yaml
argocd app sync monitoring-crds-dev
argocd app sync monitoring-crds-prod

# 4단계: [후행 작업] CRD 설치 최종 확인 후 모니터링 통합 본체 패키지 배포 및 동기화
kubectl apply -f argocd/applicationsets/monitoring-applicationset.yaml
argocd app sync monitoring-dev
argocd app sync monitoring-prod
```

---

## 🚨 트러블슈팅 가이드
### Application이 Unknown / Degraded인 경우
> **원인:** Argo CD에서 애플리케이션 상태가 정상적으로 동기화되지 않거나 구동에 실패함.
- **상세 메시지 확인 명령어:**
```bash
argocd app get <APPLICATION_NAME>
# 또는
kubectl describe application <APPLICATION_NAME> -n argocd
```
- **주요 체크리스트:** repoURL 오타 여부, Private 레포지토리 토큰 인증 문제, AppProject destination 및 sourceRepos 명세 불일치, 네임스페이스 권한 누락 여부 검증

### namespace is not permitted in project 에러 발생 시
> **원인:** AppProject가 접근할 수 있도록 허용된 Namespace 범위를 벗어난 곳에 배포를 시도함.
- **해결 방법:** AppProject 제어 설정 파일의 destinations 항목에 해당 대상을 수동 명시해야 합니다.
```YAML
destinations:
  - server: DEV_CLUSTER_SERVER_URL
    namespace: monitoring
  - server: DEV_CLUSTER_SERVER_URL
    namespace: kube-system
```

### unable to resolve parseableType for GroupVersionKind 에러 발생 시
> **원인:** Prometheus Operator CRD(사용자 정의 리소스 정의)가 클러스터에 먼저 설치되지 않은 상태에서 Alertmanager, Prometheus, ServiceMonitor 등의 커스텀 리소스를 비교·배포하려고 했을 때 발생합니다.
- **해결 방법:** 반드시 위의 '최종 인프라 배포 적용 순서' 파이프라인 단계를 엄수하여 CRD 설치셋(3단계)을 완전히 가동 및 동기화한 뒤 본체 세트(4단계)를 통과시키십시오.

### Grafana 대시보드 화면에서 No data가 출력되는 경우
> **원인:** 환경별 콘텍스트(Context) 조회 오류 또는 메트릭 수집 타겟 라우팅 단절.
- **주요 체크리스트:** Dev 그라파나 관제 환경에서 실수로 Prod 네임스페이스를 역조회하고 있지 않은지 확인, 프로메테우스 타겟 수집 상태 확인(kubectl get servicemonitor -A)

### /actuator/prometheus 접속 시 로그인 페이지가 강제 팝업되는 경우
> **원인:** Spring Security 보안 정책에 의해 프로메테우스 메트릭 수집 경로가 차단됨.
- **해결 방법:** Spring Boot 애플리케이션 소스코드의 보안 설정에서 관련 Endpoint가 인가 없이 접근할 수 있도록 permitAll() 처리가 되어야 합니다.
```java
.requestMatchers(
    "/actuator/health",
    "/actuator/info",
    "/actuator/prometheus"
).permitAll()
```

---

## 🔒 AWS 공용 계정 보안 및 이용 수칙
### [Access Key 관리 정책]
  - 본인 키 일시 비활성화 (분실 의심 시 즉시)
  - 본인 키 재활성화

**키 분실·노출 의심 시:**
  1. 즉시 본인이 키 비활성화:
     aws iam update-access-key --user-name team5-ssm \
       --access-key-id AKIAxxx --status Inactive
  2. 강사진에게 즉시 보고

### [보안 수칙]
1. 절대 Access Key를 Git에 push 금지
   .gitignore에 .aws/, *.pem, credentials 등록
2. 의심 시 즉시 키 비활성화
3. MFA 등록 권장

### [사용 안내]
- 공식 사용은 프로젝트 진행일 09:00 - 18:00에만 가능
- 18:00 이후 접근 차단
- 모든 api 호출 기록은 trail에 기록
- 개인적인 용도로 사용 금지
- 비정상적인 접근 및 시스템에 영향을 주는 행위시 법적 책임

### [금지 사항]
- 다른 팀 자원 접근·변경
- 비효율적인 자원 무단 생성 (GPU 인스턴스, 대형 RDS 등)
