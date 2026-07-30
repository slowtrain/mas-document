# IBM Maximo Application Suite 9.2 / Maximo IT 설치 환경 개요

> 실제 비밀번호, 토큰, IBM Entitlement Key, Red Hat Pull Secret 및 라이선스 원문은 이 문서에 기록하지 않습니다. 값은 Git에서 제외된 로컬 보안 디렉터리 또는 조직의 보안 저장소에서 관리합니다.
>
> - 실제 설치 절차(명령어 단위)는 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md)를 따릅니다.
> - 이번 배포의 실제 서버 접속 정보는 [SERVER_INFO.md](SERVER_INFO.md)를 참고합니다.
> - 이 문서는 (1) 환경 전반의 기준값/아키텍처 참고 자료와 (2) 배포별로 채워 넣는 입력 시트, 두 부분으로 구성됩니다.

---

## 1. 설치 기준 버전

| 구성 요소 | 기준값 | 비고 |
|-----------|--------|------|
| Red Hat OpenShift | `4.20.30` | `v9-260625-amd64` 지원 범위는 `4.16`–`4.21`; 본 설치는 `4.20.30`으로 고정 |
| IBM Maximo Operator Catalog | `v9-260625-amd64` | `icr.io/cpopen/ibm-maximo-operator-catalog:v9-260625-amd64` |
| MAS CLI 이미지 | `quay.io/ibmmas/cli:23.4.1` | `latest` 사용 금지 |
| MAS Core / Manage / Maximo IT | `9.2.x` | Maximo IT는 독립 애플리케이션이 아니라 Maximo Manage의 IT add-on |
| MongoDB | `7.0` 또는 `8.0` | `v9-260625-amd64` catalog 호환 범위 기준 |
| DB2 | catalog 호환 버전 | 이번 배포는 내부(in-cluster) DB2 사용 |

참고 문서:

- [IBM MAS CLI](https://ibm-mas.github.io/cli/)
- [IBM MAS CLI Catalogs](https://ibm-mas.github.io/cli/catalogs/)
- [IBM MAS CLI Image Mirroring](https://ibm-mas.github.io/cli/guides/image-mirroring/)
- [IBM MAS CLI Topology](https://ibm-mas.github.io/cli/reference/topology/)

세부 명령/플래그는 실제 CLI `--help` 출력으로 검증된 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md)를 따릅니다.

---

## 2. 배포 아키텍처

**애플리케이션 레벨**

```text
[ 사용자 브라우저 ]
        |
[ OpenShift Ingress / Route ]
        |
[ MAS Core / Manage Pods ]
        |
[ DB2 ]   [ MongoDB ]   [ SLS ]
```

**설치 파이프라인 레벨** (자세한 단계는 [OFFLINE_INSTALL.md §1.1](OFFLINE_INSTALL.md)):

```text
[인터넷 연결 RHEL 서버] → [Bastion] → [SNO 노드]
```

SNO(Single Node OpenShift)는 컨트롤 플레인과 워커 역할을 단일 노드에서 수행합니다. Bastion은 SNO 노드와 동일 네트워크에 위치하며, 설치 완료 후에도 운영 작업 서버로 사용합니다.

> SNO는 HA 구성이 아닙니다. 운영 환경에서 장애 허용성이 필요하면 3노드 이상 OpenShift 구성을 검토해야 합니다.

---

## 3. 서버 사양

| 구분 | OS / 상태 | CPU | RAM | 디스크 | 비고 |
|------|-----------|-----|-----|--------|------|
| 인터넷 연결 서버 | RHEL 9.6 x86_64 | 4 core 이상 | 16 GB 이상 | **500GB~1TB 권장** (⚠️ 아래 참고) | 파일 다운로드, MAS CLI로 이미지 세트 생성 |
| 전달 매체 (필요시) | Windows/USB/외장디스크 등 | - | - | 전체 전송 파일 크기 이상 | 사이트 간 물리적 이동이 필요한 경우만 |
| Bastion | RHEL 9.6 | 4 core 이상 | 16 GB 이상 | **최소 1TB, 권장 2TB** (⚠️ 아래 참고) | Podman, oc CLI, MAS CLI, Mirror Registry |
| SNO 노드 | 빈 VM 또는 물리 서버 | IBM sizing으로 확정 | IBM sizing으로 확정 | OS와 애플리케이션 데이터 분리 산정 | Agent ISO로 RHCOS+OpenShift 설치 |

⚠️ **실측 기반 정정 (2026-07-29)**: 기존에는 "인터넷 연결 서버 500GB, Bastion Registry 2TB"가 참고치였으나, 실제로 MAS 9.2(`v9-260625-amd64`) 이미지 세트를 미러링해본 결과 공식 참고 수치(MAS 8.10 기준)보다 실제 용량이 **3배 이상** 크게 나오는 것으로 확인됐습니다(예: Core+Catalog 참고치 4GB → 실측 15GB). 전체 미러 데이터가 500~600GB를 넘을 수 있으므로, Bastion은 반입 데이터 + Registry 저장분(사실상 중복 보관)까지 고려해 **최소 1TB, 여유 있게 2TB**를 목표로 합니다. 자세한 근거는 [OFFLINE_INSTALL.md §2.8](OFFLINE_INSTALL.md)을 참고하세요.

Bastion은 설치 작업 서버이므로 RHEL 계열을 권장합니다. SNO 노드는 사전에 RHEL을 설치하는 서버가 아니라, 빈 VM 또는 물리 서버를 ISO로 부팅하여 RHCOS(Red Hat CoreOS)와 OpenShift를 설치하는 대상입니다.

---

## 4. 스토리지 원칙

| 용도 | 요구 AccessMode | 권장 예시 |
|------|-----------------|-----------|
| 일반 RWO PVC | `ReadWriteOnce` | LVM Storage, ODF RBD |
| 공유 RWX PVC | `ReadWriteMany` | ODF CephFS, NFS, Portworx 등 |

> LVM Storage는 기본적으로 RWO 스토리지입니다. `Storage Class (RWX)` 값으로 LVM StorageClass(deviceClass `vg1` 기준 실제 이름은 `lvms-vg1`)를 넣으면 설치 또는 런타임에서 실패할 수 있습니다. RWO와 RWX StorageClass를 모두 준비하고 실제 PVC 마운트 테스트를 완료합니다.

---

## 5. 네트워크 포트

| 포트 | 방향 | 용도 |
|------|------|------|
| 53/tcp,udp | 클라이언트/SNO → Bastion DNS | DNS |
| 80/tcp | 클라이언트 → OpenShift Router | HTTP redirect / ACME 등 |
| 443/tcp | 클라이언트 → OpenShift Router | MAS UI, OpenShift Console |
| 6443/tcp | Bastion/관리자 → SNO | OpenShift API |
| 22623/tcp | SNO 내부 | Machine Config Server |
| 8443/tcp | Bastion 내부/SNO → Bastion | Mirror Registry 기본 포트 |
| 50000/tcp | MAS/Manage → DB2 | DB2 |
| 27017/tcp | MAS Core → MongoDB | MongoDB |

> 실제 방화벽 정책은 OpenShift 설치 방식, Mirror Registry 포트, DB 배치 위치에 따라 조정합니다.

---

## 6. 사전 준비 항목 (확보 위치)

| 항목 | 확보 위치 | 용도 |
|------|------------------|------|
| IBM Entitlement Key | [IBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary) / 문자열 | MAS Core, Manage, Maximo IT 이미지 Pull |
| MAS 라이선스 파일 | IBM License Key Center / `entitlement.lic` 또는 `.dat` | SLS(Suite License Service) AppPoints 라이선스 등록. **Maximo IT AppPoints 포함 필요** |
| Red Hat Pull Secret | [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/install/pull-secret) / `pull-secret.json` | OpenShift 설치와 Red Hat Operator 이미지 Pull |
| Maximo IT 권한 | 구매 계약 / AppPoints | 신규 설치는 라이선스 파일에 포함, 기존 MAS 사용자는 기존 파일에 추가 반영 |
| Maximo IT / Manage 데이터베이스 | Db2, Oracle, SQL Server 등 지원 DB | Manage와 함께 배포/활성화되므로 Manage DB 준비 필요 |

IBM Entitlement Key는 파일이 아니라 문자열입니다. 이미지 Pull 권한 확인은 인터넷 연결 서버에서 수행합니다.

```bash
podman login cp.icr.io -u cp --password-stdin < <entitlement-key-file>
```

Red Hat Pull Secret은 다운로드 후 권한을 제한합니다.

```bash
chmod 600 pull-secret.json
```

---

## 7. 환경 입력 시트

배포별 실제 값을 채워 넣습니다(민감정보 제외).

### 인터넷 연결 서버

| 항목 | 값 |
|------|----|
| Hostname | `<rhel-server-hostname>` |
| IP | `<rhel-server-ip>` |
| OS | `RHEL 9.6 x86_64` |
| 작업 사용자 | `maximo` |
| 이미지 작업 경로 | `/home/maximo/mas-install` |

### Bastion

| 항목 | 값 |
|------|----|
| Hostname | `<bastion-hostname>` |
| FQDN | `<bastion-fqdn>` |
| IP | `<bastion-ip>` |
| OS | `RHEL 9.6` |
| 작업 사용자 | `maximo` |
| DNS 상위 서버 | `<upstream-dns-ip>` |
| Mirror Registry FQDN | `registry.<base-domain>` |
| Mirror Registry 포트 | `8443` |
| 반입 파일 경로 | `/home/maximo/mas-install` |
| Registry 데이터 경로 | `/var/lib/mirror-registry` |

### SNO

| 항목 | 값 |
|------|----|
| Hostname | `<sno-hostname>` |
| IP | `<sno-ip>` |
| Prefix | `<prefix-length>` |
| Gateway | `<gateway-ip>` |
| NIC 이름 | `<nic-name>` |
| NIC MAC 주소 | `<nic-mac-address>` |
| OS 디스크 | `/dev/<os-disk>` |
| 데이터 디스크 | `/dev/<data-disk>` |

### OpenShift 및 MAS

| 항목 | 값 |
|------|----|
| Cluster name | `<cluster-name>` |
| Base domain | `<base-domain>` |
| OpenShift | `4.20.30` |
| MAS Catalog | `v9-260625-amd64` |
| MAS CLI | `quay.io/ibmmas/cli:23.4.1` |
| MAS Instance ID | `<mas-instance-id>` |
| Workspace ID | `<workspace-id>` |
| RWO StorageClass | `<rwo-storage-class>` |
| RWX StorageClass | `<rwx-storage-class>` |

### 보안 파일 (경로와 확보 여부만 관리, 내용은 기록 안 함)

| 항목 | 로컬 보안 경로 | 상태 |
|------|----------------|------|
| IBM Entitlement Key | `<secure-store-reference>` | `<확보/미확보>` |
| MAS 라이선스 | `<secure-dir>/<entitlement-file>` | `<확보/미확보>` |
| Red Hat Pull Secret | `<secure-dir>/pull-secret.json` | `<확보/미확보>` |
| Registry 인증정보 | `<secure-store-reference>` | `<확보/미확보>` |

---

## 8. 문서 구성

| 문서 | 내용 |
|------|------|
| [INSTALLATION.md](INSTALLATION.md) (이 문서) | 환경 기준값, 아키텍처, 서버 사양, 입력 시트 |
| [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md) | 실제 설치 절차 — 사전준비/이미지 미러링(§2), 설치 과정(§3), 실행 중 발견된 이슈(§5) |
| [SERVER_INFO.md](SERVER_INFO.md) | 이번 배포의 실제 서버 접속 정보 |
