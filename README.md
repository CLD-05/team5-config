# PetCareLog Config

> PetCareLog 서비스의 Kubernetes Manifest, Argo CD ApplicationSet, 모니터링 설정을 관리하는 GitOps 설정 저장소입니다.  
> 이 저장소는 dev/prod 환경의 애플리케이션 배포와 Prometheus/Grafana 기반 모니터링 구성을 선언적으로 관리합니다.

<br>

## 📌 프로젝트 개요

`team5-config`는 PetCareLog 프로젝트의 **배포 설정 전용 저장소**입니다.

애플리케이션 소스 코드는 `team5-app`, AWS 인프라 코드는 `team5-infra`, Kubernetes 배포 설정은 `team5-config`로 분리하여 관리합니다.  
이 저장소는 Argo CD가 바라보는 **Single Source of Truth** 역할을 하며, Git에 선언된 상태를 기준으로 dev/prod EKS 클러스터에 애플리케이션과 모니터링 리소스를 배포합니다.

<br>

## 목차

- [프로젝트 개요](#overview)
- [저장소 역할](#repository-role)
- [전체 아키텍처](#architecture)
- [기술 스택](#tech-stack)
- [디렉토리 구조](#directory-structure)
- [사전 요구 사항](#prerequisites)
- [환경 구성](#environment-config)
- [설치 및 적용 방법](#installation)
- [사용법](#usage)
- [Argo CD 구성](#argocd)
- [Kustomize 구성](#kustomize)
- [모니터링 구성](#monitoring)
- [Grafana Dashboard](#grafana-dashboard)
- [배포 흐름](#deployment-flow)
- [운영 시 주의 사항](#notes)
- [저자 및 기여자](#contributors)

<br>

<a id="repository-role"></a>

## ✅ 저장소 역할

PetCareLog는 역할에 따라 저장소를 분리하여 관리합니다.

| 저장소 | 역할 |
|---|---|
| [`team5-app`](https://github.com/CLD-05/team5-app) | Spring Boot 애플리케이션, Dockerfile, GitHub Actions |
| [`team5-infra`](https://github.com/CLD-05/team5-infra) | Terraform 기반 AWS 인프라 관리 |
| [`team5-config`](https://github.com/CLD-05/team5-config) | Kubernetes Manifest, Argo CD GitOps 배포 설정 |

<br>

`team5-config`는 다음 리소스를 관리합니다.

| 구분 | 관리 대상 |
|---|---|
| Application Manifest | Deployment, Service, Ingress, ConfigMap, ServiceAccount, HPA, ServiceMonitor |
| GitOps | Argo CD AppProject, ApplicationSet |
| Environment Overlay | dev/prod 환경별 Kustomize overlay |
| Monitoring | kube-prometheus-stack Helm values, Grafana dashboard JSON |

<br>

<a id="architecture"></a>

## 🧭 전체 아키텍처

이 프로젝트는 management EKS에 설치된 Argo CD가 dev/prod EKS 클러스터를 관리하는 구조입니다.

```text
team5-config Repository
        │
        ▼
Management EKS
└── Argo CD
    ├── petcarelog-dev        → Dev EKS / petcarelog-dev namespace
    ├── petcarelog-prod       → Prod EKS / petcarelog-prod namespace
    ├── monitoring-crds-dev   → Dev EKS / monitoring namespace
    ├── monitoring-crds-prod  → Prod EKS / monitoring namespace
    ├── monitoring-dev        → Dev EKS / monitoring namespace
    └── monitoring-prod       → Prod EKS / monitoring namespace
```

<br>

### 배포 환경

| 환경 | Namespace | 주요 용도 |
|---|---|---|
| dev | `petcarelog-dev` | 개발 및 테스트 배포 |
| prod | `petcarelog-prod` | 운영 배포 |
| monitoring | `monitoring` | Prometheus, Grafana, Alertmanager |

<br>

<a id="tech-stack"></a>

## 🛠 기술 스택

| 구분 | 기술 |
|---|---|
| Container Orchestration | Kubernetes, Amazon EKS |
| GitOps | Argo CD, ApplicationSet |
| Manifest 관리 | Kustomize |
| Ingress | AWS Load Balancer Controller, ALB Ingress |
| Auto Scaling | HorizontalPodAutoscaler |
| Monitoring | Prometheus, Grafana, Alertmanager |
| Metrics | ServiceMonitor, Spring Boot Actuator, Micrometer |
| Dashboard | Grafana Dashboard JSON |
| Secret 관리 | Kubernetes Secret, Secret Example 파일 |
| AWS 권한 | IRSA, ServiceAccount Annotation |

<br>

<a id="directory-structure"></a>

## 📁 디렉토리 구조

```text
team5-config
├── apps
│   └── petcarelog
│       ├── base
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   ├── ingress.yaml
│       │   ├── servicemonitor.yaml
│       │   └── kustomization.yaml
│       │
│       └── overlays
│           ├── dev
│           │   ├── namespace.yaml
│           │   ├── serviceaccount.yaml
│           │   ├── config.yaml
│           │   ├── hpa.yaml
│           │   ├── patch-deployment.yaml
│           │   ├── patch-ingress.yaml
│           │   ├── patch-servicemonitor.yaml
│           │   ├── secret.example.yaml
│           │   └── kustomization.yaml
│           │
│           └── prod
│               ├── namespace.yaml
│               ├── serviceaccount.yaml
│               ├── config.yaml
│               ├── hpa.yaml
│               ├── patch-deployment.yaml
│               ├── patch-ingress.yaml
│               ├── patch-servicemonitor.yaml
│               ├── secret.example.yaml
│               └── kustomization.yaml
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
    ├── grafana_dashboards
    │   ├── dev
    │   │   ├── petcarelog-application-dev.json
    │   │   ├── petcarelog-kubernetes-dev.json
    │   │   └── petcarelog-loadtest-dev.json
    │   │
    │   └── prod
    │       ├── petcarelog-application-prod.json
    │       ├── petcarelog-kubernetes-prod.json
    │       └── petcarelog-loadtest-prod.json
    │
    └── kube-prometheus-stack
        ├── values-dev.yaml
        └── values-prod.yaml
```

<br>

<a id="prerequisites"></a>

## ✅ 사전 요구 사항

이 저장소를 적용하기 전에 다음 구성이 필요합니다.

| 항목 | 설명 |
|---|---|
| AWS CLI | EKS 클러스터 인증 및 kubeconfig 설정 |
| kubectl | Kubernetes 리소스 적용 및 확인 |
| kustomize | overlay manifest 렌더링 및 검증 |
| Helm | kube-prometheus-stack Chart 확인 시 사용 |
| Argo CD CLI | Argo CD Application 확인 및 동기화 |
| EKS Cluster | management, dev, prod 클러스터 구성 |
| AWS Load Balancer Controller | Ingress 기반 ALB 생성 |
| Prometheus Operator CRD | ServiceMonitor 등 모니터링 리소스 사용 |
| metrics-server | HPA CPU 기반 오토스케일링 사용 |

<br>

### kubeconfig 설정 예시

```bash
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name team5-petcarelog-management-eks

aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name team5-petcarelog-dev-eks

aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name team5-petcarelog-prod-eks
```

<br>

<a id="environment-config"></a>

## ⚙️ 환경 구성

### 애플리케이션 ConfigMap

각 환경의 비민감 설정은 `config.yaml`에서 관리합니다.

| 파일 | 설명 |
|---|---|
| `apps/petcarelog/overlays/dev/config.yaml` | dev RDS, Redis, S3 설정 |
| `apps/petcarelog/overlays/prod/config.yaml` | prod RDS, Redis, S3 설정 |

<br>

ConfigMap에서 관리하는 값은 다음과 같습니다.

| 환경 변수 | 설명 |
|---|---|
| `DB_HOST` | RDS MySQL Endpoint |
| `DB_PORT` | MySQL 포트 |
| `DB_NAME` | 데이터베이스 이름 |
| `REDIS_HOST` | ElastiCache Redis Endpoint |
| `REDIS_PORT` | Redis 포트 |
| `REDIS_SSL_ENABLED` | Redis SSL 사용 여부 |
| `AWS_REGION` | AWS 리전 |
| `S3_BUCKET_NAME` | 반려동물 이미지 저장 S3 버킷 |
| `S3_PET_IMAGE_PREFIX` | S3 이미지 저장 prefix |

<br>

### 애플리케이션 Secret

민감 정보는 Git에 직접 올리지 않고, `secret.example.yaml`을 참고하여 별도로 생성합니다.

| 파일 | 설명 |
|---|---|
| `apps/petcarelog/overlays/dev/secret.example.yaml` | dev Secret 예시 |
| `apps/petcarelog/overlays/prod/secret.example.yaml` | prod Secret 예시 |

<br>

Secret에서 관리하는 값은 다음과 같습니다.

| 키 | 설명 |
|---|---|
| `DB_USERNAME` | DB 사용자명 |
| `DB_PASSWORD` | DB 비밀번호 |
| `REDIS_USERNAME` | Redis 사용자명 |
| `REDIS_PASSWORD` | Redis 비밀번호 |

> 실제 DB 비밀번호, Redis 비밀번호, 인증 정보는 README나 Git 저장소에 작성하지 않습니다.

<br>

### Grafana Admin Secret

`kube-prometheus-stack` values 파일은 Grafana 관리자 계정을 기존 Secret에서 읽도록 설정되어 있습니다.

| Secret 이름 | Key | 설명 |
|---|---|---|
| `grafana-admin-secret` | `admin-user` | Grafana 관리자 ID |
| `grafana-admin-secret` | `admin-password` | Grafana 관리자 비밀번호 |

예시:

```bash
kubectl create secret generic grafana-admin-secret \
  -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<CHANGE_ME>'
```

<br>

<a id="installation"></a>

## 🚀 설치 및 적용 방법

### 1. 저장소 클론

```bash
git clone https://github.com/CLD-05/team5-config.git
cd team5-config
```

<br>

### 2. dev/prod manifest 렌더링 확인

적용 전 Kustomize 결과를 먼저 확인합니다.

```bash
kubectl kustomize apps/petcarelog/overlays/dev
kubectl kustomize apps/petcarelog/overlays/prod
```

또는 kustomize CLI를 사용하는 경우:

```bash
kustomize build apps/petcarelog/overlays/dev
kustomize build apps/petcarelog/overlays/prod
```

<br>

### 3. 애플리케이션 Secret 생성

`secret.example.yaml`을 참고하여 실제 Secret을 생성합니다.

```bash
kubectl create secret generic petcarelog-secret \
  -n petcarelog-dev \
  --from-literal=DB_USERNAME='<DEV_DB_USERNAME>' \
  --from-literal=DB_PASSWORD='<DEV_DB_PASSWORD>' \
  --from-literal=REDIS_USERNAME='' \
  --from-literal=REDIS_PASSWORD=''
```

```bash
kubectl create secret generic petcarelog-secret \
  -n petcarelog-prod \
  --from-literal=DB_USERNAME='<PROD_DB_USERNAME>' \
  --from-literal=DB_PASSWORD='<PROD_DB_PASSWORD>' \
  --from-literal=REDIS_USERNAME='' \
  --from-literal=REDIS_PASSWORD=''
```

> Argo CD가 Application을 먼저 생성하는 경우 namespace가 자동 생성될 수 있습니다.  
> 수동으로 Secret을 먼저 생성해야 한다면 namespace를 먼저 생성하세요.

```bash
kubectl create namespace petcarelog-dev
kubectl create namespace petcarelog-prod
```

<br>

### 4. Grafana Admin Secret 생성

각 환경의 monitoring namespace에 Grafana 관리자 Secret을 생성합니다.

```bash
kubectl create namespace monitoring

kubectl create secret generic grafana-admin-secret \
  -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<CHANGE_ME>'
```

<br>

### 5. Argo CD Project 적용

management EKS의 Argo CD namespace에 AppProject를 적용합니다.

```bash
kubectl apply -f argocd/projects/petcarelog-dev-project.yaml
kubectl apply -f argocd/projects/petcarelog-prod-project.yaml
kubectl apply -f argocd/projects/monitoring-project.yaml
```

<br>

### 6. Monitoring CRD 먼저 배포

Prometheus Operator CRD가 먼저 설치되어야 ServiceMonitor, PrometheusRule 등의 리소스를 정상적으로 인식할 수 있습니다.

```bash
kubectl apply -f argocd/applicationsets/monitoring-crds-applicationset.yaml
```

<br>

### 7. Monitoring Stack 배포

CRD 배포가 완료된 후 kube-prometheus-stack 본체를 배포합니다.

```bash
kubectl apply -f argocd/applicationsets/monitoring-applicationset.yaml
```

<br>

### 8. PetCareLog ApplicationSet 배포

```bash
kubectl apply -f argocd/applicationsets/petcarelog-applicationset.yaml
```

<br>

<a id="usage"></a>

## 🧪 사용법

### dev 환경 리소스 확인

```bash
kubectl get all -n petcarelog-dev
kubectl get ingress -n petcarelog-dev
kubectl get hpa -n petcarelog-dev
```

<br>

### prod 환경 리소스 확인

```bash
kubectl get all -n petcarelog-prod
kubectl get ingress -n petcarelog-prod
kubectl get hpa -n petcarelog-prod
```

<br>

### 애플리케이션 Pod 로그 확인

```bash
kubectl logs -n petcarelog-dev deploy/petcarelog
kubectl logs -n petcarelog-prod deploy/petcarelog
```

<br>

### HPA 확인

```bash
kubectl get hpa -n petcarelog-dev
kubectl get hpa -n petcarelog-prod
```

실시간으로 확인하려면 다음 명령어를 사용합니다.

```bash
kubectl get hpa -n petcarelog-prod -w
```

<br>

### Grafana 접속용 Port Forward

```bash
kubectl port-forward -n monitoring svc/monitoring-dev-grafana 3000:80
```

또는 prod Grafana 서비스 이름에 맞춰 다음과 같이 접속합니다.

```bash
kubectl port-forward -n monitoring svc/monitoring-prod-grafana 3000:80
```

브라우저에서 접속합니다.

```text
http://localhost:3000
```

<br>

<a id="argocd"></a>

## 🐙 Argo CD 구성

### AppProject

AppProject는 Application이 접근할 수 있는 Git 저장소, 클러스터, namespace, 리소스 범위를 제한합니다.

| 파일 | 설명 |
|---|---|
| `argocd/projects/petcarelog-dev-project.yaml` | dev 애플리케이션 배포 권한 범위 |
| `argocd/projects/petcarelog-prod-project.yaml` | prod 애플리케이션 배포 권한 범위 |
| `argocd/projects/monitoring-project.yaml` | dev/prod 모니터링 배포 권한 범위 |

<br>

### ApplicationSet

ApplicationSet은 dev/prod 환경의 Argo CD Application을 자동 생성합니다.

| 파일 | 생성 Application | 설명 |
|---|---|---|
| `petcarelog-applicationset.yaml` | `petcarelog-dev`, `petcarelog-prod` | 애플리케이션 배포 |
| `monitoring-crds-applicationset.yaml` | `monitoring-crds-dev`, `monitoring-crds-prod` | Prometheus Operator CRD 배포 |
| `monitoring-applicationset.yaml` | `monitoring-dev`, `monitoring-prod` | kube-prometheus-stack 배포 |

<br>

<a id="kustomize"></a>

## 🧩 Kustomize 구성

공통 Kubernetes 리소스는 `base`에 정의하고, 환경별 차이는 `overlays/dev`, `overlays/prod`에서 patch합니다.

```text
base
├── Deployment
├── Service
├── Ingress
└── ServiceMonitor

Overlays
├── dev
│   ├── ConfigMap
│   ├── ServiceAccount
│   ├── HPA
│   └── Patch
│
└── prod
    ├── ConfigMap
    ├── ServiceAccount
    ├── HPA
    └── Patch
```

<br>

### dev overlay

| 항목 | 설정 |
|---|---|
| Namespace | `petcarelog-dev` |
| Replica | 1 |
| HPA | min 1 / max 3 |
| CPU target | 30% |
| Ingress | dev ALB |
| ServiceMonitor label | `release: monitoring-dev` |
| S3 권한 | dev IRSA Role |

<br>

### prod overlay

| 항목 | 설정 |
|---|---|
| Namespace | `petcarelog-prod` |
| Replica | 2 |
| HPA | min 2 / max 3 |
| CPU target | 30% |
| Ingress | prod ALB + HTTPS |
| Domain | `petcarelog.yuhyun-lab.cloud` |
| ServiceMonitor label | `release: monitoring-prod` |
| S3 권한 | prod IRSA Role |

<br>

<a id="monitoring"></a>

## 📊 모니터링 구성

모니터링은 `kube-prometheus-stack`을 사용합니다.

| 구성 요소 | 역할 |
|---|---|
| Prometheus | 메트릭 수집 및 저장 |
| Grafana | 메트릭 시각화 |
| Alertmanager | 알림 관리 |
| Prometheus Operator | Prometheus 관련 CRD 관리 |
| kube-state-metrics | Kubernetes 리소스 상태 메트릭 수집 |
| Node Exporter | 노드 CPU, Memory, Disk 등 시스템 메트릭 수집 |
| ServiceMonitor | PetCareLog `/actuator/prometheus` 수집 대상 등록 |

<br>

### values 파일

| 파일 | 설명 |
|---|---|
| `monitoring/kube-prometheus-stack/values-dev.yaml` | dev 모니터링 설정 |
| `monitoring/kube-prometheus-stack/values-prod.yaml` | prod 모니터링 설정 |

<br>

### dev/prod 차이

| 항목 | dev | prod |
|---|---:|---:|
| Prometheus Retention | 7d | 15d |
| Prometheus CPU Request | 200m | 300m |
| Prometheus Memory Request | 512Mi | 1Gi |
| Prometheus CPU Limit | 500m | 700m |
| Prometheus Memory Limit | 1Gi | 2Gi |
| Grafana PVC | 5Gi | 5Gi |

<br>

<a id="grafana-dashboard"></a>

## 📈 Grafana Dashboard

Grafana Dashboard JSON은 환경별로 분리하여 관리합니다.

```text
monitoring/grafana_dashboards
├── dev
│   ├── petcarelog-application-dev.json
│   ├── petcarelog-kubernetes-dev.json
│   └── petcarelog-loadtest-dev.json
│
└── prod
    ├── petcarelog-application-prod.json
    ├── petcarelog-kubernetes-prod.json
    └── petcarelog-loadtest-prod.json
```

<br>

### Dashboard 종류

| Dashboard | 설명 |
|---|---|
| Application Dashboard | JVM, HTTP 요청 수, 응답 시간, DB 커넥션 등 애플리케이션 지표 확인 |
| Kubernetes Dashboard | Pod, Deployment, Namespace, 리소스 사용량 확인 |
| Load Test Dashboard | 부하 테스트 시 요청 수, Pod 증가, HPA 동작, 응답 시간 변화 확인 |

<br>

### 부하 테스트 시 확인할 지표

| 지표 | 확인 목적 |
|---|---|
| HTTP Request Rate | 트래픽 증가 여부 확인 |
| Response Time | 부하 상황에서 응답 시간 변화 확인 |
| 5xx Error Rate | 장애 또는 서버 오류 발생 여부 확인 |
| JVM Heap Usage | 애플리케이션 메모리 사용량 확인 |
| Pod Replicas | HPA에 의한 Pod 확장 확인 |
| CPU Usage | HPA 기준 지표 확인 |

<br>

<a id="deployment-flow"></a>

## 🔄 배포 흐름

```text
1. 개발자가 team5-app에 코드 push
2. GitHub Actions 실행
3. Docker 이미지 빌드
4. Amazon ECR에 이미지 push
5. team5-config의 overlay 이미지 태그 수정
6. Argo CD가 Git 변경사항 감지
7. dev/prod EKS에 Kubernetes 리소스 동기화
8. ServiceMonitor를 통해 Prometheus가 메트릭 수집
9. Grafana에서 애플리케이션과 클러스터 상태 확인
```

<br>

### 이미지 태그 변경 위치

| 환경 | 파일 |
|---|---|
| dev | `apps/petcarelog/overlays/dev/patch-deployment.yaml` |
| prod | `apps/petcarelog/overlays/prod/patch-deployment.yaml` |

<br>

<a id="notes"></a>

## ⚠️ 운영 시 주의 사항

### 1. Secret은 Git에 커밋하지 않기

`secret.example.yaml`은 예시 파일입니다. 실제 DB 비밀번호, Redis 비밀번호, Grafana 관리자 비밀번호는 Git에 커밋하지 않습니다.

<br>

### 2. CRD를 먼저 배포하기

`ServiceMonitor`는 Kubernetes 기본 리소스가 아니라 Prometheus Operator CRD입니다.  
따라서 `monitoring-crds-applicationset.yaml`을 먼저 배포한 뒤 `monitoring-applicationset.yaml`을 배포해야 합니다.

<br>

### 3. ServiceMonitor label 확인하기

Prometheus가 ServiceMonitor를 수집하려면 Helm release label이 맞아야 합니다.

| 환경 | label |
|---|---|
| dev | `release: monitoring-dev` |
| prod | `release: monitoring-prod` |

<br>

### 4. HPA 사용 전 metrics-server 확인하기

HPA가 CPU 사용률 기준으로 동작하려면 metrics-server가 정상 동작해야 합니다.

```bash
kubectl top nodes
kubectl top pods -n petcarelog-dev
kubectl top pods -n petcarelog-prod
```

<br>

### 5. prod Ingress는 HTTPS 설정 확인하기

prod Ingress는 HTTPS 리스너와 ACM 인증서를 사용합니다.  
도메인, 인증서 ARN, Route 53 레코드가 올바르게 연결되어 있어야 합니다.

<br>

## 🔍 유용한 명령어

### Argo CD Application 확인

```bash
argocd app list
```

<br>

### 특정 Application 동기화

```bash
argocd app sync petcarelog-dev
argocd app sync petcarelog-prod
```

<br>

### Ingress 주소 확인

```bash
kubectl get ingress -n petcarelog-dev
kubectl get ingress -n petcarelog-prod
```

<br>

### Pod 상태 확인

```bash
kubectl get pods -n petcarelog-dev
kubectl get pods -n petcarelog-prod
```

<br>

### ServiceMonitor 확인

```bash
kubectl get servicemonitor -n petcarelog-dev
kubectl get servicemonitor -n petcarelog-prod
```

<br>

<a id="contributors"></a>

## 👥 저자 및 기여자

| 이름 | 역할 |
|---|---|
| 김유현 | 팀장 |
| 고윤성 | 팀원 |
| 이재윤 | 팀원 |
| 유관호 | 팀원 |
| 신솔미 | 팀원 |
| 김광호 | 팀원 |

> 실제 팀원 이름과 담당 역할에 맞게 수정해 주세요.

<br>

## 📌 참고

- 이 저장소는 애플리케이션 소스 코드 저장소가 아닙니다.
- 애플리케이션 코드는 `team5-app`에서 관리합니다.
- AWS 인프라 코드는 `team5-infra`에서 관리합니다.
- Kubernetes 배포 상태는 이 저장소의 manifest를 기준으로 Argo CD가 동기화합니다.

