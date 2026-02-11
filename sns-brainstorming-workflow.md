# SNS 크롤링 + AI 브레인스토밍 n8n 워크플로우

매일 오전 9시에 Google Sheets에서 키워드를 읽어 X(Twitter)와 네이버 카페를 크롤링하고,
Gemini(무료)로 데이터를 정리해 Notion에 저장합니다.
저장된 요약본을 Claude.ai / ChatGPT 채팅창에 붙여넣어 직접 브레인스토밍합니다.

---

## 전체 흐름

```
Schedule (09:00 KST)
  └→ Google Sheets (키워드 목록 읽기)
       └→ Code (키워드 아이템 분리)
            ├→ Twitter API   ─┐
            └→ Naver Cafe API ┴→ Merge (크롤링 결과 합산)
                                    └→ Code (Gemini용 프롬프트 생성)
                                         └→ Gemini Flash (무료, 데이터 정리)
                                              └→ Code (응답 파싱)
                                                   └→ Notion 저장
                                                        └→ 사용자가 직접 Claude.ai / ChatGPT에 붙여넣기

Error Trigger → 이메일 알림
```

---

## 노드 목록 (12개)

| # | 노드 이름 | 역할 |
|---|-----------|------|
| 1 | Daily Crawl Schedule | 매일 09:00 KST 트리거 |
| 2 | Get Keywords | Google Sheets에서 키워드 읽기 |
| 3 | Build Keyword Items | 활성 키워드만 필터링·분리 |
| 4 | Crawl Twitter | Twitter API v2 검색 |
| 5 | Crawl Naver Cafe | 네이버 카페 검색 API |
| 6 | Merge Crawl Results | 크롤링 결과 병합 |
| 7 | Format Data for Gemini | 데이터 정제 + 정리 프롬프트 생성 |
| 8 | Ask Gemini | Gemini 1.5 Flash로 구조화 요약 |
| 9 | Extract Summary | Gemini 응답 파싱 + Notion용 구조화 |
| 10 | Save to Notion | Notion 페이지 생성 |
| 11 | Error Trigger | 워크플로우 오류 감지 |
| 12 | Notify Error | 오류 이메일 발송 |

---

## Notion 저장 구조

매일 아래 형식으로 페이지가 생성됩니다:

```
제목: SNS 크롤링 요약 - 2026-02-11

## Gemini 정리 요약
  주요 언급 주제 TOP 5
  자주 등장한 표현/키워드
  불편함/문제 제기 모음

---

## AI 채팅용 요약본  ← 이 부분을 복사해서 Claude.ai / ChatGPT에 붙여넣기
  "다음은 오늘 수집한 SNS 데이터입니다. 앱 아이디어를 브레인스토밍해주세요: ..."
```

---

## 사전 준비 체크리스트

### 필요한 API 키 / 계정

| 서비스 | 발급 위치 | 비용 |
|--------|-----------|------|
| X (Twitter) Bearer Token | https://developer.twitter.com | 무료 (월 1,500 트윗) |
| Naver Client ID/Secret | https://developers.naver.com | 무료 (일 25,000회) |
| Google AI API Key (Gemini) | https://aistudio.google.com | **무료** (일 50회) |
| Google Sheets | https://console.cloud.google.com | 무료 (OAuth2 설정) |
| Notion | https://www.notion.so/my-integrations | 무료 |

---

## STEP 1: API 키 발급 방법

### Gemini (Google AI Studio)
1. https://aistudio.google.com 접속 → Google 계정 로그인
2. 좌측 **Get API Key** 클릭
3. **Create API Key** → 복사

### Naver Open API
1. https://developers.naver.com 접속 → 네이버 로그인
2. **Application → 애플리케이션 등록** 클릭
3. 앱 이름 입력, 사용 API에서 **검색** 체크
4. 등록 완료 후 **Client ID**와 **Client Secret** 복사

### Twitter/X API
1. https://developer.x.com 접속 → 회원가입 (영문 작성)
2. **Free 플랜** 선택 → 새 앱 생성
3. 앱 생성 후 **Keys and tokens** 탭 → **Bearer Token** 복사

---

## STEP 2: Google Sheets 준비

1. Google Sheets에서 새 스프레드시트 생성
2. 시트 이름을 `Keywords`로 변경
3. 아래 형식으로 데이터 입력:

   | A열 (keyword) | B열 (active) |
   |--------------|-------------|
   | keyword      | active      |
   | 앱 추천       | TRUE        |
   | 이런 앱 없나   | TRUE        |
   | 불편한 점      | TRUE        |
   | 개발 요청      | FALSE       |

   - 1행: 헤더 (`keyword`, `active`)
   - 2행부터: 실제 키워드
   - `active=FALSE`이면 해당 키워드 건너뜀

4. 스프레드시트 URL에서 ID 복사:
   ```
   https://docs.google.com/spreadsheets/d/[여기가 ID]/edit
   ```

---

## STEP 3: n8n 환경변수 설정

`docker-compose.yml` 파일을 열어 `environment` 섹션에 추가:

```yaml
environment:
  - GENERIC_TIMEZONE=Asia/Seoul
  - TZ=Asia/Seoul
  - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
  - N8N_RUNNERS_ENABLED=true
  - NODES_EXCLUDE=[]
  # 아래 항목들 추가
  - NAVER_CLIENT_ID=발급받은_Client_ID
  - NAVER_CLIENT_SECRET=발급받은_Client_Secret
  - GOOGLE_AI_API_KEY=AIza발급받은키
  - NOTION_DATABASE_ID=발급받을_데이터베이스_ID
  - NOTIFY_EMAIL=오류알림받을@이메일.com
```

수정 후 Docker 재시작:
```bash
docker-compose down && docker-compose up -d
```

---

## STEP 4: n8n 크레덴셜 등록

n8n UI (http://localhost:5678) → 좌측 메뉴 **Credentials** → **Add credential**

### Google Sheets OAuth2
1. `Google Sheets OAuth2 API` 검색 후 선택
2. **Connect my account** 클릭 → Google 계정 연동
3. 크레덴셜 이름: `Google Sheets` 로 저장

> Google Cloud Console에서 OAuth2 Client ID 발급이 필요합니다:
> 1. https://console.cloud.google.com → 새 프로젝트 생성
> 2. **API 및 서비스 → 사용 설정된 API** → Google Sheets API 활성화
> 3. **사용자 인증 정보 → OAuth 2.0 클라이언트 ID** 생성
> 4. n8n 리디렉션 URI 등록: `http://localhost:5678/rest/oauth2-credential/callback`

### Twitter Bearer Token
1. `HTTP Bearer Auth` 검색 후 선택
2. **Token** 필드에 Twitter Bearer Token 붙여넣기
3. 크레덴셜 이름: `Twitter Bearer Token` 으로 저장

### Notion API
1. `Notion API` 검색 후 선택
2. **Internal Integration Token** 필드에 토큰 붙여넣기
3. 크레덴셜 이름: `Notion API` 로 저장

> Notion Integration 생성 방법:
> 1. https://www.notion.so/my-integrations → **New integration**
> 2. 이름 입력 후 생성 → **Internal Integration Token** 복사
> 3. 저장할 Notion 데이터베이스 열기 → 우측 상단 `...` → **Connections** → 생성한 Integration 추가
> 4. 데이터베이스 URL에서 ID 복사:
>    ```
>    https://notion.so/[워크스페이스]/[이것이_DATABASE_ID]?v=...
>    ```

---

## STEP 5: 워크플로우에 크레덴셜 연결

워크플로우(http://localhost:5678/workflow/Iwj7UbByHCFusIT8)를 열고 각 노드에서 크레덴셜 지정:

| 노드 | 선택할 크레덴셜 | 추가 작업 |
|------|--------------|---------|
| **Get Keywords** | `Google Sheets` | `documentId` 에 실제 Google Sheets ID 입력 |
| **Crawl Twitter** | `Twitter Bearer Token` | - |
| **Save to Notion** | `Notion API` | - |

---

## STEP 6: 워크플로우 활성화

모든 설정 완료 후 워크플로우 우측 상단 토글을 켜면 매일 오전 9시에 자동 실행됩니다.

수동 테스트: 워크플로우 에디터에서 **Test workflow** 버튼 클릭

---

## AI 오케스트레이션 방안

Notion에 저장된 요약본을 활용해 AI와 더 깊이 협업하는 방법들입니다.

### 방안 1: 수동 복사 붙여넣기 (지금 당장 가능)
Notion 페이지의 **AI 채팅용 요약본** 섹션을 복사해 Claude.ai / ChatGPT에 붙여넣기.
Gemini가 미리 정리된 500자 요약본이 자동 생성되어 있어 바로 사용 가능.

```
[Notion 페이지] → 복사 → [Claude.ai 채팅] → 브레인스토밍
```

**장점:** 무료, 즉시 가능, 대화형으로 심화 질문 가능

---

### 방안 2: Claude Code + Notion MCP (지금 당장 가능)
현재 Claude Code에 Notion MCP가 이미 연결되어 있습니다.
Claude Code에서 직접 명령하면 Notion 페이지를 읽고 분석합니다.

```
"오늘 Notion에 저장된 SNS 크롤링 요약 읽고
앱 아이디어 3개 구체적으로 브레인스토밍해줘"
```

Claude가 자동으로 Notion 페이지를 검색 → 읽기 → 분석 → 응답합니다.

**장점:** 복사 붙여넣기 없음, Claude 구독만으로 무료, 파일로 저장 가능

---

### 방안 3: n8n Webhook + 사용자 트리거 (선택적 자동화)
Notion 페이지 생성 후 n8n이 Webhook URL을 이메일/Slack으로 전송.
사용자가 링크 클릭 시 → AI 심화 분석 실행 → 결과를 같은 Notion 페이지에 추가.

```
Notion 저장 완료
  └→ 이메일: "오늘 요약 준비됨. 심화 분석하려면 클릭"
       └→ [클릭] → n8n Webhook
            └→ Claude API 또는 GPT API 호출
                 └→ 분석 결과를 Notion 페이지에 추가
```

**장점:** 필요할 때만 API 호출 → 비용 최소화

---

### 방안 4: Claude Project 누적 분석 (중기 전략)
Claude.ai의 **Project** 기능에 매주 Notion 요약본을 업로드.
누적 데이터를 기반으로 트렌드 변화, 반복 패턴을 분석.

```
[1주차 요약] ─┐
[2주차 요약] ─┤→ Claude Project → "지난 4주 트렌드 분석해줘"
[3주차 요약] ─┤                    "어떤 니즈가 계속 반복되나?"
[4주차 요약] ─┘
```

**장점:** 단기 스냅샷이 아닌 트렌드 파악 가능, Claude 구독으로 무료

---

### 추천 시작 순서

```
지금 → 방안 1 (수동 복붙) + 방안 2 (Claude Code + Notion MCP)
익숙해지면 → 방안 4 (Claude Project 누적)
API 비용 감수 가능하면 → 방안 3 (Webhook 자동화)
```

---

## API 사용량 및 비용

| 서비스 | 무료 한도 | 비고 |
|--------|-----------|------|
| Twitter API (Free) | 월 1,500 트윗 | 키워드 3개 × 10건 × 30일 = 900건 (여유 있음) |
| Naver 검색 API | 일 25,000회 | 충분 |
| Gemini 1.5 Flash | 일 1,500회, 분당 15회 | 일 1회 실행 시 여유 충분 |
| Google Sheets | 무료 | - |
| Notion | 무료 | - |

**전체 운영 비용: $0 (완전 무료)**
