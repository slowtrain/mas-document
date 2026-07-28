# IBM Maximo Application Suite 9.2 설치 개요

> **상태**: 검토 반영본  
> 이 문서는 MAS 9.2 설치 문서의 공통 기준을 정의합니다. 실제 설치 명령은 [INSTALL_ONLINE.md](INSTALL_ONLINE.md), 폐쇄망 절차는 [INSTALL_OFFLINE.md](INSTALL_OFFLINE.md)를 따릅니다.

---

## 1. 설치 기준 버전

현재 문서는 IBM 공식 MAS CLI 문서의 MAS 9.2 예시 기준인 IBM Maximo Operator Catalog `v9-260625-amd64`와 MAS CLI `23.4.1` 조합을 기준으로 정리합니다.

| 구성 요소 | 기준값 | 비고 |
|-----------|--------|------|
| Red Hat OpenShift | `4.16` - `4.21` 범위 내 지원 버전 | `v9-260625-amd64` catalog 지원 범위. 실제 설치는 검증된 정확한 patch 버전으로 고정 |
| IBM Maximo Operator Catalog | `v9-260625-amd64` | `icr.io/cpopen/ibm-maximo-operator-catalog:v9-260625-amd64` |
| MAS CLI 이미지 | `quay.io/ibmmas/cli:23.4.1` | `latest` 사용 금지 |
| MAS Core / Manage / Maximo IT | `9.2.x` | Maximo IT는 독립 애플리케이션이 아니라 Maximo Manage의 IT add-on |
| MongoDB | `7.0` 또는 `8.0` | `v9-260625-amd64` catalog 호환 범위 기준 |
| DB2 | catalog 호환 버전 | MAS CLI 의존성 설치 또는 별도 DB 구성 중 하나를 선택 |

IBM MAS CLI는 catalog와 CLI 버전 조합에 민감합니다. MAS 9.2 / Maximo IT 9.2 설치 문서에서는 catalog, CLI, OpenShift, mirror 결과물을 같은 기준으로 고정합니다.

> 기존 초안의 `v9-250828-amd64` catalog와 `quay.io/ibmmas/cli:15.2.0` 조합은 같은 시기의 조합이지만, IBM catalog 문서 기준으로 MAS 9.1 계열 catalog입니다. MAS 9.2 / Maximo IT 9.2 설치 기준으로는 사용하지 않습니다.

참고:

- IBM MAS CLI: <https://ibm-mas.github.io/cli/>
- IBM MAS CLI Catalogs: <https://ibm-mas.github.io/cli/catalogs/>
- IBM MAS CLI Image Mirroring: <https://ibm-mas.github.io/cli/guides/image-mirroring/>
- IBM Maximo IT 설치: <https://www.ibm.com/docs/en/max-it/cd.0.0_cd?topic=installing-maximo-it-in-maximo-application-suite>
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

설치 시작 전 아래 항목을 확보합니다. 실제 비밀번호, IBM Entitlement Key, Pull Secret 원문은 문서에 직접 기록하지 않고 설치 시 환경 변수, 프롬프트, 보안 저장소를 통해 입력합니다.

### 5.1 공통 필수 항목

| 항목 | 확보 위치 / 형식 | 용도 |
|------|------------------|------|
| IBM Entitlement Key | IBM Container Software Library: <https://myibm.ibm.com/products-services/containerlibrary> / 문자열 | IBM Entitled Registry에서 MAS Core, Maximo Manage, Maximo IT, 공통 구성요소 이미지 Pull |
| MAS 라이선스 파일 | IBM License Key Center / `entitlement.lic` 또는 `license.dat` | SLS(Suite License Service) AppPoints 라이선스 등록. Maximo IT AppPoints 포함 필요 |
| Red Hat Pull Secret | Red Hat Hybrid Cloud Console: <https://console.redhat.com/openshift/install/pull-secret> / `pull-secret.json` | OpenShift 설치와 Red Hat Operator 이미지 Pull |
| Maximo IT 권한 | 구매 계약 / AppPoints | Maximo IT는 별도 AppPoints 권한이 필요하며, 신규 설치는 license key 파일에 포함하고 기존 MAS 사용자는 기존 license key 파일에 추가 반영 |
| Maximo IT / Manage 데이터베이스 | Db2, Oracle, SQL Server 등 지원 DB | Maximo IT는 Maximo Manage와 함께 배포/활성화되므로 Manage 데이터베이스 준비 필요 |
| OpenShift cluster-name / base-domain | 환경별 결정값 | API, Ingress, wildcard DNS 구성 |
| Bastion / SNO IP 정보 | 환경별 결정값 | DNS, Agent ISO, 방화벽, 라우팅 구성 |
| RWO StorageClass | OpenShift 스토리지 구성 후 확인 | MAS / DB / 내부 컴포넌트용 PVC |
| RWX StorageClass | 필요한 경우 별도 준비 | 공유 파일시스템이 필요한 컴포넌트용 PVC |

IBM Entitlement Key는 파일이 아니라 문자열입니다. 이미지 Pull 권한 확인은 인터넷이 가능한 작업 PC에서 다음 형태로 수행합니다.

```bash
podman login cp.icr.io -u cp
```

Password 입력란에 IBM Entitlement Key를 입력합니다.

Maximo IT는 MAS 카탈로그에서 Manage를 선택한 뒤 IT add-on을 함께 배포하거나, Manage 배포 후 별도 단계로 추가합니다. 활성화 단계에서 Manage with IT 구성이 적용되므로, 사전 준비 단계에서 Maximo IT AppPoints와 Manage 데이터베이스 준비 여부를 함께 확인합니다.

Red Hat Pull Secret은 다운로드 후 권한을 제한합니다.

```bash
chmod 600 pull-secret.json
```

### 5.2 오프라인 설치 반입 항목

폐쇄망 설치에서는 인터넷 PC에서 아래 항목을 먼저 확보한 뒤 이동식 디스크, 내부 파일 서버, SCP 등 승인된 방식으로 Bastion에 반입합니다.

| 항목 | 예시 파일 / 디렉터리 | 비고 |
|------|----------------------|------|
| MAS CLI 이미지 | `<offline-dir>/cli/` 또는 `mas-cli-23.4.1.tar` | `quay.io/ibmmas/cli:23.4.1` 기준 |
| MAS Core / Catalog 이미지 | `<offline-dir>/core/` | `--mirror-catalog`, `--mirror-core` 대상 |
| Maximo Manage / Maximo IT 이미지 | `<offline-dir>/apps/` | Manage는 `--mirror-manage`, Maximo IT는 `--mirror-icd` 대상 |
| MAS 공통 의존 이미지 | `<offline-dir>/other/` | MongoDB, SLS, TSM, CFS 등 |
| DB2 이미지 | `<offline-dir>/other/` | OpenShift 내부 DB2를 사용할 경우 `--mirror-db2` 포함. 외부 DB를 쓰면 제외 가능 |
| Red Hat/OpenShift 이미지 | `<offline-dir>/redhat/` | OpenShift Platform 이미지와 Red Hat certified/community/redhat operator catalog의 MAS 필요 항목 |
| OpenShift 설치 도구 | `openshift-client-linux.tar.gz`, `openshift-install-linux.tar.gz`, `oc-mirror.tar.gz` | 설치할 OpenShift 정확한 패치 버전 기준으로 다운로드 |
| 컨테이너 런타임 패키지 | Docker 또는 Podman RPM 및 의존 패키지 | Bastion이 인터넷에 접속할 수 없는 경우 필요 |

오프라인 반입 후 Bastion 기준 권장 디렉터리 예시는 다음과 같습니다.

```text
/home/<bastion-user>/mas-install/
  licenses/
    entitlement.lic
  redhat/
    pull-secret.json
  ocp/
    openshift-client-linux.tar.gz
    openshift-install-linux.tar.gz
    oc-mirror.tar.gz
  images/
    mas-cli-23.4.1.tar
  mirror/
    cli/
    core/
    apps/
    other/
    redhat/
  docker-rpms/
    *.rpm
```

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

> ⚠️ 미검증
>
> 위 상세 설치 문서에는 기존 초안 기준인 `v9-250828-amd64` / `quay.io/ibmmas/cli:15.2.0` 명령이 남아 있습니다. MAS 9.2 / Maximo IT 9.2 기준으로 실제 실행하기 전 `v9-260625-amd64` / `quay.io/ibmmas/cli:23.4.1` 조합에 맞춰 함께 갱신해야 합니다.

---

## 8. 현재 문서에서 검증이 필요한 영역

아래 영역은 공식 문서 또는 실제 클러스터에서 추가 확인 후 운영 절차로 확정합니다.

> ⚠️ 미검증
>
> - `mas install`이 IBM Maximo Operator Catalog(CatalogSource)를 자체적으로 자동 생성/관리하는지 여부. 자동 처리된다면 ONLINE 5.1절 / OFFLINE 6.3절의 수동 `oc apply` 단계는 생략 가능
> - "Subscription Channel" 값이 문서 표기대로 단일 `9.2.x`인지, 애플리케이션별로 서로 다른 채널명을 쓰는지
> - 폐쇄망 이미지 미러링 명령의 세부 옵션은 `quay.io/ibmmas/cli:23.4.1 mas mirror-images --help`, `mas mirror-redhat-images --help` 출력으로 재확인 필요
> - OFFLINE 문서 5장의 `imageDigestSources` mirror 경로가 실제 `mas mirror-redhat-images` 산출물과 일치하는지 (미러링 완료 후 registry 경로 실측 필요)
> - 외부 DB2 / 외부 MongoDB를 수동으로 구성하는 경우 MAS `JdbcCfg`, `MongoCfg` 연결 절차

다음 항목은 이번 검토에서 공식 문서 대조 결과 **오류로 확인되어 본문에서 정정**했습니다.

- ONLINE 문서의 CatalogSource 이름 오류(`ibm-maximo-operator-catalog` → `ibm-operator-catalog`)
- OFFLINE 문서에 IBM Maximo Operator Catalog의 CatalogSource 생성 절차 자체가 누락되어 있던 문제
- 오프라인 미러링 명령의 옵션명 대부분이 실제 CLI와 다르던 문제(예: `--working-dir`→`--dir`, `--registry-host`→`--host` 등)
- LVM StorageClass 예시 이름 오류(`odf-lvm-vg1` → `lvms-vg1`)
- IT 모듈 활성화 시 `updatedb.sh`/`runscriptfile.sh -cIT -f SETUPIT` 실행 절차 — 온프레미스 Maximo 7.x용 레거시 절차로 확인되어 MAS 9.2 오퍼레이터 흐름에서는 불필요
- BIRT 활성화 시 `manageapp` CR patch 및 `mxe.report.birt.viewerurl=http://localhost:9080/...` — 잘못된 예시로 확인되어 `ManageWorkspace` CR 기반 절차로 정정
