# PostgreSQL (Kustomize-based)

Kubernetes 환경에서 PostgreSQL을 Kustomize 구조로 배포하기 위한 설정입니다.  
StatefulSet 기반으로 구성되어 있으며, 초기화 SQL(init.sql), 설정 파일(postgresql.conf, pg_hba.conf), Secret 등은 Overlay 방식으로 커스터마이징할 수 있습니다.

---

## 📁 Directory Structure

```
postgres
├── base
│   ├── init.sql               # 최초 1회 실행되는 초기화 SQL
│   ├── kustomization.yaml     # base 리소스 정의
│   ├── pg_hba.conf            # Postgres 인증 설정
│   ├── postgresql.conf        # Postgres 메인 설정파일
│   ├── secret.yaml            # 기본 Secret 템플릿 (Overlay에서 Patch)
│   ├── service.yaml           # Headless Service (ClusterIP None)
│   └── statefulset.yaml       # StatefulSet 정의
└── overlays
    └── kustomization.yaml     # 환경별 overlay 구성
```

---

## ⚙️ 커스터마이징 방법

### 1. 사용자 이름 및 비밀번호 변경

`overlays/kustomization.yaml`에서 다음 항목을 수정하세요:

```
patches:
  - target:
      kind: Secret
      name: postgres-secret
    patch: |
      - op: replace
        path: /stringData
        value:
          POSTGRES_PASSWORD: yourpassword
          POSTGRES_USER: postgres
```

### 2. PostgreSQL 버전 변경

Docker Hub의 공식 PostgreSQL 이미지를 사용하며, 아래 항목에서 버전을 변경할 수 있습니다:

```
images:
  - name: postgres
    newTag: "15"
```

---

## 🗂️ Init SQL (postgres-initdb) 설정

초기화 SQL은 `base/init.sql` 파일입니다.  
컨테이너 최초 실행 시 `/docker-entrypoint-initdb.d/init.sql` 에 마운트되어 **최초 한 번만 실행**됩니다.

예:

```
CREATE DATABASE myapp;
```

> ⚠️ 이미 PVC에 데이터가 존재하면 init.sql은 실행되지 않습니다. 데이터 초기화가 필요하다면 PVC를 삭제해야 합니다.

---

## 🌐 PostgreSQL 접속 방법

### 1️⃣ Pod 내부에서 접속

```
kubectl exec -it postgres-0 -- bash
psql -U postgres
```

### 2️⃣ 다른 Pod에서 접속 (Kubernetes DNS 사용)

Headless Service 이름이 `postgres` 이므로 내부 DNS 주소는:

```
postgres.goormthon-1.svc.cluster.local:5432
```

예:

```
psql -h postgres.goormthon-1.svc.cluster.local:5432 -U postgres -d myapp
```

---

## 🧪 Init 적용 여부 확인

1. Pod 접속

```
kubectl exec -it postgres-0 -- bash
```

2. PostgreSQL 접속

```
psql -U postgres
```

3. DB 목록 확인

```
\l
```

4. init.sql에서 생성한 `myapp` DB 접속

```
\c myapp
```

---

## 📝 주의사항

- StatefulSet 기반이므로 Pod가 삭제되어도 데이터(PVC)는 유지됩니다.
- 데이터를 초기화하려면 다음 명령어로 PVC를 삭제합니다:

```
kubectl delete pvc -l app=postgres -n goormthon-1
```

**⚠️ 이 명령은 데이터를 완전히 삭제합니다.**

- PostgreSQL 버전을 올릴 경우 기존 PVC와 데이터 포맷 충돌이 발생할 수 있으므로 주의해야 합니다.

---

## 📦 배포 방법

```
kubectl apply -k database/postgres/overlays
```

배포 전 실제 생성될 리소스를 확인하려면:

```
kustomize build database/postgres/overlays
```

---

## 📌 연결 문자열 예시

```
postgres://postgres:<password>@postgres.goormthon-1.svc.cluster.local:5432/myapp
```

---
