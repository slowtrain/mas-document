# 사이트 착수 시트

새 사이트에 설치할 때 **먼저 채워야 할 값**과 **열어야 할 포트**입니다. 채운 뒤에는 [SERVER_INFO.md](SERVER_INFO.md)에 실측값으로 옮겨 적습니다.

설치 절차는 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md), 접속 정보는 [ACCESS.md](ACCESS.md)를 보세요.

---

## 1. 착수 전 확인 (담당자에게 물어볼 것)

| 항목 | 왜 필요한가 | 확인 |
|---|---|---|
| **사내 DNS 존재 여부** | 없으면 사용자 PC마다 `hosts` 등록이 필요해 운영 불가 | ⬜ |
| **`*.apps` 와일드카드 등록 가능 여부** | 안 되면 이름을 개별 등록하거나 존 위임 필요 | ⬜ |
| **사내 CA 존재 여부** | 없으면 자체 서명 인증서를 전 PC에 배포해야 함 | ⬜ |
| **Operational Mode** (운영 / 비운영) | 🔴 설치 후 변경 불가. 라이선스 계측에 직결 | ⬜ |
| **한국어 사용 여부** | 설치 인자로 넣어야 함. 나중에 추가하면 이미지 재빌드 | ⬜ |
| **데모 데이터 필요 여부** | maxinst 시점에만 적재. 나중에 추가하려면 DB 재생성 | ⬜ |
| **Maximo IT AppPoints 보유** | MAS Entitlement와 별개. 없으면 사용자 할당에서 막힘 | ⬜ |
| **전송매체 반입 절차** | 약 500GB를 옮길 방법과 승인 | ⬜ |

---

## 2. 네트워크 포트

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


---

## 3. 환경 입력 시트

배포별 실제 값을 채워 넣습니다(민감정보 제외).

`예시`는 이번 배포(`mas-it.itmsg.co.kr`)에 실제로 넣은 값입니다.

### Bastion

| 항목 | 값 | 예시 |
|---|---|---|
| Hostname | | `HANA-Bastion` |
| IP | | `192.168.2.210` |
| 서브넷 마스크 | | `255.255.252.0` (`/22`) |
| 게이트웨이 | | `192.168.1.1` |
| OS | | `RHEL 9.6` |
| 작업 사용자 | | `maximo` |
| DNS 상위 서버 | | 없음 (폐쇄망) |
| Mirror Registry FQDN | | `registry.mas-it.itmsg.co.kr` |
| Mirror Registry 포트 | | `8443` |
| 반입 파일 경로 | | `/home/maximo/mas-install` |
| Registry 데이터 경로 | | `/home/maximo/mirror-registry` |
| NFS 내보내기 경로 | | `/export/mas-rwx` |

### SNO

| 항목 | 값 | 예시 |
|---|---|---|
| Hostname | | `mas-it-sno` |
| IP | | `192.168.2.211` |
| Prefix | | `22` |
| Gateway | | `192.168.1.1` |
| 대역 (`machineNetwork`) | | `192.168.0.0/22` |
| NIC 이름 | | `ens192` |
| NIC MAC 주소 | | `00:50:56:bb:86:df` |
| OS 디스크 | | `/dev/sda` (300GB) — `hctl: "0:0:0:0"` 로 지정 |
| 데이터 디스크 | | `/dev/sdb` (500GB) — **빈 상태 유지**, LVMS용 |

### OpenShift 및 MAS

| 항목 | 값 | 예시 |
|---|---|---|
| Cluster name | | `mas-it` |
| Base domain | | `itmsg.co.kr` |
| OpenShift | | `4.20.30` |
| MAS Catalog | | `v9-260625-amd64` |
| MAS Channel | | `9.2.x` |
| MAS CLI | | `quay.io/ibmmas/cli:23.4.1` |
| MAS Instance ID | | `inst1` |
| Workspace ID | | `ws1` |
| Workspace 표시명 | | `Maximo IT` |
| Operational Mode | | `non-production` |
| 기본 언어 / 추가 언어 | | `EN` / `KO` |
| 타임존 | | `Asia/Seoul` |
| RWO StorageClass | | `lvms-vg1` |
| RWX StorageClass | | `nfs-client` |
| 담당자 이름 / 이메일 | | `seungwoo` `baek` / `bsw78@itmsg.co.kr` |

### 보안 파일 (경로와 확보 여부만 관리, 내용은 기록 안 함)

| 항목 | 로컬 보안 경로 | 상태 |
|------|----------------|------|
| IBM Entitlement Key | `<secure-store-reference>` | `<확보/미확보>` |
| MAS 라이선스 | `<secure-dir>/<entitlement-file>` | `<확보/미확보>` |
| Red Hat Pull Secret | `<secure-dir>/pull-secret.json` | `<확보/미확보>` |
| Registry 인증정보 | `<secure-store-reference>` | `<확보/미확보>` |

