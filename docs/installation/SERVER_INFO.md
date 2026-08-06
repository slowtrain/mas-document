# 서버 정보 (이번 배포)

> 개인 전용 저장소이므로 계정·비밀번호를 아래 **계정** 절에 함께 기록합니다. 공개 저장소로 옮기거나 공유할 때는 그 절을 먼저 제거하세요.

---

## Bastion

| 항목 | 값 |
|---|---|
| IP | `192.168.2.210` |
| OS | RHEL 9.6 |
| 겸하는 역할 | Mirror Registry, DNS(dnsmasq), NTP(chrony), **NFS(RWX)**, 설치 CLI 실행 |

### 사양 — 증설 예정

| 항목 | 현재 | 증설 후 | 근거 |
|---|---|---|---|
| CPU | 4 Core | **8 Core** | 현재로도 동작하나 Push 구간이 느림 |
| Memory | 8 GB | **16~32 GB** | Mirror Registry 공식 최소가 단독으로 8GB인데 NFS·DNS·CLI를 겸함 |
| Disk | 500 GB (1개) | **1TB 이상 (2개 분리)** | 미러 파일 + Registry 사본을 두 벌 보관 |

### 디스크 사용 계획 (실측 기준)

| 항목 | 예상 |
|---|---|
| 미러 파일 (`~/mas-install/mirror`) | 약 210~250GB |
| Mirror Registry 사본 (`/var/lib/mirror-registry`) | 약 210~250GB |
| NFS RWX (`/export/mas-rwx`) + OS + 여유 | 100GB 이상 |
| **합계** | **약 550~650GB 이상** |

증설 시 `/var/lib/mirror-registry`를 별도 디스크로 분리합니다(OFFLINE_INSTALL.md §4.4의 1번). Push 구간에서 읽기(Disk 1) / 쓰기(Disk 2)가 분산되어 속도도 개선됩니다.

---

## SNO 노드

| 항목 | 값 |
|---|---|
| 고정 IP | `192.168.2.211` |
| 서브넷 마스크 | `255.255.252.0` (`/22` → `192.168.0.0/22`) |
| 게이트웨이 | `192.168.1.1` |
| NIC MAC | `00:50:56:bb:86:df` |
| 호스트명 | `mas-it-sno` |

### 사양

| 항목 | 값 | 판정 |
|---|---|---|
| CPU | 16 Core | ✅ |
| Memory | 64 GB | ✅ |
| Disk | **300GB + 500GB (2개)** | ✅ **LVMS용 빈 디스크 확보 가능** |

**디스크 용도 계획**

| 디스크 | 용도 |
|---|---|
| 300GB | RHCOS(OS) 설치 — `install-config.yaml`의 `rootDeviceHints` 대상 |
| 500GB | **빈 상태로 유지** → LVM Storage(LVMS)가 RWO StorageClass(`lvms-vg1`)로 사용 |

**VM 사전 설정** (vSphere, 전원 끈 상태에서)

| 항목 | 값 | 이유 |
|---|---|---|
| `disk.EnableUUID` (구성 매개변수) | `TRUE` | 없으면 Agent Installer 검증 `vsphere-disk-uuid-enabled` 실패 |
| 네트워크 어댑터 | VMXNET3, **전원을 켤 때 연결** | 끊겨 있으면 설치 진행 불가 |
| MAC 주소 | **수동** 고정 | `agent-config.yaml`이 MAC으로 호스트를 식별 |
| 포트그룹 | Bastion과 동일 (`vmnet0`) | DNS·Registry 도달 |

⚠️ 500GB 디스크에 파티션이나 파일시스템이 있으면 LVMS가 사용하지 못합니다. SNO 설치 시 OS가 300GB에만 설치되도록 `rootDeviceHints`를 정확히 지정하세요. 실제 디바이스명(`/dev/sda`, `/dev/sdb` 등)은 SNO 부팅 후 확인합니다.

---

## 네트워크 확인

| 항목 | 값 | 검산 |
|---|---|---|
| 대역 | `192.168.0.0/22` (192.168.0.0 ~ 192.168.3.255) | — |
| Bastion | 192.168.2.210 | ✅ 대역 내 |
| SNO | 192.168.2.211 | ✅ 대역 내 |
| Gateway | 192.168.1.1 | ✅ 대역 내 |
| 상위 DNS | 현재 `8.8.8.8` (인터넷 연결 테스트 환경) | 사이트에는 사내 DNS 없음 → `server=` 주석 처리 예정 |

`install-config.yaml`의 `machineNetwork`는 `192.168.0.0/22`로 지정합니다.

---

## 도메인·식별자

[OFFLINE_INSTALL.md §0](OFFLINE_INSTALL.md)의 확정값과 동일합니다.

| 항목 | 값 |
|---|---|
| cluster-name | `mas-it` |
| base-domain | `itmsg.co.kr` |
| MAS instance ID | `inst1` |
| Workspace ID | `ws1` |
| Mirror Registry | `registry.mas-it.itmsg.co.kr:8443` |

---

## 계정

### Mirror Registry (Quay) — 2026-08-05 설치

| 항목 | 값 |
|---|---|
| URL | `https://registry.mas-it.itmsg.co.kr:8443` |
| 관리자 | `init` / `T3AgRHSeYw6x5u0Z2q4OE9s81Koyd7tV` |
| 설정 위치 | `/home/maximo/quay-install` |
| 이미지 저장소 | `/home/maximo/mirror-registry/quay` |
| 서비스 관리 | `systemctl --user status quay-app` (rootless, `maximo` 계정) |

읽기 전용 계정은 만들지 않았습니다. oc-mirror v2가 업스트림 네임스페이스를 그대로 재현해 Organization이 31개가 되었고, Quay Robot은 소속 Organization 안에서만 권한을 받아 자격증명 하나로 덮을 수 없기 때문입니다. **클러스터 pull secret에도 `init`을 사용합니다** — 폐쇄망 내부 전용 전제. 설치 후 좁히려면 전역 pull secret만 교체하면 됩니다.

### OpenShift SNO — 2026-08-06 설치

| 항목 | 값 |
|---|---|
| API | `https://api.mas-it.itmsg.co.kr:6443` |
| 콘솔 | `https://console-openshift-console.apps.mas-it.itmsg.co.kr` |
| 관리자 | `kubeadmin` / `oRYob-UVjUs-AI492-mjDsc` |
| `kubeconfig` | `/home/maximo/ocp-sno/auth/kubeconfig` |
| 비밀번호 파일 | `/home/maximo/ocp-sno/auth/kubeadmin-password` |
| 설정 백업 | `/home/maximo/ocp-sno-backup/` |

노트북에서 콘솔에 접속하려면 `hosts`에 개별 이름을 넣어야 합니다 — `*.apps` 와일드카드는 hosts 파일이 지원하지 않습니다.

```
192.168.2.210    registry.mas-it.itmsg.co.kr
192.168.2.211    console-openshift-console.apps.mas-it.itmsg.co.kr
192.168.2.211    oauth-openshift.apps.mas-it.itmsg.co.kr
```

로그인 시 OAuth로 리다이렉트되므로 `oauth-openshift` 항목도 필요합니다.

### SSH 키

| 항목 | 값 |
|---|---|
| `~/.ssh/quay_installer` | `mirror-registry install`이 요구하는 키. 2026-08-05 생성, `authorized_keys`에 등록됨 |
| SNO 노드 접속 | `ssh -i ~/.ssh/quay_installer core@192.168.2.211` — `install-config.yaml`의 `sshKey`로 같은 키를 사용 |
