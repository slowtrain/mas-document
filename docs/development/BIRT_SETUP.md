# Maximo Manage 9.2 BIRT 보고서 개발환경 구성

BIRT Report Designer를 개발 PC에 세우고, Maximo 제품 보고서를 열어 Preview까지 확인한 뒤 Manage에 등록하는 절차입니다. Java 커스터마이징 환경은 [DEVELOPMENT_SETUP.md](DEVELOPMENT_SETUP.md)를 보세요.

Maximo Manage 9.2는 **BIRT 4.21** 을 사용합니다.

<details>
<summary><b>목차</b></summary>

- [1. BIRT Report Designer 4.21 설치](#1-birt-report-designer-421-설치)
- [2. Maximo BIRT 개발 리소스 확인](#2-maximo-birt-개발-리소스-확인)
- [3. 경로 변수 준비](#3-경로-변수-준비)
- [4. scriptlibrary 클래스 복사](#4-scriptlibrary-클래스-복사)
- [5. JDBC Driver 복사](#5-jdbc-driver-복사)
- [6. mxreportdatasources.properties 설정](#6-mxreportdatasourcesproperties-설정)
- [7. OSGi 프래그먼트 생성 및 배치](#7-osgi-프래그먼트-생성-및-배치)
- [8. Project Classpath 설정](#8-project-classpath-설정)
- [9. Report Library · Template 경로 설정](#9-report-library--template-경로-설정)
- [10. 기존 보고서 Preview 테스트](#10-기존-보고서-preview-테스트)
- [11. Manage에 보고서 등록](#11-manage에-보고서-등록)

</details>

전체 흐름입니다.

```text
BIRT 설치(JDK 25 지정) → scriptlibrary 클래스 적용 → JDBC Driver 적용 → DB 접속정보 설정
   → OSGi 프래그먼트 생성 → birt.exe -clean → Classpath·Library·Template 설정
   → 기존 보고서 Preview 검증 → 신규/수정 보고서 개발 → Manage Report Administration으로 등록
```

🔴 **§7이 이 문서의 핵심입니다.** IBM 문서의 절차(클래스 복사 + Classpath 등록)만으로는 BIRT 4.21에서 동작하지 않습니다. 이유는 §7에 적었습니다.

---

## 1. BIRT Report Designer 4.21 설치

[Eclipse BIRT 4.21 다운로드](https://download.eclipse.org/justj/?file=birt/updates/release/4.21.0/downloads)에서 **Windows x86_64용 BIRT Report Designer All-in-One ZIP** 을 받아 압축을 풉니다.

이 문서는 아래 경로 기준입니다.

```text
C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse
```

실행 파일은 `eclipse.exe` 가 아니라 **`birt.exe`** 이고, 설정은 **`birt.ini`** 를 읽습니다.

### JDK 25 지정 (필수)

🔴 All-in-One에는 **JustJ OpenJDK 21** 이 번들로 들어 있습니다. Maximo 제품 클래스는 **class file major 69 = Java 25** 로 컴파일돼 있어 Java 21 JVM은 로드하지 못합니다.

`birt.ini` 를 열어 `-vm` 두 줄을 추가합니다. **`-vmargs` 보다 반드시 위**여야 합니다.

```ini
-product
org.eclipse.birt.branding.birt_all_in_one
-vm
C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin\javaw.exe
-vmargs
-XX:CompileCommand=quiet
```

JDK 25는 Java 커스터마이징에도 필요하므로 이미 설치돼 있습니다([DEVELOPMENT_SETUP.md](DEVELOPMENT_SETUP.md) 서두).

## 2. Maximo BIRT 개발 리소스 확인

Admin 이미지에서 추출한 SMP 트리에 이미 들어 있습니다(추출 절차는 [DEVELOPMENT_SETUP.md §1.4](DEVELOPMENT_SETUP.md)). **Docker에서 다시 꺼낼 필요가 없습니다.**

```text
C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\reports\birt
├─ libraries        MaximoSystemLibrary.rptlibrary + 앱별 .properties
├─ reports          제품 보고서 .rptdesign (93개 폴더, 236개)
├─ scriptlibrary    classes/ — Maximo 전용 Script Library 클래스
├─ templates        maximo*.rpttemplate
└─ tools
```

JDBC Driver는 보고서 리소스가 아니라 Maximo `lib` 에 있습니다.

```text
C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\applications\maximo\lib
├─ db2jcc.jar
└─ db2jcc_license_cu.jar
```

## 3. 경로 변수 준비

경로가 길고, **BIRT Viewer 플러그인 폴더명은 버전 문자열이 붙어 설치마다 다릅니다.** 변수로 한 번만 잡아두면 나머지가 단순해집니다.

```powershell
$JDK    = 'C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin'
$SMP    = 'C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo'
$ECL    = 'C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse'
$VIEWER = (Get-ChildItem "$ECL\plugins" -Directory -Filter 'org.eclipse.birt.report.viewer_*').FullName

$VIEWER
```

`org.eclipse.birt.report.viewer_4.21.…` 하나가 출력돼야 합니다. 여러 개가 나오면 BIRT가 중복 설치된 것이니 정리하세요.

## 4. scriptlibrary 클래스 복사

`classes` **안의 내용**이 `WEB-INF\classes` 아래로 들어가야 합니다.

```powershell
robocopy "$SMP\reports\birt\scriptlibrary\classes" "$VIEWER\birt\WEB-INF\classes" /E /NFL /NDL /NJH /NJS

Get-ChildItem "$VIEWER\birt\WEB-INF\classes"
Test-Path "$VIEWER\birt\WEB-INF\classes\com\ibm\tivoli\maximo\report\script\MXReportScriptContext.class"
```

🔴 **`Copy-Item` 으로 하지 마세요.** `Copy-Item "...\classes\*" "...\classes\" -Recurse` 는 `com` 디렉터리를 건너뛰고 그 **안의 내용**(`ibm\...`)을 복사합니다. 실제로 그렇게 됩니다.

| | 결과 |
|---|---|
| 올바름 | `WEB-INF\classes\com\ibm\tivoli\...` |
| `Copy-Item` 와일드카드 | `WEB-INF\classes\ibm\tivoli\...` ← 패키지 한 단계 누락 |
| `classes` 폴더째 복사 | `WEB-INF\classes\classes\com\...` |

`Test-Path` 가 `True` 여야 하고, 목록에 `com` 이 보여야 합니다. `ibm` 이 보이면 잘못된 것입니다.

> `robocopy` 는 복사에 성공해도 종료 코드 1을 반환합니다. 오류가 아닙니다.

## 5. JDBC Driver 복사

```powershell
Copy-Item "$SMP\applications\maximo\lib\db2jcc.jar",`
          "$SMP\applications\maximo\lib\db2jcc_license_cu.jar" `
          "$VIEWER\birt\WEB-INF\lib\" -Force

Get-ChildItem "$VIEWER\birt\WEB-INF\lib" -Filter 'db2*'
```

## 6. mxreportdatasources.properties 설정

Preview 때 접속할 DB 정보입니다. §4에서 `classes` 를 복사할 때 이 파일도 함께 들어갔습니다.

```text
$VIEWER\birt\WEB-INF\classes\mxreportdatasources.properties
```

형식은 원본 파일 주석에 정의돼 있습니다.

```text
<DataSourceName>.url=value
<DataSourceName>.driver=value
<DataSourceName>.username=value
<DataSourceName>.password=value
<DataSourceName>.schemaowner=value
```

`<DataSourceName>` 은 보고서가 참조하는 데이터소스 이름이며, 제품 보고서는 **`maximoDataSource`** 를 씁니다(`MaximoSystemLibrary.maximoDataSource` 를 상속).

현재 환경 기준 설정값입니다.

```properties
maximoDataSource.url=jdbc:db2://192.168.2.211:31899/BLUDB
maximoDataSource.driver=com.ibm.db2.jcc.DB2Driver
maximoDataSource.username=db2inst1
maximoDataSource.password=<비밀번호>
maximoDataSource.schemaowner=MAXIMO
```

| 항목 | 값 | 근거 |
|---|---|---|
| DB Host | `192.168.2.211` | SNO NodePort |
| DB Port | `31899` | 평문 NodePort. **설치마다 달라집니다** |
| DB Name | `BLUDB` | |
| Driver | `com.ibm.db2.jcc.DB2Driver` | |
| Schema Owner | `MAXIMO` | 세션 기본 스키마로 설정되어 SQL에 접두어가 필요 없습니다 |

계정·비밀번호는 [ACCESS.md](../installation/ACCESS.md)를, 포트 확인은 [OFFLINE_INSTALL.md §3.12](../installation/OFFLINE_INSTALL.md#312-데이터베이스db2-접속-사용자-계정)를 보세요.

🔴 **이 파일을 채우기 전에 §7로 넘어가면 안 됩니다.** §7에서 이 파일을 프래그먼트에 그대로 담습니다. 주석만 있는 원본 템플릿이 들어가면 모든 값이 `null` 이 되어 `Class.forName(null)` 에서 NullPointerException이 납니다.

🔴 **DB 비밀번호가 들어갑니다.** BIRT 설치 폴더에만 두고 저장소에 커밋하지 마세요.

> 이 파일은 **설계 시점에만** 쓰입니다. Manage에서 보고서를 실행할 때는 사용되지 않습니다.

## 7. OSGi 프래그먼트 생성 및 배치

### 왜 필요한가

§4처럼 클래스를 복사하고 §8처럼 Classpath를 등록해도 Preview에서 이 오류가 납니다.

```
Invalid javascript expression: ReferenceError: "MXReportScriptContext" is not defined
Wrapped java.lang.ClassNotFoundException: com.ibm.tivoli.maximo.report.script.MXReportScriptContext
  cannot be found by org.mozilla.rhino_1.8.0.v20250113-1000
```

마지막 줄이 핵심입니다. BIRT·Eclipse는 **플러그인(번들)마다 클래스로더가 분리된 OSGi 구조**이고, 리포트 스크립트는 Rhino 번들 안에서 실행됩니다. `Report Design → Classpath` 설정은 그 번들 클래스로더에 반영되지 않아, 클래스가 디스크에 있어도 보이지 않습니다.

**BIRT 버전 문제가 아닙니다.** 구버전에서는 될 것으로 보고 4.18(Rhino 1.7.15)을 따로 설치해 같은 조건으로 시험했으나 결과가 같았습니다.

| BIRT | Rhino | §4 + §8만 적용 | 결과 |
|---|---|---|---|
| 4.21.0 | 1.8.0 | 적용 | ❌ `not defined` |
| 4.18.0 | 1.7.15 | 적용 | ❌ `not defined` |

즉 프래그먼트는 특정 버전을 회피하는 우회책이 아니라, **IBM 문서 절차에서 빠져 있는 조각**입니다. 버전을 낮춰도 해결되지 않으므로 IBM이 명시한 4.21을 그대로 쓰면 됩니다.

**프래그먼트(fragment)** 는 매니페스트에 `Fragment-Host` 를 선언한 JAR로, 기동 시 OSGi가 그 호스트 번들에 **물리적으로 합쳐버립니다.** 합쳐진 뒤에는 원래부터 호스트 JAR 안에 있던 것과 구별되지 않습니다. 경계를 우회하는 게 아니라 없애는 방식입니다.

```text
[ org.mozilla.rhino.jar ]  +  [ 우리 JAR: Fragment-Host: org.mozilla.rhino ]
                    ↓ 기동 시 병합
[ org.mozilla.rhino ]  ← Maximo 클래스가 그 안에 있던 것처럼 취급됨
```

호스트를 3개로 하는 이유는 스크립트 클래스 조회를 실제로 수행하는 번들을 특정할 수 없기 때문입니다.

| 호스트 번들 | 역할 |
|---|---|
| `org.mozilla.rhino` | 자바스크립트 엔진 |
| `org.eclipse.birt.core` | BIRT 스크립트 실행 기반 |
| `org.eclipse.birt.report.engine` | 리포트 생성 엔진 |

### 7.1 프래그먼트 재료 준비

세 가지를 한 폴더에 모읍니다. **셋 다 같은 클래스로더에 있어야** 합니다.

| 내용 | 이유 |
|---|---|
| `com/ibm/tivoli/maximo/...` | Script Library 클래스 |
| `com/ibm/db2/jcc/...` | 드라이버가 같은 로더에 없으면 `Class.forName` 이 실패 |
| `mxreportdatasources.properties` (**루트**) | `getClass().getResourceAsStream("/mxreportdatasources.properties")` 로 읽음 |

```powershell
$STAGE = 'C:\Temp\birt-frag'
Remove-Item $STAGE -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory $STAGE -Force | Out-Null

robocopy "$SMP\reports\birt\scriptlibrary\classes\com" "$STAGE\com" /E /NFL /NDL /NJH /NJS
Copy-Item "$VIEWER\birt\WEB-INF\classes\mxreportdatasources.properties" $STAGE

& "$JDK\jar.exe" --extract --file "$SMP\applications\maximo\lib\db2jcc.jar" --dir $STAGE
& "$JDK\jar.exe" --extract --file "$SMP\applications\maximo\lib\db2jcc_license_cu.jar" --dir $STAGE
Remove-Item "$STAGE\META-INF\MANIFEST.MF" -Force -ErrorAction SilentlyContinue
```

⚠️ properties는 **SMP 원본이 아니라 §6에서 값을 채운 `$VIEWER\birt\WEB-INF\classes` 쪽**에서 복사합니다. SMP 원본은 전부 주석 처리된 템플릿입니다.

DB2 드라이버는 JAR 참조가 아니라 **풀어서** 넣습니다. 프래그먼트에 `Bundle-ClassPath` 를 선언하면 `.` 이 호스트 기준으로 해석되어 프래그먼트 루트의 리소스를 못 찾습니다.

### 7.2 프래그먼트 생성

🔴 **BIRT를 완전히 종료한 상태에서** 실행합니다. 실행 중이면 JAR이 잠깁니다.

```powershell
$fragHosts = @{
  'com.ibm.tivoli.maximo.scriptlibrary'        = 'org.mozilla.rhino'
  'com.ibm.tivoli.maximo.scriptlibrary.core'   = 'org.eclipse.birt.core'
  'com.ibm.tivoli.maximo.scriptlibrary.engine' = 'org.eclipse.birt.report.engine'
}
New-Item -ItemType Directory "$ECL\dropins" -Force | Out-Null

foreach ($k in $fragHosts.Keys) {
  $mf = "$env:TEMP\frag.mf"
  @('Manifest-Version: 1.0',
    'Bundle-ManifestVersion: 2',
    "Bundle-SymbolicName: $k",
    'Bundle-Version: 1.3.0',
    "Fragment-Host: $($fragHosts[$k])") | Set-Content -Encoding ascii $mf

  & "$JDK\jar.exe" --create --file "$ECL\dropins\${k}_1.3.0.jar" --manifest $mf -C $STAGE .
}

Get-ChildItem "$ECL\dropins"
```

만들어지는 매니페스트입니다. `Fragment-Host` 한 줄이 이 JAR을 프래그먼트로 만듭니다.

```text
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-SymbolicName: com.ibm.tivoli.maximo.scriptlibrary
Bundle-Version: 1.3.0
Fragment-Host: org.mozilla.rhino
```

명령에서 주의할 점입니다.

| 부분 | 이유 |
|---|---|
| `-Encoding ascii` | 기본 인코딩은 BOM이 붙어 매니페스트가 무시됩니다 |
| `-C $STAGE .` | JAR 루트부터 담기게 합니다. 한 단계 깊어지면 클래스도 properties도 못 찾습니다 |
| `${k}` | 중괄호가 없으면 `$k_1` 을 변수명으로 읽습니다 |

### 7.3 배치 위치

**`dropins` 이며 `plugins` 가 아닙니다.**

```text
eclipse\
├─ plugins\      ← 여기 아님. p2가 bundles.info로 관리해 임의 JAR은 무시됨
├─ dropins\      ← 여기. 기동 시 자동 등록
│  ├─ com.ibm.tivoli.maximo.scriptlibrary_1.3.0.jar
│  ├─ com.ibm.tivoli.maximo.scriptlibrary.core_1.3.0.jar
│  └─ com.ibm.tivoli.maximo.scriptlibrary.engine_1.3.0.jar
├─ birt.exe
└─ birt.ini
```

🔴 **같은 `Bundle-SymbolicName` 의 옛 버전 JAR을 반드시 지우세요.** 두 버전이 남아 있으면 어느 쪽이 붙을지 모호해집니다.

### 7.4 `-clean` 기동

`dropins` 는 **기동 시에만** 스캔됩니다. 프래그먼트를 만들거나 바꿀 때마다 필요합니다.

```powershell
& "$ECL\birt.exe" -clean
```

등록 여부는 이걸로 확인합니다.

```powershell
Select-String 'scriptlibrary' "$ECL\configuration\org.eclipse.equinox.simpleconfigurator\bundles.info"
```

```text
com.ibm.tivoli.maximo.scriptlibrary,1.3.0,dropins/com.ibm.tivoli.maximo.scriptlibrary_1.3.0.jar,4,false
```

### 7.5 접속정보를 바꿀 때

🔴 `mxreportdatasources.properties` 가 **두 벌**이 됩니다. DB 정보를 바꾸면 둘 다 갱신해야 합니다.

| 위치 | 처리 |
|---|---|
| `$VIEWER\birt\WEB-INF\classes\` | 직접 편집 |
| `dropins\*_1.3.0.jar` 안의 루트 | §7.1~7.4를 다시 수행 (버전 올리고 이전 JAR 삭제) |

## 8. Project Classpath 설정

```
Report Project 선택 → Project → Properties → Report Design → Classpath
→ Add External Class Folder → $VIEWER\birt\WEB-INF\classes
```

> 이 설정만으로는 §7의 문제가 해결되지 않습니다. Viewer 실행 등 다른 경로를 위해 등록해둡니다.

## 9. Report Library · Template 경로 설정

**Library**

```
Window → Preferences → Report Design → Resource → File System
```

```text
C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\reports\birt\libraries
```

🔴 `libraries` **까지** 지정해야 합니다. 상위 `reports\birt` 로 잡으면 `MaximoSystemLibrary.rptlibrary` 를 못 찾아 데이터소스·테마 상속이 깨집니다.

**Template**

```
Window → Preferences → Report Design → Template
```

```text
C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\reports\birt\templates
```

설정 후 BIRT를 재시작합니다.

## 10. 기존 보고서 Preview 테스트

신규 개발 전에 **제품 보고서로 환경을 검증**합니다. 여기서 통과하지 못하면 신규 보고서도 안 됩니다.

**1) 기존 Maximo 보고서 파일을 BIRT Report Designer에서 Open**

파라미터가 적은 `ASSET\asset_detail.rptdesign` 을 권합니다.

```text
C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\reports\birt\reports\ASSET\asset_detail.rptdesign
```

**2) Preview 실행**

파라미터 입력창이 뜹니다. `where` 는 Maximo가 화면의 조회조건을 넘겨주는 자리라, 단독 Preview에서는 전체 조회로 둡니다.

| 파라미터 | 입력 |
|---|---|
| `where` | `1=1` |
| `appname` | 비움 |
| `paramdelimiter` | 비움 |
| `paramstring` | 비움 |

**3) 로그로 판정**

제품 보고서의 `initialize` 스크립트가 로그 파일 경로를 지정합니다. `asset_detail` 은 `C:\temp\asset_detail.log` 입니다.

```powershell
Get-Content C:\temp\asset_detail.log -Tail 40
```

| 로그 | 의미 | 대응 |
|---|---|---|
| `Designtime DataSource [maximoDataSource] [url] = jdbc:db2://...` | 접속정보 정상 | 이후 실패는 네트워크·계정 문제 |
| `[url] = null` + `NullPointerException` | 프래그먼트 안 properties가 비어 있음 | §6 → §7 다시 |
| `ERROR MXReportJobCancelManager birtAdminService is null` | Maximo 런타임 밖이라 나는 **정상 메시지** | 무시 |

화면 오류 기준입니다.

| 오류 | 원인 | 대응 |
|---|---|---|
| `"MXReportScriptContext" is not defined` | 스크립트 클래스가 안 보임 | §7 |
| `cannot be found by org.mozilla.rhino_...` | 위와 동일 | §7 |
| 오류 없이 빈 화면 | 데이터셋이 빈 결과 | 로그의 `[url]` 확인 |

그 밖에 확인할 것들입니다.

| 확인 | 방법 |
|---|---|
| DB 포트 도달 | `Test-NetConnection 192.168.2.211 -Port 31899` |
| JDBC Driver 배치 | `Get-ChildItem "$VIEWER\birt\WEB-INF\lib" -Filter 'db2*'` |
| 프래그먼트 등록 | §7.4의 `bundles.info` 조회 |
| classes 계층 | `WEB-INF\classes\com\...` 인지 (`classes\classes` 아님) |
| Library 경로 | §9 |

## 11. Manage에 보고서 등록

개발·테스트가 끝난 산출물을 Manage에 올립니다.

| 대상 | |
|---|---|
| `.rptdesign` | 보고서 정의 |
| `.rptlibrary` | 공용 라이브러리를 수정한 경우 |
| Report Resource | 이미지·properties 등 참조 리소스 |

```
Maximo Manage → Report Administration → 보고서 등록 / Import
```

등록 후 할 일입니다.

- 보고서 메타데이터 설정
- Application 연결
- Parameter 설정
- 실제 화면에서 실행 테스트

> Java 클래스와 달리 보고서는 Customization Archive와 이미지 재빌드를 거치지 않습니다. Report Administration에서 직접 Import하므로 [DEVELOPMENT_SETUP.md §5](DEVELOPMENT_SETUP.md#5-소스-게시-및-mas-배포)의 20~30분 사이클이 필요 없습니다.
