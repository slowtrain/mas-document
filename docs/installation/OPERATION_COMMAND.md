# OpenShift / MAS 운영 명령어

이번 배포(`mas-it.itmsg.co.kr`, SNO 단일 노드) 기준입니다. 설치 절차는 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md), 서버 정보는 [SERVER_INFO.md](SERVER_INFO.md), 접속 주소·계정은 [ACCESS.md](ACCESS.md)를 보세요.

<details>
<summary><b>목차</b></summary>

- [1. 기본 명령어](#1-기본-명령어)
  - [1.1 접속 준비](#11-접속-준비)
  - [1.2 조회](#12-조회)
  - [1.3 상세 확인](#13-상세-확인)
  - [1.4 Pod 접속](#14-pod-접속)
- [2. 로그 확인](#2-로그-확인)
  - [2.1 컨테이너 로그](#21-컨테이너-로그)
  - [2.2 Manage 서버 로그](#22-manage-서버-로그)
  - [2.3 배포 과정 로그](#23-배포-과정-로그)
  - [2.4 설치 파이프라인 로그](#24-설치-파이프라인-로그)
- [3. 실행 · 제어](#3-실행--제어)
  - [3.1 재시작 · 스케일](#31-재시작--스케일)
  - [3.2 삭제 · 재실행](#32-삭제--재실행)
  - [3.3 노드 접속](#33-노드-접속)
- [4. 상태 확인](#4-상태-확인)
  - [4.1 클러스터](#41-클러스터)
  - [4.2 MAS](#42-mas)
  - [4.3 Db2](#43-db2)
  - [4.4 스토리지](#44-스토리지)
  - [4.5 오퍼레이터](#45-오퍼레이터)
  - [4.6 노드 자원](#46-노드-자원)
- [5. 접속 주소 · 계정](#5-접속-주소--계정)
  - [5.1 Route · Service](#51-route--service)
  - [5.2 Secret · ConfigMap](#52-secret--configmap)
- [6. 폐쇄망](#6-폐쇄망)
  - [6.1 미러 레지스트리 · IDMS](#61-미러-레지스트리--idms)
  - [6.2 MachineConfig (MCO)](#62-machineconfig-mco)
- [7. 자주 쓰는 조합](#7-자주-쓰는-조합)

</details>

---

## 1. 기본 명령어

### 1.1 접속 준비

kubeconfig를 지정하면 로그인이 필요 없습니다.

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig
echo 'export KUBECONFIG=~/ocp-sno/auth/kubeconfig' >> ~/.bashrc
```

토큰 로그인도 가능합니다.

```bash
oc login -u kubeadmin -p <비밀번호> https://api.mas-it.itmsg.co.kr:6443
cat ~/ocp-sno/auth/kubeadmin-password
```

```bash
oc whoami
oc whoami --show-server
oc auth can-i '*' '*' --all-namespaces      # yes 면 cluster-admin
```

`You must be logged in to the server` 는 세션 만료입니다. `KUBECONFIG` 를 다시 지정하세요.

### 1.2 조회

```bash
oc get ns
oc project                                 # 현재 네임스페이스
oc project mas-inst1-core                  # 기본 네임스페이스 변경
```

```bash
oc get pods -n mas-inst1-manage
oc get pods -A
oc get deploy,statefulset,replicaset -n mas-inst1-core
oc get secret -n mas-inst1-core
oc get route -A
```

| 옵션 | 뜻 |
|---|---|
| `-n <ns>` | 특정 네임스페이스 |
| `-A` | 전체 네임스페이스 |
| `-o wide` | 컬럼 추가 (Pod은 노드·IP 포함) |
| `-o yaml` / `-o json` | 전체 정의 |
| `-w` | 변화를 계속 지켜봄 |
| `--sort-by=<필드>` | 정렬 |
| `-l <키>=<값>` | 라벨 필터 |
| `--field-selector status.phase=Running` | 상태 필터 |

**정상이 아닌 것만 보기** — 운영 중 가장 자주 씁니다.

```bash
oc get pods -A | grep -vE 'Running|Completed'
```

| STATUS | 뜻 |
|---|---|
| `Running` | 정상 |
| `Completed` | 끝난 Job/Init Pod — 정상 |
| `Pending` | 스케줄링 대기 — 자원 부족, PVC 미바인딩 |
| `ContainerCreating` | 이미지 Pull·볼륨 마운트 중 |
| `Init:0/1` | Init 컨테이너 실행 중 |
| `ImagePullBackOff` / `ErrImagePull` | 이미지를 못 받음 — 폐쇄망 미러·IDMS 문제 |
| `CrashLoopBackOff` | 기동 직후 죽고 재시작 반복 — 로그 확인 |

리소스 종류가 기억나지 않을 때입니다.

```bash
oc api-resources | grep -i mas
oc explain manageworkspace.spec
```

### 1.3 상세 확인

`describe` 는 상태와 이벤트를 함께 보여줍니다. **문제가 생기면 가장 먼저 보는 명령**입니다.

```bash
oc describe pod <pod-name> -n <ns>
oc describe pvc <pvc-name> -n <ns>
```

이벤트는 Pod이 안 뜰 때·PVC가 안 붙을 때 원인이 적혀 있습니다.

```bash
oc get events -n <ns> --sort-by=.lastTimestamp | tail -20
oc get events -A --field-selector type=Warning --sort-by=.lastTimestamp | tail -20
```

### 1.4 Pod 접속

```bash
oc rsh -n db2u c-mas-inst1-ws1-manage-db2u-0
oc exec -n db2u c-mas-inst1-ws1-manage-db2u-0 -- df -h
oc cp "<ns>/<pod>:/logs/messages.log" ./messages.log -c all
```

🔴 **Pod 이름은 재생성될 때마다 바뀝니다.** 이름을 직접 적지 말고 컨트롤러 이름이나 변수를 쓰세요.

```bash
oc rsh -n mas-inst1-manage deploy/inst1-ws1-all

POD=$(oc get pods -n mas-inst1-manage -o name | grep -- '-all-' | grep -v build | head -1)
oc exec -n mas-inst1-manage ${POD#pod/} -c all -- ls /logs
```

⚠️ **`oc rsh deploy/...` 는 Ready 상태인 Pod을 기다립니다.** 기동 중에는 `error: timed out waiting for the condition` 이 납니다. 그때는 위처럼 `oc exec` 에 Pod 이름을 직접 넘기세요.

컨테이너가 여러 개면 `-c` 로 지정합니다.

```bash
oc get pod <pod-name> -n <ns> -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

---

## 2. 로그 확인

### 2.1 컨테이너 로그

```bash
oc logs <pod-name> -n <ns> --tail=50
oc logs <pod-name> -n <ns> -f                 # 실시간
oc logs <pod-name> -n <ns> --previous         # 재시작 전 (CrashLoop 원인)
oc logs <pod-name> -n <ns> -c manager         # 컨테이너 지정
```

Pod 이름 대신 컨트롤러·라벨로도 됩니다(`-f` 는 단일 Pod만 가능).

```bash
oc logs -n mas-inst1-core deploy/ibm-mas-operator --tail=30
oc logs -n mas-inst1-manage -l mas.ibm.com/serverBundle=all --tail=50 --prefix
```

### 2.2 Manage 서버 로그

가장 자주 보는 로그입니다. **서버가 정상 기동된 상태**라면 이 한 줄입니다.

```bash
oc rsh -c all -n mas-inst1-manage deploy/inst1-ws1-all tail -n 100 -F /logs/messages.log
```

🔴 **`oc logs` 로는 안 보입니다.** 컨테이너 표준출력에는 대기 루프만 찍히고, 실제 Liberty·Maximo 로그는 Pod 안 `/logs/` 에 있습니다.

```
Oslc servlet.... Server did not started yet. Will wait 30 sec.
```

`-F` 는 로그 롤링 시 새 파일을 다시 잡습니다. `-f` 는 이전 파일을 붙들고 있어 로그가 끊긴 것처럼 보입니다.

**서버에 문제가 있으면 위 명령부터 실패합니다.** 에러 문구로 원인이 갈립니다.

| 에러 | 상태 | 대응 |
|---|---|---|
| `timed out waiting for the condition` | Pod은 있으나 Ready 아님 — 기동 중이거나 probe 실패 | 아래 `oc exec` |
| `container not found ("all")` | Pod 생성 직후, 컨테이너 아직 미기동 | 잠시 후 재시도 |
| Pod이 아예 없음 | Deployment가 `0/0` — 배포 진행 중 | [§2.3](#23-배포-과정-로그) |

먼저 상태를 봅니다.

```bash
oc get deploy inst1-ws1-all -n mas-inst1-manage
oc get pods -n mas-inst1-manage | grep -- '-all-' | grep -v build
```

Ready가 아니어도 Pod이 `Running` 이면 `oc exec` 로 붙을 수 있습니다. **기동 실패 원인을 볼 때 쓰는 형태입니다.**

```bash
POD=$(oc get pods -n mas-inst1-manage -o name | grep -- '-all-' | grep -v build | head -1)
oc exec -n mas-inst1-manage ${POD#pod/} -c all -- tail -n 100 -F /logs/messages.log
```

```bash
oc rsh -c all -n mas-inst1-manage deploy/inst1-ws1-all ls -la /logs/
oc rsh -c all -n mas-inst1-manage deploy/inst1-ws1-all tail -100 /logs/messages.log
```

에러만 추립니다.

```bash
oc rsh -c all -n mas-inst1-manage deploy/inst1-ws1-all \
  grep -iE 'error|exception|ClassNotFound|NoClassDef' /logs/messages.log | tail -40
```

alias를 걸어두면 편합니다.

```bash
echo "alias maslog='oc rsh -c all -n mas-inst1-manage deploy/inst1-ws1-all tail -n 100 -F /logs/messages.log'" >> ~/.bashrc
source ~/.bashrc
```

기동 완료 표식입니다.

```
CWWKF0011I: The defaultServer server is ready to run a smarter planet.
```

로그 시각은 `serverTimezone` 을 따릅니다. 이번 배포는 `Asia/Seoul` 이라 한국 시각입니다.

### 2.3 배포 과정 로그

Manage는 배포 시 이미지를 **직접 빌드해 내부 Image Registry에 push**합니다. 커스터마이징을 반영하면 **admin → all** 순서로 두 번 돕니다.

| 빌드 | 내용 | 소요 |
|---|---|---|
| `admin-build-config-*` | Admin 번들 | ~10분 |
| `all-build-config-*` | 서버 번들 — EAR을 굽는 단계 | ~15분 |

```bash
oc get build -n mas-inst1-manage

BUILD=$(oc get build -n mas-inst1-manage --sort-by=.metadata.creationTimestamp -o name | tail -1)
oc logs -f $BUILD -n mas-inst1-manage --tail=30
```

빌드 직후 `Pending` 은 정상입니다. 빌더 Pod이 admin 이미지를 빌드 컨텍스트로 풀어내는(`extract-image-content`) 동안 몇 분 걸립니다.

```bash
oc describe pod -n mas-inst1-manage -l openshift.io/build.name=${BUILD##*/} | tail -25
```

`Events` 마지막 줄로 갈립니다 — `FailedScheduling` 은 자원, `ImagePullBackOff` 는 레지스트리, `no space left on device` 는 노드 디스크입니다.

빌드 뒤에는 `inst1-ws1-all` 이 `0/0` 으로 내려가고 `maxinst` 가 다시 돕니다. 이 구간에는 Manage 서버가 없습니다.

```bash
oc logs -n mas-inst1-manage deploy/inst1-ws1-manage-maxinst -f --tail=30
watch -n 15 'oc get pods -n mas-inst1-manage | grep -E "maxinst|-all-"; oc get deploy inst1-ws1-all -n mas-inst1-manage'
```

내부 Image Registry가 `Removed` 면 빌드가 실패합니다.

```bash
oc get co image-registry
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.managementState}{"\n"}{.spec.storage}{"\n"}'
```

### 2.4 설치 파이프라인 로그

```bash
oc get pipelinerun -n mas-inst1-pipelines
oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp | tail -6
```

실행 중인 Task의 로그입니다. **컨테이너 이름이 Task마다 달라 `--all-containers` 를 씁니다.**

```bash
TR=$(oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{range .items[?(@.status.conditions[0].status=="Unknown")]}{.metadata.name}{"\n"}{end}' | tail -1)

oc logs -n mas-inst1-pipelines "$(oc get pod -n mas-inst1-pipelines -l tekton.dev/taskRun=$TR -o name | head -1)" \
  --all-containers --tail=30
```

웹 콘솔이 읽기 쉽습니다.

```
https://console-openshift-console.apps.mas-it.itmsg.co.kr/k8s/ns/mas-inst1-pipelines/tekton.dev~v1beta1~PipelineRun/<이름>
```

---

## 3. 실행 · 제어

### 3.1 재시작 · 스케일

```bash
oc rollout restart deploy/<이름> -n <ns>
oc rollout status deploy/<이름> -n <ns>
oc scale deploy/<이름> --replicas=0 -n <ns>   # 중지
oc scale deploy/<이름> --replicas=1 -n <ns>   # 재개
```

Pod을 지우면 컨트롤러가 새로 만듭니다 — 사실상 재시작입니다.

```bash
oc delete pod <pod-name> -n <ns>
```

Ready가 될 때까지 기다립니다.

```bash
oc wait --for=condition=Ready pod/<pod-name> -n <ns> --timeout=30m
```

### 3.2 삭제 · 재실행

실패한 PipelineRun을 지우고 다시 실행할 때입니다.

```bash
oc delete pipelinerun <이름> -n mas-inst1-pipelines
oc delete pipelinerun --all -n mas-inst1-pipelines
```

리소스 정의 백업입니다.

```bash
oc get suite inst1 -n mas-inst1-core -o yaml > suite-backup.yaml
oc get manageworkspace inst1-ws1 -n mas-inst1-manage -o yaml > mw-backup.yaml
```

### 3.3 노드 접속

```bash
ssh -i ~/.ssh/quay_installer core@192.168.2.211
oc debug node/mas-it-sno
```

`oc debug` 는 `registry.redhat.io/rhel9/support-tools` 를 받아오므로 IDMS 적용 후에만 동작합니다.

🔴 **노드 종료·재시작은 [OFFLINE_INSTALL.md §3.11.6](OFFLINE_INSTALL.md)을 따르세요. Db2를 먼저 정지하지 않으면 데이터베이스가 손상됩니다.**

---

## 4. 상태 확인

### 4.1 클러스터

```bash
oc get co | grep -vE 'True *False *False'      # 비정상 오퍼레이터만
oc get nodes
oc get nodes -o wide
```

### 4.2 MAS

```bash
oc get suite,manageapp,manageworkspace -A
```

| 네임스페이스 | 내용 |
|---|---|
| `mas-inst1-core` | MAS Core |
| `mas-inst1-manage` | Manage + Maximo IT |
| `mas-inst1-pipelines` | 설치 파이프라인 |
| `db2u` | Db2 |
| `mongoce` | MongoDB |
| `ibm-sls` | 라이선스 서비스 |
| `redhat-marketplace` | DRO |

Suite 상태 조건 — 어디서 막혔는지 알 수 있습니다.

```bash
oc get suite inst1 -n mas-inst1-core \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status} — {.message}{"\n"}{end}'
```

| 조건 | 뜻 |
|---|---|
| `SystemDatabaseReady` | MongoDB 연결 |
| `SLSIntegrationReady` | 라이선스 서비스 연동 |
| `BASIntegrationReady` | 사용 분석 연동 |
| `RoutesReady` | Route 생성 완료 |
| `Ready` | 위 조건 모두 충족 |

Manage 워크스페이스입니다. 이름이 기억나지 않으면 먼저 조회합니다.

```bash
oc get manageworkspace -n mas-inst1-manage
```

```bash
# 상태 조건
oc get manageworkspace inst1-ws1 -n mas-inst1-manage \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status} — {.message}{"\n"}{end}'

# 설치된 컴포넌트 (icd = Maximo IT)
oc get manageworkspace inst1-ws1 -n mas-inst1-manage -o jsonpath='{.spec.components}{"\n"}'

# 언어·타임존·DB 설정
oc get manageworkspace inst1-ws1 -n mas-inst1-manage -o jsonpath='{.spec.settings}' | jq

# 등록된 Customization Archive
oc get manageworkspace inst1-ws1 -n mas-inst1-manage -o jsonpath='{.spec.settings.customizations}{"\n"}'
```

Manage Pod 역할입니다.

| Pod | 역할 |
|---|---|
| `inst1-ws1-all-*` | **Maximo 서버 번들** — 업무 화면·API. 컨테이너는 `all`, `monitoragent` |
| `inst1-ws1-manage-maxinst-*` | DB 스키마 생성·`updatedb` |
| `ibm-mas-manage-operator-*` | Manage 오퍼레이터. 배포가 안 진행되면 여기 로그 |
| `inst1-entitymgr-*` | Entity Manager (워크스페이스·상태 관리) |

### 4.3 Db2

```bash
oc get db2ucluster -n db2u
oc get pods -n db2u
oc logs -n db2u c-mas-inst1-ws1-manage-db2u-0 --tail=30
```

```bash
oc rsh -n db2u c-mas-inst1-ws1-manage-db2u-0 bash -c \
  'su - db2inst1 -c "db2 connect to BLUDB; db2 list tablespaces"'
```

### 4.4 스토리지

```bash
oc get sc
oc get pvc -A
oc get pv
```

| StorageClass | 모드 | 실체 |
|---|---|---|
| `lvms-vg1` (default) | RWO | SNO 로컬 디스크 `/dev/sdb` 500GB |
| `nfs-client` | RWX | Bastion `/export/mas-rwx` |

⚠️ `lvms-vg1` 은 `WaitForFirstConsumer` 라 **Pod이 붙기 전에는 `Pending` 이 정상**입니다.

```bash
oc describe pvc <pvc-name> -n <ns> | tail -10
oc get lvmcluster -n openshift-storage
```

NFS는 Bastion에서 직접 봅니다.

```bash
showmount -e localhost
df -h /export/mas-rwx
```

### 4.5 오퍼레이터

```bash
oc get sub -A                              # Subscription
oc get csv -A                              # 설치된 버전
oc get installplan -A                      # 수동 승인 대기 확인
oc get catalogsource -n openshift-marketplace
```

| STATE / PHASE | 뜻 |
|---|---|
| `AtLatestKnown` | Subscription이 최신에 도달 |
| `Succeeded` | CSV 설치 완료 |
| `Pending` / `Installing` | 진행 중 |
| CSV 없음 | 설치가 시작조차 안 됨 — 채널·소스 이름 확인 |

폐쇄망에서 채널 이름을 확정할 때입니다.

```bash
oc get packagemanifest lvms-operator -n openshift-marketplace \
  -o jsonpath='{.status.defaultChannel}{"  전체: "}{range .status.channels[*]}{.name}{" "}{end}{"\n"}'
```

### 4.6 노드 자원

```bash
oc adm top nodes
oc adm top pods -A --sort-by=memory | head -20
oc describe node mas-it-sno | grep -A8 'Allocated resources'
```

---

## 5. 접속 주소 · 계정

### 5.1 Route · Service

```bash
oc get route -n mas-inst1-core
oc get route -n mas-inst1-manage
oc get svc -n <ns>
```

**`hosts` 파일에 넣을 줄을 통째로 생성**합니다.

```bash
oc get route -A -o jsonpath='{range .items[*]}192.168.2.211    {.spec.host}{"\n"}{end}' | awk 'NF==2' | sort -u
```

### 5.2 Secret · ConfigMap

Secret 값은 base64라 디코딩이 필요합니다.

```bash
oc get secret <이름> -n <ns> -o jsonpath='{.data.<키>}' | base64 -d; echo
oc get secret <이름> -n <ns> -o jsonpath='{.data}' | jq 'keys'      # 키 이름 확인
```

MAS Superuser 계정입니다.

```bash
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.username}' | base64 -d; echo
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.password}' | base64 -d; echo
```

전역 Pull Secret에 등록된 레지스트리 목록입니다.

```bash
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[]'
```

---

## 6. 폐쇄망

### 6.1 미러 레지스트리 · IDMS

```bash
oc get idms                                # ImageDigestMirrorSet
oc get itms                                # ImageTagMirrorSet
```

| 이름 | 출처 |
|---|---|
| `image-digest-mirror` | §3.7 `install-config.yaml` |
| `idms-release-0` / `idms-operator-0` | §3.8.1 `oc mirror` 산출물 |
| `mas-ibm-catalog` | §3.8.2 `mas configure-airgap` |

🔴 **노드에 실제로 반영됐는지는 `registries.conf` 로 확인해야 합니다.** IDMS 오브젝트가 있어도 MCO가 렌더링에 실패하면 반영되지 않습니다([TROUBLE_SHOOTING.md 7](TROUBLE_SHOOTING.md)).

```bash
ssh -i ~/.ssh/quay_installer core@192.168.2.211 'grep -c "^\[\[registry\]\]" /etc/containers/registries.conf'
ssh -i ~/.ssh/quay_installer core@192.168.2.211 'grep -A3 "quay.io/ibmmas" /etc/containers/registries.conf'
```

레지스트리 조회는 `podman` 으로 합니다.

```bash
podman login registry.mas-it.itmsg.co.kr:8443
podman image inspect registry.mas-it.itmsg.co.kr:8443/ibmmas/cli:latest --format '{{index .RepoDigests 0}}'
curl -sk -u init:<비밀번호> https://registry.mas-it.itmsg.co.kr:8443/v2/_catalog | jq -r '.repositories[]' | head -20
```

### 6.2 MachineConfig (MCO)

```bash
oc get mcp
oc get mc --sort-by=.metadata.creationTimestamp | tail -3
oc get co machine-config
```

| 컬럼 | 판정 |
|---|---|
| `UPDATED=True`, `UPDATING=False`, `DEGRADED=False` | 정상 |
| `UPDATING=True` | 롤아웃 중 — SNO는 재부팅됨 |
| `DEGRADED=True` | 실패 — 컨트롤러 로그 확인 |

🔴 **`UPDATED=True` 만으로는 반영 완료를 판정할 수 없습니다.** MCO가 새 설정을 **만들지 못하면** 갱신할 것이 없어 그대로 `True` 로 보입니다. 새 `rendered-master-*` 가 생겼는지 함께 봐야 합니다.

```bash
oc logs -n openshift-machine-config-operator deploy/machine-config-controller --tail=20 | grep -i error
```

---

## 7. 자주 쓰는 조합

**클러스터 전반 건강 확인**

```bash
oc get co | grep -vE 'True *False *False'
oc get nodes
oc get pods -A | grep -vE 'Running|Completed'
```

**MAS 전체 상태 한눈에**

```bash
oc get suite,manageapp,manageworkspace -A
oc get pods -n mas-inst1-core   | grep -vE 'Running|Completed'
oc get pods -n mas-inst1-manage | grep -vE 'Running|Completed'
```

**설치 진행 감시**

```bash
watch -n 30 'date; oc get pipelinerun -n mas-inst1-pipelines; echo; oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp | tail -5; echo; oc get suite,manageworkspace -A'
```

**커스터마이징 배포 감시**

```bash
watch -n 15 'oc get build -n mas-inst1-manage; echo; oc get pods -n mas-inst1-manage | grep -E "build|maxinst|-all-"; echo; oc get deploy inst1-ws1-all -n mas-inst1-manage'
```

**이미지 Pull 실패 원인 찾기**

```bash
oc get pods -A | grep -E 'ImagePull|ErrImage'
oc describe pod <pod-name> -n <ns> | grep -A5 -i failed
oc get idms
ssh -i ~/.ssh/quay_installer core@192.168.2.211 'grep -c "^\[\[registry\]\]" /etc/containers/registries.conf'
```

**자원 부족 확인**

```bash
oc adm top nodes
oc describe node mas-it-sno | grep -A8 'Allocated resources'
oc get pods -A --field-selector status.phase=Pending
```
