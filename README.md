# ldh-infra-lab-lgtm

ldh-infra-lab 클러스터의 **관측(Observability) 스택**. LGTM(Loki · Grafana · Tempo · Mimir) + 수집기 Alloy.
ArgoCD 가 배포한다. Application 정의는 `ldh-infra-lab-manifests/argocd/apps/` 에 있고, 이 레포는 **values 와 설정**을 가진다.

## 왜 레포를 따로 두나

앱 워크로드 매니페스트(`ldh-infra-lab-manifests`)와 관측 스택을 분리한다.

- **소유 경계**: 앱 워크로드(manifests 레포)와 관측 시스템(이 레포)은 변경 주기·책임이 다르다
- **장애 격리**: 실무에서는 관측 스택을 별도 클러스터에 둔다. 앱 클러스터가 죽어도 "왜 죽었는지"는 보여야 하므로

## 레포 구조

| 폴더 | 무엇 | 실무에서 배포되는 곳 |
|---|---|---|
| `alloy/` | 수집기. 메트릭·로그·트레이스를 모아 백엔드로 push | **앱 클러스터** (노드·파드 옆에 있어야 함) |
| `backend/` | 저장소. SeaweedFS(S3 대용), Mimir, Loki, Tempo | **모니터링 클러스터** |
| `grafana/` | 화면. 데이터소스, 대시보드(JSON) | 모니터링 클러스터 |

```
앱 클러스터                                  모니터링 클러스터
 앱들 ─▶ Alloy ── push (테넌트 헤더) ──▶  Mimir / Loki / Tempo ─▶ SeaweedFS(S3)
                                                  ▲
                                               Grafana
```

## 로컬 랩의 타협 (알고 있는 한계)

| | 실무 | 이 랩 |
|---|---|---|
| 클러스터 | 앱 / 모니터링 분리 | **하나**. `monitoring` ns + AppProject `observability` 로만 격리 → **장애 격리는 안 됨** |
| 배포 모드 | 분산 모드 (역할별 파드, 복제 3) | 모놀리식 (프로세스 하나, 복제 1) |
| 오브젝트 스토리지 | S3 | SeaweedFS (S3 API 호환. MinIO 커뮤니티판은 2025 단종) |
| 보관 기간 | 수십 일 ~ 1년 | 약 7일 |

Phase 9(EKS)에서 클러스터를 나누면 `backend/`·`grafana/` 의 배포 대상과 Alloy 의 push 주소만 바꾸면 되도록 구성한다.

## 원칙

- 수집기는 **push 주소를 인그레스 주소로** 쓴다 (`*.monitoring.lab.local`). 같은 클러스터여도 "다른 클러스터로 보내는" 모양을 유지
- **테넌트 분리**: 프로젝트별(`msa`, `morris`)로 `X-Scope-OrgID` 를 나눈다
- 차트·이미지 버전은 전부 고정. `latest` 금지
- 시크릿은 Sealed Secrets (평문 커밋 금지)
