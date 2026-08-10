# 서버 정보

접속 주소·계정·`hosts` 등록은 [ACCESS.md](ACCESS.md)를 보세요.

> IBM Entitlement Key가 포함되어 있습니다. 공개 저장소로 옮길 때는 제거하세요.

---

## 1. Bastion

| 항목 | 값 | 판정 |
|---|---|---|
| 호스트명 | `HANA-Bastion` | |
| IP | `192.168.2.210` | |
| 서브넷 마스크 | `255.255.252.0` (`/22`) | |
| 게이트웨이 | `192.168.1.1` | |
| OS | RHEL 9.6 (`5.14.0-570.12.1.el9_6`) | |
| **CPU** | **4 Core** — Xeon Silver 4214R @ 2.40GHz | ⚠️ 동작하나 Push 구간이 느림. **8 Core 권장** |
| **Memory** | **31 GB** | ✅ 충분 |
| **Disk** | **1.5 TB** — `sda` 500GB + `sdb` 1TB를 LVM으로 합쳐 `/` 단일 볼륨 | ✅ 충분. **1TB로는 부족** |
| 역할 | Mirror Registry, DNS(dnsmasq), NTP(chrony), **NFS(RWX)**, 설치 CLI 실행 | |

### 디스크 사용량

| 항목 | 사용량 | 내용 |
|---|---|---|
| `~/mas-install` | **492 GB** | 반입한 미러 파일 (Red Hat + MAS) |
| `~/mirror-registry` | **407 GB** | Quay 이미지 저장소 |
| `/export/mas-rwx` | **25 GB** | NFS RWX — 내부 Image Registry 볼륨 |
| OS·기타 | 약 26 GB | |
| **합계** | **950 GB / 1.5 TB (63%)** | 여유 572 GB |

미러 파일과 Registry 사본을 두 벌 보관하므로 **최소 1.2TB, 여유를 보면 1.5TB**가 필요합니다.

⚠️ 디스크 2개가 **하나의 볼륨 그룹으로 합쳐져** 있어 미러 파일과 Registry 사본이 같은 물리 볼륨에 있습니다. Push 구간에서 읽기·쓰기가 경합하므로 분리하면 속도가 개선됩니다.

⬜ 설치가 끝났으므로 `~/mas-install`(492GB)은 삭제해 공간을 회수할 수 있습니다. 다만 재설치나 다른 사이트 배포에 다시 필요하니 **삭제 전에 보관 여부를 결정**하세요.

### 파일 위치

| 항목 | 경로 |
|---|---|
| `kubeconfig` | `/home/maximo/ocp-sno/auth/kubeconfig` |
| kubeadmin 비밀번호 | `/home/maximo/ocp-sno/auth/kubeadmin-password` |
| SNO 설정 백업 | `/home/maximo/ocp-sno-backup/` |
| Quay 설정 | `/home/maximo/quay-install` |
| Quay 이미지 저장소 | `/home/maximo/mirror-registry/quay` |
| MAS 라이선스 | `/home/maximo/mas-install/licenses/lincense_poc.dat` |
| IBM Entitlement Key | `/home/maximo/mas-install/licenses/entitlement_key.key` |
| 미러 파일 | `/home/maximo/mas-install/mirror` |

Quay 서비스 관리는 rootless(`maximo` 계정)입니다.

```bash
systemctl --user status quay-app
```

IBM Entitlement Key (JWT)

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE3ODQ4NTg0MTMsImp0aSI6IjZjNmZkMzYzY2ZjMzQzNTliNWEzOTRmMDc4Mzg5ZmRmIn0.5nly-wNPU4utGfLXO5yuJ2Eg48cM-5kdS1jVoWlOVgM
```

⚠️ Quay에 읽기 전용 계정은 만들지 않았고 **클러스터 pull secret에도 `init`을 사용**합니다. oc-mirror v2가 업스트림 네임스페이스를 그대로 재현해 Organization이 31개가 되었고, Quay Robot은 소속 Organization 안에서만 권한을 받아 자격증명 하나로 덮을 수 없기 때문입니다. 권한을 좁히려면 전역 pull secret만 교체하면 됩니다.

### 사양 조회

```bash
lscpu | grep -E '^CPU\(s\)|^Model name'; free -h | head -2
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT; df -hT | grep -vE 'tmpfs|devtmpfs'
cat /etc/redhat-release; ip -4 addr show | grep 'inet ' | grep -v 127.0.0.1
du -sh ~/mas-install ~/mirror-registry /export/mas-rwx
```

---

## 2. SNO 노드

| 항목 | 값 | 판정 |
|---|---|---|
| 호스트명 | `mas-it-sno` | |
| IP | `192.168.2.211` | |
| 서브넷 마스크 | `255.255.252.0` (`/22`) | |
| 게이트웨이 | `192.168.1.1` | |
| 대역 (`machineNetwork`) | `192.168.0.0/22` (192.168.0.0 ~ 192.168.3.255) | |
| NIC MAC | `00:50:56:bb:86:df` | `agent-config.yaml`이 MAC으로 호스트 식별 |
| OS | RHCOS 9.6.20260715-0 | |
| OpenShift | `4.20.30` | |
| kubelet | `v1.33.13` | |
| **CPU** | **20 Core** (16 → 20 증설) | ✅ 16 Core로는 이미지 빌드가 막힘 |
| **Memory** | **63 GB** (`65937632Ki`) | ✅ PoC 기준 충분 |
| **Disk** | `sda` **300GB** (OS) + `sdb` **500GB** (LVMS) | ✅ 500GB를 **빈 상태로** 두어야 LVMS가 사용 |
| 스토리지 사용 | LVMS thin pool 449.6GB 중 약 94GB | |
| 역할 | OpenShift + MAS Core / Manage / Maximo IT / Db2 / MongoDB | |

### VM 사전 설정 (vSphere, 전원 끈 상태에서)

| 항목 | 값 | 이유 |
|---|---|---|
| `disk.EnableUUID` (구성 매개변수) | `TRUE` | 없으면 Agent Installer 검증 `vsphere-disk-uuid-enabled` 실패 |
| 네트워크 어댑터 | VMXNET3, **전원을 켤 때 연결** | 끊겨 있으면 설치 진행 불가 |
| MAC 주소 | **수동** 고정 | `agent-config.yaml`이 MAC으로 호스트를 식별 |
| 포트그룹 | Bastion과 동일 (`vmnet0`) | DNS·Registry 도달 |

⚠️ 500GB 디스크에 파티션이나 파일시스템이 있으면 LVMS가 사용하지 못합니다. SNO 설치 시 OS가 300GB에만 설치되도록 `rootDeviceHints`를 정확히 지정하세요.

### 사양 조회

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig
oc get node -o custom-columns='NAME:.metadata.name,CPU:.status.capacity.cpu,MEM:.status.capacity.memory,VERSION:.status.nodeInfo.kubeletVersion,OS:.status.nodeInfo.osImage'
oc describe node mas-it-sno | grep -A6 'Allocated resources'
ssh -i ~/.ssh/quay_installer core@192.168.2.211 'lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE'
```

---

## 3. MAS 구성 정보

| 항목 | 값 | 설명 |
|---|---|---|
| cluster-name | `mas-it` | OpenShift 클러스터 이름. 모든 주소의 앞부분이 됩니다 (`api.mas-it.…`) |
| base-domain | `itmsg.co.kr` | 도메인. `<cluster-name>.<base-domain>` 이 클러스터 FQDN |
| Instance ID | `inst1` | MAS 인스턴스 식별자. 주소와 네임스페이스에 들어갑니다 (`mas-inst1-core`) |
| Workspace ID | `ws1` | MAS 워크스페이스 식별자. Manage 주소 앞에 붙습니다 (`ws1.manage.…`) |
| Workspace 표시명 | `Maximo IT` | 화면에 보이는 이름 |
| Mirror Registry | `registry.mas-it.itmsg.co.kr:8443` | 폐쇄망 이미지 저장소 (Quay). Bastion에서 동작 |
| MAS Core 버전 | `9.2.0` | Suite 본체 |
| Manage 버전 | `9.2.0` (`base` + `icd`) | `icd`가 Maximo IT |
| Catalog | `v9-260625-amd64` | IBM Maximo Operator Catalog. **모든 컴포넌트 버전을 결정** |
| Channel | `9.2.x` | 오퍼레이터 구독 채널 |
| Operational Mode | **non-production** | 🔴 설치 후 변경 불가. 라이선스 계측에 반영 |
| 기본 언어 | `EN` | DB 기본 데이터 언어 |
| 추가 언어 | `KO` | 사용자별로 전환 가능 |
| 타임존 | `Asia/Seoul` | Manage 서버 · Db2 |
| StorageClass (RWO) | `lvms-vg1` | 단일 Pod 전용 볼륨. SNO 로컬 디스크 |
| StorageClass (RWX) | `nfs-client` | 여러 Pod 공유 볼륨. Bastion NFS |
| 데이터베이스 | Db2 전용 인스턴스 `mas-inst1-ws1-manage` | `db2u` 네임스페이스 |
| 담당자 이름 | `seungwoo` `baek` | SLS 등록 정보 |
| 담당자 이메일 | `bsw78@itmsg.co.kr` | 〃 |

