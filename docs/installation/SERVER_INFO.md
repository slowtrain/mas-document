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
| 읽기 전용 계정 | ⬜ 미생성 — §3.6에서 OCP 노드에 배포할 pull-only 계정 |

설치 시 `mas mirror-images`에 넘긴 `REGISTRY_USERNAME`은 `masadmin`이었습니다. 목적지 경로 생성에만 쓰였고 실제 Quay 계정과는 무관합니다 — §3.5 push에는 위 `init` 계정을 씁니다.

### SSH 키

| 항목 | 값 |
|---|---|
| `~/.ssh/quay_installer` | `mirror-registry install`이 요구하는 키. 2026-08-05 생성, `authorized_keys`에 등록됨 |
