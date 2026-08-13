# Maximo IT — Java 커스터마이징 개발 환경

MAS 9.2 / Manage 9.2.0(`base` + `icd`) 환경에서 필드 클래스·Business Object를 Java로 작성하기 위한 개발 환경 구성 절차입니다. Admin·Server Bundle 이미지를 개발 PC로 가져와 로컬에서 Manage를 기동하고, 완성된 변경은 Customization Archive로 실제 MAS에 반영합니다.

**개발 PC에는 JDK 25가 필요합니다.** Manage 런타임이 Semeru 25.0.3(OpenJ9)이고 제품 jar의 클래스 파일 major가 69(=Java 25)라, 하위 버전 JDK로는 `businessobjects.jar` 를 읽지 못해 컴파일이 시작되지 않습니다. JRE가 아니라 **JDK**여야 `javac` 가 포함됩니다. IBM Semeru Runtime Open Edition 25(런타임과 동일 계열) 또는 Eclipse Temurin 25 등 OpenJDK 25 계열을 씁니다.

```powershell
java -version; javac -version; $env:JAVA_HOME
```

## 참조 문서

| # | 문서 | 내용 | 관련 절 |
|---|---|---|---|
| 1 | [Setting up a local Maximo Manage development environment](https://www.ibm.com/docs/en/masv-and-l/maximo-manage/cd?topic=developing-setting-up-local-maximo-manage-development-environment) | 로컬 개발환경 전체 개요. Podman 기반으로 Java/JSP를 로컬에서 개발·테스트하는 방식과 Admin / ServerBundle 이미지 구조 | 전체 |
| 2 | [Preparing a local development environment](https://www.ibm.com/docs/en/masv-and-l/maximo-manage/cd?topic=developing-preparing-local-development-environment) | OpenShift 내부 Registry에서 Admin·ServerBundle 이미지를 pull / tag | §1, §2 |
| 3 | [Building and deploying development images](https://www.ibm.com/docs/en/masv-and-l/maximo-manage/cd?topic=environment-building-deploying-development-images) | 로컬 `applications` 변경 파일을 IBM 제공 Dockerfile로 반영해 `managedev` 이미지 생성·실행. DB 연결정보와 `CRYPTO`/`CRYPTOX` 키 포함 | §3, §4 |
| 4 | [Creating customization archives](https://www.ibm.com/docs/en/masv-and-l/cd?topic=archives-creating-customization) | Customization Archive 구조와 생성 방법. `/SMP/maximo` 구조에 맞춘 ZIP 구성 | §5 |
| 5 | [Adding customizations](https://www.ibm.com/docs/en/masv-and-l/maximo-manage/cd?topic=customizing-adding-customizations) | Archive를 Manage가 접근 가능한 위치에 배치하고 MAS 관리화면에서 URL 등록 후 Apply Changes | §5, §6 |

---

## 1. Admin 이미지 pull & `/opt/IBM/SMP` 디렉터리 생성

Admin 이미지는 EAR을 빌드하는 쪽이며 개발 소스(`/opt/IBM/SMP/writeable/maximo`)가 들어 있습니다. Bastion에서 받습니다.

### 1.1 대상 확인

```bash
export KUBECONFIG=/home/maximo/ocp-sno/auth/kubeconfig
oc get is -n mas-inst1-manage
```

| ImageStream | 태그 |
|---|---|
| `inst1-ws1-admin` | `20260807T104658-9.2.197`, `latest` |

### 1.2 레지스트리 연결

내부 레지스트리는 클러스터 안에서만 보이므로 포트 포워딩으로 붙습니다. Route를 노출하지 않아 클러스터 설정을 건드리지 않습니다.

```bash
oc login -u kubeadmin -p oRYob-UVjUs-AI492-mjDsc https://api.mas-it.itmsg.co.kr:6443
oc port-forward -n openshift-image-registry svc/image-registry 5000:5000 &
podman login -u kubeadmin -p $(oc whoami -t) localhost:5000 --tls-verify=false
```

설치용 `kubeconfig` 는 인증서 기반이라 토큰이 없습니다. `oc login` 을 먼저 해야 `oc whoami -t` 가 값을 반환합니다.

### 1.3 pull

```bash
TAG=20260807T104658-9.2.197
podman pull localhost:5000/mas-inst1-manage/inst1-ws1-admin:$TAG --tls-verify=false
podman images | grep inst1-ws1-admin
```

**13.2 GB**입니다. 받는 데 시간이 걸리며 `Handling connection for 5000` 이 계속 찍히는 것은 포트 포워딩이 동작 중이라는 뜻입니다.

### 1.4 개발 소스(`/opt/IBM/SMP`) 확보

이미지를 기동하지 않고 파일만 꺼냅니다. 기본 진입점이 maxinst(DB 초기화)라 실행하면 DB에 붙으려 하므로 `podman create` 로 생성만 합니다.

크기를 먼저 확인합니다.

```bash
IMG=localhost:5000/mas-inst1-manage/inst1-ws1-admin:$TAG
podman run --rm --entrypoint /bin/sh $IMG -c 'du -sh /opt/IBM/SMP /opt/IBM/SMP/maximo /opt/IBM/SMP/writeable'
```

```bash
mkdir -p ~/manage-src && cd ~/manage-src
CID=$(podman create $IMG)
podman cp $CID:/opt/IBM/SMP ./SMP
podman rm $CID
du -sh SMP
```

확보되는 구조입니다. 🔴 **실물은 `writeable/` 아래에 있습니다.**

```text
SMP/
├─ maximo/
│  └─ tools/                        ← 이미지에는 이것만 있음 (기동 시 채워짐)
└─ writeable/maximo/                ← 실제 Maximo 설치 트리
   ├─ applications/
   │  ├─ maximo/
   │  │  ├─ businessobjects/        psdi, com — 제품 클래스 / 커스텀 클래스 위치
   │  │  ├─ maximouiweb/            JSP·웹 리소스
   │  │  ├─ lib/                    의존 jar
   │  │  ├─ properties/
   │  │  └─ buildfoundation.xml, basebuild.cmd   EAR 빌드 스크립트
   │  ├─ maximohelp/  graphite/  mmstandalone/  mxiehs/  maximormiregistry/
   ├─ deployment/                   배포 서술자
   ├─ tools/
   ├─ maxinst.sh  prepare-db.sh  cos.sh  reencrypt.sh  …   운영 스크립트
   └─ manage_build_list.txt  maxfoundation_build_list.txt  빌드 목록
```

IDE classpath와 아카이브 디렉터리 구조의 기준은 모두 `SMP/writeable/maximo/applications/maximo/` 입니다.

### 1.5 개발 PC로 전송

`businessobjects/classes` 아래에 작은 파일이 수만 개라 그대로 `scp` 하면 매우 느립니다. tar로 묶어서 보냅니다.

```bash
tar cf SMP.tar SMP
ls -lh SMP.tar
```

```powershell
scp maximo@192.168.2.210:~/manage-src/SMP.tar <로컬 경로>\SMP.tar
tar xf <로컬 경로>\SMP.tar -C <로컬 경로>
```

## 2. Server Bundle 이미지 pull

Liberty 런타임 쪽입니다. Admin이 만든 EAR을 얹어 실제로 실행하는 베이스이며 **§3~4 로컬 실행에 사용합니다.**

### 2.1 pull

§1.2의 포트 포워딩과 `podman login` 이 살아 있으면 pull만 하면 됩니다.

⚠️ **포트 포워딩은 백그라운드 작업이라 셸을 닫으면 함께 종료됩니다.** `connection refused` 가 나면 §1.2를 다시 실행하세요.

```bash
oc port-forward -n openshift-image-registry svc/image-registry 5000:5000 &
sleep 2
podman login -u kubeadmin -p $(oc whoami -t) localhost:5000 --tls-verify=false
```

```bash
podman pull localhost:5000/mas-inst1-manage/inst1-ws1-all:$TAG --tls-verify=false
podman images | grep inst1-ws1
```

### 2.2 내용 확인

Admin 이미지에서 SMP 실물이 `writeable/` 아래에 있었던 것처럼, 이미지에 무엇이 들어 있는지는 직접 확인합니다.

```bash
ALLIMG=localhost:5000/mas-inst1-manage/inst1-ws1-all:$TAG
podman run --rm --entrypoint /bin/sh $ALLIMG -c \
  'ls -la /opt/ibm/wlp/usr/servers/defaultServer/apps/; ls -d /opt/*'
```

| 항목 | 값 |
|---|---|
| 이미지 크기 | **5.85 GB** (Admin은 13.2 GB) |
| `apps/maximo-all.ear` | **2,280,694,291 바이트** — 실행 중인 파드와 동일 |
| `/opt` 구성 | `criu`, `ibm`(Liberty `wlp`), `java`(Semeru 25.0.3) |

**EAR은 이미지에 구워져 있습니다.** Admin 이미지와 달리 기동 시점에 채워지는 부분이 없어, 이 이미지만으로 Liberty 기동이 가능합니다.

### 2.3 개발 PC로 내려받기

로컬에서 실행하려면 **두 이미지 모두 개발 PC에 있어야 합니다** (Admin 13.2 GB + Server Bundle 5.85 GB = 약 19 GB).

#### 방법 A — 개발 PC에서 직접 pull (권장)

Bastion을 거치지 않아 전송이 한 번으로 끝납니다. 개발 PC에 `oc` 가 필요하며, 인터넷이 되므로 바로 받을 수 있습니다.

```powershell
# oc 설치 후
oc login -u kubeadmin -p <비밀번호> https://api.mas-it.itmsg.co.kr:6443
Start-Process oc -ArgumentList 'port-forward','-n','openshift-image-registry','svc/image-registry','5000:5000'

$TAG = '20260807T104658-9.2.197'
docker login -u kubeadmin -p $(oc whoami -t) localhost:5000
docker pull localhost:5000/mas-inst1-manage/inst1-ws1-admin:$TAG
docker pull localhost:5000/mas-inst1-manage/inst1-ws1-all:$TAG
docker images | Select-String inst1-ws1
```

`api.mas-it.itmsg.co.kr` 은 `hosts` 에 이미 등록돼 있습니다. Docker는 `localhost` 레지스트리를 기본적으로 insecure로 취급하지만, 인증서 오류가 나면 Settings → Docker Engine 에 추가합니다.

```json
{ "insecure-registries": ["localhost:5000"] }
```

#### 방법 B — Bastion에서 tar로 옮기기

개발 PC에 `oc` 를 설치하지 않는 경우입니다. Bastion에 19 GB 임시 공간이 추가로 필요합니다.

```bash
cd ~/manage-src
podman save -o admin.tar localhost:5000/mas-inst1-manage/inst1-ws1-admin:$TAG
podman save -o all.tar   localhost:5000/mas-inst1-manage/inst1-ws1-all:$TAG
ls -lh admin.tar all.tar
```

```powershell
scp maximo@192.168.2.210:~/manage-src/admin.tar <로컬 경로>\admin.tar
scp maximo@192.168.2.210:~/manage-src/all.tar   <로컬 경로>\all.tar
docker load -i <로컬 경로>\admin.tar
docker load -i <로컬 경로>\all.tar
```

### 2.4 포트 포워딩 정리

이미지를 다 받았으면 백그라운드 포워딩을 종료합니다.

```bash
kill %1
```

## 3. 이미지 load & Manage 기동

### 3.1 이미지 load

```powershell
docker load -i C:\Source\mas\all.tar
docker load -i C:\Source\mas\admin.tar
docker images | Select-String inst1-ws1
```

Docker Desktop이 실행 중이어야 합니다. 꺼져 있으면 `failed to connect to the docker API at npipe:…dockerDesktopLinuxEngine` 이 납니다.

| 이미지 | tar | Docker 적재 후 |
|---|---|---|
| `inst1-ws1-admin` | 12.3 GB | **26.8 GB** |
| `inst1-ws1-all` | 5.45 GB | **11.7 GB** |

레이어 압축이 풀리면서 커집니다. 두 이미지 합계로 약 39 GB의 디스크가 필요합니다.

### 3.2 이미지 tag

Dockerfile이 `manage-admin-dev` / `manage-<BUNDLE>-dev` 라는 이름을 참조하므로 받아온 이미지를 그 이름으로 태그합니다.

```powershell
$SRC='localhost:5000/mas-inst1-manage'; $TAG='20260807T104658-9.2.197'
docker tag "$SRC/inst1-ws1-admin:$TAG" manage-admin-dev:latest
docker tag "$SRC/inst1-ws1-all:$TAG"   manage-all-dev:latest
```

### 3.3 빌드 컨텍스트 구성

```text
C:\Source\mas\mas-deployment\
├─ applications\          커스터마이징 투입 위치 (처음에는 비어 있음)
└─ manage-developer\
   └─ Dockerfile
```

### 3.4 기동에 필요한 값

실행 중인 파드의 환경변수를 기준으로 로컬용 값을 정리합니다.

```bash
oc get pod -n mas-inst1-manage $POD -o jsonpath='{range .spec.containers[?(@.name=="all")].env[*]}{.name}{" = "}{.value}{"  [secret:"}{.valueFrom.secretKeyRef.name}{"/"}{.valueFrom.secretKeyRef.key}{"]"}{"\n"}{end}'
```

**그대로 쓰는 값**

```
MXE_DB_SCHEMAOWNER=maximo     MXE_DB_DRIVER=com.ibm.db2.jcc.DB2Driver
MXE_MAS_INSTANCEID=inst1      MXE_MAS_WORKSPACEID=ws1
MXE_BUNDLE_TYPE=all           TZ=Asia/Seoul
```

**바꾸는 값**

| 변수 | 클러스터 | 로컬 |
|---|---|---|
| `MXE_DB_URL` | `c-mas-inst1-ws1-manage-db2u-engn-svc.db2u.svc:50001` | `192.168.2.211:31899` (NodePort 평문) |
| `DB_SSL_ENABLED` | `ssl` | 해제 |
| `APP_URL` | `https://manage.inst1.apps.mas-it.itmsg.co.kr` | `http://localhost:9080` |
| `MXE_USEAPPSERVERSECURITY` | `1` (Liberty OIDC) | `0` — Maximo 자체 로그인 |

**제외하는 값** — `*.svc` / `*.svc.cluster.local` 은 클러스터 안에서만 해석되므로 로컬에서는 닿지 않습니다.

```
AUTH_TOKEN_URL  VALIDATION_ENDPOINT_URL  JWK_ENDPOINT_URL
MXE_MASINTERNALAPI  MXE_MASLICENSINGAPI  MXE_MASADOPTIONUSAGEAPI_*
MXE_MREFSERVICEAPI  MXE_MASINTERNALPUSHNOTIFAPI
```

**암호화 키** — 같은 Db2를 쓰므로 클러스터와 **반드시 동일한 키**여야 합니다. 다르면 암호화 컬럼을 복호화하지 못해 로그인에서 막힙니다.

```bash
oc get secret -n mas-inst1-manage -o json > /tmp/sec.json
python3 - <<'EOF'
import json, base64
for s in json.load(open('/tmp/sec.json'))['items']:
    for k, v in s.get('data', {}).items():
        if 'crypt' in k.lower():
            print('%-45s %-28s %s' % (s['metadata']['name'], k, base64.b64decode(v).decode()))
EOF
```

시크릿 `ws1-manage-encryptionsecret` / `ws1-manage-encryptionsecret-operator` 에 있습니다. 신규 설치라 `OLD` 키와 현재 키가 같습니다.

| 변수 | 값 |
|---|---|
| `MXE_SECURITY_CRYPTO_KEY` | `IAshvBJAkPArfYMRcgKPKDEu` |
| `MXE_SECURITY_OLD_CRYPTO_KEY` | `IAshvBJAkPArfYMRcgKPKDEu` |
| `MXE_SECURITY_CRYPTOX_KEY` | `dceVSktsCyUZWOiEpEXmChpb` |
| `MXE_SECURITY_OLD_CRYPTOX_KEY` | `dceVSktsCyUZWOiEpEXmChpb` |

### 3.5 Dockerfile

IBM이 Dockerfile을 **RTF로 압축해 배포**합니다. 받아서 텍스트로 변환한 뒤 사용합니다.

```
https://www.ibm.com/docs/en/SSLPL8_cd/com.ibm.mam.doc/config/dockerfile.zip
```

🔴 **원본을 그대로 쓰면 안 됩니다.** 원본은 경로를 `/opt/IBM/SMP/maximo/…` 로 쓰는데, 실제 Admin 이미지의 실물은 **`/opt/IBM/SMP/writeable/maximo/…`** 아래에 있습니다(§1.4). 모든 `WORKDIR` 을 `writeable` 경로로 바꿔야 합니다.

원본과 달라지는 지점을 정리하면 이렇습니다.

| 항목 | 원본 | 이 환경 |
|---|---|---|
| SMP 경로 | `/opt/IBM/SMP/maximo` | **`/opt/IBM/SMP/writeable/maximo`** |
| `FROM` | `localhost/manage-…-dev` | `manage-admin-dev:latest` / `manage-all-dev:latest` |
| `server.xml` 교체 | 1단계(Admin) | **2단계(ServerBundle)의 `/opt/ibm/wlp/usr/servers/defaultServer`** |
| `web.xml` 교체 대상 | `maximo-x` 포함 | `maximo-foundation/mboweb`·`meaweb` 포함, `maximo-x` 없음 |

`web.xml` 을 `web-dev.xml` 로 바꾸는 대상은 다음 11개입니다.

```text
maximo-all/{maximouiweb, maxrestweb, mboweb, meaweb}
maximo-foundation/{mboweb, meaweb}
maximo-cron
maximo-mea/{maxrestweb, meaweb}
maximo-report
maximo-ui
```

제품이 운영용(OIDC)과 개발용(dev) 서술자를 **둘 다 이미지에 넣어두므로**, 이름만 바꿔치기하면 MAS SSO 없이 Maximo 자체 로그인으로 뜹니다.

환경변수 블록은 이 환경 값으로 채웁니다.

```dockerfile
ENV MXE_DB_URL='jdbc:db2://192.168.2.211:31899/BLUDB'
ENV MXE_DB_SCHEMAOWNER=maximo
ENV MXE_DB_DRIVER='com.ibm.db2.jcc.DB2Driver'
ENV MXE_DB_USER='db2inst1'
ENV MXE_DB_PASSWORD='X5D4gNDiRlMsPUu'

ENV MXE_SECURITY_CRYPTO_KEY='IAshvBJAkPArfYMRcgKPKDEu'
ENV MXE_SECURITY_OLD_CRYPTO_KEY='IAshvBJAkPArfYMRcgKPKDEu'
ENV MXE_SECURITY_CRYPTOX_KEY='dceVSktsCyUZWOiEpEXmChpb'
ENV MXE_SECURITY_OLD_CRYPTOX_KEY='dceVSktsCyUZWOiEpEXmChpb'
```

`MXE_DB_URL` 은 NodePort 평문이라 `sslConnection` 을 붙이지 않습니다. 클러스터 값(`c-mas-…db2u.svc:50001` + SSL)과 다른 이유입니다.

### 3.6 빌드 및 기동

```powershell
cd C:\Source\mas\mas-deployment
docker build --progress=plain --build-arg BUNDLE=all -t managedev:latest -f .\manage-developer\Dockerfile .
docker images | findstr managedev
```

1단계에서 `maximo-all.sh` 가 EAR(약 2.1 GB)을 새로 굽기 때문에 시간이 걸립니다. `--progress=plain` 을 주면 진행 상황이 보입니다.

```powershell
docker run -d --name maximo-dev -p 9080:9080 managedev:latest
docker ps
docker logs -f maximo-dev
```

접속 주소입니다.

```
http://localhost:9080/maximo
```



## 4. 소스 변경 및 테스트

커스터마이징을 로컬에 반영하고 확인하는 반복 루프입니다. 컴파일 방법은 §5~6을 보세요.

### 4.1 변경 파일 배치

빌드 컨텍스트의 `applications` 가 Admin 이미지의 SMP 위에 덮어써집니다(`ADD applications /opt/IBM/SMP/writeable/maximo/applications`). 따라서 **제품과 동일한 디렉터리 구조**로 넣어야 합니다.

```text
C:\Source\mas\mas-deployment\applications\maximo\
├─ businessobjects\classes\<패키지>\*.class     Java 클래스
└─ maximouiweb\webmodule\...                    JSP·웹 리소스
```

### 4.2 기존 컨테이너 제거

```powershell
docker rm -f maximo-dev
```

### 4.3 빌드

```powershell
cd C:\Source\mas\mas-deployment
docker build --build-arg BUNDLE=all -t managedev:latest -f .\manage-developer\Dockerfile .
docker images | findstr managedev
```

빌드가 끝나면 `managedev:latest` 의 IMAGE ID가 바뀌어 있어야 합니다. 태그가 같으므로 이전 이미지는 `<none>` 으로 남습니다.

### 4.4 실행

```powershell
docker run -d --name maximo-dev -p 9080:9080 managedev:latest
docker logs -f maximo-dev
```

```
http://localhost:9080/maximo
```

빌드마다 EAR을 다시 굽기 때문에 한 번에 수 분 이상 걸립니다. 클래스 하나를 고칠 때마다 전체 루프를 도는 것보다, **여러 변경을 모아서 한 번에 반영**하는 편이 효율적입니다.

### 4.5 중지

```powershell
docker stop maximo-dev
```

다시 띄울 때는 `docker start maximo-dev` 로 재사용하고, 소스를 바꿨다면 4.2부터 다시 진행합니다.

```powershell
docker start maximo-dev
docker logs -f maximo-dev
```

컨테이너와 이미지를 모두 정리하려면 다음과 같습니다.

```powershell
docker rm -f maximo-dev
docker rmi managedev:latest
```

## 5. 소스 게시 및 MAS 배포

§4에서 로컬 검증까지 끝난 변경을 실제 MAS에 반영합니다.

MAS Manage는 Customization Archive를 **URL로 내려받습니다.** 파일 업로드가 아니라 클러스터가 접근 가능한 HTTP 주소를 등록하는 방식입니다. 폐쇄망이라 외부 스토리지를 쓸 수 없으므로 Bastion을 배포 지점으로 씁니다.

```text
applications 배치 → ZIP 압축 → Bastion 업로드 → MAS에 URL 등록 → Apply Changes
                                                                      │
                                              Server Bundle 재빌드 ◀──┘
                                                      │
                                              Workspace 활성화 → 기능 확인
```

배포 서버(nginx) 구성은 [OFFLINE_INSTALL.md §4](../installation/OFFLINE_INSTALL.md#4-커스터마이징-배포-서버-구성)에서 끝났다고 가정합니다.

| 항목 | 값 |
|---|---|
| 배포 디렉터리 | `/mas/mas-deployment` (`maximo` 소유) |
| URL 형식 | `http://192.168.2.210:8080/<파일명>` |

⚠️ 클래스 하나 고칠 때마다 이 절을 도는 것은 비효율적입니다. Server Bundle 이미지를 통째로 다시 굽기 때문에 **한 번에 20~30분** 걸립니다. §4의 로컬 컨테이너에서 충분히 검증한 뒤 **여러 변경을 모아서** 올리세요.

### 5.1 배포 파일 배치

`applications` 아래에 `.class`·JSP·XML을 **제품과 동일한 디렉터리 구조**로 놓습니다. §4.1과 같은 트리이므로, 로컬 검증에 쓰던 것을 그대로 씁니다.

```text
applications
└─ maximo
   ├─ businessobjects
   │  └─ classes
   │     └─ com
   │        └─ AssetBean.class
   └─ maximouiweb
      └─ webmodule
         └─ WEB-INF
            └─ classes
               └─ com
                  └─ TestWeb.class
```

`.java` 는 넣지 않습니다. 컴파일된 `.class` 만 들어갑니다.

### 5.2 ZIP 압축

🔴 **ZIP을 열었을 때 최상위가 `applications` 여야 합니다.** 상위 폴더(`mas-deployment`)를 압축하면 경로가 한 단계 어긋나 빌드는 성공하는데 반영은 되지 않습니다 — 원인을 찾기 가장 어려운 형태의 실패입니다.

```text
maximo-custom-1.0.0.zip
└─ applications
   └─ maximo
      └─ ...
```

```powershell
cd C:\Source\mas\mas-deployment

Compress-Archive `
  -Path .\applications `
  -DestinationPath .\maximo-custom-1.0.0.zip `
  -Force
```

파일명에 버전을 넣습니다. 같은 이름으로 덮어쓰면 지금 배포된 것이 어느 빌드인지 구분되지 않습니다.

압축 결과는 올리기 전에 확인하는 편이 빠릅니다.

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
[IO.Compression.ZipFile]::OpenRead("C:\Source\mas\mas-deployment\maximo-custom-1.0.0.zip").Entries |
  Select-Object -First 5 FullName
```

`applications/maximo/...` 로 시작해야 합니다.

### 5.3 Bastion 업로드

```powershell
scp C:\Source\mas\mas-deployment\maximo-custom-1.0.0.zip maximo@192.168.2.210:/mas/mas-deployment/
```

```bash
ls -lh /mas/mas-deployment/
```

**게시 확인** — Bastion 자신에서 먼저 봅니다.

```bash
curl -sI http://192.168.2.210:8080/maximo-custom-1.0.0.zip
```

`HTTP/1.1 200 OK` 와 `Content-Length` 가 업로드한 파일 크기와 같아야 합니다.

다음으로 **클러스터 안에서** 같은 URL이 열리는지 봅니다. 실제로 내려받는 주체는 Bastion이 아니라 Manage 파드이므로 이 확인이 본질입니다.

```bash
export KUBECONFIG=/home/maximo/ocp-sno/auth/kubeconfig
POD=$(oc get pod -n mas-inst1-manage --field-selector=status.phase=Running -o name | head -1)
oc rsh -n mas-inst1-manage $POD curl -sI http://192.168.2.210:8080/maximo-custom-1.0.0.zip
```

`oc` 인증이 풀려 있으면 SNO 노드에서 대신 확인합니다.

```bash
ssh -i ~/.ssh/quay_installer core@192.168.2.211 \
  'curl -sI http://192.168.2.210:8080/maximo-custom-1.0.0.zip'
```

여기서 200이 확인된 주소를 그대로 §5.4에 등록합니다.

### 5.4 MAS에 Customization Archive 등록

MAS Admin 화면에서 진행합니다.

```
관리 → Suite → 작업공간 → Manage → 구성 업데이트 → 사용자 정의
```

**사용자 정의**에서 Customization Archive를 추가하고 **File address**에 §5.3에서 확인한 URL을 넣습니다.

```
http://192.168.2.210:8080/maximo-custom-1.0.0.zip
```

등록된 값은 CR에서도 확인됩니다. 비어 있으면 아카이브가 걸리지 않은 것이라, 빌드가 끝나도 반영되지 않습니다.

```bash
oc get manageworkspace inst1-ws1 -n mas-inst1-manage \
  -o jsonpath='{.spec.settings.customizations}{"\n"}'
```

### 5.5 Apply Changes

**Apply Changes** 를 실행하면 Manage가 ZIP을 내려받아 Server Bundle을 다시 빌드하고 배포합니다.

### 5.6 배포 진행 상태 확인

MAS 작업공간 상태 화면의 카드가 이 순서로 넘어갑니다.

```
Build in progress → Running reconciliation → 작업공간 활성화 완료
```

화면은 요약만 보여줍니다. 실제로 어디까지 갔는지는 OpenShift 쪽에서 봅니다.

```bash
oc get build,pods -n mas-inst1-manage
```

빌드는 **admin → all** 순서로 두 번 돕니다. 번호가 하나씩 늘어납니다(`admin-build-config-2`, `all-build-config-2`).

| 순서 | 빌드 | 내용 | 소요 |
|---|---|---|---|
| 1 | `admin-build-config-*` | Admin 번들 이미지 | ~10분 |
| 2 | `all-build-config-*` | **Server Bundle — EAR을 굽는 단계. 커스터마이징이 실제로 들어감** | ~15분 |

로그는 마지막 빌드를 잡아서 붙입니다.

```bash
BUILD=$(oc get build -n mas-inst1-manage --sort-by=.metadata.creationTimestamp -o name | tail -1)
oc logs -f $BUILD -n mas-inst1-manage --tail=30
```

`Push successful` 과 함께 새 태그(`<날짜>T<시각>-9.2.197`)가 찍히면 그 빌드는 끝입니다. 태그 날짜가 오늘로 바뀌었는지가 이미지가 실제로 새로 구워졌는지의 판정 기준입니다.

빌드 시작 직후 `Pending` 은 정상입니다. 빌더 파드가 admin 이미지를 빌드 컨텍스트로 풀어내는(`extract-image-content`) 동안 몇 분 걸립니다. 오래 멈춰 있으면 이벤트를 봅니다.

```bash
oc describe pod -n mas-inst1-manage -l openshift.io/build.name=all-build-config-2 | tail -25
```

`FailedScheduling` 이면 자원, `ImagePullBackOff` 면 레지스트리, `no space left on device` 면 노드 디스크 문제입니다.

빌드가 아예 시작되지 않으면 operator를 봅니다.

```bash
oc logs -n mas-inst1-manage deploy/ibm-mas-manage-operator -c manager --tail=50
```

`all` 빌드가 끝나면 `inst1-ws1-all` 파드가 새 이미지로 교체됩니다. 기동 완료 표식은 서버 로그에서 확인합니다.

```bash
oc rsh -n mas-inst1-manage deploy/inst1-ws1-all tail -f /logs/messages.log
```

```
CWWKF0011I: The defaultServer server is ready to run a smarter planet.
```

🔴 `oc logs` 로는 안 보입니다. 컨테이너 표준출력에는 대기 루프만 찍히고 실제 Maximo 로그는 파드 안 `/logs/messages.log` 에 쌓입니다.

전체 완료 판정은 CR 조건으로 봅니다.

```bash
oc get manageworkspace inst1-ws1 -n mas-inst1-manage \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status} — {.message}{"\n"}{end}'
```

### 5.7 기능 확인

작업공간 활성화가 끝나면 Maximo Manage에 접속해 배포한 기능이 동작하는지 확인합니다.

```
https://ws1.manage.inst1.apps.mas-it.itmsg.co.kr/maximo
```

클래스가 실제로 교체됐는지는 파드 안에서 직접 볼 수 있습니다. 화면 동작이 애매할 때 이쪽이 확실합니다.

```bash
oc rsh -n mas-inst1-manage deploy/inst1-ws1-all \
  ls -l /opt/IBM/SMP/maximo/applications/maximo/businessobjects/classes/com/
```

타임스탬프가 이번 빌드 시각이면 반영된 것입니다. 파일이 없거나 예전 날짜면 §5.2의 ZIP 최상위 구조부터 다시 보세요.

