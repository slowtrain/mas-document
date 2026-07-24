# MAS 9.2 설치 가이드 — 온라인 환경

> **상태**: 검토 반영본  
> 인터넷 연결이 가능한 환경 기준입니다. 폐쇄망 환경은 [INSTALL_OFFLINE.md](INSTALL_OFFLINE.md)를 참고합니다.

---

## 목차

- [1. 기준 및 전제](#1-기준-및-전제)
- [2. 사전 준비](#2-사전-준비)
- [3. OCP SNO 설치](#3-ocp-sno-설치)
- [4. 스토리지 및 Registry 구성](#4-스토리지-및-registry-구성)
- [5. MAS 9.2 설치](#5-mas-92-설치)
- [6. IT 모듈 활성화](#6-it-모듈-활성화)
- [7. BIRT 구성](#7-birt-구성)
- [8. 설치 후 점검](#8-설치-후-점검)

---

## 1. 기준 및 전제

이 문서는 다음 버전 조합을 기준으로 작성합니다.

| 구성 요소 | 기준값 |
|-----------|--------|
| OpenShift | `4.16.x` |
| IBM Maximo Operator Catalog | `v9-250828-amd64` |
| MAS CLI | `quay.io/ibmmas/cli:15.2.0` |
| MAS / Manage Channel | `9.2.x` (앱별 개별 채널, 값은 설치 전 재확인) |

> `quay.io/ibmmas/cli:latest`는 사용하지 않습니다. catalog `v9-250828-amd64`를 사용할 때는 같은 시기의 MAS CLI 버전으로 고정해야 합니다.
>
> ⚠️ 미검증
>
> `quay.io/ibmmas/cli:15.2.0`은 `v9-250828-amd64` catalog와 같은 시기의 CLI로 정리한 값입니다. 실제 설치 전 IBM MAS CLI catalog 문서와 `mas install --help` 출력으로 조합을 재확인합니다.

설치 방식은 MAS CLI가 MAS Core, Manage, SLS, MongoDB, DB2 등 필요한 의존성을 설치/구성하는 흐름을 기본으로 합니다.

> ⚠️ 미검증
>
> DB2 또는 MongoDB를 사전에 수동으로 생성해서 MAS에 연결하는 방식은 별도 `JdbcCfg`, `MongoCfg` 구성 검증이 필요합니다. 이 문서의 기본 절차에서는 수동 DB2/MongoDB 생성 절차를 사용하지 않습니다.

---

## 2. 사전 준비

### 2.1 서버 구성

| 구분 | OS / 상태 | 역할 | 최소 사양 |
|------|-----------|------|-----------|
| Bastion | RHEL 9.x 권장 | ISO 생성, MAS CLI, oc CLI, DNS | 4 core / 8 GB RAM / 100 GB |
| SNO 노드 | OS 미설치 VM 또는 물리 서버 | OCP 컨트롤 플레인 + 워커 통합 | 16 core / 64 GB RAM / OS 300 GB + 데이터 500 GB |

Bastion은 설치 도구와 컨테이너 런타임을 안정적으로 사용하기 위해 RHEL 계열을 권장합니다. 현장 표준이 있으면 RHEL 8/9 또는 호환 배포판을 사용할 수 있지만, 패키지 설치 명령은 OS에 맞게 조정해야 합니다.

SNO 노드는 사전에 RHEL 같은 일반 OS를 설치하지 않습니다. Assisted Installer에서 생성한 Discovery ISO로 부팅하면 설치 과정에서 RHCOS(Red Hat CoreOS)와 OpenShift가 노드에 설치됩니다. VM으로 구성하는 경우에도 “빈 VM + ISO 부팅” 형태로 준비합니다.

SNO 노드는 디스크 2개 구성을 권장합니다. 첫 번째 디스크는 RHCOS/OpenShift용, 두 번째 디스크는 LVM Storage용으로 사용합니다.

### 2.2 필수 파일

| 파일 | 용도 |
|------|------|
| IBM Entitlement Key | IBM 이미지 pull 인증 |
| `entitlement.lic` | SLS AppPoints 라이선스 |
| Red Hat Pull Secret | OpenShift 설치 |
| SSH public key | Assisted Installer 노드 접근 |

### 2.3 DNS 구성

예시 값은 다음과 같습니다.

| 항목 | 예시 |
|------|------|
| Cluster name | `<cluster-name>` |
| Base domain | `<base-domain>` |
| SNO IP | `<sno-ip>` |
| Bastion IP | `<bastion-ip>` |

`/etc/dnsmasq.d/mas.conf` 예시:

```ini
server=<upstream-dns-ip>

address=/api.<cluster-name>.<base-domain>/<sno-ip>
address=/api-int.<cluster-name>.<base-domain>/<sno-ip>
address=/.apps.<cluster-name>.<base-domain>/<sno-ip>
```

서비스 활성화:

```bash
dnf install -y dnsmasq
systemctl enable --now dnsmasq
firewall-cmd --permanent --add-service=dns
firewall-cmd --reload
```

DNS 확인:

```bash
nslookup api.<cluster-name>.<base-domain>
nslookup api-int.<cluster-name>.<base-domain>
nslookup test.apps.<cluster-name>.<base-domain>
```

3개 모두 `<sno-ip>`로 응답해야 합니다.

### 2.4 Bastion CLI 준비

```bash
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.16/openshift-client-linux.tar.gz
tar -xvf openshift-client-linux.tar.gz
mv oc kubectl /usr/local/bin/

dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf install -y docker-ce docker-ce-cli
systemctl enable --now docker

ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

MAS CLI 버전 확인:

```bash
docker run -ti --rm \
  -v ~:/mnt/home \
  quay.io/ibmmas/cli:15.2.0 mas install --help
```

---

## 3. OCP SNO 설치

### 3.1 Assisted Installer에서 클러스터 생성

1. Red Hat Hybrid Cloud Console 접속
2. **Create cluster → Datacenter → Assisted Installer**
3. 아래 값을 입력합니다.

| 항목 | 값 |
|------|-----|
| Cluster name | `<cluster-name>` |
| Base domain | `<base-domain>` |
| OpenShift version | `4.16.x` |
| Single Node OpenShift | 선택 |
| Network | Static IP 권장 |
| SSH public key | `~/.ssh/id_rsa.pub` 내용 |

### 3.2 Discovery ISO 부팅 및 설치

1. Discovery ISO 다운로드
2. SNO 노드에 ISO 마운트 후 부팅
3. Assisted Installer 콘솔에서 노드 감지 확인
4. 설치 조건이 충족되면 **Install cluster** 실행

설치 완료 후 kubeconfig를 Bastion에 저장합니다.

```bash
mkdir -p ~/.kube
cp <kubeconfig-path> ~/.kube/config

oc get nodes
oc get clusterversion
oc get co
```

모든 Cluster Operator가 `AVAILABLE=True`, `PROGRESSING=False`, `DEGRADED=False` 상태여야 합니다.

---

## 4. 스토리지 및 Registry 구성

### 4.1 LVM Operator 설치

LVM Storage는 RWO 용도로만 사용합니다.

```bash
oc new-project openshift-storage

oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: lvms-operator
  namespace: openshift-storage
spec:
  installPlanApproval: Automatic
  name: lvms-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

디스크 확인:

```bash
oc debug node/<node-name> -- chroot /host lsblk
```

LVMCluster 생성:

```bash
oc apply -f - <<EOF
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: vg1
        deviceSelector:
          paths:
            - /dev/<data-disk>
        thinPoolConfig:
          name: thin-pool-1
          sizePercent: 90
          overprovisionRatio: 10
EOF
```

StorageClass 확인:

```bash
oc get storageclass
```

> LVM Storage operator가 deviceClass `vg1`로 생성하는 StorageClass 이름은 `lvms-vg1`입니다(예: `odf-lvm-vg1`이 아님. `lvms-<deviceClass 이름>` 규칙). `oc get storageclass`로 실제 이름을 확인한 뒤 `Storage Class (RWO)` 값에 사용합니다. 이 StorageClass는 `ReadWriteOnce` 용도로만 사용합니다. `Storage Class (RWX)`에는 별도의 RWX 지원 StorageClass를 입력해야 합니다.
>
> volumeBindingMode는 기본적으로 `WaitForFirstConsumer`입니다. SNO는 노드가 1개뿐이라 파드 배치 최적화 목적은 의미가 없지만, PVC가 이를 소비하는 파드가 생성되기 전까지 `Pending` 상태로 보이는 것은 정상 동작입니다. 아래 명령으로 확인합니다.
>
> ```bash
> oc get storageclass lvms-vg1 -o yaml | grep volumeBindingMode
> ```
>
> ⚠️ 미검증
>
> IBM MAS SNO 참고 문서(`ibm-mas-manage.github.io/sno`)에 따르면 SNO에서는 모든 파드가 단일 노드에서 실행되므로 RWO StorageClass만으로 충분하며 별도의 RWX StorageClass가 필요하지 않을 수 있습니다. 실제 `mas install` 프롬프트에서 RWX가 어떤 컴포넌트에 요구되는지 확인한 뒤, 불필요하면 RWX StorageClass 준비를 생략할 수 있습니다.

### 4.2 RWX Storage 준비

MAS/Manage 구성에 RWX PVC가 필요한 경우 별도 RWX StorageClass를 준비합니다.

| 구분 | 예시 |
|------|------|
| RWO StorageClass | `<rwo-storage-class>` |
| RWX StorageClass | `<rwx-storage-class>` |

> ⚠️ 미검증
>
> 현재 실습 환경에 RWX StorageClass가 없다면 MAS 설치 프롬프트에서 어떤 항목이 RWX를 요구하는지 실제 `mas install` 프롬프트와 설치 로그로 확인해야 합니다. LVM StorageClass를 RWX 값으로 대체하지 않습니다.

### 4.3 Image Registry 구성

SNO 개발/검증 환경에서는 OpenShift Image Registry를 `emptyDir`로 구성할 수 있습니다.

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"managementState":"Managed","storage":{"emptyDir":{}}}}'
```

> `emptyDir` registry storage는 운영용 권장 구성이 아닙니다. 노드 재시작/장애 시 registry 데이터가 보존되지 않을 수 있습니다.

---

## 5. MAS 9.2 설치

### 5.1 IBM Maximo Operator Catalog 등록

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: IBM Maximo Operators (v9-250828-amd64)
  image: icr.io/cpopen/ibm-maximo-operator-catalog:v9-250828-amd64
  publisher: IBM
  sourceType: grpc
  priority: 90
  updateStrategy:
    registryPoll:
      interval: 45m
EOF
```

> CatalogSource 이름은 `ibm-operator-catalog`로 고정합니다. IBM 공식 카탈로그 페이지(`v9-250828-amd64`)에 게시된 실제 이름과 일치해야 하며, 임의의 이름(예: `ibm-maximo-operator-catalog`)을 사용하면 이후 `mas install`의 Catalog Source 프롬프트 값과 실제 클러스터의 CatalogSource가 어긋나 설치가 실패할 수 있습니다.
>
> ⚠️ 미검증
>
> `mas install`은 내부적으로 `--mas-catalog-version` 값을 기준으로 IBM Maximo Operator Catalog 설치를 자동 수행하는 것으로 보입니다(공식 CLI 문서의 Ansible 자동화 목록 기준). 즉 이 수동 `oc apply` 단계가 아예 불필요하거나, CLI가 자체적으로 만드는 CatalogSource와 이름이 겹칠 수 있습니다. 실제 설치 전 `mas install --help`로 카탈로그 자동 관리 여부를 확인하고, 자동 생성된다면 이 수동 단계는 생략합니다.

READY 확인:

```bash
oc get catalogsource -n openshift-marketplace ibm-operator-catalog
oc get packagemanifest -n openshift-marketplace | grep -i maximo
```

### 5.2 MAS CLI 실행

```bash
docker run -ti --rm \
  -v ~:/mnt/home \
  -v ~/.kube:/root/.kube \
  quay.io/ibmmas/cli:15.2.0 mas install
```

> ⚠️ 미검증
>
> 공식 CLI 문서에서 확인된 흐름은 컨테이너 내부의 대화형 프롬프트에서 OpenShift 서버 URL/토큰을 직접 입력하거나, 컨테이너 안에서 `oc login`을 먼저 실행하는 방식입니다. `~/.kube:/root/.kube` 마운트로 kubeconfig를 그대로 전달하는 방식은 공식 문서에서 확인되지 않았습니다. 우연히 동작할 수 있으나, 실패 시 아래 대안을 사용합니다.
>
> ```bash
> docker run -ti --rm \
>   -v ~:/mnt/home \
>   quay.io/ibmmas/cli:15.2.0 bash -c "oc login --token=<token> --server=<api-url> && mas install"
> ```

대화형 프롬프트 기준 입력값:

| 항목 | 값 |
|------|-----|
| IBM Entitlement Key | `<ibm-entitlement-key>` |
| MAS License File | `/mnt/home/<entitlement-file>` |
| Catalog Source | `ibm-operator-catalog` |
| Subscription Channel (Core/Manage 등 앱별로 개별 입력) | `9.2.x` |
| MAS Instance ID | `<mas-instance-id>` |
| Workspace ID | `<workspace-id>` |
| Storage Class (RWO) | `<rwo-storage-class>` |
| Storage Class (RWX) | `<rwx-storage-class>` |

> 프롬프트 문구는 MAS CLI 버전에 따라 다를 수 있습니다. 이 문서는 `quay.io/ibmmas/cli:15.2.0` 기준으로 catalog를 고정합니다.
>
> ⚠️ 미검증
>
> "Subscription Channel"은 단일 값이 아니라 `--mas-channel`, `--manage-channel` 등 애플리케이션별 채널로 나뉘어 있을 가능성이 높습니다. 또한 catalog `v9-250828-amd64` 기준 채널 값은 `9.1.x`, `9.1.x-feature`처럼 "x-suffix" 형식이므로, `9.2`도 실제로는 `9.2.x` 형태일 수 있습니다. 설치 전 `oc get packagemanifest -n openshift-marketplace <package-name> -o yaml`로 실제 지원 채널 목록을 확인합니다.

### 5.3 설치 상태 확인

```bash
oc get suite -A
oc get subscriptions -A
oc get installplan -A
oc get pods -A | grep -v Running | grep -v Completed | grep -v Succeeded
```

MAS 네임스페이스 예시:

```bash
oc get pods -n mas-<mas-instance-id>-core
oc get events -n mas-<mas-instance-id>-core --sort-by=.lastTimestamp
```

> ⚠️ 미검증
>
> Operator pod label은 catalog/버전에 따라 달라질 수 있습니다. 로그 확인 시 `oc get pods -n mas-<mas-instance-id>-core`로 실제 pod 이름을 먼저 확인한 뒤 `oc logs`를 실행합니다.

---

## 6. IT 모듈 활성화

### 6.1 Suite Administration에서 활성화

MAS Admin URL 예시:

```text
https://admin.<mas-instance-id>.apps.<cluster-name>.<base-domain>
```

진행 순서:

1. Suite Administration 로그인
2. Catalog 또는 Applications 메뉴에서 Manage 확인
3. IT 관련 Industry Solution 또는 Add-on 활성화
4. Workspace `<workspace-id>`에 적용

> ⚠️ 미검증
>
> 메뉴 명칭은 MAS/Manage fix pack에 따라 달라질 수 있습니다. 실제 UI에서 `Manage`, `IT`, `Industry Solution`, `Workspace` 항목을 기준으로 확인합니다.

### 6.2 DB 초기화 스크립트

`updatedb.sh` / `runscriptfile.sh -cIT -f SETUPIT`는 **온프레미스 Maximo Asset Management 7.x**에서 DBC(database configuration) 스크립트를 수동 적용할 때 쓰던 레거시 도구입니다. MAS 오퍼레이터 기반 배포(9.2)에는 해당하지 않으며, IBM 공식 문서는 Industry Solution/Add-on 활성화가 Suite Administration 배포 마법사에서 애플리케이션 버전 선택 → DB 연결 → JDBC 구성 → Industry Solution 선택 절차로 진행되고, 이후 DB 스키마 반영은 오퍼레이터가 자동으로 수행한다고 설명합니다.
따라서 이 스크립트를 수동으로 실행할 필요는 없습니다. Manage pod에 직접 접속해 스크립트를 실행하는 절차는 사용하지 않습니다.

---

## 7. BIRT 구성

MAS Manage의 보고서 기능은 Manage 배포 상태와 라이선스/애플리케이션 구성에 따라 달라집니다. 기본 설치 후 먼저 Manage UI에서 보고서 메뉴가 활성화되는지 확인합니다.

확인 항목:

- Manage 애플리케이션이 정상 기동했는지 확인
- 보고서 메뉴 접근 가능 여부 확인
- 샘플 보고서 실행 가능 여부 확인

`mxe.report.birt.viewerurl` 시스템 속성은 기본값이 비어 있으며, 클러스터형 환경에서 리포트 처리를 별도의 BIRT Report-Only Server(BROS)로 오프로드할 때만 설정하는 값입니다. 기본 리포트 뷰잉에는 필요하지 않습니다. `http://localhost:9080/...`처럼 로컬 주소를 직접 넣는 방식은 OpenShift Route 환경에서는 애초에 해석되지 않으므로 사용하지 않습니다.

BIRT를 위한 별도 서버 번들이 필요한 경우, IBM 공식 절차는 `manageapp`을 직접 patch하는 것이 아니라 `ManageWorkspace` CR의 `serverBundles` 필드에 report 타입 번들을 추가하는 방식입니다(Ansible role `suite_manage_birt_report_config` 기준). 예:

```bash
oc get manageworkspace -n mas-<mas-instance-id>-manage
oc edit manageworkspace <manageworkspace-name> -n mas-<mas-instance-id>-manage
```

> ⚠️ 미검증
>
> `serverBundles` 필드의 정확한 스키마는 설치된 Manage 버전의 CRD(`oc explain manageworkspace.spec.serverBundles`)로 확인한 뒤 적용합니다.

---

## 8. 설치 후 점검

```bash
oc get nodes
oc get co
oc get suite -A
oc get manageapp -A
oc get pods -A | grep -v Running | grep -v Completed | grep -v Succeeded
```

브라우저 확인:

| 항목 | URL 예시 |
|------|----------|
| OpenShift Console | `https://console-openshift-console.apps.<cluster-name>.<base-domain>` |
| MAS Admin | `https://admin.<mas-instance-id>.apps.<cluster-name>.<base-domain>` |
| MAS Manage | 실제 Route를 `oc get route -A`로 확인 |

Route 확인:

```bash
oc get route -A | grep -E 'mas|manage'
```
