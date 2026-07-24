# IBM Maximo Application Suite 9.2 설치 개요

> **상태**: 검토 반영본  
> 이 문서는 MAS 9.2 설치 문서의 공통 기준을 정의합니다. 실제 설치 명령은 [INSTALL_ONLINE.md](INSTALL_ONLINE.md), 폐쇄망 절차는 [INSTALL_OFFLINE.md](INSTALL_OFFLINE.md)를 따릅니다.

---

## 1. 설치 기준 버전

현재 문서는 기존 초안의 IBM Maximo Operator Catalog `v9-250828-amd64`를 유지하는 기준으로 정리합니다. 이 catalog를 사용할 경우 MAS CLI도 같은 시기의 버전으로 고정합니다.

| 구성 요소 | 기준값 | 비고 |
|-----------|--------|------|
| Red Hat OpenShift | `4.16.x` | `v9-250828` catalog 지원 범위 내 버전 |
| IBM Maximo Operator Catalog | `v9-250828-amd64` | `icr.io/cpopen/ibm-maximo-operator-catalog:v9-250828-amd64` |
| MAS CLI 이미지 | `quay.io/ibmmas/cli:15.2.0` | `latest` 사용 금지 |
| MAS Core / Manage | `9.2` | 설치 시 애플리케이션별 channel(예: `9.2.x`) 선택. 단일 "9.2" 채널이 아니라 `--mas-channel`, `--manage-channel` 등 앱별로 개별 지정 |
| MongoDB | `6.0` 또는 `7.0` | MAS CLI / catalog 호환 범위 기준 |
| DB2 | catalog 호환 버전 | MAS CLI 의존성 설치 또는 별도 DB 구성 중 하나를 선택 |

> IBM MAS CLI는 catalog와 CLI 버전 조합에 민감합니다. 최신 CLI를 사용하려면 catalog도 현재 지원 catalog로 다시 선택해야 합니다. 기존 `v9-250828` catalog와 `quay.io/ibmmas/cli:latest` 조합은 설치 실패 가능성이 있습니다.
>
> ⚠️ 미검증
>
> `quay.io/ibmmas/cli:15.2.0`은 `v9-250828` catalog와 같은 시기의 CLI로 정리한 값입니다. 실제 설치 전 IBM MAS CLI catalog 문서와 `mas install --help`, `mas mirror-images --help` 출력으로 조합을 재확인합니다.

참고:

- IBM MAS CLI: <https://ibm-mas.github.io/cli/>
- IBM MAS CLI Catalogs: <https://ibm-mas.github.io/cli/catalogs/>
- IBM MAS CLI Topology: <https://ibm-mas.github.io/cli/reference/topology/>

---

## 2. 배포 아키텍처

```text
[ 사용자 브라우저 ]
        |
[ OpenShift Ingress / Route ]
        |
[ MAS Core / Manage Pods ]
        |
[ DB2 ]   [ MongoDB ]   [ SLS ]

[ Bastion ]
  - DNS
  - oc / kubectl
  - MAS CLI 컨테이너 실행
  - 오프라인 환경에서는 Mirror Registry 운영
```

SNO(Single Node OpenShift)는 컨트롤 플레인과 워커 역할을 단일 노드에서 수행합니다. Bastion은 SNO 노드와 동일 네트워크에 위치하며, 설치 완료 후에도 운영 작업 서버로 사용합니다.

---

## 3. 서버 사양

| 구분 | OS / 상태 | CPU | RAM | 디스크 | 비고 |
|------|-----------|-----|-----|--------|------|
| Bastion | RHEL 9.x 권장 | 4 core 이상 | 8 GB 이상 | 온라인 100 GB 이상 / 오프라인 1 TB 이상 | Docker 또는 Podman, oc CLI, MAS CLI 실행 |
| SNO 노드 | OS 미설치 VM 또는 물리 서버 | 16 core 이상 | 64 GB 이상 | OS 300 GB + 데이터 500 GB 이상 | Discovery ISO / Agent ISO로 RHCOS와 OpenShift 설치 |

Bastion은 설치 작업 서버이므로 RHEL 계열을 권장합니다. SNO 노드는 사전에 RHEL을 설치하는 서버가 아니라, 빈 VM 또는 물리 서버를 ISO로 부팅하여 RHCOS(Red Hat CoreOS)와 OpenShift를 설치하는 대상입니다.

> SNO는 HA 구성이 아닙니다. 운영 환경에서 장애 허용성이 필요하면 3노드 이상 OpenShift 구성을 검토해야 합니다.

---

## 4. 스토리지 원칙

스토리지는 설치 가능 여부를 좌우하는 핵심 조건입니다.

| 용도 | 요구 AccessMode | 권장 예시 |
|------|-----------------|-----------|
| 일반 RWO PVC | `ReadWriteOnce` | LVM Storage, ODF RBD |
| 공유 RWX PVC | `ReadWriteMany` | ODF CephFS, NFS, Portworx 등 |

> LVM Storage는 기본적으로 RWO 스토리지입니다. `Storage Class (RWX)` 값으로 LVM StorageClass(deviceClass `vg1` 기준 실제 이름은 `lvms-vg1`)를 넣으면 설치 또는 런타임에서 실패할 수 있습니다.
>
> IBM MAS SNO 참고 문서에 따르면, SNO는 모든 파드가 단일 노드에서 실행되므로 RWO만으로 충분하며 별도 RWX StorageClass가 필요 없을 수 있습니다. 실제 `mas install` 프롬프트로 RWX 요구 여부를 확인한 뒤 준비 범위를 결정합니다.

---

## 5. 사전 준비 항목

설치 시작 전 아래 항목을 확보합니다.

| 항목 | 용도 |
|------|------|
| IBM Entitlement Key | IBM Container Registry 이미지 pull |
| `entitlement.lic` | SLS AppPoints 라이선스 등록 |
| Red Hat Pull Secret | OpenShift 설치 및 Red Hat 이미지 pull |
| OpenShift cluster-name / base-domain | API, Ingress, wildcard DNS 구성 |
| RWO StorageClass | MAS / DB / 내부 컴포넌트용 |
| RWX StorageClass | 공유 파일시스템이 필요한 컴포넌트용 |

환경별 값은 문서에 직접 고정하지 않고 `<placeholder>` 형식으로 작성합니다.

---

## 6. 네트워크 포트

| 포트 | 방향 | 용도 |
|------|------|------|
| 53/tcp,udp | 클라이언트/SNO → Bastion DNS | DNS |
| 80/tcp | 클라이언트 → OpenShift Router | HTTP redirect / ACME 등 환경별 사용 |
| 443/tcp | 클라이언트 → OpenShift Router | MAS UI, OpenShift Console |
| 6443/tcp | Bastion/관리자 → SNO | OpenShift API |
| 22623/tcp | SNO 내부 | Machine Config Server |
| 5000/tcp | SNO → Bastion | 오프라인 Mirror Registry |
| 50000/tcp | MAS/Manage → DB2 | DB2 |
| 27017/tcp | MAS Core → MongoDB | MongoDB |

> 실제 방화벽 정책은 OpenShift 설치 방식, Mirror Registry 포트, DB 배치 위치에 따라 조정합니다.

---

## 7. 설치 문서 목록

| 문서 | 환경 | 내용 |
|------|------|------|
| [INSTALL_ONLINE.md](INSTALL_ONLINE.md) | 인터넷 연결 가능 | DNS → OCP SNO → 스토리지 → MAS CLI → IT 모듈 → BIRT |
| [INSTALL_OFFLINE.md](INSTALL_OFFLINE.md) | 폐쇄망 | Mirror Registry → 이미지 미러링 → OCP SNO → Airgap → MAS CLI |

---

## 8. 현재 문서에서 검증이 필요한 영역

아래 영역은 공식 문서 또는 실제 클러스터에서 추가 확인 후 운영 절차로 확정합니다.

> ⚠️ 미검증
>
> - `mas install`이 IBM Maximo Operator Catalog(CatalogSource)를 자체적으로 자동 생성/관리하는지 여부. 자동 처리된다면 ONLINE 5.1절 / OFFLINE 6.3절의 수동 `oc apply` 단계는 생략 가능
> - "Subscription Channel" 값이 문서 표기대로 단일 `9.2.x`인지, 애플리케이션별로 서로 다른 채널명을 쓰는지
> - 폐쇄망 이미지 미러링 명령의 세부 옵션은 사용하려는 MAS CLI 버전의 `--help` 출력으로 재확인 필요(이 문서의 옵션명은 CLI 15.2.0 소스코드 기준으로 정정했으나, 실제 실행 전 최종 확인 필요)
> - OFFLINE 문서 5장의 `imageDigestSources` mirror 경로가 실제 `mas mirror-redhat-images` 산출물과 일치하는지 (미러링 완료 후 registry 경로 실측 필요)
> - 외부 DB2 / 외부 MongoDB를 수동으로 구성하는 경우 MAS `JdbcCfg`, `MongoCfg` 연결 절차

다음 항목은 이번 검토에서 공식 문서 대조 결과 **오류로 확인되어 본문에서 정정**했습니다.

- ONLINE 문서의 CatalogSource 이름 오류(`ibm-maximo-operator-catalog` → `ibm-operator-catalog`)
- OFFLINE 문서에 IBM Maximo Operator Catalog의 CatalogSource 생성 절차 자체가 누락되어 있던 문제
- 오프라인 미러링 명령의 옵션명 대부분이 실제 CLI와 다르던 문제(예: `--working-dir`→`--dir`, `--registry-host`→`--host` 등)
- LVM StorageClass 예시 이름 오류(`odf-lvm-vg1` → `lvms-vg1`)
- IT 모듈 활성화 시 `updatedb.sh`/`runscriptfile.sh -cIT -f SETUPIT` 실행 절차 — 온프레미스 Maximo 7.x용 레거시 절차로 확인되어 MAS 9.2 오퍼레이터 흐름에서는 불필요
- BIRT 활성화 시 `manageapp` CR patch 및 `mxe.report.birt.viewerurl=http://localhost:9080/...` — 잘못된 예시로 확인되어 `ManageWorkspace` CR 기반 절차로 정정
