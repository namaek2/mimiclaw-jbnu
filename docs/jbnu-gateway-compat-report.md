# JBNU / MindLogic Gateway 호환성 작업 보고서

> 대상 문서: <https://docs.mindlogic.ai/docs/jbnu/gateway/api-reference/chat-completions>
> 작업일: 2026-06-08
> 작업 범위: `main/mimi_config.h`, `main/llm/llm_proxy.c`

---

## 1. 개요

MimiClaw 펌웨어가 JBNU/MindLogic Gateway의 **Chat Completions API**(OpenAI 호환)와
정상적으로 통신하도록 LLM 프록시 코드를 수정했다.

기존 코드에는 이미 `provider = "openai"` 경로(OpenAI 형식의 요청/응답 변환)가
구현되어 있었으나, **엔드포인트 주소가 잘못 지정**되어 있어 실제로는 게이트웨이에
도달할 수 없는 상태였다. 이를 문서 사양에 맞게 바로잡았다.

---

## 2. 문서 API 사양 요약

| 항목 | 값 |
| --- | --- |
| Endpoint | `https://factchat-cloud.mindlogic.ai/v1/gateway/chat/completions/` |
| Method | `POST` |
| 인증 헤더 | `Authorization: Bearer <API_KEY>` |
| 콘텐츠 헤더 | `Content-Type: application/json` |
| 요청 형식 | OpenAI Chat Completions 호환 (`model`, `messages`, `max_tokens`/`max_completion_tokens`, `tools`, `tool_choice`, `stream` …) |
| 응답 형식 | OpenAI 표준 (`choices[].message`, `tool_calls`, `finish_reason`, `usage`) |
| 멀티 프로바이더 | OpenAI(gpt), Anthropic(claude), Gemini, xAI(Grok), Perplexity 등을 단일 게이트웨이로 중계 |

---

## 3. 기존 코드 분석

### 3.1 통신 경로 구조

LLM 호출은 `llm_http_call()`에서 두 갈래로 분기한다 (`main/llm/llm_proxy.c`).

- **직접 경로** `llm_http_direct()` — `esp_http_client`에 **전체 URL**(`llm_api_url()`)을 전달.
- **프록시 경로** `llm_http_via_proxy()` — CONNECT 터널을 열고 **호스트**(`llm_api_host()`)와
  **경로**(`llm_api_path()`)를 직접 조립해 수동으로 HTTP 요청을 작성.

즉 URL·HOST·PATH 세 값이 **항상 서로 일치**해야 두 경로 모두 같은 서버로 향한다.

### 3.2 이미 호환되던 부분 (수정 불필요)

문서의 OpenAI 호환 사양과 일치하므로 그대로 둔다.

- **인증 헤더**: openai 분기에서 `Authorization: Bearer <key>` 전송
  (직접 경로 `llm_proxy.c:268-273`, 프록시 경로 `:296-304`).
- **요청 바디**: `convert_messages_openai()` / `convert_tools_openai()`가
  Anthropic 내부 메시지 포맷을 OpenAI `messages` / `tools`(`type:function`) 형식으로 변환
  (`:400-533`, `:366-398`). `tool_choice: "auto"`, `max_completion_tokens` 포함(`:562-578`).
- **응답 파싱**: `choices[0].message.content`, `message.tool_calls`,
  `finish_reason == "tool_calls"`를 표준대로 해석 (`:638-690`).

### 3.3 발견된 결함

| # | 위치 | 문제 | 영향 |
| --- | --- | --- | --- |
| **B1** | `mimi_config.h` `MIMI_OPENAI_API_URL` | `…/v1/gateway` 로만 지정 — `/chat/completions/` 누락 | **직접 경로**가 잘못된 URL로 POST → 404/리다이렉트 실패 |
| **B2** | `llm_proxy.c` `llm_api_host()` | openai일 때 `"api.openai.com"` 하드코딩 | **프록시 경로**가 게이트웨이가 아닌 OpenAI 본사로 접속 |
| **B3** | `llm_proxy.c` `llm_api_path()` | openai일 때 `"/v1/chat/completions"` 하드코딩 | 프록시 경로의 요청 라인이 게이트웨이 경로와 불일치 |

세 값이 제각각이라 직접 경로와 프록시 경로가 **서로 다른 서버**로 향하는 모순 상태였다.

---

## 4. 변경 내용

### 4.1 `main/mimi_config.h`

HOST / PATH 를 단일 출처로 정의하고, 전체 URL을 이 둘로부터 합성하도록 변경.
이로써 URL·HOST·PATH의 불일치 가능성을 구조적으로 제거했다.

```c
/* MindLogic / JBNU gateway: OpenAI-compatible chat completions endpoint. */
#define MIMI_OPENAI_API_HOST  "factchat-cloud.mindlogic.ai"
#define MIMI_OPENAI_API_PATH  "/v1/gateway/chat/completions/"
#define MIMI_OPENAI_API_URL   "https://" MIMI_OPENAI_API_HOST MIMI_OPENAI_API_PATH
```

- 결함 **B1** 해결: `MIMI_OPENAI_API_URL` 이
  `https://factchat-cloud.mindlogic.ai/v1/gateway/chat/completions/` 로 정확히 해석됨.
- 문서의 **후행 슬래시**(`/completions/`)를 그대로 유지 (게이트웨이가 Django 계열로
  추정되어 trailing slash에 민감할 수 있으므로 보존).

### 4.2 `main/llm/llm_proxy.c`

하드코딩된 OpenAI 본사 주소를 게이트웨이 상수로 교체.

```c
static const char *llm_api_host(void)
{
    return provider_is_openai() ? MIMI_OPENAI_API_HOST : "api.anthropic.com";
}

static const char *llm_api_path(void)
{
    return provider_is_openai() ? MIMI_OPENAI_API_PATH : "/v1/messages";
}
```

- 결함 **B2/B3** 해결: 프록시 경로도 직접 경로와 동일하게
  `factchat-cloud.mindlogic.ai` + `/v1/gateway/chat/completions/` 로 향함.

---

## 5. 변경 후 동작 확인

- `grep`으로 잔존 하드코딩(`api.openai.com`, `/v1/chat/completions`)이 없음을 확인.
- URL·HOST·PATH 세 상수가 하나의 출처에서 합성되어 **항상 일치**함을 보장.
- 직접 경로와 프록시 경로 모두 동일한 게이트웨이 엔드포인트를 사용.

> 참고: 편집기 LSP(clang)가 보고하는 `esp_err.h not found`, `MIMI_LLM_* undeclared`
> 경고는 ESP-IDF 툴체인 헤더 경로가 LSP에 설정되지 않아 생기는 **환경 한계**이며,
> 이번 변경(매크로 값 수정)과는 무관한 기존 잡음이다. 실제 빌드는 `idf.py build`로 수행한다.

---

## 6. 사용 방법 (런타임 설정)

게이트웨이를 사용하려면 기기에서 다음을 설정한다 (시리얼 CLI 또는 온보딩 웹).

```
set_model_provider openai
set_api_key <JBNU_GATEWAY_API_KEY>
set_model <게이트웨이가 제공하는 모델 ID>   # 예: gpt-4o, claude-... 등
```

- 모델 ID 목록은 게이트웨이의 `/v1/gateway/models/` 에서 조회 가능 (문서 참조).
- `provider` 를 `anthropic` 으로 두면 기존 Anthropic 직접 호출 경로가 유지된다.

---

## 7. 향후 개선 여지 (선택)

- **스트리밍(SSE)**: 문서는 `stream: true` / `stream_options.include_usage` 를 지원하나
  현재 구현은 비스트리밍 단일 응답만 사용한다. 필요 시 SSE 파서 추가 가능.
- **provider 명칭**: 현재 `"openai"` 값이 사실상 "OpenAI 호환 게이트웨이"를 의미하므로,
  추후 `"jbnu"`/`"mindlogic"` 등 별칭을 추가해 의미를 명확히 할 수 있다.
- **usage 로깅**: 응답의 `usage` 토큰 카운트를 기록하면 사용량 추적에 유용하다.
