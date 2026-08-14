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

## 2. scriptlibrary 클래스 복사

`classes` **안의 내용**이 `WEB-INF\classes` 아래로 들어가야 합니다.

```
<SMP_ROOT>\writeable\maximo\reports\birt\scriptlibrary\classes 
```
->

```
<BIRT_REPORT_DESIGNER_ROOT>\plugins\org.eclipse.birt.report.viewer_4.21.0.v202508260649\birt\WEB-INF\classes
```

## 3. JDBC Driver 복사
데이터베이스 접속을 위한 드라이버를 복사합니다.

```
<SMP_ROOT>\writeable\maximo\applications\maximo\lib\db2jcc.jar
<SMP_ROOT>\writeable\maximo\applications\maximo\lib\db2jcc_license_cu.jar
```
->

```
<BIRT_REPORT_DESIGNER_ROOT>\plugins\org.eclipse.birt.report.viewer_4.21.0.v202508260649\birt\WEB-INF\lib\db2jcc.jar
<BIRT_REPORT_DESIGNER_ROOT>\plugins\org.eclipse.birt.report.viewer_4.21.0.v202508260649\birt\WEB-INF\lib\db2jcc_license_cu.jar
```

## 4. mxreportdatasources.properties 설정
편집기로 열어서 아래 내용 입력
<BIRT_REPORT_DESIGNER_ROOT>\plugins\org.eclipse.birt.report.viewer_4.21.0.v202508260649\birt\WEB-INF\classes\mxreportdatasources.properties


```
maximoDataSource.url=jdbc:db2://192.168.2.211:31899/BLUDB
maximoDataSource.driver=com.ibm.db2.jcc.DB2Driver
maximoDataSource.username=db2inst1
maximoDataSource.password=<비밀번호>
maximoDataSource.schemaowner=MAXIMO
```

## 5. Project 생성 

```
Create a project → Report Project → Input "Project name"   → Finish    
```

## 7. Design Import 

```
Report Project 선택  → import → General → File System → select folder  → Check Design file   → Finish

```

## 6. Project Classpath 설정

```
Report Project 선택 → Project → Properties → Report Design → Classpath
→ Add External Class Folder → $VIEWER\birt\WEB-INF\classes
```

## 6. Report Library  경로 설정

```
Window → Preferences → Report Design → Resource → File System
```

```text
C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\reports\birt\libraries
```

## 7. Template 경로 설정

```
Window → Preferences → Report Design → Template
```

```text
C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\reports\birt\templates
```

## 8. Report 수정후 preview 테스트

수정하고자하는 *.rptdesign 파일선택후 수정 후 preview 클릭.

## 9. Manage에 보고서 등록

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




## 10. OSGi 프래그먼트 생성 및 배치 (정상 preview 안될시)

### 왜 필요한가

9.2에서 정상적으로 동작하는 리포트가 Preview에서 오류 발생하여 우회

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

### 10.1 프래그먼트 재료 준비

세 가지를 한 폴더에 모읍니다. **셋 다 같은 클래스로더에 있어야** 합니다.

| 내용 | 이유 |
|---|---|
| `com/ibm/tivoli/maximo/...` | Script Library 클래스 |
| `com/ibm/db2/jcc/...` | 드라이버가 같은 로더에 없으면 `Class.forName` 이 실패 |
| `mxreportdatasources.properties` (**루트**) | `getClass().getResourceAsStream("/mxreportdatasources.properties")` 로 읽음 |

작업 폴더는 `C:\Temp\birt-frag` 입니다. PowerShell에서 아래를 순서대로 실행합니다.

**1) 작업 폴더 비우고 만들기**

```powershell
Remove-Item "C:\Temp\birt-frag" -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory "C:\Temp\birt-frag" -Force
```

**2) Script Library 클래스 복사**

```powershell
robocopy "C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\reports\birt\scriptlibrary\classes\com" "C:\Temp\birt-frag\com" /E /NFL /NDL /NJH /NJS
```

**3) 접속정보 복사**

```powershell
Copy-Item "C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse\plugins\org.eclipse.birt.report.viewer_4.21.0.v202508260649\birt\WEB-INF\classes\mxreportdatasources.properties" "C:\Temp\birt-frag"
```

⚠️ SMP 원본이 아니라 **§4에서 값을 채운 BIRT 쪽 파일**을 복사합니다. SMP 원본은 전부 주석 처리된 템플릿이라, 그게 들어가면 접속정보가 모두 `null` 이 되어 NullPointerException이 납니다.

**4) DB2 드라이버 풀어 넣기**

```powershell
& "C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin\jar.exe" --extract --file "C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\applications\maximo\lib\db2jcc.jar" --dir "C:\Temp\birt-frag"

& "C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin\jar.exe" --extract --file "C:\Source\mas\mas-files\smp-maximo\SMP\writeable\maximo\applications\maximo\lib\db2jcc_license_cu.jar" --dir "C:\Temp\birt-frag"

Remove-Item "C:\Temp\birt-frag\META-INF\MANIFEST.MF" -Force -ErrorAction SilentlyContinue
```

JAR을 그대로 넣지 않고 푸는 이유는, 프래그먼트에 `Bundle-ClassPath` 를 선언하면 `.` 이 호스트 기준으로 해석되어 프래그먼트 루트의 리소스를 못 찾기 때문입니다.

**5) 확인** — 셋 다 `True` 여야 합니다.

```powershell
Test-Path "C:\Temp\birt-frag\com\ibm\tivoli\maximo\report\script\MXReportScriptContext.class"
Test-Path "C:\Temp\birt-frag\com\ibm\db2\jcc\DB2Driver.class"
Test-Path "C:\Temp\birt-frag\mxreportdatasources.properties"
```

### 10.2 프래그먼트 생성

🔴 **BIRT를 완전히 종료한 상태에서** 진행합니다. 실행 중이면 JAR이 잠겨 만들거나 지울 수 없습니다.

**1) 매니페스트 파일 3개 작성**

`C:\Temp\birt-frag-mf` 폴더를 만들고, 편집기로 아래 3개 파일을 만듭니다. 내용은 `Bundle-SymbolicName` 과 `Fragment-Host` 두 줄만 다릅니다.

`rhino.mf`

```text
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-SymbolicName: com.ibm.tivoli.maximo.scriptlibrary
Bundle-Version: 1.3.0
Fragment-Host: org.mozilla.rhino
```

`core.mf`

```text
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-SymbolicName: com.ibm.tivoli.maximo.scriptlibrary.core
Bundle-Version: 1.3.0
Fragment-Host: org.eclipse.birt.core
```

`engine.mf`

```text
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-SymbolicName: com.ibm.tivoli.maximo.scriptlibrary.engine
Bundle-Version: 1.3.0
Fragment-Host: org.eclipse.birt.report.engine
```

🔴 **마지막 `Fragment-Host` 줄을 입력한 뒤 `Enter` 를 한 번 누르세요.** 개행이 없으면 `jar` 명령이 그 줄을 **조용히 버립니다.** 오류도 안 나고 JAR 크기도 같아서 알아채기 어렵습니다.

🔴 메모장이면 인코딩을 **`ANSI`** 로 저장하세요. BOM이 붙으면 매니페스트가 통째로 무시됩니다.

개행이 들어갔는지 확인합니다. 끝이 `0D 0A` 로 나와야 합니다.

```powershell
Format-Hex "C:\Temp\birt-frag-mf\rhino.mf" | Select-Object -Last 1
```

**2) dropins 폴더 만들기**

```powershell
New-Item -ItemType Directory "C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse\dropins" -Force
```

**3) JAR 3개 생성**

```powershell
& "C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin\jar.exe" --create --file "C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse\dropins\com.ibm.tivoli.maximo.scriptlibrary_1.3.0.jar" --manifest "C:\Temp\birt-frag-mf\rhino.mf" -C "C:\Temp\birt-frag" .

& "C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin\jar.exe" --create --file "C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse\dropins\com.ibm.tivoli.maximo.scriptlibrary.core_1.3.0.jar" --manifest "C:\Temp\birt-frag-mf\core.mf" -C "C:\Temp\birt-frag" .

& "C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin\jar.exe" --create --file "C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse\dropins\com.ibm.tivoli.maximo.scriptlibrary.engine_1.3.0.jar" --manifest "C:\Temp\birt-frag-mf\engine.mf" -C "C:\Temp\birt-frag" .
```

`-C "C:\Temp\birt-frag" .` 가 중요합니다. 이렇게 해야 JAR 루트에 `com\...` 과 `mxreportdatasources.properties` 가 들어갑니다. 한 단계 깊어지면 클래스도 접속정보도 못 찾습니다.

**4) 확인 — `Fragment-Host` 가 JAR 안에 들어갔는지**

JAR 3개가 각각 6MB 남짓이어야 합니다.

```powershell
Get-ChildItem "C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse\dropins"
```

🔴 **크기만 보고 넘어가면 안 됩니다.** 매니페스트를 꺼내 `Fragment-Host` 줄이 있는지 직접 확인하세요.

```powershell
Remove-Item "C:\Temp\mfcheck" -Recurse -Force -ErrorAction SilentlyContinue

& "C:\Jdk\openjdk-25+36_windows-x64_bin\jdk-25\bin\jar.exe" --extract --file "C:\IDE\birt-report-designer-all-in-one-4.21.0-202509121029-win32.win32.x86_64\eclipse\dropins\com.ibm.tivoli.maximo.scriptlibrary_1.3.0.jar" --dir "C:\Temp\mfcheck" META-INF/MANIFEST.MF

Get-Content "C:\Temp\mfcheck\META-INF\MANIFEST.MF"
```

이렇게 나와야 정상입니다.

```text
Manifest-Version: 1.0
Bundle-ManifestVersion: 2
Bundle-SymbolicName: com.ibm.tivoli.maximo.scriptlibrary
Bundle-Version: 1.3.0
Fragment-Host: org.mozilla.rhino
Created-By: 25 (Oracle Corporation)
```

`Fragment-Host` 가 없으면 §10.2-1의 개행 문제입니다. `.mf` 파일 끝에 개행을 넣고 JAR을 다시 만드세요. 나머지 두 JAR도 같은 방법으로 확인합니다.

### 10.3 배치 위치

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

### 10.4 `-clean` 기동

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

### 10.5 접속정보를 바꿀 때

🔴 `mxreportdatasources.properties` 가 **두 벌**이 됩니다. DB 정보를 바꾸면 둘 다 갱신해야 합니다.

| 위치 | 처리 |
|---|---|
| `$VIEWER\birt\WEB-INF\classes\` | 직접 편집 |
| `dropins\*_1.3.0.jar` 안의 루트 | §7.1~7.4를 다시 수행 (버전 올리고 이전 JAR 삭제) |
