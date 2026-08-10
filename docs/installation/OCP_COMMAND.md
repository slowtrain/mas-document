# OpenShift / MAS 운영 명령어

이번 배포(`mas-it.itmsg.co.kr`, SNO 단일 노드) 기준입니다. 설치 절차는 [OFFLINE_INSTALL.md](OFFLINE_INSTALL.md), 서버 정보는 [SERVER_INFO.md](SERVER_INFO.md), 접속 주소·계정은 [ACCESS.md](ACCESS.md)를 보세요.

<details>
<summary><b>목차</b></summary>

- [0. 접속 준비](#0-접속-준비)
- [1. 조회의 기본](#1-조회의-기본)
- [2. Pod](#2-pod)
- [3. 로그](#3-로그)
- [4. Deployment · StatefulSet](#4-deployment--statefulset)
- [5. 이벤트 — 문제의 첫 단서](#5-이벤트--문제의-첫-단서)
- [6. 스토리지 (PVC · PV · StorageClass)](#6-스토리지-pvc--pv--storageclass)
- [7. 노드와 자원](#7-노드와-자원)
- [8. Route · Service — 접속 주소 찾기](#8-route--service--접속-주소-찾기)
- [9. Secret · ConfigMap](#9-secret--configmap)
- [10. 오퍼레이터 (OLM)](#10-오퍼레이터-olm)
- [11. 폐쇄망 — 미러 레지스트리와 IDMS](#11-폐쇄망--미러-레지스트리와-idms)
- [12. MachineConfig (MCO)](#12-machineconfig-mco)
- [13. Tekton 파이프라인 (MAS 설치)](#13-tekton-파이프라인-mas-설치)
- [14. MAS 전용 리소스](#14-mas-전용-리소스)
- [15. Db2](#15-db2)
- [16. 이미지 빌드 (Manage)](#16-이미지-빌드-manage)
- [17. Manage 서버 로그](#17-manage-서버-로그)
- [18. 자주 쓰는 조합](#18-자주-쓰는-조합)

</details>

---

## 0. 접속 준비

Bastion에서 `oc`를 쓰려면 kubeconfig를 지정합니다. 로그인할 필요가 없어 편합니다.

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig
```

매번 입력하기 번거로우면 `~/.bashrc`에 넣어둡니다.

```bash
echo 'export KUBECONFIG=~/ocp-sno/auth/kubeconfig' >> ~/.bashrc
```

토큰으로 로그인하는 방법도 있습니다.

```bash
oc login -u kubeadmin -p <비밀번호> https://api.mas-it.itmsg.co.kr:6443
```

검증 : 현재 누구로 어디에 붙어 있는지 확인

```bash
oc whoami
oc whoami --show-server
oc auth can-i '*' '*' --all-namespaces      # yes 면 cluster-admin
```

---

## 1. 조회의 기본

`oc get <리소스> -n <네임스페이스>` 가 기본형입니다.

| 옵션 | 뜻 |
|---|---|
| `-n <ns>` | 특정 네임스페이스 |
| `-A` | 전체 네임스페이스 |
| `-o wide` | 컬럼을 더 보여줌 (Pod은 노드·IP 포함) |
| `-o yaml` | 전체 정의를 YAML로 |
| `-o json` | 전체 정의를 JSON으로 (`jq`와 조합) |
| `-w` | 변화를 계속 지켜봄 (watch) |
| `--sort-by=<필드>` | 정렬 |
| `-l <키>=<값>` | 라벨로 필터 |

```bash
oc get pods -n mas-inst1-core              # 특정 네임스페이스의 Pod
oc get pods -A                             # 전체 Pod
oc get pods -n mas-inst1-core -o wide      # 노드·IP까지
oc get pods -n mas-inst1-core -w           # 변화를 지켜봄
```

네임스페이스 목록과 현재 컨텍스트입니다.

```bash
oc get ns
oc project                                 # 현재 네임스페이스
oc project mas-inst1-core                  # 기본 네임스페이스 변경
```

`describe`는 상태와 이벤트를 사람이 읽기 좋게 보여줍니다. **문제가 생기면 가장 먼저 보는 명령**입니다.

```bash
oc describe pod <pod-name> -n <ns>
oc describe pvc <pvc-name> -n <ns>
```

리소스 종류 이름이 기억나지 않을 때 씁니다.

```bash
oc api-resources | grep -i mas             # MAS 관련 CRD 찾기
oc explain manageworkspace.spec            # 필드 설명
```

---

## 2. Pod

```bash
oc get pods -n mas-inst1-manage
```

**정상이 아닌 것만 보기** — 운영 중 가장 자주 쓰는 형태입니다.

```bash
oc get pods -n mas-inst1-core | grep -vE 'Running|Completed'
oc get pods -A | grep -vE 'Running|Completed'
```

| STATUS | 뜻 |
|---|---|
| `Running` | 정상 실행 중 |
| `Completed` | 작업이 끝난 Job/Init Pod — 정상 |
| `Pending` | 스케줄링 대기 — 자원 부족, PVC 미바인딩, affinity 충돌 |
| `ContainerCreating` | 이미지 Pull·볼륨 마운트 중 |
| `Init:0/1` | Init 컨테이너 실행 중 |
| `ImagePullBackOff` / `ErrImagePull` | **이미지를 못 받음** — 폐쇄망에서 미러·IDMS 문제 |
| `CrashLoopBackOff` | 기동 직후 죽고 재시작 반복 — 로그를 봐야 함 |

Pod 안에서 명령을 실행하거나 셸로 들어갑니다.

```bash
oc exec -n db2u c-mas-inst1-ws1-manage-db2u-0 -- df -h
oc rsh -n db2u c-mas-inst1-ws1-manage-db2u-0
```

Pod을 지우면 컨트롤러가 새로 만듭니다 — 사실상 재시작입니다.

```bash
oc delete pod <pod-name> -n <ns>
```

---

## 3. 로그

```bash
oc logs <pod-name> -n <ns>
oc logs <pod-name> -n <ns> --tail=50          # 마지막 50줄
oc logs <pod-name> -n <ns> -f                 # 실시간 추적
oc logs <pod-name> -n <ns> --previous         # 재시작 전 로그 (CrashLoop 원인 추적)
```

컨테이너가 여러 개면 `-c`로 지정합니다. 지정하지 않으면 `Defaulted container ...` 안내가 뜨며 첫 번째가 선택됩니다.

```bash
oc logs <pod-name> -n <ns> -c manager
```

Deployment 이름으로도 볼 수 있습니다 — Pod 이름을 몰라도 됩니다.

```bash
oc logs -n mas-inst1-core deploy/ibm-mas-operator --tail=30
oc logs -n openshift-machine-config-operator deploy/machine-config-controller --tail=20
```

라벨로 여러 Pod을 한 번에 봅니다(`-f`는 단일 Pod만 가능).

```bash
oc logs -n mas-inst1-manage -l app=manage --tail=20
```

---

## 4. Deployment · StatefulSet

```bash
oc get deploy -n mas-inst1-core
oc get statefulset -n db2u
oc get replicaset -n mas-inst1-core
```

재시작·스케일·상태 확인입니다.

```bash
oc rollout restart deploy/<이름> -n <ns>      # 재시작
oc rollout status deploy/<이름> -n <ns>       # 진행 상황
oc scale deploy/<이름> --replicas=0 -n <ns>   # 중지
oc scale deploy/<이름> --replicas=1 -n <ns>   # 재개
```

```bash
oc rollout restart deploy/nfs-client-provisioner -n nfs-provisioner
```

---

## 5. 이벤트 — 문제의 첫 단서

Pod이 안 뜰 때, PVC가 안 붙을 때, 원인이 여기 적혀 있습니다.

```bash
oc get events -n <ns> --sort-by=.lastTimestamp | tail -20
oc get events -A --sort-by=.lastTimestamp | tail -20
```

```bash
oc get events -n nfs-provisioner --sort-by=.lastTimestamp | tail -5
```

경고만 추립니다.

```bash
oc get events -A --field-selector type=Warning --sort-by=.lastTimestamp | tail -20
```

---

## 6. 스토리지 (PVC · PV · StorageClass)

```bash
oc get sc                                  # StorageClass 목록
oc get pvc -n <ns>                         # 네임스페이스의 PVC
oc get pvc -A                              # 전체 PVC
oc get pv                                  # PersistentVolume
```

이번 배포의 StorageClass입니다.

| 이름 | 모드 | 실체 |
|---|---|---|
| `lvms-vg1` (default) | RWO | SNO 로컬 디스크 `/dev/sdb` 500GB |
| `nfs-client` | RWX | Bastion `/export/mas-rwx` |

PVC가 `Pending`이면 이유를 봅니다.

```bash
oc describe pvc <pvc-name> -n <ns> | tail -10
```

⚠️ `lvms-vg1`은 `WaitForFirstConsumer`라 **Pod이 붙기 전에는 `Pending`이 정상**입니다.

LVMS 실제 사용량 확인입니다.

```bash
oc get lvmcluster -n openshift-storage
oc describe lvmcluster my-lvmcluster -n openshift-storage | grep -A5 -i device
```

NFS 쪽은 Bastion에서 직접 봅니다.

```bash
ls -R /export/mas-rwx/
showmount -e localhost
df -h /export/mas-rwx
```

---

## 7. 노드와 자원

```bash
oc get nodes
oc get nodes -o wide
oc describe node mas-it-sno | head -40
```

자원 사용량입니다(metrics 필요).

```bash
oc adm top nodes
oc adm top pods -A --sort-by=memory | head -20
oc adm top pods -n mas-inst1-manage
```

노드에 할당된 요청량 합계 — 스케줄링이 안 될 때 확인합니다.

```bash
oc describe node mas-it-sno | grep -A8 'Allocated resources'
```

노드에 SSH로 접속합니다.

```bash
ssh -i ~/.ssh/quay_installer core@192.168.2.211
```

**노드 종료·재시작**은 [OFFLINE_INSTALL.md §3.11.6](OFFLINE_INSTALL.md)를 따르세요. 🔴 **Db2를 먼저 정지하지 않으면 데이터베이스가 손상됩니다.**

`oc debug`는 `registry.redhat.io/rhel9/support-tools`를 받아오므로 IDMS가 적용된 뒤에만 동작합니다.

```bash
oc debug node/mas-it-sno
```

---

## 8. Route · Service — 접속 주소 찾기

```bash
oc get route -n <ns>
oc get route -A
oc get svc -n <ns>
```

**`hosts` 파일에 넣을 줄을 통째로 생성**합니다.

```bash
oc get route -A -o jsonpath='{range .items[*]}192.168.2.211    {.spec.host}{"\n"}{end}' | awk 'NF==2' | sort -u
```

MAS 주소만 봅니다.

```bash
oc get route -n mas-inst1-core
oc get route -n mas-inst1-manage
```

---

## 9. Secret · ConfigMap

```bash
oc get secret -n <ns>
oc get cm -n <ns>
```

Secret 값은 base64라 디코딩이 필요합니다.

```bash
oc get secret <이름> -n <ns> -o jsonpath='{.data.<키>}' | base64 -d; echo
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

키 이름을 모를 때는 이렇게 훑습니다.

```bash
oc get secret <이름> -n <ns> -o jsonpath='{.data}' | jq 'keys'
```

---

## 10. 오퍼레이터 (OLM)

```bash
oc get sub -A                              # Subscription
oc get csv -A                              # 설치된 오퍼레이터 버전
oc get installplan -A                      # 설치 계획 (수동 승인 대기 확인)
oc get catalogsource -n openshift-marketplace
```

| STATE / PHASE | 뜻 |
|---|---|
| `AtLatestKnown` | Subscription이 최신 버전에 도달 |
| `Succeeded` | CSV 설치 완료 |
| `Pending` / `Installing` | 진행 중 |
| CSV 없음 | 설치가 시작조차 안 됨 — 채널·소스 이름 확인 |

카탈로그에 어떤 오퍼레이터·채널이 있는지 확인합니다. **폐쇄망에서 채널 이름을 확정할 때 씁니다.**

```bash
oc get packagemanifest -n openshift-marketplace | grep -i lvms

oc get packagemanifest lvms-operator -n openshift-marketplace \
  -o jsonpath='{.status.defaultChannel}{"  전체: "}{range .status.channels[*]}{.name}{" "}{end}{"\n"}'
```

---

## 11. 폐쇄망 — 미러 레지스트리와 IDMS

```bash
oc get idms                                # ImageDigestMirrorSet
oc get itms                                # ImageTagMirrorSet
oc get idms <이름> -o yaml
```

이번 배포의 IDMS입니다.

| 이름 | 출처 |
|---|---|
| `image-digest-mirror` | §3.7 `install-config.yaml` |
| `idms-release-0` / `idms-operator-0` | §3.8.1 `oc mirror` 산출물 |
| `mas-ibm-catalog` | §3.8.2 `mas configure-airgap` |

🔴 **노드에 실제로 반영됐는지는 registries.conf로 확인해야 합니다.** IDMS 오브젝트가 있어도 MCO가 렌더링에 실패하면 반영되지 않습니다([TROUBLE_SHOOTING.md 7](TROUBLE_SHOOTING.md)).

```bash
ssh -i ~/.ssh/quay_installer core@192.168.2.211 'grep -c "^\[\[registry\]\]" /etc/containers/registries.conf'
ssh -i ~/.ssh/quay_installer core@192.168.2.211 'grep -A3 "quay.io/ibmmas" /etc/containers/registries.conf'
```

미러 레지스트리 조회는 `podman`으로 합니다.

```bash
podman login registry.mas-it.itmsg.co.kr:8443
podman pull registry.mas-it.itmsg.co.kr:8443/ibmmas/cli:latest

# 이미지 다이제스트 확인
podman image inspect registry.mas-it.itmsg.co.kr:8443/ibmmas/cli:latest --format '{{index .RepoDigests 0}}'

# 저장소 목록
curl -sk -u init:<비밀번호> https://registry.mas-it.itmsg.co.kr:8443/v2/_catalog | jq -r '.repositories[]' | head -20
```

---

## 12. MachineConfig (MCO)

노드의 OS 설정(레지스트리, 커널 파라미터 등)을 관리합니다.

```bash
oc get mcp                                              # MachineConfigPool
oc get mc --sort-by=.metadata.creationTimestamp | tail -3
oc get co machine-config
```

| 컬럼 | 판정 |
|---|---|
| `UPDATED=True`, `UPDATING=False`, `DEGRADED=False` | 정상 |
| `UPDATING=True` | 롤아웃 중 — SNO는 재부팅됨 |
| `DEGRADED=True` | 실패 — 컨트롤러 로그 확인 |

🔴 **`UPDATED=True`만으로는 "반영 완료"를 판정할 수 없습니다.** MCO가 새 설정을 **만들지 못하면** 갱신할 것이 없어 그대로 `True`로 보입니다. 새 `rendered-master-*`가 생겼는지, 컨트롤러에 오류가 없는지 함께 봐야 합니다.

```bash
oc logs -n openshift-machine-config-operator deploy/machine-config-controller --tail=20 | grep -i error
```

---

## 13. Tekton 파이프라인 (MAS 설치)

```bash
oc get pipelinerun -n mas-inst1-pipelines
oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp | tail -6
```

진행을 계속 지켜봅니다.

```bash
watch -n 30 'date; oc get pipelinerun -n mas-inst1-pipelines; echo; oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp | tail -6'
```

실행 중인 Task의 로그를 봅니다. **컨테이너 이름이 Task마다 다르므로 `--all-containers`를 씁니다.**

```bash
TR=$(oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{range .items[?(@.status.conditions[0].status=="Unknown")]}{.metadata.name}{"\n"}{end}' | tail -1)
echo "$TR"

oc logs -n mas-inst1-pipelines "$(oc get pod -n mas-inst1-pipelines -l tekton.dev/taskRun=$TR -o name | head -1)" \
  --all-containers --tail=30
```

실패한 PipelineRun을 지우고 다시 실행할 때 씁니다.

```bash
oc delete pipelinerun <이름> -n mas-inst1-pipelines
oc delete pipelinerun --all -n mas-inst1-pipelines
```

웹 콘솔에서 보는 편이 읽기 쉽습니다.

```
https://console-openshift-console.apps.mas-it.itmsg.co.kr/k8s/ns/mas-inst1-pipelines/tekton.dev~v1beta1~PipelineRun/<이름>
```

---

## 14. MAS 전용 리소스

```bash
oc get suite -A                            # MAS Core
oc get manageapp -A                        # Manage 애플리케이션
oc get manageworkspace -A                  # Manage 워크스페이스
```

Suite 상태의 개별 조건을 봅니다 — 어디서 막혔는지 알 수 있습니다.

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
| `Ready` | 위 조건이 모두 충족됨 |

Manage 워크스페이스 설정 확인입니다.

```bash
# 설치된 컴포넌트 (icd = Maximo IT)
oc get manageworkspace mas-inst1-manage-ws1 -n mas-inst1-manage -o jsonpath='{.spec.components}{"\n"}'

# 언어·타임존·DB 설정
oc get manageworkspace mas-inst1-manage-ws1 -n mas-inst1-manage -o jsonpath='{.spec.settings}' | jq

# 상태 조건
oc get manageworkspace mas-inst1-manage-ws1 -n mas-inst1-manage \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status} — {.message}{"\n"}{end}'
```

MAS 네임스페이스 구성입니다.

| 네임스페이스 | 내용 |
|---|---|
| `mas-inst1-core` | MAS Core |
| `mas-inst1-manage` | Manage + Maximo IT |
| `mas-inst1-pipelines` | 설치 파이프라인 |
| `db2u` | Db2 |
| `mongoce` | MongoDB |
| `ibm-sls` | 라이선스 서비스 |
| `redhat-marketplace` | DRO |

---

## 15. Db2

```bash
oc get db2ucluster -n db2u
oc get pods -n db2u
```

| STATE | 뜻 |
|---|---|
| `Ready` | 정상 |
| `NotReady` | 기동 중 또는 문제 |

```bash
oc logs -n db2u c-mas-inst1-ws1-manage-db2u-0 --tail=30
oc get pvc -n db2u
```

Db2 셸에 들어가 직접 확인합니다.

```bash
oc rsh -n db2u c-mas-inst1-ws1-manage-db2u-0 bash -c 'su - db2inst1 -c "db2 connect to BLUDB; db2 list tablespaces"'
```

---

## 16. 이미지 빌드 (Manage)

Manage는 배포 시 이미지를 **직접 빌드해 내부 Image Registry에 push**합니다.

```bash
oc get build -n mas-inst1-manage
oc logs -f build/all-build-config-1 -n mas-inst1-manage --tail=20
```

| 빌드 | 내용 |
|---|---|
| `admin-build-config-*` | Admin 번들 |
| `all-build-config-*` | 서버 번들 — Maximo IT 포함, 더 큼 |

내부 Image Registry 상태입니다. **`Removed`면 빌드가 실패합니다.**

```bash
oc get co image-registry
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.managementState}{"\n"}{.spec.storage}{"\n"}'
oc get pvc -n openshift-image-registry
```

---

## 17. Manage 서버 로그

**Maximo 로그 실시간 확인** — 이 한 줄이면 됩니다.

```bash
oc rsh -n mas-inst1-manage deploy/inst1-ws1-all tail -f /logs/messages.log
```

자주 쓰면 alias를 걸어두세요.

```bash
echo "alias maslog='oc rsh -n mas-inst1-manage deploy/inst1-ws1-all tail -f /logs/messages.log'" >> ~/.bashrc
source ~/.bashrc
```

로그 파일 목록과 마지막 부분만 볼 때입니다.

```bash
oc rsh -n mas-inst1-manage deploy/inst1-ws1-all ls -la /logs/
oc rsh -n mas-inst1-manage deploy/inst1-ws1-all tail -100 /logs/messages.log
```

---

아래는 Pod을 직접 다뤄야 할 때의 상세입니다.

Manage 관련 Pod은 역할이 나뉩니다.

| Pod 이름 | 역할 |
|---|---|
| `inst1-ws1-manage-maxinst-*` | **DB 스키마 생성** — 최초 배포 시 한 번. 완료 후 사라짐 |
| `*-all-*` | **Maximo 서버 번들** — 실제 애플리케이션 |
| `ibm-mas-manage-operator-*` | Manage 오퍼레이터 |
| `inst1-entitymgr-*` | Entity Manager (워크스페이스·상태 관리) |

🔴 **Pod 이름은 재생성될 때마다 바뀝니다**(`...-all-6cf6676cc-l6rfx` 의 뒷부분). 이름을 직접 적지 말고 아래 둘 중 하나를 쓰세요.

**방법 1 — 컨트롤러 이름으로 부르기** (권장). Pod이 바뀌어도 그대로 동작합니다.

```bash
oc get deploy,statefulset -n mas-inst1-manage | grep -i all      # 이름 확인
oc logs -n mas-inst1-manage deploy/<이름> -f --tail=50
```

**방법 2 — 변수로 잡기**

```bash
POD=$(oc get pods -n mas-inst1-manage -o name | grep -- '-all-' | head -1)
echo "$POD"

oc logs -n mas-inst1-manage "$POD" -f --tail=50
oc logs -n mas-inst1-manage "$POD" --previous            # 재시작 전 로그
```

🔴 **컨테이너 로그(`oc logs`)에는 기동 대기 루프만 찍힙니다.**

```
Oslc servlet.... Server did not started yet. Will wait 30 sec.
```

**실제 Liberty·Maximo 로그는 Pod 안 `/logs/`** 에 있습니다. 컨테이너 이름은 `all`입니다(`monitoragent`도 함께 들어 있음).

```bash
oc exec -n mas-inst1-manage "${POD#pod/}" -c all -- tail -30 /logs/messages.log
oc exec -n mas-inst1-manage "${POD#pod/}" -c all -- ls /logs/
```

셸로 들어가 따라가려면 이렇게 합니다.

```bash
oc rsh -n mas-inst1-manage -c all "$POD"
tail -f /logs/messages.log
```

파일을 로컬로 꺼냅니다.

```bash
oc cp "mas-inst1-manage/${POD#pod/}:/logs/messages.log" ./messages.log -c all
```

기동 완료 표식입니다.

```
CWWKF0011I: The defaultServer server is ready to run a smarter planet.
```

⚠️ replica가 2개 이상이면 Pod마다 로그가 다릅니다. 전부 보려면 라벨로 조회하세요(`-f`는 단일 Pod만 가능).

```bash
oc logs -n mas-inst1-manage -l mas.ibm.com/serverBundle=all --tail=50 --prefix
```

DB 스키마 생성 진행 상황(최초 배포 시)입니다.

```bash
oc logs -n mas-inst1-manage deploy/inst1-ws1-manage-maxinst -f --tail=20
```

로그 시각은 `serverTimezone` 설정을 따릅니다. 이번 배포는 `Asia/Seoul`이라 한국 시각으로 찍힙니다.

---

## 18. 자주 쓰는 조합

**클러스터 전반 건강 확인**

```bash
oc get co | grep -vE 'True *False *False'      # 비정상 오퍼레이터만
oc get nodes
oc get pods -A | grep -vE 'Running|Completed'
```

**MAS 전체 상태 한눈에**

```bash
oc get suite,manageapp,manageworkspace -A
oc get pods -n mas-inst1-core | grep -vE 'Running|Completed'
oc get pods -n mas-inst1-manage | grep -vE 'Running|Completed'
```

**설치 진행 감시**

```bash
watch -n 30 'date; oc get pipelinerun -n mas-inst1-pipelines; echo; oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp | tail -5; echo; oc get suite,manageworkspace -A'
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

**리소스 정의를 파일로 백업**

```bash
oc get suite inst1 -n mas-inst1-core -o yaml > suite-backup.yaml
oc get manageworkspace mas-inst1-manage-ws1 -n mas-inst1-manage -o yaml > mw-backup.yaml
```
