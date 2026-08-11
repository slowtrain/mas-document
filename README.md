# IBM Maximo Application Suite (MAS) 설치·구성 문서

이 저장소는 IBM Maximo Application Suite(MAS)를 설치하고 구성하는 절차와 방법을 담은 문서 저장소입니다.

## 문서 구조

```text
docs/
  installation/
    INSTALL_SERVER.md   폐쇄망 설치용 서버·용량·계정 준비
    INSTALL_FILES.md    인터넷 구간 파일 및 실제 이미지 세트 준비
    INSTALL_OFFLINE.md  폐쇄망 설치 절차
    INSTALL_ONLINE.md   온라인 설치 절차
    INSTALL_SUMMARY.md  설치 기준 및 준비 항목 요약
    INSTALLATION.md     환경별 입력값 기록 양식
  development/
    DEVELOPMENT_SETUP.md  Java 커스터마이징 개발 환경 흐름
```

## 에이전트 작업 규칙

코드·문서 에이전트(Claude, Codex 등)는 작업 시작 전 아래 파일을 읽습니다.

- `CLAUDE.md` — Claude Code 전용 규칙
- `AGENTS.md` — 에이전트 공통 규칙
