# 트러블슈팅

[OFFLINE_INSTALL.md](OFFLINE_INSTALL.md) 진행 중 실제로 막혔던 지점과 해결 방법입니다. 각 항목은 증상 → 원인 → 해결 순서입니다.

<details>
<summary><b>목차</b></summary>

- [1. `:Z` vs `:z` SELinux 라벨 충돌](#1-z-vs-z-selinux-라벨-충돌)
- [2. MAS 콘텐츠 미러링 중 `cp.icr.io` OAuth 토큰 타임아웃](#2-mas-콘텐츠-미러링-중-cpicrio-oauth-토큰-타임아웃)
- [3. Red Hat 콘텐츠 미러링 — 타임아웃과 캐시 소실](#3-red-hat-콘텐츠-미러링--타임아웃과-캐시-소실)
- [4. SSH 접속을 끊으면 미러링 컨테이너가 죽음](#4-ssh-접속을-끊으면-미러링-컨테이너가-죽음)
- [5. `mas mirror-redhat-images --mode from-filesystem` 을 쓰지 않는 이유](#5-mas-mirror-redhat-images---mode-from-filesystem-을-쓰지-않는-이유)
- [6. `install-config.yaml` 검증 실패 — `imageDigestSources`](#6-install-configyaml-검증-실패--imagedigestsources)
- [7. MCO가 `registries.conf`를 렌더링하지 못함 — `conflicting mirrorSourcePolicy`](#7-mco가-registriesconf를-렌더링하지-못함--conflicting-mirrorsourcepolicy)
- [8. Manage 이미지 빌드 Pod이 스케줄되지 않음 — `Insufficient cpu`](#8-manage-이미지-빌드-pod이-스케줄되지-않음--insufficient-cpu)
- [9. Db2 재생성 후 설정 확인이 무한 재시도 — `CHNGPGS_THRESH`](#9-db2-재생성-후-설정-확인이-무한-재시도--chngpgs_thresh)
- [10. `oc rollout restart`가 되돌려짐 — OLM 관리 오퍼레이터](#10-oc-rollout-restart가-되돌려짐--olm-관리-오퍼레이터)

</details>


---

## 1. `:Z` vs `:z` SELinux 라벨 충돌

**증상**: 미러링을 여러 터미널에서 동시에 실행하면 로그 파일 쓰기 단계에서 `Permission denied`로 실패.

**원인**: 두 컨테이너가 같은 디렉터리를 각자 `-v ...:Z`(대문자, 배타적 라벨)로 마운트해서 MCS 카테고리가 충돌. 나중에 시작한 컨테이너가 재라벨링하면 먼저 실행 중인 컨테이너의 접근 권한이 무효화됩니다.

```bash
sudo grep avc /var/log/audit/audit.log | tail -30
```
```
avc: denied { write } for ... scontext=...:s0:c280,c427  tcontext=...:s0:c285,c450
```

**해결**: `:Z` 대신 **`:z`**(소문자, 공유 라벨)를 사용합니다. 이 문서의 모든 `podman run`은 `:z`로 되어 있습니다.

## 2. MAS 콘텐츠 미러링 중 `cp.icr.io` OAuth 토큰 타임아웃

**증상**: `mas mirror-images` 실행 중 `Fail if mirror is not successful` 태스크에서 실패. 로그에 아래 에러.

```
error: unable to push cp.icr.io/cp/manage/aip-optimizer: failed to retrieve blob ...:
Get "https://cp.icr.io/oauth/token?...": net/http: request canceled (Client.Timeout exceeded)
```

**원인**: IBM 레지스트리의 OAuth 토큰 발급이 일시적으로 타임아웃. `oc image mirror`는 이미지 하나라도 실패하면 전체를 실패로 처리합니다.

**해결**: 받은 콘텐츠는 `mirror/<stage>/v2`에 남아 있고 재실행 시 건너뜁니다. **디렉터리를 비우지 말고 같은 명령을 다시 실행**하세요.

## 3. Red Hat 콘텐츠 미러링 — 타임아웃과 캐시 소실

**증상**: 약 1.5시간 지점부터 `rhoai`(OpenShift AI)의 대용량 이미지에서 반복 실패하고 `rc=4`로 종료. 이미지 하나가 실패하면 해당 Operator 번들 전체가 스킵됩니다.

```
error: copying image 1/1 from manifest list: writing blob:
Patch "http://localhost:55000/v2/rhoai/odh-.../blobs/uploads/...": context deadline exceeded
```

**원인 1 — 타임아웃**: `localhost:55000`은 oc-mirror v2의 로컬 캐시 레지스트리입니다. 다운로드가 아니라 **캐시에 쓰는 작업**이 이미지당 기본 타임아웃(10분)을 넘긴 것입니다. 실효 속도 5~9MB/s에서 10분에 처리 가능한 크기는 약 3~5GB인데 rhoai 이미지는 그보다 큽니다.

**원인 2 — 캐시 소실**: `--cache-dir` 기본값이 **`$HOME`**(컨테이너 내부)이라, `--rm` 컨테이너가 종료될 때 캐시가 함께 삭제됩니다. 실제로 5시간 미러링이 실패했을 때 산출물은 8.4GB뿐이었고 컨테이너에 쌓인 116GB가 전부 사라졌습니다. Red Hat 공식 절차는 `oc mirror`를 호스트에서 직접 실행하므로 이 문제가 없습니다.

**해결**: §2.3의 8번을 1) imageset 생성 / 2) 미러링으로 분리하고, 2)에서 `oc mirror`를 직접 호출하며 두 옵션을 추가했습니다.

| 옵션 | 목적 |
|---|---|
| `--cache-dir /mnt/cache` | 캐시를 호스트 볼륨에 둬 **실패해도 이어받기** |
| `--image-timeout 60m` | 이미지당 기본 10분 → 60분 |

시작 10분 뒤 호스트 캐시가 커지고 컨테이너 SIZE는 작게 유지되어야 정상입니다.

```bash
du -sh ~/mas-install/oc-mirror-cache
podman ps --size --format "{{.Status}}  SIZE={{.Size}}"
```

반복 실패하면 `--parallel-images 4`로 동시성을 낮춥니다. `imageset-ocp4.20.yml`에서 `rhods-operator`를 제거하는 방법은 공식 기본 범위를 벗어나므로 최후 수단입니다.

**부작용 — 불완전한 `working-dir`**

1)에서 래퍼를 중단하면 반쯤 만들어진 `working-dir`이 남고, 2)가 이를 기존 작업으로 인식해 실패한다.

```
failed to find release image in index: ... release-images/ocp-release/4.20.30-x86_64/index.json: no such file or directory
```

1) 마지막에 `rm -rf ~/mas-install/mirror/redhat/working-dir ~/mas-install/mirror/redhat/logs`를 실행해 `config.json`과 `imageset-ocp4.20.yml` 두 개만 남겨야 한다. 단 **2)가 실패해 재시도하는 경우에는 `working-dir`을 지우지 않는다** — 그때는 oc-mirror가 스스로 만든 정상 상태다.

## 4. SSH 접속을 끊으면 미러링 컨테이너가 죽음

**증상**: 백그라운드로 띄운 미러링이 접속을 끊은 시점에 중단. 로그가 이미지 복사 도중 끊기고 마지막 줄이 아래와 같다.

```
container not running: No such process
```

`podman ps`에 컨테이너가 없고, OOM 기록(`dmesg | grep -i "killed process"`)도 없다.

**원인**: rootless podman 컨테이너는 사용자 세션에 속한다. `nohup`/`disown`은 SIGHUP만 막고, `systemd-logind`가 **마지막 세션 종료 시 user slice를 정리**하는 것은 막지 못한다.

```bash
loginctl show-user maximo | grep -i linger     # Linger=no 면 이 원인
```

**해결**: `linger`를 켜면 세션과 무관하게 사용자 프로세스가 유지된다. 켠 뒤 같은 명령을 재실행하면 캐시에서 이어받는다.

```bash
sudo loginctl enable-linger maximo
loginctl show-user maximo | grep -i linger     # Linger=yes
```

## 5. `mas mirror-redhat-images --mode from-filesystem` 을 쓰지 않는 이유

IBM CLI 23.4.1의 `mirror_ocp` 역할이 아래 명령을 실행합니다.

```
oc mirror --remove-signatures -c <dir>/imageset-ocp<ver>.yml \
  --dest-tls-verify=false --parallel-images=1 \
  --from file://<dir>/mirror_seq1_000000.tar docker://<registry> --v2
```

문제가 두 가지 겹쳐 있습니다.

| | 내용 |
|---|---|
| 파일명 | `mirror_seq1_000000.tar`가 하드코딩돼 있는데, 같은 이미지에 내장된 oc-mirror(4.22.0)는 **`mirror_000001.tar`** 로 만듭니다 |
| 문법 | `--from`에 **tar 파일 경로**를 주는 건 oc-mirror **v1** 방식입니다. `--v2`에서는 `--from file://<디렉터리>`를 받아 그 안의 `mirror_*.tar`를 스스로 찾습니다 |

`to-filesystem.yml`은 파일명을 전혀 다루지 않고 `oc mirror`를 그대로 호출합니다. 즉 산출물 이름은 oc-mirror가 정하는데 반대편 역할만 v1 시절 이름과 문법에 묶여 있는 것으로, **IBM CLI 자체의 버전 불일치**입니다.

실제로 실행하면 이렇게 끝납니다.

```
[INFO]  : 🔀 workflow mode: diskToMirror
[ERROR] : [Executor] no tar archives matching "mirror_[0-9]{6}\.tar"
          found in "/mnt/workspace/redhat/mirror_seq1_000000.tar"
```

`--from` 값을 **디렉터리로 열어서** 그 안의 `mirror_[0-9]{6}.tar`를 찾습니다. 산출물 `mirror_000001.tar`는 이 패턴에 이미 부합하므로, **리네임하면 오히려 패턴에서 벗어납니다.**

**해결**: 파일명을 바꾸지 말고 §3.6.4처럼 `oc mirror`를 직접 호출합니다. §2.3 8번에서 이미 같은 이유로 래퍼를 우회했으므로 양방향이 대칭이 됩니다.

```
--from file:///mnt/workspace/redhat        # 디렉터리 — tar 파일명 무관
```

파일 목록 검증(`transfer-files.sha256`)과도 어긋나지 않는다는 이점이 있습니다.

**참고** — 역할이 실제로 실행하는 명령입니다. 래퍼를 우회할 때 그대로 쓰면 됩니다.

```
DOCKER_CONFIG=<dir> oc mirror --remove-signatures -c <dir>/imageset-ocp4.20.yml \
  --dest-tls-verify=false --parallel-images=1 \
  --from file://<dir>/mirror_seq1_000000.tar docker://<registry> --v2
```

## 6. `install-config.yaml` 검증 실패 — `imageDigestSources`

**증상**: §3.7.6 ISO 생성이 실패합니다.

```
FATAL failed to load asset "Install Config": invalid install-config configuration:
      imageDigestSources[23].source: Invalid value: "docker.io/grafana":
      the repository provided is invalid: a lowercase RFC 1123 subdomain must consist of ...
```

**원인**: `idms-oc-mirror.yaml`의 미러 34건을 `install-config.yaml`에 그대로 옮기면, `openshift-install`의 저장소 이름 검증이 `docker.io/grafana` 같은 항목을 거부합니다. **IDMS CRD는 받아들이는 형식인데 설치 프로그램 쪽 검증이 더 엄격합니다.**

**해결**: release 미러 2건만 넣습니다. 둘은 쓰이는 시점이 다릅니다.

| | 언제 | 어디에 |
|---|---|---|
| release 미러 2건 | **부트스트랩** — 클러스터가 아직 없을 때 릴리스 이미지를 받는 경로 | `install-config.yaml` |
| 오퍼레이터 미러 32건 | 클러스터 기동 후 오퍼레이터·워크로드 이미지 | §3.8에서 `oc apply -f idms-oc-mirror.yaml` |

버리는 것이 아니라 **적용 시점을 뒤로 미루는 것**이며, Red Hat 공식 예시도 `install-config.yaml`에는 release만 넣습니다.

```bash
# idms-release-0 문서만 추출
awk '/name: idms-release-0/{d=1} d&&/^  imageDigestMirrors:/{f=1;next} /^---/{f=0;d=0} f' \
  ~/mas-install/mirror/redhat/working-dir/cluster-resources/idms-oc-mirror.yaml
```

## 7. MCO가 `registries.conf`를 렌더링하지 못함 — `conflicting mirrorSourcePolicy`

**증상**: §3.8.2 이후 IDMS 오브젝트는 존재하고 `oc get mcp`도 `UPDATED=True`인데, **노드의 `registries.conf`에 반영되지 않습니다.** 새 `rendered-master-*`가 생성되지 않는 것이 단서입니다.

MAS 설치(§3.10)에서 `TaskRunImagePullFailed`로 드러납니다 — Tekton이 `quay.io/ibmmas/cli@sha256:...`를 당기는데 미러 매핑이 없기 때문입니다.

```bash
oc logs -n openshift-machine-config-operator deploy/machine-config-controller --tail=20
```

```
Error syncing image config openshift-config: could not Create/Update MachineConfig:
could not update registries config with new changes:
conflicting mirrorSourcePolicy is set for the same source "icr.io/cpopen"
in imagedigestmirrorsets, imagetagmirrorsets, or imagecontentsourcepolicies
```

**원인**: §3.8.1의 `idms-operator-0`(oc-mirror 생성)과 §3.8.2의 `mas-ibm-catalog`(`mas configure-airgap` 생성)가 **같은 source에 다른 `mirrorSourcePolicy`** 를 지정합니다.

| IDMS | `icr.io/cpopen` 정책 |
|---|---|
| `idms-operator-0` | 기본값 (`AllowContactingSource`) |
| `mas-ibm-catalog` | `NeverContactSource` |

정책이 충돌하면 MCO는 **registries.conf 전체 렌더링을 중단**합니다. 그 결과 `mas-ibm-catalog`의 매핑이 **하나도 반영되지 않습니다** — `quay.io/ibmmas`, `cp.icr.io/cp`, `icr.io/db2u` 등 MAS 이미지 경로 전부입니다.

**해결**: 충돌하는 source의 정책을 맞춥니다.

```bash
IDX=$(oc get idms idms-operator-0 -o json | \
  jq -r '.spec.imageDigestMirrors | to_entries[] | select(.value.source=="icr.io/cpopen") | .key')

oc patch idms idms-operator-0 --type=json \
  -p "[{\"op\":\"add\",\"path\":\"/spec/imageDigestMirrors/$IDX/mirrorSourcePolicy\",\"value\":\"NeverContactSource\"}]"
```

적용되면 새 `rendered-master-*`가 생기고 항목 수가 34 → 43으로 늘어납니다.

```bash
oc get mc --sort-by=.metadata.creationTimestamp | tail -2
ssh -i ~/.ssh/quay_installer core@<sno-ip> 'grep -c "^\[\[registry\]\]" /etc/containers/registries.conf'
```

⚠️ **`quay.io/ibmmas` IDMS를 따로 만들지 마세요.** `mas-ibm-catalog`에 이미 들어 있어 중복 정의가 되고, 같은 충돌을 하나 더 만듭니다.


---

## 8. Manage 이미지 빌드 Pod이 스케줄되지 않음 — `Insufficient cpu`

**증상**: 언어 추가·커스터마이징 등으로 Manage 이미지를 다시 빌드할 때 빌드 Pod이 `Pending`에 머물다 오퍼레이터가 취소합니다. 반복되며 진행되지 않습니다.

```bash
oc describe pod admin-build-config-1-build -n mas-inst1-manage | tail -5
```

```
0/1 nodes are available: 1 Insufficient cpu.
preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
```

`ManageWorkspace` 상태에는 이렇게 나옵니다.

```
BuildReady: False — Request Build Failed : Unexpected error occured while requesting admin build
```

**원인**: SNO 노드의 CPU 요청 총량이 한계에 도달했습니다. 설치 직후 실측이 **14551m / 15600m (93%)** 였습니다.

```bash
oc describe node <노드명> | grep -A4 'Allocated resources'
```

빌드 Pod은 1~2코어를 요구하므로 들어갈 자리가 없습니다.

**해결** — 노드 CPU를 늘립니다. 16 → 20코어면 요청률이 74%로 내려가 여유가 생깁니다.

```bash
# 1) Db2 정지 후 노드 종료 (OFFLINE_INSTALL.md §3.11.6)
# 2) vSphere에서 vCPU 변경
# 3) 기동 후 확인
oc describe node <노드명> | grep -A6 'Capacity:'
oc describe node <노드명> | grep -A4 'Allocated resources'
```

⚠️ **워크로드를 줄이는 방식으로는 해결되지 않습니다.** 쓰지 않는 Pod을 `oc scale --replicas=0`으로 내려도 오퍼레이터가 되살립니다.

주요 CPU 요청처입니다(설치 직후 실측).

| Pod | 요청 |
|---|---|
| `mongoce/mas-mongo-ce-0` | 1000m |
| `mas-inst1-manage/inst1-ws1-slackproxy` | **1000m** — Slack 미사용인데도 기동됨 |
| `mas-inst1-manage/inst1-ws1-all` (Maximo 서버) | 600m |
| `mongodb-kubernetes-operator` / `db2u-operator` / `ibm-sls-controller` / `slackproxy-operator` / `entitymgr-suite` | 각 500m |
| `inst1-ws1-manage-maxinst` | 500m — DB 작업 후에도 상주 |

**예방**: SNO를 20코어 이상으로 시작하세요. 16코어는 설치는 되지만 이후 변경 작업에서 막힙니다.

---

## 9. Db2 재생성 후 설정 확인이 무한 재시도 — `CHNGPGS_THRESH`

**증상**: `suite-db2-setup-manage` Task가 끝나지 않습니다.

```
TASK [ibm.mas_devops.suite_db2_setup_for_manage : Check DB2 cfg is take effect]
FAILED - RETRYING: Check DB2 cfg is take effect (5 retries left).
FAILED - RETRYING: Check DB2 cfg is take effect (4 retries left).
```

이 체크는 `CHNGPGS_THRESH = 40` 을 기다리는데, 실제 값은 기본값 `80`입니다.

```bash
oc rsh -n db2u c-mas-inst1-ws1-manage-db2u-0 bash -c \
  'su - db2inst1 -c "db2 get db cfg for BLUDB | grep -i chngpgs"'
```

**원인**: Db2를 재생성할 때 `db2u` 네임스페이스의 **`mas-inst1-ws1-manage-enforce-config` ConfigMap을 남겨두면** 역할이 "이미 설정 적용됨"으로 판단해 Db2 설정을 기록하는 단계를 통째로 건너뜁니다.

```
TASK [... : Verify if DB2 is already enforced]
  name: mas-inst1-ws1-manage-enforce-config
  creationTimestamp: "<이전 설치 시각>"

TASK [... : include_tasks]     ← db2_dbconfig.yml
  skipping: false_condition "db2_cfg | length == 0 or db2_configured_version != db2_config_version"
```

실제 Db2는 새로 만든 것이라 기본값 상태인데 설정만 적용되지 않아 체크가 영원히 통과하지 못합니다.

**즉시 해결** — 값을 직접 넣으면 다음 재시도에서 통과합니다.

```bash
oc rsh -n db2u c-mas-inst1-ws1-manage-db2u-0 bash -c \
  'su - db2inst1 -c "db2 update db cfg for BLUDB using CHNGPGS_THRESH 40"'
```

체크와 같은 방식으로 확인합니다 — `exit=0` 이어야 합니다.

```bash
oc exec -n db2u c-mas-inst1-ws1-manage-db2u-0 -- \
  su -lc 'db2 get db cfg for BLUDB | grep "(CHNGPGS_THRESH) = 40"' db2inst1
echo "exit=$?"
```

**예방**: Db2를 재생성할 때는 Db2uCluster·PVC만이 아니라 **ConfigMap까지** 지웁니다. 절차는 [OFFLINE_INSTALL.md §3.10.5](OFFLINE_INSTALL.md)에 있습니다.

```bash
oc delete cm mas-inst1-ws1-manage-enforce-config -n db2u
```

---

## 10. `oc rollout restart`가 되돌려짐 — OLM 관리 오퍼레이터

**증상**: 오퍼레이터를 재시작했는데 잠시 새 Pod이 떴다가 사라지고 옛 Pod이 그대로 남습니다.

```bash
oc rollout restart deploy/ibm-mas-manage-operator -n mas-inst1-manage
```

```
deployment.apps/ibm-mas-manage-operator restarted
```

몇 분 뒤 확인하면 이전 Pod이 그대로입니다.

**원인**: 오퍼레이터 Deployment는 **OLM(Operator Lifecycle Manager)이 CSV 기준으로 관리**합니다. `rollout restart`가 추가한 어노테이션을 OLM이 원래 상태로 되돌리면서 새 ReplicaSet이 정리됩니다. `oc scale`로 내리는 것도 같은 이유로 복구됩니다.

**해결** — Pod을 직접 지웁니다. ReplicaSet이 새로 만들어 주므로 OLM과 충돌하지 않습니다.

```bash
oc delete pod <오퍼레이터-pod-이름> -n mas-inst1-manage
```

이름을 몰라도 되게 쓰려면 이렇게 합니다.

```bash
oc delete pod -n mas-inst1-manage \
  $(oc get pods -n mas-inst1-manage -o name | grep manage-operator | head -1)
```

같은 원리가 `db2u`·`mongoce` 등 다른 OLM 오퍼레이터에도 적용됩니다.
