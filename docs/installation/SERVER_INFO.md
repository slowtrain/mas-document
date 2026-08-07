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
| CPU | 16 Core → **20 Core 증설** | ⚠️ 아래 참고 |
| Memory | 64 GB | ✅ (PoC 기준) |
| Disk | **300GB + 500GB (2개)** | ✅ **LVMS용 빈 디스크 확보 가능** |

🔴 **16 Core로는 설치는 되지만 이후 변경 작업이 막힙니다.** 설치 완료 시점에 CPU 요청이 **14551m / 15600m (93%)** 이었고, 언어팩 추가(`secondaryLangs`)를 시도하자 이미지 빌드 Pod이 `Insufficient cpu`로 스케줄되지 못했습니다. 오퍼레이터가 기다리다 빌드를 취소하는 일이 반복됐습니다.

```
0/1 nodes are available: 1 Insufficient cpu
BuildReady: False — Request Build Failed : Unexpected error occured while requesting admin build
```

빌드는 **언어 추가·커스터마이징·업그레이드마다** 필요하므로 여유 CPU가 상시 필요합니다. 20 Core로 증설했습니다.

주요 CPU 요청처(설치 직후 실측)

| Pod | 요청 |
|---|---|
| `mongoce/mas-mongo-ce-0` | 1000m |
| `mas-inst1-manage/inst1-ws1-slackproxy` | **1000m** — Slack 미사용인데도 기동됨 |
| `mas-inst1-manage/inst1-ws1-all` (Maximo 서버) | 600m |
| `mongodb-kubernetes-operator` / `db2u-operator` / `ibm-sls-controller` / `slackproxy-operator` / `entitymgr-suite` | 각 500m |
| `inst1-ws1-manage-maxinst` | 500m — DB 작업 후에도 상주 |

⚠️ `slackproxy`를 `oc scale ... --replicas=0` 으로 내려도 **오퍼레이터가 되살립니다.** 워크로드를 줄이는 방식으로는 해결되지 않습니다.

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

### 접속 주소 모음

| 주소 | 용도 | 계정 | 비밀번호 |
|---|---|---|---|
| `https://admin.inst1.apps.mas-it.itmsg.co.kr` | **MAS Suite Administration** | `O1Soo6ql8f0fXzaLKGhzSMa400EDFNvQ` | `qIDqNdwuttnWw94dF6B9SUdJ6BxNKClA` |
| `https://home.inst1.apps.mas-it.itmsg.co.kr` | MAS 홈 | 〃 | 〃 |
| `https://ws1.manage.inst1.apps.mas-it.itmsg.co.kr` | **Maximo Manage / Maximo IT** | 〃 | 〃 |
| `https://console-openshift-console.apps.mas-it.itmsg.co.kr` | OpenShift 웹 콘솔 | `kubeadmin` | `oRYob-UVjUs-AI492-mjDsc` |
| `https://api.mas-it.itmsg.co.kr:6443` | OpenShift API (`oc login`) | `kubeadmin` | 〃 |
| `https://registry.mas-it.itmsg.co.kr:8443` | Mirror Registry (Quay) | `init` | `T3AgRHSeYw6x5u0Z2q4OE9s81Koyd7tV` |
| `ssh maximo@192.168.2.210` | Bastion | `maximo` | `dltmdrb1!` |
| `ssh -i ~/.ssh/quay_installer core@192.168.2.211` | SNO 노드 | `core` | (키 인증) |

접속 전 [노트북 `hosts` 등록](#노트북-hosts-등록)이 필요하며, 자체 서명 인증서라 **호스트마다 브라우저 경고를 한 번씩 수락**해야 합니다.

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

### MAS — 2026-08-07 설치

| 항목 | 값 |
|---|---|
| Suite Administration | `https://admin.inst1.apps.mas-it.itmsg.co.kr` |
| Manage / Maximo IT | `https://manage.inst1.apps.mas-it.itmsg.co.kr` |
| Superuser | `O1Soo6ql8f0fXzaLKGhzSMa400EDFNvQ` / `qIDqNdwuttnWw94dF6B9SUdJ6BxNKClA` |
| 담당자 (Contact) | first `seungwoo` / last `baek` / `bsw78@itmsg.co.kr` |
| Instance ID / Workspace ID | `inst1` / `ws1` (표시명 `Maximo IT`) |
| Operational Mode | **non-production** — 🔴 설치 후 변경 불가 |
| 버전 | MAS Core 9.2.0 / Manage 9.2.0 (`base` + `icd`) |
| Catalog / Channel | `v9-260625-amd64` / `9.2.x` |
| StorageClass (RWO / RWX) | `lvms-vg1` / `nfs-client` |
| 데이터베이스 | Db2 전용 인스턴스 `mas-inst1-ws1-manage` (`db2u` 네임스페이스) |
| 라이선스 파일 | `/home/maximo/mas-install/licenses/lincense_poc.dat` |
| IBM Entitlement Key | `/home/maximo/mas-install/licenses/entitlement_key.key` (아래) |

⚠️ **Superuser는 CLI가 랜덤 생성한 값입니다.** 대화형에서 입력한 `masadmin` / `Itmsg4u!` 는 적용되지 않았습니다 — 비대화형 재실행에서 Superuser를 묻지 않기 때문입니다. 실제 값은 시크릿에서 확인합니다.

```bash
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.username}' | base64 -d; echo
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.password}' | base64 -d; echo
```

⬜ Superuser는 초기 구성 전용입니다. 설치가 끝나면 Suite Administration에서 정식 관리자 계정을 만들고 그것을 사용하세요.

IBM Entitlement Key (JWT)

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE3ODQ4NTg0MTMsImp0aSI6IjZjNmZkMzYzY2ZjMzQzNTliNWEzOTRmMDc4Mzg5ZmRmIn0.5nly-wNPU4utGfLXO5yuJ2Eg48cM-5kdS1jVoWlOVgM
```

### 노트북 `hosts` 등록

폐쇄망에 사내 DNS가 없고 Bastion의 dnsmasq는 클러스터 전용이라, 웹 UI 접속용 이름을 직접 넣어야 합니다. `*.apps` 와일드카드는 `hosts` 파일이 지원하지 않아 개별 이름이 필요합니다.

| OS | 경로 |
|---|---|
| Windows | `C:\Windows\System32\drivers\etc\hosts` (관리자 권한) |
| macOS / Linux | `/etc/hosts` (root) |

```
192.168.2.210    registry.mas-it.itmsg.co.kr

192.168.2.211    api.mas-it.itmsg.co.kr
192.168.2.211    console-openshift-console.apps.mas-it.itmsg.co.kr
192.168.2.211    oauth-openshift.apps.mas-it.itmsg.co.kr
192.168.2.211    downloads-openshift-console.apps.mas-it.itmsg.co.kr

192.168.2.211    admin.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    api.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    auth.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    home.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    manage.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    manage-api.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    ws1.manage.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    ws1-all.manage.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    maxinst.manage.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    ws1-mcp.manage.inst1.apps.mas-it.itmsg.co.kr

192.168.2.211    alertmanager-main-openshift-monitoring.apps.mas-it.itmsg.co.kr
192.168.2.211    prometheus-k8s-openshift-monitoring.apps.mas-it.itmsg.co.kr
192.168.2.211    prometheus-k8s-federate-openshift-monitoring.apps.mas-it.itmsg.co.kr
192.168.2.211    thanos-querier-openshift-monitoring.apps.mas-it.itmsg.co.kr
192.168.2.211    pipelines-as-code-controller-openshift-pipelines.apps.mas-it.itmsg.co.kr
192.168.2.211    tekton-results-api-openshift-pipelines.apps.mas-it.itmsg.co.kr
192.168.2.211    tkn-cli-serve-openshift-pipelines.apps.mas-it.itmsg.co.kr
```

| 이름 | 용도 |
|---|---|
| `registry.…` | Mirror Registry (Quay) — **Bastion IP** |
| `api.mas-it.…` | OCP API — 노트북에서 `oc` 사용 시 |
| `console-openshift-console.…` | OCP 웹 콘솔 |
| `oauth-openshift.…` | 콘솔 로그인 리다이렉트 — **빠지면 로그인 불가** |
| `admin.inst1.…` | **Suite Administration** |
| `api.inst1.…` | MAS API — **빠지면 Suite Administration이 로딩에서 멈춤** |
| `auth.inst1.…` | MAS 로그인 리다이렉트 |
| `home.inst1.…` | MAS 홈 |
| **`ws1.manage.inst1.…`** | **Maximo Manage / Maximo IT 메인** — 워크스페이스 ID가 앞에 붙습니다 |
| `ws1-all.manage.inst1.…` | `all` 서버 번들 직접 접근 |
| `maxinst.manage.inst1.…` | 관리 도구 (ERD, toolsapi) |
| `manage.inst1.…` / `manage-api.inst1.…` | 오퍼레이터가 먼저 만드는 Route. 실제 서비스는 `ws1.` 쪽 |

⚠️ Route는 설치가 진행되면서 늘어납니다. Manage 활성화 후 다시 뽑아 추가하세요.

```bash
oc get route -A -o jsonpath='{range .items[*]}192.168.2.211    {.spec.host}{"\n"}{end}' | sort -u
```

### SSH 키

| 항목 | 값 |
|---|---|
| `~/.ssh/quay_installer` | `mirror-registry install`이 요구하는 키. 2026-08-05 생성, `authorized_keys`에 등록됨 |
| SNO 노드 접속 | `ssh -i ~/.ssh/quay_installer core@192.168.2.211` — `install-config.yaml`의 `sshKey`로 같은 키를 사용 |
