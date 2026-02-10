# SNS 크롤링 + 멀티 AI 브레인스토밍 n8n 워크플로우

매일 오전 9시에 Google Sheets에서 키워드를 읽어 X(Twitter)와 네이버 카페를 크롤링하고,
Claude · Gemini · ChatGPT에게 사용자 니즈 분석 및 앱 아이디어를 브레인스토밍시켜 Notion에 저장하는 자동화 워크플로우.

---

## 전체 흐름

```
Schedule (09:00 KST)
  └→ Google Sheets (키워드 목록 읽기)
       └→ Code (키워드 아이템 분리)
            ├→ Twitter API  ─┐
            └→ Naver Cafe API ┴→ Merge (크롤링 결과 합산)
                                    └→ Code (데이터 정제 + AI 프롬프트 생성)
                                         ├→ Claude  ─┐
                                         ├→ Gemini  ─┤→ Merge → Code (리포트 생성) → Notion
                                         └→ ChatGPT ─┘

Error Trigger → 이메일 알림
```

---

## 노드 목록 (15개)

| # | 노드 이름 | 역할 |
|---|-----------|------|
| 1 | Daily Crawl Schedule | 매일 09:00 KST 트리거 |
| 2 | Get Keywords | Google Sheets에서 키워드 읽기 |
| 3 | Build Keyword Items | 활성 키워드만 필터링·분리 |
| 4 | Crawl Twitter | Twitter API v2 검색 |
| 5 | Crawl Naver Cafe | 네이버 카페 검색 API |
| 6 | Merge Crawl Results | 크롤링 결과 병합 |
| 7 | Clean Data & Build Prompt | 데이터 정제 + AI 프롬프트 생성 |
| 8 | Ask Claude | Anthropic claude-opus-4-5 호출 |
| 9 | Ask Gemini | Google gemini-1.5-pro 호출 |
| 10 | Ask ChatGPT | OpenAI gpt-4o 호출 |
| 11 | Merge AI Responses | AI 3개 응답 병합 |
| 12 | Generate Final Report | 응답 파싱 + 리포트 구성 |
| 13 | Save to Notion | Notion 페이지 생성 |
| 14 | Error Trigger | 워크플로우 오류 감지 |
| 15 | Notify Error | 오류 이메일 발송 |

---

## 사전 준비 체크리스트

### 필요한 API 키 / 계정

| 서비스 | 발급 위치 | 비용 |
|--------|-----------|------|
| X (Twitter) Bearer Token | https://developer.twitter.com | 무료 (월 1,500 트윗) |
| Naver Client ID/Secret | https://developers.naver.com | 무료 (일 25,000회) |
| Anthropic API Key | https://console.anthropic.com | 사용량 기반 유료 |
| Google AI API Key | https://aistudio.google.com | 무료 티어 있음 |
| OpenAI API Key | https://platform.openai.com | 사용량 기반 유료 ($5 선불) |
| Google Sheets | https://console.cloud.google.com | 무료 (OAuth2 설정) |
| Notion | https://www.notion.so/my-integrations | 무료 |

---

## STEP 1: API 키 발급 방법

### Claude (Anthropic)
1. https://console.anthropic.com 접속 → 회원가입
2. 좌측 메뉴 **API Keys** → **Create Key**
3. 키 복사 후 안전한 곳에 저장 (한 번만 표시됨)

### Gemini (Google AI Studio)
1. https://aistudio.google.com 접속 → Google 계정 로그인
2. 좌측 **Get API Key** 클릭
3. **Create API Key** → 복사

### ChatGPT (OpenAI)
1. https://platform.openai.com 접속 → 회원가입
2. 우측 상단 프로필 아이콘 → **API Keys**
3. **Create new secret key** → 복사
4. **Billing** 메뉴에서 최소 $5 크레딧 충전 필요

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
  - N8N_BASIC_AUTH_ACTIVE=true
  - N8N_BASIC_AUTH_USER=user
  - N8N_BASIC_AUTH_PASSWORD=password
  # 아래 항목들 추가
  - NAVER_CLIENT_ID=발급받은_Client_ID
  - NAVER_CLIENT_SECRET=발급받은_Client_Secret
  - ANTHROPIC_API_KEY=sk-ant-발급받은키
  - GOOGLE_AI_API_KEY=AIza발급받은키
  - OPENAI_API_KEY=sk-발급받은키
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

> Notion Integration 생성 방법:
> 1. https://www.notion.so/my-integrations → **New integration**
> 2. 이름 입력 후 생성 → **Internal Integration Token** 복사
> 3. 저장할 Notion 데이터베이스 열기 → 우측 상단 `...` → **Connections** → 생성한 Integration 추가
> 4. 데이터베이스 URL에서 ID 복사:
>    ```
>    https://notion.so/[워크스페이스]/[이것이_DATABASE_ID]?v=...
>    ```
>    (32자리 hex 문자열, 하이픈 포함 또는 미포함)

---

## STEP 5: 워크플로우에 크레덴셜 연결

워크플로우를 열고 각 노드에서 크레덴셜 지정:

1. **Get Keywords** 노드 클릭 → Credential 드롭다운에서 `Google Sheets` 선택
2. `documentId` 필드에 복사한 Google Sheets ID 입력 (`YOUR_GOOGLE_SHEET_ID` 교체)
3. **Crawl Twitter** 노드 클릭 → Credential에서 `Twitter Bearer Token` 선택
4. **Save to Notion** 노드 클릭 → Credential에서 `Notion API` 선택

---

## STEP 6: 워크플로우 활성화

모든 설정 완료 후 워크플로우 우측 상단 토글을 켜면 매일 오전 9시에 자동 실행됩니다.

수동 테스트: 워크플로우 에디터에서 **Test workflow** 버튼 클릭

---

## 워크플로우 Import JSON

> n8n UI에서 **Workflows → Import from JSON**으로 임포트 가능합니다.
> 임포트 후 크레덴셜 연결 및 Google Sheets ID만 수정하면 됩니다.

```json
{
  "name": "SNS 크롤링 + 멀티 AI 브레인스토밍",
  "nodes": [
    {
      "id": "node-schedule",
      "name": "Daily Crawl Schedule",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [100, 300],
      "parameters": {
        "rule": {
          "interval": [{"field": "cronExpression", "expression": "0 9 * * *"}]
        }
      }
    },
    {
      "id": "node-sheets",
      "name": "Get Keywords",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4.4,
      "position": [300, 300],
      "parameters": {
        "operation": "read",
        "documentId": "YOUR_GOOGLE_SHEET_ID",
        "sheetName": "Keywords",
        "dataStartRow": 2,
        "keyRow": 1
      },
      "credentials": {
        "googleSheetsOAuth2Api": {"id": "1", "name": "Google Sheets"}
      }
    },
    {
      "id": "node-split",
      "name": "Build Keyword Items",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [500, 300],
      "parameters": {
        "jsCode": "const items = [];\nfor (const item of $input.all()) {\n  const keyword = item.json.keyword;\n  const active = item.json.active;\n  if (keyword && String(active).toUpperCase() !== 'FALSE') {\n    items.push({ json: { keyword, timestamp: new Date().toISOString() } });\n  }\n}\nreturn items;"
      }
    },
    {
      "id": "node-twitter",
      "name": "Crawl Twitter",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [700, 200],
      "parameters": {
        "method": "GET",
        "url": "https://api.twitter.com/2/tweets/search/recent",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpBearerAuth",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {"name": "query", "value": "={{ $json.keyword }} lang:ko -is:retweet"},
            {"name": "max_results", "value": "10"},
            {"name": "tweet.fields", "value": "created_at,public_metrics,text"}
          ]
        },
        "continueOnFail": true
      },
      "credentials": {
        "httpBearerAuth": {"id": "2", "name": "Twitter Bearer Token"}
      }
    },
    {
      "id": "node-naver",
      "name": "Crawl Naver Cafe",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [700, 400],
      "parameters": {
        "method": "GET",
        "url": "https://openapi.naver.com/v1/search/cafearticle.json",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [
            {"name": "query", "value": "={{ $json.keyword }}"},
            {"name": "display", "value": "30"},
            {"name": "sort", "value": "date"}
          ]
        },
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {"name": "X-Naver-Client-Id", "value": "={{ $env.NAVER_CLIENT_ID }}"},
            {"name": "X-Naver-Client-Secret", "value": "={{ $env.NAVER_CLIENT_SECRET }}"}
          ]
        },
        "continueOnFail": true
      }
    },
    {
      "id": "node-merge-crawl",
      "name": "Merge Crawl Results",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 3,
      "position": [900, 300],
      "parameters": {"mode": "combine", "combinationMode": "multiplex"}
    },
    {
      "id": "node-clean",
      "name": "Clean Data & Build Prompt",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1100, 300],
      "parameters": {
        "jsCode": "const today = new Date().toISOString().split('T')[0];\nconst posts = [];\n\nfor (const item of $input.all()) {\n  const json = item.json;\n\n  if (json.data && Array.isArray(json.data)) {\n    for (const tweet of json.data) {\n      if (tweet.text && tweet.text.length > 30) {\n        posts.push({ source: 'Twitter', text: tweet.text.replace(/\\n/g, ' ') });\n      }\n    }\n  }\n\n  if (json.items && Array.isArray(json.items)) {\n    for (const article of json.items) {\n      const text = article.description?.replace(/<[^>]*>/g, '') || '';\n      const title = article.title?.replace(/<[^>]*>/g, '') || '';\n      if (text.length > 20 || title.length > 10) {\n        posts.push({ source: 'NaverCafe', text: `${title}: ${text}` });\n      }\n    }\n  }\n}\n\nconst seen = new Set();\nconst uniquePosts = posts.filter(p => {\n  const key = p.text.substring(0, 50);\n  if (seen.has(key)) return false;\n  seen.add(key);\n  return true;\n});\n\nconst formattedPosts = uniquePosts\n  .slice(0, 150)\n  .map(p => `[${p.source}] ${p.text}`)\n  .join('\\n---\\n');\n\nconst prompt = `다음은 ${today}에 수집한 SNS/카페 데이터입니다.\\n이 데이터를 분석하여 아래 형식으로 응답해주세요:\\n\\n## 1. 핵심 사용자 니즈 TOP 5\\n각 니즈에 대해 근거가 되는 게시물을 1-2개 인용\\n\\n## 2. 반복되는 불편사항 패턴\\n자주 언급되는 문제점 3가지\\n\\n## 3. 추천 앱 아이디어 3개\\n각 아이디어에 대해:\\n- 해결하는 문제\\n- 핵심 기능 3가지\\n- 타겟 사용자\\n- 시장 기회 (상/중/하)\\n\\n## 4. 종합 평가\\n가장 주목할만한 기회와 그 이유 (100자 내)\\n\\n---\\n수집 데이터 (${uniquePosts.length}개 게시물):\\n${formattedPosts}`.trim();\n\nreturn [{ json: { prompt, postCount: uniquePosts.length, date: today } }];"
      }
    },
    {
      "id": "node-claude",
      "name": "Ask Claude",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1300, 200],
      "parameters": {
        "method": "POST",
        "url": "https://api.anthropic.com/v1/messages",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {"name": "x-api-key", "value": "={{ $env.ANTHROPIC_API_KEY }}"},
            {"name": "anthropic-version", "value": "2023-06-01"},
            {"name": "content-type", "value": "application/json"}
          ]
        },
        "sendBody": true,
        "contentType": "json",
        "jsonBody": "={{ JSON.stringify({model: 'claude-opus-4-5-20251101', max_tokens: 2000, messages: [{role: 'user', content: $json.prompt}]}) }}"
      }
    },
    {
      "id": "node-gemini",
      "name": "Ask Gemini",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1300, 350],
      "parameters": {
        "method": "POST",
        "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent",
        "sendQuery": true,
        "queryParameters": {
          "parameters": [{"name": "key", "value": "={{ $env.GOOGLE_AI_API_KEY }}"}]
        },
        "sendBody": true,
        "contentType": "json",
        "jsonBody": "={{ JSON.stringify({contents: [{parts: [{text: $json.prompt}]}], generationConfig: {maxOutputTokens: 2000}}) }}"
      }
    },
    {
      "id": "node-chatgpt",
      "name": "Ask ChatGPT",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1300, 500],
      "parameters": {
        "method": "POST",
        "url": "https://api.openai.com/v1/chat/completions",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {"name": "Authorization", "value": "=Bearer {{ $env.OPENAI_API_KEY }}"},
            {"name": "Content-Type", "value": "application/json"}
          ]
        },
        "sendBody": true,
        "contentType": "json",
        "jsonBody": "={{ JSON.stringify({model: 'gpt-4o', max_tokens: 2000, messages: [{role: 'user', content: $json.prompt}]}) }}"
      }
    },
    {
      "id": "node-merge-ai",
      "name": "Merge AI Responses",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 3,
      "position": [1500, 350],
      "parameters": {"mode": "combine", "combinationMode": "multiplex"}
    },
    {
      "id": "node-report",
      "name": "Generate Final Report",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1700, 350],
      "parameters": {
        "jsCode": "const items = $input.all();\nconst date = new Date().toISOString().split('T')[0];\n\nfunction extractResponse(item) {\n  const json = item.json;\n  if (json.content?.[0]?.text) return json.content[0].text;\n  if (json.candidates?.[0]?.content?.parts?.[0]?.text) {\n    return json.candidates[0].content.parts[0].text;\n  }\n  if (json.choices?.[0]?.message?.content) {\n    return json.choices[0].message.content;\n  }\n  return '응답 없음';\n}\n\nconst responses = {\n  claude: items[0] ? extractResponse(items[0]) : '응답 없음',\n  gemini: items[1] ? extractResponse(items[1]) : '응답 없음',\n  chatgpt: items[2] ? extractResponse(items[2]) : '응답 없음',\n};\n\nreturn [{\n  json: {\n    date,\n    title: `앱 아이디어 브레인스토밍 - ${date}`,\n    claude: responses.claude,\n    gemini: responses.gemini,\n    chatgpt: responses.chatgpt,\n  }\n}];"
      }
    },
    {
      "id": "node-notion",
      "name": "Save to Notion",
      "type": "n8n-nodes-base.notion",
      "typeVersion": 2.2,
      "position": [1900, 350],
      "parameters": {
        "resource": "page",
        "operation": "create",
        "databaseId": "={{ $env.NOTION_DATABASE_ID }}",
        "title": "={{ $json.title }}",
        "propertiesUi": {
          "propertyValues": [
            {"key": "Date", "type": "date", "dateValue": "={{ $json.date }}"},
            {"key": "Status", "type": "select", "selectValue": "New"}
          ]
        },
        "blockUi": {
          "blockValues": [
            {"type": "heading_2", "textContent": "Claude 분석"},
            {"type": "paragraph", "textContent": "={{ $json.claude }}"},
            {"type": "heading_2", "textContent": "Gemini 분석"},
            {"type": "paragraph", "textContent": "={{ $json.gemini }}"},
            {"type": "heading_2", "textContent": "ChatGPT 분석"},
            {"type": "paragraph", "textContent": "={{ $json.chatgpt }}"}
          ]
        }
      },
      "credentials": {
        "notionApi": {"id": "3", "name": "Notion API"}
      }
    },
    {
      "id": "node-error",
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [100, 600],
      "parameters": {}
    },
    {
      "id": "node-notify",
      "name": "Notify Error",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [300, 600],
      "parameters": {
        "toEmail": "={{ $env.NOTIFY_EMAIL }}",
        "subject": "=[n8n] 브레인스토밍 워크플로우 오류 - {{ $json.execution.error.message }}",
        "text": "=워크플로우: {{ $json.workflow.name }}\n오류: {{ $json.execution.error.message }}\n시간: {{ $json.execution.startedAt }}"
      }
    }
  ],
  "connections": {
    "Daily Crawl Schedule": {"main": [[{"node": "Get Keywords", "type": "main", "index": 0}]]},
    "Get Keywords": {"main": [[{"node": "Build Keyword Items", "type": "main", "index": 0}]]},
    "Build Keyword Items": {"main": [[{"node": "Crawl Twitter", "type": "main", "index": 0}, {"node": "Crawl Naver Cafe", "type": "main", "index": 0}]]},
    "Crawl Twitter": {"main": [[{"node": "Merge Crawl Results", "type": "main", "index": 0}]]},
    "Crawl Naver Cafe": {"main": [[{"node": "Merge Crawl Results", "type": "main", "index": 1}]]},
    "Merge Crawl Results": {"main": [[{"node": "Clean Data & Build Prompt", "type": "main", "index": 0}]]},
    "Clean Data & Build Prompt": {"main": [[{"node": "Ask Claude", "type": "main", "index": 0}, {"node": "Ask Gemini", "type": "main", "index": 0}, {"node": "Ask ChatGPT", "type": "main", "index": 0}]]},
    "Ask Claude": {"main": [[{"node": "Merge AI Responses", "type": "main", "index": 0}]]},
    "Ask Gemini": {"main": [[{"node": "Merge AI Responses", "type": "main", "index": 1}]]},
    "Ask ChatGPT": {"main": [[{"node": "Merge AI Responses", "type": "main", "index": 2}]]},
    "Merge AI Responses": {"main": [[{"node": "Generate Final Report", "type": "main", "index": 0}]]},
    "Generate Final Report": {"main": [[{"node": "Save to Notion", "type": "main", "index": 0}]]},
    "Error Trigger": {"main": [[{"node": "Notify Error", "type": "main", "index": 0}]]}
  },
  "settings": {
    "timezone": "Asia/Seoul",
    "callerPolicy": "workflowsFromSameOwner",
    "availableInMCP": false
  },
  "pinData": {}
}
```

---

## 임포트 후 수정 필요 항목

임포트하면 크레덴셜 ID가 초기화됩니다. 아래 노드에서 크레덴셜을 다시 선택해야 합니다:

1. **Get Keywords** → Google Sheets 크레덴셜 선택 + `documentId` 에 실제 시트 ID 입력
2. **Crawl Twitter** → Twitter Bearer Token 크레덴셜 선택
3. **Save to Notion** → Notion API 크레덴셜 선택

---

## API 사용량 및 비용 참고

| 서비스 | 무료 한도 | 초과 시 |
|--------|-----------|---------|
| Twitter API (Free) | 월 1,500 트윗 읽기 | Basic 플랜 $100/월 |
| Naver 검색 API | 일 25,000회 | 무료 초과 없음 (제한만 있음) |
| Claude claude-opus-4-5 | 없음 (유료) | 입력 $15 / 출력 $75 (1M 토큰) |
| Gemini 1.5 Pro | 분당 2회, 일 50회 | 초과 시 유료 전환 |
| GPT-4o | 없음 (유료) | ~$2.50 / 1M 입력 토큰 |

> 키워드 3개 기준 일 1회 실행 시 AI 비용 약 $0.05~0.15 예상
