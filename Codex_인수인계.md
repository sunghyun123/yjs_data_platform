# YJS Data Platform — 다음 Codex 세션 인수인계

> 작성 기준: 2026-08-20 KST
> 상태: 발견·기획 완료, 구현 전
> 다음 세션의 우선 목표: NAS 보안 전제 확인과 Universal Search 파일럿 완료

## Goal

ThinkWise 업무 이력과 Synology NAS 문서를 읽기 전용으로 통합하고, 출처가 있는 프로젝트별 AI 업무 요약을 MCP 도구로 제공한다. 첫 유용한 결과물은 **ThinkWise 최근 변경 + 주간업무보고의 계획/진척/미달 사유를 결합한 프로젝트 주간 브리프**다.

## Context

### 기존 시스템

- `yjs_backoffice`: 1차 납품이 끝난 경영 대시보드. 이 프로젝트와 배포 생명주기를 분리한다.
- `thinkwise-wiki`: ThinkWise MariaDB의 `work_log`를 PK 커서 기반으로 약 60초마다 읽기 전용 증분 동기화한다. 약 54만 행의 `data/wiki_index.db`를 대시보드와 위키가 함께 사용한다.
- `yjs_ai_manager`: NAS의 주간업무보고 Excel 파서와 AI 검토 MVP가 있다. 금액 원문을 외부 AI에 보내지 않는 정책과 파싱 코드를 검토해 재사용한다.
- Windows 상시 서버: `192.168.0.76`. 대시보드, ThinkWise 공유 인덱스, Tailscale을 운영한다. 이 플랫폼도 여기서 실행하는 방향이다.
- Synology NAS: `192.168.0.67`, 호스트명 `yec`, DS923+, DSM 7.2.2 계열.

### NAS에서 읽기 전용으로 확인한 사실

- `/volume1`은 약 7.0TB 중 5.6TB 사용, 약 1.5TB 여유(80% 사용)다.
- 대상은 `/volume1/주식회사 영전사/9. 주간업무계획`이다.
- 시스템 메타데이터 폴더를 제외하면 약 140개 파일, 36개 디렉터리, 8.1MB다.
- 대략 `xlsx` 129개, `db` 8개, `xlsm` 2개, `txt` 1개다.
- Universal Search 패키지 `SynoFinder 1.7.1-0800`, Synology Drive `3.5.2-26110`, Universal Viewer가 설치되어 있다.
- `synopkg status SynoFinder`는 stopped/상태 조회 실패로 표시됐지만 `synoelasticd`, `fileindexd`, `synoindexd` 계열 프로세스는 실행 중이었다. 패키지 상태 명령만 믿지 말고 DSM UI와 실제 검색으로 확인한다.
- 실제 DSM API 목록에는 `SYNO.Finder.FileIndexing.Search`, `Status`, `Folder`, `Term`과 공식 `SYNO.FileStation.Search`가 노출된다.
- `SYNO.Finder.*`는 공개 개발자 문서가 없는 내부 API다. 사용하더라도 어댑터 뒤에 격리하고 실패 시 대체 경로를 둔다.
- DS923+에는 AI Search가 요구하는 GPU가 없다. Synology의 의미 검색/AI Search를 전제로 설계하지 않는다. Universal Search는 정확한 키워드·본문 검색 기반으로만 사용한다.

### 매우 중요한 보안 상태

- 사용자가 채팅에 입력한 임시 NAS 비밀번호는 이미 노출된 것으로 간주하고 **반드시 교체**한다. 이 저장소 어디에도 그 값을 기록하지 않았다.
- 확인 당시 임시 NAS 계정은 `administrators` 그룹에 속했다. 따라서 읽기 전용 애플리케이션 계정이 아니며 MCP에 사용하면 안 된다.
- Synology SSH는 일반적으로 관리자 계정이 필요할 수 있다. 관리/진단용 SSH 계정과 서비스용 읽기 전용 계정을 분리한다.
- 서비스용 계정은 SSH 없이, 대상 공유 폴더에만 읽기 권한을 부여하고 File Station API 또는 SMB로 사용한다.

## Architecture decision

권장안은 **NAS 원본 + Windows 검색/요약 계층의 하이브리드 구조**다.

```text
ThinkWise MariaDB
  -> 기존 thinkwise-wiki 증분 동기화
  -> wiki_index.db -----------------------------+
                                                  \
                                                   -> Retrieval API -> MCP -> Codex
                                                  /             \
NAS 원본 -> Universal Search / File Station API -+               -> AI weekly brief
        \-> LocalIndexProvider(내부 API 장애 시 대체)
```
결정 이유:

- NAS는 파일 원본과 사람용 검색을 담당해 운영 복잡도를 낮춘다.
- Windows 서버가 데이터 연결, 프로젝트 매핑, 접근 통제, 인용, MCP를 담당한다.
- 기존 ThinkWise 인덱스를 재사용해 동기화·장애 지점을 중복시키지 않는다.
- Synology 내부 API 변경 위험을 공급자 인터페이스 뒤에 격리한다.
- 지금은 Container Manager, Elasticsearch, 별도 벡터 DB를 설치하지 않는다. 140개 문서 파일럿에는 과도하다.

검토했던 대안:

1. NAS Universal Search만 사용: 가장 빠르지만 MCP와 통합 요약, 안정적인 API 계약이 부족하다.
2. 모든 문서를 Windows로 복제·자체 색인: 통제력은 높지만 삭제 반영, 중복 저장, 운영 부담이 커진다.
3. 벡터 DB부터 도입: 의미 검색은 좋아질 수 있으나 권한·인용·평가가 선행되지 않아 현재는 과잉 설계다.

## Scope

### 첫 파일럿에 포함

- NAS `9. 주간업무계획` 폴더만 내용 인덱싱
- 파일명과 셀 본문 키워드 검색
- 공식 File Station API를 이용한 목록·메타데이터·허용 문서 읽기
- Finder 내부 검색 API의 실제 요청/응답 계약 조사
- `wiki_index.db` 읽기 전용 검색 어댑터
- 프로젝트 별칭 테이블을 통한 명시적 NAS↔ThinkWise 연결
- 근거 링크와 데이터 기준 시각이 있는 수동 실행 주간 브리프
- 읽기 전용 MCP 도구와 상태 확인 도구

### 지금 제외

- NAS 전체 볼륨 색인
- Synology AI Search/의미 검색 의존
- 문서 또는 ThinkWise 원본 수정
- 임의 경로 파일 읽기 MCP 도구
- 공개 인터넷 노출 또는 Tailscale Funnel
- 자동 쓰기/승인/메일 발송 MCP 도구
- 검증 전 자동 스케줄 요약
- 별도 벡터 DB와 대형 검색 클러스터

## Planned interfaces

검색 공급자는 최소 다음 계약으로 분리한다. 구체적인 Python 타입은 구현 세션에서 테스트와 함께 확정한다.

- `ThinkwiseIndexProvider`: 기존 SQLite 인덱스 검색과 freshness 반환
- `NasSearchProvider`: NAS 검색의 추상 인터페이스
- `SynologyFinderProvider`: 내부 Finder 검색 API 어댑터
- `LocalIndexProvider`: Finder가 불안정할 때 사용하는 보조 인덱스
- `SynologyFileStationClient`: 공식 API 인증, 목록, 메타데이터, 제한된 다운로드
- `ProjectAliasRepository`: 사람이 승인한 프로젝트 이름 연결
- `CitationBuilder`: 문서 ID, 경로, 수정 시각, 업무 이력 ID를 요약 근거로 변환

초기 MCP 도구:

- `search_thinkwise(query, project_id?, date_from?, date_to?, limit?)`
- `search_nas_documents(query, project_id?, date_from?, date_to?, limit?)`
- `get_document_excerpt(document_id, locator?)`
- `get_project_context(project_id, date_from?, date_to?)`
- `get_source_health()`

`document_id`는 검색 결과가 발급한 허용 범위 ID여야 한다. 사용자가 넣은 절대 경로나 UNC 경로를 그대로 여는 도구는 금지한다.

## Plan

### Phase 0 — 보안 전제 정리

1. 채팅에 노출된 임시 NAS 비밀번호를 교체한다.
2. 관리/SSH 계정과 서비스 계정을 분리한다.
3. 서비스 계정을 관리자 그룹에서 제외하고 `9. 주간업무계획`에만 읽기 권한을 준다.
4. 서비스 계정으로 허용 폴더는 읽고 다른 공유 폴더는 읽지 못하는지 확인한다.
5. 실제 값은 로컬 `.env`에만 입력하고 Git 상태로 추적되지 않음을 확인한다.

검토 지점: 위 조건이 충족되기 전에는 MCP 또는 자동 수집 코드에 NAS 자격 증명을 연결하지 않는다.

### Phase 1 — Universal Search 파일럿

1. DSM Universal Search에서 `주식회사 영전사/9. 주간업무계획`만 색인 대상으로 추가한다.
2. 방식은 파일 내용 인덱싱, 유형은 문서로 제한한다.
3. Excel 셀 본문에만 있고 파일명에는 없는 알려진 문자열 10개를 선정한다.
4. 10개 중 최소 9개 검색, 올바른 파일 연결, 권한 밖 문서 미노출을 확인한다.
5. 인덱싱 중 NAS CPU·메모리·디스크 부하와 검색 지연을 기록한다.

실패 시: 전체 공유 폴더로 범위를 넓히지 않는다. 지원되지 않는 Excel 구조, 암호화 파일, 오래된 형식, 인덱스 완료 여부를 구분해 기록한다.

### Phase 2 — API 탐색과 계약 고정

1. 브라우저 개발자 도구 또는 읽기 전용 요청으로 DSM UI가 보내는 Finder 검색 요청을 관찰한다.
2. 요청 파라미터, 페이징, 결과 ID, 경로, 하이라이트, 오류를 민감정보 없이 기록한다.
3. 공식 File Station API로 제한 계정 로그인, 폴더 목록, 메타데이터, 파일 읽기를 검증한다.
4. Finder 실패·응답 변경·권한 오류를 합성한 계약 테스트를 먼저 만든다.
5. Finder 결과가 불안정하면 즉시 `LocalIndexProvider` 파일럿으로 전환한다.

검토 지점: Finder는 비공개 API이므로 원시 응답 구조가 도메인 계층 밖으로 새지 않게 한다.

### Phase 3 — 검색 수직 슬라이스

1. Python 3.11+ 프로젝트 구조, 설정 검증, 로깅 마스킹, 테스트 기반을 만든다.
2. `ThinkwiseIndexProvider`로 기존 `wiki_index.db`를 읽기 전용 연결한다.
3. NAS 검색과 제한 문서 조회 어댑터를 구현한다.
4. `ProjectAliasRepository`를 만들고 자동 fuzzy 매칭 없이 승인된 별칭만 사용한다.
5. 검색 결과에 source, source_id, title/path, modified_at, indexed_at, excerpt를 포함한다.
6. 삭제·이동된 NAS 파일은 파생 인덱스와 캐시에서 제거되는지 검증한다.

### Phase 4 — MCP

1. 현재 공식 MCP Python SDK와 Codex MCP 설정 문서를 다시 확인한다.
2. 위 5개 읽기 전용 도구를 먼저 STDIO 또는 localhost에서 제공한다.
3. 원격 사용은 Streamable HTTP + Tailscale로 제한한다.
4. 단일 사용자 파일럿은 환경 변수 기반 bearer token, 다중 사용자 전환 시 OAuth를 검토한다.
5. 도구별 입력 제한, 최대 결과 수, timeout, 감사 이벤트를 둔다. 본문은 로그하지 않는다.

### Phase 5 — AI 주간 브리프

1. `yjs_ai_manager`의 Excel 파서와 금액 마스킹을 검토해 재사용 가능 부분을 분리한다.
2. ThinkWise 변경, 이번 주 계획/진척/미달 사유를 프로젝트별로 묶는다.
3. 모든 문장에 출처를 연결하고, 자료 간 충돌과 미확정 연결은 명시한다.
4. 10~20개 실제 사례를 사람이 평가한다.
5. 근거 없는 사실 주장 0건, 모든 핵심 주장 인용 100%를 만족한 뒤에만 스케줄 실행을 검토한다.

## Commands for the next session

먼저 저장소와 관련 자료를 읽는다.

```powershell
Set-Location 'C:\Users\조성현\Documents\GitHub\yjs_data_platform'
Get-Content -Raw .\AGENTS.md
Get-Content -Raw .\Codex_인수인계.md
Get-Content -Raw .\개발진행현황.md
git status -sb

Get-Content -Raw ..\thinkwise-wiki\README.md
Get-Content -Raw ..\yjs_ai_manager\README.md
rg -n "wiki_index|NAS|주간업무|금액|mask|xlsx" ..\thinkwise-wiki ..\yjs_ai_manager
```

로컬 비밀 설정을 준비하되 값은 출력하지 않는다.

```powershell
Copy-Item .env.example .env
git status --short
git check-ignore -v .env
```

초기 구현이 생긴 뒤의 기본 검증 명령은 프로젝트 도구에 맞게 확정하고 README에 기록한다. Python을 채택한다면 최소 기준은 다음과 같다.

```powershell
python -m pytest
python -m compileall -q src tests
python -m pip check
```

## Verification

### 보안

- 서비스 계정이 관리자 그룹에 속하지 않는다.
- 허용 폴더 밖의 목록·검색·다운로드가 모두 거부된다.
- `.env`, 토큰, 세션, NAS 문서, 로컬 인덱스가 Git 추적 대상이 아니다.
- 서비스가 `127.0.0.1` 또는 Tailscale 승인 경로에만 노출된다.

### 검색

- 파일명에 없는 Excel 셀 문자열 10개 중 9개 이상을 찾는다.
- 결과가 올바른 파일과 위치를 가리킨다.
- 수정·삭제 이후 freshness와 제거 동작이 정의대로 반영된다.
- Finder 장애를 주입해도 오류가 제한되고 대체 공급자로 전환 가능하다.

### 요약

- 10~20개 평가 사례에서 근거 없는 사실 주장이 0건이다.
- 핵심 주장마다 파일 또는 ThinkWise 이력 ID와 기준 시각이 있다.
- 프로젝트 이름이 모호하면 자동 연결하지 않고 확인을 요구한다.
- 외부 AI 요청에 금액 원문과 인증 정보가 포함되지 않는다.

## Stop conditions

- 서비스 계정 최소 권한이 입증되지 않음
- 원본 NAS 파일 또는 ThinkWise DB에 쓰기가 필요함
- 대상 폴더를 넘어선 색인·검색이 발생함
- 인덱싱으로 NAS 운영 성능이 눈에 띄게 저하됨
- Finder API 사용에 관리자 세션이 필요하거나 안정적인 권한 격리가 안 됨
- 프로젝트 매핑 정확도를 측정하지 못한 상태에서 자동 요약을 켜야 함
- 출처 없는 AI 결과가 사용자에게 사실처럼 노출됨
- 기존 대시보드/Tailscale 서비스 포트 또는 설정과 충돌함

## Rollback and review points

- Universal Search 부하가 크면 대상 폴더 인덱싱만 해제하고 원본 파일은 건드리지 않는다.
- Finder API가 바뀌면 `SynologyFinderProvider`만 비활성화하고 File Station + 로컬 보조 인덱스로 전환한다.
- MCP 원격 노출에 문제가 생기면 Tailscale Serve 구성을 되돌리고 STDIO/localhost로 제한한다.
- 요약 품질이 기준에 못 미치면 자동 실행을 중단하고 검색 결과 + 수동 Codex 요약 단계로 유지한다.
- 각 Phase 완료 시 diff, 테스트 결과, 실제 접근 범위를 사용자에게 보여준 뒤 다음 Phase로 이동한다.

## Reference links

- Synology Universal Search 도움말: <https://kb.synology.com/en-global/DSM/help/SynoFinder/universalsearch_overview?version=7>
- Synology DSM 소프트웨어 사양: <https://www.synology.com/en-global/dsm/7.3/software_spec/dsm>
- Synology File Station API Guide: <https://global.download.synology.com/download/Document/Software/DeveloperGuide/Package/FileStation/All/enu/Synology_File_Station_API_Guide.pdf>
- Synology DSM Login Web API Guide: <https://global.download.synology.com/download/Document/Software/DeveloperGuide/Os/DSM/All/enu/DSM_Login_Web_API_Guide_enu.pdf>
- MCP 공식 사이트: <https://modelcontextprotocol.io/>
- Codex MCP 설정: <https://learn.chatgpt.com/docs/extend/mcp>

## Start prompt

다음 세션은 아래 요청으로 시작하면 된다.

```text
Goal:
yjs_data_platform의 Phase 0과 Phase 1을 진행한다. NAS 최소 권한 조건을 먼저 검증하고, Universal Search에서 `9. 주간업무계획` 폴더만 파일 내용 인덱싱하여 검색 파일럿을 완료한다.

Context:
먼저 AGENTS.md, README.md, Codex_인수인계.md, 개발진행현황.md를 모두 읽는다. 기존 ThinkWise 인덱스는 thinkwise-wiki의 wiki_index.db를 재사용하며, NAS 원본과 ThinkWise DB에는 절대 쓰지 않는다. 채팅에 노출된 기존 NAS 비밀번호와 관리자 그룹의 임시 계정은 서비스에 사용하지 않는다.

Scope:
계정·권한의 읽기 전용 확인, 대상 폴더 한정 Universal Search 설정 지원, 알려진 문자열 10개 검색 평가, NAS 부하와 권한 격리 기록까지만 한다. 전체 공유 폴더 색인, MCP 구현, AI 자동 요약은 이번 세션 범위가 아니다.

Plan:
1. 저장소 문서와 관련 프로젝트를 읽고 현재 상태를 확인한다.
2. DSM에서 관리 계정과 읽기 전용 서비스 계정이 분리되었는지 검증한다.
3. 대상 폴더만 Universal Search 내용 인덱싱한다.
4. 10개 테스트 검색과 접근 거부 테스트를 수행한다.
5. 결과와 근거를 개발진행현황.md에 기록하고 Phase 2 진행 여부를 제안한다.

Verification:
검색 10건 중 9건 이상 성공, 올바른 파일 연결, 권한 밖 결과 0건, 운영 부하 이상 없음, 비밀정보 Git/로그 노출 0건.

Stop conditions:
서비스 계정이 관리자이거나 폴더 밖을 읽을 수 있으면 중단한다. 색인 부하가 크거나 대상 밖을 색인하면 중단한다. 설정 변경 전후 상태를 기록하고 되돌릴 수 없으면 실행하지 않는다.
```
