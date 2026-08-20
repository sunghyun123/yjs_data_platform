# YJS Data Platform

영전사의 ThinkWise 업무 이력과 Synology NAS 문서를 안전하게 연결해, 근거가 있는 AI 업무 요약과 읽기 전용 MCP 도구를 제공하기 위한 통합 데이터 플랫폼입니다.

## 현재 상태

- 2026-08-20: 아키텍처와 첫 실행 계획 수립
- 구현 코드와 운영 설정은 아직 없음
- 다음 작업: NAS 계정 권한 분리 후 `9. 주간업무계획` 폴더 검색 파일럿

## 목표 구조

```text
ThinkWise MariaDB
  -> thinkwise-wiki의 증분 인덱스(wiki_index.db) 재사용
                                             \
                                              -> YJS Data Platform -> MCP -> Codex/업무 도구
                                             /
Synology NAS
  -> Universal Search + File Station API
  -> 필요 시 로컬 보조 인덱스
```

핵심 원칙은 원본 시스템을 수정하지 않는 읽기 전용 통합, 검색 결과의 출처 표시, 최소 권한, 그리고 교체 가능한 검색 어댑터입니다.

## 먼저 읽을 문서

1. [AGENTS.md](AGENTS.md) — 이 저장소의 필수 작업 규칙
2. [Codex_인수인계.md](Codex_인수인계.md) — 발견 사항, 결정, 실행 계획, 검증 기준
3. [개발진행현황.md](개발진행현황.md) — 실제 구현 진척과 검증 기록

## 보안 주의

- 자격 증명, 토큰, 실제 문서 내용은 Git에 커밋하지 않습니다.
- NAS와 ThinkWise 원본은 읽기 전용으로만 접근합니다.
- 서비스용 NAS 계정에는 관리자/SSH 권한을 부여하지 않습니다.
- 외부 AI로 금액 원문을 전송하지 않는 기존 정책을 유지합니다.

## 관련 저장소

- `yjs_backoffice`: 경영 대시보드
- `thinkwise-wiki`: ThinkWise 증분 동기화와 공유 SQLite 인덱스
- `yjs_ai_manager`: 주간업무보고 Excel 파서와 AI 검토 MVP
