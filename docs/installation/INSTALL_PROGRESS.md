# Maximo IT 오프라인 설치 — 진행 상황

설치 절차는 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md)를 참고하세요. 이 문서는 어디까지 진행됐는지만 기록합니다.

**최종 갱신**: 2026-08-05

## 요약

| 단계 | 상태 |
|---|---|
| §2 사전준비 (인터넷 연결 서버) | ✅ **완료** — tar 조각 25개 준비됨, FileZilla 반출 대기 |
| §3 설치 (사이트 Bastion) | 🔄 **§3.4까지 진행** — Mirror Registry 기동 완료, CA 신뢰 등록 남음 |

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

첫 실행에서 **Stage 1의 MAS Core가 `[FAILURE]`였는데 재시도되지 않은 상태**였습니다. `[SUCCESS] Selected Dependencies`와 같은 로그에 섞여 있어 놓쳤고, §2.4의 검증이 `[SUCCESS]`만 세도록 되어 있어 걸러지지 않았습니다. 그대로 반입했다면 §3.9 `mas install`에서 막혔을 상황입니다.

재실행으로 복구했고(Core 15GB → 16GB), 문서에 세 가지를 반영했습니다.

- `[FAILURE]`가 0건인지 확인하도록 검증 기준 변경 (성공 개수만 세지 않음)
- `grep` 패턴에서 `^` 앵커 제거 — `mas` CLI 출력에 ANSI 색상 코드가 앞에 붙어 매칭되지 않음
- 단계마다 실행·진행 확인·검증을 짝지어 배치

**Maximo IT는 상태 목록에 별도 줄이 없습니다** — Manage 확장이라 Manage 매니페스트에 포함됩니다. `mirror/apps/v2/cp/manage/extension-icd` 실물로 확인했습니다.

### 반입 전 발견한 IBM CLI 버그

반입하기 전에 `mirror_ocp` 역할의 `from-filesystem.yml`을 열어 확인한 결과, §3.5가 찾는 파일명이 산출물과 다릅니다.

| | |
|---|---|
| 역할이 하드코딩한 이름 | `mirror_seq1_000000.tar` |
| 내장 oc-mirror(4.22.0)가 만드는 이름 | `mirror_000001.tar` |

`to-filesystem.yml`은 파일명을 다루지 않고 `oc mirror`를 그대로 호출하므로, 래퍼로만 진행해도 어긋납니다 — **IBM CLI 23.4.1 자체의 버전 불일치**입니다.

**반입 전에는 손대지 않기로 했습니다.** 지금 바꾸면 `transfer-files.sha256`과 어긋나 검증이 복잡해집니다. 체크섬 검증을 마친 뒤 §3.5의 2)에서 `mv`로 처리합니다. 상세는 [OFFLINE_INSTALL.md §4.5](OFFLINE_INSTALL.md).

같은 파일에서 래퍼가 `--dest-tls-verify=false`와 `--parallel-images=1`을 붙인다는 것도 확인해 §3.5에 기록했습니다.

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

## §3 설치 — §3.4 진행 중

| 절 | 상태 |
|---|---|
| 3.1 RPM 저장소 등록·도구 설치 | ✅ |
| 3.2 DNS(dnsmasq)·NTP(chrony) | ✅ 폐쇄망 상태로 검증 — 상위 DNS 차단 후 내부 이름만 해석되는지, NTP가 `0.0.0.0:123` Stratum 10으로 서비스하는지 확인 |
| 3.3 MAS CLI 이미지 로드 | ✅ `quay.io/ibmmas/cli:23.4.1` |
| 3.4 Mirror Registry 설치 | 🔄 Quay 기동 완료(`failed=0`), **CA 신뢰 등록·검증·읽기 전용 계정 생성 남음** |
| 3.5 이후 | ⬜ |

계정 정보는 [SERVER_INFO.md](SERVER_INFO.md)의 **계정** 절에 있습니다.

### §3.4에서 확인한 것

**`mirror-registry install`은 SSH 키를 자동 생성하지 않습니다.** `~/.ssh/`가 없는 상태로 실행하면 이미지 로드 직후 `Could not find ssh key at ~/.ssh/quay_installer`로 중단됩니다. `-v` 없이 실행하면 이 메시지만 덜렁 나와 원인 파악이 어렵습니다. 자기 자신에게 설치해도 ansible이 SSH로 접속하므로 키 생성 + `authorized_keys` 등록이 전제 조건입니다 — §3.4의 2)로 문서에 추가했습니다.

`--targetHostname`을 IP로 준 것도 유효했습니다. 기본값인 호스트명이 IPv6 링크로컬(`fe80::`)로만 해석되는 상태였습니다.

설치 소요 **약 15분**. `Copy Images`(2.5GB를 SSH로 전송)와 Quay 첫 기동 대기(`FAILED - RETRYING` 2회)가 대부분입니다. 두 구간 모두 출력이 멈춘 것처럼 보이므로 진행 확인 명령을 문서에 넣었습니다.

설치 프로그램이 마지막에 `Enable lingering`을 수행합니다 — Quay는 rootless systemd user 서비스라서 §2.3 0번의 `loginctl enable-linger`와 같은 이유입니다.

## 실행 전 확인이 필요한 항목

### 실행하면서 즉시 확인 가능

| 항목 | 확인 방법 | 관련 절 |
|---|---|---|
| `from-filesystem`이 `working-dir`도 필요한지 (tar 외에) | §3.5 첫 실행 | §3.5 |
| `mas configure-airgap --setup-redhat-catalogs` 실존 여부 | `mas configure-airgap --help` | §3.7.1 |
| `imageDigestSources` vs `imageContentSources` | `openshift-install explain installconfig \| grep -i image` | §3.6 |
| `LVMCluster`의 `apiVersion` | `oc explain lvmcluster` | §3.8.1 |
| NFS 프로비저너 SCC 부여 필요 여부 | 배포 후 Pod 상태 | §3.8.2 |

### 공식 문서를 브라우저로 직접 확인해야 하는 것

| 항목 | 비고 |
|---|---|
| 🔴 **한국어 VARGRAPHIC 설정 위치** | ansible-devops `db2` role에 문자셋 전용 변수 없음. Db2 레벨인지 Manage 활성화 레벨인지 불명. **되돌리기 가장 어려운 결정** |
| **Maximo IT AppPoints 권한** | 라이선스 파일 보유가 IT 권한 보유를 의미하지 않음. MAS Entitlement와 Maximo IT Entitlement가 각각 필요 |
| Manage + Maximo IT 배포 절차 (§3.10) | IBM 지식센터가 자동 조회 환경에서 403. Db2 ROW·Server Bundle 등 일부만 검증됨 |
| Manage가 내부 Image Registry를 쓰는지 | §3.8.3의 용량 산정에 영향 |
| SNO에서 PTR(역방향) DNS 필수 여부 | §3.2 |

## 구조적 리스크 (의사결정 필요)

| 항목 |
|---|
| 🔴 **사이트 Bastion 디스크 1.5TB 필요** — 반입 489GB + Registry 사본 약 489GB. 현재 500GB이므로 증설 필수 |
| `rhods-operator`(OpenShift AI)가 Red Hat 콘텐츠 373GB의 대부분. MAS 미사용이고 IBM imageset에서 *Optional* 로 분류. 제거하면 양쪽 서버 용량이 크게 줄지만 재미러링 6시간 |
| `mirror registry for Red Hat OpenShift`는 OCP 설치용 소규모·비HA Registry — MAS 장기 운영 Registry로는 지원 범위 밖 |
| Bastion이 Mirror Registry + DNS + NTP + NFS를 모두 겸함 → 단일 장애점 |
| NFS 동적 프로비저너는 커뮤니티 프로젝트 — Red Hat/IBM 미지원 |
