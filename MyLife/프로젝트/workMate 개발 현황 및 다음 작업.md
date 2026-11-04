---
type: project
title: workMate 개발 현황 및 다음 작업
source: Notion
source_url: https://app.notion.com/p/3f36bc51992b81bb9c85e5835649b63e?pvs=204
notion_last_edited: 2026-11-03T07:32:10.909Z
verification: unverified
imported: 2026-11-05
tags:
  - 프로젝트
  - AI 비서
  - 업무 자동화
---

# workMate 개발 현황 및 다음 작업

> Notion 원본: https://app.notion.com/p/3f36bc51992b81bb9c85e5835649b63e?pvs=204
> Notion 최종 수정: 2026-11-03T07:32:10.909Z
> 검증 상태: unverified

## 프로젝트 통합 결정 — Secretary → workMate

**최종 명칭은 workMate로 통일한다.** 별도로 기획했던 Secretary는 독립 프로젝트로 개발하지 않으며, 설계와 기능을 기존 workMate에 흡수한다. 기존 workMate 구현을 기준으로 중복 기능을 만들지 않는다.

- **기존 구현 유지:** Electron + React + FastAPI, OpenAI Agent / Tool Calling, Task, Google Calendar, 승인 Action Card.
- **Secretary에서 흡수할 기능:** macOS 메뉴바·Windows 트레이 상주, 전역 단축키, OS 알림, 아침 브리핑, 선제적 업무 알림·리마인더, Bizmeka 메일·일정·주소록 연동, 메일 초안·발송 승인 흐름.
- **WorkMemory:** 독립 프로젝트와 데이터 저장소를 유지하고 workMate의 Memory Tool에서 API로 조회한다. 업무자료 검색·RAG 기능을 workMate에 중복 구현하지 않는다.
- **개발 원칙:** 기능별 구현 상태를 비교해 미구현 항목만 추가한다. 기존 코드·데이터·동작을 보존하며 외부 상태를 바꾸는 작업은 사용자 승인 후 실행한다.
- **명칭 정책:** 향후 제품/UI/문서에서는 workMate만 사용하며 Secretary 명칭은 통합 이력 설명에만 남긴다.

### 통합 후 역할

**workMate = 업무를 수행하고 먼저 챙겨주는 개인 AI 비서**  

**WorkMemory = 업무 기록과 근거 자료를 보관·검색하는 독립 기억 엔진**

### 다음 개발 우선순위

1. 기존 workMate 구현 상태와 통합 기능 갭 확인
2. 상주 UI·단축키·알림 및 Task/Reminder 연결
3. 아침 브리핑·선제 알림
4. Bizmeka 메일/일정 연동과 승인 흐름
5. WorkMemory API 연동

---

> 
**목적:** workMate를 Electron + React + FastAPI 기반의 개인용 AI 업무비서로 개발한다. 이 문서는 설계 목표와 과거 구현 보고를 함께 기록한 문서이며, 현재 소스 기준 구현·실행 검증을 보증하지 않는다.

## 최신 구현 현황 (2026-11-03, 소스 기준)

> 
**검증 범위:** pytest 107개·npm build·lint 통과. 실제 Electron 창 조작, 실제 Bizmeka 서버, 새 말투의 실제 OpenAI 응답은 **아직 검증하지 않았다**. 아래 과거 보고의 Mail Mock·Mock 중심 항목은 이 섹션이 최신이다.

### 추가 구현

- **업무·메모·지식베이스:** Task CRUD, 독립 Memo, 지식베이스 레이아웃 개편, 홈을 실제 업무 데이터에 연결
- **메이트(구 Assistant) Tool Calling:** 업무·메모 생성/수정/완료/삭제는 propose\_\* 도구로 승인 카드만 생성, 승인 후 정확히 1회 실행. 대상이 여러 개면 임의 선택 없이 되묻기
- **Google Calendar 실연동:** OAuth(루프백+PKCE), 토큰은 OS 키체인, 일정 CRUD 승인 카드, 실제 캘린더 테스트 완료. 반복 일정 변경/삭제는 미지원
- **대화 기록 영속화:** SQLite 저장, 메뉴 이동·재시작 후 복원, 새 대화/삭제, 실행 불가 카드는 만료 처리, 키·토큰은 마스킹
- **Bizmeka 사내 메일:** Provider none(기본)/mock/bizmeka 환경변수 선택. 메일 원문은 LLM에 전달하지 않고(건수·상태만) 화면에만 표시, 대화 DB·로그·활동 기록에 저장 안 함. 발송·삭제(휴지통 이동만)·이동·읽음 변경은 승인 카드 필수, 발송 타임아웃 시 자동 재발송 없이 '확인 필요'. WORKMATE\_MAIL\_ALLOW\_LIVE=1 이 아니면 실서버 요청 자체를 만들지 않음
- **업무 기간 설정:** start\_at 추가(기존 due\_at은 종료일/마감으로 유지), 둘 다 선택, 시작일 ≤ 종료일 검증, 컬럼 추가 방식 마이그레이션으로 기존 데이터 보존. 업무 화면에서 개별 입력·삭제, 목록에 기간 표시. 메이트는 말하지 않은 날짜를 만들지 않음
- **UI·말투:** 메뉴 명칭 Assistant → 메이트, 대화 스타일을 개인 비서형 해요체로 변경, Electron 레이아웃 깨짐 CSS 수정

### 현재 확인된 한계

- Bizmeka 인증 방식(로그인/토큰 API 없음), 경로 오타(moblieApi) 여부, 요청 형식, 발송 멱등성 등 사내 IT 확인 필요 (docs/[bizmeka-mail-setup.md](http://bizmeka-mail-setup.md)의 체크리스트)
- 메일 전달 시 원본 첨부 자동 첨부 없음, 예약발송 서버 동작 미확인
- 현재 .env의 WORKMATE\_MAIL\_PROVIDER=mock 이면 Mock 샘플 메일이 표시됨 (숨기려면 none)

### 갱신된 다음 작업

1. Bizmeka 실연동 전 사내 IT 확인 → 읽기 전용 호출부터 단계적 검증
2. 새 대화 스타일·업무 기간 입력을 실제 OpenAI/Electron 화면에서 검증
3. 상주 UI·단축키·알림, 아침 브리핑·선제 알림 (기존 우선순위 2\~3번)
4. WorkMemory API 연동
5. 운영 배포용 credential vault·Python runtime packaging

## 구현 상태 표기 안내

**주의:** 아래 상태표와 테스트·빌드 기록은 당시 개발 작업 보고를 토대로 작성된 이력이며, 현재 저장소에서 직접 재검증한 결과가 아니다. `완료`로 표기된 항목도 현시점에서는 **과거 구현 보고(현재 미검증)** 로 읽어야 한다. Secretary에서 통합하기로 한 기능은 별도 검증 전까지 **설계·계획** 상태다.

## 과거 구현 보고 (현재 상태 미검증)

| 영역 | 상태 | 구현 내용 |
| --- | --- | --- |
| Desktop UI | 완료 | Electron, React, Home·Assistant·업무·활동 기록·설정 화면 |
| Backend | 완료 | FastAPI, SQLite 업무 저장, Dashboard·Chat API |
| LLM Agent | 구현 완료 | OpenAI Responses API Provider와 Mock Provider를 환경변수로 전환 |
| Tool Calling | 구현 완료 | 일정·업무·메일·메모리 Tool Registry, 다중 Tool Call, 대화 문맥 |
| 일반 대화 | 구현 완료 | Tool 없이 LLM 텍스트 응답을 반환하도록 Responses output parsing 보완 |
| Calendar | V0.3 구현 완료 | Mock / Google Provider 전환, OAuth Desktop 연결·토큰 갱신·일정 조회/검색/생성/수정/삭제 |
| 승인 흐름 | 구현 완료 | Calendar 변경은 Action Card 승인 후 실행, 중복 승인 방지 |
| Mail | 구현 완료 (Mock 기준) | 2026-11-03 갱신: Provider 구조·Mail 화면·승인 카드 구현. 실제 Bizmeka 서버는 미검증 (아래 최신 현황 참고) |
| Memory | Mock | 실제 WorkMemory 연동은 후속 단계 |

## 과거 작업 보고 (현재 재검증 필요)

### V0.2 — OpenAI Agent

- OpenAI Responses API의 일반 텍스트 응답과 function call 응답을 모두 처리하도록 Provider parsing 수정
- Tool이 없는 인사·일반 대화는 LLM이 직접 답변
- 날씨처럼 실시간 데이터가 필요한데 Tool이 없으면 지어내지 않고 연결되지 않았다고 안내
- Electron 터미널에 provider, 선택 Tool, 성공 여부, 실행 시간을 안전하게 출력
- API key, 프롬프트 원문, OAuth token, 메일 본문은 로그에 기록하지 않음

### V0.3 — Google Calendar

- `WORKMATE_CALENDAR_PROVIDER=mock|google` 전환
- `GoogleCalendarProvider`와 `MockCalendarProvider`를 공통 Provider 경계 뒤로 분리
- Google OAuth Desktop local-server flow 구현
- refresh token을 로컬 사용자 전용 권한 파일로 저장하고 Git 제외
- `Asia/Seoul` 기준 오늘·내일 일정 조회, 검색, 종일 일정, 반복 일정 occurrence 정규화
- Agent는 캘린더에 즉시 쓰지 않고 Action Card만 제안
- 사용자 승인 후에만 create/update/delete 실행
- Settings에서 상태 조회, Google 계정 연결, 연결 확인, 로컬 token 연결 해제 제공
- Home과 NEXT UP이 선택 Calendar Provider 결과를 사용하도록 반영

## 과거 검증 보고 (현재 재검증 필요)

- Backend test: **11 passed**
- TypeScript typecheck: 통과
- ESLint: 통과
- Production build: 통과
- Google Provider 미설정 상태에서 앱이 중단되지 않고 “연결되지 않음”으로 표시되는 것 확인
- Electron에서 새 Renderer UI와 Settings 화면 로딩 확인

## 과거 개발 실행 기록 (현재 재검증 필요)

기록 기준일: 2026-10-08

- `npm run dev` 실행 성공
- Vite: `http://127.0.0.1:5173`
- FastAPI: `http://127.0.0.1:8765`
- Health와 Dashboard API: 200 OK
- 현재 로그에는 Chat 요청이 없으므로 OpenAI 또는 Mock Provider 선택 로그는 아직 없음

## 바로 해야 할 일

- [ ] 프로젝트 루트에 `.env` 생성
  - `/bin/cp "/Users/bizntech/Desktop/github/workMate/.env.example" "/Users/bizntech/Desktop/github/workMate/.env"`
- [ ] `.env`에 실제 OpenAI 설정 입력
  - `WORKMATE_LLM_PROVIDER=openai`
  - `OPENAI_API_KEY=...`
  - `WORKMATE_LLM_MODEL=gpt-4.1-mini`
- [ ] Electron을 완전히 종료한 뒤 `npm run dev` 재실행
- [ ] Assistant에서 “안녕 너는 누구야”를 보내고 로그의 `agent_request provider=openai` 확인
- [ ] 업무 질문을 보내 `agent_tools_selected provider=openai` 및 Tool 결과 확인

## Google Calendar 실계정 연결 절차

- [ ] Google Cloud Console에서 Google Calendar API 활성화
- [ ] OAuth consent screen 구성
- [ ] **Desktop app** 타입 OAuth Client 생성
- [ ] Client JSON을 저장소 밖 안전한 위치에 보관
- [ ] `.env`에 다음 값 추가
  - `WORKMATE_CALENDAR_PROVIDER=google`
  - `GOOGLE_CALENDAR_CREDENTIALS_PATH=/절대경로/google-oauth-desktop-client.json`
  - `GOOGLE_CALENDAR_ID=primary`
- [ ] 앱 재시작 → 설정 → Google 계정 연결
- [ ] Google 로그인/동의 후 오늘 일정이 Home에 표시되는지 확인
- [ ] “내일 오후 2시에 workMate 테스트 회의 등록해” → Action Card 승인 → Google/Samsung Calendar 동기화 확인

## 보안 메모

- `.env`, OAuth Client JSON, Calendar refresh token은 Git에 포함하지 않는다.
- 현재 token 저장은 개발 단계의 `FileTokenStore`이며 파일 권한은 사용자 전용이다.
- 정식 배포 시 같은 추상화 뒤에 macOS Keychain / Windows Credential Manager 저장소를 구현한다.
- “연결 해제”는 이 기기의 로컬 token만 삭제한다. Google 계정의 OAuth grant revoke는 별도 동작으로 유지한다.

## 후속 로드맵

1. OpenAI 실계정 통신 및 Tool Calling 실사용 검증
2. Google Calendar 실계정 CRUD와 Samsung Calendar 동기화 검증
3. Bizmeka 실제 Mail API 연동
4. WorkMemory 실제 API 연동
5. 운영 배포용 OS credential vault 및 Python runtime packaging

