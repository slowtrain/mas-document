# 접속 정보

> 계정·비밀번호가 포함되어 있습니다. 공개 저장소로 옮기거나 공유할 때는 먼저 제거하세요.

폐쇄망에 사내 DNS가 없어 **PC마다 `hosts` 등록과 인증서 수락이 필요합니다.** 서버 사양·구성은 [SERVER_INFO.md](SERVER_INFO.md)를 보세요.

---

## 1. 접속 주소

| 주소 | 용도 | 계정 | 비밀번호 |
|---|---|---|---|
| `https://admin.inst1.apps.mas-it.itmsg.co.kr` | **MAS Suite Administration** | `O1Soo6ql8f0fXzaLKGhzSMa400EDFNvQ` | `qIDqNdwuttnWw94dF6B9SUdJ6BxNKClA` |
| `https://home.inst1.apps.mas-it.itmsg.co.kr` | MAS 홈 | 〃 | 〃 |
| `https://ws1.manage.inst1.apps.mas-it.itmsg.co.kr` | **Maximo Manage / Maximo IT** | 〃 | 〃 |
| `https://console-openshift-console.apps.mas-it.itmsg.co.kr` | OpenShift 웹 콘솔 | `kubeadmin` | `oRYob-UVjUs-AI492-mjDsc` |
| `https://api.mas-it.itmsg.co.kr:6443` | OpenShift API (`oc login`) | `kubeadmin` | 〃 |
| `https://registry.mas-it.itmsg.co.kr:8443` | Mirror Registry (Quay) | `init` | `T3AgRHSeYw6x5u0Z2q4OE9s81Koyd7tV` |
| `ssh maximo@192.168.2.210` | Bastion | `maximo` | `Itmsg4u!` |
| `ssh -i ~/.ssh/quay_installer core@192.168.2.211` | SNO 노드 | `core` | (키 인증) |
| `192.168.2.211:31899` | **Db2** — `BLUDB` / 스키마 `maximo` (§5) | `db2inst1` | `X5D4gNDiRlMsPUu` |

접속 전 [`hosts` 등록](#2-hosts-등록)이 필요하며, 자체 서명 인증서라 **호스트마다 브라우저 경고를 한 번씩 수락**해야 합니다.

⚠️ **Superuser는 CLI가 랜덤 생성한 값입니다.** 설치 시 입력한 `masadmin`은 적용되지 않았습니다. 실제 값은 시크릿에서 확인합니다.

```bash
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.username}' | base64 -d; echo
oc get secret inst1-credentials-superuser -n mas-inst1-core -o jsonpath='{.data.password}' | base64 -d; echo
```

⬜ Superuser는 초기 구성 전용입니다. Suite Administration에서 정식 관리자 계정을 만들고 그것을 사용하세요.


---

## 2. `hosts` 등록

폐쇄망에 사내 DNS가 없고 Bastion의 dnsmasq는 클러스터 전용이라, 웹 UI 접속용 이름을 직접 넣어야 합니다. `*.apps` 와일드카드는 `hosts` 파일이 지원하지 않아 개별 이름이 필요합니다.

| OS | 경로 |
|---|---|
| Windows | `C:\Windows\System32\drivers\etc\hosts` (관리자 권한) |
| macOS / Linux | `/etc/hosts` (root) |

묶음별로 나누고 각 줄에 용도를 주석으로 붙였습니다. 안 쓰는 줄은 `#`으로 막아두고 필요할 때 풀면 됩니다.

```
# ════════ MAS 사용자 접속 (필수) ════════
# Suite Administration — 관리 콘솔
192.168.2.211    admin.inst1.apps.mas-it.itmsg.co.kr
# MAS API — 빠지면 화면이 로딩에서 멈춤
192.168.2.211    api.inst1.apps.mas-it.itmsg.co.kr
# 로그인 리다이렉트 — 빠지면 로그인 불가
192.168.2.211    auth.inst1.apps.mas-it.itmsg.co.kr
# MAS 홈 (애플리케이션 목록)
192.168.2.211    home.inst1.apps.mas-it.itmsg.co.kr
192.168.2.211    ws1.home.inst1.apps.mas-it.itmsg.co.kr
# Maximo Manage / Maximo IT — 실제 업무 화면
192.168.2.211    ws1.manage.inst1.apps.mas-it.itmsg.co.kr

# ════════ OpenShift (관리자용) ════════
# OCP API — 노트북에서 oc 명령 사용 시
192.168.2.211    api.mas-it.itmsg.co.kr
# OCP 웹 콘솔
192.168.2.211    console-openshift-console.apps.mas-it.itmsg.co.kr
# 콘솔 로그인 리다이렉트 — 빠지면 콘솔 로그인 불가
192.168.2.211    oauth-openshift.apps.mas-it.itmsg.co.kr
# oc / kubectl 다운로드 페이지
192.168.2.211    downloads-openshift-console.apps.mas-it.itmsg.co.kr

# ════════ Mirror Registry (Quay UI 볼 때만) ════════
# Bastion IP (.210) — 다른 줄과 IP가 다름에 주의
#192.168.2.210    registry.mas-it.itmsg.co.kr

# ════════ MAS 관리·진단 (문제 생겼을 때) ════════
# all 서버 번들 직접 접근 — 앞단을 건너뛰고 서버 상태 확인
192.168.2.211    ws1-all.manage.inst1.apps.mas-it.itmsg.co.kr
# 관리 도구 — /erd (DB 관계도), /toolsapi
192.168.2.211    maxinst.manage.inst1.apps.mas-it.itmsg.co.kr
# Db2 직접 접속 — DBeaver 등 SQL 툴
192.168.2.211    mas-inst1-ws1-manage-db2u.apps.mas-it.itmsg.co.kr
# MCP 프레임워크 (AI 연동)
192.168.2.211    ws1-mcp.manage.inst1.apps.mas-it.itmsg.co.kr
# Slack 알림 프록시 — 미사용
192.168.2.211    ws1.slackproxy.manage.inst1.apps.mas-it.itmsg.co.kr

# ════════ 모니터링·부가 (필요할 때 주석 해제) ════════
# Grafana 대시보드
#192.168.2.211    mas-grafana-route-grafana5.apps.mas-it.itmsg.co.kr
# Prometheus / Alertmanager / Thanos — 클러스터 메트릭
#192.168.2.211    alertmanager-main-openshift-monitoring.apps.mas-it.itmsg.co.kr
#192.168.2.211    prometheus-k8s-openshift-monitoring.apps.mas-it.itmsg.co.kr
#192.168.2.211    prometheus-k8s-federate-openshift-monitoring.apps.mas-it.itmsg.co.kr
#192.168.2.211    thanos-querier-openshift-monitoring.apps.mas-it.itmsg.co.kr
#192.168.2.211    federate-openshift-user-workload-monitoring.apps.mas-it.itmsg.co.kr
#192.168.2.211    thanos-ruler-openshift-user-workload-monitoring.apps.mas-it.itmsg.co.kr
# IBM DRO — 라이선스 사용량 보고
#192.168.2.211    ibm-data-reporter-redhat-marketplace.apps.mas-it.itmsg.co.kr
#192.168.2.211    rhm-data-service-redhat-marketplace.apps.mas-it.itmsg.co.kr
# Tekton 파이프라인 콘솔
#192.168.2.211    pipelines-as-code-controller-openshift-pipelines.apps.mas-it.itmsg.co.kr
#192.168.2.211    tekton-results-api-openshift-pipelines.apps.mas-it.itmsg.co.kr
#192.168.2.211    tkn-cli-serve-openshift-pipelines.apps.mas-it.itmsg.co.kr
```

⚠️ Route는 설치가 진행되면서 늘어납니다. Manage 활성화 후 다시 뽑아 추가하세요.

```bash
oc get route -A -o jsonpath='{range .items[*]}192.168.2.211    {.spec.host}{"\n"}{end}' | awk 'NF==2' | sort -u
```


---

## 3. 인증서

MAS는 자체 서명 CA로 인증서를 발급하므로 브라우저가 경고를 띄웁니다. 방법이 두 가지이고 **A가 기본, B는 임시**입니다.

### A) 루트 CA를 PC에 설치 (권장)

Bastion에서 추출합니다.

```bash
oc get secret inst1-cert-public -n mas-inst1-core \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > mas-root-ca.crt
```

검증 : **self-signed 루트**여야 합니다 — `subject`와 `issuer`가 같아야 합니다

```bash
openssl crl2pkcs7 -nocrl -certfile mas-root-ca.crt | openssl pkcs7 -print_certs -noout
```

```
subject=C=GB, L=London, ... CN=public.inst1.mas.ibm.com
issuer=C=GB, L=London, ... CN=public.inst1.mas.ibm.com
```

파일을 PC로 옮겨 설치합니다.

| OS | 방법 |
|---|---|
| Windows | 파일 더블클릭 → **인증서 설치** → **로컬 컴퓨터** → **신뢰할 수 있는 루트 인증 기관** |
| Windows (일괄) | GPO → 컴퓨터 구성 → 정책 → Windows 설정 → 보안 설정 → 공개 키 정책 → 신뢰할 수 있는 루트 인증 기관 |
| macOS | 키체인 접근 → **시스템** → 드래그 후 "항상 신뢰" |

🔴 **저장 위치를 틀리면 효과가 없습니다.** "현재 사용자"나 "개인"이 아니라 **로컬 컴퓨터 / 신뢰할 수 있는 루트 인증 기관**이어야 합니다. Chrome·Edge는 Windows 저장소를 쓰므로 설치 후 브라우저를 **완전히 종료**했다 다시 여세요.

### B) 브라우저에서 경고 수락 (임시)

🔴 **호스트마다 따로 수락해야 합니다.** `admin.inst1…` 만 수락하면 Suite Administration이 **로딩 화면에서 멈춥니다** — `api.inst1…` 로 가는 XHR이 전부 차단되기 때문입니다.

새 탭에서 아래 주소를 하나씩 열어 "고급 → 계속"을 누르세요. 흰 화면이나 404가 떠도 무방합니다.

```
https://api.inst1.apps.mas-it.itmsg.co.kr
https://auth.inst1.apps.mas-it.itmsg.co.kr
https://home.inst1.apps.mas-it.itmsg.co.kr
https://ws1.manage.inst1.apps.mas-it.itmsg.co.kr
```

### CA를 설치했는데도 경고가 남을 때

인증서의 SAN(Subject Alternative Name)에 접속 주소가 없는 경우입니다.

```bash
openssl s_client -connect 192.168.2.211:443 \
  -servername ws1.manage.inst1.apps.mas-it.itmsg.co.kr </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -A3 'Subject Alternative Name'
```

⚠️ 와일드카드는 **한 레이블만** 덮습니다. `*.inst1.apps.…` 는 `admin.inst1.apps.…` 에는 맞지만 `ws1.manage.inst1.apps.…` 처럼 레이블이 더 깊으면 맞지 않습니다.

---

## 4. SSH 키

| 항목 | 값 |
|---|---|
| `~/.ssh/quay_installer` | `mirror-registry install`이 요구하는 키. 2026-08-05 생성, `authorized_keys`에 등록됨 |
| SNO 노드 접속 | `ssh -i ~/.ssh/quay_installer core@192.168.2.211` — `install-config.yaml`의 `sshKey`로 같은 키를 사용 |

---

## 5. 데이터베이스 (Db2) 직접 접속

DBeaver 같은 SQL 툴로 Maximo DB를 직접 볼 때 씁니다.

| 항목 | 값 |
|---|---|
| 데이터베이스 | `BLUDB` |
| 스키마 | `maximo` |
| 사용자 | `db2inst1` |
| 비밀번호 | `X5D4gNDiRlMsPUu` |
| SSL | 활성화 (`TLSv1.2`) |

접속 경로가 셋입니다.

| 경로 | 주소 | 비고 |
|---|---|---|
| **NodePort (평문)** | `192.168.2.211:31899` | 가장 간단. `hosts` 등록 불필요 |
| NodePort (SSL) | `192.168.2.211:32703` | Db2 CA 인증서 필요 |
| Route (SSL) | `mas-inst1-ws1-manage-db2u.apps.mas-it.itmsg.co.kr:443` | `hosts` 등록 + CA 인증서 필요 |
| 클러스터 내부 | `c-mas-inst1-ws1-manage-db2u-engn-svc.db2u.svc:50001` | Manage가 쓰는 경로 |

Manage가 실제로 사용하는 JDBC URL입니다.

```
jdbc:db2://c-mas-inst1-ws1-manage-db2u-engn-svc.db2u.svc:50001/BLUDB:sslConnection=true;sslVersion=TLSv1.2;
```

**SSL로 붙을 때**는 Db2 전용 CA가 필요합니다. MAS 루트 CA와 다릅니다(`CN=ca.db2u`, 2046년까지 유효).

```bash
oc get secret inst1-ws1-jdbccfg-workspace-application-binding -n mas-inst1-manage \
  -o jsonpath='{.data.certificates_0}' | base64 -d | \
  sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' > db2-ca.crt
```

⚠️ **접속 정보를 다시 뽑을 때**입니다.

```bash
# 계정·URL·인증서
oc get secret inst1-ws1-jdbccfg-workspace-application-binding -n mas-inst1-manage \
  -o jsonpath='{.data}' | jq -r 'to_entries[] | "\(.key): \(.value|@base64d)"'

# NodePort 확인
oc get svc c-mas-inst1-ws1-manage-db2u-engn-svc -n db2u
```

🔴 **DB 직접 접속은 조회 용도로만 쓰세요.** Maximo는 애플리케이션 계층에서 무결성을 관리하므로 SQL로 데이터를 고치면 정합성이 깨집니다.

---

## 6. 운영 전환 — 사내 DNS

🔴 **`hosts` 등록은 구축 담당자 검증용입니다.** 사내 사용자 전체에게 배포할 수 없으므로 운영 전환 전에 사내 DNS에 등록해야 합니다.

```
*.apps.mas-it.itmsg.co.kr   A   192.168.2.211
api.mas-it.itmsg.co.kr      A   192.168.2.211
registry.mas-it.itmsg.co.kr A   192.168.2.210
```

와일드카드 한 줄이면 Route가 늘어나도 손댈 필요가 없습니다. 사내 DNS가 없다면 Bastion의 dnsmasq를 사내 DNS로 승격하거나, 상위 DNS에서 해당 존을 Bastion으로 위임해야 합니다.

⬜ **확인 필요** — 사내 DNS 존재 여부, `*.apps` 와일드카드 지원 여부, 사내 CA 존재 여부.
