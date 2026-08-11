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
| 5 | [Adding customizations](https://www.ibm.com/docs/en/masv-and-l/maximo-manage/cd?topic=customizing-adding-customizations) | Archive를 Manage가 접근 가능한 위치에 배치하고 MAS 관리화면에서 URL 등록 후 Apply Changes | §6, §7 |

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
C:\Source\mas\manage\
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
cd C:\Source\mas\manage
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
C:\Source\mas\manage\applications\maximo\
├─ businessobjects\classes\<패키지>\*.class     Java 클래스
└─ maximouiweb\webmodule\...                    JSP·웹 리소스
```

### 4.2 기존 컨테이너 제거

```powershell
docker rm -f maximo-dev
```

### 4.3 빌드

```powershell
cd C:\Source\mas\manage
docker build --build-arg BUNDLE=all -t managedev:latest -f .\manage-developer\Dockerfile .
```

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

## 5. Customization Archive 구성

## 6. Bastion HTTP 서버 준비 및 게시

## 7. MAS에 Customization Archive 등록

## 8. Build / Reconcile 및 배포 확인

## 9. 클래스 연결 (Database Configuration)

## 10. 동작 테스트
