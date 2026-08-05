# Maximo IT 오프라인(Airgap) 설치 가이드

IBM MAS CLI 공식 문서(`ibm-mas.github.io/cli`)와 Red Hat OpenShift 공식 문서를 근거로 작성한 Maximo IT 오프라인 설치 가이드입니다.

---

## 1. 설치 요약 및 흐름

### 1.1 설치 구성

| 항목 | 값 |
|---|---|
| Red Hat OpenShift | `4.20.30` (SNO, Single Node OpenShift) |
| IBM Maximo Operator Catalog | `v9-260625-amd64` (지원 범위 OCP 4.16–4.21) |
| MAS CLI 이미지 | `quay.io/ibmmas/cli:23.4.1` |
| 설치 대상 | MAS Core + Manage + **Maximo IT**(Manage의 add-on) |
| 미사용 대상 | Cloud Pak for Data, Visual Inspection, Assist, IoT, Monitor, Optimizer 등 |
| 데이터베이스 | **내부(in-cluster) Db2** |
| 스토리지 | RWO = **LVM Storage**(`lvms-vg1`) / RWX = **외부 NFS — Bastion 겸용** (§3.8) |

### 1.2 필요 자원

**1) 인터넷 연결 RHEL 서버**

인터넷이 연결되어 있는 RHEL 서버에서 설치에 필요한 파일 다운로드 및 이미지를 생성합니다.

| 항목 | 사양 |
|---|---|
| OS | RHEL 9.6 / x86_64 |
| CPU | 4 Core 이상 |
| Memory | 8 GB 이상 |
| Disk | 🔴 **1 TB 이상** — 피크 약 760GB (§2.1의 순서 준수 시) |
| 네트워크 | 인터넷 연결 (`registry.redhat.io`, `cp.icr.io`, `quay.io`, `mirror.openshift.com`) |

**2) Bastion 서버**

설치 파일 및 이미지 반입 후 설치를 진행합니다. Mirror Registry, DNS, NTP, NFS, 설치 CLI를 모두 겸합니다.

| 항목 | 사양 |
|---|---|
| OS | RHEL 9.6 / x86_64 |
| CPU | **8 Core** (Mirror Registry 공식 최소는 2 vCPU이나 Push 구간이 느림) |
| Memory | **최소 16 GB / 권장 32 GB** (Mirror Registry 단독 최소가 8GB) |
| Disk | 🔴 **1.5 TB 이상** — 반입 데이터 500GB + 설치 중 생성 1TB (§2.1) |
| 디스크 구성 | Registry 저장소(`~/mirror-registry`)를 **별도 디스크**로 분리 권장 |
| 네트워크 | 폐쇄망. SNO와 동일 대역 |

**3) Single Node OpenShift (SNO)**

실제 MAS를 구동하기 위한 자원입니다.

| 항목 | 사양 |
|---|---|
| OS | RHCOS (OpenShift 4.20.30) |
| CPU | 16 Core 이상 |
| Memory | 64 GB 이상 |
| Disk | **2개 필요** — 300 GB(OS) + 500 GB |
| 500GB 디스크 | 🔴 **빈 상태로 유지** — LVM Storage가 RWO StorageClass로 사용. 파티션·파일시스템이 있으면 사용 불가 |
| 네트워크 | 고정 IP, 폐쇄망 |

### 1.3 설치 흐름

```text
[인터넷 연결 RHEL 서버]
  - 라이선스·Pull Secret·설치 도구·RPM 확보
  - Red Hat 콘텐츠 미러링   : oc mirror         (to-filesystem)
  - MAS 콘텐츠 미러링       : mas mirror-images (to-filesystem)
        |
        v  전송매체로 반입
[사이트 Bastion]
  - DNS(dnsmasq) · NTP(chrony)
  - Mirror Registry 구축 (별도 디스크)
  - 반입 이미지를 Registry로 Push (from-filesystem)
  - SNO 설치 ISO 생성
  - NFS 서버 (RWX 제공)
  - mas install 실행
        |
        v
[SNO 노드]
  - OpenShift 4.20.30 (Agent-based Installer)
  - 디스크 2개: 300GB(OS) + 500GB(LVMS용 — 반드시 빈 상태)
  - MAS Core + Manage + Maximo IT
```

### 1.4 환경 정보

| 항목 | 설명 | 값 |
|---|---|---|
| cluster-name | OCP 클러스터 이름. 모든 도메인의 첫 단계 | `mas-it` |
| base-domain | 클러스터 상위 도메인 | `itmsg.co.kr` |
| Bastion IP | DNS·NTP·Registry·NFS 제공 서버 | `192.168.2.210` |
| SNO IP | `api`·`api-int`·`*.apps`가 가리키는 주소 | `192.168.2.211` |
| 상위 DNS IP | dnsmasq가 외부 이름을 넘길 대상. 폐쇄망이면 주석 처리 | `8.8.8.8` |
| 네트워크 대역 | `machineNetwork`, NTP `allow` 범위 | `192.168.0.0/22` |
| Gateway | SNO 기본 게이트웨이 | `192.168.1.1` |
| SNO 호스트명 | `agent-config.yaml`의 `hostname` | `mas-it-sno` |
| SNO NIC MAC | Agent Installer가 NIC를 식별하는 값 | `00:50:56:bb:86:df` |
| Registry 관리 계정 | 이미지 Push용. §3.4 설치 시 발급 | `masadmin` |
| Registry 읽기전용 계정 | Pull Secret에 담겨 **모든 OCP 노드에 배포**되므로 반드시 읽기 전용 | `maspull` |
| MAS Instance ID | MAS 인스턴스 식별자. Admin URL에 포함 | `inst1` |
| Workspace ID | Manage 워크스페이스 식별자 | `ws1` |
| 작업 계정 | 설치 작업 계정. 홈은 `~/mas-install` | `maximo` |

### 1.5 참고한 공식 문서

- [IBM MAS CLI](https://ibm-mas.github.io/cli/)
- [IBM MAS CLI Catalogs](https://ibm-mas.github.io/cli/catalogs/)
- [Image Mirroring (IBM)](https://ibm-mas.github.io/cli/guides/image-mirroring/) / [원본](https://github.com/ibm-mas/cli/blob/master/docs/guides/image-mirroring.md)
- [Image Mirroring (Red Hat) 커맨드 레퍼런스](https://ibm-mas.github.io/cli/commands/mirror-redhat-images/) / [원본](https://github.com/ibm-mas/cli/blob/master/docs/commands/mirror-redhat-images.md)
- [Airgap Configuration](https://ibm-mas.github.io/cli/guides/configure-airgap/) / [원본](https://github.com/ibm-mas/cli/blob/master/docs/guides/configure-airgap.md)
- [MAS CLI Topology Reference](https://ibm-mas.github.io/cli/reference/topology/)
- IBM Maximo IT 설치: `ibm.com/docs/en/masv-and-l/max-it` 계열 (WebFetch 403 — 검색 스니펫으로만 확인, ⚠️ 원문 직접 확인 필요)

---

## 2. 사전준비 사항 (설치 파일 및 이미지 생성)

Bastion 서버로 옮기기 위한 설치 파일 및 이미지를 확보하기 위해서 인터넷이 연결되어 있는 서버에서 준비합니다. 별도 표시가 없으면 사용자 계정으로 실행합니다.

### 2.1 저장공간

실측값입니다.

**반입 대상 — 489GB**

| 항목 | 용량 |
|---|---|
| `mirror/redhat` — `mirror_000001.tar` 373GB + `working-dir` 25GB | **398GB** |
| `mirror/core` (Stage 1 Core+Catalog) | 16GB |
| `mirror/apps` (Stage 2 Manage+Maximo IT) | 25GB |
| `mirror/other` (Stage 4 Mongo/TSM/SLS/CFS/Db2) | 43GB |
| `mirror/cli` (Stage 5) | 4.9GB |
| `mas-cli`, `registry`, `ocp`, `rpms`, `nfs-provisioner`, `licenses` | 약 2.8GB |
| **합계** | **약 489GB** |

🔴 **Red Hat 콘텐츠가 공식 참고치(약 80GB)의 5배입니다.** `rhods-operator`(OpenShift AI)의 PyTorch·CUDA·vLLM 이미지가 대부분입니다. IBM imageset에서 *Optional Packages* 로 분류돼 있고 MAS는 사용하지 않으므로, 용량이 부담되면 `imageset-ocp4.20.yml`에서 제거할 수 있습니다(⚠️ 제거 후 용량은 미측정).

**인터넷 연결 서버 — 1TB 이상**

`oc mirror`는 캐시에 모은 뒤 tar로 내보내므로 **둘을 동시에 보관**합니다. 그래서 §2.3의 **8번(Red Hat)을 먼저 끝내고 캐시를 삭제한 뒤 9번(MAS)** 을 진행해야 피크가 낮아집니다.

| 시점 | 사용량 |
|---|---|
| 8번 진행 중 (캐시 355GB + 산출물 398GB) | **약 760GB** ← 피크 |
| 캐시 삭제 후 | 398GB |
| 9번 완료 | 489GB |

⚠️ 순서를 바꿔 9번을 먼저 하면 피크가 **841GB**가 됩니다(실측). 디스크 사용률이 90%에 가까워지면 캐시 쓰기가 지연되어 §4.3의 타임아웃이 재발합니다.

**사이트 Bastion — 1.5TB 이상**

| 구분 | 내용 | 용량 |
|---|---|---|
| 반입 데이터 | `~/mas-install` | **500GB** |
| 설치 중 생성 | Registry 사본 489GB + NFS `/export/mas-rwx` + OS·여유 | **1TB** |
| **합계** | | **1.5TB** |

Registry 사본은 §3.5 push 때 `~/mirror-registry/quay`에 만들어집니다 — 반입 파일과 별개로 같은 양이 한 번 더 쌓입니다.

**1) 확인 및 검증**

```bash
du -sh ~/mas-install/mirror/*
df -h /home
```

### 2.2 인터넷 연결 서버에 도구 설치 (root 계정)

§2 작업에 필요한 도구를 설치합니다. 사이트 Bastion용 패키지는 §2.3의 4번에서 RPM으로 확보하므로 **여기서 미리 설치하지 마세요.**

**1) 실행**

```bash
dnf install -y podman curl tar rsync dnf-plugins-core createrepo_c
```

**2) 확인 및 검증**

```bash
cat /etc/redhat-release          # RHEL 9.6
uname -m                         # x86_64
timedatectl status               # 시간 동기화 정상
dnf repolist --enabled           # appstream, baseos 활성
podman --version
```

### 2.3 확보해야 할 항목

| # | 항목 | 저장 위치 | 확보 방법 |
|---|---|---|---|
| 1 | **IBM Entitlement Key** | `~/mas-install/licenses/entitlement_key.key` | [IBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary)에서 문자열로 수동 발급 |
| 2 | **MAS 라이선스** (AppPoints) | `~/mas-install/licenses/<license>.dat` | IBM License Key Center에서 수동 발급 (⚠️ Maximo IT AppPoints 포함 여부 확인 필요) |
| 3 | **Red Hat Pull Secret** | `~/mas-install/redhat/pull-secret.json` | [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/install/pull-secret)에서 수동 다운로드 |
| 4 | **사이트 Bastion용 RPM 오프라인 저장소** | `~/mas-install/rpms/` | 이 서버의 `dnf` 저장소 (아래 명령) |
| 5 | **OpenShift Client / Installer / checksum** | `~/mas-install/ocp/` | `mirror.openshift.com` (아래 명령) |
| 6 | **MAS CLI 이미지** (`quay.io/ibmmas/cli:23.4.1`) | `~/mas-install/mas-cli/mas-cli-23.4.1.tar` | `quay.io`에서 pull + save |
| 7 | **Mirror Registry 설치 패키지** | `~/mas-install/registry/mirror-registry-amd64.tar.gz` | Red Hat Downloads 포털 |
| 8 | **Red Hat 콘텐츠 이미지 세트** (OCP 플랫폼 + Operator) | `~/mas-install/mirror/redhat/mirror_000001.tar` | imageset 생성 후 `oc mirror` 직접 실행 — **6시간 이상** |
| 9 | **MAS 콘텐츠 이미지 세트** (Core/Catalog, Manage+Maximo IT, 공통의존성+Db2, CLI) | `~/mas-install/mirror/{core,apps,other,cli}/` | `mas mirror-images` Stage 1/2/4/5 |
| 10 | **NFS 동적 프로비저너 이미지** (RWX용) | `~/mas-install/nfs-provisioner/` | `podman pull` + `save` — ⚠️ MAS/Red Hat 미러링에 **포함되지 않음** |

#### 0. 디렉터리 생성 + linger 활성화

다른 항목보다 먼저 실행합니다.

🔴 **`linger`를 켜지 않으면 SSH 접속을 끊는 순간 미러링 컨테이너가 죽습니다.** `nohup`/`disown`은 SIGHUP만 막고, `systemd-logind`가 세션 종료 시 사용자 프로세스를 정리하는 것은 막지 못합니다. 8·9번이 수 시간~수십 시간 걸리므로 반드시 먼저 켜세요.

**1) 실행**

```bash
mkdir -p ~/mas-install/{licenses,redhat,rpms,ocp,mas-cli,registry,nfs-provisioner,mirror}

sudo loginctl enable-linger maximo
```

**2) 확인 및 검증**

```bash
ls ~/mas-install
loginctl show-user maximo | grep -i linger      # Linger=yes 여야 함
```

#### 1~3. IBM Entitlement Key / MAS 라이선스 / Red Hat Pull Secret

자동 다운로드 명령이 없습니다. 위 표의 링크에서 브라우저로 발급받아 해당 경로에 저장합니다.

**1) 실행** — 발급받은 파일을 저장한 뒤 권한을 제한합니다.

```bash
chmod 600 ~/mas-install/licenses/* ~/mas-install/redhat/pull-secret.json
```

**2) 확인 및 검증**

```bash
ls -l ~/mas-install/licenses/ ~/mas-install/redhat/
head -c 60 ~/mas-install/redhat/pull-secret.json    # {"auths":{... 로 시작
```

#### 4. 사이트 Bastion용 RPM 오프라인 저장소

**4. 사이트 Bastion용 RPM 오프라인 저장소**

사이트 Bastion(§3)에서 설치할 패키지를 RPM으로 내려받아 오프라인 저장소를 만듭니다. **폐쇄망에서는 나중에 추가할 수 없습니다.**

| 패키지 | 용도 |
|---|---|
| `podman` | `mas` CLI 실행 (§3.3 이후) |
| `dnsmasq`, `bind-utils` | DNS 서버·`nslookup` (§3.2) |
| `chrony` | NTP 서버 (§3.2.1) |
| `firewalld` | 방화벽 (§3.2) |
| `nfs-utils` | RWX용 NFS 서버 (§3.8.2) |
| `nmstate` | Agent-based Installer의 `networkConfig` 검증 (§3.6) |
| `jq` | Pull Secret 병합 (§3.6, §3.7.2) |
| `tar`, `gzip` | `oc`·`mirror-registry` 압축 해제 (§3.4, §3.6) |
| `openssl`, `rsync`, `createrepo_c` | 예비 |

⚠️ 이 표는 **§3.1의 설치 목록과 동일**해야 합니다.

**1) 실행** (root 계정)

```bash
rm -rf /home/maximo/mas-install/rpms
mkdir -p /home/maximo/mas-install/rpms

dnf download --resolve --alldeps --disablerepo=mas-offline \
  --destdir /home/maximo/mas-install/rpms \
  podman openssl jq rsync tar gzip dnsmasq bind-utils firewalld createrepo_c \
  nfs-utils nmstate chrony \
  sssd-nfs-idmap container-selinux passt-selinux nmstate-libs

createrepo_c /home/maximo/mas-install/rpms
chown -R maximo:maximo /home/maximo/mas-install/rpms
```

- `--alldeps` — 이미 설치된 의존성까지 받습니다. 빼면 사이트에서 깨집니다.
- 뒤 4개는 **조건부(rich) 의존성**이라 `--resolve`가 잡지 못해 직접 지정합니다. §3.1에서는 dnf가 자동으로 끌어오므로 설치 목록에는 넣지 않습니다.

| 패키지 | 요구 조건 |
|---|---|
| `sssd-nfs-idmap` | `nfs-utils` 설치 시 `sssd-common`이 요구 |
| `container-selinux` | `podman`이 `selinux-policy` 있을 때 요구 |
| `passt-selinux` | `passt`가 `selinux-policy-targeted` 있을 때 요구 |
| `nmstate-libs` | `nmstate`와 버전 일치 필요 |

**2) 확인 및 검증**

```bash
# 서명·체크섬 — 출력이 없어야 정상
rpm -K /home/maximo/mas-install/rpms/*.rpm | grep -v 'digests signatures OK'

# 의존성 완결성 — 미해결 0건이어야 정상
dnf repoclosure --repofrompath=chk,file:///home/maximo/mas-install/rpms \
  --repo=chk --check=chk
```

미해결이 나오면 그 패키지명을 1)의 목록에 추가하고 반복합니다.

```
package: podman-5:5.4.0-1.el9.x86_64 from chk
  unresolved deps (1):
    (container-selinux if selinux-policy)     ← container-selinux 를 추가
```

🔴 `--skip-broken` / `--nobest`로 넘기지 마세요. 사이트에서 그대로 재현되고, 그때는 인터넷이 없습니다.

#### 5. OpenShift Client / Installer / checksum

§3.6에서 `oc`·`kubectl`·`openshift-install`로 사용합니다.

**1) 실행**

```bash
export OCP_BASE_URL="https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.20.30"

curl -fL "$OCP_BASE_URL/openshift-client-linux.tar.gz"  -o ~/mas-install/ocp/openshift-client-linux.tar.gz
curl -fL "$OCP_BASE_URL/openshift-install-linux.tar.gz" -o ~/mas-install/ocp/openshift-install-linux.tar.gz
curl -fL "$OCP_BASE_URL/sha256sum.txt"                  -o ~/mas-install/ocp/sha256sum.txt
```

**2) 확인 및 검증** — 두 파일 모두 `OK`여야 합니다.

```bash
cd ~/mas-install/ocp
sha256sum -c <(grep -E 'openshift-(client|install)-linux.tar.gz' sha256sum.txt)
```

#### 6. MAS CLI 이미지

§3.3에서 불러와 `mas` 명령을 실행합니다.

**1) 실행**

```bash
podman pull --arch amd64 quay.io/ibmmas/cli:23.4.1
podman save -o ~/mas-install/mas-cli/mas-cli-23.4.1.tar quay.io/ibmmas/cli:23.4.1
```

**2) 확인 및 검증** — `linux/amd64`여야 합니다.

```bash
podman image inspect quay.io/ibmmas/cli:23.4.1 --format '{{.Os}}/{{.Architecture}}'
ls -lh ~/mas-install/mas-cli/
```

#### 7. Mirror Registry 설치 패키지

§3.4에서 폐쇄망 Registry를 구축하는 데 사용합니다. [Red Hat OpenShift Downloads](https://console.redhat.com/openshift/downloads)의 **"mirror registry for Red Hat OpenShift"** 에서 RHEL 9 x86_64용 URL을 브라우저로 발급받습니다(로그인 필요, URL에 유효시간 있음).

**1) 실행** — 발급받은 URL을 붙여넣습니다.

```bash
read -r MIRROR_REGISTRY_DOWNLOAD_URL
curl -fL "$MIRROR_REGISTRY_DOWNLOAD_URL" -o ~/mas-install/registry/mirror-registry-amd64.tar.gz
unset MIRROR_REGISTRY_DOWNLOAD_URL
```

**2) 확인 및 검증**

```bash
ls -lh ~/mas-install/registry/
tar -tzf ~/mas-install/registry/mirror-registry-amd64.tar.gz | head
```

> `oc-mirror`는 별도로 받을 필요가 없습니다 — `oc`와 함께 MAS CLI 이미지에 내장되어 있습니다.

#### 8. Red Hat 콘텐츠 이미지 세트

OCP 4.20.30 플랫폼 release payload(190개)와 Red Hat Operator(379개)를 미러링합니다. 실측 **약 6시간** + tar 생성 15분. 산출물은 `mirror_000001.tar` **373GB**입니다.

**0) 초기화** — 처음부터 다시 받을 때만 실행합니다. **재시도(이어받기)라면 건너뛰세요.**

```bash
podman ps
podman kill $(podman ps -q --filter ancestor=quay.io/ibmmas/cli:23.4.1) 2>/dev/null

rm -rf ~/mas-install/mirror/redhat
rm -rf ~/mas-install/oc-mirror-cache
rm -f ~/mas-install/mirror/redhat-mirror*.log ~/mas-install/mirror/redhat-imageset.log

df -h /home
```

**1) 실행 — imageset 생성**

`mas mirror-redhat-images`로 `imageset-ocp4.20.yml`과 `config.json`을 만들고, 생성되면 중단합니다. 미러링은 2)에서 수행합니다.

```bash
podman run --rm --name mas-imageset \
  -v "$HOME/mas-install/mirror":/mnt/workspace:z \
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
  > "$HOME/mas-install/mirror/redhat-imageset.log" 2>&1 &

until [ -s ~/mas-install/mirror/redhat/imageset-ocp4.20.yml ] \
   && [ -s ~/mas-install/mirror/redhat/config.json ]; do sleep 5; done

podman kill mas-imageset

# 래퍼를 중단하면 working-dir이 반쯤 만들어진 채 남습니다.
# 그대로 두면 2)가 "failed to find release image in index"로 실패하므로 삭제합니다.
rm -rf ~/mas-install/mirror/redhat/working-dir ~/mas-install/mirror/redhat/logs

ls -la ~/mas-install/mirror/redhat/
```

`config.json`과 `imageset-ocp4.20.yml` **두 개만** 남아야 합니다.

**2) 실행 — 미러링**

```bash
mkdir -p ~/mas-install/oc-mirror-cache

nohup podman run --rm \
  -v "$HOME/mas-install/mirror":/mnt/workspace:z \
  -v "$HOME/mas-install/oc-mirror-cache":/mnt/cache:z \
  -e DOCKER_CONFIG=/mnt/workspace/redhat \
  quay.io/ibmmas/cli:23.4.1 \
  oc mirror --remove-signatures --v2 \
    -c /mnt/workspace/redhat/imageset-ocp4.20.yml \
    file:////mnt/workspace/redhat \
    --cache-dir /mnt/cache \
    --image-timeout 60m \
  > "$HOME/mas-install/mirror/redhat-mirror.nohup.log" 2>&1 &

disown
echo "PID: $!"
```

| 옵션 | 이유 |
|---|---|
| `--cache-dir /mnt/cache` | 캐시 기본값은 컨테이너 내부 `$HOME`이라 `--rm` 종료 시 소실됩니다. 호스트로 빼야 실패 후 이어받기가 됩니다. |
| `--image-timeout 60m` | 기본 10분으로는 `rhoai` 등 대용량 이미지가 완료되지 않습니다. |

**3) 확인 및 검증**

진행 중 — `oc-mirror-cache`가 계속 커지면 정상입니다.

```bash
watch -n 10 'du -sh ~/mas-install/oc-mirror-cache ~/mas-install/mirror/redhat; df -h /home | tail -1; podman ps --format "{{.Status}}"'
```

```bash
tail -f ~/mas-install/mirror/redhat-mirror.nohup.log

# 타임아웃 발생 여부 (0 이어야 정상)
grep -c "context deadline exceeded" ~/mas-install/mirror/redhat-mirror.nohup.log

# 실패한 Operator (위 값이 0이 아닐 때)
grep -ohP 'Operators: \[\K[^\]]+' ~/mas-install/mirror/redhat/working-dir/logs/*.txt | sort | uniq -c | sort -rn
```

완료 판정 — 로그 마지막이 아래 형태여야 하고, `👋 Goodbye` **뒤에 `[ERROR]` 줄이 없어야** 합니다.

```
✓  190 / 190 release images mirrored successfully
✓  379 / 379 operator images mirrored successfully
📦 Preparing the tarball archive...
👋 Goodbye, thank you for using oc-mirror
```

🔴 **하나라도 실패하면 tar를 만들지 않습니다.** `✗ 373 / 379` 처럼 나오면 실패이며, 대상에서 제외할 Operator를 정하거나 재실행해야 합니다.

산출물은 tar 하나입니다.

```bash
ls -lh ~/mas-install/mirror/redhat/mirror_*.tar      # 실측 373GB
```

⚠️ 로그가 이미지 복사 도중 끊기고 `container not running: No such process`로 끝났다면 **SSH 세션 종료로 컨테이너가 죽은 것**입니다(§4.4).

실패했으면 **2)를 그대로 다시 실행**합니다 — 캐시에 있는 것은 건너뜁니다(§4.3 참고). 성공 후 캐시는 삭제합니다.

```bash
rm -rf ~/mas-install/oc-mirror-cache
```

#### 9. MAS 콘텐츠 이미지 세트

Catalog/Core, Manage + Maximo IT, 공통 의존성 + Db2, CLI를 **순차 미러링**합니다. 앞 단계 완료를 확인한 뒤 다음을 실행하세요 — 동시 실행하면 SELinux 라벨이 충돌합니다(§4.1).

⛔ **Cloud Pak for Data는 받지 않습니다.** `--mirror-cp4d`, `--mirror-wsl`, `--mirror-wml`, `--mirror-spark`, `--mirror-cognos`를 어떤 명령에도 넣지 마세요 — 이번 구성에 불필요하고 용량이 매우 큽니다. RWX를 NFS로 하므로 `--mirror-odf`도 제외합니다.

**1) 실행 — 환경변수** (셸마다 한 번)

`to-filesystem` 모드에서도 `-H/-P/-u/-p`가 필수입니다(생략하면 `--help`만 출력하고 종료). Registry Host/Port는 목적지 이미지 경로 생성에 쓰이므로 **최종 폐쇄망 Registry의 실제 값**을 넣으세요.

```bash
export LOCAL_DIR="$HOME/mas-install/mirror"
export MIRROR_SINGLE_ARCH=amd64
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)

export REGISTRY_HOST=registry.mas-it.itmsg.co.kr
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=masadmin
read -s REGISTRY_PASSWORD
export REGISTRY_PASSWORD
```

**2) 실행 — core** (Catalog + MAS Core, 약 16GB)

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
  --no-confirm > "$HOME/mas-install/mirror/stage-core.nohup.log" 2>&1 &

disown
```

진행 확인 : 프로세스 수가 `1` 이상이면 실행 중입니다. `0`이면 종료된 것이니 아래 검증으로 넘어갑니다

```bash
watch -n 10 'du -sh ~/mas-install/mirror/core; ps aux | grep -c "[m]as mirror-images"'
```

검증 : 프로세스가 사라진 뒤 `[SUCCESS] IBM Maximo Application Suite Core`가 나오고 `[FAILURE]`가 없어야 함

```bash
tr '\r' '\n' < ~/mas-install/mirror/stage-core.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

**3) 실행 — apps** (Manage + Maximo IT, 약 25GB)

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
  --no-confirm > "$HOME/mas-install/mirror/stage-apps.nohup.log" 2>&1 &

disown
```

진행 확인 : 프로세스 수가 `1` 이상이면 실행 중입니다. `0`이면 종료된 것이니 아래 검증으로 넘어갑니다

```bash
watch -n 10 'du -sh ~/mas-install/mirror/apps; ps aux | grep -c "[m]as mirror-images"'
```

검증 : 프로세스가 사라진 뒤 `[SUCCESS] IBM Maximo Manage`가 나오고 `[FAILURE]`가 없어야 함

```bash
tr '\r' '\n' < ~/mas-install/mirror/stage-apps.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

검증 : 🔴 **Maximo IT는 상태 목록에 나오지 않습니다** — Manage 확장이라 Manage 매니페스트에 포함됩니다. `extension-icd`가 있어야 §3.10에서 활성화할 수 있으므로 직접 확인하세요.

```bash
ls ~/mas-install/mirror/apps/v2/cp/manage/ | grep -i icd
```

**4) 실행 — other** (Mongo/TSM/SLS/CFS/Db2, 약 43GB)

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
  --no-confirm > "$HOME/mas-install/mirror/stage-other.nohup.log" 2>&1 &

disown
```

진행 확인 : 프로세스 수가 `1` 이상이면 실행 중입니다. `0`이면 종료된 것이니 아래 검증으로 넘어갑니다

```bash
watch -n 10 'du -sh ~/mas-install/mirror/other; ps aux | grep -c "[m]as mirror-images"'
```

검증 : 프로세스가 사라진 뒤 `[SUCCESS] Selected Dependencies`가 나오고 `[FAILURE]`가 없어야 함

```bash
tr '\r' '\n' < ~/mas-install/mirror/stage-other.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

**5) 실행 — cli** (MAS CLI, 약 5GB)

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
  --no-confirm > "$HOME/mas-install/mirror/stage-cli.nohup.log" 2>&1 &

disown
```

진행 확인 : 프로세스 수가 `1` 이상이면 실행 중입니다. `0`이면 종료된 것이니 아래 검증으로 넘어갑니다

```bash
watch -n 10 'du -sh ~/mas-install/mirror/cli; ps aux | grep -c "[m]as mirror-images"'
```

검증 : 프로세스가 사라진 뒤 `[SUCCESS] IBM Maximo CLI`가 나오고 `[FAILURE]`가 없어야 함

```bash
tr '\r' '\n' < ~/mas-install/mirror/stage-cli.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

**6) 확인 및 검증** — 4단계 전체

검증 : `[SUCCESS]` 4건, `[FAILURE]` **0건**

```bash
for f in ~/mas-install/mirror/stage-*.nohup.log; do
  echo "===== $(basename $f)"
  tr '\r' '\n' < "$f" | grep -aE '\[SUCCESS\]|\[FAILURE\]'
done
```

🔴 **`[FAILURE]`와 `[SUCCESS]`가 같이 나올 수 있습니다.** 한 단계에서 여러 컴포넌트를 미러링하므로 일부만 실패할 수 있습니다. `[SUCCESS]` 개수만 세지 말고 **`[FAILURE]`가 0건인지** 확인하세요.

실패하면 디렉터리를 비우지 말고 **해당 단계를 그대로 다시 실행**합니다 — 이미 받은 이미지는 건너뜁니다.

#### 10. NFS 동적 프로비저너 이미지 (RWX용)

§3.8.2에서 RWX StorageClass를 제공하는 `nfs-subdir-external-provisioner`입니다.

🔴 이 이미지는 **8번·9번 미러링 어느 쪽에도 포함되지 않습니다.** 커뮤니티 프로젝트라 별도로 받아야 하며, 폐쇄망 반입 후에는 확보할 수 없습니다.

**1) 실행**

```bash
export NFS_PROV_IMAGE=registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
export NFS_PROV_REF=nfs-subdir-external-provisioner-4.0.18   # 릴리스 태그 고정

podman pull --arch amd64 "$NFS_PROV_IMAGE"
podman save -o ~/mas-install/nfs-provisioner/nfs-subdir-external-provisioner.tar "$NFS_PROV_IMAGE"

cd ~/mas-install/nfs-provisioner
for f in rbac.yaml deployment.yaml class.yaml test-claim.yaml test-pod.yaml; do
  curl -fLO "https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/${NFS_PROV_REF}/deploy/$f" \
    || curl -fLO "https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/$f"
done
```

**2) 확인 및 검증** — tar 1개 + YAML 5개, 이미지는 `linux/amd64`.

```bash
podman image inspect "$NFS_PROV_IMAGE" --format '{{.Os}}/{{.Architecture}}'
ls -lh ~/mas-install/nfs-provisioner/
```

> `csi-driver-nfs`를 쓰면 CSI sidecar까지 5~6개 이미지를 받아야 합니다. 단순함을 위해 `nfs-subdir` 방식을 택했습니다.

### 2.4 설치 파일 최종 확인

§2.3의 모든 항목이 끝나면, 반입 전에 아래를 전부 통과해야 합니다.

기대 구조:

```text
~/mas-install/
  licenses/         entitlement_key.key, <license>.dat
  redhat/           pull-secret.json
  rpms/             RPM 229개 (135MB) + repodata/
  ocp/              openshift-client-linux.tar.gz, openshift-install-linux.tar.gz, sha256sum.txt
  mas-cli/          mas-cli-23.4.1.tar
  registry/         mirror-registry-amd64.tar.gz
  nfs-provisioner/  nfs-subdir-external-provisioner.tar + YAML 5개
  mirror/
    redhat/         mirror_000001.tar 373GB + working-dir 25GB
    core/ apps/ other/ cli/
```

**1) 확인 및 검증**

실행 : `podman ps -a`
검증 : 결과가 없어야 함 (모든 미러링 컨테이너가 `--rm`으로 삭제됨)

```bash
podman ps -a
```

실행 : MAS 콘텐츠 Stage 결과
검증 : Stage별로 `[SUCCESS]`가 나오고 `[FAILURE]`가 없어야 함

```bash
grep -aE '\[SUCCESS\]|\[FAILURE\]' ~/mas-install/mirror/stage*.nohup.log
```

> `^` 앵커를 쓰면 안 됩니다 — `mas` CLI 출력에 ANSI 색상 코드가 앞에 붙어 매칭되지 않습니다.

실행 : 🔴 Red Hat 산출물
검증 : `mirror_000001.tar` 373GB. 없으면 §2.3의 8번으로 돌아가세요

```bash
ls -lh ~/mas-install/mirror/redhat/mirror_*.tar
```

실행 : Red Hat 미러링 최종 결과
검증 : release **190/190**, operator **379/379** 모두 `✓`

```bash
grep -A3 '=== Results ===' ~/mas-install/mirror/redhat-mirror.nohup.log | tail -4
```

실행 : 최종 용량
검증 : redhat 398G / core 16G / apps 25G / other 43G / cli 4.9G. 몇 분 간격으로 재확인해 더 이상 안 늘어야 함

```bash
du -sh ~/mas-install/mirror/*
df -h /home
```

### 2.5 설치 파일 전달

§2.4를 통과하면 전달 매체로 복사해 사이트 Bastion에 반입합니다. 반입 대상은 **489GB**입니다. 라이선스·Pull Secret은 일반 매체와 분리된 절차로 반입하는 것을 권장합니다.

**1) 실행 — 체크섬 생성**

489GB를 해싱하므로 수 시간 걸립니다.

```bash
cd "$HOME/mas-install"
nohup sh -c "find licenses redhat ocp registry mas-cli rpms nfs-provisioner mirror \
  -type f -print0 | sort -z | xargs -0 sha256sum > transfer-files.sha256" &
```

```bash
wc -l ~/mas-install/transfer-files.sha256      # 더 이상 안 늘면 완료
```

**2) tar로 묶어 전송**

🔴 미러 산출물에는 `blobs/sha256:...` 처럼 **콜론이 들어간 파일이 4천 개 이상** 있습니다. 콜론은 NTFS/exFAT에서 파일명에 쓸 수 없어, Windows 파일시스템에 디렉터리째로 복사하면 그 파일들이 조용히 실패합니다. 경로 길이 제한(260자)도 걸립니다.

**그래서 tar로 묶어 조각 파일만 옮깁니다.** tar 안에서는 콜론이 문제되지 않고, 사이트 Linux에서 풀면 원래 이름이 그대로 복원됩니다 — 별도 복구 작업은 없습니다. 매체가 NTFS·exFAT·ext4 무엇이든 동일하게 동작합니다.

⚠️ **외장하드에서 tar를 풀지 마세요.** Windows에서 풀면 그 순간 콜론 파일 생성에 실패합니다. tar 조각은 손대지 않고 사이트 Linux로 옮긴 뒤 거기서 풉니다.

**실행 — tar로 묶어 분할** (인터넷 연결 서버, 489GB 추가 공간 필요)

스테이징 디렉터리는 반드시 `~/mas-install` **바깥**에 만드세요. 안에 만들면 tar가 자기가 만드는 조각을 다시 담습니다.

```bash
mkdir -p ~/mas-install-transfer
cd "$HOME/mas-install"

nohup sh -c 'tar -cf - licenses redhat ocp registry mas-cli rpms nfs-provisioner mirror transfer-files.sha256 \
  | split -b 20G -d - ~/mas-install-transfer/mas-offline.tar.part-' > ~/mas-install-transfer/tar.log 2>&1 &
```

진행 확인 : 조각 수와 용량이 늘어야 정상. 디스크 사용률이 90% 근처까지 올라갑니다

```bash
watch -n 10 'ls ~/mas-install-transfer | wc -l; du -sh ~/mas-install-transfer; df -h /home | tail -1'
```

검증 : 조각 약 25개, 합계 **489GB**. `tar.log`가 비어 있어야 함

```bash
ls -lh ~/mas-install-transfer
du -sh ~/mas-install-transfer
cat ~/mas-install-transfer/tar.log
```

**실행 — 조각 체크섬**

489GB를 다시 읽으므로 이것도 수 시간 걸립니다.

```bash
cd ~/mas-install-transfer
nohup sh -c 'sha256sum mas-offline.tar.part-* > transfer-parts.sha256' &
```

진행 확인 : 줄 수가 조각 수(25)에 도달하면 완료

```bash
watch -n 10 'wc -l ~/mas-install-transfer/transfer-parts.sha256'
```

검증 : 조각 수와 체크섬 줄 수가 같아야 함

```bash
ls ~/mas-install-transfer/mas-offline.tar.part-* | wc -l
wc -l ~/mas-install-transfer/transfer-parts.sha256
```

**실행 — 전송**

`~/mas-install-transfer`의 모든 파일(조각 + `transfer-parts.sha256`)을 FileZilla(SFTP)로 외장하드에 받고, 외장하드를 사이트 Bastion으로 옮겨 다시 SFTP로 올립니다.

**실행 — 사이트 Bastion에서 복원**

```bash
cd <조각을 올린 경로>
sha256sum -c transfer-parts.sha256                       # 조각 무결성

mkdir -p ~/mas-install
cat mas-offline.tar.part-* | tar -xf - -C ~/mas-install
```

전송이 끝나면 인터넷 연결 서버의 스테이징을 지워 489GB를 회수합니다.

```bash
rm -rf ~/mas-install-transfer
```

---

**3) 확인 및 검증** — 출력이 없어야 §3.1로 진행합니다.

```bash
cd ~/mas-install
sha256sum -c transfer-files.sha256 | grep -v ': OK$'
du -sh ~/mas-install/*
find ~/mas-install/mirror -name '*:*' | wc -l      # 4000 이상이어야 정상
```

---

## 3. 설치 과정

**실제 사이트 Bastion 서버**에서 진행합니다.

### 계정 사용 원칙

각 절 제목에 실행 계정을 표시했습니다. **사용자 계정**(이 문서의 예시는 `maximo`)으로 SSH 접속한 뒤, root가 필요한 구간에서만 `sudo -i`로 전환합니다.

```bash
ssh maximo@192.168.2.210
sudo -i          # root 구간(§3.1, §3.2, §3.4 CA 등록, §3.8.2 NFS 서버) 진입
exit             # 끝나면 사용자 계정으로 복귀
```

| 구간 | 계정 | root가 필요한 이유 |
|---|---|---|
| §3.1 | **root** | `/etc/yum.repos.d/`, 시스템 패키지 설치 |
| §3.2 | **root** | `/etc/dnsmasq.d/`, `systemctl`, `firewall-cmd` |
| §3.4 (일부) | **root** | `/etc/containers/certs.d/`·`/etc/pki/ca-trust/` CA 등록 |
| §3.8.2 A-1 | **root** | `/etc/exports`, `nfs-server`, 방화벽 |
| 그 외 전부 | 사용자 계정 | `mas` CLI, Mirror Registry 설치·운영, 이미지 push, `mas install` 등 (rootless podman) |

🔴 **주의 — `~` 경로 함정**: `sudo -i` 후에는 `$HOME`이 `/root`가 되므로 **`~/mas-install/...`이 `/root/mas-install/...`로 해석**됩니다. 그래서 이 문서의 **root 구간은 모두 절대 경로**(`/home/maximo/mas-install/...`)로 작성했고, **사용자 계정 구간은 `~`** 를 씁니다. 명령을 복사할 때 현재 어느 셸에 있는지 (`whoami`로) 확인하세요.

```bash
whoami   # root 인지 maximo 인지 확인
```

### 3.1 RPM 오프라인 저장소 등록 및 도구 설치 (root 계정)

§2.3의 4번에서 반입한 `rpms/`를 로컬 저장소로 등록하고 §3에 필요한 도구를 설치합니다.

🔴 RHEL은 GPG 키 **파일**은 제공하지만 **RPM DB에 임포트되지 않은 경우**가 있습니다. 이 상태로 `gpgcheck=1` 저장소를 쓰면 `"you do not have any GPG public keys installed"` 로 설치가 전부 실패합니다.

**1) 실행**

```bash
# GPG 키 임포트
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

# 저장소 등록
cat > /etc/yum.repos.d/mas-offline.repo <<'EOF'
[mas-offline]
name=MAS Offline RPM Repository
baseurl=file:///home/maximo/mas-install/rpms
enabled=1
gpgcheck=1
repo_gpgcheck=0
EOF

# 도구 설치
dnf --disablerepo='*' --enablerepo=mas-offline clean metadata
dnf --disablerepo='*' --enablerepo=mas-offline install -y \
  podman openssl jq rsync tar gzip dnsmasq bind-utils firewalld createrepo_c \
  nfs-utils nmstate chrony
```

⚠️ `clean metadata`를 빠뜨리면 dnf가 이전 캐시를 써서 새 패키지를 `No match for argument`로 보고합니다.

**2) 확인 및 검증**

```bash
rpm -qa 'gpg-pubkey*'          # 항목이 나와야 정상
podman --version
rpm -q podman dnsmasq bind-utils firewalld nfs-utils nmstate chrony
nmstatectl --version
```

### 3.2 DNS 구성 (root 계정)

🔴 아래 4개 이름이 resolve되지 않으면 **SNO 설치가 완료되지 않습니다** (Red Hat Agent-based Installer 요구사항).

아래 값을 정하여 `mas.conf` 파일을 작성합니다.

`<cluster-name>`, `<base-domain>`, `<sno-ip>`, `<bastion-ip>`

**1) 실행 — `mas.conf` 작성**

```bash
vi /etc/dnsmasq.d/mas.conf
```

```ini
# 상위 DNS: 사내 DNS가 있으면 server=<dns-ip> 로 추가, 없으면 넣지 않음
server=<dns-ip>
address=/api.<cluster-name>.<base-domain>/<sno-ip>
address=/api-int.<cluster-name>.<base-domain>/<sno-ip>
address=/.apps.<cluster-name>.<base-domain>/<sno-ip>
address=/registry.<cluster-name>.<base-domain>/<bastion-ip>
```

예시)

```ini
#사내 DNS 가 없으면 주석 처리
#server=8.8.8.8
address=/api.mas-it.itmsg.co.kr/192.168.2.211
address=/api-int.mas-it.itmsg.co.kr/192.168.2.211
address=/.apps.mas-it.itmsg.co.kr/192.168.2.211
address=/registry.mas-it.itmsg.co.kr/192.168.2.210
```

**2) 실행 — 서비스·방화벽 기동**

```bash
systemctl enable --now dnsmasq firewalld

firewall-cmd --permanent --add-service=dns     # 53   DNS
firewall-cmd --permanent --add-port=8443/tcp   # 8443 Mirror Registry
firewall-cmd --permanent --add-port=6443/tcp   # 6443 OpenShift API
firewall-cmd --permanent --add-port=443/tcp    # 443  Router (MAS UI, Console)
firewall-cmd --permanent --add-port=80/tcp     # 80   HTTP redirect
firewall-cmd --reload

firewall-cmd --list-all  # 적용 확인

```

**3) 실행 — Bastion 자신이 dnsmasq를 쓰도록 설정**

빼먹으면 §3.5 push와 §3.7 `oc login`이 실패합니다.

먼저 연결 이름을 확인합니다.

```bash
nmcli -t -f NAME,DEVICE connection show --active
```

형식)

```bash
CONN=<연결 이름>

# 127.0.0.1이 먼저여야 dnsmasq가 클러스터 이름을 먼저 응답합니다
# 사내 DNS가 없으면 <dns-ip>를 빼고 "127.0.0.1" 만
nmcli connection modify "$CONN" ipv4.dns "127.0.0.1 <dns-ip>"
nmcli connection modify "$CONN" ipv4.ignore-auto-dns yes
nmcli connection up "$CONN"

systemctl restart dnsmasq
```

예시)

```bash
CONN=ens192

# 사내 DNS 없음
nmcli connection modify "$CONN" ipv4.dns "127.0.0.1"
nmcli connection modify "$CONN" ipv4.ignore-auto-dns yes
nmcli connection up "$CONN"

systemctl restart dnsmasq
```

⚠️ `nmcli connection up`이 네트워크를 잠시 끊습니다. `ipv4.dns`만 바꾸므로 IP는 유지되지만, SSH가 재연결될 수 있습니다.

**4) 확인 및 검증**

검증 : 4개 모두 지정한 IP로 응답해야 함

```bash
cat /etc/resolv.conf                    # nameserver 127.0.0.1
nslookup api.mas-it.itmsg.co.kr         # → 192.168.2.211
nslookup api-int.mas-it.itmsg.co.kr     # → 192.168.2.211
nslookup test.apps.mas-it.itmsg.co.kr   # → 192.168.2.211
nslookup registry.mas-it.itmsg.co.kr    # → 192.168.2.210
```

검증 : 외부 이름은 **실패해야** 정상입니다 (폐쇄망)

```bash
nslookup quay.io
```

> dnsmasq 로그에 `ignoring nameserver 127.0.0.1 - local interface`가 뜨는 것은 정상입니다 — 자기 자신을 상위 DNS로 참조하지 않는다는 뜻입니다.

<details>
<summary>참고 — 상위 DNS 제거 / PTR / 방화벽 범위 / 클라이언트</summary>

사내 DNS가 없다면 `server=` 줄을 주석 처리합니다.

```bash
sed -i 's/^server=/#server=/' /etc/dnsmasq.d/mas.conf
nmcli connection modify "$CONN" ipv4.dns "127.0.0.1"
nmcli connection up "$CONN"
systemctl restart dnsmasq
nslookup api.mas-it.itmsg.co.kr        # 여전히 응답해야 함
```

- **PTR** — SNO에서 필수인지 권장인지 ⚠️ 미검증. `mas-it-sno.mas-it.itmsg.co.kr` → `192.168.2.211`, 역방향 2건(SNO/Bastion)을 준비해두는 편이 안전합니다.
- **방화벽 범위** — 위 포트는 Bastion 기준입니다. 6443/443/80은 SNO로 향하므로 중간 네트워크 장비에서도 허용되어야 합니다.
- **클라이언트** — MAS UI에 접근할 PC도 이 Bastion을 DNS로 지정하거나 `hosts`에 등록해야 합니다.
- ⚠️ `itmsg.co.kr`처럼 실제 공인 도메인을 쓰는 경우, 공인 DNS에 같은 이름을 만들지 마세요.

</details>

#### 3.2.1 NTP 구성 (root 계정)

🔴 시간 동기가 어긋나면 인증서 검증 실패, etcd 이상, Operator 오류로 이어집니다. 폐쇄망에서는 외부 NTP에 접근할 수 없으므로 **Bastion을 내부 NTP 서버로** 구성합니다.

**1) 실행 — 시간 보정**

🔴 `local stratum 10`은 **이 서버 시계를 정답으로 삼는다**는 뜻입니다. 시간이 틀어진 상태로 구성하면 SNO 전체가 틀린 시간을 받아 인증서 검증이 깨집니다. **먼저 시간을 맞추세요.**

```bash
timedatectl                                    # 현재 시간·시간대 확인
timedatectl set-timezone Asia/Seoul            # 필요 시
```

형식)

```bash
timedatectl set-time '<YYYY-MM-DD HH:MM:SS>'
```

예시)

```bash
timedatectl set-time '2026-08-05 17:16:00'
```

⚠️ `chronyd`가 실행 중이면 `set-time`이 거부됩니다. 그 경우 잠시 멈추고 맞춘 뒤 다시 시작하세요.

```bash
systemctl stop chronyd
timedatectl set-time '2026-08-05 17:16:00'
hwclock --systohc                              # BIOS 시계에도 반영
```

**2) 실행 — chrony 구성**

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
```

> 사이트에 상위 NTP가 있으면 `local stratum 10` 대신 `server <upstream-ntp-ip> iburst`를 씁니다. 그러면 시간 보정도 자동으로 됩니다.
> SNO에는 §3.6 `agent-config.yaml`의 `additionalNTPSources`로 Bastion IP를 전달합니다.

**3) 확인 및 검증**

검증 : `Stratum 10`, `Reference ID 7F7F0101`(=127.127.1.1)이면 로컬 클럭 기준으로 서비스 중

```bash
chronyc tracking
```

검증 : `0.0.0.0:123`으로 대기해야 SNO가 질의할 수 있음

```bash
ss -ulnp | grep 123
```

검증 : 시간이 실제 시각과 맞아야 함

```bash
timedatectl
```

> `System clock synchronized: no`는 정상입니다 — 상위 NTP 없이 자기 클럭을 기준으로 쓸 때 chrony는 "동기화됨"으로 보고하지 않습니다. SNO에 시간을 제공하는 기능은 정상 동작합니다.
>
> `chronyc clients`는 SNO 설치 후에야 항목이 나옵니다.

### 3.3 MAS CLI 이미지 불러오기 (사용자 계정)

인터넷 연결 서버에서 저장해 온 tar를 불러옵니다(§2.3의 6번).

**1) 실행**

```bash
podman load -i ~/mas-install/mas-cli/mas-cli-23.4.1.tar
```

**2) 확인 및 검증**

```bash
podman image exists quay.io/ibmmas/cli:23.4.1 && echo OK
podman run --rm -t quay.io/ibmmas/cli:23.4.1 mas mirror-images --help | head -10
```

첫 줄에 `IBM Maximo Application Suite Air Gap Image Mirror (v23.4.1)` 이 나오면 정상입니다.

> - `mas --version`·`mas --help`는 없는 파라미터입니다(`unknown parameter`). `--help`는 서브커맨드에만 있습니다.
> - `-t`가 없으면 `tput: No value for $TERM` 경고가 나옵니다.
> - 끝에 `write /dev/stdout: broken pipe`가 뜨는 것은 `head`가 파이프를 먼저 닫아서이며 무해합니다.

### 3.4 Mirror Registry 설치 (사용자 계정)

Bastion은 같은 이미지를 **두 벌 보관**합니다.

| | 정체 | 위치 | 용량 |
|---|---|---|---|
| 미러 파일 | §2.5에서 반입한 원본 | `~/mas-install/mirror/` | 489GB |
| Registry 사본 | §3.5 push 때 Quay가 생성 | `~/mirror-registry/quay` | 약 489GB |

> 빈 디스크가 있으면 `~/mirror-registry`에 마운트하세요. §3.5 push 때 읽기(미러 파일)와 쓰기(Registry)가 다른 디스크로 분산되어 빨라집니다. 없으면 그냥 진행하면 됩니다 — 루트 파일시스템에 약 489GB 여유만 있으면 동작합니다.

**1) 실행 — 디렉터리 준비**

```bash
mkdir -p ~/quay-install ~/mirror-registry/{quay,sqlite}
chmod 750 ~/quay-install ~/mirror-registry
```

**2) 실행 — Registry 설치**

압축을 풀면 `image-archive.tar`(Quay·Redis·sqlite 이미지)가 함께 나옵니다 — 폐쇄망에서 추가로 받을 것이 없습니다.

```bash
mkdir -p ~/mas-install/registry/bin
tar -xzf ~/mas-install/registry/mirror-registry-amd64.tar.gz -C ~/mas-install/registry/bin
cd ~/mas-install/registry/bin
ls -la                # image-archive.tar, execution-environment.tar, sqlite3.tar, mirror-registry
```

형식)

```bash
./mirror-registry install -v \
  --targetHostname <bastion-ip> \
  --quayHostname   registry.<cluster-name>.<base-domain>:8443 \
  --quayRoot       "$HOME/quay-install" \
  --quayStorage    "$HOME/mirror-registry/quay" \
  --sqliteStorage  "$HOME/mirror-registry/sqlite"
```

예시)

```bash
./mirror-registry install -v \
  --targetHostname 192.168.2.210 \
  --quayHostname   registry.mas-it.itmsg.co.kr:8443 \
  --quayRoot       "$HOME/quay-install" \
  --quayStorage    "$HOME/mirror-registry/quay" \
  --sqliteStorage  "$HOME/mirror-registry/sqlite"
```

| 옵션 | 이유 |
|---|---|
| `--targetHostname` | 기본값이 `$HOST`(호스트명)인데 IPv6 링크로컬로 해석되면 SSH가 실패합니다. IP를 명시합니다 |
| `--quayHostname` **`:8443`** | 기본값 형태가 `<host>:8443`입니다. 포트가 빠지면 Quay가 생성하는 URL에 포트가 없어 리다이렉트가 깨질 수 있습니다 |
| `--quayStorage`·`--sqliteStorage` | 지정하지 않으면 podman named volume(`quay-storage`, `sqlite-storage`)에 저장돼 **별도 디스크로 분리할 수 없습니다** |

5~15분 걸립니다. `image-archive.tar`에서 이미지를 로드하는 시간이 대부분입니다.

🔴 마지막에 출력되는 **초기 사용자명·비밀번호를 반드시 기록하세요.** 다시 볼 수 없고 §3.5 push와 §3.7.2에서 씁니다.

```
Quay is available at https://registry.mas-it.itmsg.co.kr:8443
with credentials (init, <생성된 비밀번호>)
```

실패하면 되돌릴 수 있습니다.

```bash
./mirror-registry uninstall
```

🔴 **§3.6에서 쓸 읽기 전용(pull-only) 계정을 지금 만들어 두세요.** Quay 웹 UI(`https://registry.mas-it.itmsg.co.kr:8443`)에서 사용자를 추가하고 읽기 권한만 부여합니다. 이 자격증명은 모든 OCP 노드에 배포됩니다.

**3) 실행 — Bastion에 Registry CA 신뢰 등록** (root 계정)

podman과 시스템 전역이 이 Registry의 자체 서명 인증서를 신뢰하도록 등록합니다. 빼먹으면 §3.5 push가 TLS 오류로 실패합니다.

```bash
REGISTRY=registry.mas-it.itmsg.co.kr:8443

REGISTRY_CA="$(find /home/maximo/quay-install -name rootCA.pem -type f -print -quit)"
install -d -m 0755 "/etc/containers/certs.d/$REGISTRY"
install -m 0644 "$REGISTRY_CA" "/etc/containers/certs.d/$REGISTRY/ca.crt"
install -m 0644 "$REGISTRY_CA" "/etc/pki/ca-trust/source/anchors/${REGISTRY%:*}.crt"
update-ca-trust
```

**4) 확인 및 검증**

검증 : Quay 컨테이너가 떠 있어야 함

```bash
podman ps
```

검증 : `401` 이 나오면 Registry가 응답하고 인증을 요구하는 정상 상태

```bash
curl -s -o /dev/null -w '%{http_code}\n' "https://registry.mas-it.itmsg.co.kr:8443/v2/"
```

> ⚠️ `mirror registry for Red Hat OpenShift`는 Red Hat 공식적으로 **OCP 설치용 소규모·비HA Registry**입니다. MAS 전체 이미지를 담는 장기 운영 Registry는 지원 범위 밖이므로, 운영 전환 시 Quay/Harbor 등을 검토하세요.

### 3.5 Registry로 이미지 Push (from-filesystem) (사용자 계정)

반입한 미러 데이터를 Mirror Registry로 올립니다. `from-filesystem`에는 `-c`/`-C`가 필요하고, `--release` 계열은 `to-filesystem` 전용이라 쓰지 않습니다.

**1) 실행 — 환경변수**

```bash
export REGISTRY_HOST=registry.mas-it.itmsg.co.kr
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=masadmin
export LOCAL_DIR="$HOME/mas-install/mirror"
read -s REGISTRY_PASSWORD
export REGISTRY_PASSWORD
```

**2) 실행 — tar 파일명 변경**

🔴 래퍼가 `mirror_seq1_000000.tar` 를 하드코딩하고 있는데 oc-mirror는 `mirror_000001.tar` 로 만듭니다(§4.5). 이름을 맞춰야 합니다.

⚠️ **§2.5의 체크섬 검증을 먼저 끝내세요.** 이름을 바꾸면 `transfer-files.sha256`과 어긋나 검증에서 걸립니다.

```bash
cd ~/mas-install/mirror/redhat
mv mirror_000001.tar mirror_seq1_000000.tar
ls -lh mirror_seq1_000000.tar
```

**3) 실행 — Red Hat 콘텐츠 push**

```bash
podman run --rm -v "$LOCAL_DIR":/mnt/workspace:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-redhat-images \
  --mode from-filesystem --dir /mnt/workspace/redhat \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  --no-confirm
```

> 래퍼가 내부적으로 실행하는 명령입니다 — 실패 시 직접 실행할 때 참고하세요.
>
> ```
> DOCKER_CONFIG=<dir> oc mirror --remove-signatures -c <dir>/imageset-ocp4.20.yml \
>   --dest-tls-verify=false --parallel-images=1 \
>   --from file://<dir>/mirror_seq1_000000.tar docker://<registry> --v2
> ```

**4) 실행 — MAS 콘텐츠 push**

`STAGE`와 `FLAGS`만 바꿔 **4회 반복**합니다.

| 순서 | `STAGE` | `FLAGS` |
|---|---|---|
| 1 | `core` | `--mirror-catalog --mirror-core` |
| 2 | `apps` | `--mirror-manage --mirror-icd` |
| 3 | `other` | `--mirror-mongo --mirror-tsm --mirror-sls --mirror-cfs --mirror-db2` |
| 4 | `cli` | `--mirror-cli` |

```bash
STAGE=core                                    # ← 표에서 골라 교체
FLAGS="--mirror-catalog --mirror-core"        # ← 표에서 골라 교체

podman run --rm -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/$STAGE \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  $FLAGS --no-confirm
```

**5) 확인 및 검증**

🔴 **첫 Stage(core) 직후 반드시 확인** — manifest 목적지가 실제 Registry로 재생성됐는지. `0`이 아니면 즉시 중단하세요.

```bash
grep -c localhost ~/mas-install/mirror/core/manifests/from-filesystem/ibm-mas_9.2.0.txt
```

```bash
curl -sk -u "$REGISTRY_USERNAME" https://$REGISTRY_HOST:$REGISTRY_PORT/v2/_catalog | head
unset REGISTRY_PASSWORD
```

### 3.6 OCP SNO 클러스터 설치 (사용자 계정)

Agent-based Installer로 SNO를 설치합니다.

**1) 실행 — CLI 도구 준비**

```bash
mkdir -p ~/.local/bin
tar -xzf ~/mas-install/ocp/openshift-client-linux.tar.gz  -C ~/mas-install/ocp
tar -xzf ~/mas-install/ocp/openshift-install-linux.tar.gz -C ~/mas-install/ocp
install -m 0755 ~/mas-install/ocp/{oc,kubectl,openshift-install} ~/.local/bin/

# §3.7 이후에도 새 셸에서 그대로 쓰이도록 영구 등록
cat >> ~/.bashrc <<'EOF'
export PATH="$HOME/.local/bin:$PATH"
export KUBECONFIG="$HOME/ocp-sno/auth/kubeconfig"
EOF
source ~/.bashrc

oc version --client && openshift-install version
```

**2) 실행 — 설정 파일에 넣을 값 3개 수집**

Pull Secret 병합 — 🔴 이 값은 **모든 OCP 노드에 배포**되므로 반드시 **읽기 전용 계정**을 쓰세요(§3.4에서 생성).

```bash
mkdir -p ~/ocp-sno && cd ~/ocp-sno
read -s PULL_ONLY_PASSWORD

cp ~/mas-install/redhat/pull-secret.json ./pull-secret-merged.json

printf '%s' "$PULL_ONLY_PASSWORD" | podman login \
  registry.mas-it.itmsg.co.kr:8443 \
  --username maspull --password-stdin \
  --authfile ./pull-secret-merged.json

unset PULL_ONLY_PASSWORD
chmod 600 ./pull-secret-merged.json

jq -c . ./pull-secret-merged.json          # ← pullSecret 값
```

Registry CA — `additionalTrustBundle` 값:

```bash
cat "$(find /home/maximo/quay-install -name rootCA.pem -type f -print -quit)"
```

IDMS — `imageDigestSources`의 source/mirror 쌍. `spec.imageDigestMirrors` 항목을 옮겨 적습니다(구조 거의 동일).

```bash
cat ~/mas-install/mirror/redhat/working-dir/cluster-resources/idms-oc-mirror.yaml
```

**3) 실행 — `install-config.yaml` 작성**

```yaml
apiVersion: v1
baseDomain: itmsg.co.kr
metadata:
  name: mas-it
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
pullSecret: '<2)의 jq -c 출력>'
sshKey: '<ssh-public-key>'
additionalTrustBundle: |
  <2)의 rootCA.pem 내용>
imageDigestSources:
  - source: <2)에서 확인한 source>
    mirrors:
      - <2)에서 확인한 mirror>
```

⚠️ **`imageDigestSources` vs `imageContentSources`** — 문서마다 표기가 엇갈립니다. 적용 전 확인하세요.

```bash
openshift-install explain installconfig | grep -iE 'imageDigestSources|imageContentSources'
```

**4) 실행 — `agent-config.yaml` 작성**

```yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: mas-it
rendezvousIP: 192.168.2.211
additionalNTPSources:
  - 192.168.2.210
hosts:
  - hostname: mas-it-sno
    role: master
    interfaces:
      - name: <nic-name>             # SNO 부팅 후 확인
        macAddress: "00:50:56:bb:86:df"
    rootDeviceHints:
      deviceName: /dev/<os-disk>     # 🔴 300GB(OS) 디스크 — 500GB는 LVMS용으로 비워둘 것
    networkConfig:
      interfaces:
        - name: <nic-name>
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 192.168.2.211
                prefix-length: 22    # /22
            dhcp: false
      dns-resolver:
        config:
          server:
            - 192.168.2.210           # Bastion dnsmasq
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: <nic-name>
```

🔴 **디스크 지정 주의**: SNO는 디스크가 2개(300GB/500GB)입니다. `rootDeviceHints`에 **OS용 300GB만** 지정해 500GB를 빈 상태로 남겨야 §3.8.1의 LVMS가 동작합니다. **잘못 지정하면 §3.9에서 막히고 SNO 재설치가 필요합니다.**

**5) 실행 — ISO 생성 및 설치**

⚠️ `openshift-install`은 두 yaml을 **소비하며 삭제**하므로 반드시 백업 후 실행하세요.

```bash
mkdir -p ~/ocp-sno-backup && cp install-config.yaml agent-config.yaml ~/ocp-sno-backup/

openshift-install agent create image --dir .
# → agent.x86_64.iso 를 SNO에 연결·부팅

openshift-install agent wait-for bootstrap-complete --dir . --log-level=info
openshift-install agent wait-for install-complete   --dir . --log-level=info
```

**6) 확인 및 검증**

```bash
oc get nodes
oc get co        # 전부 AVAILABLE=True / PROGRESSING=False / DEGRADED=False

# 500GB 디스크의 FSTYPE/MOUNTPOINT가 비어 있어야 LVMS 사용 가능
oc debug node/mas-it-sno -- chroot /host lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

<details>
<summary>참고 — imageDigestSources 개념, 디스크 힌트 대안</summary>

`imageDigestSources`/`imageContentSources`는 **부트스트랩 단계**(클러스터가 아직 없을 때)의 이미지 리다이렉션용이고, `ImageDigestMirrorSet`(IDMS)는 **클러스터 생성 후** 적용하는 리소스입니다(§3.7). 둘 다 필요하며 서로를 대체하지 않습니다.

디바이스명이 불확실하면 `deviceName` 대신 크기·식별자 기반 힌트를 쓸 수 있습니다.

```yaml
rootDeviceHints:
  minSizeGigabytes: 200
  # 또는 wwn / serial / model
```

</details>
### 3.7 Airgap 구성 (사용자 계정)

이 단계는 **두 부분**으로 나뉩니다. 순서를 반드시 지키세요.

#### 3.7.1 oc-mirror 생성 리소스 적용 + OperatorHub 기본 소스 비활성화

🔴 `mas configure-airgap`은 **Red Hat Operator용 CatalogSource를 생성하지 않습니다.** 이 단계를 건너뛰면 cert-manager / DRO / Grafana 등 MAS 필수 Operator를 설치할 수 없어 §3.9가 실패합니다.

⚠️ **미검증** — IBM 설치 가이드는 `mas configure-airgap --setup-redhat-catalogs`를 안내하는데 커맨드 레퍼런스의 옵션 목록에는 없습니다. 실제 도움말로 먼저 확인하세요. 플래그가 있으면 §3.7.2에서 추가해 한 번에 처리하고 아래를 생략할 수 있습니다.

```bash
podman run --rm quay.io/ibmmas/cli:23.4.1 mas configure-airgap --help
```

**1) 실행**

```bash
# 기본 OperatorHub 소스(인터넷 대상) 비활성화
oc patch OperatorHub cluster --type json \
  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

# oc-mirror가 생성한 IDMS/ITMS/CatalogSource 적용 (내용 먼저 검토)
ls -la ~/mas-install/mirror/redhat/working-dir/cluster-resources/
oc apply -f ~/mas-install/mirror/redhat/working-dir/cluster-resources/
```

IDMS/ITMS 적용은 **MachineConfig 롤아웃(노드 재시작)** 을 유발합니다. 안정화 후 다음으로 넘어가세요.

**2) 확인 및 검증**

```bash
watch oc get mcp          # UPDATING=False, DEGRADED=False 될 때까지

oc get nodes
oc get imagedigestmirrorset
oc get imagetagmirrorset
oc get catalogsource -n openshift-marketplace          # 모두 READY

# MAS가 요구하는 Red Hat Operator 조회
oc get packagemanifest -n openshift-marketplace | grep -E 'cert-manager|grafana|ibm-data-reporter|ibm-metrics|lvms'
```

> IBM Maximo Operator Catalog는 이 시점에 **없는 것이 정상**입니다 — §3.9의 `mas install`이 생성합니다.

#### 3.7.2 mas configure-airgap 실행

Registry 접근 검증, 전역 Pull Secret, IDMS, 사용자 CA 신뢰를 구성합니다. MachineConfig 롤아웃으로 30~60분 걸릴 수 있습니다.

**1) 실행**

```bash
REGISTRY_CA="$(find "$HOME/quay-install" -name rootCA.pem -type f -print -quit)"
read -s REGISTRY_PASSWORD
export REGISTRY_PASSWORD

podman run -ti --rm \
  -e REGISTRY_PASSWORD \
  -v "$REGISTRY_CA":/mnt/registry-ca.pem:ro,z \
  quay.io/ibmmas/cli:23.4.1 bash -c "
    oc login --token=<ocp-token> --server=https://api.mas-it.itmsg.co.kr:6443 &&
    mas configure-airgap \
      -H registry.mas-it.itmsg.co.kr -P 8443 \
      -u masadmin -p \"\$REGISTRY_PASSWORD\" \
      --ca-file /mnt/registry-ca.pem \
      --no-confirm
  "
unset REGISTRY_PASSWORD
```

**2) 확인 및 검증**

```bash
watch oc get mcp                    # 롤아웃 완료 대기

oc get imagedigestmirrorset

# 전역 Pull Secret에 미러 Registry 항목 추가 확인
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[]'

# 사용자 CA 신뢰 번들
oc get configmap -n openshift-config user-ca-bundle -o yaml | head -20
```

미러 Registry에서 실제로 Pull이 되는지 최종 확인합니다. `ImagePullBackOff`면 IDMS·Pull Secret·CA 중 하나가 잘못된 것입니다.

```bash
oc run test-pull --image=registry.mas-it.itmsg.co.kr:8443/ibmmas/cli:23.4.1 --restart=Never
oc get pod test-pull -w
oc delete pod test-pull
```

### 3.8 스토리지 준비 (RWO + RWX StorageClass) (사용자 계정)

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

SNO의 로컬 디스크를 RWO StorageClass로 제공합니다. `lvms-operator`(채널 `stable-4.20`)는 §2.3의 8번 미러링에 포함되어 있습니다(`imageset-ocp4.20.yml`).

**1) 실행 — 사전 확인**

Operator 조회. 안 보이면 §3.7.1의 CatalogSource 적용 상태를 먼저 확인하세요.

```bash
oc get packagemanifest -n openshift-marketplace | grep -i lvms
```

LVMS는 **비어 있는(파티션·파일시스템 없는) 디스크**가 필요합니다. 500GB의 `FSTYPE`/`MOUNTPOINT`가 비어 있는지 확인하고 그 디바이스명을 아래 `deviceSelector.paths`에 씁니다.

```bash
oc debug node/mas-it-sno -- chroot /host lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

⚠️ 빈 디스크가 없으면 §3.6의 `rootDeviceHints`가 잘못 지정되어 OS가 500GB에 설치된 것입니다 — SNO 재설치가 필요합니다.

**2) 실행 — Operator 설치**

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

> `source`에는 §3.7.1에서 적용된 실제 CatalogSource 이름을 넣습니다 — `oc get catalogsource -n openshift-marketplace`로 확인하세요(`cs-redhat-operator-index` 형태일 수 있습니다).

```bash
oc get csv -n openshift-storage -w     # Succeeded 대기
```

**3) 실행 — LVMCluster 생성**

`deviceSelector.paths`에 1)에서 확인한 빈 디스크를 지정합니다.

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

⚠️ `LVMCluster`의 `apiVersion`(`lvm.topolvm.io/v1alpha1`)은 미검증입니다. 적용 전 `oc explain lvmcluster`로 확인하세요.

**4) 확인 및 검증** — deviceClass 이름이 `vg1`이면 `lvms-vg1`이 생성됩니다.

```bash
oc get storageclass
oc get lvmcluster -n openshift-storage -o jsonpath='{.items[0].status}{"\n"}'
```

#### 3.8.2 RWX — 방식 선택

**⚠️ 두 방식 모두 폐쇄망에서는 추가 이미지 반입이 필요합니다. 인터넷 연결이 살아있는 §2 단계에서 미리 확보해야 합니다.**

---

##### 방식 A — 외부 NFS (이번 배포 채택, Bastion 겸용)

⚠️ Bastion이 **런타임 의존성**이 됩니다 — NFS가 내려가면 MAS의 RWX 볼륨이 끊깁니다. Mirror Registry까지 겸하므로 단일 장애점이고, 프로비저너는 **커뮤니티 프로젝트로 Red Hat/IBM 미지원**입니다.

**1) 실행 — Bastion에 NFS 서버 구성** (root 계정)

```bash
install -d -m 0777 /export/mas-rwx

# 기존 export 보존을 위해 /etc/exports 대신 별도 파일 사용
cat > /etc/exports.d/mas-rwx.exports <<'EOF'
/export/mas-rwx 192.168.0.0/22(rw,sync,no_subtree_check,no_root_squash)
EOF

exportfs -rav
systemctl enable --now nfs-server

firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd}
firewall-cmd --reload
```

> `no_root_squash`는 컨테이너가 root로 쓰기 위해 필요하지만 보안 완화 설정입니다. 사이트 정책에 맞춰 접근 대역을 좁히세요.

**2) 실행 — 프로비저너 이미지 push** (사용자 계정)

§2.3의 10번에서 받아둔 자산을 사용합니다.

```bash
podman load -i ~/mas-install/nfs-provisioner/nfs-subdir-external-provisioner.tar

read -s REGISTRY_PASSWORD
podman login registry.mas-it.itmsg.co.kr:8443 -u masadmin --password-stdin <<< "$REGISTRY_PASSWORD"

podman tag  registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2 \
            registry.mas-it.itmsg.co.kr:8443/sig-storage/nfs-subdir-external-provisioner:v4.0.2
podman push registry.mas-it.itmsg.co.kr:8443/sig-storage/nfs-subdir-external-provisioner:v4.0.2

unset REGISTRY_PASSWORD
```

**3) 실행 — YAML 수정 후 배포**

받은 YAML의 기본값: SA·Deployment 이름 `nfs-client-provisioner`, StorageClass `nfs-client`, namespace `default`, NFS 주소·경로가 `env`와 `volumes` **총 4곳에 중복**.

```bash
cd ~/mas-install/nfs-provisioner
oc create ns nfs-provisioner

sed -i 's/namespace: default/namespace: nfs-provisioner/g' rbac.yaml deployment.yaml
sed -i 's/10\.3\.243\.101/192.168.2.210/g'                 deployment.yaml   # NFS 서버 (2곳)
sed -i 's|/ifs/kubernetes|/export/mas-rwx|g'               deployment.yaml   # NFS 경로 (2곳)
sed -i 's|registry.k8s.io/sig-storage/|registry.mas-it.itmsg.co.kr:8443/sig-storage/|' deployment.yaml

grep -nE 'namespace:|image:|NFS_SERVER|NFS_PATH|server:|path:' deployment.yaml   # 치환 확인

oc apply -f rbac.yaml -f deployment.yaml -f class.yaml
```

**4) 확인 및 검증**

```bash
showmount -e localhost               # /export/mas-rwx 가 보여야 함
oc get pods -n nfs-provisioner
oc get storageclass                  # nfs-client 가 보여야 함
```

Pod가 `CrashLoopBackOff`/권한 오류면 SCC를 부여합니다(⚠️ 필요 여부 미검증).

```bash
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-client-provisioner -n nfs-provisioner
```

<details>
<summary>참고 — 프로비저너 선택지, archiveOnDelete, 제공 테스트 매니페스트</summary>

| 프로비저너 | 이미지 수 | 특징 |
|---|---|---|
| `nfs-subdir-external-provisioner` (채택) | 1개 | 단순, 스냅샷 미지원 |
| `csi-driver-nfs` | 5~6개 | CSI 표준·스냅샷 지원, 반입 부담 큼 |

`class.yaml`의 `archiveOnDelete: "false"`는 **PVC 삭제 시 NFS 데이터도 삭제**합니다. 운영에서는 `"true"`로 두어 보관하는 편이 안전합니다.

프로젝트 제공 테스트 매니페스트로도 확인 가능합니다. 단 `test-pod.yaml`이 외부 이미지를 참조하면 폐쇄망에서 Pull이 실패하므로, 그때는 §3.8.4의 자체 테스트를 쓰세요.

```bash
oc apply -n nfs-provisioner -f test-claim.yaml -f test-pod.yaml
oc get pvc,pod -n nfs-provisioner
ls -la /export/mas-rwx/
oc delete -n nfs-provisioner -f test-pod.yaml -f test-claim.yaml
```

</details>

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

🔴 `platform: none` 설치에서는 내부 Image Registry가 자동 구성되지 않습니다(`managementState: Removed` 또는 스토리지 없음). Manage는 배포 중 **Admin/Server Bundle 이미지를 직접 빌드해 Push**하므로, 저장할 레지스트리가 없으면 배포가 중단됩니다.

**1) 실행**

```bash
# 현재 상태 확인
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.managementState}{"\n"}{.spec.storage}{"\n"}'

# PVC 생성 후 Managed 전환
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

⚠️ **미검증**: `200Gi`는 예시값이며 Manage 워크스페이스 수와 재빌드 이력에 따라 달라집니다. Manage가 내부 Registry를 쓰는지 별도 Registry를 지정하는지도 IBM 공식 문서로 재확인이 필요합니다.

**2) 확인 및 검증**

```bash
oc get pvc -n openshift-image-registry
oc get pods -n openshift-image-registry
oc get co image-registry     # AVAILABLE=True, DEGRADED=False
```

#### 3.8.4 스토리지 검증 — PVC 바인딩 + 쓰기 테스트

`mas install` 전에 두 StorageClass가 실제로 동작하는지 확인합니다.

**1) 실행 — PVC 바인딩**

```bash
oc create ns storage-test

for M in ReadWriteOnce:lvms-vg1:rwo ReadWriteMany:nfs-client:rwx; do
  MODE=${M%%:*}; REST=${M#*:}; SC=${REST%%:*}; NAME=${REST##*:}
  cat <<EOF | oc apply -n storage-test -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-$NAME
spec:
  accessModes: [$MODE]
  resources:
    requests:
      storage: 1Gi
  storageClassName: $SC
EOF
done

oc get pvc -n storage-test        # 둘 다 Bound 여야 함
```

`Pending`이면 원인을 확인합니다 — 프로비저너 미동작, SCC 권한, NFS export 권한 등.

```bash
oc describe pvc -n storage-test test-rwx
```

**2) 실행 — 쓰기 테스트**

NFS는 마운트만 되고 쓰기가 실패하는 경우가 흔하므로 반드시 확인합니다. 이미지 경로는 미러 Registry에 실제 존재하는 것으로 교체하세요(`curl -sk -u <user> https://<registry>:8443/v2/_catalog`).

```bash
cat <<'EOF' | oc apply -n storage-test -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-writer
spec:
  restartPolicy: Never
  containers:
    - name: writer
      image: registry.mas-it.itmsg.co.kr:8443/openshift4/ose-tools-rhel9:latest
      command: ["/bin/sh","-c","echo ok > /data/test.txt && cat /data/test.txt"]
      volumeMounts: [{name: data, mountPath: /data}]
  volumes:
    - name: data
      persistentVolumeClaim: {claimName: test-rwx}
EOF
```

**3) 확인 및 검증**

```bash
oc logs -n storage-test test-writer -f      # ok 출력되면 정상
ls -la /export/mas-rwx/                     # Bastion에서 파일 생성 확인

oc delete ns storage-test                   # 정리
```

### 3.9 MAS Core 설치 (사용자 계정)

**1) 실행**

```bash
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)

podman run -ti --rm \
  -e IBM_ENTITLEMENT_KEY \
  -v "$HOME":/mnt/home:z \
  quay.io/ibmmas/cli:23.4.1 bash -c "
    oc login --token=<ocp-token> --server=https://api.mas-it.itmsg.co.kr:6443 &&
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
| 도메인 | 기본 서브도메인 또는 커스텀 도메인 | `apps.mas-it.itmsg.co.kr` 기반 |
| 고급 | **Admin Mode** | 관리자 전용 접근 모드 여부 |
| 고급 | **Network Routing Mode** | Route 노출 방식 |
| 고급 | SSO / Guided Tours 등 | 환경 정책에 따라 |
| 앱 선택 | **설치할 애플리케이션** | **Manage 선택** (Assist/IoT/Monitor/Optimizer/Predict/VI 등은 선택하지 않음) |
| DB | 데이터베이스 | **내장 Db2 자동 프로비저닝 선택** |
| 기타 | Pod QoS 등 | 기본값 권장 |

클러스터에 airgap용 `ImageDigestMirrorSet`이 이미 있으면 `mas install`이 이를 감지해 폐쇄망 설치 흐름으로 자동 전환됩니다.

> `mas install`은 마지막에 **동일 설치를 재현할 수 있는 비대화형 명령**을 출력합니다. 재설치·다른 사이트 배포를 위해 **반드시 보관**하세요(비밀번호는 마스킹 후).

⚠️ **미검증**: 위 항목 목록은 공식 install 가이드 기준이며 CLI 23.4.1의 실제 프롬프트와 순서·명칭이 다를 수 있습니다. 특히 `Pipeline Storage`, `Operational Mode`의 선택지는 실행 화면에서 확인하세요.

**2) 확인 및 검증**

```bash
oc get suite -A
oc get subscriptions -A
oc get catalogsource -n openshift-marketplace
oc get pods -n mas-inst1-core | grep -v Running | grep -v Completed
oc get route -A | grep -E 'admin|mas'
```

### 3.10 Manage + Maximo IT 배포·활성화 (웹 UI, Suite Administration 관리자 계정)

⚠️ **이 절은 검증 수준이 가장 낮습니다.** IBM 지식센터 원문을 이 문서 작성 환경에서 직접 열지 못해 검색 스니펫 기준으로 작성했습니다. **진행 전 반드시 아래 공식 문서를 브라우저로 직접 열어 대조하세요.**
>
> - Maximo IT 배포: <https://www.ibm.com/docs/en/max-it/cd.0.0_cd?topic=it-deploying-maximo-maximo-manage>
> - Maximo IT 라이선스: <https://www.ibm.com/docs/en/max-it/cd.0.0_cd?topic=suite-licensing-maximo-it-in-maximo-application>

**1) 실행 — 기본 흐름**

1. Suite Administration(`https://admin.inst1.apps.mas-it.itmsg.co.kr`)에 §3.9의 Superuser로 로그인
2. Catalog에서 **Manage** 선택
3. **버전 선택** — ⚠️ 아래 "버전 호환성" 참고
4. Workspace **Components**에서 **Maximo IT(ICD) add-on** 활성화
5. **데이터베이스 구성** — JDBC 연결, schema, tablespace (⚠️ 아래 "한국어 사용 시" 참고)
6. **언어 설정** — 기본 언어 및 추가 언어 선택 (⚠️ 설치 후 변경이 어려우므로 이 단계에서 확정)
7. **Server Bundle 구성** — 워크로드 분리 방식(`all`, `snd` 등) 선택
8. **Activate Manage** 실행 → 이미지 빌드·배포 진행 (내부 Image Registry 사용, §3.8.3)
9. 활성화 완료 후 **관리자 권한 동기화 및 재로그인**
10. Maximo IT 애플리케이션 접근 확인

**2) 확인 및 검증** — 진행 전 아래 항목을 확정하세요.

| 항목 | 내용 | 근거 |
|---|---|---|
| **Db2 테이블 구성 (ROW)** | 🔴 **Manage는 row-organized 테이블을 요구합니다.** Db2가 column-organized(기본값이 그럴 수 있음)로 설정되어 있으면 Manage가 정상 동작하지 않습니다. `DFT_TABLE_ORG`를 `ROW`로 설정해야 합니다. | ✅ 공식(MAS Performance Wiki, ansible-devops db2 role) |
| **Db2 워크로드 설정** | `db2_workload=maximo` 레지스트리 설정 시 `WLM_ADMISSION_CTRL`이 `NO`로 자동 구성됩니다. ansible-devops db2 role 기준 Manage용 권장값은 `db2_table_org: ROW`. | ✅ 공식 |
| **Db2 디스크 성능** | 권장 **처리량 250MB/s 이상**, **10~100 IOPS/GB**. 시스템/사용자/백업/트랜잭션 로그/임시 테이블스페이스를 **분리된 디스크**에 두는 것을 권장. | ✅ 공식 |
| **버전 호환성** | Manage 버전과 Maximo IT(ICD) 버전은 **호환되는 조합**이어야 합니다. `latest`와 특정 버전을 **혼용하지 마세요** — 한쪽만 `latest`면 이후 업데이트에서 조합이 깨집니다. 양쪽 모두 명시적 버전 고정 권장. | ⚠️ 미검증 |
| **한국어 사용 시 VARGRAPHIC** | 🔴 한국어 등 멀티바이트 언어 사용 시 Maximo가 **VARGRAPHIC** 컬럼 타입을 쓰도록 설정해야 할 수 있습니다. **DB 생성/Manage 활성화 시점에 결정되며 이후 변경이 매우 어렵습니다.** | ⚠️ **미검증 — 반드시 확인** (아래 참고) |
| **언어 설정** | 활성화 시 기본 언어와 추가 언어를 지정합니다. 추가 언어는 DB 크기와 활성화 시간을 늘립니다. | ⚠️ 미검증 |
| **Server Bundle 구성** | **UI / CRON / MIF(Integration) / Report 워크로드를 별도 번들로 분리**하면 동일 JVM 내 리소스 경합이 줄어듭니다. `ManageWorkspace` CR에서 번들별 replica·CPU·메모리를 조정합니다. | ✅ 공식(Performance Wiki) |
| **관리자 권한 동기화** | 활성화 직후 Maximo IT 메뉴가 안 보일 수 있습니다. 권한 동기화 후 **로그아웃 → 재로그인** 필요. | ⚠️ 미검증 |
| **활성화 후 설정** | 활성화 완료가 끝이 아닙니다 — 사용자/그룹 권한, AppPoints 할당, IT 초기 설정이 이어집니다. | ⚠️ 미검증 |

> ⚠️ "Server Bundle 분리"와 "Db2 테이블스페이스 디스크 분리"는 다중 노드/전용 스토리지 전제의 성능 권장사항입니다. SNO 단일 노드에서는 기본 구성으로 시작하세요.

🔴 **VARGRAPHIC과 AppPoints는 진행 전에 확정하세요.** 둘 다 나중에 되돌리기 어렵습니다.

| 확인할 것 | 왜 |
|---|---|
| 한국어 사용 시 VARGRAPHIC 설정 **위치** (Db2 레벨 vs Manage 활성화 레벨) | ansible-devops `db2` role에 문자셋 전용 변수가 없어 어디서 설정하는지 미확정. DB 생성/활성화 시점에 결정되면 이후 변경이 매우 어려움 |
| 라이선스에 **Maximo IT용 AppPoints 항목**이 실제로 포함되어 있는지 | MAS Entitlement와 Maximo IT Entitlement는 **각각** 필요. 없으면 Manage는 배포되지만 IT add-on 활성화 또는 사용자 할당에서 막힘 |

참고 문서:

- [Db2 configuration — MAS](https://www.ibm.com/docs/en/masv-and-l/cd?topic=deployment-configuring-db2)
- [Database configuration details for Maximo Manage](https://www.ibm.com/docs/en/masv-and-l/cd?topic=install-database-configuration-details-maximo-manage)
- [MAS Performance Wiki — Manage best practice](https://ibm-mas.github.io/mas-performance/mas/manage/bestpractice/)
- [ansible-devops db2 role](https://ibm-mas.github.io/ansible-devops/roles/db2/)

### 3.11 설치 후 확인 (사용자 계정)

**1) 확인 및 검증**

```bash
oc get nodes
oc get co
oc get mcp
oc get suite -A
oc get manageapp -A
oc get manageworkspace -A
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

## 4. 트러블슈팅

### 4.1 `:Z` vs `:z` SELinux 라벨 충돌

**증상**: 미러링을 여러 터미널에서 동시에 실행하면 로그 파일 쓰기 단계에서 `Permission denied`로 실패.

**원인**: 두 컨테이너가 같은 디렉터리를 각자 `-v ...:Z`(대문자, 배타적 라벨)로 마운트해서 MCS 카테고리가 충돌. 나중에 시작한 컨테이너가 재라벨링하면 먼저 실행 중인 컨테이너의 접근 권한이 무효화됩니다.

```bash
sudo grep avc /var/log/audit/audit.log | tail -30
```
```
avc: denied { write } for ... scontext=...:s0:c280,c427  tcontext=...:s0:c285,c450
```

**해결**: `:Z` 대신 **`:z`**(소문자, 공유 라벨)를 사용합니다. 이 문서의 모든 `podman run`은 `:z`로 되어 있습니다.

### 4.2 MAS 콘텐츠 미러링 중 `cp.icr.io` OAuth 토큰 타임아웃

**증상**: `mas mirror-images` 실행 중 `Fail if mirror is not successful` 태스크에서 실패. 로그에 아래 에러.

```
error: unable to push cp.icr.io/cp/manage/aip-optimizer: failed to retrieve blob ...:
Get "https://cp.icr.io/oauth/token?...": net/http: request canceled (Client.Timeout exceeded)
```

**원인**: IBM 레지스트리의 OAuth 토큰 발급이 일시적으로 타임아웃. `oc image mirror`는 이미지 하나라도 실패하면 전체를 실패로 처리합니다.

**해결**: 받은 콘텐츠는 `mirror/<stage>/v2`에 남아 있고 재실행 시 건너뜁니다. **디렉터리를 비우지 말고 같은 명령을 다시 실행**하세요.

### 4.3 Red Hat 콘텐츠 미러링 — 타임아웃과 캐시 소실

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

### 4.4 SSH 접속을 끊으면 미러링 컨테이너가 죽음

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

### 4.5 `from-filesystem`이 tar를 못 찾음 — 파일명 불일치

**증상**: §3.5의 Red Hat 콘텐츠 push가 tar 파일을 찾지 못해 실패.

**원인**: IBM CLI 23.4.1의 `mirror_ocp` 역할이 아래처럼 파일명을 하드코딩하고 있습니다.

```
--from file://{{ mirror_working_dir }}/mirror_seq1_000000.tar
```

그런데 같은 이미지에 내장된 oc-mirror(4.22.0)는 산출물을 **`mirror_000001.tar`** 로 만듭니다. 래퍼로만 진행해도 to-filesystem과 from-filesystem의 파일명이 어긋나므로 **IBM CLI 자체의 버전 불일치**입니다.

`to-filesystem.yml`은 파일명을 전혀 다루지 않고 `oc mirror`를 그대로 호출합니다. 즉 산출물 이름은 oc-mirror가 정하며, 두 역할이 서로 어긋나 있는 것입니다.

**해결**: §3.5의 2)에서 파일명을 바꿉니다. **§2.5의 체크섬 검증을 마친 뒤에** 하세요 — 먼저 바꾸면 `transfer-files.sha256`과 어긋납니다.

```bash
cd ~/mas-install/mirror/redhat
mv mirror_000001.tar mirror_seq1_000000.tar
```

역할 파일을 고쳐 쓰는 방법도 있습니다(폐쇄망에서도 가능). 다만 §3.5를 실행할 때마다 마운트를 붙여야 해서 이름을 바꾸는 편이 간단합니다.

```bash
R=/opt/app-root/lib64/python3.12/site-packages/ansible_collections/ibm/mas_devops/roles/mirror_ocp/tasks/actions/from-filesystem.yml
podman run --rm quay.io/ibmmas/cli:23.4.1 cat $R > ~/from-filesystem.yml
sed -i 's|mirror_seq1_000000.tar|mirror_000001.tar|' ~/from-filesystem.yml
# 이후 podman run 에 -v ~/from-filesystem.yml:$R:ro,z 추가
```

**참고** — 역할이 실제로 실행하는 명령입니다. 래퍼를 우회할 때 그대로 쓰면 됩니다.

```
DOCKER_CONFIG=<dir> oc mirror --remove-signatures -c <dir>/imageset-ocp4.20.yml \
  --dest-tls-verify=false --parallel-images=1 \
  --from file://<dir>/mirror_seq1_000000.tar docker://<registry> --v2
```
