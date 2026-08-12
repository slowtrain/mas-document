# Maximo IT 오프라인 설치 — 진행 상황

설치 절차는 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md)를 참고하세요. 이 문서는 어디까지 진행됐는지만 기록합니다.

**최종 갱신**: 2026-08-08

## 설치 완료 (2026-08-07)

`inst1-install-260807-1016` 파이프라인으로 MAS Core + Manage + Maximo IT 설치를 완료했고 웹 UI에서 동작을 확인했습니다.

| 항목 | 결과 |
|---|---|
| MAS Core / Manage | 9.2.0 |
| 컴포넌트 | `base` + **`icd`** (Maximo IT) |
| 언어 | base `EN` + **`KO`** |
| 데모 데이터 | 적재됨 |
| 타임존 | Manage · Db2 모두 `Asia/Seoul` |
| Db2 | 전용 인스턴스, `dftTableOrg: ROW`, `db2Vargraphic: true` |

접속 주소·계정은 [ACCESS.md](ACCESS.md), 재설치·초기화 절차는 [OFFLINE_INSTALL.md §3.10.3, §3.10.5](OFFLINE_INSTALL.md)를 보세요.

### 남은 작업

- [ ] 정식 관리자 계정 생성 — Superuser는 CLI가 랜덤 생성한 값이며 초기 구성 전용
- [ ] Db2 라이선스 — **평가판 90일**(`SQL8007W`). IBM에 활성화 키 확인 후 등록
- [ ] 🔴 사내 DNS — `*.apps` 와일드카드 등록 (없으면 사용자 PC마다 `hosts` 필요)
- [ ] 🔴 사내 CA — 자체 서명 인증서 교체 또는 루트 CA 배포
- [ ] Maximo IT AppPoints 권한 확인

---

## 요약

| 단계 | 상태 |
|---|---|
| §2 사전준비 (인터넷 연결 서버) | ✅ **완료** — tar 조각 25개 준비됨, FileZilla 반출 대기 |
| §3 설치 (사이트 Bastion) | ✅ **완료** — Core + Manage + Maximo IT, 언어·데모 데이터·타임존 반영 |

## §2 사전준비 — 완료

| # | 항목 | 결과 |
|---|---|---|
| 0 | 디렉터리 + linger | ✅ `Linger=yes` |
| 1 | IBM Entitlement Key | ✅ |
| 2 | MAS 라이선스 | ✅ ⚠️ Maximo IT AppPoints 포함 여부 미확인 |
| 3 | Red Hat Pull Secret | ✅ |
| 4 | RPM 오프라인 저장소 | ✅ 323개 / 168MB, 서명 정상, `repoclosure` 미해결 0건 |
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

상세는 [TROUBLE_SHOOTING.md 3, 4](TROUBLE_SHOOTING.md).

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

**리네임은 하지 않기로 했습니다.** §2.3 8번에서 이미 `oc mirror`를 직접 호출했으므로 §3.6도 대칭으로 직접 호출합니다 — 파일명이 무관해지고 `transfer-files.sha256`과도 어긋나지 않습니다. 상세는 [TROUBLE_SHOOTING.md 5](TROUBLE_SHOOTING.md).

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

## §3 설치 — 완료

| 절 | 상태 |
|---|---|
| 3.1 RPM 저장소 등록·도구 설치 | ✅ |
| 3.2 DNS (dnsmasq) | ✅ 폐쇄망 상태로 검증 — 상위 DNS 차단 후 내부 이름만 해석되는지 확인 |
| 3.3 NTP (chrony) | ✅ `0.0.0.0:123` Stratum 10으로 서비스 |
| 3.4 MAS CLI 이미지 로드 | ✅ `quay.io/ibmmas/cli:23.4.1` |
| 3.5 Mirror Registry (Quay) 설치 | ✅ 컨테이너 3개 `Up`, `curl … /v2/` → `401`(`-k` 없이), `podman login` → `Login Succeeded!` |
| 3.6 이미지 Push | ✅ Red Hat 373GB + MAS 4단계, `[FAILURE]` 0건. Quay Organization 31개 |
| 3.7 SNO 설치 | ✅ `4.20.30` / 노드 `Ready` / `sdb` 500GB 빈 상태 확인 |
| 3.8 Airgap(폐쇄망) 구성 | ✅ IDMS 33건 + CatalogSource 3건. ⚠️ `icr.io/cpopen` 정책 충돌 해소 필요 (§3.8.2 4단계) |
| 3.9 스토리지 준비 | ✅ RWO `lvms-vg1` / RWX `nfs-client` / 쓰기 테스트 통과 / 내부 Image Registry `Managed` |
| 3.10 MAS 설치 (Core + Manage + Maximo IT) | ✅ `inst1-install-260807-1016` 완료. 언어 `EN`+`KO`, 데모 데이터, 타임존 `Asia/Seoul` |
| 3.11 접속·확인 (웹 UI) | ✅ `hosts` 등록 후 웹 UI에서 메뉴·한국어·데모 데이터 확인. ⬜ 사내 DNS·CA는 운영 전환 시 필요 |

### §3.10에서 확인한 것

**대화형 프롬프트를 전부 실측해 [OFFLINE_INSTALL.md §3.10.2](OFFLINE_INSTALL.md)에 순서대로 기록했습니다.** 예상 목록만으로는 따라갈 수 없어 화면 문구 기준으로 다시 작성했습니다.

🔴 **13.1 `Create Manage dedicated Db2 instance ...` 에서 `n`을 고르면 안 됩니다.** 화면 설명은 "시스템 Db2 공유 vs 전용"으로 읽히지만 실제 `n`은 **외부 DB를 JDBC로 연결**하는 경로이고, `JDBC Connection String`을 직접 입력하라고 요구합니다. 외부 DB가 없으면 `y`로 전용 Db2를 만들어야 합니다.

**`$HOME`을 컨테이너에 마운트할 수 없습니다** — `Error: SELinux relabeling of /home/maximo is not allowed`. 라이선스 디렉터리(`:ro`)와 설정 디렉터리(쓰기 가능)를 따로 마운트해야 합니다. `Select Local configuration directory`가 JDBC 설정 파일을 **생성**하므로 읽기 전용 경로를 주면 거기서 막힙니다.

**Entitlement Key 검증은 실패하는 게 정상입니다.** CLI가 `cp.icr.io`로 확인하려는데 폐쇄망이라 나갈 수 없습니다 — `2` (Continue anyway)로 넘깁니다.

**폐쇄망에서는 CLI 이미지 다이제스트를 직접 입력해야 합니다.**

```bash
podman image inspect quay.io/ibmmas/cli:23.4.1 --format '{{.Digest}}'
```

입력하면 Tekton 정의가 `quay.io/ibmmas/cli@sha256:...` 형태로 변환되고, §3.8.1의 IDMS가 이를 미러로 리다이렉트합니다.

**Maximo IT는 `11.1) Maximo Manage Components`에서 켭니다.** 기본값은 Health만 활성화라 그대로 두면 빠집니다. 목록 중간(`Maximo IT`)에 있어 무심코 넘기기 쉽습니다. 제대로 선택되면 `11.2) Maximo IT License Terms` 화면이 추가로 나옵니다.

🔴 **대화형(§3.10.2)은 언어·데모 데이터를 묻지 않고, 타임존은 입력해도 반영되지 않습니다.** 처음 설치에서 이것들이 빠져 재설치해야 했습니다. **§3.10.3 비대화형으로 플래그를 넘기는 것이 유일하게 확실한 방법**입니다.

```
--manage-base-language EN --manage-secondary-languages "KO"
--manage-demodata
--manage-server-timezone Asia/Seoul --db2-timezone Asia/Seoul
```

Db2 자원도 화면 표시(CPU 4000m / 볼륨 425Gi)와 달리 `server-bundle-size: dev` 기준(**300m / 60Gi**)이 적용됩니다. SNO에는 이쪽이 적합합니다.

**`db2wh`의 row-organized 문제는 없었습니다.** IBM 역할이 Db2uCluster CR에 `dftTableOrg: ROW`를 명시적으로 넣습니다 — 미확정으로 남겨뒀던 항목이 해소됐습니다.

**한국어 VARGRAPHIC도 자동으로 켜집니다.** `ManageWorkspace` CR에 `db2Vargraphic: true`가 기본값으로 들어갑니다.

최종 `ManageWorkspace` 설정값입니다.

| 항목 | 값 |
|---|---|
| `baseLang` / `secondaryLangs` | `EN` / **`["KO"]`** |
| `demodata` | **`true`** |
| `serverTimezone` | **`Asia/Seoul`** |
| `db2Vargraphic` | `true` |
| `dbSchema` / `tableSpace` / `indexSpace` | `maximo` / `MAXDATA` / `MAXINDEX` |
| `serverBundles` | `all` 1개, replica 1 |
| `components` | `base` + **`icd`** |

⚠️ `app-cfg-manage`의 워크스페이스 대기 타임아웃은 **18시간**입니다(360초 × 180회). Manage 이미지 빌드와 DB 스키마 생성(maxinst)이 이 구간에서 일어납니다.

**재설치할 때는 잔여 오브젝트를 반드시 지워야 합니다.** `db2u`의 `mas-inst1-ws1-manage-enforce-config` ConfigMap을 남겼다가 Db2 설정 적용 단계가 통째로 스킵되어 파이프라인이 멈췄습니다. 절차는 [OFFLINE_INSTALL.md §3.10.5](OFFLINE_INSTALL.md)에 있습니다.

**SNO 16코어로는 부족합니다.** 설치 직후 CPU 요청이 93%라 Manage 이미지 빌드 Pod이 `Insufficient cpu`로 스케줄되지 못했습니다. **20코어로 증설**한 뒤 해결됐습니다(§SERVER_INFO).

### §3.9에서 확인한 것

**NFS 프로비저너는 SCC 부여가 필수입니다.** 기본 `restricted` SCC가 NFS 볼륨을 허용하지 않아 **Pod 생성 자체가 거부**됩니다(`oc get pods`에 아무것도 안 나옴). 문서에 "필요 여부 미검증"으로 남아 있던 항목이 필수로 확정됐습니다.

```
spec.volumes[0]: Invalid value: "nfs": nfs volumes are not allowed to be used
```

```bash
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-client-provisioner -n nfs-provisioner
```

**§3.9.3(스토리지 검증)을 §3.9.4(내부 Image Registry)보다 먼저 하도록 순서를 바꿨습니다.** Image Registry가 200Gi NFS PVC를 만드는데, NFS 쓰기가 안 되는 상태에서 진행하면 원인이 스토리지인지 레지스트리 설정인지 구분되지 않습니다.

`lvms-vg1`은 `WaitForFirstConsumer`라 **PVC만 만들면 `Pending`이 정상**입니다 — Pod이 붙어야 볼륨이 생성됩니다.

스토리지 쓰기 테스트 이미지는 **`ibmmas/cli:latest`** 를 씁니다. `rhel9/support-tools:latest`는 `repository not found`가 납니다 — `oc debug`가 그 이미지를 쓰긴 하지만 릴리스 페이로드의 **다이제스트**로 당기는 것이라 미러에 `latest` 태그가 없습니다.

`platform: none` 설치는 내부 Image Registry가 `managementState: Removed`, 스토리지 `{}` 상태로 시작합니다. §3.11 Manage 활성화가 이미지를 빌드해 push하므로 반드시 `Managed`로 전환해야 합니다. PVC 200Gi는 미검증 예시값이며 NFS는 thin provisioning이라 실사용분만 차지합니다.

### §3.8에서 확인한 것

**`mas configure-airgap`에는 `--setup-redhat-catalogs`·`--setup-redhat-release` 플래그가 실제로 있습니다** — 오랫동안 미검증으로 남아 있던 항목이 해소됐습니다. 다만 쓰지 않았습니다. IBM CLI가 만드는 IDMS·CatalogSource는 일반형인데 우리 미러는 prefix 없이 업스트림 네임스페이스를 그대로 쓰는 레이아웃이라, `oc mirror`가 생성한 `cluster-resources/`가 경로상 정확합니다.

**CLI는 클러스터에 로그인돼 있지 않으면 `--help`조차 출력하지 않습니다.** kubeconfig를 컨테이너에 마운트하는 방식이 `oc login --token`보다 낫습니다 — 토큰을 뽑을 필요도, 만료도 없습니다.

MachineConfig 롤아웃은 SNO에서 **1~2분 만에 끝나 `UPDATING=True`를 못 볼 수 있습니다.** 완료 판정은 `oc get mcp`의 `CONFIG` 이름이 새로 바뀌었는지로 합니다. 노드의 `/etc/containers/registries.conf`에 `[[registry]]` 블록이 34개(IDMS 33 + ITMS 1) 있으면 반영된 것입니다.

IDMS 오브젝트는 네 개이며 출처가 다릅니다.

| 이름 | 출처 |
|---|---|
| `image-digest-mirror` | §3.7 `install-config.yaml`의 `imageDigestSources` |
| `idms-release-0` / `idms-operator-0` | §3.8.1 `cluster-resources/` |
| `mas-ibm-catalog` | §3.8.2 `mas configure-airgap` |

미러 Pull 테스트 이미지는 **`ibmmas/cli:latest`** 입니다. `:23.4.1`로 시도하면 `manifest unknown`이 납니다 — `mas mirror-images`가 `latest` 태그로 올립니다.

### §3.7에서 확인한 것

**`install-config.yaml`의 `imageDigestSources`에는 release 미러 2건만 넣습니다.** IDMS 34건을 그대로 옮기면 `openshift-install` 검증이 `docker.io/grafana`를 거부합니다 — IDMS CRD보다 설치 프로그램 쪽 검증이 엄격합니다. 오퍼레이터 미러는 §3.8에서 `oc apply`로 적용하므로 버리는 것이 아니라 시점을 미루는 것입니다([TROUBLE_SHOOTING.md 6](TROUBLE_SHOOTING.md)).

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

계정 정보는 [ACCESS.md](ACCESS.md)에 있습니다.

### §3.5에서 확인한 것

**`mirror-registry install`은 SSH 키를 자동 생성하지 않습니다.** `~/.ssh/`가 없는 상태로 실행하면 이미지 로드 직후 `Could not find ssh key at ~/.ssh/quay_installer`로 중단됩니다. `-v` 없이 실행하면 이 메시지만 덜렁 나와 원인 파악이 어렵습니다. 자기 자신에게 설치해도 ansible이 SSH로 접속하므로 키 생성 + `authorized_keys` 등록이 전제 조건입니다 — §3.5.2로 문서에 추가했습니다.

`--targetHostname`을 IP로 준 것도 유효했습니다. 기본값인 호스트명이 IPv6 링크로컬(`fe80::`)로만 해석되는 상태였습니다.

설치 소요 **약 15분**. `Copy Images`(2.5GB를 SSH로 전송)와 Quay 첫 기동 대기(`FAILED - RETRYING` 2회)가 대부분입니다. 두 구간 모두 출력이 멈춘 것처럼 보이므로 진행 확인 명령을 문서에 넣었습니다.

설치 프로그램이 마지막에 `Enable lingering`을 수행합니다 — Quay는 rootless systemd user 서비스라서 §2.3 0번의 `loginctl enable-linger`와 같은 이유입니다.

## 실행 전 확인이 필요한 항목

### 실행하면서 즉시 확인 가능

**해소된 항목** — 실행으로 확인했습니다.

| 항목 | 결과 |
|---|---|
| ~~한국어 VARGRAPHIC 설정 위치~~ | ✅ `ManageWorkspace`의 `db2Vargraphic: true` — **기본값으로 자동 적용** |
| ~~Db2 `db2wh`의 row-organized 문제~~ | ✅ Db2uCluster CR에 `dftTableOrg: ROW` 자동 설정 |
| ~~Manage가 내부 Image Registry를 쓰는지~~ | ✅ 사용함. `admin-build-config` / `all-build-config` 두 빌드가 push |
| ~~NFS 프로비저너 SCC 부여 필요 여부~~ | ✅ **필수** — 없으면 Pod 생성 자체가 거부 |
| ~~`from-filesystem`이 `working-dir`도 필요한지~~ | ✅ IBM 래퍼를 쓰지 않고 `oc mirror` 직접 호출로 해결([TROUBLE_SHOOTING.md 5](TROUBLE_SHOOTING.md)) |
| ~~`imageDigestSources` vs `imageContentSources`~~ | ✅ `imageDigestSources` 사용, release 2건만([TROUBLE_SHOOTING.md 6](TROUBLE_SHOOTING.md)) |
| ~~`LVMCluster`의 `apiVersion`~~ | ✅ 확인 완료 |
| ~~`mas configure-airgap`의 Red Hat 플래그 실존 여부~~ | ✅ 존재하나 미사용 — `oc mirror` 산출물이 경로상 정확 |

**남은 미확정 항목**

| 항목 | 비고 |
|---|---|
| 🔴 **사내 DNS 존재 여부** | 없으면 사용자 PC마다 `hosts` 등록이 필요해 운영 불가. `*.apps` 와일드카드 등록 또는 Bastion dnsmasq 위임 필요(§3.11.5) |
| 🔴 **사내 CA 존재 여부** | 자체 서명 인증서는 호스트마다 브라우저 경고가 뜨고, `api.inst1…` 을 수락하지 않으면 Suite Administration이 로딩에서 멈춤. 사내 CA 발급 인증서 또는 루트 CA 배포 필요(§3.11.5) |
| **Maximo IT AppPoints 권한** | 라이선스 파일 보유가 IT 권한 보유를 의미하지 않음. 실제 사용 시 확인 필요 |
| **한국어 추가 언어 적용 방법** | `baseLang: EN` / `secondaryLangs: []` 로 설치됨. 한국어를 쓰려면 추가 절차 필요 |
| **Server Timezone 반영 방법** | 설치 시 `Asia/Seoul` 입력이 무시되고 `GMT`로 들어감. `ManageWorkspace` CR 수정으로 되는지 미확인 |
| **Superuser 계정 변경 방법** | 시크릿 수정 후 MAS가 재기동 시 다시 읽는지 미확인. 정식 관리자 계정 생성이 권장 경로 |
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
