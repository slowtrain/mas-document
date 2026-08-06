# Maximo IT 오프라인 설치 — 진행 상황

설치 절차는 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md)를 참고하세요. 이 문서는 어디까지 진행됐는지만 기록합니다.

**최종 갱신**: 2026-08-06

## 요약

| 단계 | 상태 |
|---|---|
| §2 사전준비 (인터넷 연결 서버) | ✅ **완료** — tar 조각 25개 준비됨, FileZilla 반출 대기 |
| §3 설치 (사이트 Bastion) | 🔄 **§3.6 진행 중** — Red Hat push 완료, MAS push 진행 중 |

## §2 사전준비 — 완료

| # | 항목 | 결과 |
|---|---|---|
| 0 | 디렉터리 + linger | ✅ `Linger=yes` |
| 1 | IBM Entitlement Key | ✅ |
| 2 | MAS 라이선스 | ✅ ⚠️ Maximo IT AppPoints 포함 여부 미확인 |
| 3 | Red Hat Pull Secret | ✅ |
| 4 | RPM 오프라인 저장소 | ✅ 229개 / 135MB, 서명 정상, `repoclosure` 미해결 0건 |
| 5 | OpenShift Client / Installer | ✅ checksum 통과 |
| 6 | MAS CLI 이미지 | ✅ `linux/amd64` |
| 7 | Mirror Registry 패키지 | ✅ |
| 8 | Red Hat 콘텐츠 | ✅ release **190/190**, operator **379/379** → `mirror_000001.tar` 373GB + working-dir 25GB = **398GB** |
| 9 | MAS 콘텐츠 | ✅ **재실행으로 복구** — core 16GB / apps 25GB / other 43GB / cli 4.9GB = **89GB** |
| 10 | NFS 프로비저너 | ✅ tar 43MB + YAML 5개 |

**총 반입량 489GB.** 인터넷 연결 서버는 디스크를 1.5TB로 증설했고 피크 사용량은 841GB였습니다 — MAS 콘텐츠를 먼저 받아둔 상태라 높았고, 8번→캐시 삭제→9번 순서면 약 760GB입니다.

### 8번이 네 번 걸린 이유

| 시도 | 결과 | 원인 | 대응 |
|---|---|---|---|
| 1차 (5시간) | 실패 `rc=4` | 이미지당 타임아웃 기본 **10분** — `rhoai` 대용량 이미지 39건 초과. 캐시가 컨테이너 내부라 116GB 소실 | `--image-timeout 60m`, `--cache-dir`를 호스트로 |
| 2차 (4h15m) | 절단 | **SSH 세션 종료** — rootless podman이 user slice와 함께 정리됨 (`Linger=no`) | `loginctl enable-linger` |
| 3차 (5h53m) | 실패 6건 | **디스크 89%** 압박으로 후반 캐시 쓰기 지연 → 재타임아웃 | 디스크 +1TB 증설 |
| 4차 (1h13m) | ✅ **성공** | — | 캐시 355GB에서 이어받아 실패분만 수신 + tar 생성 |

상세는 [OFFLINE_INSTALL.md §4.3, §4.4](OFFLINE_INSTALL.md).

### 9번에서 발견한 것

첫 실행에서 **Stage 1의 MAS Core가 `[FAILURE]`였는데 재시도되지 않은 상태**였습니다. `[SUCCESS] Selected Dependencies`와 같은 로그에 섞여 있어 놓쳤고, §2.4의 검증이 `[SUCCESS]`만 세도록 되어 있어 걸러지지 않았습니다. 그대로 반입했다면 §3.10 `mas install`에서 막혔을 상황입니다.

재실행으로 복구했고(Core 15GB → 16GB), 문서에 세 가지를 반영했습니다.

- `[FAILURE]`가 0건인지 확인하도록 검증 기준 변경 (성공 개수만 세지 않음)
- `grep` 패턴에서 `^` 앵커 제거 — `mas` CLI 출력에 ANSI 색상 코드가 앞에 붙어 매칭되지 않음
- 단계마다 실행·진행 확인·검증을 짝지어 배치

**Maximo IT는 상태 목록에 별도 줄이 없습니다** — Manage 확장이라 Manage 매니페스트에 포함됩니다. `mirror/apps/v2/cp/manage/extension-icd` 실물로 확인했습니다.

### 반입 전 발견한 IBM CLI 버그

반입하기 전에 `mirror_ocp` 역할의 `from-filesystem.yml`을 열어 확인한 결과, §3.6에서 찾는 파일명이 산출물과 다릅니다.

| | |
|---|---|
| 역할이 하드코딩한 이름 | `mirror_seq1_000000.tar` |
| 내장 oc-mirror(4.22.0)가 만드는 이름 | `mirror_000001.tar` |

`to-filesystem.yml`은 파일명을 다루지 않고 `oc mirror`를 그대로 호출하므로, 래퍼로만 진행해도 어긋납니다 — **IBM CLI 23.4.1 자체의 버전 불일치**입니다.

**§3.6 직전에 역할 파일을 다시 읽어보니 문제가 파일명만이 아니었습니다.** `--from`에 tar 파일 경로를 주는 것 자체가 oc-mirror **v1** 문법이고, `--v2`는 `--from file://<디렉터리>`를 받아 tar를 스스로 찾습니다. 이름을 맞춰줘도 통하지 않습니다.

**리네임은 하지 않기로 했습니다.** §2.3 8번에서 이미 `oc mirror`를 직접 호출했으므로 §3.6도 대칭으로 직접 호출합니다 — 파일명이 무관해지고 `transfer-files.sha256`과도 어긋나지 않습니다. 상세는 [OFFLINE_INSTALL.md §4.5](OFFLINE_INSTALL.md).

### §2.5 전달 준비 — 완료

| 항목 | 결과 |
|---|---|
| `oc-mirror-cache` 삭제 | ✅ 355GB 회수 → `mas-install` 489GB |
| 파일 체크섬 (`transfer-files.sha256`) | ✅ **9991개** — 디스크 파일 수와 일치 |
| tar 분할 (`~/mas-install-transfer`) | ✅ **25조각** (`part-00`~`part-24`, 마지막 8.2GB), 합계 489GB |
| 조각 체크섬 (`transfer-parts.sha256`) | ✅ 25줄 |

콜론 문제 때문에 tar 방식을 택했습니다 — 미러 산출물에 `blobs/sha256:...` 파일이 4227개 있어 NTFS/exFAT에 직접 복사하면 실패합니다. tar 안에서는 문제되지 않고 사이트 Linux에서 원래 이름으로 복원됩니다.

### 남은 작업

- [ ] FileZilla로 조각 25개 + `transfer-parts.sha256` 반출
- [ ] 사이트 Bastion 디스크 1.5TB 증설 (현재 500GB)
- [ ] 사이트 Bastion에서 복원 및 검증 → §3.1 시작

## §3 설치 — §3.6 진행 중

| 절 | 상태 |
|---|---|
| 3.1 RPM 저장소 등록·도구 설치 | ✅ |
| 3.2 DNS (dnsmasq) | ✅ 폐쇄망 상태로 검증 — 상위 DNS 차단 후 내부 이름만 해석되는지 확인 |
| 3.3 NTP (chrony) | ✅ `0.0.0.0:123` Stratum 10으로 서비스 |
| 3.4 MAS CLI 이미지 로드 | ✅ `quay.io/ibmmas/cli:23.4.1` |
| 3.5 Mirror Registry (Quay) 설치 | ✅ 컨테이너 3개 `Up`, `curl … /v2/` → `401`(`-k` 없이), `podman login` → `Login Succeeded!` |
| 3.6 이미지 Push | ✅ Red Hat 373GB + MAS 4단계, `[FAILURE]` 0건. Quay Organization 31개 |
| 3.7 SNO 설치 | ✅ `4.20.30` / 노드 `Ready` / `sdb` 500GB 빈 상태 확인 |
| 3.8 이후 | ⬜ |

### §3.7에서 확인한 것

**`install-config.yaml`의 `imageDigestSources`에는 release 미러 2건만 넣습니다.** IDMS 34건을 그대로 옮기면 `openshift-install` 검증이 `docker.io/grafana`를 거부합니다 — IDMS CRD보다 설치 프로그램 쪽 검증이 엄격합니다. 오퍼레이터 미러는 §3.8에서 `oc apply`로 적용하므로 버리는 것이 아니라 시점을 미루는 것입니다(§4.6).

**디스크는 `hctl`로 지정합니다.** `deviceName: /dev/sda`는 인식 순서에 따라 달라지는데다 항상 무언가에 매칭돼 조용히 틀립니다. vSphere의 `SCSI(0:0)` → `hctl: "0:0:0:0"`. 힌트에 맞는 디스크가 없으면 설치가 검증 실패로 멈추므로 데이터를 잃지 않습니다.

**NIC은 `identifier: mac-address`로 MAC 매칭**합니다. VMXNET3는 보통 `ens192`지만 보장이 아닙니다.

**Pull Secret에서 `cloud.openshift.com`을 제거**했습니다. 폐쇄망에서 텔레메트리 전송을 시도해 Insights 오퍼레이터가 Degraded로 가는 것을 막기 위해서입니다.

`openshift-install agent create image`는 두 yaml을 **소비하며 삭제**합니다. 백업(§3.7.5)을 빼먹으면 오타 하나에 §3.7.2부터 다시 해야 합니다.

부팅 후 두 곳에서 막혔고 **둘 다 ISO 바깥의 문제**였습니다.

| 증상 | 원인 | 조치 |
|---|---|---|
| `Unable to pull OpenShift release image` | Bastion `/etc/dnsmasq.conf`의 `interface=lo`·`bind-interfaces` 때문에 dnsmasq가 루프백에만 응답. Bastion 자신의 `nslookup`은 전부 통과해 §3.2.4 검증에서 안 걸림 | `mas.conf`에 `interface=ens192` 추가 |
| `1 hosts not validated` | vSphere `disk.EnableUUID` 미설정 (`vsphere-disk-uuid-enabled Status:failure`) | VM 전원 끄고 구성 매개변수에 `disk.EnableUUID=TRUE` |

ISO에 넣은 값은 모두 맞았습니다 — `ens192`에 IP가 붙었고, `hctl: "0:0:0:0"`이 300GB `sda`와 일치했으며, `api`·`api-int`·`*.apps` DNS 검증도 success였습니다.

CD/DVD를 분리할 때 **ESXi가 "게스트가 CD-ROM을 잠갔습니다" 질문을 띄우고 VM이 정지**합니다. 응답하기 전까지 API가 끊기므로 `no route to host`가 계속 나옵니다. vSphere 요약의 CPU가 `0 Hz`면 이 상태입니다.

`openshift-samples`는 `managementState: Removed`입니다 — 폐쇄망에서 외부 레지스트리에 못 닿아 스스로 비활성화한 정상 상태입니다.

`oc debug node/...`는 `registry.redhat.io/rhel9/support-tools`를 받아오므로 **§3.8에서 오퍼레이터 IDMS를 적용하기 전에는 동작하지 않습니다.** 그 전에는 SSH로 노드에 접속해 확인합니다.

### §3.6에서 확인한 것

**IBM 래퍼(`mas mirror-redhat-images --mode from-filesystem`)는 CLI 23.4.1에서 동작하지 않습니다.** 실제 실행 결과입니다.

```
[INFO]  : 🔀 workflow mode: diskToMirror
[ERROR] : [Executor] no tar archives matching "mirror_[0-9]{6}\.tar"
          found in "/mnt/workspace/redhat/mirror_seq1_000000.tar"
```

`--from` 값을 **디렉터리로 열어서** 그 안의 `mirror_[0-9]{6}.tar`를 찾습니다. 산출물 `mirror_000001.tar`는 이 패턴에 이미 맞으므로 **리네임하면 오히려 어긋납니다.** Red Hat 공식 문법(`--from file://<디렉터리>`)으로 직접 호출해 해결했습니다.

Red Hat push **3시간 29분** 소요, 오류 0건. `working-dir/cluster-resources/`에 IDMS·ITMS·CatalogSource가 생성됐습니다 — §3.8.1의 "`--setup-redhat-catalogs` 플래그 실존 여부" 미확인 항목이 이것으로 해소됩니다. 그 플래그가 없어도 `cs-*.yaml`을 `oc apply`하면 됩니다.

Quay 저장 용량은 tar 373GB → **325GB**입니다. 이미지 간 공유 레이어를 중복 저장하지 않습니다.

**ODF는 추가로 받을 필요가 없었습니다.** `--mirror-odf`가 받는 건 ODF 4.15용 PostgreSQL 이미지 한 개뿐이고, 실제 오퍼레이터(`odf-operator`, `ocs-operator`, `mcg-operator`, `rook-ceph-operator` 등)는 §2.3 8번 imageset에 이미 포함돼 Registry에 올라가 있습니다. 멀티 노드로 전환하더라도 인터넷 없이 OperatorHub에서 바로 설치할 수 있습니다.

MAS push에는 **`--ibm-entitlement`가 필수**입니다 — 폐쇄망이라 인증에 쓰이지 않는데도 없으면 CLI가 `--help`만 출력하고 종료합니다.

계정 정보는 [SERVER_INFO.md](SERVER_INFO.md)의 **계정** 절에 있습니다.

### §3.5에서 확인한 것

**`mirror-registry install`은 SSH 키를 자동 생성하지 않습니다.** `~/.ssh/`가 없는 상태로 실행하면 이미지 로드 직후 `Could not find ssh key at ~/.ssh/quay_installer`로 중단됩니다. `-v` 없이 실행하면 이 메시지만 덜렁 나와 원인 파악이 어렵습니다. 자기 자신에게 설치해도 ansible이 SSH로 접속하므로 키 생성 + `authorized_keys` 등록이 전제 조건입니다 — §3.5.2로 문서에 추가했습니다.

`--targetHostname`을 IP로 준 것도 유효했습니다. 기본값인 호스트명이 IPv6 링크로컬(`fe80::`)로만 해석되는 상태였습니다.

설치 소요 **약 15분**. `Copy Images`(2.5GB를 SSH로 전송)와 Quay 첫 기동 대기(`FAILED - RETRYING` 2회)가 대부분입니다. 두 구간 모두 출력이 멈춘 것처럼 보이므로 진행 확인 명령을 문서에 넣었습니다.

설치 프로그램이 마지막에 `Enable lingering`을 수행합니다 — Quay는 rootless systemd user 서비스라서 §2.3 0번의 `loginctl enable-linger`와 같은 이유입니다.

## 실행 전 확인이 필요한 항목

### 실행하면서 즉시 확인 가능

| 항목 | 확인 방법 | 관련 절 |
|---|---|---|
| `from-filesystem`이 `working-dir`도 필요한지 (tar 외에) | §3.6 첫 실행 | §3.6 |
| `imageDigestSources` vs `imageContentSources` | `openshift-install explain installconfig \| grep -i image` | §3.7 |
| `LVMCluster`의 `apiVersion` | `oc explain lvmcluster` | §3.9.1 |
| NFS 프로비저너 SCC 부여 필요 여부 | 배포 후 Pod 상태 | §3.9.2 |

### 공식 문서를 브라우저로 직접 확인해야 하는 것

| 항목 | 비고 |
|---|---|
| 🔴 **한국어 VARGRAPHIC 설정 위치** | ansible-devops `db2` role에 문자셋 전용 변수 없음. Db2 레벨인지 Manage 활성화 레벨인지 불명. **되돌리기 가장 어려운 결정** |
| **Maximo IT AppPoints 권한** | 라이선스 파일 보유가 IT 권한 보유를 의미하지 않음. MAS Entitlement와 Maximo IT Entitlement가 각각 필요 |
| Manage + Maximo IT 배포 절차 (§3.11) | IBM 지식센터가 자동 조회 환경에서 403. Db2 ROW·Server Bundle 등 일부만 검증됨 |
| Manage가 내부 Image Registry를 쓰는지 | §3.9.3의 용량 산정에 영향 |
| SNO에서 PTR(역방향) DNS 필수 여부 | §3.2 |

## 구조적 리스크 (의사결정 필요)

| 항목 |
|---|
| 🔴 **사이트 Bastion 디스크 1.5TB 필요** — 반입 489GB + Registry 사본 약 489GB. 현재 500GB이므로 증설 필수 |
| `rhods-operator`(OpenShift AI)가 Red Hat 콘텐츠 373GB의 대부분. MAS 미사용이고 IBM imageset에서 *Optional* 로 분류. 제거하면 양쪽 서버 용량이 크게 줄지만 재미러링 6시간 |
| `mirror registry for Red Hat OpenShift`는 OCP 설치용 소규모·비HA Registry — MAS 장기 운영 Registry로는 지원 범위 밖 |
| Bastion이 Mirror Registry + DNS + NTP + NFS를 모두 겸함 → 단일 장애점 |
| **watsonx AI 에이전트는 이번 범위 밖** — 폐쇄망이라 SaaS 연동이 불가능하고, on-prem은 IBM Software Hub(구 CP4D) 위에 올라갑니다. §2.3 9번에서 `--mirror-cp4d`·`--mirror-wsl`·`--mirror-wml`을 제외했고 SNO 1대(16C/64GB, GPU 없음)로는 수용 불가입니다. 도입한다면 **클러스터 재설계 + GPU 노드 + 수백 GB 추가 미러링**이 필요합니다 |
| NFS 동적 프로비저너는 커뮤니티 프로젝트 — Red Hat/IBM 미지원 |
