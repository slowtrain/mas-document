# 서버 정보 (이번 배포)

> ⚠️ 접속 계정/비밀번호는 이 문서에 기록하지 않습니다. 별도 보안 저장소에서 관리하세요.
> (현재 평문 기재된 항목이 있다면 제거하고 `.gitignore` 처리를 권장합니다.)

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
