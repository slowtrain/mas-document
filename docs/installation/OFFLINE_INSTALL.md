# Maximo IT 오프라인(Airgap) 설치 가이드

> **기준일**: 2026-07-29
> **신뢰도 안내**: 이 문서는 IBM MAS CLI 공식 문서(`ibm-mas.github.io/cli`, GitHub 원본 `github.com/ibm-mas/cli`)와 Red Hat OpenShift 공식 문서를 근거로 작성했습니다.
>
> 표기 규칙:
> - **✅ 검증 완료** — 실제 CLI 실행 결과 또는 공식 문서 원문으로 확인한 내용
> - **⚠️ 미검증** — 근거가 불충분하거나 문서 간 내용이 엇갈리는 내용. 실행 전 반드시 재확인 필요
>
> ⚠️ IBM 지식센터(`ibm.com/docs`) 일부 페이지는 **이 문서를 작성한 환경의 자동 조회 도구에서 403이 반환되어** 검색 스니펫으로만 확인했습니다(브라우저로는 접근 가능할 수 있음). 해당 항목은 본문에 미검증으로 표시했으니 **브라우저로 원문을 직접 열어 확인**하세요. 특히 Manage/Maximo IT 배포(§3.10)와 라이선스 관련 내용이 여기 해당됩니다.
>
> 실행 전 항상 `mas <command> --help`로 플래그를 재확인하세요 — 문서와 실제 CLI가 다른 사례를 이 배포에서 여러 번 확인했습니다(§5 참고).

---

## 0. 이번 설치의 확정 값

| 항목 | 값 |
|---|---|
| Red Hat OpenShift | `4.20.30` (SNO, Single Node OpenShift) |
| IBM Maximo Operator Catalog | `v9-260625-amd64` (지원 범위 OCP 4.16–4.21) |
| MAS CLI 이미지 | `quay.io/ibmmas/cli:23.4.1` |
| 설치 대상 | MAS Core + Manage + **Maximo IT**(Manage의 add-on) |
| 미사용 대상 | Cloud Pak for Data, Visual Inspection, Assist, IoT, Monitor, Optimizer 등 |
| 데이터베이스 | **내부(in-cluster) Db2** |
| 스토리지 | RWO = **LVM Storage**(`lvms-vg1`) / RWX = **외부 NFS — Bastion 겸용** (§3.8) |
| Bastion | `192.168.2.210`, RHEL 9.6 (DNS, Mirror Registry, 설치 CLI 실행) |
| SNO 노드 | `192.168.2.211` / 255.255.252.0 / GW `192.168.1.1` / MAC `00:50:56:bb:86:df` |
| 작업 계정 | `maximo` (`~/mas-install`) |

접속 정보(계정/비밀번호)는 이 문서에 기록하지 않습니다. 별도 보안 저장소를 확인하세요.

---

## 1. 설치 요약 및 흐름

### 1.1 전체 아키텍처

```text
[인터넷 연결 RHEL 서버]  (준비 단계 전용 — 사이트 Bastion과 겸용 가능)
  - 설치 도구 / 라이선스 다운로드
  - MAS CLI로 이미지를 로컬 디스크에 미러링(to-filesystem)
        |
        v  (전송매체 또는 동일 서버 재사용)
[사이트 Bastion = 192.168.2.210]
  - Mirror Registry 운영
  - 로컬에 쌓인 이미지를 Registry로 Push(from-filesystem)
  - DNS, 설치 CLI 실행
        |
        v
[SNO 노드 = 192.168.2.211]
  - OpenShift 4.20.30 설치 (Agent-based Installer)
  - MAS Core + Manage + Maximo IT 배포
```

> **서버 구성 두 가지 경우**
>
> | 구성 | §2(준비) 실행 위치 | §2.5(전달) | §3(설치) 실행 위치 |
> |---|---|---|---|
> | **A. 분리** (실제 사이트 배포) | 인터넷 연결 서버 | **필요** — tar 분할·반입 | 폐쇄망 사이트 Bastion |
> | **B. 겸용** (테스트/검증) | Bastion 자체 (인터넷 임시 허용) | 생략 가능 | 같은 서버에서 이어서 진행 |
>
> 이 문서는 두 경우를 모두 지원합니다. 서버 이름·IP는 §0의 값으로 치환해 사용하세요. 겸용 구성에서는 §2와 §3이 같은 서버에서 순서대로 진행되며, §2.5의 tar 분할/반입 단계만 건너뜁니다.

### 1.2 공식 문서 기준 설치 순서 (6단계)

기존에 알려진 "OCP용 `oc-mirror`" + "MAS용 `mas mirror-images`"라는 **두 개의 별도 도구를 쓰는 방식이 아닙니다.** IBM MAS CLI 공식 가이드는 **Red Hat 콘텐츠(OCP 플랫폼 + 필수 Operator)와 MAS 콘텐츠 모두를 `mas` CLI 하나로 미러링**하도록 설계되어 있습니다.

| 단계 | 작업 | 명령 | 실행 위치 |
|---|---|---|---|
| 1 | Red Hat 콘텐츠 미러링 (OCP 플랫폼 + 필수 Operator: DRO, Cert Manager, Grafana) | `mas mirror-redhat-images --mirror-platform --mirror-operators` | 인터넷 연결 서버 (to-filesystem) → Bastion (from-filesystem) |
| 2 | MAS 콘텐츠 미러링 (Core/Catalog → Manage+IT → 공통 의존성/Db2 → CLI) | `mas mirror-images` (Stage 1/2/4/5, **Stage 3=CP4D는 미사용이라 제외**) | 인터넷 연결 서버 (to-filesystem) → Bastion (from-filesystem) |
| 3 | OCP SNO 클러스터 설치 | `openshift-install agent create image` / 부팅 | Bastion → SNO |
| 4 | Airgap 구성 — **(a)** oc-mirror 생성 리소스(IDMS/CatalogSource) 적용 + OperatorHub 기본 소스 비활성화, **(b)** `mas configure-airgap` | `oc apply` + `mas configure-airgap` | Bastion (클러스터 대상) |
| 5 | **스토리지 준비** (RWO + RWX StorageClass) — `mas install` 전 필수 | LVM Storage(RWO) + NFS 프로비저너(RWX) | Bastion (클러스터 대상) |
| 6 | MAS 설치 및 Manage/Maximo IT 배포·활성화 | `mas install` + Suite Administration | Bastion (클러스터 대상) |

✅ **검증 완료** (2026-07-29, 실제 실행 로그로 확인): `mas mirror-redhat-images --mirror-platform`은 oc-mirror를 대체하는 게 아니라 **내부적으로 `oc mirror` v2를 그대로 호출하는 래퍼**입니다. 실행 로그(`/mnt/workspace/redhat/logs/mirror-to-filesystem-ocp4.20.log` 대상)에 다음 태스크가 그대로 찍힙니다.

```
Command: DOCKER_CONFIG=/mnt/workspace/redhat oc mirror --remove-signatures \
  -c /mnt/workspace/redhat/imageset-ocp4.20.yml file:////mnt/workspace/redhat --v2
```

즉 MAS CLI가 `ImageSetConfiguration`(`imageset-ocp4.20.yml`)을 자동 생성하고 `oc mirror --v2`를 대신 실행해줍니다. 별도로 받아둔 `oc-mirror.tar.gz`가 불필요했던 이유도 이것입니다 — **MAS CLI 이미지(`quay.io/ibmmas/cli:23.4.1`) 안에 `oc`와 `oc-mirror` 플러그인이 이미 내장**되어 있습니다.

```bash
docker run --rm quay.io/ibmmas/cli:23.4.1 mas mirror-redhat-images --help
docker run --rm quay.io/ibmmas/cli:23.4.1 mas mirror-images --help
```

### 1.3 참고한 공식 문서

- [IBM MAS CLI](https://ibm-mas.github.io/cli/)
- [IBM MAS CLI Catalogs](https://ibm-mas.github.io/cli/catalogs/)
- [Image Mirroring (IBM)](https://ibm-mas.github.io/cli/guides/image-mirroring/) / [원본](https://github.com/ibm-mas/cli/blob/master/docs/guides/image-mirroring.md)
- [Image Mirroring (Red Hat) 커맨드 레퍼런스](https://ibm-mas.github.io/cli/commands/mirror-redhat-images/) / [원본](https://github.com/ibm-mas/cli/blob/master/docs/commands/mirror-redhat-images.md)
- [Airgap Configuration](https://ibm-mas.github.io/cli/guides/configure-airgap/) / [원본](https://github.com/ibm-mas/cli/blob/master/docs/guides/configure-airgap.md)
- [MAS CLI Topology Reference](https://ibm-mas.github.io/cli/reference/topology/)
- IBM Maximo IT 설치: `ibm.com/docs/en/masv-and-l/max-it` 계열 (WebFetch 403 — 검색 스니펫으로만 확인, ⚠️ 원문 직접 확인 필요)

---

## 2. 사전준비 사항 (설치 파일 및 이미지 생성)

이 장은 **인터넷 연결 서버**에서 진행합니다(§1.1의 구성 A/B 참고 — 겸용 구성이면 사이트 Bastion에서 진행). 별도 표시가 없으면 `maximo` 계정으로 실행합니다.

### 2.1 저장공간 참고 수치 (공식 문서, MAS 8.10 기준 — 최신 수치 아님, 참고용)

| 구성 | 용량 |
|---|---|
| MAS Core | 4GB |
| Manage | 8GB |
| MongoDB | 500MB |
| TSM | 1GB |
| SLS | 1GB |
| CFS | 21GB |
| Db2(내부) | 73GB |
| Red Hat 필수 Operator 3종 | 약 80GB |

Cloud Pak for Data, Visual Inspection 등 미사용 컴포넌트는 포함하지 않습니다. 실제 값은 미러링 진행 중 `du -sh "$LOCAL_DIR"` 로 재확인하세요.

✅ **실측값 (2026-07-30, 이 배포에서 직접 측정)** — 위 참고치는 MAS 8.10(2023년) 기준이라 **항목별로 크게 다릅니다. 실측을 우선 신뢰하세요.**

| 단계 | 참고치 | **실측** | 비고 |
|---|---|---|---|
| Stage 1 — Core + Catalog | ~4GB | **15GB** | 참고치의 약 3.8배 |
| Stage 2 — Manage + Maximo IT | ~8GB | **25GB** | 약 3배 |
| Stage 4 — Mongo/TSM/SLS/CFS/Db2 | ~96GB | **43GB** | **참고치보다 오히려 작음** |
| Stage 5 — CLI | — | 약 4GB | CLI 이미지 자체 크기 |
| Red Hat 콘텐츠 (플랫폼+Operator) | 약 80GB | 측정 중 | 완료 후 갱신 |
| **MAS 콘텐츠 소계** | ~108GB | **약 87GB** | |

⚠️ **이전 개정본의 "500~600GB 이상" 추정은 오류였습니다.** Stage 1이 참고치의 3.8배였다는 이유로 전체를 같은 배수로 외삽했는데, **Stage 4는 반대로 참고치의 절반 이하**로 나왔습니다. 항목마다 증감이 달라 단일 배수 외삽은 유효하지 않습니다.

**디스크 산정 기준**: 사이트 Bastion은 ① 반입한 미러 데이터와 ② Mirror Registry에 Push된 사본을 **동시에 보관**하므로 미러 총량의 약 2배가 필요합니다. 실측 기준으로 MAS 87GB + Red Hat(측정 중, 80~150GB 추정) ≈ 총 170~240GB이므로, **2배 + OS/여유를 감안해 사이트 Bastion은 500GB 이상, 안전하게 1TB를 권장**합니다. Red Hat 미러링이 끝나면 실측 총합으로 최종 확정하세요.

```bash
# 최종 실측 (모든 미러링 완료 후)
du -sh ~/mas-install/mirror/*
du -sh ~/mas-install
df -h /home
```

### 2.2 서버 기본 도구 설치 (podman 등)

이미지 미러링 명령(`podman run ...`)을 실행하려면 먼저 이 서버에 `podman`을 비롯한 기본 도구가 설치되어 있어야 합니다. root 계정에서 실행합니다.

```bash
hostnamectl
uname -m
cat /etc/redhat-release
timedatectl status

subscription-manager status
dnf repolist --enabled

dnf install -y \
  podman openssl tar gzip curl jq rsync \
  dnf-plugins-core createrepo_c

podman --version
```

완료 기준: RHEL 9.6 / x86_64 확인, 시간 동기화 정상, RHEL Subscription 또는 승인된 저장소 활성화, `podman --version` 정상 출력.

### 2.3 확보해야 할 항목

| # | 항목 | 상태 / 위치 | 확보 방법 |
|---|---|---|---|
| 1 | **IBM Entitlement Key** | ✅ `~/mas-install/licenses/entitlement_key.key` | [IBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary)에서 문자열로 수동 발급 |
| 2 | **MAS 라이선스** (AppPoints) | ✅ `~/mas-install/licenses/lincense_poc.dat` — 정식 라이선스로 확보됨 (⚠️ Maximo IT AppPoints 포함 여부는 별도 확인 필요, 아래 참고) | IBM License Key Center에서 수동 발급 |
| 3 | **Red Hat Pull Secret** | ✅ `~/mas-install/redhat/pull-secret.json` | [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/install/pull-secret)에서 수동 다운로드 |
| 4 | **사이트 Bastion용 RPM 오프라인 저장소** | ✅ `~/mas-install/rpms/` | 이 서버의 `dnf` 저장소 (아래 명령으로 다운로드) |
| 5 | **OpenShift Client / Installer / checksum** | ✅ `~/mas-install/ocp/` | `mirror.openshift.com`에서 아래 명령으로 다운로드 |
| 6 | **MAS CLI 이미지** (`quay.io/ibmmas/cli:23.4.1`) | ✅ `~/mas-install/mas-cli/mas-cli-23.4.1.tar` | `quay.io`에서 아래 명령으로 pull+save |
| 7 | **Mirror Registry 설치 패키지** | ✅ `~/mas-install/registry/mirror-registry-amd64.tar.gz` | Red Hat Downloads 포털 (아래 참고) |
| 8 | **Red Hat 콘텐츠 이미지 세트** (OCP 플랫폼 + 필수 Operator) | 아래 "8." 항목 참고 | `mas mirror-redhat-images`로 미러링, 수 시간 소요 |
| 9 | **MAS 콘텐츠 이미지 세트** (Core/Catalog, Manage+Maximo IT, 공통의존성+Db2, CLI) | 아래 "9." 항목 참고 | `mas mirror-images`로 Stage 1/2/4/5 미러링 |
| 10 | **NFS 동적 프로비저너 이미지** (RWX용) | 아래 "10." 항목 참고 | `podman pull` + `save` — ⚠️ MAS/Red Hat 미러링에 **포함되지 않음** |

**1. IBM Entitlement Key / 2. MAS 라이선스 / 3. Red Hat Pull Secret**은 각각 포털에서 수동으로 발급받아 저장합니다(자동화된 다운로드 명령이 없음, 위 목록의 링크 참고).

```bash
chmod 600 ~/mas-install/licenses/<entitlement-file>
chmod 600 ~/mas-install/redhat/pull-secret.json
```

**4. 사이트 Bastion용 RPM 오프라인 저장소**

🔴 **폐쇄망에서는 나중에 추가할 수 없으므로 아래 항목이 반드시 포함되어야 합니다.**

| 패키지 | 왜 필요한가 |
|---|---|
| `podman` | 모든 `mas` CLI 실행 (§3.3 이후 전부) |
| `dnsmasq`, `bind-utils` | DNS 서버 구성 및 `nslookup` 검증 (§3.2) |
| `firewalld` | 방화벽 (§3.2) |
| **`nfs-utils`** | Bastion을 RWX용 NFS 서버로 겸용 (§3.8.2 방식 A) |
| **`nmstate`** | ⚠️ **Agent-based Installer가 정적 네트워크(`networkConfig`) 검증에 `nmstatectl`을 요구** (§3.6) — 없으면 ISO 생성 단계에서 실패 |
| **`chrony`** | 폐쇄망 NTP 서버 (§3.2.1) — 시간 동기 실패는 인증서/클러스터 오류로 이어짐 |
| `openssl`, `jq`, `rsync`, `tar`, `gzip`, `createrepo_c` | 각종 검증·전달·저장소 작업 |

```bash
dnf download \
  --resolve \
  --alldeps \
  --destdir "$HOME/mas-install/rpms" \
  podman openssl jq rsync tar gzip \
  dnsmasq bind-utils firewalld createrepo_c \
  nfs-utils nmstate chrony

createrepo_c "$HOME/mas-install/rpms"

# 핵심 패키지 포함 확인
ls ~/mas-install/rpms | grep -iE 'nfs-utils|nmstate|chrony'
```

세 개 모두 나와야 합니다. `nmstate`가 빠지면 §3.6에서 `openshift-install agent create image`가 실패합니다.

**5. OpenShift Client / Installer / checksum**

```bash
export OCP_VERSION=4.20.30
export OCP_DIR="$HOME/mas-install/ocp"
export OCP_BASE_URL="https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/$OCP_VERSION"

mkdir -p "$OCP_DIR"

curl -fL "$OCP_BASE_URL/openshift-client-linux.tar.gz" -o "$OCP_DIR/openshift-client-linux.tar.gz"
curl -fL "$OCP_BASE_URL/openshift-install-linux.tar.gz" -o "$OCP_DIR/openshift-install-linux.tar.gz"
curl -fL "$OCP_BASE_URL/sha256sum.txt" -o "$OCP_DIR/sha256sum.txt"

cd "$OCP_DIR"
sha256sum -c <(grep -E 'openshift-(client|install)-linux.tar.gz' sha256sum.txt)
```

두 파일 모두 `OK`로 표시돼야 합니다.

**6. MAS CLI 이미지** (인터넷 연결 서버에서 pull 후 tar로 저장)

```bash
podman pull --arch amd64 quay.io/ibmmas/cli:23.4.1
podman image inspect quay.io/ibmmas/cli:23.4.1 --format '{{.Os}}/{{.Architecture}}'

podman save -o ~/mas-install/mas-cli/mas-cli-23.4.1.tar quay.io/ibmmas/cli:23.4.1
sha256sum ~/mas-install/mas-cli/mas-cli-23.4.1.tar
```

검사 결과는 `linux/amd64`여야 합니다.

**7. Mirror Registry 설치 패키지**

Red Hat OpenShift Downloads(<https://console.redhat.com/openshift/downloads>)의 **"mirror registry for Red Hat OpenShift"** 항목에서 RHEL 9 x86_64용 파일을 받습니다. 로그인 필요/시간 제한이 있는 다운로드 URL이라 브라우저로 직접 받거나, 발급된 URL을 아래처럼 사용합니다.

```bash
read -r MIRROR_REGISTRY_DOWNLOAD_URL
curl -fL "$MIRROR_REGISTRY_DOWNLOAD_URL" -o ~/mas-install/registry/mirror-registry-amd64.tar.gz
unset MIRROR_REGISTRY_DOWNLOAD_URL
```

> `oc-mirror` 플러그인은 별도로 받을 필요가 없습니다 — §1.2에서 확인했듯 MAS CLI 이미지(`quay.io/ibmmas/cli:23.4.1`) 안에 이미 내장되어 있습니다.

**8. Red Hat 콘텐츠 이미지 세트**

✅ **검증 완료** (`quay.io/ibmmas/cli:23.4.1 mas mirror-redhat-images --help` 실행 결과, 2026-07-29): 아래 플래그(`--mode`, `--dir`, `--pullsecret`, `--mirror-platform`, `--mirror-operators`, `--release`, `--min-version`, `--max-version`, `--no-confirm`)는 모두 실제 존재하는 이름으로 확인됐습니다. `-H/-P/-u/-p`(레지스트리 접속 정보)는 `direct`/`from-filesystem` 모드에서만 필요하고 `to-filesystem` 모드(아래 명령)에서는 불필요합니다.

이 단계는 수 시간이 걸릴 수 있어 SSH 연결이 끊겨도 계속 진행되도록 **`nohup`으로 백그라운드 실행**합니다.

**첫 실행이면 아래 블록을 건너뛰세요.** 재시도 시에는 아래 "재시도 원칙"에 따라 판단합니다.

> ### 🔁 재시도 원칙 (§5.2·§5.3 실측 근거)
>
> **미러링이 실패했다고 무조건 디렉터리를 비우지 마세요.** 원인에 따라 대응이 다릅니다.
>
> | 실패 원인 | 대응 | 이유 |
> |---|---|---|
> | 네트워크/레지스트리 타임아웃, 개별 이미지 실패 | **비우지 말고 그대로 재실행** | 이미 받은 콘텐츠는 건너뛰므로 훨씬 빠름 (§5.2에서 23GB 보존 확인) |
> | SELinux/권한 오류 (§5.1) | 원인(`:Z`→`:z`) 해결 후 **비우지 않고** 재실행 | 데이터 자체는 정상 |
> | 미러링 대상/버전/Registry 주소를 **변경**한 경우 | **완전 초기화** 후 재실행 | manifest·레이아웃이 달라짐 |
> | 산출물이 손상된 것으로 판단될 때 | **완전 초기화** | — |
>
> ⚠️ `rm -rf <dir>/*`는 `working-dir`(oc-mirror의 캐시·workspace·증분 메타데이터 포함)까지 삭제하므로 **다음 실행이 전체 재다운로드**가 됩니다. 완전 초기화가 정말 필요한 경우에만 사용하세요.

```bash
rm -rf ~/mas-install/mirror/redhat/*
ls -la ~/mas-install/mirror/redhat
```

```bash
export LOCAL_DIR="$HOME/mas-install/mirror"
export REGISTRY_AUTH_FILE="$HOME/mas-install/redhat/pull-secret.json"

nohup podman run --rm \
  -v "$LOCAL_DIR":/mnt/workspace:z \
  -v "$HOME/mas-install/redhat":/mnt/redhat:ro,z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-redhat-images \
  --mode to-filesystem \
  --dir /mnt/workspace/redhat \
  --pullsecret /mnt/redhat/pull-secret.json \
  --mirror-platform \
  --mirror-operators \
  --release 4.20 \
  --min-version 4.20.30 \
  --max-version 4.20.30 \
  --no-confirm \
  > "$HOME/mas-install/mirror/redhat-mirror.nohup.log" 2>&1 &

disown
echo "PID: $!"
```

**진행 상황 확인 명령** (다른 세션에서, 또는 같은 세션에서 언제든):

```bash
# nohup 자체 출력(대화형 진행률/오류)
tail -f ~/mas-install/mirror/redhat-mirror.nohup.log

# ansible 래퍼 로그 (파일명에 실행 시각이 매번 바뀌므로 와일드카드로 최신 것 확인)
tail -f ~/mas-install/mirror/redhat/logs/mirror-*-redhat.log

# oc-mirror 내부 상세 로그 (실제 이미지별 복사 성공/실패, 파일명 고정)
tail -f ~/mas-install/mirror/redhat/logs/mirror-to-filesystem-ocp4.20.log

# 프로세스가 살아서 CPU를 쓰고 있는지 (이름은 oc-mirror, 공백 아닌 하이픈)
ps aux | grep -i mirror | grep -v grep

# 받은 용량이 계속 늘어나는지 (⚠️ 아래 참고 — 이 디렉터리는 한동안 안 늘어날 수 있음)
watch -n 15 'du -sh ~/mas-install/mirror/redhat'

# 실제 진행량은 컨테이너 내부 저장소에서 확인 (아래 ⚠️ 참고)
podman system df -v

# 지금까지 실패한 이미지가 있는지 (완료 후 최종 확인용)
grep -c ERROR ~/mas-install/mirror/redhat/logs/mirror-to-filesystem-ocp4.20.log
```

이 명령이 미러링하는 대상(공식 문서 및 실제 실행 로그 확인):
- **OCP 4.20.30 플랫폼 release payload**
- **Red Hat Operator 다수**: 공식 문서에는 "필수 3종(DRO, Cert Manager, Grafana)"만 명시돼 있으나, 실제 실행 로그에서는 이보다 훨씬 많은 Operator가 미러링되는 것이 확인됨 — Strimzi/Kafka, OpenShift Serverless(Knative), Authorino, ODF(OpenShift Data Foundation), Grafana, PostgreSQL 등. "이 단계를 건너뛰면 MAS 설치가 실패한다"고 공식 문서에 명시됨
- 참고 저장공간: 공식 문서 기준 최소 약 **80GB** — 위 관찰 결과 실제로는 이보다 클 수 있음, `du -sh`로 실측 필요

⚠️ **중요 (실제 실행으로 확인, 2026-07-29)**: `-d /mnt/workspace/redhat`로 지정한 목적지 디렉터리는 미러링 도중 한동안 용량이 거의 늘지 않을 수 있습니다. oc-mirror가 이미지를 먼저 **컨테이너 내부 저장소(podman 로컬 볼륨)**에 캐시했다가 나중에 목적지로 내보내는 것으로 보이며, 실제 진행량은 `du -sh` 대신 `podman system df -v`의 해당 컨테이너 `SIZE` 컬럼으로 확인해야 정확합니다. **이 컨테이너는 `--rm` 옵션으로 실행되므로, 도중에 프로세스를 중단(kill)하면 컨테이너 내부에 쌓인 진행분(수십 GB)이 전부 소실됩니다 — 완료될 때까지 절대 중단하지 마세요.**

**9. MAS 콘텐츠 이미지 세트**

Red Hat 콘텐츠 미러링과 마찬가지로 시간이 오래 걸리므로 **`nohup`으로 백그라운드 실행**합니다. 하나의 스크립트로 묶지 않고 **Stage별로 독립된 `nohup` 프로세스 + 로그 파일**로 분리합니다 — 하나가 실패해도 다른 Stage 로그에 영향이 없고, 어디까지 끝났는지 명확합니다. 앞 Stage가 끝난 걸 확인한 뒤 다음 Stage를 실행하세요.

✅ **검증 완료 (실제 실행으로 확인, 2026-07-29)**: `mas mirror-images`는 `mas mirror-redhat-images`와 달리 **`to-filesystem` 모드에서도 `-H/-P/-u/-p`가 필수**입니다(생략하면 인자 파싱에 실패해 `--help`만 출력하고 종료됨).

🔴 **여기에는 최종 폐쇄망 Registry의 실제 주소를 넣으세요.** 공식 가이드는 Stage 0에서 `REGISTRY_HOST`/`REGISTRY_PORT`/`REGISTRY_USERNAME`/`REGISTRY_PASSWORD`를 한 번 export하고 모든 Stage가 이를 상속하도록 설계되어 있습니다(`--help`에도 *"each option may also be defined by setting the appropriate environment variable"* 명시). Registry Host/Port는 목적지 이미지 경로 생성에 사용되므로 더미값을 넣는 것은 권장되지 않습니다.

<details>
<summary>⚠️ 이번 배포는 더미값(<code>localhost:443</code>)으로 진행했으며, <b>실측 결과 산출물에는 영향이 없음을 확인</b>했습니다 (2026-07-30) — 클릭해서 근거 보기</summary>

`mas mirror-images`는 모드별로 manifest를 **3개** 생성하며, 목적지 표기가 각각 다릅니다.

| manifest | 목적지 표기 | 이번 실행에서 |
|---|---|---|
| `manifests/direct/` | `localhost:443/cp/mas/...` | 미사용 |
| **`manifests/to-filesystem/`** | **`file:///cp/mas/...`** | ✅ 실제 사용 — 레지스트리 주소 없음 |
| `manifests/from-filesystem/` | `localhost:443/...` | 아직 미사용 (§3.5에서 재생성됨) |

실증 확인 결과:
```bash
# to-filesystem manifest는 file:/// 목적지 → 레지스트리 주소 무관
head -3 ~/mas-install/mirror/core/manifests/to-filesystem/ibm-mas_9.2.0.txt
#   cp.icr.io/cp/mas/accapppoints@sha256:...=file:///cp/mas/accapppoints:3.11.74-amd64

# 디스크 레이아웃도 경로 기반 (localhost:443 디렉터리 없음)
ls ~/mas-install/mirror/core/v2/     # → cp, cpopen 만 존재
```

manifest들은 매 실행 시 CASE 번들에서 재생성되므로(ansible 태스크 `Generate the mirror manifest from the CASE bundle`, `Copy images-mapping-from-filesystem`), §3.5의 `from-filesystem` 실행 시 실제 Registry 주소로 다시 만들어집니다. 따라서 **이미 미러링한 데이터를 다시 받을 필요는 없습니다.**

</details>

**공통 (한 번만 export)**:

```bash
export LOCAL_DIR="$HOME/mas-install/mirror"
export MIRROR_SINGLE_ARCH=amd64
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)

# 최종 폐쇄망 Registry의 실제 값 (더미값 대신 이 값을 사용)
export REGISTRY_HOST=registry.<base-domain>
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=<registry-user>
read -s REGISTRY_PASSWORD
export REGISTRY_PASSWORD
```

**Stage 1: Core + Catalog**

**첫 실행이면 이 블록을 건너뛰세요.** 재시도 시에는 §2.3의 "재시도 원칙"에 따라 판단합니다 — 네트워크 오류면 **비우지 말고** 그대로 재실행하는 것이 빠릅니다.

```bash
rm -rf ~/mas-install/mirror/core/*
ls -la ~/mas-install/mirror/core
```

```bash
nohup podman run --rm \
  -e IBM_ENTITLEMENT_KEY -e MIRROR_SINGLE_ARCH \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m to-filesystem -d /mnt/registry/core \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-catalog --mirror-core \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/stage1-core.nohup.log" 2>&1 &
disown
echo "Stage1 PID: $!"
```

Stage 1 진행 상황 확인 명령 (로그 확인 + 실제 다운로드 진행 확인 — ★ `podman system df -v`가 아니라 `du -sh`로 봐야 함, 위 ✅ 정정 참고):

```bash
tail -f ~/mas-install/mirror/stage1-core.nohup.log
```

```bash
watch -n 15 'du -sh ~/mas-install/mirror/core/* 2>/dev/null'
```

**Stage 2: Manage + Maximo IT** (Stage 1 완료 확인 후 실행)

**첫 실행이면 이 블록을 건너뛰세요.** 재시도 시에는 §2.3의 "재시도 원칙"에 따라 판단합니다 — 네트워크 오류면 **비우지 말고** 그대로 재실행하는 것이 빠릅니다.

```bash
rm -rf ~/mas-install/mirror/apps/*
ls -la ~/mas-install/mirror/apps
```

```bash
nohup podman run --rm \
  -e IBM_ENTITLEMENT_KEY -e MIRROR_SINGLE_ARCH \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m to-filesystem -d /mnt/registry/apps \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-manage --mirror-icd \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/stage2-apps.nohup.log" 2>&1 &
disown
echo "Stage2 PID: $!"
```

Stage 2 진행 상황 확인 명령 (로그 확인 + 실제 다운로드 진행 확인):

```bash
tail -f ~/mas-install/mirror/stage2-apps.nohup.log
```

```bash
watch -n 15 'du -sh ~/mas-install/mirror/apps/* 2>/dev/null'
```

**Stage 3: Cloud Pak for Data (CP4D) — ⛔ 설치하지 않음, 실행 금지**

공식 IBM MAS CLI 이미지 미러링 가이드의 5단계 중 Stage 3은 Cloud Pak for Data(Watson Studio, Watson Machine Learning, Spark, Cognos 등) 콘텐츠입니다. **이번 설치는 Core + Manage + Maximo IT만 대상이며 CP4D는 사용하지 않으므로(§0 "미사용 대상" 참고), Stage 3은 의도적으로 건너뜁니다.** `--mirror-cp4d`, `--mirror-wsl`, `--mirror-wml`, `--mirror-spark`, `--mirror-cognos` 플래그는 어떤 Stage 명령에도 포함하지 않습니다. 실수로라도 이 플래그들을 넣지 않도록 주의하세요 — 용량이 매우 크고(CP4D 전체 약 273GB, MAS 8.10 기준 참고치) 이번 구성에는 불필요합니다.

**Stage 4: 공통 의존성 + 내부 Db2** (Stage 2 완료 확인 후 실행)

**첫 실행이면 이 블록을 건너뛰세요.** 재시도 시에는 §2.3의 "재시도 원칙"에 따라 판단합니다 — 네트워크 오류면 **비우지 말고** 그대로 재실행하는 것이 빠릅니다.

```bash
rm -rf ~/mas-install/mirror/other/*
ls -la ~/mas-install/mirror/other
```

```bash
nohup podman run --rm \
  -e IBM_ENTITLEMENT_KEY -e MIRROR_SINGLE_ARCH \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m to-filesystem -d /mnt/registry/other \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-mongo --mirror-tsm --mirror-sls --mirror-cfs --mirror-db2 \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/stage4-other.nohup.log" 2>&1 &
disown
echo "Stage4 PID: $!"
```

Stage 4 진행 상황 확인 명령 (로그 확인 + 실제 다운로드 진행 확인):

```bash
tail -f ~/mas-install/mirror/stage4-other.nohup.log
```

```bash
watch -n 15 'du -sh ~/mas-install/mirror/other/* 2>/dev/null'
```

**Stage 5: CLI 자체 이미지** (Stage 4 완료 확인 후 실행)

**첫 실행이면 이 블록을 건너뛰세요.** 재시도 시에는 §2.3의 "재시도 원칙"에 따라 판단합니다 — 네트워크 오류면 **비우지 말고** 그대로 재실행하는 것이 빠릅니다.

```bash
rm -rf ~/mas-install/mirror/cli/*
ls -la ~/mas-install/mirror/cli
```

```bash
nohup podman run --rm \
  -e IBM_ENTITLEMENT_KEY \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m to-filesystem -d /mnt/registry/cli \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-cli \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/stage5-cli.nohup.log" 2>&1 &
disown
echo "Stage5 PID: $!"
```

Stage 5 진행 상황 확인 명령 (로그 확인 + 실제 다운로드 진행 확인):

```bash
tail -f ~/mas-install/mirror/stage5-cli.nohup.log
```

```bash
watch -n 15 'du -sh ~/mas-install/mirror/cli/* 2>/dev/null'
```

**공통 진행 상황 확인 명령** (아무 Stage든 실행 중일 때):

```bash
# 프로세스가 살아있는지
ps aux | grep -i "mas mirror-images" | grep -v grep

# Stage별 받은 용량 (★ 이게 진짜 진행 지표 — 아래 참고)
du -sh ~/mas-install/mirror/core/* ~/mas-install/mirror/apps/* ~/mas-install/mirror/other/* ~/mas-install/mirror/cli/* 2>/dev/null
```

⚠️→✅ **정정 (실제 실행으로 확인, 2026-07-29)**: `mas mirror-images`는 `mas mirror-redhat-images`(oc-mirror 기반, 컨테이너 내부 캐시를 거쳐 나중에 목적지로 내보냄)와 달리, **`oc image mirror --dir=...`로 목적지(`/mnt/registry/<stage>/v2/`)에 바로 씁니다.** 그래서 이 Stage들은 `podman system df -v`(컨테이너 내부 SIZE)가 계속 31MB 근처로 고정돼 있어도 정상이며, **진짜 진행 상황은 `du -sh ~/mas-install/mirror/<stage>/v2` (또는 위 명령처럼 하위 디렉터리 전체)로 확인해야 합니다.** (위 8번 Red Hat 콘텐츠는 반대로 `podman system df -v`가 맞는 지표이니 헷갈리지 않도록 주의.)

또한 `mas mirror-images`는 이미지별 상세 진행 로그를 남기지 않습니다. `~/mas-install/mirror/<stage>/logs/mirror-*.log` 파일 하나만 있고, ansible의 "Run mirror command" 태스크가 끝나야 결과가 한꺼번에 찍히는 방식이라 진행 중에는 상세히 안 보입니다 — 대신 `du -sh`로 실시간 진행을 확인하세요.

✅ **검증 완료** (`quay.io/ibmmas/cli:23.4.1 mas mirror-images --help` 실행 결과, 2026-07-29): `--mirror-icd`는 정확히 **"Mirror image for IBM Maximo IT (Separately entitled IBM Maximo Manage extension)"** — 즉 Maximo IT 자체를 가리키는 플래그이며 MongoDB와 무관합니다. MongoDB는 `--mirror-mongo`(또는 특정 버전이 필요하면 `--mirror-mongo-v7`)가 별도로 존재합니다. Stage 4에서 쓰는 `--mirror-mongo`는 그대로 맞습니다.

참고로 `--help` 출력에서 추가로 확인된 플래그: `--mirror-odf`(OpenShift Data Foundation), `--mirror-everything`(전체 일괄 미러링). **이번 배포는 RWX를 외부 NFS로 결정했으므로 `--mirror-odf`는 실행하지 않습니다**(§3.8.2 참고). 나중에 ODF로 전환하려면 인터넷 연결 환경에서 이 미러링을 추가로 수행해야 합니다.

**10. NFS 동적 프로비저너 이미지 (RWX용)**

🔴 **주의**: 이 이미지는 **MAS 미러링(`mas mirror-images`)에도, Red Hat 미러링(`mas mirror-redhat-images`)에도 포함되지 않습니다.** 커뮤니티 프로젝트라 별도로 받아야 하며, 폐쇄망에 반입한 뒤에는 받을 수 없습니다. 이번 배포는 RWX를 Bastion NFS로 제공하므로(§3.8.2 방식 A) **반드시 지금 확보**해야 합니다.

`nfs-subdir-external-provisioner`(이미지 1개, 구성 단순)를 사용합니다.

```bash
mkdir -p ~/mas-install/nfs-provisioner

# 태그는 프로젝트 최신 릴리스를 확인해 조정 (v4.0.2는 널리 쓰이는 안정 버전)
export NFS_PROV_IMAGE=registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2

podman pull --arch amd64 "$NFS_PROV_IMAGE"
podman image inspect "$NFS_PROV_IMAGE" --format '{{.Os}}/{{.Architecture}}'

podman save -o ~/mas-install/nfs-provisioner/nfs-subdir-external-provisioner.tar "$NFS_PROV_IMAGE"
ls -lh ~/mas-install/nfs-provisioner/
```

배포용 YAML도 함께 받아둡니다 — 폐쇄망에서는 GitHub에 접근할 수 없습니다.

⚠️ `master` 브랜치는 언제든 바뀔 수 있으므로 **릴리스 태그로 고정**하는 것이 안전합니다(아래는 이미지 버전과 맞춘 `nfs-subdir-external-provisioner-4.0.18` 예시 — 태그가 없으면 `master`로 대체).

```bash
cd ~/mas-install/nfs-provisioner
export NFS_PROV_REF=nfs-subdir-external-provisioner-4.0.18   # 실패 시 master 로 변경

for f in rbac.yaml deployment.yaml class.yaml test-claim.yaml test-pod.yaml; do
  curl -fLO "https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/${NFS_PROV_REF}/deploy/$f" \
    || curl -fLO "https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/$f"
done

ls -la
```

YAML 5개(`rbac`, `deployment`, `class`, `test-claim`, `test-pod`)가 모두 있어야 합니다 — 뒤 2개는 §3.8.4 검증에 사용합니다.

✅ **확보 완료 (2026-07-30)**: `v4.0.2` 태그 유효(`linux/amd64` 확인), tar 43MB, YAML 5개 정상 수신. 받은 `deployment.yaml`은 반입 후 NFS 서버 주소/경로/이미지 경로를 수정해야 하며, 구체적인 `sed` 명령은 §3.8.2 방식 A의 ②에 실제 기본값 기준으로 정리해두었습니다.

> `csi-driver-nfs`를 대신 쓸 경우 CSI sidecar까지 5~6개 이미지(`csi-provisioner`, `csi-node-driver-registrar`, `livenessprobe` 등)를 모두 받아야 합니다 — 이번 배포에서는 단순함을 위해 `nfs-subdir` 방식을 택했습니다.

이 이미지도 §2.5의 전달 대상에 포함해야 합니다(`nfs-provisioner/` 디렉터리).

### 2.4 설치 파일 최종 확인

§2.3의 8/9번(Red Hat 콘텐츠, MAS 콘텐츠 Stage 1/2/4/5) 미러링이 모두 끝나면, 다음 단계(전달/설치)로 넘어가기 전 아래 순서로 정상 종료를 확인합니다.

**1) 디렉터리 구조 확인**

```text
~/mas-install/
  licenses/   (entitlement_key.key, lincense_poc.dat)
  redhat/     (pull-secret.json)
  ocp/        (openshift-client-linux.tar.gz, openshift-install-linux.tar.gz, sha256sum.txt)
  registry/   (mirror-registry-amd64.tar.gz)
  mas-cli/    (mas-cli-23.4.1.tar)
  rpms/            (사이트 Bastion용 RPM — nfs-utils 포함 확인 필요)
  nfs-provisioner/ (nfs-subdir-external-provisioner.tar + rbac/deployment/class.yaml)
  mirror/
    redhat/   (mas mirror-redhat-images 결과물)
    core/ apps/ other/ cli/  (mas mirror-images 결과물, Stage별)
```

**2) 컨테이너가 정상 종료 확인**

```bash
podman ps -a
```

결과에 아무것도 안 나와야 합니다 (모든 미러링 컨테이너가 `--rm`으로 자동 삭제됨).

**3) 각 단계 로그에서 성공/실패 확인** — `[SKIPPED]`는 정상(선택 안 한 항목), `[SUCCESS]`가 있어야 하고 `[FAILURE]`가 없어야 합니다.

```bash
grep -E "^\[SUCCESS\]|^\[FAILURE\]" ~/mas-install/mirror/redhat-mirror.nohup.log
grep -E "^\[SUCCESS\]|^\[FAILURE\]" ~/mas-install/mirror/stage1-core.nohup.log
grep -E "^\[SUCCESS\]|^\[FAILURE\]" ~/mas-install/mirror/stage2-apps*.nohup.log
grep -E "^\[SUCCESS\]|^\[FAILURE\]" ~/mas-install/mirror/stage4-other.nohup.log
grep -E "^\[SUCCESS\]|^\[FAILURE\]" ~/mas-install/mirror/stage5-cli.nohup.log
```

**4) (Red Hat/oc-mirror 전용) 개별 이미지 실패 목록 확인** — MAS 콘텐츠(Stage 1/2/4/5)는 `working-dir` 구조가 없어 해당 없음, Red Hat 콘텐츠만 해당됩니다. 아래 결과가 비어 있어야(오류 없어야) 정상입니다.

```bash
find "$LOCAL_DIR" -path '*/working-dir/logs/mirroring_errors_*' -type f -size +0c -print
```

**5) 최종 용량 확인** (더 이상 안 늘어나는지 몇 분 간격으로 재확인)

```bash
du -sh ~/mas-install/mirror/redhat ~/mas-install/mirror/core/* ~/mas-install/mirror/apps/* ~/mas-install/mirror/other/* ~/mas-install/mirror/cli/* 2>/dev/null
```

### 2.5 설치 파일 전달

§2.4의 최종 확인이 모두 통과하면 전달용 tar 파일을 생성해 실제 사이트 Bastion 서버로 옮겨서 설치합니다.

**1) 체크섬 생성** (무결성 검증용)

```bash
cd "$HOME/mas-install"
find licenses redhat ocp registry mas-cli rpms nfs-provisioner mirror \
  -type f -print0 | sort -z | xargs -0 sha256sum > transfer-files.sha256
```

**2) 전달 매체로 옮길 tar 조각 생성** — 실제 사이트 서버가 USB/외장디스크로 반입해야 하는 폐쇄망이라면, 통째로 옮기기 전 tar로 묶고 용량이 크므로 분할합니다.

```bash
export EXPORT_DIR=<전달매체 마운트 경로>/mas-9.2-offline
mkdir -p "$EXPORT_DIR"

tar -C "$HOME/mas-install" -cf - \
  licenses redhat ocp registry mas-cli rpms nfs-provisioner mirror transfer-files.sha256 \
  | split -b 50G - "$EXPORT_DIR/mas-offline.tar.part-"

cd "$EXPORT_DIR"
sha256sum mas-offline.tar.part-* > transfer-parts.sha256

du -sh "$EXPORT_DIR"
```

**3) 실제 Bastion 서버로 반입**

tar 조각들을 실제 bastion 으로 반입합니다.

**4) 반입 후 파일 압축 해제 및 파일 검증** (§3.1 진행 전에 먼저 실행)

```bash
cd <전달매체 마운트 경로>/mas-9.2-offline
sha256sum -c transfer-parts.sha256

mkdir -p ~/mas-install
cat mas-offline.tar.part-* | tar -xf - -C ~/mas-install

cd ~/mas-install
sha256sum -c transfer-files.sha256
```

두 checksum 검증이 모두 통과해야(`OK`) 다음 단계(§3.1)로 진행합니다. 라이선스/Pull Secret 등 보안 파일은 일반 전송 매체와 분리된 절차로 반입하는 것을 권장합니다(IBM Entitlement Key, `lincense_poc.dat`, `pull-secret.json`).

> 참고: 지금처럼 같은 서버에서 계속 진행하는 테스트 단계에서는 1)/2)/3)번을 생략하고 `~/mas-install`을 그대로 둔 채 §3으로 넘어가면 됩니다. 실제 사이트 서버로 옮길 때만 필요합니다.

---

## 3. 설치 과정

**실제 사이트 Bastion 서버**에서 진행합니다.

### 계정 사용 원칙

각 절 제목에 실행 계정을 표시했습니다. `maximo` 계정으로 SSH 접속한 뒤, root가 필요한 구간에서만 `sudo -i`로 전환하는 방식을 권장합니다.

```bash
ssh maximo@192.168.2.210
sudo -i          # root 구간(§3.1, §3.2, §3.4 CA 등록, §3.8.2 NFS 서버) 진입
exit             # 끝나면 maximo로 복귀
```

| 구간 | 계정 | root가 필요한 이유 |
|---|---|---|
| §3.1 | **root** | `/etc/yum.repos.d/`, 시스템 패키지 설치 |
| §3.2 | **root** | `/etc/dnsmasq.d/`, `systemctl`, `firewall-cmd` |
| §3.4 (일부) | **root** | `/var/lib/mirror-registry` 생성, `/etc/containers/certs.d/`·`/etc/pki/ca-trust/` CA 등록 |
| §3.8.2 A-1 | **root** | `/etc/exports`, `nfs-server`, 방화벽 |
| 그 외 전부 | `maximo` | `mas` CLI, Mirror Registry 설치·운영, 이미지 push, `mas install` 등 (rootless podman) |

🔴 **주의 — `~` 경로 함정**: `sudo -i` 후에는 `$HOME`이 `/root`가 되므로 **`~/mas-install/...`이 `/root/mas-install/...`로 해석**됩니다. 그래서 이 문서의 **root 구간은 모두 절대 경로**(`/home/maximo/mas-install/...`)로 작성했고, **`maximo` 구간은 `~`** 를 씁니다. 명령을 복사할 때 현재 어느 셸에 있는지 (`whoami`로) 확인하세요.

```bash
whoami   # root 인지 maximo 인지 확인
```

### 3.1 RPM 오프라인 저장소 등록 및 도구 설치 (root 계정)

```bash
cat > /etc/yum.repos.d/mas-offline.repo <<'EOF'
[mas-offline]
name=MAS Offline RPM Repository
baseurl=file:///home/maximo/mas-install/rpms
enabled=1
gpgcheck=1
repo_gpgcheck=0
EOF

dnf --disablerepo='*' --enablerepo=mas-offline makecache
dnf --disablerepo='*' --enablerepo=mas-offline install -y \
  podman openssl jq rsync tar gzip dnsmasq bind-utils firewalld createrepo_c \
  nfs-utils
```

`--disablerepo='*'`로 외부 저장소를 전부 끊고 반입한 `mas-offline`만 사용하므로 인터넷 없이 설치됩니다. 설치 후 확인합니다.

```bash
podman --version
dnsmasq --version | head -1
rpm -q nfs-utils firewalld bind-utils
```

- `nfs-utils`는 §3.8.2(Bastion을 RWX용 NFS 서버로 구성)에서 사용합니다.
- ⚠️ `gpgcheck=1`이므로 RPM 서명 검증에 Red Hat GPG 키가 필요합니다. 최소 설치 등으로 키가 없어 실패하면 `/etc/pki/rpm-gpg/`에 키가 있는지 확인하고, 없으면 인터넷 연결 서버에서 함께 반입하세요(정책상 불가피할 때만 `gpgcheck=0`으로 완화 — 권장하지 않음).

### 3.2 DNS 구성 (root 계정)

✅ **공식 문서 확인** (Red Hat Agent-based Installer 사전 요구사항): SNO 설치가 완료되려면 아래 이름이 반드시 resolve되어야 합니다. **누락 시 SNO 설치 자체가 끝나지 않습니다.**

| 이름 | 값 |
|---|---|
| `api.<cluster-name>.<base-domain>` | SNO IP (`192.168.2.211`) |
| `api-int.<cluster-name>.<base-domain>` | SNO IP (`192.168.2.211`, api와 동일) |
| `*.apps.<cluster-name>.<base-domain>` | SNO IP (`192.168.2.211`) |
| `registry.<base-domain>` | Bastion IP (`192.168.2.210`) |

`/etc/dnsmasq.d/mas.conf`:

```ini
server=<upstream-dns-ip>

address=/api.<cluster-name>.<base-domain>/192.168.2.211
address=/api-int.<cluster-name>.<base-domain>/192.168.2.211
address=/.apps.<cluster-name>.<base-domain>/192.168.2.211
address=/registry.<base-domain>/192.168.2.210
```

```bash
systemctl enable --now dnsmasq
systemctl enable --now firewalld

firewall-cmd --permanent --add-service=dns          # 53/tcp,udp — SNO/클라이언트 → Bastion DNS
firewall-cmd --permanent --add-port=8443/tcp        # Mirror Registry
firewall-cmd --permanent --add-port=6443/tcp        # OpenShift API (Bastion/관리자 → SNO)
firewall-cmd --permanent --add-port=443/tcp         # OpenShift Router (MAS UI, Console)
firewall-cmd --permanent --add-port=80/tcp          # HTTP redirect
firewall-cmd --reload
firewall-cmd --list-all

nslookup api.<cluster-name>.<base-domain>
nslookup api-int.<cluster-name>.<base-domain>
nslookup test.apps.<cluster-name>.<base-domain>
nslookup registry.<base-domain>
```

> 위 포트는 이 Bastion 자체 방화벽 기준입니다. 6443/443/80은 SNO 노드로 향하는 트래픽이므로 SNO 쪽(및 중간 네트워크 장비)에서도 허용되어야 하며, SNO 내부 통신용 `22623/tcp`(Machine Config Server)는 노드 자체에서 처리합니다. 실제 정책은 사이트 방화벽 규정과 Red Hat OCP 4.20 네트워크 요구사항을 함께 적용하세요.

4개 모두 정상 응답(위 표의 IP)해야 다음 단계로 진행합니다. SNO 노드와 클라이언트가 이 Bastion을 DNS 상위 서버로 사용하도록 네트워크 설정(DHCP 또는 고정 DNS)도 함께 확인합니다.

**추가 확인 — 정방향 외 레코드**

사이트 환경에 따라 아래도 필요할 수 있습니다. Red Hat 베어메탈/UPI 설치 요구사항에서 역방향(PTR) 레코드를 권장하므로, 사이트 DNS 정책에 맞춰 확인하세요.

| 항목 | 확인 내용 |
|---|---|
| SNO 호스트명 | `<sno-hostname>.<cluster-name>.<base-domain>` → `192.168.2.211` |
| PTR(역방향) | `192.168.2.211` → SNO 호스트명, `192.168.2.210` → Bastion |

```bash
nslookup 192.168.2.211
nslookup 192.168.2.210
```

⚠️ **미검증**: SNO 단일 노드 구성에서 PTR 레코드가 필수인지 권장인지는 사이트 환경/설치 방식에 따라 다릅니다. 최소한 정방향 4건은 반드시 동작해야 하며, PTR은 준비해두는 편이 안전합니다.

#### 3.2.1 NTP 구성 (root 계정)

🔴 **필수**: 시간 동기가 어긋나면 인증서 검증 실패, etcd 이상, Operator 오류 등으로 이어집니다. 폐쇄망에서는 외부 NTP에 접근할 수 없으므로 **Bastion을 내부 NTP 서버로** 구성합니다.

```bash
rpm -q chrony || dnf --disablerepo='*' --enablerepo=mas-offline install -y chrony
```

`/etc/chrony.conf`에 내부 네트워크 대상 서비스를 허용합니다.

```bash
cat >> /etc/chrony.conf <<'EOF'

# 폐쇄망 내부 NTP 서비스 제공
allow 192.168.0.0/22
# 상위 NTP가 없는 완전 폐쇄망이면 로컬 클럭을 기준으로 제공
local stratum 10
EOF

systemctl enable --now chronyd
systemctl restart chronyd

firewall-cmd --permanent --add-service=ntp     # 123/udp
firewall-cmd --reload

chronyc tracking
chronyc clients
timedatectl status
```

> 사이트에 상위 NTP 서버가 있으면 `local stratum 10` 대신 `server <upstream-ntp-ip> iburst`를 넣는 것이 정확합니다. `local stratum 10`은 상위 시간원이 전혀 없는 완전 폐쇄망용 대안입니다.

**SNO 노드에 이 NTP를 알려주는 방법**: §3.6의 `agent-config.yaml`에 `additionalNTPSources`로 Bastion IP를 지정합니다(해당 절 참고). DHCP로 NTP를 배포하는 환경이면 DHCP 옵션으로도 가능합니다.

### 3.3 MAS CLI 이미지 불러오기 (`maximo` 계정)

사이트 Bastion은 인터넷이 없어 `quay.io`에서 `quay.io/ibmmas/cli:23.4.1`을 직접 pull할 수 없습니다. 인터넷 연결 서버에서 미리 저장해 온 tar 파일을 불러옵니다.

```bash
podman load -i ~/mas-install/mas-cli/mas-cli-23.4.1.tar
podman image exists quay.io/ibmmas/cli:23.4.1
```

### 3.4 Mirror Registry 설치 (`maximo` 계정)

먼저 Registry 데이터 저장 디렉터리를 **root 계정**으로 만들고 `maximo`에게 소유권을 줍니다. `/var/lib/mirror-registry`에는 별도의 대용량 디스크를 마운트한 상태여야 합니다(§2.1 참고 — 최소 1TB 권장).

```bash
install -d -m 0750 -o maximo -g maximo /home/maximo/quay-install
install -d -m 0750 -o maximo -g maximo /var/lib/mirror-registry
install -d -m 0750 -o maximo -g maximo /var/lib/mirror-registry/quay
install -d -m 0750 -o maximo -g maximo /var/lib/mirror-registry/sqlite

findmnt -T /var/lib/mirror-registry
df -hT /var/lib/mirror-registry
```

이제 `maximo` 계정으로 설치합니다.

```bash
mkdir -p ~/mas-install/registry/bin
tar -xzf ~/mas-install/registry/mirror-registry-amd64.tar.gz -C ~/mas-install/registry/bin

cd ~/mas-install/registry/bin
./mirror-registry install -v \
  --quayHostname registry.<base-domain> \
  --quayRoot /home/maximo/quay-install \
  --quayStorage /var/lib/mirror-registry/quay \
  --sqliteStorage /var/lib/mirror-registry/sqlite
```

⚠️ `--quayStorage`/`--sqliteStorage` 플래그명은 mirror-registry 버전에 따라 다를 수 있습니다(구버전은 PostgreSQL 기반으로 `--pgStorage` 사용). 실행 전 `./mirror-registry install --help`로 확인하세요.

설치 결과로 나오는 초기 사용자명/비밀번호는 보안 저장소에만 보관합니다.

⚠️ **지원 범위 주의 (Red Hat 공식)**: `mirror registry for Red Hat OpenShift`는 **OCP 설치용 미러링을 위한 소규모·비HA·로컬 저장소 기반 Registry**입니다. OCP와 MAS 전체 이미지를 담는 **장기 운영 Registry로 쓰는 것은 지원 범위를 벗어납니다.** 현재 구성은 **PoC/검증 용도**로 보고, 운영 전환 시에는 Red Hat Quay나 사내 표준 Registry(Harbor/Artifactory 등) 등 운영급 Registry를 검토하세요.

또한 **이후 §3.6의 Pull Secret 준비를 위해 읽기 전용(pull-only) 계정을 지금 함께 만들어 두세요** — Quay 웹 UI(`https://registry.<base-domain>:8443`)에서 별도 사용자를 만들고 해당 조직/리포지토리에 읽기 권한만 부여합니다.

Bastion 자체에도 이 Registry의 CA를 신뢰하도록 등록합니다 (아래는 **root 계정**에서 실행 — 시스템 전역 인증서 저장소를 건드리므로).

```bash
REGISTRY_CA="$(find /home/maximo/quay-install -name rootCA.pem -type f -print -quit)"
install -d -m 0755 /etc/containers/certs.d/registry.<base-domain>:8443
install -m 0644 "$REGISTRY_CA" /etc/containers/certs.d/registry.<base-domain>:8443/ca.crt
install -m 0644 "$REGISTRY_CA" /etc/pki/ca-trust/source/anchors/registry.<base-domain>.crt
update-ca-trust
```

### 3.5 Registry로 이미지 Push (from-filesystem) (`maximo` 계정)

✅ **검증(2026-07-30, GitHub 원본 `docs/guides/image-mirroring.md` 확인)**: `from-filesystem` 모드에도 `-c`(catalog)/`-C`(channel)가 `to-filesystem`과 동일하게 필요합니다. 단 `--release`/`--min-version`/`--max-version`은 `to-filesystem` 전용이라 여기선 안 씁니다.

🔴 **실행 전 필수 확인 — manifest 목적지 주소**: `to-filesystem` 단계에서 Registry 주소로 더미값을 썼다면(§2.3의 9번 참고), `manifests/from-filesystem/` manifest에 그 더미값이 남아 있을 수 있습니다. 이 manifest는 매 실행 시 재생성되지만, **첫 Stage를 실행한 직후 반드시 실제 Registry 주소로 바뀌었는지 확인**하세요.

```bash
# 첫 Stage(core) 실행 후 확인 — 실제 Registry 주소가 보여야 하고 localhost가 없어야 함
head -3 ~/mas-install/mirror/core/manifests/from-filesystem/ibm-mas_9.2.0.txt
grep -c localhost ~/mas-install/mirror/core/manifests/from-filesystem/ibm-mas_9.2.0.txt   # 0 이어야 정상
```

`localhost`가 그대로 남아 있으면 **push 목적지가 잘못되므로 즉시 중단**하고, 해당 manifest가 왜 재생성되지 않았는지(캐시/권한) 확인해야 합니다.

```bash
export REGISTRY_HOST=registry.<base-domain>
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=<registry-user>
export LOCAL_DIR="$HOME/mas-install/mirror"
read -s REGISTRY_PASSWORD
export REGISTRY_PASSWORD
```

**Red Hat 콘텐츠**

```bash
podman run --rm \
  -v "$LOCAL_DIR":/mnt/workspace:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-redhat-images \
  --mode from-filesystem \
  --dir /mnt/workspace/redhat \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  --no-confirm
```

> `from-filesystem` 모드는 대상 Registry로 push하는 단계이므로 `--pullsecret`(외부 Red Hat Registry 인증)이 필요하지 않습니다 — 공식 문서 예시에도 없습니다.

**MAS 콘텐츠 — Stage 1: Core + Catalog**

```bash
podman run --rm \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/core \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-catalog --mirror-core \
  --no-confirm
```

**MAS 콘텐츠 — Stage 2: Manage + Maximo IT**

```bash
podman run --rm \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/apps \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-manage --mirror-icd \
  --no-confirm
```

**MAS 콘텐츠 — Stage 4: 공통 의존성 + 내부 Db2**

```bash
podman run --rm \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/other \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-mongo --mirror-tsm --mirror-sls --mirror-cfs --mirror-db2 \
  --no-confirm
```

**MAS 콘텐츠 — Stage 5: CLI 자체 이미지**

```bash
podman run --rm \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/cli \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-cli \
  --no-confirm

unset REGISTRY_PASSWORD
```

### 3.6 OCP SNO 클러스터 설치 (`maximo` 계정)

`oc`/`kubectl`/`openshift-install`을 이 서버의 PATH에 준비합니다 (§2.3의 5번에서 받아둔 파일 사용).

```bash
mkdir -p ~/.local/bin

tar -xzf ~/mas-install/ocp/openshift-client-linux.tar.gz -C ~/mas-install/ocp
install -m 0755 ~/mas-install/ocp/oc ~/mas-install/ocp/kubectl ~/.local/bin/

tar -xzf ~/mas-install/ocp/openshift-install-linux.tar.gz -C ~/mas-install/ocp
install -m 0755 ~/mas-install/ocp/openshift-install ~/.local/bin/

export PATH="$HOME/.local/bin:$PATH"
oc version --client
openshift-install version
```

```bash
mkdir -p ~/ocp-sno && cd ~/ocp-sno
```

**Pull Secret 준비 — Red Hat + Mirror Registry 병합**

🔴 **필수**: `install-config.yaml`의 `pullSecret`에는 **Red Hat Pull Secret과 폐쇄망 Registry 인증정보를 병합한 JSON**이 필요합니다. 이 값은 **모든 OCP 노드에 배포**되므로 아래 두 가지를 지켜야 합니다.

- ⚠️ **Registry에는 읽기 전용(pull-only) 계정을 사용하세요.** 이미지 Push용 쓰기 계정을 넣으면 모든 노드에 Registry 쓰기 권한이 배포됩니다.
- 폐쇄망이라 `registry.redhat.io` 등 외부 항목은 실제로 사용되지 않지만, Installer 검증을 위해 형식상 남겨둡니다.

```bash
# Mirror Registry에 읽기 전용 계정을 먼저 생성해 두세요 (Quay 웹 UI 또는 API)
read -s PULL_ONLY_PASSWORD

cp ~/mas-install/redhat/pull-secret.json ~/ocp-sno/pull-secret-merged.json

printf '%s' "$PULL_ONLY_PASSWORD" | podman login \
  registry.<base-domain>:8443 \
  --username <registry-pull-only-user> \
  --password-stdin \
  --authfile ~/ocp-sno/pull-secret-merged.json

unset PULL_ONLY_PASSWORD
chmod 600 ~/ocp-sno/pull-secret-merged.json

# 병합 확인 — registry.<base-domain>:8443 항목이 추가되어야 함
jq -r '.auths | keys[]' ~/ocp-sno/pull-secret-merged.json
```

원본 `~/mas-install/redhat/pull-secret.json`은 수정하지 않습니다. 이렇게 만든 `pull-secret-merged.json`의 **전체 내용을 한 줄로** `install-config.yaml`의 `pullSecret` 값에 넣습니다.

```bash
jq -c . ~/ocp-sno/pull-secret-merged.json
```

**CA 인증서 준비** — `additionalTrustBundle`에 넣을 Mirror Registry의 root CA입니다.

```bash
find /home/maximo/quay-install -name rootCA.pem -type f
cat "$(find /home/maximo/quay-install -name rootCA.pem -type f -print -quit)"
```

`install-config.yaml` (플레이스홀더는 실제 값으로 교체):

```yaml
apiVersion: v1
baseDomain: <base-domain>
metadata:
  name: <cluster-name>
compute:
  - name: worker
    replicas: 0
controlPlane:
  name: master
  replicas: 1
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  machineNetwork:
    - cidr: 192.168.0.0/22
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
pullSecret: '<mirror-registry-authfile-json>'
sshKey: '<ssh-public-key>'
additionalTrustBundle: |
  <mirror-registry-root-ca-pem>
imageDigestSources:
  - source: <mirror-redhat-images 미러링 로그의 source>
    mirrors:
      - <mirror-redhat-images 미러링 로그의 mirror>
```

⚠️ **미검증 — 필드명(`imageDigestSources` vs `imageContentSources`)**: 최신 OCP Installer 타입에는 `imageDigestSources`가 존재하지만, **Red Hat Agent-based Installer 공식 예제는 여전히 `imageContentSources`를 사용**하는 곳이 있어 문서 간 표기가 엇갈립니다. **OCP 4.20.30 실제 Installer로 검증 전에는 확정하지 마세요.**

```bash
# 실제 Installer가 인식하는 필드 확인 — 잘못된 필드는 경고/오류로 걸러짐
openshift-install explain installconfig | grep -iE 'imageDigestSources|imageContentSources'
```

인식되지 않으면 다른 쪽 필드명으로 바꿔서 다시 시도하세요.

> 개념 정리: `imageDigestSources`/`imageContentSources`는 **부트스트랩 단계**(클러스터가 아직 없을 때) 이미지 리다이렉션용이고, `ImageDigestMirrorSet`(IDMS)는 **클러스터 생성 후** 적용하는 리소스입니다(§3.7.1의 `oc apply`, §3.7.2의 `mas configure-airgap`). 둘 다 필요하며 서로를 대체하지 않습니다.

다만 `imageDigestSources`에 넣을 정확한 source/mirror 쌍의 파일 위치는 완전히 확정하지 못했습니다. oc-mirror v2가 생성하는 클러스터 리소스는 보통 `working-dir/cluster-resources/`(예: `idms-oc-mirror.yaml`) 아래에 있습니다. **첫 실행 시 아래로 실제 위치를 확인**하세요.

```bash
find ~/mas-install/mirror/redhat/working-dir -iname "*idms*" -o -iname "*cluster-resources*"
cat ~/mas-install/mirror/redhat/working-dir/cluster-resources/idms-oc-mirror.yaml 2>/dev/null
```

찾은 IDMS yaml의 `spec.imageDigestMirrors` 목록(각 `source`/`mirrors`)을 install-config.yaml의 `imageDigestSources`에 그대로 옮겨 적습니다(필드 구조가 거의 동일).

`agent-config.yaml`:

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: <cluster-name>
rendezvousIP: 192.168.2.211
additionalNTPSources:
  - 192.168.2.210          # §3.2.1에서 구성한 Bastion NTP
hosts:
  - hostname: <sno-hostname>
    role: master
    interfaces:
      - name: <nic-name>
        macAddress: "00:50:56:bb:86:df"
    rootDeviceHints:
      deviceName: /dev/<os-disk>
    networkConfig:
      interfaces:
        - name: <nic-name>
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.2.211
                prefix-length: 22
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.2.210
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: <nic-name>
```

⚠️ **중요**: `openshift-install`은 실행 시 `install-config.yaml`과 `agent-config.yaml`을 **소비(consume)하며 원본을 삭제**합니다. 재시도할 때 처음부터 다시 작성하지 않도록 반드시 사본을 먼저 백업하세요.

```bash
mkdir -p ~/ocp-sno-backup
cp install-config.yaml agent-config.yaml ~/ocp-sno-backup/
```

```bash
openshift-install agent create image --dir .
# agent.x86_64.iso를 SNO 노드에 연결·부팅

openshift-install agent wait-for bootstrap-complete --dir . --log-level=info
openshift-install agent wait-for install-complete --dir . --log-level=info

export KUBECONFIG="$HOME/ocp-sno/auth/kubeconfig"
oc get nodes
oc get co
```

모든 Cluster Operator가 `AVAILABLE=True / PROGRESSING=False / DEGRADED=False`인지 확인합니다.

### 3.7 Airgap 구성 (`maximo` 계정)

이 단계는 **두 부분**으로 나뉩니다. 순서를 반드시 지키세요.

#### 3.7.1 oc-mirror 생성 리소스 적용 + OperatorHub 기본 소스 비활성화

🔴 **필수 단계**: `mas configure-airgap`은 Registry 접근 검증 / 전역 Pull Secret / IDMS / 사용자 CA 신뢰 / MachineConfig 롤아웃을 처리하지만, **공식 커맨드 레퍼런스 기준으로 Red Hat Operator용 CatalogSource는 생성하지 않습니다.** 따라서 **폐쇄망에서 OLM이 미러링된 Operator 카탈로그를 볼 수 있게 만드는 작업은 이 단계에서 직접 해야 합니다.** 이걸 건너뛰면 cert-manager / DRO / Grafana 등 **MAS 필수 Red Hat Operator를 설치할 수 없어 `mas install`이 실패합니다.**

⚠️ **미검증 — 두 문서가 엇갈립니다**: IBM 공식 **설치 가이드**는 DRO/Grafana Operator 설치를 위해 `mas configure-airgap --setup-redhat-catalogs`를 사용하도록 안내하는 반면, **커맨드 레퍼런스**의 옵션 목록(`-H`,`-P`,`-u`,`-p`,`--ca-file`,`--no-confirm`,`-h`)에는 이 플래그가 없습니다. **CLI 23.4.1의 실제 도움말로 먼저 확인하세요.**

```bash
podman run --rm quay.io/ibmmas/cli:23.4.1 mas configure-airgap --help
```

- 플래그가 **있으면** → §3.7.2에서 `--setup-redhat-catalogs`를 추가해 CatalogSource까지 한 번에 구성(아래 수동 절차 생략 가능)
- 플래그가 **없으면** → 아래 수동 절차를 그대로 수행

먼저 폐쇄망 클러스터에서 기본 OperatorHub 소스(인터넷의 `registry.redhat.io` 등을 가리킴)를 비활성화합니다.

```bash
oc patch OperatorHub cluster --type json \
  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

oc get catalogsource -n openshift-marketplace
```

다음으로 `mas mirror-redhat-images`(내부적으로 `oc mirror --v2`)가 생성한 클러스터 리소스(IDMS/ITMS/CatalogSource)를 적용합니다.

```bash
# 생성된 리소스 위치 확인
find ~/mas-install/mirror/redhat/working-dir -type d -name "cluster-resources"
ls -la ~/mas-install/mirror/redhat/working-dir/cluster-resources/

# 내용을 먼저 검토한 뒤 적용
oc apply -f ~/mas-install/mirror/redhat/working-dir/cluster-resources/
```

IDMS/ITMS 적용은 **MachineConfig 롤아웃(노드 재시작)**을 유발하므로, 다음 단계로 넘어가기 전 반드시 안정화를 기다립니다.

```bash
watch oc get mcp          # UPDATING=False, DEGRADED=False 될 때까지
oc get nodes
oc get imagedigestmirrorset
oc get imagetagmirrorset
oc get catalogsource -n openshift-marketplace
```

CatalogSource가 모두 `READY` 상태여야 하고, **MAS가 필요로 하는 Red Hat Operator**가 조회되어야 합니다.

```bash
oc get packagemanifest -n openshift-marketplace | grep -E 'cert-manager|grafana|ibm-data-reporter|ibm-metrics|lvms'
```

> 여기서 확인하는 건 **Red Hat/커뮤니티 Operator**입니다. **IBM Maximo Operator Catalog는 이 시점에 없어도 정상**입니다 — §3.9의 `mas install`이 선택한 카탈로그 버전으로 CatalogSource를 생성합니다.

#### 3.7.2 mas configure-airgap 실행

```bash
REGISTRY_CA="$(find "$HOME/quay-install" -name rootCA.pem -type f -print -quit)"
read -s REGISTRY_PASSWORD
export REGISTRY_PASSWORD

podman run -ti --rm \
  -e REGISTRY_PASSWORD \
  -v "$REGISTRY_CA":/mnt/registry-ca.pem:ro,z \
  quay.io/ibmmas/cli:23.4.1 bash -c "
    oc login --token=<ocp-token> --server=https://api.<cluster-name>.<base-domain>:6443 &&
    mas configure-airgap \
      -H registry.<base-domain> -P 8443 \
      -u <registry-user> -p \"\$REGISTRY_PASSWORD\" \
      --ca-file /mnt/registry-ca.pem \
      --no-confirm
  "
unset REGISTRY_PASSWORD
```

공식 문서 확인 사항: 이 명령은 **ImageDigestMirrorSet(IDMS)**를 생성해 클러스터의 이미지 요청을 Mirror Registry로 리다이렉트합니다. 노드 재시작(MachineConfigPool 롤아웃)으로 30~60분 소요될 수 있습니다.

```bash
watch oc get mcp
oc get imagedigestmirrorset
oc get catalogsource -n openshift-marketplace
oc get packagemanifest -n openshift-marketplace | grep -i maximo
```

**검증 — Pull Secret / CA / 실제 이미지 Pull**

`configure-airgap`은 IDMS 외에 전역 Pull Secret과 CA 신뢰까지 구성하므로, 세 가지를 모두 확인합니다.

```bash
# 1) IDMS
oc get imagedigestmirrorset

# 2) 전역 Pull Secret에 미러 Registry 항목이 추가됐는지
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[]'

# 3) 사용자 CA 신뢰 번들
oc get configmap -n openshift-config user-ca-bundle -o yaml | head -20

# 4) MachineConfig 롤아웃 완료 대기 (30~60분 소요 가능)
watch oc get mcp
```

**실제로 미러 Registry에서 Pull이 되는지** 최종 확인합니다(공식 문서 권장 검증).

```bash
oc run test-pull --image=registry.<base-domain>:8443/ibmmas/cli:23.4.1 --restart=Never
oc get pod test-pull -w
oc delete pod test-pull
```

Pod가 `Completed`/`Running`까지 가면 성공이고, `ImagePullBackOff`면 IDMS·Pull Secret·CA 중 하나가 잘못된 것입니다.

> ⚠️ **IBM Maximo Operator Catalog(`packagemanifest`에 maximo 항목)는 이 시점에 없는 것이 정상**입니다 — §3.9의 `mas install`이 선택한 카탈로그 버전으로 CatalogSource를 생성하기 때문입니다. 이걸 진행 조건으로 삼으면 정상 설치도 막힙니다.

### 3.8 스토리지 준비 (RWO + RWX StorageClass) (`maximo` 계정)

🔴 **필수 단계 (공식 문서 원문)**: *"MAS requires both a `ReadWriteMany` and a `ReadWriteOnce` capable storage class"* — 두 StorageClass가 **`mas install` 실행 전에 이미 클러스터에 존재해야** 합니다. `mas install`은 스토리지를 만들어주지 않고, 기존 StorageClass 목록에서 선택하게만 합니다. 준비 안 하면 이 단계에서 막힙니다.

```bash
oc get storageclass
oc get csv -A | grep -iE 'lvm|odf|ocs'
```

| 용도 | 요구 AccessMode | 이번 배포 방식 | 주 사용처 |
|---|---|---|---|
| RWO | `ReadWriteOnce` | **LVM Storage(LVMS) Operator** — StorageClass 이름은 deviceClass가 `vg1`이면 `lvms-vg1` | Db2, MongoDB 등 |
| RWX | `ReadWriteMany` | **외부 NFS — 이번 배포는 Bastion을 NFS 서버로 겸용** (§3.8.2 방식 A) | MAS Core/Manage 공유 볼륨 |

⚠️ **가장 흔한 실수**: LVM Storage는 **RWO 전용**입니다. RWX 자리에 `lvms-vg1`을 넣으면 설치 도중 또는 런타임에 PVC 바인딩이 실패합니다. RWX는 반드시 별도로 준비하세요.

> **Db2는 반드시 RWO를 사용합니다.** NFS 위에 Db2 데이터를 두면 안 되므로, `mas install`에서 RWO/RWX를 헷갈리지 않도록 주의하세요.

#### 3.8.1 RWO — LVM Storage (LVMS) Operator

SNO의 로컬 디스크를 RWO StorageClass로 제공합니다. `lvms-operator`는 §2.3의 8번 미러링에 포함되어 있습니다(`imageset-ocp4.20.yml`의 `lvms-operator` 항목으로 확인).

**① Operator 조회 확인**

```bash
oc get packagemanifest -n openshift-marketplace | grep -i lvms
```

안 보이면 §3.7.1의 CatalogSource 적용 상태를 먼저 확인하세요.

**② 대상 디스크 확인** — LVMS가 사용할 **비어 있는(파티션·파일시스템 없는) 디스크**가 필요합니다.

```bash
oc debug node/<sno-node-name> -- chroot /host lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

⚠️ OS가 설치된 디스크는 쓸 수 없습니다. **데이터용 빈 디스크(예: `/dev/sdb`)가 SNO에 물려 있어야** 하며, 없으면 VM에 디스크를 추가한 뒤 진행하세요.

**③ Operator 설치** (Namespace / OperatorGroup / Subscription)

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-storage
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  targetNamespaces:
    - openshift-storage
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: lvms
  namespace: openshift-storage
spec:
  installPlanApproval: Automatic
  name: lvms-operator
  source: <catalogsource-name>
  sourceNamespace: openshift-marketplace
  channel: stable-4.20
EOF
```

> `source`에는 §3.7.1에서 적용된 실제 CatalogSource 이름을 넣습니다 — `oc get catalogsource -n openshift-marketplace`로 확인하세요(oc-mirror가 생성한 이름은 `cs-redhat-operator-index` 형태일 수 있습니다).

```bash
oc get csv -n openshift-storage -w     # Succeeded 대기
```

**④ LVMCluster 생성** — `deviceSelector.paths`에 ②에서 확인한 빈 디스크를 지정합니다.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: vg1
        default: true
        deviceSelector:
          paths:
            - /dev/sdb        # ②에서 확인한 실제 빈 디스크로 교체
        thinPoolConfig:
          name: thin-pool-1
          sizePercent: 90
          overprovisionRatio: 10
        fstype: xfs
EOF

oc get lvmcluster -n openshift-storage -w    # Ready 대기
```

**⑤ StorageClass 확인** — deviceClass 이름이 `vg1`이면 `lvms-vg1`이 생성됩니다.

```bash
oc get storageclass
oc get lvmcluster -n openshift-storage -o jsonpath='{.items[0].status}{"\n"}'
```

⚠️ **미검증**: 위 `Subscription`의 `channel`(`stable-4.20`)과 `LVMCluster` CR의 `apiVersion`(`lvm.topolvm.io/v1alpha1`)은 LVMS 버전에 따라 다를 수 있습니다. 적용 전 `oc get packagemanifest lvms-operator -n openshift-marketplace -o jsonpath='{.status.channels[*].name}'`로 채널을, `oc explain lvmcluster`로 API 버전을 확인하세요.

#### 3.8.2 RWX — 방식 선택

**⚠️ 두 방식 모두 폐쇄망에서는 추가 이미지 반입이 필요합니다. 인터넷 연결이 살아있는 동안(=§2 단계에서) 미리 확보해야 합니다.**

---

##### 방식 A — 외부 NFS (이번 배포 채택, Bastion을 NFS 서버로 겸용)

**장점**: SNO 리소스 부담이 거의 없고 구성이 단순합니다.
**⚠️ 운영 리스크**: Bastion이 **런타임 의존성**이 됩니다 — 설치 때만 쓰이는 게 아니라, Bastion(NFS 서버)이 내려가면 MAS의 RWX 볼륨이 끊깁니다. Bastion은 이미 Mirror Registry도 겸하고 있어 단일 장애점이 됩니다. 운영 환경에서는 별도 NFS 스토리지 장비를 권장합니다.

**A-1. Bastion에 NFS 서버 구성** (root 계정)

`nfs-utils`는 §3.1에서 이미 설치했습니다. 설치되어 있는지 먼저 확인합니다.

```bash
rpm -q nfs-utils || dnf --disablerepo='*' --enablerepo=mas-offline install -y nfs-utils

# 공유 디렉터리 생성 (충분한 용량의 디스크에 위치)
install -d -m 0777 /export/mas-rwx

# ⚠️ /etc/exports 를 덮어쓰지 않도록 별도 파일로 추가 (기존 export 보존)
cat > /etc/exports.d/mas-rwx.exports <<'EOF'
/export/mas-rwx 192.168.0.0/22(rw,sync,no_subtree_check,no_root_squash)
EOF

exportfs -rav
systemctl enable --now nfs-server

# 방화벽 (NFSv4는 2049만으로 충분, v3 호환 필요 시 rpc-bind/mountd 추가)
firewall-cmd --permanent --add-service=nfs
firewall-cmd --permanent --add-service=rpc-bind
firewall-cmd --permanent --add-service=mountd
firewall-cmd --reload

exportfs -s
showmount -e localhost
```

> `no_root_squash`는 컨테이너가 root로 파일을 쓸 수 있게 하려면 필요하지만 보안상 완화 설정입니다. 사이트 보안 정책에 따라 조정하고, 접근 대역(`192.168.0.0/22`)을 실제 필요한 범위로 좁히세요.

**A-2. OpenShift에 NFS 동적 프로비저너 배포**

`mas install`은 StorageClass 이름을 요구하므로 **동적 프로비저닝이 가능한 프로비저너가 필요**합니다(정적 PV만으로는 부족). 두 가지 선택지가 있고 **둘 다 커뮤니티 프로젝트(Red Hat 미지원)**입니다.

| 프로비저너 | 필요 이미지 | 특징 |
|---|---|---|
| `nfs-subdir-external-provisioner` | 1개 | 구성 단순, 볼륨 스냅샷 미지원. 이번 PoC에 적합 |
| `csi-driver-nfs` | 5~6개 (CSI sidecar 포함) | 최신 CSI 표준, 스냅샷 지원. 이미지 수가 많아 반입 부담 |

이번 배포는 **`nfs-subdir-external-provisioner`(이미지 1개)** 를 권장합니다. §2.3의 10번에서 받아둔 tar와 YAML을 사용합니다.

**① 프로비저너 이미지를 미러 Registry에 push** (`maximo` 계정)

```bash
podman load -i ~/mas-install/nfs-provisioner/nfs-subdir-external-provisioner.tar

read -s REGISTRY_PASSWORD
podman login registry.<base-domain>:8443 -u <registry-user> --password-stdin <<< "$REGISTRY_PASSWORD"

podman tag \
  registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2 \
  registry.<base-domain>:8443/sig-storage/nfs-subdir-external-provisioner:v4.0.2

podman push \
  registry.<base-domain>:8443/sig-storage/nfs-subdir-external-provisioner:v4.0.2

unset REGISTRY_PASSWORD
```

**② YAML 수정 후 배포**

✅ **검증 완료 (2026-07-30, 실제 받은 YAML 확인)**: 받은 파일의 기본값은 다음과 같습니다 — ServiceAccount·Deployment 이름 모두 `nfs-client-provisioner`, StorageClass 이름 `nfs-client`, provisioner `k8s-sigs.io/nfs-subdir-external-provisioner`, namespace는 `default`. **NFS 서버 주소와 경로가 `env`(NFS_SERVER/NFS_PATH)와 `volumes.nfs`(server/path) 총 4곳에 중복 기재**되어 있으므로 전부 바꿔야 합니다(아래 `sed`가 4곳을 한 번에 처리).

```bash
cd ~/mas-install/nfs-provisioner
oc create ns nfs-provisioner

# 1) namespace: default → nfs-provisioner (rbac.yaml 3곳, deployment.yaml 1곳)
sed -i 's/namespace: default/namespace: nfs-provisioner/g' rbac.yaml deployment.yaml

# 2) NFS 서버 IP: 기본값 10.3.243.101 → Bastion (env + volumes, 2곳)
sed -i 's/10\.3\.243\.101/192.168.2.210/g' deployment.yaml

# 3) NFS 경로: 기본값 /ifs/kubernetes → 실제 export 경로 (env + volumes, 2곳)
sed -i 's|/ifs/kubernetes|/export/mas-rwx|g' deployment.yaml

# 4) 이미지 경로: 미러 Registry로 교체
sed -i 's|registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2|registry.<base-domain>:8443/sig-storage/nfs-subdir-external-provisioner:v4.0.2|' deployment.yaml
```

수정 결과를 반드시 눈으로 확인합니다.

```bash
grep -nE 'namespace:|image:|NFS_SERVER|NFS_PATH|server:|path:|value:' deployment.yaml
grep -n 'namespace:' rbac.yaml
```

`192.168.2.210`이 2번, `/export/mas-rwx`가 2번, 이미지가 미러 Registry 경로로 바뀌었는지 확인한 뒤 적용합니다.

```bash
oc apply -f rbac.yaml
oc apply -f deployment.yaml
oc apply -f class.yaml
```

> `class.yaml`의 `archiveOnDelete: "false"`는 **PVC 삭제 시 NFS의 데이터도 함께 삭제**한다는 뜻입니다. 운영 환경에서 실수로 인한 데이터 유실을 막으려면 `"true"`로 바꿔 데이터를 보관(archive)하도록 하세요.

⚠️ **OpenShift 특이사항 (미검증)**: 이 프로비저너의 ServiceAccount는 기본 SCC로 NFS 마운트 권한이 부족할 수 있습니다. Pod가 `CrashLoopBackOff`/권한 오류를 내면 SCC를 부여합니다.

```bash
oc adm policy add-scc-to-user hostmount-anyuid \
  -z nfs-client-provisioner -n nfs-provisioner
```

**③ 배포 확인**

```bash
oc get pods -n nfs-provisioner
oc logs -n nfs-provisioner deploy/nfs-client-provisioner
oc get storageclass          # nfs-client 가 보여야 함
```

Pod가 `Running`이고 StorageClass `nfs-client`가 보이면 §3.8.3(내부 Image Registry) → §3.8.4(검증)로 진행합니다. 프로젝트가 제공하는 테스트 매니페스트로 곧바로 확인해볼 수도 있습니다.

```bash
oc apply -n nfs-provisioner -f test-claim.yaml -f test-pod.yaml
oc get pvc,pod -n nfs-provisioner
# 정상이면 Bastion에서 파일이 생성됐는지 직접 확인
ls -la /export/mas-rwx/
oc delete -n nfs-provisioner -f test-pod.yaml -f test-claim.yaml
```

⚠️ `test-pod.yaml`은 기본적으로 외부 이미지(`gcr.io/google_containers/busybox` 등)를 참조할 수 있어 폐쇄망에서 Pull이 실패할 수 있습니다. 그 경우 §3.8.4의 자체 테스트(미러 Registry 이미지 사용)를 쓰세요.

---

##### 방식 B — ODF (OpenShift Data Foundation) — 대안

**장점**: Red Hat 지원 제품이고 CephFS로 RWX를 제공하며, Bastion에 런타임 의존하지 않습니다.
**단점**: **SNO에서 리소스 요구량이 매우 큽니다**(CPU/메모리/추가 디스크). 단일 노드에 MAS + Db2 + ODF를 모두 올리는 구성은 사이징을 신중히 확인해야 합니다.

- **Operator 이미지**: §2.3의 8번(Red Hat 콘텐츠) 미러링에 이미 포함되어 있습니다 — `imageset-ocp4.20.yml`의 `odf-operator`, `ocs-operator`, `mcg-operator`, `rook-ceph-operator`, `cephcsi-operator` 등으로 확인됨.
- **⚠️ 추가 확인 필요**: MAS 쪽 ODF 관련 이미지가 별도로 필요하면 `mas mirror-images --mirror-odf`를 추가로 실행해야 합니다(§2.3의 9번). **이번 배포는 NFS로 결정했으므로 실행하지 않았습니다** — 나중에 ODF로 전환한다면 인터넷 연결 환경에서 이 미러링을 다시 수행해야 합니다.

배포 시:

```bash
oc get packagemanifest -n openshift-marketplace | grep -iE 'odf|ocs'
# ODF Operator 설치 → StorageSystem 생성 → StorageClass 확인
oc get storageclass | grep -i cephfs   # RWX용: ocs-storagecluster-cephfs
```

---

#### 3.8.3 OpenShift 내부 Image Registry 구성

🔴 **`platform: none`(베어메탈/UPI) 설치에서는 내부 Image Registry가 자동 구성되지 않습니다.** 기본값이 `managementState: Removed`이거나 스토리지가 없어 `Degraded` 상태로 남습니다.

**왜 필요한가**: Maximo Manage는 배포 과정에서 **Admin/Server Bundle 이미지를 직접 빌드해 레지스트리에 Push**합니다. 저장할 레지스트리가 없으면 **Manage 배포가 중단**됩니다. 따라서 `mas install` 전에 내부 Registry에 스토리지를 붙여 활성화해야 합니다.

```bash
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.managementState}{"\n"}{.spec.storage}{"\n"}'
oc get co image-registry
```

`Removed`이거나 스토리지가 비어 있으면 PVC를 붙여 `Managed`로 전환합니다. **RWX StorageClass(§3.8.2에서 만든 `nfs-client`)를 쓰면 향후 복제본 확장에도 안전**합니다.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 200Gi
  storageClassName: nfs-client
EOF

oc patch configs.imageregistry.operator.openshift.io cluster --type merge -p \
  '{"spec":{"managementState":"Managed","storage":{"pvc":{"claim":"image-registry-storage"}}}}'
```

활성화 확인:

```bash
oc get pvc -n openshift-image-registry
oc get pods -n openshift-image-registry
oc get co image-registry     # AVAILABLE=True, DEGRADED=False
```

⚠️ **미검증**: 용량 `200Gi`는 Manage 번들 이미지를 여유 있게 담기 위한 예시값입니다. 실제 필요량은 Manage 워크스페이스 수와 재빌드 이력에 따라 달라지므로 사이트 기준으로 조정하세요. 또 **Manage가 내부 Registry를 쓰는지, 아니면 별도 고객 Registry를 지정하는지는 `mas install`/Manage 배포 설정에 따라 달라질 수 있어** IBM 공식 Manage 배포 문서로 재확인이 필요합니다.

#### 3.8.4 검증 — 실제 PVC 바인딩 테스트

두 StorageClass가 모두 준비되면 실제로 바인딩되는지 검증합니다.

```bash
oc create ns storage-test

# RWO 테스트 (예: lvms-vg1)
cat <<'EOF' | oc apply -n storage-test -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-rwo
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
  storageClassName: <rwo-storage-class>
EOF

# RWX 테스트 (예: nfs-client 또는 ocs-storagecluster-cephfs)
cat <<'EOF' | oc apply -n storage-test -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-rwx
spec:
  accessModes: [ReadWriteMany]
  resources:
    requests:
      storage: 1Gi
  storageClassName: <rwx-storage-class>
EOF

oc get pvc -n storage-test
```

두 PVC 모두 `Bound` 상태여야 합니다. `Pending`이면 `oc describe pvc -n storage-test <name>`으로 원인을 확인하세요(프로비저너 미동작, SCC 권한, NFS export 권한 등).

**실제 쓰기 테스트**까지 하는 것을 권장합니다 — 특히 NFS는 마운트는 되지만 권한 때문에 쓰기가 실패하는 경우가 흔합니다.

```bash
cat <<'EOF' | oc apply -n storage-test -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-rwx-writer
spec:
  containers:
    - name: writer
      image: registry.<base-domain>:8443/openshift4/ose-tools-rhel9:latest
      command: ["/bin/sh","-c","echo ok > /data/test.txt && cat /data/test.txt && sleep 30"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: test-rwx
  restartPolicy: Never
EOF

oc logs -n storage-test test-rwx-writer -f
```

`ok`가 출력되면 정상입니다(이미지 경로는 미러 Registry에 실제로 있는 것으로 교체하세요). 확인 후 정리합니다.

```bash
oc delete ns storage-test
```

### 3.9 MAS Core 설치 (`maximo` 계정)

```bash
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)

podman run -ti --rm \
  -e IBM_ENTITLEMENT_KEY \
  -v "$HOME":/mnt/home:z \
  quay.io/ibmmas/cli:23.4.1 bash -c "
    oc login --token=<ocp-token> --server=https://api.<cluster-name>.<base-domain>:6443 &&
    mas install
  "
unset IBM_ENTITLEMENT_KEY
```

**사전 조건**: `oc login`에 사용하는 계정은 **`cluster-admin` 권한**이 있어야 합니다.

```bash
oc auth can-i '*' '*' --all-namespaces     # yes 여야 함
```

**대화형 입력 항목** (공식 install 가이드 흐름 순서 기준). 실제 프롬프트는 CLI 버전에 따라 순서·문구가 다를 수 있으니 화면을 읽으며 진행하세요.

| 구분 | 항목 | 이번 배포 값 / 판단 기준 |
|---|---|---|
| 클러스터 | 연결 방식 | 위에서 이미 `oc login` 완료 |
| 카탈로그 | Catalog Version | `v9-260625-amd64` |
| 카탈로그 | MAS Channel | `9.2.x` |
| 라이선스 | 라이선스 동의 | 동의 |
| 라이선스 | License File 경로 | `/mnt/home/mas-install/licenses/lincense_poc.dat` |
| 라이선스 | IBM Entitlement Key | 환경변수로 전달됨 |
| 스토리지 | Storage Class (RWO) | §3.8.1의 `lvms-vg1` |
| 스토리지 | Storage Class (RWX) | §3.8.2의 `nfs-client` — ⚠️ **RWO 클래스를 넣으면 실패** |
| 스토리지 | Pipeline Storage Class | MAS 설치 파이프라인(Tekton)용 — 보통 RWX 지정 |
| 인스턴스 | MAS Instance ID | 환경별 결정값 (소문자·짧게) |
| 인스턴스 | Workspace ID / Display Name | 환경별 결정값 |
| 인스턴스 | **Operational Mode** | 운영/비운영(production / non-production) — **라이선스 소비에 영향** |
| 계정 | **Superuser** 사용자명/비밀번호 | MAS 최초 관리자 계정 |
| 계정 | **담당자 정보**(Contact) | 이름/이메일 — SLS 등록 정보 |
| 도메인 | 기본 서브도메인 또는 커스텀 도메인 | `apps.<cluster-name>.<base-domain>` 기반 |
| 고급 | **Admin Mode** | 관리자 전용 접근 모드 여부 |
| 고급 | **Network Routing Mode** | Route 노출 방식 |
| 고급 | SSO / Guided Tours 등 | 환경 정책에 따라 |
| 앱 선택 | **설치할 애플리케이션** | **Manage 선택** (Assist/IoT/Monitor/Optimizer/Predict/VI 등은 선택하지 않음) |
| DB | 데이터베이스 | **내장 Db2 자동 프로비저닝 선택** |
| 기타 | Pod QoS 등 | 기본값 권장 |

클러스터에 airgap용 `ImageDigestMirrorSet`(이름 예: `mas-and-dependencies`)이 이미 있으면 `mas install`이 이를 감지해 폐쇄망 설치 흐름으로 자동 전환됩니다(인터넷에서 직접 받는 옵션은 제거됨).

> `mas install`은 마지막에 **동일 설치를 재현할 수 있는 비대화형 명령**을 출력합니다. 이후 재설치·다른 사이트 배포를 위해 **반드시 보관**하세요(비밀번호는 마스킹 후).

⚠️ **미검증**: 위 항목 목록은 공식 install 가이드의 흐름을 정리한 것으로, CLI 23.4.1의 실제 프롬프트와 순서·명칭이 다를 수 있습니다. 특히 `Pipeline Storage`, `Operational Mode`의 정확한 선택지는 실행 화면에서 확인하세요.

### 3.10 Manage + Maximo IT 배포·활성화 (웹 UI, Suite Administration 관리자 계정)

⚠️ **이 절은 검증 수준이 가장 낮습니다.** IBM 지식센터 원문을 이 문서 작성 환경에서 직접 열지 못해 검색 스니펫 기준으로 작성했습니다. **진행 전 반드시 아래 공식 문서를 브라우저로 직접 열어 대조하세요.**
>
> - Maximo IT 배포: <https://www.ibm.com/docs/en/max-it/cd.0.0_cd?topic=it-deploying-maximo-maximo-manage>
> - Maximo IT 라이선스: <https://www.ibm.com/docs/en/max-it/cd.0.0_cd?topic=suite-licensing-maximo-it-in-maximo-application>

**기본 흐름**

1. Suite Administration(`https://admin.<mas-instance-id>.apps.<cluster-name>.<base-domain>`)에 §3.9의 Superuser로 로그인
2. Catalog에서 **Manage** 선택
3. **버전 선택** — ⚠️ 아래 "버전 호환성" 참고
4. Workspace **Components**에서 **Maximo IT(ICD) add-on** 활성화
5. **데이터베이스 구성** — JDBC 연결, schema, tablespace (⚠️ 아래 "한국어 사용 시" 참고)
6. **언어 설정** — 기본 언어 및 추가 언어 선택 (⚠️ 설치 후 변경이 어려우므로 이 단계에서 확정)
7. **Server Bundle 구성** — 워크로드 분리 방식(`all`, `snd` 등) 선택
8. **Activate Manage** 실행 → 이미지 빌드·배포 진행 (내부 Image Registry 사용, §3.8.3)
9. 활성화 완료 후 **관리자 권한 동기화 및 재로그인**
10. Maximo IT 애플리케이션 접근 확인

**⚠️ 반드시 확인할 항목들**

| 항목 | 내용 |
|---|---|
| **버전 호환성** | Manage 버전과 Maximo IT(ICD) 버전은 **호환되는 조합**이어야 합니다. `latest`와 특정 버전을 **혼용하지 마세요** — 한쪽만 `latest`로 두면 이후 업데이트에서 조합이 깨집니다. 양쪽 모두 명시적 버전으로 고정하는 것을 권장합니다. |
| **한국어 사용 시 Db2 VARGRAPHIC** | 🔴 한국어 등 멀티바이트 언어를 사용하려면 Db2에서 **VARGRAPHIC** 데이터 타입 설정이 필요합니다. 이 설정은 **DB 생성/Manage 활성화 시점에 결정**되며 나중에 바꾸기 매우 어렵습니다. 한국어 사용이 확정이면 **활성화 전에** Manage DB 설정에서 반드시 지정하세요. |
| **언어와 Server Bundle** | 추가 언어를 넣으면 DB 크기와 활성화 시간이 늘어납니다. Server Bundle 구성(`all` vs 분리)은 SNO 리소스를 고려해 결정하세요. |
| **관리자 권한 동기화** | 활성화 직후에는 Maximo IT 메뉴가 보이지 않을 수 있습니다. 권한 동기화 후 **로그아웃 → 재로그인**이 필요합니다. |
| **활성화 후 설정** | 활성화 완료가 끝이 아닙니다 — 사용자/그룹 권한, AppPoints 할당, IT 관련 초기 설정이 이어집니다. |

⚠️ **미검증**: 위 표의 세부 사항(특히 VARGRAPHIC 지정 위치, Server Bundle 선택지 명칭, 권한 동기화 절차)은 공식 문서 원문으로 확정하지 못했습니다. 실제 진행 시 화면과 위 공식 문서를 대조하며 수행하고, 확인된 내용으로 이 절을 갱신하세요.

**라이선스 — ⚠️ 확인 필요**: IBM 공식 문서 기준으로 Maximo IT는 **MAS Entitlement와 별개로 Maximo IT 자체 구매·Entitlement가 모두 필요**합니다. 라이선스 파일(`lincense_poc.dat`)을 보유하고 있다는 사실만으로는 **그 안에 Maximo IT AppPoints가 포함되어 있다는 보장이 되지 않습니다.**

배포 전 아래를 확인하세요.

- IBM License Key Center에서 발급한 라이선스에 **Maximo IT용 AppPoints 항목이 실제로 포함**되어 있는지
- 구매 계약에 Maximo IT 권한이 있는지
- 설치 후 SLS(Suite License Service)에서 IT AppPoints가 인식되는지 (§3.11 체크리스트)

AppPoints가 없으면 Manage 배포는 되더라도 **Maximo IT add-on 활성화 또는 사용자 할당 단계에서 막힙니다.**

### 3.11 설치 후 확인 (`maximo` 계정)

```bash
oc get nodes
oc get co
oc get mcp
oc get suite -A
oc get manageapp -A
oc get manageworkspace -A
oc get subscriptions -A
oc get catalogsource -n openshift-marketplace
oc get imagedigestmirrorset
oc get pods -A | grep -v Running | grep -v Completed | grep -v Succeeded
oc get route -A | grep -E 'mas|manage|admin'
```

체크리스트:
- [ ] OCP 노드가 외부 Registry에 접근하지 않고 Mirror Registry에서만 이미지 Pull
- [ ] MAS Core, Manage Operator 정상(Ready)
- [ ] Suite Administration에서 Manage + Maximo IT add-on이 Ready/Active
- [ ] Maximo IT 메뉴/애플리케이션 접근 가능
- [ ] AppPoints 소비 정상 확인

---

## 4. 검증 상태 요약

### 4.1 ⚠️ 아직 열린 항목 (실행 전 반드시 확인)

**A. 실행하면서 즉시 확인 가능한 것 (`--help` / 실제 명령)**

| # | 항목 | 확인 방법 | 관련 절 |
|---|---|---|---|
| A1 | `mas configure-airgap --setup-redhat-catalogs` 실존 여부 — **IBM 설치 가이드와 커맨드 레퍼런스가 엇갈림** | `mas configure-airgap --help` | §3.7.1 |
| A2 | `install-config.yaml`의 필드명이 `imageDigestSources`인지 `imageContentSources`인지 | `openshift-install explain installconfig \| grep -i image` | §3.6 |
| A3 | `mirror-registry install`의 `--quayStorage`/`--sqliteStorage` 플래그명 (구버전은 `--pgStorage`) | `./mirror-registry install --help` | §3.4 |
| A4 | oc-mirror 산출 `cluster-resources` 실제 경로·파일명 (IDMS source/mirror 값 출처) | `find .../working-dir -name "*idms*"` | §3.6, §3.7.1 |
| A5 | LVMS `Subscription` 채널명과 `LVMCluster` API 버전 | `oc get packagemanifest lvms-operator -o jsonpath='{.status.channels[*].name}'`, `oc explain lvmcluster` | §3.8.1 |
| A6 | MongoDB 버전 — `--mirror-mongo` 기본값 vs `--mirror-mongo-v7` 중 카탈로그 `v9-260625-amd64`에 맞는 것 | `mas mirror-images` 대화형 모드로 확인 | §2.3 |
| A7 | NFS 프로비저너에 SCC(`hostmount-anyuid`) 부여가 실제로 필요한지 | 배포 후 Pod 상태 | §3.8.2 |

**B. 공식 문서 원문을 브라우저로 직접 확인해야 하는 것**

| # | 항목 | 비고 |
|---|---|---|
| B1 | **Manage + Maximo IT 배포 절차 전반** (§3.10) — 이 문서에서 검증 수준이 가장 낮음 | 버전 호환성, 한국어 **Db2 VARGRAPHIC**, 언어/Server Bundle, 권한 동기화 |
| B2 | **Maximo IT AppPoints 권한** — 라이선스 파일 보유가 IT 권한 보유를 의미하지 않음 | MAS Entitlement와 Maximo IT Entitlement가 **각각** 필요 |
| B3 | Manage가 내부 Image Registry를 쓰는지, 별도 Registry 지정이 가능한지 | §3.8.3의 용량 산정에 영향 |
| B4 | SNO에서 PTR(역방향) DNS가 필수인지 권장인지 | §3.2 |

**C. 구조적·운영 리스크 (기술 오류는 아니지만 의사결정 필요)**

| # | 항목 |
|---|---|
| C1 | **`mirror registry for Red Hat OpenShift`는 OCP 설치용 소규모·비HA Registry** — MAS 전체 이미지의 장기 운영 Registry로 쓰는 것은 지원 범위를 벗어남. 현 구성은 PoC 용도로 보고 운영 전환 시 Quay/Harbor 등 운영급 Registry 검토 |
| C2 | **Bastion이 Mirror Registry + DNS + NTP + NFS를 모두 겸함** → 단일 장애점이며 MAS 런타임이 Bastion 가동에 의존. 운영 전환 시 NFS·Registry 분리 검토 |
| C3 | **NFS 동적 프로비저너는 커뮤니티 프로젝트** — Red Hat/IBM 공식 지원 대상 아님 |
| C4 | 사이트 Bastion 디스크 — 실측 기준 **500GB 이상, 안전하게 1TB 권장**(§2.1). 반입 데이터 + Registry 사본을 동시 보관하므로 미러 총량의 약 2배 필요 |

**D. 절차는 추가했으나 아직 실제 실행 검증은 안 된 것**

§3.2(DNS·NTP), §3.5(Registry Push), §3.6(SNO 설치·Pull Secret 병합), §3.7(CatalogSource·airgap), §3.8(스토리지·내부 Registry) — 모두 공식 문서 근거로 작성했으나 **이 배포에서 아직 실행하지 않았습니다.** 실행하며 확인된 내용으로 갱신하세요.

### 4.2 ✅ 검증 완료된 항목

| 항목 | 결과 | 근거 |
|---|---|---|
| `oc-mirror`를 별도로 준비해야 하는가 | **불필요** — `mas mirror-redhat-images`가 내부적으로 `oc mirror --v2`를 호출하는 래퍼이며 `oc`/`oc-mirror`는 MAS CLI 이미지에 내장 | 실행 로그 (§1.2) |
| CLI 플래그 철자 | `--pullsecret`(o), `--release`(o, `--ocp-release` 아님) | `--help` 출력 |
| `--mirror-icd`의 의미 | **Maximo IT 자체** (MongoDB 아님 — MongoDB는 `--mirror-mongo`) | `--help` 원문 |
| `to-filesystem` 모드의 `-H/-P/-u/-p` | **필수** (없으면 `--help`만 출력하고 종료) | 실행 결과 (§2.3) |
| `to-filesystem`에 더미 Registry를 쓰면 산출물이 오염되는가 | **오염되지 않음, 재미러링 불필요** — 해당 manifest 목적지가 `file:///`이고 디스크 레이아웃도 경로 기반 | 실측 (§5.3) |
| `from-filesystem`의 `-c`/`-C` 필요 여부 | **필요** (`--release` 계열은 `to-filesystem` 전용) | GitHub 원본 |
| MAS 스토리지 요구사항 | **RWO와 RWX 각각 필요** ("both a ReadWriteMany and a ReadWriteOnce capable storage class") — "둘 다 지원하는 클래스 하나"는 오류였음 | 공식 install 가이드 |
| `mas configure-airgap`의 역할 | Registry 검증 + 전역 Pull Secret + IDMS + CA 신뢰 + MCP 롤아웃. **Red Hat Operator CatalogSource는 생성하지 않음** | 공식 configure-airgap 가이드 |
| DNS 요구사항 | `api.*`, `api-int.*`(api와 동일 IP), `*.apps.*` 필수 — **누락 시 SNO 설치가 완료되지 않음** | Red Hat Agent-based Installer 문서 |
| 실측 용량 | Stage1 15GB / Stage2 25GB / Stage4 43GB (참고치와 항목별로 크게 다름 — 외삽 불가) | 직접 측정 (§2.1) |
| `:Z` vs `:z` SELinux | 병렬 실행 시 **`:z` 필수** | AVC 로그 (§5.1) |
| 미러링 재시도 전략 | 네트워크 오류면 **비우지 말고 재실행**이 빠름 | 실측 (§5.2) |
| NFS 프로비저너 자산 | 이미지 `v4.0.2` 유효(`linux/amd64`, 43MB), SA·Deployment 이름 `nfs-client-provisioner`, StorageClass `nfs-client`, NFS 주소/경로가 **4곳**에 중복 | 실제 다운로드·확인 |

## 5. 실제 실행 중 발견된 이슈와 해결 (트러블슈팅 기록)

### 5.1 `:Z` vs `:z` SELinux 라벨 충돌 (확인됨, 2026-07-29)

**증상**: §2.3의 8번(Red Hat 콘텐츠 미러링)과 9번(MAS 콘텐츠 미러링)을 여러 터미널에서 동시에 실행하면, 둘 다 또는 하나가 로그 파일 쓰기 단계에서 `Permission denied`로 실패한다.

**원인**: 두 컨테이너가 같은 상위 디렉터리(`$LOCAL_DIR` = `~/mas-install/mirror`)를 각자 `-v ...:Z`(대문자, 해당 컨테이너 전용 배타적 SELinux 라벨)로 마운트했기 때문. `:Z`는 마운트 시 전체 트리를 그 컨테이너만 접근 가능한 고유 MCS 카테고리로 재라벨링하므로, 두 번째 컨테이너가 시작되며 같은 트리를 자기 것으로 다시 라벨링하면 첫 번째 컨테이너의 접근 권한이 무효화된다.

**확인 방법**:
```bash
getenforce  # Enforcing 확인
sudo grep avc /var/log/audit/audit.log | tail -30
```
실제로 다음과 같은 AVC 거부 로그로 확정했다.
```
avc: denied { write } for ... path="/mnt/registry/core/logs/mirror-...-core.log" ...
  scontext=system_u:system_r:container_t:s0:c280,c427
  tcontext=system_u:object_r:container_file_t:s0:c285,c450
```
(scontext와 tcontext의 MCS 카테고리 번호가 다름 — 서로 다른 컨테이너가 부여한 라벨이 충돌한 증거)

**해결**: 여러 컨테이너가 같은 호스트 디렉터리를 동시에 접근할 때는 `:Z` 대신 **`:z`(소문자, 공유 라벨)**를 사용한다. 이 문서의 모든 `podman run` 명령은 이미 `:z`로 수정되어 있다. `:z`는 여러 컨테이너가 같은 콘텐츠를 공유해서 쓸 수 있게 라벨링하므로 병렬 실행에 안전하다.

**교훈**: §2.3의 8번과 9번을 병렬로 실행할 계획이라면 반드시 `:z`를 사용해야 하며, 만약 순차 실행(하나씩만 실행)한다면 `:Z`/`:z` 어느 쪽을 써도 문제되지 않는다.

### 5.2 Stage 2(Manage+Maximo IT) 미러링 중 `cp.icr.io` OAuth 토큰 타임아웃 (확인됨, 2026-07-29)

**증상**: `mas mirror-images --mirror-manage --mirror-icd` 실행 1시간 17분 만에(23GB 수신) `ibm-mas-manage : Fail if mirror is not successful` 태스크에서 실패.

**원인**: ansible 로그의 "Debug mirror command" 태스크 출력 마지막 부분에서 실제 에러 확인.
```
error: unable to push cp.icr.io/cp/manage/aip-optimizer: failed to retrieve blob ...:
Get "https://cp.icr.io/oauth/token?...": net/http: request canceled (Client.Timeout exceeded while awaiting headers)
info: Mirroring completed in 1h17m50.61s (5.088MB/s)
error: one or more errors occurred while uploading images
```
거의 다 받은 상태(23GB)에서 IBM 레지스트리(`cp.icr.io`)의 OAuth 토큰 발급 요청이 일시적으로 타임아웃난 것 — 네트워크 일시 문제로 추정. `oc image mirror`는 이미지 하나라도 실패하면 `mirror_images` 역할이 전체를 실패로 처리한다(oc-mirror v2와 달리 부분 실패를 넘어가지 않음).

**해결**: 이미 받은 23GB는 로컬 파일(`~/mas-install/mirror/apps/v2`)에 그대로 남아있고, `oc image mirror`는 재실행 시 이미 있는 콘텐츠를 다시 받지 않고 건너뛴다. 따라서 **디렉터리를 비우지 말고 동일한 Stage 2 명령을 그대로 재실행**하면 실패했던 이미지만 다시 받아 훨씬 빨리 끝난다.

**교훈**: `mas mirror-images` 실패 시 항상 디렉터리를 비우고 처음부터 재시도할 필요는 없다 — 로그에서 실패 원인이 SELinux/권한 문제(§5.1)인지 단순 네트워크 타임아웃인지 먼저 확인하고, 후자라면 비우지 않고 재시도하는 것이 훨씬 빠르다.

### 5.3 `to-filesystem`에 더미 Registry 주소를 쓴 것이 산출물에 영향을 주는가 (검증됨, 2026-07-30)

**배경**: §2.3의 9번에서 `-H/-P/-u/-p`가 `to-filesystem` 모드에서도 필수임을 발견하고, "이 단계는 로컬 파일로 저장하는 것이니 접속 정보는 안 쓰일 것"이라 **추론**해 더미값(`localhost`/`443`/`dummy`/`dummy`)으로 4개 Stage(약 85GB)를 모두 미러링했다. 이후 "Registry Host/Port는 목적지 이미지 경로 생성에도 사용되므로 더미값이 산출물을 오염시켰을 수 있다"는 지적을 받아 실증 검증을 수행했다.

**검증 결과 — 산출물은 오염되지 않았다. 재미러링 불필요.**

`mas mirror-images`는 모드별로 manifest를 3개 생성하고, 목적지 표기가 서로 다르다.

| manifest | 목적지 표기 | 비고 |
|---|---|---|
| `manifests/direct/` | `localhost:443/cp/mas/...` | direct 모드용, 미사용 |
| `manifests/to-filesystem/` | **`file:///cp/mas/...`** | 실제 사용 — **레지스트리 주소가 들어가지 않음** |
| `manifests/from-filesystem/` | `localhost:443/...` | §3.5에서 재생성됨 |

```bash
# 실제로 사용된 to-filesystem manifest — 목적지가 file:///
head -3 ~/mas-install/mirror/core/manifests/to-filesystem/ibm-mas_9.2.0.txt
#   cp.icr.io/cp/mas/accapppoints@sha256:...=file:///cp/mas/accapppoints:3.11.74-amd64

# 디스크 레이아웃도 경로 기반 — localhost:443 디렉터리가 없음
ls ~/mas-install/mirror/core/v2/       # → cp, cpopen
```

**원인 분석**: `to-filesystem` 모드에서는 목적지가 `file:///<repo-path>` 형태라 레지스트리 호스트명이 개입하지 않으며, 디스크의 v2 레이아웃도 repo 경로(`cp/mas/...`)로만 키가 잡힌다. `-H/-P`는 인자 검증(파싱) 통과에는 필요하지만 `to-filesystem` 산출물 경로에는 반영되지 않는다. 또한 manifest 파일들은 매 실행 시 CASE 번들에서 재생성되므로(ansible 태스크 `Generate the mirror manifest from the CASE bundle`, `Copy images-mapping-from-filesystem`), `from-filesystem` 실행 시 그때 지정한 실제 Registry 주소로 다시 만들어진다.

**교훈 및 권장**:
- 결과적으로 문제는 없었지만, **처음부터 실제 Registry 주소를 넣는 것이 공식 의도**다(공식 가이드는 Stage 0에서 `REGISTRY_HOST` 등을 export해 모든 Stage가 상속하게 함). 이 문서의 명령은 실제 값을 쓰도록 수정했다.
- 더미값으로 이미 미러링한 경우라도 **§3.5 첫 Stage 실행 직후 `manifests/from-filesystem/`에 `localhost`가 남아있지 않은지 반드시 확인**하라(§3.5의 확인 절차 참고).
- 일반 교훈: "안 쓰일 것"이라는 추론으로 대량 작업을 시작하지 말고, 소량으로 먼저 산출물을 검증한 뒤 진행하라.
