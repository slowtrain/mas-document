# Maximo IT 오프라인(Airgap) 설치 가이드

IBM MAS CLI 공식 문서(`ibm-mas.github.io/cli`)와 Red Hat OpenShift 공식 문서를 근거로 작성한 Maximo IT 오프라인 설치 가이드입니다.

<details>
<summary><b>목차</b></summary>

- [1. 설치 요약 및 흐름](#1-설치-요약-및-흐름)
  - [1.1 설치 구성](#11-설치-구성)
  - [1.2 필요 자원](#12-필요-자원)
  - [1.3 설치 흐름](#13-설치-흐름)
  - [1.4 환경 정보](#14-환경-정보)
  - [1.5 참고한 공식 문서](#15-참고한-공식-문서)
- [2. 사전준비 사항 (설치 파일 및 이미지 생성)](#2-사전준비-사항-설치-파일-및-이미지-생성)
  - [2.1 저장공간](#21-저장공간)
    - [2.1.1 확인 및 검증](#211-확인-및-검증)
  - [2.2 인터넷 연결 서버에 도구 설치 (root 계정)](#22-인터넷-연결-서버에-도구-설치-root-계정)
    - [2.2.1 실행](#221-실행)
    - [2.2.2 확인 및 검증](#222-확인-및-검증)
  - [2.3 확보해야 할 항목](#23-확보해야-할-항목)
    - [0. 디렉터리 생성 + linger 활성화](#0-디렉터리-생성--linger-활성화)
    - [1~3. IBM Entitlement Key / MAS 라이선스 / Red Hat Pull Secret](#13-ibm-entitlement-key--mas-라이선스--red-hat-pull-secret)
    - [4. 사이트 Bastion용 RPM 오프라인 저장소](#4-사이트-bastion용-rpm-오프라인-저장소)
    - [5. OpenShift Client / Installer / checksum](#5-openshift-client--installer--checksum)
    - [6. MAS CLI 이미지](#6-mas-cli-이미지)
    - [7. Mirror Registry 설치 패키지](#7-mirror-registry-설치-패키지)
    - [8. Red Hat 콘텐츠 이미지 세트](#8-red-hat-콘텐츠-이미지-세트)
    - [9. MAS 콘텐츠 이미지 세트](#9-mas-콘텐츠-이미지-세트)
    - [10. NFS 동적 프로비저너 이미지 (RWX용)](#10-nfs-동적-프로비저너-이미지-rwx용)
  - [2.4 설치 파일 최종 확인](#24-설치-파일-최종-확인)
    - [2.4.1 확인 및 검증](#241-확인-및-검증)
  - [2.5 설치 파일 전달](#25-설치-파일-전달)
    - [2.5.1 실행 — 체크섬 생성](#251-실행--체크섬-생성)
    - [2.5.2 tar로 묶어 전송](#252-tar로-묶어-전송)
    - [2.5.3 확인 및 검증](#253-확인-및-검증)
- [3. 설치 과정](#3-설치-과정)
  - [계정 사용 원칙](#계정-사용-원칙)
  - [3.1 RPM 오프라인 저장소 등록 및 도구 설치 (root 계정)](#31-rpm-오프라인-저장소-등록-및-도구-설치-root-계정)
    - [3.1.1 실행](#311-실행)
    - [3.1.2 확인 및 검증](#312-확인-및-검증)
  - [3.2 DNS 구성 (root 계정)](#32-dns-구성-root-계정)
    - [3.2.1 실행 — `mas.conf` 작성](#321-실행--masconf-작성)
    - [3.2.2 실행 — 서비스·방화벽 기동](#322-실행--서비스방화벽-기동)
    - [3.2.3 실행 — Bastion 자신이 dnsmasq를 쓰도록 설정](#323-실행--bastion-자신이-dnsmasq를-쓰도록-설정)
    - [3.2.4 확인 및 검증](#324-확인-및-검증)
  - [3.3 NTP 구성 (root 계정)](#33-ntp-구성-root-계정)
    - [3.3.1 실행 — 시간 보정](#331-실행--시간-보정)
    - [3.3.2 실행 — chrony 구성](#332-실행--chrony-구성)
    - [3.3.3 확인 및 검증](#333-확인-및-검증)
  - [3.4 MAS CLI 이미지 불러오기 (사용자 계정)](#34-mas-cli-이미지-불러오기-사용자-계정)
    - [3.4.1 실행](#341-실행)
    - [3.4.2 확인 및 검증](#342-확인-및-검증)
  - [3.5 Mirror Registry (Quay) 설치 (사용자 계정)](#35-mirror-registry-quay-설치-사용자-계정)
    - [3.5.1 실행 — 디렉터리 준비](#351-실행--디렉터리-준비)
    - [3.5.2 실행 — SSH 키 준비](#352-실행--ssh-키-준비)
    - [3.5.3 실행 — Registry 설치](#353-실행--registry-설치)
    - [3.5.4 실행 — Bastion에 Registry CA 신뢰 등록 (root 계정)](#354-실행--bastion에-registry-ca-신뢰-등록-root-계정)
    - [3.5.5 확인 및 검증 (사용자 계정)](#355-확인-및-검증-사용자-계정)
  - [3.6 Registry로 이미지 Push (사용자 계정)](#36-registry로-이미지-push-사용자-계정)
    - [3.6.1 실행 — 환경변수](#361-실행--환경변수)
    - [3.6.2 실행 — push용 인증 파일 생성](#362-실행--push용-인증-파일-생성)
    - [3.6.3 실행 — Red Hat 콘텐츠 push (IBM 방식)](#363-실행--red-hat-콘텐츠-push-ibm-방식)
    - [3.6.4 실행 — Red Hat 콘텐츠 push (Red Hat 공식 방식, 대체)](#364-실행--red-hat-콘텐츠-push-red-hat-공식-방식-대체)
    - [3.6.5 실행 — MAS 콘텐츠 push : core](#365-실행--mas-콘텐츠-push--core)
    - [3.6.6 실행 — MAS 콘텐츠 push : apps](#366-실행--mas-콘텐츠-push--apps)
    - [3.6.7 실행 — MAS 콘텐츠 push : other](#367-실행--mas-콘텐츠-push--other)
    - [3.6.8 실행 — MAS 콘텐츠 push : cli](#368-실행--mas-콘텐츠-push--cli)
    - [3.6.9 확인 및 검증](#369-확인-및-검증)
  - [3.7 OCP SNO 클러스터 설치 (사용자 계정)](#37-ocp-sno-클러스터-설치-사용자-계정)
    - [3.7.1 실행 — CLI 도구 준비](#371-실행--cli-도구-준비)
    - [3.7.2 실행 — 설정 파일에 넣을 값 3개 수집](#372-실행--설정-파일에-넣을-값-3개-수집)
    - [3.7.3 실행 — `install-config.yaml` 작성](#373-실행--install-configyaml-작성)
    - [3.7.4 실행 — `agent-config.yaml` 작성](#374-실행--agent-configyaml-작성)
    - [3.7.5 실행 — 백업](#375-실행--백업)
    - [3.7.6 실행 — ISO 생성](#376-실행--iso-생성)
    - [3.7.7 실행 — SNO 부팅](#377-실행--sno-부팅)
    - [3.7.8 확인 및 검증](#378-확인-및-검증)
  - [3.8 Airgap(폐쇄망) 구성 (사용자 계정)](#38-airgap폐쇄망-구성-사용자-계정)
    - [3.8.1 oc-mirror 생성 리소스 적용 + OperatorHub 기본 소스 비활성화](#381-oc-mirror-생성-리소스-적용--operatorhub-기본-소스-비활성화)
    - [3.8.2 mas configure-airgap 실행](#382-mas-configure-airgap-실행)
  - [3.9 스토리지 준비 (RWO + RWX StorageClass) (사용자 계정)](#39-스토리지-준비-rwo--rwx-storageclass-사용자-계정)
    - [3.9.1 RWO — LVM Storage (LVMS) Operator](#391-rwo--lvm-storage-lvms-operator)
    - [3.9.2 RWX — 방식 선택](#392-rwx--방식-선택)
    - [3.9.3 스토리지 검증 — PVC 바인딩 + 쓰기 테스트](#393-스토리지-검증--pvc-바인딩--쓰기-테스트)
    - [3.9.4 OpenShift 내부 Image Registry 구성](#394-openshift-내부-image-registry-구성)
  - [3.10 MAS 설치 — Core + Manage + Maximo IT (사용자 계정)](#310-mas-설치--core--manage--maximo-it-사용자-계정)
    - [3.10.1 실행 — 사전 준비](#3101-실행--사전-준비)
    - [3.10.2 실행 — mas install](#3102-실행--mas-install)
    - [3.10.3 실행 — 비대화형 (권장)](#3103-실행--비대화형-권장)
    - [3.10.4 확인 및 검증](#3104-확인-및-검증)
    - [3.10.5 실패 시 초기화 — Manage 재설치](#3105-실패-시-초기화--manage-재설치)
  - [3.11 Manage + Maximo IT 확인 (웹 UI)](#311-manage--maximo-it-확인-웹-ui)
    - [3.11.1 확인 및 검증](#3111-확인-및-검증)
    - [3.11.2 웹 UI 확인](#3112-웹-ui-확인)
    - [3.11.3 설치 후 작업](#3113-설치-후-작업)
  - [3.12 설치 후 확인 (사용자 계정)](#312-설치-후-확인-사용자-계정)
    - [3.12.1 실행 — 접속용 `hosts` 등록 (검증용)](#3121-실행--접속용-hosts-등록-검증용)
    - [3.12.2 운영 전환 — DNS와 인증서](#3122-운영-전환--dns와-인증서)
    - [3.12.3 확인 및 검증](#3123-확인-및-검증)
    - [3.12.4 노드 종료·재시작 절차](#3124-노드-종료재시작-절차)
- [4. 트러블슈팅](#4-트러블슈팅)
  - [4.1 `:Z` vs `:z` SELinux 라벨 충돌](#41-z-vs-z-selinux-라벨-충돌)
  - [4.2 MAS 콘텐츠 미러링 중 `cp.icr.io` OAuth 토큰 타임아웃](#42-mas-콘텐츠-미러링-중-cpicrio-oauth-토큰-타임아웃)
  - [4.3 Red Hat 콘텐츠 미러링 — 타임아웃과 캐시 소실](#43-red-hat-콘텐츠-미러링--타임아웃과-캐시-소실)
  - [4.4 SSH 접속을 끊으면 미러링 컨테이너가 죽음](#44-ssh-접속을-끊으면-미러링-컨테이너가-죽음)
  - [4.5 `mas mirror-redhat-images --mode from-filesystem` 을 쓰지 않는 이유](#45-mas-mirror-redhat-images---mode-from-filesystem-을-쓰지-않는-이유)
  - [4.6 `install-config.yaml` 검증 실패 — `imageDigestSources`](#46-install-configyaml-검증-실패--imagedigestsources)
  - [4.7 MCO가 `registries.conf`를 렌더링하지 못함 — `conflicting mirrorSourcePolicy`](#47-mco가-registriesconf를-렌더링하지-못함--conflicting-mirrorsourcepolicy)

</details>

















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
| 스토리지 | RWO = **LVM Storage**(`lvms-vg1`) / RWX = **외부 NFS — Bastion 겸용** (§3.9) |

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
| Registry 계정 | §3.5 설치 시 자동 발급. Push(§3.6)와 클러스터 Pull(§3.7)에 모두 사용 | `init` |
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
- [oc-mirror plugin v2로 폐쇄망 미러링 (Red Hat, OCP 4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/disconnected_environments/about-installing-oc-mirror-v2) — §2.3 8번·§3.6.4의 근거

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

Registry 사본은 §3.6 push 때 `~/mirror-registry/quay`에 만들어집니다 — 반입 파일과 별개로 같은 양이 한 번 더 쌓입니다.

#### 2.1.1 확인 및 검증

```bash
du -sh ~/mas-install/mirror/*
df -h /home
```

### 2.2 인터넷 연결 서버에 도구 설치 (root 계정)

§2 작업에 필요한 도구를 설치합니다. 사이트 Bastion용 패키지는 §2.3의 4번에서 RPM으로 확보하므로 **여기서 미리 설치하지 마세요.**

#### 2.2.1 실행

```bash
dnf install -y podman curl tar rsync dnf-plugins-core createrepo_c
```

#### 2.2.2 확인 및 검증

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
| `podman` | `mas` CLI 실행 (§3.4 이후) |
| `dnsmasq`, `bind-utils` | DNS 서버·`nslookup` (§3.2) |
| `chrony` | NTP 서버 (§3.3) |
| `firewalld` | 방화벽 (§3.2) |
| `nfs-utils` | RWX용 NFS 서버 (§3.9.2) |
| `nmstate` | Agent-based Installer의 `networkConfig` 검증 (§3.7) |
| `jq` | Pull Secret 병합 (§3.7, §3.8.2) |
| `tar`, `gzip` | `oc`·`mirror-registry` 압축 해제 (§3.5, §3.7) |
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

§3.7에서 `oc`·`kubectl`·`openshift-install`로 사용합니다.

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

§3.4에서 불러와 `mas` 명령을 실행합니다.

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

§3.5에서 폐쇄망 Registry를 구축하는 데 사용합니다. [Red Hat OpenShift Downloads](https://console.redhat.com/openshift/downloads)의 **"mirror registry for Red Hat OpenShift"** 에서 RHEL 9 x86_64용 URL을 브라우저로 발급받습니다(로그인 필요, URL에 유효시간 있음).

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

⛔ **Cloud Pak for Data는 받지 않습니다.** `--mirror-cp4d`, `--mirror-wsl`, `--mirror-wml`, `--mirror-spark`, `--mirror-cognos`를 어떤 명령에도 넣지 마세요 — 이번 구성에 불필요하고 용량이 매우 큽니다. `--mirror-odf`도 제외합니다 — ODF 오퍼레이터는 8번(Red Hat 콘텐츠)에 이미 포함되어 있습니다.

**1) 실행 — 환경변수** (셸마다 한 번)

`to-filesystem` 모드에서도 `-H/-P/-u/-p`가 필수입니다(생략하면 `--help`만 출력하고 종료). Registry Host/Port는 목적지 이미지 경로 생성에 쓰이므로 **최종 폐쇄망 Registry의 실제 값**을 넣으세요.

```bash
export LOCAL_DIR="$HOME/mas-install/mirror"
export MIRROR_SINGLE_ARCH=amd64
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)

export REGISTRY_HOST=registry.mas-it.itmsg.co.kr
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=init
```

비밀번호는 **한 줄씩 따로** 실행하세요. 블록으로 붙여넣으면 `read`가 뒷줄을 먹어 값이 깨집니다.

```bash
read -s REGISTRY_PASSWORD
```

```bash
export REGISTRY_PASSWORD
echo "${#REGISTRY_PASSWORD}자"
```

`-u`/`-p`는 이 단계에서 **인증에 쓰이지 않습니다** — 목적지가 파일시스템이라 CLI의 인자 검증을 통과시키기 위한 값입니다. Registry가 아직 없다면 임의 값이어도 무방하고, 이미 §3.5에서 구축했다면 그 계정을 넣으면 됩니다.

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

검증 : 🔴 **Maximo IT는 상태 목록에 나오지 않습니다** — Manage 확장이라 Manage 매니페스트에 포함됩니다. `extension-icd`가 있어야 §3.10에서 `--manage-components "base=latest,icd=latest"` 로 설치할 수 있으므로 직접 확인하세요.

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

검증 : `[FAILURE]` **0건**

```bash
for f in ~/mas-install/mirror/stage-*.nohup.log; do
  echo "===== $(basename $f)"
  tr '\r' '\n' < "$f" | grep -aE '\[SUCCESS\]|\[FAILURE\]'
done
```

🔴 **`[FAILURE]`와 `[SUCCESS]`가 같이 나올 수 있습니다.** 한 단계에서 여러 컴포넌트를 미러링하므로 일부만 실패할 수 있습니다. `[SUCCESS]` 개수만 세지 말고 **`[FAILURE]`가 0건인지** 확인하세요.

실패하면 디렉터리를 비우지 말고 **해당 단계를 그대로 다시 실행**합니다 — 이미 받은 이미지는 건너뜁니다.

#### 10. NFS 동적 프로비저너 이미지 (RWX용)

§3.9.2에서 RWX StorageClass를 제공하는 `nfs-subdir-external-provisioner`입니다.

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

#### 2.4.1 확인 및 검증

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

#### 2.5.1 실행 — 체크섬 생성

489GB를 해싱하므로 수 시간 걸립니다.

```bash
cd "$HOME/mas-install"
nohup sh -c "find licenses redhat ocp registry mas-cli rpms nfs-provisioner mirror \
  -type f -print0 | sort -z | xargs -0 sha256sum > transfer-files.sha256" &
```

```bash
wc -l ~/mas-install/transfer-files.sha256      # 더 이상 안 늘면 완료
```

#### 2.5.2 tar로 묶어 전송

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

#### 2.5.3 확인 및 검증

출력이 없어야 §3.1로 진행합니다.

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
sudo -i          # root 구간(§3.1, §3.2, §3.5 CA 등록, §3.9.2 NFS 서버) 진입
exit             # 끝나면 사용자 계정으로 복귀
```

| 구간 | 계정 | root가 필요한 이유 |
|---|---|---|
| §3.1 | **root** | `/etc/yum.repos.d/`, 시스템 패키지 설치 |
| §3.2 | **root** | `/etc/dnsmasq.d/`, `systemctl`, `firewall-cmd` |
| §3.5 (일부) | **root** | `/etc/containers/certs.d/`·`/etc/pki/ca-trust/` CA 등록 |
| §3.9.2 A-1 | **root** | `/etc/exports`, `nfs-server`, 방화벽 |
| 그 외 전부 | 사용자 계정 | `mas` CLI, Mirror Registry 설치·운영, 이미지 push, `mas install` 등 (rootless podman) |

🔴 **주의 — `~` 경로 함정**: `sudo -i` 후에는 `$HOME`이 `/root`가 되므로 **`~/mas-install/...`이 `/root/mas-install/...`로 해석**됩니다. 그래서 이 문서의 **root 구간은 모두 절대 경로**(`/home/maximo/mas-install/...`)로 작성했고, **사용자 계정 구간은 `~`** 를 씁니다. 명령을 복사할 때 현재 어느 셸에 있는지 (`whoami`로) 확인하세요.

```bash
whoami   # root 인지 maximo 인지 확인
```

### 3.1 RPM 오프라인 저장소 등록 및 도구 설치 (root 계정)

§2.3의 4번에서 반입한 `rpms/`를 로컬 저장소로 등록하고 §3에 필요한 도구를 설치합니다.

🔴 RHEL은 GPG 키 **파일**은 제공하지만 **RPM DB에 임포트되지 않은 경우**가 있습니다. 이 상태로 `gpgcheck=1` 저장소를 쓰면 `"you do not have any GPG public keys installed"` 로 설치가 전부 실패합니다.

#### 3.1.1 실행

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

#### 3.1.2 확인 및 검증

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

#### 3.2.1 실행 — `mas.conf` 작성

```bash
vi /etc/dnsmasq.d/mas.conf
```

```ini
# LAN 인터페이스에서도 응답하도록 지정 — 없으면 SNO 질의에 응답하지 않습니다
interface=<bastion-nic>

# 상위 DNS: 사내 DNS가 있으면 server=<dns-ip> 로 추가, 없으면 넣지 않음
server=<dns-ip>
address=/api.<cluster-name>.<base-domain>/<sno-ip>
address=/api-int.<cluster-name>.<base-domain>/<sno-ip>
address=/.apps.<cluster-name>.<base-domain>/<sno-ip>
address=/registry.<cluster-name>.<base-domain>/<bastion-ip>
```

예시)

```ini
interface=ens192

#사내 DNS 가 없으면 주석 처리
#server=8.8.8.8
address=/api.mas-it.itmsg.co.kr/192.168.2.211
address=/api-int.mas-it.itmsg.co.kr/192.168.2.211
address=/.apps.mas-it.itmsg.co.kr/192.168.2.211
address=/registry.mas-it.itmsg.co.kr/192.168.2.210
```

🔴 **`interface=` 를 빼먹으면 Bastion에서만 이름이 풀립니다.** `/etc/dnsmasq.conf`에 `interface=lo`·`bind-interfaces`가 들어 있으면 dnsmasq가 루프백에만 바인딩되어, `getent hosts`는 성공하는데 **SNO는 이름을 못 찾아 §3.7.7 부팅에서 `Unable to pull OpenShift release image`로 멈춥니다.**

```bash
grep -rE '^(interface|bind-interfaces|listen-address)' /etc/dnsmasq.conf /etc/dnsmasq.d/
```

#### 3.2.2 실행 — 서비스·방화벽 기동

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

#### 3.2.3 실행 — Bastion 자신이 dnsmasq를 쓰도록 설정

빼먹으면 §3.6 push와 §3.8 `oc login`이 실패합니다.

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

#### 3.2.4 확인 및 검증

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

🔴 **여기까지는 Bastion 자신의 질의만 확인한 것입니다.** dnsmasq가 루프백에만 바인딩돼 있어도 전부 통과합니다. **LAN에서 응답하는지 반드시 함께 보세요.**

```bash
ss -ulnp | grep ':53'
```

`127.0.0.1:53`만 있고 `<bastion-ip>:53`이 없으면 §3.2.1의 `interface=`를 확인해야 합니다. SNO가 부팅한 뒤에는 노드에서 직접 확인할 수 있습니다.

```bash
# SNO 노드에서
getent hosts registry.<cluster-name>.<base-domain>
curl -sS -o /dev/null -w '%{http_code}\n' https://registry.<cluster-name>.<base-domain>:8443/v2/
```

> dnsmasq 로그에 `ignoring nameserver 127.0.0.1 - local interface`가 뜨는 것은 정상입니다 — 자기 자신을 상위 DNS로 참조하지 않는다는 뜻입니다.


### 3.3 NTP 구성 (root 계정)

🔴 시간 동기가 어긋나면 인증서 검증 실패, etcd 이상, Operator 오류로 이어집니다. 폐쇄망에서는 외부 NTP에 접근할 수 없으므로 **Bastion을 내부 NTP 서버로** 구성합니다.

#### 3.3.1 실행 — 시간 보정

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

#### 3.3.2 실행 — chrony 구성

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
> SNO에는 §3.7 `agent-config.yaml`의 `additionalNTPSources`로 Bastion IP를 전달합니다.

#### 3.3.3 확인 및 검증

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

### 3.4 MAS CLI 이미지 불러오기 (사용자 계정)

인터넷 연결 서버에서 저장해 온 tar를 불러옵니다(§2.3의 6번).

#### 3.4.1 실행

```bash
podman load -i ~/mas-install/mas-cli/mas-cli-23.4.1.tar
```

#### 3.4.2 확인 및 검증

```bash
podman image exists quay.io/ibmmas/cli:23.4.1 && echo OK
podman run --rm -t quay.io/ibmmas/cli:23.4.1 mas mirror-images --help | head -10
```

첫 줄에 `IBM Maximo Application Suite Air Gap Image Mirror (v23.4.1)` 이 나오면 정상입니다.

> - `mas --version`·`mas --help`는 없는 파라미터입니다(`unknown parameter`). `--help`는 서브커맨드에만 있습니다.
> - `-t`가 없으면 `tput: No value for $TERM` 경고가 나옵니다.
> - 끝에 `write /dev/stdout: broken pipe`가 뜨는 것은 `head`가 파이프를 먼저 닫아서이며 무해합니다.

### 3.5 Mirror Registry (Quay) 설치 (사용자 계정)

Bastion은 같은 이미지를 **두 벌 보관**합니다.

| | 정체 | 위치 | 용량 |
|---|---|---|---|
| 미러 파일 | §2.5에서 반입한 원본 | `~/mas-install/mirror/` | 489GB |
| Registry 사본 | §3.6 push 때 Quay가 생성 | `~/mirror-registry/quay` | 약 489GB |

> 빈 디스크가 있으면 `~/mirror-registry`에 마운트하세요. §3.6 push 때 읽기(미러 파일)와 쓰기(Registry)가 다른 디스크로 분산되어 빨라집니다. 없으면 그냥 진행하면 됩니다 — 루트 파일시스템에 약 489GB 여유만 있으면 동작합니다.

#### 3.5.1 실행 — 디렉터리 준비

```bash
mkdir -p ~/quay-install ~/mirror-registry/{quay,sqlite}
chmod 750 ~/quay-install ~/mirror-registry
```

#### 3.5.2 실행 — SSH 키 준비

설치 프로그램은 ansible로 대상 호스트에 **SSH 접속**해서 작업합니다. 자기 자신에게 설치하더라도 키가 필요하고, **자동 생성해주지 않습니다** — 없으면 `Could not find ssh key at ~/.ssh/quay_installer`로 중단됩니다.

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/quay_installer

cat ~/.ssh/quay_installer.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

형식) 호스트 키를 미리 등록해 첫 접속 확인 프롬프트를 없앱니다

```bash
ssh-keyscan -H <bastion-ip> >> ~/.ssh/known_hosts 2>/dev/null
ssh -i ~/.ssh/quay_installer maximo@<bastion-ip> hostname
```

예시)

```bash
ssh-keyscan -H 192.168.2.210 >> ~/.ssh/known_hosts 2>/dev/null
ssh -i ~/.ssh/quay_installer maximo@192.168.2.210 hostname
```

검증 : 비밀번호를 묻지 않고 호스트명이 출력되어야 함

#### 3.5.3 실행 — Registry 설치

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

5~15분 걸립니다. `Copy Images` 단계에서 `image-archive.tar` 2.5GB를 SSH로 옮기고 `podman load`로 로드하는 시간이 대부분입니다 — 출력이 멈춘 것처럼 보여도 정상입니다.

진행 확인 : 다른 터미널에서 실행합니다. `image-archive.tar`가 2.5GB까지 커진 뒤 이미지 3개(`quay-rhel8`, `redis`, `pause`)가 생기고, 마지막에 컨테이너가 뜹니다

```bash
watch -n 10 'du -sh ~/quay-install 2>/dev/null; echo ---; podman images; echo ---; podman ps'
```

🔴 마지막에 출력되는 **초기 사용자명·비밀번호를 반드시 기록하세요.** 다시 볼 수 없고 §3.6 push와 §3.8.2에서 씁니다.

```
Quay is available at https://registry.mas-it.itmsg.co.kr:8443
with credentials (init, <생성된 비밀번호>)
```

실패하면 되돌릴 수 있습니다.

```bash
./mirror-registry uninstall
```

#### 3.5.4 실행 — Bastion에 Registry CA 신뢰 등록 (root 계정)

podman과 시스템 전역이 이 Registry의 자체 서명 인증서를 신뢰하도록 등록합니다. 빼먹으면 §3.6 push가 TLS 오류로 실패합니다.

```bash
REGISTRY=registry.mas-it.itmsg.co.kr:8443

REGISTRY_CA="$(find /home/maximo/quay-install -name rootCA.pem -type f -print -quit)"
install -d -m 0755 "/etc/containers/certs.d/$REGISTRY"
install -m 0644 "$REGISTRY_CA" "/etc/containers/certs.d/$REGISTRY/ca.crt"
install -m 0644 "$REGISTRY_CA" "/etc/pki/ca-trust/source/anchors/${REGISTRY%:*}.crt"
update-ca-trust
```

#### 3.5.5 확인 및 검증 (사용자 계정)

Quay는 rootless 컨테이너라 **root로 확인하면 아무것도 안 보입니다.** 설치한 계정에서 실행하세요.

검증 : `quay-app`, `quay-redis`, `-infra` 3개가 `Up` 이어야 함

```bash
podman ps
```

검증 : `401` — Registry가 응답하고 인증을 요구하는 정상 상태

`-k`를 일부러 붙이지 않습니다. §3.5.4의 CA 등록이 실제로 먹혔는지 함께 확인하기 위해서입니다. `000`이 나오면 TLS 검증 실패이므로 §3.5.4를 다시 보세요.

```bash
curl -s -o /dev/null -w '%{http_code}\n' "https://registry.mas-it.itmsg.co.kr:8443/v2/"
```

검증 : `Login Succeeded!` — §3.6 push의 전제 조건입니다

§3.5.3에서 출력된 초기 계정으로 로그인합니다. 성공하면 자격증명이 `${XDG_RUNTIME_DIR}/containers/auth.json`에 저장돼 §3.6에서 재사용됩니다.

```bash
podman login registry.mas-it.itmsg.co.kr:8443 -u init
```

**서비스 관리**

Quay는 설치한 계정의 **systemd user 서비스**로 등록됩니다. root의 systemd가 아니므로 `--user`가 필요하고, `podman stop`으로 멈추면 systemd가 곧바로 되살립니다.

```bash
systemctl --user list-units 'quay*'
systemctl --user status quay-app
systemctl --user restart quay-app
journalctl --user -u quay-app -f
```

재부팅 후에도 자동 기동하는 것은 설치 프로그램이 마지막에 실행한 `loginctl enable-linger` 덕분입니다 — 없으면 로그아웃과 함께 내려갑니다(§4.4).

| 데이터 | 경로 |
|---|---|
| 설정·인증서 | `~/quay-install` |
| 이미지 저장소 | `~/mirror-registry/quay` |
| 메타데이터 DB (sqlite) | `~/mirror-registry/sqlite` |

> ⚠️ `mirror registry for Red Hat OpenShift`는 Red Hat 공식적으로 **OCP 설치용 소규모·비HA Registry**입니다. MAS 전체 이미지를 담는 장기 운영 Registry는 지원 범위 밖이므로, 운영 전환 시 Quay/Harbor 등을 검토하세요.

### 3.6 Registry로 이미지 Push (사용자 계정)

반입한 미러 데이터를 Mirror Registry로 올립니다. Red Hat 콘텐츠와 MAS 콘텐츠는 만든 방식이 다르므로 올리는 방식도 다릅니다.

| | 만든 도구 | 산출물 | 올리는 방법 |
|---|---|---|---|
| Red Hat | `oc mirror --v2` (§2.3 8번) | `mirror_000001.tar` + `working-dir` | 3) IBM 래퍼 → 실패 시 §3.6.4 `oc mirror` 직접 |
| MAS | `mas mirror-images` (`oc image mirror` 기반) | 디렉터리 트리 | 4) `mas mirror-images` 래퍼 |

⚠️ Red Hat 콘텐츠는 CLI 23.4.1에서 **IBM 래퍼가 동작하지 않습니다.** 래퍼가 `--from` 에 **tar 파일 경로**를 넘기는데 이건 oc-mirror v1 문법이고, v2는 `--from file://<디렉터리>` 를 받아 그 안의 `mirror_*.tar` 를 스스로 찾습니다(§4.5). §3.6.3으로 먼저 확인한 뒤 §3.6.4로 넘어가는 순서로 적어두었습니다 — CLI 버전이 올라가면 §3.6.3만으로 끝날 수 있습니다.

#### 3.6.1 실행 — 환경변수

셸마다 한 번 설정합니다. 계정은 §3.5.3에서 발급된 **Quay 관리자**입니다.

형식)

```bash
export REGISTRY_HOST=registry.<cluster-name>.<base-domain>
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=init
export REGISTRY_PASSWORD='<설치 시 출력된 비밀번호>'
export LOCAL_DIR="$HOME/mas-install/mirror"
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)
```

예시)

```bash
export REGISTRY_HOST=registry.mas-it.itmsg.co.kr
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=init
export REGISTRY_PASSWORD='T3AgRHSeYw6x5u0Z2q4OE9s81Koyd7tV'
export LOCAL_DIR="$HOME/mas-install/mirror"
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)
```

`IBM_ENTITLEMENT_KEY`는 폐쇄망에서 외부 인증에 쓰이지 않습니다. **없으면 CLI가 `--help`만 출력하고 종료**하므로 인자 검증을 통과시키기 위해 넣습니다.

검증 : 자릿수까지 확인합니다

```bash
echo "$LOCAL_DIR / $REGISTRY_HOST:$REGISTRY_PORT / $REGISTRY_USERNAME / 비밀번호 ${#REGISTRY_PASSWORD}자 / 키 ${#IBM_ENTITLEMENT_KEY}자"
```

#### 3.6.2 실행 — push용 인증 파일 생성

컨테이너 안의 `oc mirror`가 쓸 자격증명입니다. 호스트의 `podman login` 결과는 컨테이너에서 보이지 않으므로 별도 파일로 만듭니다.

```bash
mkdir -p ~/mas-install/push-auth
podman login "$REGISTRY_HOST:$REGISTRY_PORT" -u "$REGISTRY_USERNAME" \
  --authfile ~/mas-install/push-auth/config.json
```

검증 : `Login Succeeded!` 그리고 파일에 Registry 항목이 있어야 함

```bash
grep -o "$REGISTRY_HOST:$REGISTRY_PORT" ~/mas-install/push-auth/config.json
```

#### 3.6.3 실행 — Red Hat 콘텐츠 push (IBM 방식)

IBM 문서가 안내하는 방식입니다. **먼저 이걸로 시도하세요.** 실패하면 §3.6.4로 넘어갑니다.

```bash
podman run --rm --network host \
  -v "$LOCAL_DIR":/mnt/workspace:z \
  -v "$HOME/mas-install/redhat":/mnt/redhat:ro,z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-redhat-images \
  --mode from-filesystem --dir /mnt/workspace/redhat \
  --pullsecret /mnt/redhat/pull-secret.json \
  --mirror-platform --mirror-operators \
  --release 4.20 --min-version 4.20.30 --max-version 4.20.30 \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  --no-confirm 2>&1 | tee ~/mas-install/mirror/redhat-push-wrapper.log
```

Red Hat Pull Secret은 push에 쓸 일이 없는데도 요구합니다 — `direct` 모드와 인자 검증을 공유하는 탓입니다. 폐쇄망에서 외부로 나가지는 않습니다.

검증 : 화면이 아니라 **역할이 남기는 로그**를 봐야 합니다

```bash
tail -20 ~/mas-install/mirror/redhat/logs/mirror-from-filesystem-ocp4.20.log
```

🔴 **CLI 23.4.1에서는 아래처럼 실패합니다** — 실측 결과입니다(§4.5).

```
[INFO]  : 🔀 workflow mode: diskToMirror
[ERROR] : [Executor] no tar archives matching "mirror_[0-9]{6}\.tar"
          found in "/mnt/workspace/redhat/mirror_seq1_000000.tar"
```

#### 3.6.4 실행 — Red Hat 콘텐츠 push (Red Hat 공식 방식, 대체)

3)이 실패하면 래퍼가 대신 실행해주던 명령을 직접 호출합니다. Red Hat 공식 문서의 diskToMirror 문법 그대로입니다.

```
oc mirror --v2 -c <imageset> --from file://<디렉터리> docker://<registry>
```

`mirrorToDisk` 때와 대칭입니다. 만들 때 `file:////mnt/workspace/redhat` 로 썼으니 올릴 때도 같은 **디렉터리**를 `--from` 에 줍니다 — 파일명은 바꾸지 않습니다.

```bash
mkdir -p ~/mas-install/oc-mirror-cache

nohup podman run --rm --network host \
  -v "$LOCAL_DIR":/mnt/workspace:z \
  -v "$HOME/mas-install/push-auth":/mnt/auth:z \
  -v "$HOME/mas-install/oc-mirror-cache":/mnt/cache:z \
  -e DOCKER_CONFIG=/mnt/auth \
  quay.io/ibmmas/cli:23.4.1 \
  oc mirror --remove-signatures --v2 \
    -c /mnt/workspace/redhat/imageset-ocp4.20.yml \
    --from file:///mnt/workspace/redhat \
    docker://$REGISTRY_HOST:$REGISTRY_PORT \
    --dest-tls-verify=false \
    --cache-dir /mnt/cache \
    --image-timeout 60m \
  > "$HOME/mas-install/mirror/redhat-push.nohup.log" 2>&1 &

disown
```

| 옵션 | 이유 |
|---|---|
| `--network host` | Bastion의 `/etc/resolv.conf`가 `127.0.0.1`(dnsmasq)을 가리킵니다. 컨테이너 네트워크에서는 그 주소가 컨테이너 자신이라 Registry 이름 해석이 실패합니다 |
| `--dest-tls-verify=false` | Quay 자체 서명 인증서. 컨테이너 안에는 §3.5.4의 CA가 없습니다 |
| `--cache-dir /mnt/cache` | 지정하지 않으면 컨테이너 내부 `$HOME`에 쌓이고 `--rm`과 함께 사라집니다(§4.3) |
| `--image-timeout 60m` | 기본 10분으로는 대용량 이미지가 초과합니다(§4.3) |

진행 확인 : 프로세스가 `1`이고 용량이 늘어야 함. 프로세스가 `0`이면 끝난 것입니다

첫 단계인 `Extracting mirror archive(s)`는 373GB tar를 캐시로 푸는 작업이라 **로그가 한 줄에서 멈춰 있습니다.** 진행 여부는 로그가 아니라 캐시 용량으로 판단하세요. 추출이 끝나면 push 로그가 쏟아지기 시작합니다.

```bash
watch -n 10 'df -h / | tail -1; du -sh ~/mas-install/oc-mirror-cache ~/mirror-registry/quay 2>/dev/null; ps aux | grep -c "[o]c mirror"; tail -2 ~/mas-install/mirror/redhat-push.nohup.log'
```

⚠️ 디스크를 함께 봐야 합니다 — 미러 원본 489GB + 추출본 373GB + Registry 사본 373GB가 동시에 존재합니다. push가 끝나면 `~/mas-install/oc-mirror-cache`와 `mirror_000001.tar`를 지워 회수할 수 있습니다.

검증 : 마지막에 오류 없이 끝나야 하고 `cluster-resources`에 IDMS/ITMS YAML이 생성됩니다

```bash
tr '\r' '\n' < ~/mas-install/mirror/redhat-push.nohup.log | grep -aiE 'error|failed|context deadline' | head
ls -la ~/mas-install/mirror/redhat/working-dir/cluster-resources/
```

🔴 `cluster-resources/`는 **§3.8 Airgap(폐쇄망) 구성에서 그대로 씁니다.** §2.3 8번에서 비어 있던 이유가 이것입니다 — 목적지 Registry를 알아야 만들 수 있습니다.

| 파일 | 용도 |
|---|---|
| `idms-oc-mirror.yaml` | ImageDigestMirrorSet — 다이제스트 기반 경로 리다이렉션 |
| `itms-oc-mirror.yaml` | ImageTagMirrorSet — 태그 기반 |
| `cs-*-operator-index-v4-20.yaml` | **CatalogSource** — Red Hat / Certified / Community 오퍼레이터 카탈로그 |
| `cc-*-operator-index-v4-20.yaml` | ClusterCatalog (OLM v1용) |
| `signature-configmap.yaml` | 릴리스 서명 검증용 |

실행 : 캐시 삭제 — 추출본은 push가 끝나면 쓸모가 없습니다

```bash
rm -rf ~/mas-install/oc-mirror-cache
df -h /
```

⚠️ **재시도할 계획이 있으면 지우지 마세요.** 캐시가 있으면 이미 올린 이미지를 건너뜁니다. 오류 0건을 확인한 뒤에 지우세요.

#### 3.6.5 실행 — MAS 콘텐츠 push : core

반입한 4개 디렉터리를 **순차로** 올립니다. 앞 단계의 `[FAILURE]` 0건을 확인한 뒤 다음으로 넘어가세요 — 동시에 실행하면 SELinux 라벨이 충돌합니다(§4.1).

`--network host`가 필요한 이유는 §3.6.4와 같습니다. 컨테이너 기본 네트워크에서는 `127.0.0.1`을 가리키는 Bastion의 resolv.conf로 Registry 이름을 해석할 수 없습니다.

Catalog + MAS Core, 약 16GB.

```bash
nohup podman run --rm --network host \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/core \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-catalog --mirror-core \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/push-core.nohup.log" 2>&1 &

disown
```

진행 확인 : 프로세스가 `1`이어야 함. `0`이면 끝난 것입니다

```bash
watch -n 10 'df -h / | tail -1; du -sh ~/mirror-registry/quay; ps aux | grep -c "[m]as mirror-images"; tail -2 ~/mas-install/mirror/push-core.nohup.log'
```

검증 : `[SUCCESS] IBM Maximo Application Suite Core`, `[FAILURE]` 0건

```bash
tr '\r' '\n' < ~/mas-install/mirror/push-core.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

🔴 **core 직후 반드시 확인** — manifest 목적지가 실제 Registry로 재생성됐는지 봅니다. `0`이 아니면 이미지가 `localhost`를 가리킨 채 올라간 것이므로 **즉시 중단**하세요.

```bash
grep -c localhost ~/mas-install/mirror/core/manifests/from-filesystem/ibm-mas_9.2.0.txt
```

#### 3.6.6 실행 — MAS 콘텐츠 push : apps

Manage + Maximo IT, 약 25GB.

```bash
nohup podman run --rm --network host \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/apps \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-manage --mirror-icd \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/push-apps.nohup.log" 2>&1 &

disown
```

진행 확인

```bash
watch -n 10 'df -h / | tail -1; du -sh ~/mirror-registry/quay; ps aux | grep -c "[m]as mirror-images"; tail -2 ~/mas-install/mirror/push-apps.nohup.log'
```

검증 : `[SUCCESS] IBM Maximo Manage`, `[FAILURE]` 0건

```bash
tr '\r' '\n' < ~/mas-install/mirror/push-apps.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

#### 3.6.7 실행 — MAS 콘텐츠 push : other

Mongo / TSM / SLS / CFS / Db2, 약 43GB. 가장 오래 걸립니다.

```bash
nohup podman run --rm --network host \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/other \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-mongo --mirror-tsm --mirror-sls --mirror-cfs --mirror-db2 \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/push-other.nohup.log" 2>&1 &

disown
```

진행 확인

```bash
watch -n 10 'df -h / | tail -1; du -sh ~/mirror-registry/quay; ps aux | grep -c "[m]as mirror-images"; tail -2 ~/mas-install/mirror/push-other.nohup.log'
```

검증 : `[SUCCESS] Selected Dependencies`, `[FAILURE]` 0건

```bash
tr '\r' '\n' < ~/mas-install/mirror/push-other.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

#### 3.6.8 실행 — MAS 콘텐츠 push : cli

MAS CLI, 약 5GB.

```bash
nohup podman run --rm --network host \
  -v "$LOCAL_DIR":/mnt/registry:z \
  quay.io/ibmmas/cli:23.4.1 mas mirror-images \
  -m from-filesystem -d /mnt/registry/cli \
  -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
  -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
  -c v9-260625-amd64 -C 9.2.x \
  --mirror-cli \
  --ibm-entitlement "$IBM_ENTITLEMENT_KEY" \
  --no-confirm > "$HOME/mas-install/mirror/push-cli.nohup.log" 2>&1 &

disown
```

진행 확인

```bash
watch -n 10 'df -h / | tail -1; du -sh ~/mirror-registry/quay; ps aux | grep -c "[m]as mirror-images"; tail -2 ~/mas-install/mirror/push-cli.nohup.log'
```

검증 : `[SUCCESS] IBM Maximo CLI`, `[FAILURE]` 0건

```bash
tr '\r' '\n' < ~/mas-install/mirror/push-cli.nohup.log | grep -aE '\[SUCCESS\]|\[FAILURE\]'
```

#### 3.6.9 확인 및 검증

검증 : 4단계 전체에서 `[FAILURE]` **0건**

```bash
for f in ~/mas-install/mirror/push-*.nohup.log; do
  echo "===== $(basename $f)"
  tr '\r' '\n' < "$f" | grep -aE '\[SUCCESS\]|\[FAILURE\]'
done
```

검증 : Registry에 저장소가 올라와 있어야 함

```bash
curl -sk -u "$REGISTRY_USERNAME" https://$REGISTRY_HOST:$REGISTRY_PORT/v2/_catalog | head
unset REGISTRY_PASSWORD
```

### 3.7 OCP SNO 클러스터 설치 (사용자 계정)

Agent-based Installer로 SNO를 설치합니다.

#### 3.7.1 실행 — CLI 도구 준비

```bash
mkdir -p ~/.local/bin
tar -xzf ~/mas-install/ocp/openshift-client-linux.tar.gz  -C ~/mas-install/ocp
tar -xzf ~/mas-install/ocp/openshift-install-linux.tar.gz -C ~/mas-install/ocp
install -m 0755 ~/mas-install/ocp/{oc,kubectl,openshift-install} ~/.local/bin/

# §3.8 이후에도 새 셸에서 그대로 쓰이도록 영구 등록
cat >> ~/.bashrc <<'EOF'
export PATH="$HOME/.local/bin:$PATH"
export KUBECONFIG="$HOME/ocp-sno/auth/kubeconfig"
EOF
source ~/.bashrc

oc version --client && openshift-install version
```

#### 3.7.2 실행 — 설정 파일에 넣을 값 3개 수집

Pull Secret 병합 — Red Hat Pull Secret에 Mirror Registry 자격증명을 합칩니다.

```bash
mkdir -p ~/ocp-sno && cd ~/ocp-sno

cp ~/mas-install/redhat/pull-secret.json ./pull-secret-merged.json

printf '%s' "$REGISTRY_PASSWORD" | podman login \
  "$REGISTRY_HOST:$REGISTRY_PORT" \
  --username "$REGISTRY_USERNAME" --password-stdin \
  --authfile ./pull-secret-merged.json

jq 'del(.auths."cloud.openshift.com")' pull-secret-merged.json > tmp && mv tmp pull-secret-merged.json
chmod 600 ./pull-secret-merged.json

jq -r '.auths | keys[]' ./pull-secret-merged.json
```

`cloud.openshift.com`을 지우는 이유는 텔레메트리입니다. 폐쇄망에서는 어차피 닿지 않는데 자격증명이 있으면 Insights 오퍼레이터가 계속 전송을 시도해 **§3.7.8의 `oc get co` 검증에서 Degraded로 나타납니다.**

`quay.io`·`registry.redhat.io`는 남겨둡니다 — 나중에 추가 미러링이나 인터넷 개방 시 씁니다.

검증 : `registry.mas-it.itmsg.co.kr:8443` 이 목록에 있고 `cloud.openshift.com` 이 없어야 합니다

⚠️ **여기 넣는 계정은 클러스터 전체에 배포됩니다.** Red Hat은 쓰기 권한 계정을 쓰지 말라고 명시합니다.

> Do not use this image registry credentials file as the pull secret when you install a cluster. If you provide this file when you install cluster, all of the machines in the cluster will have write access to your mirror registry.

이번 배포는 **관리자 계정(`init`)을 그대로 사용합니다.** oc-mirror v2가 업스트림 네임스페이스를 그대로 재현해 Quay Organization이 31개가 되었고, Quay의 Robot은 소속 Organization 안에서만 권한을 받기 때문에 자격증명 하나로 전부 덮을 수 없습니다. 폐쇄망 내부 전용이라는 전제로 감수합니다.

설치 후에 좁힐 수 있습니다 — 읽기 전용 계정을 만들어 전역 pull secret만 교체하면 되고 재설치는 필요 없습니다.

```bash
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=<새 파일>
```

Registry CA — `additionalTrustBundle` 값:

```bash
cat "$(find /home/maximo/quay-install -name rootCA.pem -type f -print -quit)"
```

IDMS — `imageDigestSources`의 source/mirror 쌍. `spec.imageDigestMirrors` 항목을 옮겨 적습니다(구조 거의 동일).

```bash
cat ~/mas-install/mirror/redhat/working-dir/cluster-resources/idms-oc-mirror.yaml
```

#### 3.7.3 실행 — `install-config.yaml` 작성

미러 목록이 30개를 넘어 손으로 옮기면 반드시 틀립니다. §3.7.2의 값들을 그대로 읽어 파일을 생성합니다.

먼저 키 이름을 확인하세요 — 버전에 따라 `imageDigestSources` / `imageContentSources`로 갈립니다.

```bash
openshift-install explain installconfig | grep -iE 'imageDigestSources|imageContentSources'
```

형식) `baseDomain`·`metadata.name`·`machineNetwork`만 환경에 맞게 바꿉니다

```bash
cd ~/ocp-sno

cat > install-config.yaml <<EOF
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
    - cidr: <machine-network-cidr>
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
pullSecret: '$(jq -c . pull-secret-merged.json)'
sshKey: '$(cat ~/.ssh/quay_installer.pub)'
additionalTrustBundle: |
$(sed 's/^/  /' "$(find $HOME/quay-install -name rootCA.pem -type f -print -quit)")
imageDigestSources:
$(awk '/name: idms-release-0/{d=1} d&&/^  imageDigestMirrors:/{f=1;next} /^---/{f=0;d=0} f' ~/mas-install/mirror/redhat/working-dir/cluster-resources/idms-oc-mirror.yaml)
EOF
```

예시)

```bash
cd ~/ocp-sno

cat > install-config.yaml <<EOF
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
pullSecret: '$(jq -c . pull-secret-merged.json)'
sshKey: '$(cat ~/.ssh/quay_installer.pub)'
additionalTrustBundle: |
$(sed 's/^/  /' "$(find $HOME/quay-install -name rootCA.pem -type f -print -quit)")
imageDigestSources:
$(awk '/name: idms-release-0/{d=1} d&&/^  imageDigestMirrors:/{f=1;next} /^---/{f=0;d=0} f' ~/mas-install/mirror/redhat/working-dir/cluster-resources/idms-oc-mirror.yaml)
EOF
```

🔴 **`imageDigestSources`에는 release 미러 2개만 넣습니다.** 부트스트랩에 필요한 것이 그것뿐이고, 오퍼레이터 미러까지 넣으면 `openshift-install`의 저장소 이름 검증에 걸립니다.

```
FATAL invalid install-config configuration: imageDigestSources[23].source:
      Invalid value: "docker.io/grafana": the repository provided is invalid
```

`openshift-install`의 검증이 IDMS CRD보다 엄격해서 `docker.io/grafana` 같은 항목을 거부합니다. 오퍼레이터 미러는 §3.8에서 `idms-oc-mirror.yaml`을 그대로 `oc apply` 하면 되므로 여기 넣을 이유가 없습니다.

위 스크립트의 `awk`가 `idms-release-0` 문서만 골라내는 이유입니다.

검증 : `source:` 2건

```bash
grep -c 'source:' install-config.yaml
tail -8 install-config.yaml
```

검증 : 치환이 안 된 자리가 없어야 합니다

```bash
grep -nE '\$\(|<.*>' install-config.yaml
```

#### 3.7.4 실행 — `agent-config.yaml` 작성

형식)

```bash
cd ~/ocp-sno

cat > agent-config.yaml <<'EOF'
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: <cluster-name>
rendezvousIP: <sno-ip>
additionalNTPSources:
  - <bastion-ip>
hosts:
  - hostname: <sno-hostname>
    role: master
    interfaces:
      - name: <nic-name>
        macAddress: "<sno-mac>"
    rootDeviceHints:
      hctl: "<os-disk-hctl>"
    networkConfig:
      interfaces:
        - name: <nic-name>
          identifier: mac-address
          mac-address: "<sno-mac>"
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: <sno-ip>
                prefix-length: <prefix>
            dhcp: false
      dns-resolver:
        config:
          server:
            - <bastion-ip>
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: <gateway-ip>
            next-hop-interface: <nic-name>
EOF
```

예시)

```bash
cd ~/ocp-sno

cat > agent-config.yaml <<'EOF'
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
      - name: ens192
        macAddress: "00:50:56:bb:86:df"
    rootDeviceHints:
      hctl: "0:0:0:0"
    networkConfig:
      interfaces:
        - name: ens192
          identifier: mac-address
          mac-address: "00:50:56:bb:86:df"
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
            next-hop-interface: ens192
EOF
```

`<<'EOF'` 로 따옴표를 준 것은 치환할 값이 없기 때문입니다. `install-config.yaml`(§3.7.3)과 달리 셸 확장이 일어나면 안 됩니다.

검증 : 치환이 안 된 자리가 없어야 하고, 파일 3개가 준비돼 있어야 합니다

```bash
grep -nE '<.*>' agent-config.yaml
ls -la ~/ocp-sno
```

| 파일 | 만든 곳 |
|---|---|
| `pull-secret-merged.json` | §3.7.2 |
| `install-config.yaml` | §3.7.3 |
| `agent-config.yaml` | §3.7.4 |

🔴 **디스크 지정** — SNO는 디스크가 2개(300GB OS / 500GB LVMS용)입니다. OS를 500GB에 깔아버리면 LVMS가 쓸 빈 디스크가 없어져 **SNO 재설치**가 필요합니다.

`deviceName: /dev/sda`를 쓰지 마세요. 커널 인식 순서에 따라 달라지는데다 **항상 무언가에 매칭돼** 조용히 틀립니다.

| 힌트 | 판단 |
|---|---|
| **`hctl`** | SCSI `Host:Channel:Target:LUN`. vSphere의 가상 디바이스 노드와 직결되어 부팅 전에 확정 가능 |
| `wwn` / `serialNumber` | 물리 서버에 적합. VM은 `disk.EnableUUID` 필요 |
| `deviceName` | ❌ 순서 의존 |
| `minSizeGigabytes` | ❌ 조건을 만족하는 첫 디스크를 잡아 구분 불가 |

vSphere **설정 편집 → 하드 디스크 → 가상 디바이스 노드**에서 확인합니다. `SCSI(0:0)` → `hctl: "0:0:0:0"`, `SCSI(0:1)` → `hctl: "0:0:1:0"`.

힌트에 맞는 디스크가 없으면 설치 프로그램은 다른 디스크를 고르지 않고 **검증 실패로 멈춥니다.** 정확한 힌트를 주면 최악의 경우에도 데이터를 잃지 않습니다.

**NIC 이름** — VMXNET3는 보통 `ens192`로 잡히지만 보장은 아닙니다. `identifier: mac-address`를 넣으면 nmstate가 이름이 아니라 MAC으로 매칭하므로 이름이 달라도 설정이 적용됩니다.

⚠️ vSphere에서 **네트워크 어댑터가 "연결됨" + "전원을 켤 때 연결"** 인지 확인하세요. 끊겨 있으면 설치가 진행되지 않습니다.

🔴 **vSphere VM은 `disk.EnableUUID`를 반드시 켜야 합니다.** 없으면 부팅은 되지만 검증에서 멈춥니다.

```
{ID:vsphere-disk-uuid-enabled Status:failure
 Message:VSphere disk.EnableUUID isn't enabled for this virtual machine,
         it's necessary for disks to be mounted properly}
```

VM **전원을 끈 상태**에서 설정 편집 → **VM 옵션** → **고급** → **구성 매개변수 편집** → 매개변수 추가.

| 이름 | 값 |
|---|---|
| `disk.EnableUUID` | `TRUE` |

게스트가 디스크 UUID(`/dev/disk/by-id/`)를 볼 수 있게 하는 설정입니다. 전원이 켜져 있으면 추가할 수 없습니다.

#### 3.7.5 실행 — 백업

🔴 `openshift-install`은 두 yaml을 **읽고 나서 삭제합니다.** 백업 없이 실행하면 오타 하나를 고치려고 §3.7.2부터 다시 해야 합니다.

```bash
mkdir -p ~/ocp-sno-backup
cp install-config.yaml agent-config.yaml pull-secret-merged.json ~/ocp-sno-backup/
ls -la ~/ocp-sno-backup
```

검증 : 파일 3개가 복사돼 있어야 합니다

#### 3.7.6 실행 — ISO 생성

```bash
cd ~/ocp-sno
openshift-install agent create image --dir .
```

검증 : ISO가 1GB 내외로 생성되어야 합니다

```bash
ls -lh agent.x86_64.iso
```

두 yaml의 문법·값 오류가 이 단계에서 걸러집니다. 실패하면 백업에서 되돌려 고친 뒤 다시 실행하세요.

```bash
cp ~/ocp-sno-backup/*.yaml ~/ocp-sno/
```

#### 3.7.7 실행 — SNO 부팅

ISO를 SNO에 연결하고 부팅합니다.

| 환경 | 방법 |
|---|---|
| vSphere | 데이터스토어에 ISO 업로드 → VM 설정에서 CD/DVD를 **데이터스토어 ISO 파일**로 지정 → **전원을 켤 때 연결** 체크 → CD 부팅 |
| 물리 서버 | iDRAC/iLO 가상 미디어로 마운트 |

부팅하면 노드가 스스로 설치를 진행합니다. Bastion에서 진행 상황을 지켜봅니다.

```bash
cd ~/ocp-sno
openshift-install agent wait-for bootstrap-complete --dir . --log-level=info
```

이어서 설치 완료를 기다립니다. 전체 40~90분 걸립니다.

```bash
openshift-install agent wait-for install-complete --dir . --log-level=info
```

🔴 마지막에 출력되는 **kubeadmin 비밀번호를 기록하세요.** `~/ocp-sno/auth/kubeadmin-password`에도 저장됩니다.

진행이 멈춘 것처럼 보이면 노드 콘솔에서 직접 확인할 수 있습니다.

```bash
ssh -i ~/.ssh/quay_installer core@192.168.2.211
sudo journalctl -u assisted-service -f
```

⚠️ 네트워크 설정이 잘못되면 SSH도 안 되므로 **vSphere 콘솔에서 `ip a`** 로 확인하세요. `ens192`에 `192.168.2.211`이 붙어 있어야 합니다. 인터페이스 이름이 다르면 §3.7.4의 `name:`을 고치고 ISO부터 다시 만듭니다.

#### 3.7.8 확인 및 검증

검증 : 클러스터 버전이 `AVAILABLE=True`, `PROGRESSING=False` 이고 노드가 `Ready`

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig
oc get clusterversion
oc get nodes
```

초기화 중에는 `Unable to apply ...: some cluster operators are not available`이 계속 나옵니다 — 실패가 아니라 진행 상황입니다. 진척도는 이걸로 봅니다.

```bash
oc get clusterversion -o jsonpath='{.items[0].status.conditions[?(@.type=="Progressing")].message}{"\n"}'
```

검증 : `DEGRADED=True` 인 오퍼레이터가 없어야 합니다

```bash
oc get co | grep -vE 'True *False *False'
```

`kube-apiserver`·`kube-controller-manager`·`kube-scheduler`가 `NodeInstallerProgressing`으로 남는 것은 정상입니다. SNO는 노드가 하나라 static pod을 순차 교체하며 각각 몇 분씩 걸리고, 그 동안 `authentication`이 잠깐 `AVAILABLE=False`로 되돌아가기도 합니다. **`DEGRADED`가 `False`인지만 보세요.**

`openshift-samples`는 컬럼이 비어 있는 것이 정상입니다 — 외부 레지스트리에 닿지 못해 스스로 비활성화합니다.

```bash
oc get configs.samples.operator.openshift.io cluster -o jsonpath='{.spec.managementState}{"\n"}'   # Removed
```

검증 : OS가 300GB에 설치되고 **500GB는 `FSTYPE`·`MOUNTPOINT`가 비어 있어야** 합니다. 채워져 있으면 §3.9.1 LVMS를 진행할 수 없습니다

⚠️ `oc debug`는 `registry.redhat.io/rhel9/support-tools`를 받아오는데 그 미러 매핑은 §3.8에서 적용합니다. **여기서는 SSH로 확인하세요.**

```bash
ssh -i ~/.ssh/quay_installer core@<sno-ip> 'lsblk -o NAME,SIZE,HCTL,TYPE,FSTYPE,MOUNTPOINT'
```

검증 : 부트스트랩 미러가 반영됐는지 — `idms-release-0`이 있어야 합니다

```bash
oc get idms
```

🔴 마지막에 **`kubeadmin` 비밀번호를 기록하세요.**

```bash
cat ~/ocp-sno/auth/kubeadmin-password
```

### 3.8 Airgap(폐쇄망) 구성 (사용자 계정)

기본 상태의 OpenShift는 이미지를 `registry.redhat.io`·`quay.io`에서, OperatorHub 카탈로그를 `registry.redhat.io/redhat/redhat-operator-index`에서 가져옵니다. **이 참조를 전부 미러 Registry로 돌려놓는 단계**입니다. IBM CLI 명령 이름(`mas configure-airgap`)을 따라 Airgap 구성이라 부릅니다.

§3.7에서는 부트스트랩용 release 미러 2건만 넣었습니다. 나머지 오퍼레이터 미러 32건과 CatalogSource를 여기서 적용합니다.

이 단계 전에는 외부 이미지를 참조하는 명령이 모두 실패합니다 — `oc debug node/...`가 대표적입니다.

**두 부분으로 나뉘며 순서를 반드시 지키세요.** 둘 다 MachineConfig 롤아웃(노드 재부팅)을 유발하므로 합쳐서 30~60분 걸립니다.

#### 3.8.1 oc-mirror 생성 리소스 적용 + OperatorHub 기본 소스 비활성화

이 단계를 건너뛰면 cert-manager / DRO / Grafana 등 MAS 필수 Operator를 설치할 수 없어 §3.10이 실패합니다.

> `mas configure-airgap`에는 `--setup-redhat-catalogs`·`--setup-redhat-release` 플래그가 있어 같은 일을 대신할 수 있습니다. **쓰지 않습니다** — IBM CLI가 만드는 IDMS·CatalogSource는 일반형인데, 우리 미러는 `oc mirror`가 만든 실제 레이아웃(prefix 없이 `registry:8443/openshift4`, `/rhel9` …)을 따릅니다. `cluster-resources/`는 그 미러링 결과에서 생성돼 경로가 정확히 일치하고, 이것이 Red Hat 공식 절차이기도 합니다.

**1) 실행 — 환경변수**

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig
export REGISTRY_HOST=registry.mas-it.itmsg.co.kr
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=init
export REGISTRY_PASSWORD='T3AgRHSeYw6x5u0Z2q4OE9s81Koyd7tV'
export REGISTRY_CA="$(find "$HOME/quay-install" -name rootCA.pem -type f -print -quit)"

oc whoami && echo "CA $REGISTRY_CA"
```

**2) 실행 — 리소스 적용**

적용 전에 CatalogSource가 가리키는 경로가 실제 미러 레이아웃과 맞는지 확인합니다. `spec.image`가 `registry.<cluster-name>.<base-domain>:8443/redhat/redhat-operator-index:v4.20` 형태여야 합니다.

```bash
ls -la ~/mas-install/mirror/redhat/working-dir/cluster-resources/
cat ~/mas-install/mirror/redhat/working-dir/cluster-resources/cs-redhat-operator-index-v4-20.yaml
```

```bash
# 기본 OperatorHub 소스(인터넷 대상) 비활성화
oc patch OperatorHub cluster --type json \
  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

oc apply -f ~/mas-install/mirror/redhat/working-dir/cluster-resources/
```

> `cc-*.yaml`은 ClusterCatalog(OLM v1)입니다. CRD가 없는 버전이면 그 세 개만 실패하고 나머지는 정상 적용됩니다.

IDMS/ITMS 적용은 **MachineConfig 롤아웃(노드 재시작)** 을 유발합니다. SNO는 노드가 하나라 그동안 API가 끊깁니다 — 정상입니다.

진행 확인 : `master` MCP의 `CONFIG` 이름이 새로 바뀌고 `UPDATED=True`, `DEGRADED=False` 가 되어야 합니다

```bash
watch -n 30 'date; oc get mcp; echo; oc get nodes'
```

⚠️ `UPDATING=True` 를 못 볼 수도 있습니다. SNO에서는 1~2분 만에 끝나기도 해서, 지켜보기 시작했을 때 이미 완료돼 있는 경우가 있습니다. **완료 판정은 `CONFIG` 이름이 방금 만들어진 것인지로 하세요.**

```bash
oc get mc --sort-by=.metadata.creationTimestamp | tail -3   # AGE가 방금인 rendered-master-*
oc get co machine-config                                     # AVAILABLE=True, PROGRESSING=False
```

**3) 확인 및 검증**

검증 : 노드의 `registries.conf`에 미러가 반영돼야 합니다. IDMS 33건 + ITMS 1건 = **34**

```bash
ssh -i ~/.ssh/quay_installer core@<sno-ip> 'sudo grep -c "^\[\[registry\]\]" /etc/containers/registries.conf'
```

여기서 `0`이면 롤아웃이 아직 안 된 것입니다. 오브젝트 자체는 2건(`idms-release-0`, `idms-operator-0`)으로 보입니다 — 그 안에 항목이 33개 들어 있습니다.

```bash
oc get imagedigestmirrorset
oc get imagetagmirrorset
```

검증 : CatalogSource Pod 3개가 `Running` 이어야 합니다

```bash
oc get catalogsource -n openshift-marketplace
oc get pods -n openshift-marketplace
```

`DISPLAY`·`PUBLISHER`가 비어 있는 것은 정상입니다 — oc-mirror가 그 필드를 넣지 않습니다. Pod이 `ImagePullBackOff`면 인덱스 이미지가 Registry에 없거나 경로가 어긋난 것입니다.

```bash
oc logs -n openshift-marketplace -l olm.catalogSource=cs-redhat-operator-index-v4-20 --tail=30
```

검증 : MAS가 요구하는 Red Hat Operator가 조회되어야 합니다

```bash
oc get packagemanifest -n openshift-marketplace | grep -E 'cert-manager|grafana|ibm-data-reporter|ibm-metrics|lvms'
```

다섯 개가 모두 나와야 합니다. 전체 개수가 30개 미만인 것은 정상입니다 — imageset에서 고른 것만 미러링했으므로 전체 카탈로그가 아닙니다.

검증 : 외부 이미지가 미러를 통해 당겨지는지 — §3.7.8에서 실패했던 `oc debug`가 이제 동작해야 합니다

```bash
oc debug node/mas-it-sno -- chroot /host lsblk -o NAME,SIZE,HCTL,TYPE,FSTYPE,MOUNTPOINT
```

> IBM Maximo Operator Catalog는 이 시점에 **없는 것이 정상**입니다 — §3.10의 `mas install`이 생성합니다.

#### 3.8.2 mas configure-airgap 실행

Registry 접근 검증, 전역 Pull Secret, IDMS, 사용자 CA 신뢰를 구성합니다. MachineConfig 롤아웃으로 30~60분 걸릴 수 있습니다.

**1) 실행 — 환경변수**

§3.6.1과 같습니다. 셸이 바뀌었으면 다시 설정하세요.

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig
export REGISTRY_HOST=registry.mas-it.itmsg.co.kr
export REGISTRY_PORT=8443
export REGISTRY_USERNAME=init
export REGISTRY_PASSWORD='T3AgRHSeYw6x5u0Z2q4OE9s81Koyd7tV'
export REGISTRY_CA="$(find "$HOME/quay-install" -name rootCA.pem -type f -print -quit)"

echo "$REGISTRY_HOST:$REGISTRY_PORT / $REGISTRY_USERNAME / 비밀번호 ${#REGISTRY_PASSWORD}자 / CA $REGISTRY_CA"
```

**2) 실행 — 도움말 확인**

CLI는 클러스터에 로그인돼 있지 않으면 `--help`조차 출력하지 않습니다. **kubeconfig를 컨테이너에 물려줍니다** — 문서에 흔히 나오는 `oc login --token=<token>` 방식보다 낫습니다. 토큰을 따로 뽑을 필요가 없고 만료도 없습니다.

```bash
podman run --rm --network host \
  -v "$KUBECONFIG":/tmp/kubeconfig:ro,z \
  -e KUBECONFIG=/tmp/kubeconfig \
  quay.io/ibmmas/cli:23.4.1 mas configure-airgap --help
```

옵션 목록에 **`--setup-redhat-catalogs`** 가 있는지 확인하세요. 있으면 §3.8.1의 CatalogSource 적용을 이 명령이 대신할 수 있습니다.

**3) 실행 — configure-airgap**

```bash
nohup podman run --rm --network host \
  -v "$KUBECONFIG":/tmp/kubeconfig:ro,z \
  -v "$REGISTRY_CA":/mnt/registry-ca.pem:ro,z \
  -e KUBECONFIG=/tmp/kubeconfig \
  quay.io/ibmmas/cli:23.4.1 mas configure-airgap \
    -H "$REGISTRY_HOST" -P "$REGISTRY_PORT" \
    -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD" \
    --ca-file /mnt/registry-ca.pem \
    --no-confirm > "$HOME/mas-install/configure-airgap.log" 2>&1 &

disown
```

`--network host`가 필요한 이유는 §3.6.4와 같습니다 — 컨테이너 기본 네트워크에서는 Bastion의 resolv.conf(`127.0.0.1`)로 이름을 해석할 수 없습니다.

진행 확인 : 프로세스가 `1`이어야 함. `0`이면 끝난 것입니다

```bash
watch -n 15 'date; ps aux | grep -c "[c]onfigure-airgap"; echo; tail -5 ~/mas-install/configure-airgap.log; echo; oc get mcp 2>&1 | tail -2'
```

MachineConfig 롤아웃으로 노드가 재부팅하면 `oc`가 잠시 실패합니다 — 정상입니다.

**4) 실행 — IDMS 정책 충돌 해소**

🔴 **반드시 실행해야 합니다.** §3.8.1의 `idms-operator-0`(oc-mirror 생성)와 §3.8.2의 `mas-ibm-catalog`(configure-airgap 생성)가 **같은 source `icr.io/cpopen`에 다른 `mirrorSourcePolicy`** 를 지정합니다.

| IDMS | `icr.io/cpopen` 정책 |
|---|---|
| `idms-operator-0` | 기본값 (`AllowContactingSource`) |
| `mas-ibm-catalog` | `NeverContactSource` |

충돌하면 MCO가 **registries.conf 전체 렌더링을 중단**해서 `mas-ibm-catalog`의 매핑이 하나도 반영되지 않습니다(§4.7). 오브젝트는 정상으로 보이고 `oc get mcp`도 `UPDATED=True`라 그냥 넘어가기 쉽습니다.

```bash
IDX=$(oc get idms idms-operator-0 -o json | \
  jq -r '.spec.imageDigestMirrors | to_entries[] | select(.value.source=="icr.io/cpopen") | .key')
echo "index=$IDX"

oc patch idms idms-operator-0 --type=json \
  -p "[{\"op\":\"add\",\"path\":\"/spec/imageDigestMirrors/$IDX/mirrorSourcePolicy\",\"value\":\"NeverContactSource\"}]"
```

검증 : 충돌이 더 남아 있지 않아야 합니다. **다른 source에서 또 날 수 있습니다**

```bash
oc logs -n openshift-machine-config-operator deploy/machine-config-controller --tail=20 | grep -i conflicting
```

출력이 있으면 그 source 이름으로 위 patch를 한 번 더 실행하세요 (`select(.value.source=="<source>")`).

출력이 없으면 새 `rendered-master-*`가 생기고 노드가 재부팅됩니다.

```bash
watch -n 20 'date; oc get mc --sort-by=.metadata.creationTimestamp | tail -2; echo; oc get mcp | head -3'
```

⚠️ `quay.io/ibmmas` 등 부족해 보이는 매핑을 **IDMS로 따로 만들지 마세요.** `mas-ibm-catalog`에 이미 들어 있어 중복 정의가 되고 같은 충돌을 하나 더 만듭니다.

**5) 확인 및 검증**

🔴 **`oc get mcp`가 `UPDATED=True`인 것만으로는 판정할 수 없습니다.** MCO가 새 설정을 **만들지 못하면** 갱신할 것이 없어 그대로 `UPDATED=True`로 보입니다. 실제로 이번 배포에서 그렇게 통과 판정이 났고, MAS 설치 단계에 가서야 드러났습니다(§4.7).

검증 : 노드의 `registries.conf` 항목이 §3.8.1 때보다 **늘어야** 합니다 (34 → 43)

```bash
ssh -i ~/.ssh/quay_installer core@<sno-ip> 'grep -c "^\[\[registry\]\]" /etc/containers/registries.conf'
```

검증 : **새 `rendered-master-*`** 가 생성되어야 합니다 (`AGE`가 방금)

```bash
oc get mc --sort-by=.metadata.creationTimestamp | tail -2
oc get mcp
```

검증 : MCO 컨트롤러에 오류가 없어야 합니다

```bash
oc logs -n openshift-machine-config-operator deploy/machine-config-controller --tail=20 | grep -i error
```

`conflicting mirrorSourcePolicy` 오류가 보이면 §4.7로 가세요.

검증 : IDMS에 **`mas-ibm-catalog`** 가 추가돼 있어야 합니다 — 이 명령이 만든 것입니다

```bash
oc get imagedigestmirrorset
```

| 이름 | 만든 곳 |
|---|---|
| `image-digest-mirror` | §3.7 `install-config.yaml` |
| `idms-release-0` / `idms-operator-0` | §3.8.1 |
| `mas-ibm-catalog` | §3.8.2 ← **이번에 생겨야 하는 것** |

검증 : 전역 Pull Secret에 **미러 Registry 항목**이 있어야 합니다

```bash
oc get secret pull-secret -n openshift-config \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq -r '.auths | keys[]'
```

`registry.<cluster-name>.<base-domain>:8443` 이 목록에 나와야 합니다.

검증 : CA 신뢰 번들에 **인증서 본문**이 들어 있어야 합니다

```bash
oc get configmap -n openshift-config user-ca-bundle -o yaml | head -5
```

`ca-bundle.crt:` 아래에 `-----BEGIN CERTIFICATE-----` 가 보이면 정상입니다.

검증 : 미러 Registry에서 실제로 Pull이 되는지 — 이것이 §3.8 전체의 최종 확인입니다

```bash
oc delete pod test-pull --ignore-not-found

oc run test-pull --image=registry.mas-it.itmsg.co.kr:8443/ibmmas/cli:latest \
  --restart=Never --command -- sleep 60
```

```bash
oc get pod test-pull -w
```

`Running`이 되면 성공입니다. `ImagePullBackOff`면 원인을 봅니다.

```bash
oc describe pod test-pull | tail -20
```

| 메시지 | 원인 |
|---|---|
| `manifest unknown` | 경로·태그가 Registry에 없음. 인증·인증서는 정상 |
| `x509` / `certificate signed by unknown authority` | CA 신뢰 문제 — §3.8.2 재확인 |
| `401` / `unauthorized` | 전역 Pull Secret 문제 |

`manifest unknown`이면 실제 태그를 매니페스트에서 확인하세요. MAS CLI 이미지는 `:latest`로 올라갑니다.

```bash
grep -o "$REGISTRY_HOST:$REGISTRY_PORT/[^ ]*" ~/mas-install/mirror/cli/manifests/from-filesystem/*.txt | head
```

확인 후 정리합니다.

```bash
oc delete pod test-pull
```

### 3.9 스토리지 준비 (RWO + RWX StorageClass) (사용자 계정)

🔴 **필수 단계 (공식 문서 원문)**: *"MAS requires both a `ReadWriteMany` and a `ReadWriteOnce` capable storage class"* — 두 StorageClass가 **`mas install` 실행 전에 이미 클러스터에 존재해야** 합니다. `mas install`은 스토리지를 만들어주지 않고, 기존 StorageClass 목록에서 선택하게만 합니다. 준비 안 하면 이 단계에서 막힙니다.

```bash
oc get storageclass
oc get csv -A | grep -iE 'lvm|odf|ocs'
```

| 용도 | 요구 AccessMode | 이번 배포 방식 | 주 사용처 |
|---|---|---|---|
| RWO | `ReadWriteOnce` | **LVM Storage(LVMS) Operator** — StorageClass 이름은 deviceClass가 `vg1`이면 `lvms-vg1` | Db2, MongoDB 등 |
| RWX | `ReadWriteMany` | **외부 NFS — 이번 배포는 Bastion을 NFS 서버로 겸용** (§3.9.2 방식 A) | MAS Core/Manage 공유 볼륨 |

⚠️ **가장 흔한 실수**: LVM Storage는 **RWO 전용**입니다. RWX 자리에 `lvms-vg1`을 넣으면 설치 도중 또는 런타임에 PVC 바인딩이 실패합니다. RWX는 반드시 별도로 준비하세요.

> **Db2는 반드시 RWO를 사용합니다.** NFS 위에 Db2 데이터를 두면 안 되므로, `mas install`에서 RWO/RWX를 헷갈리지 않도록 주의하세요.

#### 3.9.1 RWO — LVM Storage (LVMS) Operator

SNO의 로컬 디스크를 RWO StorageClass로 제공합니다. `lvms-operator`(채널 `stable-4.20`)는 §2.3의 8번 미러링에 포함되어 있습니다(`imageset-ocp4.20.yml`).

**1) 실행 — 사전 확인**

2)와 3)에 넣을 값 세 개를 여기서 확정합니다.

검증 : StorageClass가 아직 없어야 정상입니다

```bash
oc get sc
```

검증 : `lvms-operator`가 조회되어야 합니다. 안 보이면 §3.8.1의 CatalogSource 적용 상태를 먼저 확인하세요

```bash
oc get packagemanifest lvms-operator -n openshift-marketplace \
  -o jsonpath='{.status.catalogSource}{"  채널: "}{range .status.channels[*]}{.name}{" "}{end}{"\n"}'
```

출력된 **CatalogSource 이름**과 **채널**을 2)의 `source`·`channel`에 넣습니다.

검증 : 500GB 디스크의 `FSTYPE`·`MOUNTPOINT`가 비어 있어야 합니다. 그 디바이스명을 3)의 `deviceSelector.paths`에 씁니다

```bash
oc debug node/mas-it-sno -- chroot /host lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

⚠️ 빈 디스크가 없으면 §3.7의 `rootDeviceHints`가 잘못 지정되어 OS가 500GB에 설치된 것입니다 — SNO 재설치가 필요합니다.

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
  source: <1)에서 확인한 CatalogSource 이름>
  sourceNamespace: openshift-marketplace
  channel: <1)에서 확인한 채널>
EOF
```

예시)

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
  source: cs-redhat-operator-index-v4-20
  sourceNamespace: openshift-marketplace
  channel: stable-4.20
EOF
```

검증 : 플레이스홀더가 그대로 들어가지 않았는지 확인합니다

```bash
oc get sub lvms -n openshift-storage -o custom-columns=SOURCE:.spec.source,CHANNEL:.spec.channel
```

```bash
oc get csv -n openshift-storage -w     # Succeeded 대기
```

**3) 실행 — LVMCluster 생성**

검증 : `apiVersion`을 확인합니다. **CRD는 2)의 오퍼레이터가 설치되면서 생기므로 여기서 조회해야 합니다**

```bash
oc explain lvmcluster | head -3
```

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
            - /dev/sdb        # 1)에서 확인한 실제 빈 디스크로 교체
        thinPoolConfig:
          name: thin-pool-1
          sizePercent: 90
          overprovisionRatio: 10
        fstype: xfs
EOF

oc get lvmcluster -n openshift-storage -w    # Ready 대기
```

**4) 확인 및 검증** — deviceClass 이름이 `vg1`이면 `lvms-vg1`이 생성됩니다.

```bash
oc get storageclass
oc get lvmcluster -n openshift-storage -o jsonpath='{.items[0].status}{"\n"}'
```

#### 3.9.2 RWX — 방식 선택

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

검증 : export가 보이고 서비스가 `active`, 방화벽에 `nfs`·`rpc-bind`·`mountd`가 있어야 합니다

```bash
showmount -e localhost
systemctl is-active nfs-server
firewall-cmd --list-services
```

검증 : SNO에서도 export가 보여야 합니다 — 여기까지 돼야 프로비저너가 마운트할 수 있습니다

```bash
ssh -i ~/.ssh/quay_installer core@<sno-ip> 'showmount -e <bastion-ip>'
```

**2) 실행 — 프로비저너 이미지 push** (사용자 계정)

§2.3의 10번에서 받아둔 자산을 사용합니다.

```bash
podman load -i ~/mas-install/nfs-provisioner/nfs-subdir-external-provisioner.tar
```

검증 : `registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2` 가 보여야 합니다

```bash
podman images | grep nfs-subdir
```

```bash
podman login "$REGISTRY_HOST:$REGISTRY_PORT" -u "$REGISTRY_USERNAME" -p "$REGISTRY_PASSWORD"

podman tag  registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2 \
            "$REGISTRY_HOST:$REGISTRY_PORT/sig-storage/nfs-subdir-external-provisioner:v4.0.2"
podman push "$REGISTRY_HOST:$REGISTRY_PORT/sig-storage/nfs-subdir-external-provisioner:v4.0.2"
```

검증 : Registry에서 다시 받아지는지 확인합니다

```bash
podman rmi "$REGISTRY_HOST:$REGISTRY_PORT/sig-storage/nfs-subdir-external-provisioner:v4.0.2"
podman pull "$REGISTRY_HOST:$REGISTRY_PORT/sig-storage/nfs-subdir-external-provisioner:v4.0.2"
```

**3) 실행 — YAML 작성 후 배포**

받은 YAML은 예제값이 들어 있어 그대로 쓸 수 없습니다. 바꿔야 할 곳은 이렇습니다.

| 파일 | 바꿀 곳 |
|---|---|
| `rbac.yaml` | `namespace: default` → 전용 namespace (**5곳**) |
| `deployment.yaml` | namespace, `image`, `NFS_SERVER`/`NFS_PATH`(env), `server`/`path`(volumes) |
| `class.yaml` | 없음 — `nfs-client`, provisioner 이름이 deployment의 `PROVISIONER_NAME`과 일치 |

⚠️ NFS 주소·경로가 **`env`와 `volumes` 두 곳에 중복**으로 들어갑니다. 한쪽만 고치면 Pod은 뜨는데 PVC가 엉뚱한 경로에 생깁니다.

🔴 **namespace를 먼저 만드세요.** 없으면 `oc apply`가 `namespaces "nfs-provisioner" not found`로 실패합니다.

```bash
cd ~/mas-install/nfs-provisioner
oc create ns nfs-provisioner
oc get ns nfs-provisioner
```

형식) `rbac.yaml` — `namespace`만 바꿉니다

```bash
cat > rbac.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: <provisioner-ns>
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: <provisioner-ns>
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: <provisioner-ns>
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: <provisioner-ns>
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: <provisioner-ns>
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
EOF
```

예시)

```bash
cat > rbac.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
EOF
```

형식) `deployment.yaml`

```bash
cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  namespace: <provisioner-ns>
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: <registry>:8443/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: <bastion-ip>
            - name: NFS_PATH
              value: <nfs-export-path>
      volumes:
        - name: nfs-client-root
          nfs:
            server: <bastion-ip>
            path: <nfs-export-path>
EOF
```

예시)

```bash
cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.mas-it.itmsg.co.kr:8443/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.2.210
            - name: NFS_PATH
              value: /export/mas-rwx
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.2.210
            path: /export/mas-rwx
EOF
```

검증 : 예제값(`10.3.243.101`, `/ifs/kubernetes`, `registry.k8s.io`, `default`)이 남아 있지 않아야 합니다

```bash
grep -nE '10\.3\.243\.101|/ifs/kubernetes|registry\.k8s\.io|namespace: default' rbac.yaml deployment.yaml
```

🔴 **SCC를 먼저 부여해야 합니다.** OpenShift 기본 `restricted` SCC는 **NFS 볼륨을 허용하지 않아** Pod 생성 자체가 거부됩니다.

```
Error creating: pods "nfs-client-provisioner-..." is forbidden:
unable to validate against any security context constraint:
spec.volumes[0]: Invalid value: "nfs": nfs volumes are not allowed to be used
```

```bash
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-client-provisioner -n nfs-provisioner
```

배포합니다.

```bash
oc apply -f rbac.yaml -f deployment.yaml -f class.yaml
```

> `oc apply` 시 PodSecurity 경고(`would violate PodSecurity "restricted:latest"`)가 나오는 것은 정상입니다 — 경고일 뿐 SCC 부여가 되어 있으면 Pod은 정상 생성됩니다.

**4) 확인 및 검증**

검증 : 프로비저너 Pod이 `1/1 Running` 이어야 합니다. NFS 마운트에 실패하면 여기서 걸립니다

```bash
oc get pods -n nfs-provisioner
```

검증 : StorageClass **두 개**가 있어야 합니다 — `lvms-vg1 (default)`(RWO, §3.9.1)와 `nfs-client`(RWX)

```bash
oc get sc
```

| 상태 | 원인 |
|---|---|
| Pod이 아예 없음 | SCC 미부여 — `oc get events -n nfs-provisioner --sort-by=.lastTimestamp \| tail -5` 로 확인 |
| `ContainerCreating`에서 멈춤 | NFS 마운트 실패 — Bastion의 export·방화벽, `deployment.yaml`의 `server`/`path` 확인 |
| `CrashLoopBackOff` | 이미지 또는 RBAC 문제 — `oc logs -n nfs-provisioner deploy/nfs-client-provisioner` |


---
##### 방식 B — ODF (OpenShift Data Foundation) — 대안

**장점**: Red Hat 지원 제품이고 CephFS로 RWX를 제공하며, Bastion에 런타임 의존하지 않습니다.
**단점**: **SNO에서 리소스 요구량이 매우 큽니다**(CPU/메모리/추가 디스크). 단일 노드에 MAS + Db2 + ODF를 모두 올리는 구성은 사이징을 신중히 확인해야 합니다.

- **Operator 이미지**: §2.3의 8번(Red Hat 콘텐츠) 미러링에 이미 포함되어 있습니다 — `imageset-ocp4.20.yml`의 `odf-operator`, `ocs-operator`, `mcg-operator`, `rook-ceph-operator`, `cephcsi-operator` 등으로 확인됨.

배포 시:

```bash
oc get packagemanifest -n openshift-marketplace | grep -iE 'odf|ocs'
# ODF Operator 설치 → StorageSystem 생성 → StorageClass 확인
oc get storageclass | grep -i cephfs   # RWX용: ocs-storagecluster-cephfs
```

---

#### 3.9.3 스토리지 검증 — PVC 바인딩 + 쓰기 테스트

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
```

```bash
oc get pvc -n storage-test
```

검증 : `test-rwx`는 **`Bound`**, `test-rwo`는 **`Pending`** 이 정상입니다

`lvms-vg1`은 `VOLUMEBINDINGMODE=WaitForFirstConsumer`라 Pod이 붙을 때 볼륨이 생성됩니다. `test-rwo`의 `Pending`은 오류가 아니라 2)에서 해소됩니다.

`test-rwx`가 `Pending`이면 그건 문제입니다 — 프로비저너 미동작, NFS export 권한 등을 확인합니다.

```bash
oc describe pvc -n storage-test test-rwx | tail -10
oc logs -n nfs-provisioner deploy/nfs-client-provisioner --tail=20
```

**2) 실행 — 쓰기 테스트**

NFS는 마운트만 되고 쓰기가 실패하는 경우가 흔하므로 반드시 확인합니다. **두 볼륨을 한 Pod에 붙여** RWO·RWX를 함께 검증하고, 동시에 1)에서 `Pending`이던 `test-rwo`도 해소합니다.

이미지는 §3.8.2 Pull 테스트에서 확인된 `ibmmas/cli:latest`를 씁니다.

⚠️ `rhel9/support-tools:latest` 는 쓸 수 없습니다. `oc debug`가 그 이미지를 쓰긴 하지만 **릴리스 페이로드의 다이제스트로 당기는 것**이라, 미러에 `latest` 태그로는 존재하지 않습니다(`repository not found`).

```bash
oc delete pod test-writer -n storage-test --ignore-not-found

cat <<'EOF' | oc apply -n storage-test -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-writer
spec:
  restartPolicy: Never
  containers:
    - name: writer
      image: registry.mas-it.itmsg.co.kr:8443/ibmmas/cli:latest
      command: ["/bin/sh","-c","echo rwx-ok > /rwx/test.txt && echo rwo-ok > /rwo/test.txt && cat /rwx/test.txt /rwo/test.txt"]
      volumeMounts:
        - {name: rwx, mountPath: /rwx}
        - {name: rwo, mountPath: /rwo}
  volumes:
    - name: rwx
      persistentVolumeClaim: {claimName: test-rwx}
    - name: rwo
      persistentVolumeClaim: {claimName: test-rwo}
EOF
```

**3) 확인 및 검증**

검증 : PVC **둘 다 `Bound`**, Pod은 `Completed`

```bash
oc get pvc,pod -n storage-test
```

검증 : 로그에 `rwx-ok`, `rwo-ok` 두 줄이 나와야 합니다

```bash
oc logs -n storage-test test-writer
```

검증 : Bastion의 NFS export에 파일이 실제로 생겨야 합니다

```bash
ls -R /export/mas-rwx/
```

정리합니다.

```bash
oc delete ns storage-test
```

#### 3.9.4 OpenShift 내부 Image Registry 구성

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

검증 : PVC가 `Bound`

```bash
oc get pvc -n openshift-image-registry
```

검증 : `image-registry-*` Pod이 `Running`

```bash
oc get pods -n openshift-image-registry
```

검증 : `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False`

```bash
oc get co image-registry
```

검증 : `managementState`가 `Managed`로 바뀌고 PVC가 연결돼야 합니다

```bash
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.managementState}{"\n"}{.spec.storage}{"\n"}'
```

### 3.10 MAS 설치 — Core + Manage + Maximo IT (사용자 계정)

#### 3.10.1 실행 — 사전 준비

**1) 실행 — 설정 디렉터리 생성**

`mas install`이 JDBC 설정 파일을 여기에 만듭니다. 미리 만들어 두지 않으면 podman이 root 소유로 생성해 쓰기가 막힙니다.

```bash
mkdir -p ~/mas-install/mas-config
```

**2) 확인 및 검증**

검증 : `cluster-admin` 권한이 있어야 합니다 — `yes`

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig
oc auth can-i '*' '*' --all-namespaces
```

검증 : 라이선스 파일과 Entitlement Key가 있어야 합니다

```bash
ls -la ~/mas-install/licenses/
```

검증 : StorageClass가 **RWO·RWX 두 개** 있어야 합니다 (§3.9.1, §3.9.2)

```bash
oc get sc
```

검증 : 내부 Image Registry가 **`Managed`** 여야 합니다 (§3.9.4). `Removed`면 Manage 이미지 빌드가 실패합니다

```bash
oc get co image-registry
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.managementState}{"\n"}'
```

검증 : 폐쇄망 미러 매핑이 **노드에 반영**돼 있어야 합니다 (§3.8.2). 오브젝트 존재만으로는 부족합니다

```bash
ssh -i ~/.ssh/quay_installer core@<sno-ip> 'grep -c "^\[\[registry\]\]" /etc/containers/registries.conf'
oc logs -n openshift-machine-config-operator deploy/machine-config-controller --tail=20 | grep -i conflicting
```

`conflicting` 출력이 있으면 §3.8.2의 4)를 먼저 처리하세요.

검증 : 미러 Registry에서 CLI 이미지를 받을 수 있어야 합니다

```bash
export REGISTRY=registry.mas-it.itmsg.co.kr:8443
podman pull "$REGISTRY/ibmmas/cli:latest"
```

#### 3.10.2 실행 — mas install

라이선스 디렉터리와 kubeconfig를 마운트합니다. 대화형이라 `-ti`가 필요합니다.

```bash
mkdir -p ~/mas-install/mas-config

export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)

podman run -ti --rm --network host \
  -v "$HOME/mas-install/licenses":/mnt/licenses:ro,z \
  -v "$HOME/mas-install/mas-config":/mnt/config:z \
  -v "$KUBECONFIG":/tmp/kubeconfig:ro,z \
  -e KUBECONFIG=/tmp/kubeconfig \
  -e IBM_ENTITLEMENT_KEY \
  quay.io/ibmmas/cli:23.4.1 mas install
```

`Select Local configuration directory` 프롬프트에는 **`/mnt/config`** 를 입력합니다. `mas install`이 설치 설정을 파일로 남기는데, 쓰기 가능한 호스트 디렉터리를 마운트해두지 않으면 `--rm`과 함께 사라집니다. 라이선스 디렉터리는 `:ro`라 쓸 수 없습니다.

⚠️ `$HOME`을 통째로 마운트하지 마세요 — SELinux가 홈 디렉터리 재라벨링을 거부합니다.

```
Error: SELinux relabeling of /home/maximo is not allowed
```

`oc login --token=<token>` 방식보다 낫습니다 — 토큰을 뽑을 필요도, 만료도 없습니다(§3.8.2와 동일).

⚠️ **SSH가 끊기면 중단됩니다.** 대화형이라 `nohup`을 쓸 수 없으므로 `tmux`나 `screen` 안에서 실행하는 것을 권합니다.

```bash
tmux new -s masinstall
```

**대화형 입력 — 실제 프롬프트 순서** (CLI 23.4.1 실측)

| # | 프롬프트 | 입력 | 이유 |
|---|---|---|---|
| 1 | `Set Target OpenShift Cluster` — `Proceed with this cluster?` | `y` | kubeconfig 마운트로 이미 연결됨 |
| 2 | `Choose Install Mode` — `Show advanced installation options?` | **`n`** (Simplified) | 아래 참고 |
| 3 | `IBM Maximo Operator Catalog Selection` | **`v9-260625-amd64`** | §2.3에서 미러링한 카탈로그. 다른 것을 고르면 이미지가 없어 실패 |
| 3-1 | `Select release` | **`9.2`** | `9.2-feature`가 아닙니다. feature 릴리스는 다음 GA까지만 지원되는 평가판이고, §2.3에서 채널 `9.2.x`로 미러링했습니다 |
| 4 | `License Terms` — `Do you accept the license terms?` | `y` | `MAS92` / **`MAXIT92`** / `MAXESRI92` 세 건 제시 |
| 5 | `Configure Storage Class Usage` — `Use the auto-detected storage classes?` | **`n`** | 자동 감지가 RWO도 `nfs-client`로 잡음 |
| 5-1 | Storage class (ReadWriteOnce) | **`lvms-vg1`** | NFS를 RWO로 쓰면 Db2·MongoDB 성능이 떨어집니다 |
| 5-2 | Storage class (ReadWriteMany) | **`nfs-client`** | |
| 6 | `Configure AppPoint Licensing` — `License file` | **`/mnt/licenses/<라이선스 파일>`** | 호스트 경로가 아니라 **컨테이너 안 경로** |
| 6-1 | `Contact e-mail address` / `first name` / `last name` | 담당자 정보 | SLS 등록 정보 |
| 7 | `Configure MAS Superuser Account` — Username / Password | 계정/비밀번호 | 초기 구성과 관리자 계정 생성에만 사용. **비대화형 실행에서는 묻지 않고 랜덤 생성**됩니다(§3.10.3) |
| 8 | `Configure IBM Container Registry` — `IBM entitlement key` | 키 붙여넣기 | `cat ~/mas-install/licenses/entitlement_key.key` |
| 8-1 | `Warning: IBM entitlement key validation failed` | **`2` (Continue anyway)** | 🔴 폐쇄망에서는 **정상**입니다. CLI가 `cp.icr.io`로 키를 검증하려 하는데 나갈 수 없습니다. 키는 §2.3 미러링에서 이미 검증됐고 여기서는 Pull Secret 구성에만 쓰입니다 |
| 9 | `Configure Operational Mode` | **`2` (Non-Production)** | 🔴 설치 후 변경 불가 |
| 10 | `Application Selection` | **Manage만 `y`** | IoT·Monitor·Optimizer·Visual Inspection·Facilities·AI Service·Assist는 모두 `n`. §2.3에서 Manage만 미러링했으므로 다른 것을 고르면 이미지가 없어 실패 |
| 11.1 | `Maximo Manage Components` — `Select components to enable?` | **`y`** | 🔴 기본값은 **Health만** 켜집니다. `n`으로 두면 **Maximo IT가 빠집니다** |
| 11.1-1 | 컴포넌트 목록 | **`Maximo IT`만 `y`** | 나머지 전부 `n` — ACM, Aviation, Civil Infrastructure, Envizi, Health, HSE, Nuclear, Oil & Gas, Oracle/SAP Connector, Service Provider, Spatial, Strategize, Transportation, Tririga, Utilities, Workday, AIP, Collaborate |
| 11.2 | `Maximo IT License Terms` | `y` | 이 화면이 나오면 **Maximo IT가 제대로 선택된 것**입니다. ⚠️ CLI가 안내하는 링크는 `MAXIT81-License`(8.1)로 오래된 것이며, 실제 조건은 4)의 `MAXIT92-License` |
| 11.3 | `Maximo Manage Settings - Customization` — `Include customization archive?` | `n` | 신규 설치라 커스터마이징 아카이브가 없습니다 |
| — | `Maximo Manage Settings - Database` | **나오지 않음** | 13.1에서 `n`(외부 DB)을 고를 때만 표시됩니다. 전용 Db2로 가면 `maximo` / `maxdata` / `maxindex` 기본값이 자동 적용되고, 한국어 저장에 필요한 `db2Vargraphic: true`도 자동으로 켜집니다 |
| 12 | `Configure MongoDb` — `Create MongoDb cluster using MongoDb CE Operator?` | `y` | §2.3에서 `--mirror-mongo`로 확보. 기존 MongoDB 없음 |
| 13.1 | `Database Configuration for Maximo Manage` — `Create Manage dedicated Db2 instance using the IBM Db2 Universal Operator?` | 🔴 **`y`** | 아래 참고 |
| 13.1-1 | `Db2 License file` | **Enter (빈 값)** | Db2 v12부터 라이선스 활성화 키를 묻지만, MAS에 포함되어 제공되므로 별도 파일이 없습니다. 화면에도 "없으면 Enter로 계속"이라고 안내합니다 |
| 13.2 | `Configure Db2 instance?` | `y` → 기본값 | 화면에는 CPU 4000m·볼륨 425Gi로 보이지만 **실제로는 `server-bundle-size: dev` 기준(CPU 300m, 볼륨 합계 60Gi)이 적용**됩니다. SNO에는 이쪽이 적합하며 `lvms-vg1`이 확장 가능이라 나중에 늘릴 수 있습니다 |
| 13.2-1 | `Set node affinity/tolerations?` | `n` | SNO는 노드가 하나 |
| 13.2-2 | `Set database in HADR mode?` | `n` | primary/standby 두 벌이 같은 노드에 올라가 자원만 두 배 소모 |
| — | `Select Local configuration directory` | **`/mnt/config`** | 쓰기 가능한 마운트여야 합니다 |
| 14 | `Maximo AI Configuration` — `Do you want to configure AiCfg?` | `n` | AI Service를 설치하지 않았고 관련 이미지도 미러링 대상이 아닙니다 |
| — | `Install Grafana?` | `y` | §2.3 8번에 `grafana-operator` 포함. MAS Core 표준 구성 |
| 15 | `Advanced Settings` — `Configure Guided Tour?` | `n` | 기본값 유지 |
| 15-1 | `Set Manage Server Timezone?` | `y` → `Asia/Seoul` | ⚠️ 실측에서 이 입력이 **반영되지 않고** `GMT`로 설치됐습니다. §3.10.3의 `--manage-server-timezone` 으로 넘기는 것이 확실합니다 |
| 15-2 | `Use additional configurations?` | `n` | 추가로 적용할 YAML이 없습니다 |
| 16 | `Non-Interactive Install Command` | — | 🔴 **출력되는 명령을 반드시 보관하세요.** 재설치·다른 사이트 배포에 그대로 씁니다 |
| 17 | `Review Settings` — `Proceed with these settings?` | `y` | 아래 확인 항목 참고 |
| 18 | `Disconnected OpenShift Preparation` — `Enter quay.io/ibmmas/cli:23.4.1 image digest` | 다이제스트 | 폐쇄망이라 CLI가 `quay.io`에서 조회하지 못합니다 |

🔴 **18번 다이제스트를 손으로 붙여넣으면 공백이 섞여 파이프라인이 즉시 실패합니다.** §3.10.3의 `--use-cli-digest` 방식을 쓰세요.

**대화형은 언어·데모 데이터·타임존을 묻지 않거나 반영하지 않습니다.** 이 셋이 필요하면 §3.10.3의 비대화형 실행을 쓰세요 — 나중에 추가하려면 이미지 재빌드나 DB 재생성이 필요합니다.

#### 3.10.3 실행 — 비대화형 (권장)

**신규 설치·재설치 모두 이 방식을 권장합니다.** 프롬프트 20여 개를 입력할 필요가 없고, 대화형이 묻지 않는 언어·데모 데이터·타임존까지 한 번에 지정하며, 다이제스트를 `--use-cli-digest`로 넘겨 붙여넣기 공백 사고를 막습니다. 대화형(§3.10.2)은 화면 흐름을 확인할 때만 참고하세요.

사전 조건은 §3.10.1과 동일합니다 — `cluster-admin` 권한, 라이선스 파일, `~/mas-install/mas-config` 디렉터리.

실패한 PipelineRun이 있으면 먼저 지웁니다.

```bash
oc get pipelinerun -n mas-inst1-pipelines
oc delete pipelinerun <이름> -n mas-inst1-pipelines
```

🔴 **다이제스트는 미러에 있는 값을 써야 합니다.** `quay.io` 원본 이미지의 다이제스트와 다릅니다 — `mas mirror-images`가 push하면서 매니페스트를 다시 만들기 때문입니다. 원본 값을 쓰면 IDMS가 미러로 보내줘도 `manifest unknown`이 납니다.

```bash
export REGISTRY=registry.mas-it.itmsg.co.kr:8443
export IBM_ENTITLEMENT_KEY=$(cat ~/mas-install/licenses/entitlement_key.key)

podman pull "$REGISTRY/ibmmas/cli:latest"
export CLI_DIGEST=$(podman image inspect "$REGISTRY/ibmmas/cli:latest" \
  --format '{{index .RepoDigests 0}}' | cut -d@ -f2)
```

검증 : 공백이 없어야 합니다 — 대괄호가 바로 붙어야 정상

```bash
echo "[$CLI_DIGEST]"
```

예시)

```bash
podman run -ti --rm --network host \
  -v "$HOME/mas-install/licenses":/mnt/licenses:ro,z \
  -v "$HOME/mas-install/mas-config":/mnt/config:z \
  -v "$KUBECONFIG":/tmp/kubeconfig:ro,z \
  -e KUBECONFIG=/tmp/kubeconfig \
  -e IBM_ENTITLEMENT_KEY \
  quay.io/ibmmas/cli:23.4.1 \
  mas install --mas-catalog-version v9-260625-amd64 --ibm-entitlement-key "$IBM_ENTITLEMENT_KEY" \
    --mas-channel 9.2.x --mas-instance-id inst1 --mas-workspace-id ws1 --mas-workspace-name "Maximo IT" \
    --non-prod --admin-mode cluster --routing subdomain --servicemesh false \
    --mas-issuer-kind ClusterIssuer \
    --storage-class-rwo lvms-vg1 --storage-class-rwx nfs-client \
    --storage-pipeline lvms-vg1 --storage-accessmode ReadWriteOnce \
    --license-file /mnt/licenses/lincense_poc.dat \
    --contact-email bsw78@itmsg.co.kr --contact-firstname seungwoo --contact-lastname baek \
    --mongodb-namespace mongoce \
    --manage-channel 9.2.x --manage-jdbc workspace-application \
    --manage-components "base=latest,icd=latest" --manage-server-bundle-size dev \
    --manage-base-language EN --manage-secondary-languages "KO" \
    --manage-demodata \
    --manage-server-timezone Asia/Seoul \
    --db2-manage --db2-channel v120104.0 --db2-namespace db2u --db2-type db2wh \
    --db2-timezone Asia/Seoul \
    --db2-cpu-requests 300m --db2-cpu-limits 6000m \
    --db2-memory-requests 8Gi --db2-memory-limits 12Gi \
    --db2-backup-storage 10Gi --db2-data-storage 20Gi \
    --db2-logs-storage 10Gi --db2-meta-storage 10Gi --db2-temp-storage 10Gi \
    --use-cli-digest "$CLI_DIGEST" \
    --accept-license --no-confirm
```

🔴 **비대화형 실행은 Superuser를 묻지 않고 CLI가 랜덤 생성합니다.** 대화형에서 입력했던 값은 적용되지 않으므로, 설치 후 시크릿에서 실제 값을 확인해야 합니다.

```bash
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.username}' | base64 -d; echo
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.password}' | base64 -d; echo
```

명령은 **PipelineRun을 제출하고 끝납니다.** 이후는 클러스터 안에서 Tekton이 진행하므로 SSH가 끊겨도 무관합니다.

진행 확인 : Task가 순차로 `Succeeded`가 되어야 합니다

```bash
watch -n 30 'date; oc get pipelinerun -n mas-inst1-pipelines; echo; oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp | tail -6'
```

Task 순서와 대략적인 소요입니다.

| Task | 내용 | 소요 |
|---|---|---|
| `pipeline-start` ~ `suite-certs` | 사전 검증, 카탈로그, cert-manager, Grafana, MongoDB, DRO, SLS | 20~30분 |
| **`db2-manage`** | Db2 인스턴스 생성·기동 | **15~40분** |
| `suite-install` / `suite-config` / `suite-verify` | MAS Core | 20~40분 |
| **`app-install` 계열** | **Manage + Maximo IT** — 이미지 빌드 포함 | **1~2시간** |

⚠️ `db2-manage`의 `FAILED - RETRYING: ... Wait for Db2uCluster instance to be ready`는 **실패가 아니라 5분 간격 재시도**입니다(최대 25회). Db2 자체 상태로 판단하세요.

```bash
oc get db2ucluster -n db2u
oc get pods -n db2u | grep db2u-0
```

실행 중인 Task의 로그를 보려면 Pod을 지정합니다.

```bash
TR=$(oc get taskrun -n mas-inst1-pipelines --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{range .items[?(@.status.conditions[0].status=="Unknown")]}{.metadata.name}{"\n"}{end}' | tail -1)
echo "$TR"

oc logs -n mas-inst1-pipelines "$(oc get pod -n mas-inst1-pipelines -l tekton.dev/taskRun=$TR -o name | head -1)" \
  --all-containers --tail=30
```

**`app-install` 단계 상세 확인 — Manage 이미지 빌드부터 서버 기동까지.** 이 단계는 Tekton TaskRun 하나로는 안이 안 보이고, 실제로는 `mas-inst1-manage` 네임스페이스 안에서 아래 순서로 진행됩니다(총 1~2시간).

| 순서 | 확인 대상 | 무슨 일 | 소요 |
|---|---|---|---|
| 1 | `admin-build-config-*` (Build) | Admin 번들 이미지 빌드 → 내부 Image Registry에 push | ~10분 |
| 2 | `all-build-config-*` (Build) | 서버 번들(Maximo IT 포함) 이미지 빌드 → push | ~15분 |
| 3 | `inst1-ws1-manage-maxinst-*` (Pod) | 방금 만든 이미지로 **DB 스키마 생성** — Db2에 테이블·인덱스 생성, `demodata`면 데모데이터까지 적재 | 90분~2시간 |
| 4 | `*-all-*` (Deployment) | maxinst 완료 후 생성 — 실제 Maximo 서버(Liberty) 기동 | 수분~10여 분 |

```bash
# 지금 어느 순서인지 한눈에
oc get build,pods -n mas-inst1-manage
```

- 빌드 2개가 `Complete`인데 `-all-*` Pod이 안 보이면 → **3번(maxinst) 진행 중**이 정상입니다. `*-all-*`은 maxinst가 끝나야 생성됩니다.
- 각 단계 로그:

```bash
# 1, 2 — 빌드 로그
oc logs -f build/all-build-config-1 -n mas-inst1-manage --tail=30
oc logs -f build/admin-build-config-1 -n mas-inst1-manage --tail=30

# 3 — maxinst 진행 상황 (계속 새 줄이 찍히면 정상, 몇 분째 멈춰 있으면 문제)
oc logs -n mas-inst1-manage deploy/inst1-ws1-manage-maxinst -f --tail=20

# 3 완료 판정 — 상태가 0/1 Completed로 바뀌면 끝
oc get pods -n mas-inst1-manage | grep maxinst

# 4 — 서버 기동. 컨테이너 로그엔 대기 루프만 찍히므로 Pod 안 /logs/messages.log를 봐야 함
POD=$(oc get pods -n mas-inst1-manage -o name | grep -- '-all-' | head -1)
oc exec -n mas-inst1-manage "${POD#pod/}" -c all -- tail -30 /logs/messages.log
```

기동 완료 표식(4번): `CWWKF0011I: The defaultServer server is ready to run a smarter planet.`

전체 완료 판정은 `ManageWorkspace` 조건으로 봅니다.

```bash
oc get manageworkspace inst1-ws1 -n mas-inst1-manage \
  -o jsonpath='{range .status.conditions[*]}{.type}: {.status} — {.message}{"\n"}{end}'
```

더 자세한 Pod 역할·명령은 [OCP_COMMAND.md §16~17](OCP_COMMAND.md#16-이미지-빌드-manage)을 참고하세요.

웹 콘솔이 더 보기 편합니다 — `mas install`이 마지막에 URL을 출력합니다.

```
https://console-openshift-console.apps.<cluster>.<domain>/k8s/ns/mas-inst1-pipelines/tekton.dev~v1beta1~PipelineRun/<이름>
```

검증 : MAS Core가 올라오면 Suite가 `Ready`가 됩니다

```bash
oc get suite -A
oc get route -n mas-inst1-core
```

#### 3.10.4 확인 및 검증

검증 : Suite가 생성되고 `Ready` 상태여야 합니다

```bash
oc get suite -A
```

검증 : Core 네임스페이스에 비정상 Pod이 없어야 합니다 (헤더만 남으면 정상)

```bash
oc get pods -n mas-inst1-core | grep -vE 'Running|Completed'
```

검증 : IBM Maximo Operator Catalog가 생성돼야 합니다 — `mas install`이 만듭니다

```bash
oc get catalogsource -n openshift-marketplace
oc get subscriptions -A | grep -i ibm
```

검증 : Suite Administration Route가 있어야 합니다 — §3.11에서 이 주소로 접속합니다

```bash
oc get route -A | grep -E 'admin|mas'
```

🔴 노트북에서 접속하려면 `hosts`에 해당 이름을 추가해야 합니다 — `*.apps` 와일드카드는 hosts 파일이 지원하지 않습니다.

```
192.168.2.211    admin.inst1.apps.mas-it.itmsg.co.kr
```

#### 3.10.5 실패 시 초기화 — Manage 재설치

🔴 **잔여 오브젝트를 남기면 다음 설치가 조용히 어긋납니다.** 오퍼레이터와 Ansible 역할이 "이미 구성됨"으로 판단해 필요한 단계를 건너뛰기 때문입니다. 실제로 `db2u`의 ConfigMap 하나를 남겼다가 Db2 설정 적용 단계가 통째로 스킵되어 파이프라인이 멈췄습니다.

**범위** — OCP·MAS Core·MongoDB·미러 레지스트리·스토리지는 그대로 두고 **Manage와 Db2만** 지웁니다.

**1) 실행 — 파이프라인 정리**

```bash
oc delete pipelinerun --all -n mas-inst1-pipelines --wait=false
```

**2) 실행 — Manage 워크스페이스·애플리케이션 제거**

```bash
oc delete manageworkspace --all -n mas-inst1-manage --wait=false
oc delete manageapp --all -n mas-inst1-manage --wait=false
sleep 30
```

finalizer 때문에 30초 뒤에도 남아 있으면 직접 제거합니다.

```bash
oc get manageworkspace,manageapp -n mas-inst1-manage
for r in manageworkspace manageapp; do
  for n in $(oc get $r -n mas-inst1-manage -o name 2>/dev/null); do
    oc patch $n -n mas-inst1-manage --type=merge -p '{"metadata":{"finalizers":null}}'
  done
done
```

**3) 실행 — Manage 잔여물 제거**

빌드·이미지스트림·설정이 남으면 다음 설치가 옛 값을 재사용합니다.

```bash
oc delete build,bc,is --all -n mas-inst1-manage --wait=false
oc delete cm,secret -n mas-inst1-manage -l mas.ibm.com/instanceId=inst1 --wait=false
oc delete deploy,statefulset -n mas-inst1-manage -l mas.ibm.com/instanceId=inst1 --wait=false
```

**4) 실행 — Db2 제거**

🔴 **ConfigMap과 Secret까지 지워야 합니다.** Db2uCluster와 PVC만 지우면 다음 설치가 설정 단계를 건너뜁니다.

🔴 **반드시 라벨(`-l app=mas-inst1-ws1-manage`)로 범위를 좁혀서 지우세요.** `--all`로 지우면 `db2u-operator-manager`, `db2u-day2-ops-controller-manager` 오퍼레이터 Deployment까지 같이 삭제됩니다.

```bash
oc delete db2ucluster --all -n db2u --wait=false
sleep 10
```

finalizer 때문에 안 지워지면 직접 제거합니다.

```bash
for n in $(oc get db2ucluster -n db2u -o name 2>/dev/null); do
  oc patch $n -n db2u --type=merge -p '{"metadata":{"finalizers":null}}'
done
```

```bash
oc delete statefulset,deploy,job,cronjob -n db2u -l app=mas-inst1-ws1-manage --wait=false
oc delete pvc --all -n db2u --wait=false
oc delete cm mas-inst1-ws1-manage-enforce-config -n db2u --ignore-not-found
oc delete cm,secret -n db2u -l app=mas-inst1-ws1-manage --ignore-not-found
oc delete route --all -n db2u --ignore-not-found
```

🔴 **`mas-inst1-ws1-manage-enforce-config`는 라벨이 없어 위 라벨 삭제로는 지워지지 않으므로 이름으로 직접 지워야 합니다.** 이 절 맨 위의 🔴 경고(파이프라인이 스킵되며 멈춘 사고)가 바로 이 ConfigMap을 빠뜨렸을 때 일어났습니다.

**5) 확인 및 검증**

```bash
sleep 20
```

검증 : `db2u`에 오퍼레이터 2개(`db2u-operator-manager`, `db2u-day2-ops-controller-manager`)만 남고 PVC·ConfigMap이 없어야 합니다

```bash
oc get db2ucluster,pods,pvc,cm -n db2u
```

검증 : Manage 리소스가 남아 있지 않아야 합니다

```bash
oc get manageworkspace,manageapp,build,bc,is -n mas-inst1-manage
```

검증 : NFS에 Db2 디렉터리가 남아 있지 않아야 합니다

```bash
ls /export/mas-rwx/
```

정리가 끝나면 §3.10.3을 다시 실행합니다.

### 3.11 Manage + Maximo IT 확인 (웹 UI)

🔴 **별도의 활성화 작업은 필요 없습니다.** `mas install`이 Manage 배포와 Maximo IT(`icd`) 활성화, DB 스키마 생성(maxinst), 이미지 빌드까지 모두 수행합니다. Suite Administration에서 "Activate Manage"를 누를 일이 없습니다.

#### 3.11.1 확인 및 검증

검증 : 워크스페이스가 `Ready` 여야 합니다

```bash
oc get manageworkspace -A
```

검증 : Maximo IT(`icd`)가 컴포넌트에 들어 있어야 합니다

```bash
oc get manageworkspace inst1-ws1 -n mas-inst1-manage -o jsonpath='{.spec.components}{"\n"}'
```

```
{"base":{"version":"latest"},"icd":{"version":"latest"}}
```

검증 : 언어·데모 데이터·타임존이 의도대로 들어갔는지

```bash
oc get manageworkspace inst1-ws1 -n mas-inst1-manage -o jsonpath='{.spec.settings}' | jq
```

| 항목 | 기대값 |
|---|---|
| `languages.baseLang` / `secondaryLangs` | `EN` / `["KO"]` |
| `db.maxinst.demodata` | `true` |
| `db.maxinst.db2Vargraphic` | `true` (기본값) |
| `deployment.serverTimezone` | `Asia/Seoul` |

검증 : 서버 Pod이 `2/2 Running` 이어야 합니다

```bash
oc get pods -n mas-inst1-manage | grep -- '-all-'
```

검증 : 접속 주소는 **워크스페이스 ID가 앞에 붙습니다**

```bash
oc get route -n mas-inst1-manage
```

| Route | 용도 |
|---|---|
| **`ws1.manage.inst1.apps.…`** | **Maximo Manage / Maximo IT 메인** |
| `ws1-all.manage.inst1.apps.…` | `all` 서버 번들 직접 접근 |
| `maxinst.manage.inst1.apps.…` | 관리 도구 (ERD, toolsapi) |

⚠️ `manage.inst1.apps.…`(워크스페이스 ID 없는 주소)는 오퍼레이터가 먼저 만드는 Route이며 서비스되지 않습니다.

#### 3.11.2 웹 UI 확인

1. `https://ws1.manage.inst1.apps.<cluster>.<domain>` 접속 — `hosts` 등록과 인증서 수락이 필요합니다(§3.12.1)
2. §3.10.3에서 확인한 Superuser로 로그인
3. **Service Desk / Incident / Problem / Service Request** 메뉴가 보이면 Maximo IT 정상
4. 사용자 프로필에서 언어를 한국어로 전환해 확인
5. 자산·작업오더 목록에 데모 데이터가 있는지 확인

#### 3.11.3 설치 후 작업

| 항목 | 내용 |
|---|---|
| **정식 관리자 계정 생성** | Superuser는 초기 구성 전용입니다. Suite Administration에서 담당자 계정을 만들고 이후로는 그것을 사용하세요 |
| **AppPoints 할당** | ⬜ 라이선스에 Maximo IT용 AppPoints가 포함되어 있는지 확인 필요. MAS Entitlement와 Maximo IT Entitlement는 각각 필요합니다 |
| 사용자·그룹 권한 | Maximo IT 애플리케이션별 권한 설정 |

### 3.12 설치 후 확인 (사용자 계정)

#### 3.12.1 실행 — 접속용 `hosts` 등록 (검증용)

⚠️ **이 절은 구축 담당자가 검증할 때 쓰는 방법입니다.** 사내 사용자에게 이렇게 배포할 수는 없습니다 — 운영 전환 방안은 §3.12.3을 보세요.

폐쇄망에는 사내 DNS가 없고, Bastion의 dnsmasq는 클러스터 전용입니다. **노트북·PC에서 웹 UI에 접속하려면 `hosts` 파일에 이름을 직접 넣어야 합니다** — `*.apps` 와일드카드는 hosts 파일이 지원하지 않아 개별 이름이 필요합니다.

🔴 **호스트마다 인증서를 따로 수락해야 합니다.** 자체 서명이라 브라우저가 호스트별로 신뢰를 묻습니다. `admin.inst1…` 만 수락하면 Suite Administration이 **로딩 화면에서 멈춥니다** — `api.inst1…` 로 가는 XHR이 전부 차단되기 때문입니다(개발자 도구 Network에 `(failed)` 로 보임).

새 탭에서 아래 주소를 하나씩 열어 "고급 → 계속"으로 수락하세요. 흰 화면이나 404가 떠도 무방합니다.

```
https://api.inst1.apps.<cluster-name>.<base-domain>
https://auth.inst1.apps.<cluster-name>.<base-domain>
https://home.inst1.apps.<cluster-name>.<base-domain>
```

MAS 루트 CA를 PC 신뢰 저장소에 넣으면 이 과정이 필요 없습니다.

```bash
oc get secret inst1-cert-public-ca -n mas-inst1-core -o jsonpath='{.data.ca\.crt}' | base64 -d > mas-root-ca.crt
```

Bastion에서 필요한 줄을 통째로 생성합니다.

```bash
export KUBECONFIG=~/ocp-sno/auth/kubeconfig

{
  echo "192.168.2.210    registry.mas-it.itmsg.co.kr"
  echo "192.168.2.211    api.mas-it.itmsg.co.kr"
  oc get route -A -o jsonpath='{range .items[*]}192.168.2.211    {.spec.host}{"\n"}{end}'
} | sort -u
```

출력을 그대로 복사해 `hosts` 파일 끝에 붙여넣습니다.

| OS | 경로 |
|---|---|
| Windows | `C:\Windows\System32\drivers\etc\hosts` (관리자 권한) |
| macOS / Linux | `/etc/hosts` (root) |

주요 이름은 이렇습니다.

| 이름 | 용도 |
|---|---|
| `registry.<cluster>.<domain>` | Mirror Registry (Quay) — **Bastion IP** |
| `api.<cluster>.<domain>` | OCP API — 노트북에서 `oc` 사용 시 |
| `console-openshift-console.apps.…` | OCP 웹 콘솔 |
| `oauth-openshift.apps.…` | 콘솔 로그인 리다이렉트 — **빠지면 로그인 불가** |
| `admin.inst1.apps.…` | MAS Suite Administration |
| `home.inst1.apps.…` | MAS 홈 |
| `auth.inst1.apps.…` | MAS 인증 |
| `manage.inst1.apps.…` | Maximo Manage / Maximo IT |

⚠️ Route는 설치가 진행되면서 늘어납니다. Manage 서버가 올라온 뒤 위 명령을 **다시 실행해** 추가된 이름(`ws1.manage.…` 등)을 넣으세요.

#### 3.12.2 운영 전환 — DNS와 인증서

🔴 **`hosts` 등록과 인증서 개별 수락은 사내 사용자에게 적용할 수 없습니다.** 운영 전환 전에 아래 둘을 확정해야 합니다.

**DNS — 사내 DNS에 와일드카드 등록**

```
*.apps.<cluster-name>.<base-domain>   A   <sno-ip>
api.<cluster-name>.<base-domain>      A   <sno-ip>
registry.<cluster-name>.<base-domain> A   <bastion-ip>
```

와일드카드 한 줄이면 Route가 늘어나도 손댈 필요가 없습니다. 사내 DNS가 없다면 Bastion의 dnsmasq를 사내 DNS로 승격하거나, 상위 DNS에서 해당 존을 Bastion으로 위임해야 합니다.

**인증서**

| 방식 | 판단 |
|---|---|
| **사내 CA 발급 인증서** | 사내 PC는 이미 사내 CA를 신뢰하므로 경고가 없습니다. 가장 깔끔합니다 |
| MAS 자체 서명 CA를 전 PC에 배포 | GPO 등으로 루트 CA 배포. 사내 CA가 없을 때 |
| 공인 인증서 | 폐쇄망 도메인이라 발급이 어렵습니다 |

MAS는 설치 시 커스텀 도메인·인증서를 지정할 수 있습니다 — §3.10.2의 `Configure custom domain and certificate management?` 입니다. 이번 배포는 `n`으로 자체 서명을 택했고, 운영 전환 시에는 여기서 사내 인증서를 넣거나 설치 후 교체합니다.

⬜ **사이트 구축 전 확정 필요** — 사내 DNS 존재 여부, 사내 CA 존재 여부.

#### 3.12.3 확인 및 검증

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

#### 3.12.4 노드 종료·재시작 절차

VM 사양 변경, 점검, 이설 등으로 SNO를 내려야 할 때의 순서입니다.

🔴 **Db2를 먼저 정지해야 합니다.** 순서를 지키지 않으면 데이터베이스가 손상되고, 순환 로깅(`LOGARCHMETH1: OFF`) 구성이라 **복구가 불가능해 Manage DB를 재생성해야 합니다.**

**1) 실행 — Db2 정지**

```bash
oc rsh -n db2u c-mas-inst1-ws1-manage-db2u-0 bash -c \
  'su - db2inst1 -c "db2 force applications all; db2 deactivate db BLUDB; db2stop"'
```

검증 : 오류 없이 `DB2STOP processing was successful` 이 나와야 합니다

**2) 실행 — Db2 StatefulSet 정지**

오퍼레이터가 다시 띄우지 않도록 내려둡니다.

```bash
oc scale statefulset c-mas-inst1-ws1-manage-db2u -n db2u --replicas=0
oc get pods -n db2u
```

**3) 실행 — 노드 종료**

SNO는 워크로드를 옮길 다른 노드가 없어 `cordon`·`drain`은 의미가 없습니다.

```bash
ssh -i ~/.ssh/quay_installer core@<sno-ip> 'sudo shutdown -h now'
```

검증 : 응답이 끊겨야 종료 완료입니다

```bash
ping -c 3 <sno-ip>
```

🔴 **NFS를 RWX로 쓰면 종료가 지연됩니다.** SSH는 `Connection refused`인데 ping이 계속 응답하고 vSphere에서도 전원이 안 꺼지는 상태가 됩니다 — Bastion NFS 언마운트에서 걸린 것입니다.

**이때 강제 전원 끄기를 하면 안 됩니다.** SSH가 끊긴 것은 "모든 것이 정리됐다"는 뜻이 아닙니다. 1)~2)를 먼저 해두면 이 시점에 Db2가 이미 안전하게 내려가 있으므로 지연이 있어도 기다렸다가 정상 종료를 확인하면 됩니다.

**4) 실행 — 재시작 후 복구**

전원을 켜면 노드는 5~10분 안에 `Ready`가 되지만, MAS 전체가 정상화되는 데는 더 걸립니다.

```bash
oc get nodes
oc scale statefulset c-mas-inst1-ws1-manage-db2u -n db2u --replicas=1
```

검증 : Db2가 먼저 올라와야 Manage가 붙습니다

```bash
oc get db2ucluster -n db2u
oc get pods -n db2u | grep db2u-0
```

검증 : Manage 서버가 `2/2` 가 되어야 합니다 — Db2보다 먼저 뜨면 연결 실패로 재시작을 반복합니다

```bash
oc get pods -n mas-inst1-manage | grep -E '\-all\-'
oc get manageworkspace -A
```

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

### 4.5 `mas mirror-redhat-images --mode from-filesystem` 을 쓰지 않는 이유

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

### 4.6 `install-config.yaml` 검증 실패 — `imageDigestSources`

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

### 4.7 MCO가 `registries.conf`를 렌더링하지 못함 — `conflicting mirrorSourcePolicy`

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

