# Plan: SNS 크롤링 + 멀티 AI 브레인스토밍 n8n 워크플로우

## 목적
매일 정해진 시간에 키워드로 X(Twitter) + 네이버 카페를 크롤링하고,
Claude + Gemini + ChatGPT에게 사용자 니즈 분석 및 앱 아이디어를 브레인스토밍시켜
결과를 Notion 페이지로 저장하는 자동화 워크플로우.

---

## 전체 아키텍처

```
[Phase 1] Schedule → Google Sheets (키워드 로드)
[Phase 2] Code (키워드 → 아이템 분리)
              ↓              ↓
[Phase 3]  Twitter API    Naver Cafe API   (병렬 크롤링)
              ↓              ↓
           Merge Crawl Results
[Phase 4] Code (데이터 정제 + AI 프롬프트 생성)
              ↓         ↓         ↓
[Phase 5]  Claude     Gemini    ChatGPT     (병렬 AI 분석)
              ↓         ↓         ↓
           Merge AI Responses
[Phase 6] Code (리포트 생성) → Notion 페이지 저장
[Error]   Error Trigger → 에러 알림 (Email/Slack)
```

---

## 구현 방식

**워크플로우 생성 방법:** n8n-mcp MCP 툴 사용
```
1. n8n_create_workflow - 빈 워크플로우 생성
2. n8n_update_partial_workflow - 노드 순차 추가
3. n8n_validate_workflow - 검증
4. n8n_test_workflow - 테스트 실행
```

---

## Phase별 노드 상세

### Phase 1: 트리거 & 키워드 로드 (2 nodes)

**Node 1: Schedule Trigger**
```json
{
  "name": "Daily Crawl Schedule",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "parameters": {
    "rule": {
      "interval": [{
        "field": "cronExpression",
        "expression": "0 9 * * *"
      }]
    }
  }
}
```
- 매일 오전 9시 실행 (변경 가능)
- timezone: Asia/Seoul (워크플로우 settings에 설정)

**Node 2: Get Keywords from Google Sheets**
```json
{
  "name": "Get Keywords",
  "type": "n8n-nodes-base.googleSheets",
  "typeVersion": 4.4,
  "parameters": {
    "operation": "read",
    "documentId": "{{GOOGLE_SHEET_ID}}",
    "sheetName": "Keywords",
    "dataStartRow": 2,
    "keyRow": 1
  },
  "credentials": {
    "googleSheetsOAuth2Api": { "id": "{{CRED_ID}}", "name": "Google Sheets" }
  }
}
```
- 스프레드시트 컬럼: `keyword` (A열), `active` (B열, TRUE/FALSE)
- active=TRUE인 키워드만 필터링

---

### Phase 2: 키워드 전처리 (1 node)

**Node 3: Build Keyword Items**
```javascript
// Code Node - JavaScript
// Google Sheets에서 읽은 키워드를 개별 아이템으로 분리
const items = [];
for (const item of $input.all()) {
  const keyword = item.json.keyword;
  const active = item.json.active;
  if (keyword && String(active).toUpperCase() !== 'FALSE') {
    items.push({ json: { keyword, timestamp: new Date().toISOString() } });
  }
}
return items;
```
- 출력: 키워드 개수만큼 개별 아이템 생성
- 이후 노드들은 각 키워드에 대해 한 번씩 실행됨

---

### Phase 3: 병렬 크롤링 (3 nodes)

**Node 4: Crawl X (Twitter API v2)**
```json
{
  "name": "Crawl Twitter",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "GET",
    "url": "https://api.twitter.com/2/tweets/search/recent",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpBearerAuth",
    "sendQuery": true,
    "queryParameters": {
      "parameters": [
        { "name": "query", "value": "={{$json.keyword}} lang:ko -is:retweet" },
        { "name": "max_results", "value": "10" },
        { "name": "tweet.fields", "value": "created_at,public_metrics,text" }
      ]
    },
    "continueOnFail": true
  },
  "credentials": {
    "httpBearerAuth": { "id": "{{CRED_ID}}", "name": "Twitter Bearer Token" }
  }
}
```
- Free Tier: 월 1,500 트윗 읽기 제한 → max_results=10
- continueOnFail: true (API 한도 초과 시에도 워크플로우 계속)
- 한국어 결과만 (lang:ko)

**Node 5: Crawl Naver Cafe**
```json
{
  "name": "Crawl Naver Cafe",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "GET",
    "url": "https://openapi.naver.com/v1/search/cafearticle.json",
    "sendQuery": true,
    "queryParameters": {
      "parameters": [
        { "name": "query", "value": "={{$json.keyword}}" },
        { "name": "display", "value": "30" },
        { "name": "sort", "value": "date" }
      ]
    },
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "X-Naver-Client-Id", "value": "={{$env.NAVER_CLIENT_ID}}" },
        { "name": "X-Naver-Client-Secret", "value": "={{$env.NAVER_CLIENT_SECRET}}" }
      ]
    },
    "continueOnFail": true
  }
}
```
- 네이버 개발자센터에서 무료 앱 등록 → Client ID/Secret
- 일 25,000 호출 무료

**Node 6: Merge Crawl Results**
```json
{
  "name": "Merge Crawl Results",
  "type": "n8n-nodes-base.merge",
  "typeVersion": 3,
  "parameters": {
    "mode": "combine",
    "combinationMode": "multiplex"
  }
}
```
- 두 소스 (Twitter + Naver) 결과를 키워드별로 결합

---

### Phase 4: 데이터 정제 (1 node)

**Node 7: Clean Data & Build Prompt**
```javascript
// Code Node - JavaScript
// 모든 키워드의 크롤링 결과를 모아서 AI 프롬프트 생성
const today = new Date().toISOString().split('T')[0];
const posts = [];

for (const item of $input.all()) {
  const json = item.json;

  // Twitter 결과 파싱
  if (json.data && Array.isArray(json.data)) {
    for (const tweet of json.data) {
      if (tweet.text && tweet.text.length > 30) {
        posts.push({ source: 'Twitter', text: tweet.text.replace(/\n/g, ' ') });
      }
    }
  }

  // Naver Cafe 결과 파싱
  if (json.items && Array.isArray(json.items)) {
    for (const article of json.items) {
      const text = article.description?.replace(/<[^>]*>/g, '') || '';
      const title = article.title?.replace(/<[^>]*>/g, '') || '';
      if (text.length > 20 || title.length > 10) {
        posts.push({ source: 'NaverCafe', text: `${title}: ${text}` });
      }
    }
  }
}

// 중복 제거 (텍스트 앞 50자 기준)
const seen = new Set();
const uniquePosts = posts.filter(p => {
  const key = p.text.substring(0, 50);
  if (seen.has(key)) return false;
  seen.add(key);
  return true;
});

// AI 프롬프트 생성 (최대 150개 포스트, 토큰 제한 고려)
const formattedPosts = uniquePosts
  .slice(0, 150)
  .map(p => `[${p.source}] ${p.text}`)
  .join('\n---\n');

const prompt = `
다음은 ${today}에 수집한 SNS/카페 데이터입니다.
이 데이터를 분석하여 아래 형식으로 응답해주세요:

## 1. 핵심 사용자 니즈 TOP 5
각 니즈에 대해 근거가 되는 게시물을 1-2개 인용

## 2. 반복되는 불편사항 패턴
자주 언급되는 문제점 3가지

## 3. 추천 앱 아이디어 3개
각 아이디어에 대해:
- 해결하는 문제
- 핵심 기능 3가지
- 타겟 사용자
- 시장 기회 (상/중/하)

## 4. 종합 평가
가장 주목할만한 기회와 그 이유 (100자 내)

---
수집 데이터 (${uniquePosts.length}개 게시물):
${formattedPosts}
`.trim();

return [{ json: { prompt, postCount: uniquePosts.length, date: today } }];
```

---

### Phase 5: 병렬 AI 분석 (4 nodes)

**Node 8: Ask Claude**
```json
{
  "name": "Ask Claude",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "POST",
    "url": "https://api.anthropic.com/v1/messages",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "x-api-key", "value": "={{$env.ANTHROPIC_API_KEY}}" },
        { "name": "anthropic-version", "value": "2023-06-01" },
        { "name": "content-type", "value": "application/json" }
      ]
    },
    "sendBody": true,
    "bodyParameters": {
      "parameters": [{
        "name": "",
        "value": "={\"model\":\"claude-opus-4-5-20251101\",\"max_tokens\":2000,\"messages\":[{\"role\":\"user\",\"content\":\"{{$json.prompt}}\"}]}"
      }]
    }
  }
}
```

**Node 9: Ask Gemini**
```json
{
  "name": "Ask Gemini",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "POST",
    "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent",
    "sendQuery": true,
    "queryParameters": {
      "parameters": [{ "name": "key", "value": "={{$env.GOOGLE_AI_API_KEY}}" }]
    },
    "sendBody": true,
    "contentType": "json",
    "jsonBody": "={\"contents\":[{\"parts\":[{\"text\":\"{{$json.prompt}}\"}]}],\"generationConfig\":{\"maxOutputTokens\":2000}}"
  }
}
```

**Node 10: Ask ChatGPT**
```json
{
  "name": "Ask ChatGPT",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "parameters": {
    "method": "POST",
    "url": "https://api.openai.com/v1/chat/completions",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{$env.OPENAI_API_KEY}}" },
        { "name": "Content-Type", "value": "application/json" }
      ]
    },
    "sendBody": true,
    "contentType": "json",
    "jsonBody": "={\"model\":\"gpt-4o\",\"max_tokens\":2000,\"messages\":[{\"role\":\"user\",\"content\":\"{{$json.prompt}}\"}]}"
  }
}
```

**Node 11: Merge AI Responses**
```json
{
  "name": "Merge AI Responses",
  "type": "n8n-nodes-base.merge",
  "typeVersion": 3,
  "parameters": {
    "mode": "combine",
    "combinationMode": "multiplex"
  }
}
```

---

### Phase 6: 리포트 생성 & Notion 저장 (2 nodes)

**Node 12: Generate Final Report**
```javascript
// Code Node - JavaScript
// 3개 AI 응답을 하나의 Notion 페이지 콘텐츠로 통합
const items = $input.all();
const date = new Date().toISOString().split('T')[0];

// AI 응답 추출 함수
function extractResponse(item) {
  const json = item.json;
  // Claude 응답
  if (json.content?.[0]?.text) return json.content[0].text;
  // Gemini 응답
  if (json.candidates?.[0]?.content?.parts?.[0]?.text) {
    return json.candidates[0].content.parts[0].text;
  }
  // ChatGPT 응답
  if (json.choices?.[0]?.message?.content) {
    return json.choices[0].message.content;
  }
  return '응답 없음';
}

const responses = {
  claude: extractResponse(items[0] || {}),
  gemini: extractResponse(items[1] || {}),
  chatgpt: extractResponse(items[2] || {}),
};

return [{
  json: {
    date,
    title: `앱 아이디어 브레인스토밍 - ${date}`,
    claude: responses.claude,
    gemini: responses.gemini,
    chatgpt: responses.chatgpt,
  }
}];
```

**Node 13: Save to Notion**
```json
{
  "name": "Save to Notion",
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "parameters": {
    "resource": "page",
    "operation": "create",
    "databaseId": "={{$env.NOTION_DATABASE_ID}}",
    "title": "={{$json.title}}",
    "propertiesUi": {
      "propertyValues": [
        { "key": "Date", "type": "date", "dateValue": "={{$json.date}}" },
        { "key": "Status", "type": "select", "selectValue": "New" }
      ]
    },
    "blockUi": {
      "blockValues": [
        {
          "type": "heading_2",
          "textContent": "Claude 분석"
        },
        {
          "type": "paragraph",
          "textContent": "={{$json.claude}}"
        },
        {
          "type": "heading_2",
          "textContent": "Gemini 분석"
        },
        {
          "type": "paragraph",
          "textContent": "={{$json.gemini}}"
        },
        {
          "type": "heading_2",
          "textContent": "ChatGPT 분석"
        },
        {
          "type": "paragraph",
          "textContent": "={{$json.chatgpt}}"
        }
      ]
    }
  },
  "credentials": {
    "notionApi": { "id": "{{CRED_ID}}", "name": "Notion API" }
  }
}
```

---

### Phase 7: 에러 핸들링 (2 nodes)

**Node 14: Error Trigger**
```json
{
  "name": "Error Trigger",
  "type": "n8n-nodes-base.errorTrigger",
  "typeVersion": 1
}
```

**Node 15: Send Error Notification**
```json
{
  "name": "Notify Error",
  "type": "n8n-nodes-base.emailSend",
  "typeVersion": 2.1,
  "parameters": {
    "toEmail": "={{$env.NOTIFY_EMAIL}}",
    "subject": "=[n8n] 브레인스토밍 워크플로우 오류 - {{$json.execution.error.message}}",
    "text": "=워크플로우: {{$json.workflow.name}}\n오류: {{$json.execution.error.message}}\n시간: {{$json.execution.startedAt}}"
  }
}
```
- 또는 Slack DM으로 대체 가능

---

## 노드 연결 맵 (Connections)

```
Daily Crawl Schedule → Get Keywords
Get Keywords → Build Keyword Items
Build Keyword Items → Crawl Twitter
Build Keyword Items → Crawl Naver Cafe
Crawl Twitter → Merge Crawl Results [input 0]
Crawl Naver Cafe → Merge Crawl Results [input 1]
Merge Crawl Results → Clean Data & Build Prompt
Clean Data & Build Prompt → Ask Claude
Clean Data & Build Prompt → Ask Gemini
Clean Data & Build Prompt → Ask ChatGPT
Ask Claude → Merge AI Responses [input 0]
Ask Gemini → Merge AI Responses [input 1]
Ask ChatGPT → Merge AI Responses [input 2]
Merge AI Responses → Generate Final Report
Generate Final Report → Save to Notion
```

---

## 사전 준비 체크리스트

### API 키 / 인증 설정
| 서비스 | 발급 위치 | 환경변수 | 비용 |
|--------|---------|---------|------|
| X (Twitter) | developer.twitter.com | `TWITTER_BEARER_TOKEN` | 무료 (1,500트윗/월) |
| Naver Search | developers.naver.com | `NAVER_CLIENT_ID`, `NAVER_CLIENT_SECRET` | 무료 |
| Anthropic Claude | console.anthropic.com | `ANTHROPIC_API_KEY` | 사용량 기반 |
| Google Gemini | aistudio.google.com | `GOOGLE_AI_API_KEY` | 무료 티어 있음 |
| OpenAI ChatGPT | platform.openai.com | `OPENAI_API_KEY` | 사용량 기반 |
| Google Sheets | console.cloud.google.com | OAuth2 설정 | 무료 |
| Notion | notion.so/my-integrations | `NOTION_DATABASE_ID` | 무료 |

### Google Sheets 구조
```
A열: keyword    B열: active
앱 추천         TRUE
이런 앱 없나    TRUE
불편한 점       TRUE
개발 요청       FALSE  ← 비활성화 예시
```

### n8n 환경변수 설정 (.env)
```
TWITTER_BEARER_TOKEN=xxx
NAVER_CLIENT_ID=xxx
NAVER_CLIENT_SECRET=xxx
ANTHROPIC_API_KEY=xxx
GOOGLE_AI_API_KEY=xxx
OPENAI_API_KEY=xxx
NOTION_DATABASE_ID=xxx
NOTIFY_EMAIL=your@email.com
```

---

## 구현 순서

```
Step 1: n8n_create_workflow (빈 워크플로우, settings.timezone="Asia/Seoul")
Step 2: Phase 1 노드 추가 + 연결 (Schedule → Sheets)
Step 3: Phase 2 노드 추가 (Code - keyword split)
Step 4: Phase 3 노드 추가 + 연결 (크롤러 2개 + Merge)
Step 5: Phase 4 노드 추가 (Data cleaning + prompt)
Step 6: Phase 5 노드 추가 + 연결 (AI 3개 + Merge)
Step 7: Phase 6 노드 추가 (Report + Notion)
Step 8: Phase 7 노드 추가 (Error handling)
Step 9: n8n_validate_workflow (검증)
Step 10: n8n_test_workflow (수동 테스트)
Step 11: 워크플로우 활성화
```

---

## 검증 전략

1. **수동 실행 테스트**
   - 워크플로우를 활성화 전에 Manual Trigger 추가하여 테스트
   - 각 Phase별 중간 결과 확인

2. **Google Sheets 연결 확인**
   - `Get Keywords` 노드만 단독 실행
   - 키워드 배열이 정상 출력되는지 확인

3. **크롤링 테스트**
   - `Crawl Twitter` 단독 실행 → API 응답 구조 확인
   - `Crawl Naver Cafe` 단독 실행 → items 배열 확인

4. **AI 호출 테스트**
   - 짧은 테스트 프롬프트로 각 AI API 호출 확인
   - 응답 파싱 경로 검증 (content[0].text, candidates[0].content 등)

5. **Notion 저장 테스트**
   - 하드코딩된 테스트 데이터로 Notion 페이지 생성 확인

6. **전체 통합 테스트**
   - 전체 워크플로우 Manual 실행
   - 실행 로그에서 각 노드 성공/실패 확인
   - Notion에 페이지가 정상 생성되는지 확인

---

## 주요 파일 참조

- 워크플로우 패턴: `n8n-skills/skills/n8n-workflow-patterns/`
  - `scheduled_tasks.md` - Schedule 노드 설정 상세
  - `http_api_integration.md` - HTTP Request 노드 패턴
  - `ai_agent_workflow.md` - AI 노드 연결 패턴
- MCP 툴 사용법: `n8n-skills/skills/n8n-mcp-tools-expert/SKILL.md`
- JavaScript 코드: `n8n-skills/skills/n8n-code-javascript/SKILL.md`
- 검증: `n8n-skills/skills/n8n-validation-expert/SKILL.md`
- 워크플로우 예시: `n8n-mcp/src/mcp/workflow-examples.ts`
