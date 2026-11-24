# AI TOP 100 고객 상담 에이전트

가상의 산지직송 쇼핑몰 고객 문의를 카카오톡으로 받아, LLM이 운영 규칙에 따라 조회 API로 주문 정보를 확인하고 환불·배송지 변경·이관 등을 접수하는 상담 에이전트 서버입니다.

AI TOP 100 예선 제출용으로 만들었습니다. 이 저장소는 공개용 정리본이고, 대회에서 제공한 운영 자료와 API 명세는 제외했습니다. 실제 운영 API는 포함되지 않아 예제 도구와 모의 응답으로 실행 흐름을 확인할 수 있습니다.

## 서비스 구조

```
카카오톡 봇 웹훅 ──▶ /webhook (1초 안에 204 응답, 처리는 비동기)
                         │
                         ▼
               입력 검사 (길이 제한, 인젝션 감지, <user_message> 격리)
                         │
                         ▼
               LLM + 도구 호출 (최대 5회) ──▶ 운영진 액션 API (조회/접수)
                         │
                         ▼
               출력 검사 (지시문 유출 차단) ──▶ 카카오 메시지 콜백 API로 답장
```

## 개발 중 고려한 문제

| 문제 | 대응 |
|---|---|
| 웹훅은 1초 안에 응답해야 함 | 204를 먼저 돌려주고 LLM 처리와 답장은 비동기로 진행 |
| 같은 이벤트가 재전송됨 | 이벤트 ID로 중복 제거 (10분 보관) |
| 한 사용자가 연속으로 메시지를 보냄 | 사용자별 락으로 순서대로 처리 |
| 프롬프트 인젝션 ("이전 지시 무시해" 등) | 패턴 감지, 사용자 입력을 `<user_message>` 태그로 격리 |
| 지시문 유출 | 답변에 시스템 프롬프트 일부가 섞이면 차단하고 대체 문구로 응답 |
| 잘못된 대상 처리 | 지시문에 "확인된 주문 1건만 조회·접수" 규칙을 두고, 변경성 도구는 고객 확인 후에만 호출 |
| 콜백 실패 | 재시도 |
| 키 없이 개발 | `LLM_PROVIDER=fake`, `MOCK_ACTIONS=1`로 LLM·API 없이 전체 흐름 테스트 |

## 저장소 구성

| 파일 | 역할 |
|---|---|
| `app.py` | FastAPI 서버 (웹훅, LLM 호출, 도구 실행, 콜백) |
| `prompts/system.md` | 상담 에이전트 지시문 (응답 형식, 처리 절차, 이관 기준) |
| `data/` | 운영 지식 `*.md`를 넣으면 지시문 뒤에 자동으로 합쳐짐. 대회 자료라 공개본에서는 비워 둠 |
| `tools.example.json` | 액션 API를 도구로 정의하는 형식 예시. 실제 사용 시 `tools.json`으로 작성 |
| `test_local.py` | 가짜 카카오 콜백 서버를 띄워 웹훅부터 답장까지 통합 테스트 |
| `Dockerfile` | Render 등 컨테이너 배포용 |

## 실행 방법

```bash
pip install -r requirements.txt
cp .env.example .env              # 키 입력
cp tools.example.json tools.json  # 사용할 API에 맞게 수정

# 키 없이 동작 확인
LLM_PROVIDER=fake MOCK_ACTIONS=1 python test_local.py "배송지 바꾸고 싶어요"

# 실제 LLM으로 테스트
LLM_PROVIDER=anthropic ANTHROPIC_API_KEY=... python test_local.py "질문1" "질문2"

# 서버 실행
uvicorn app:app --port 8000
```

LLM은 `LLM_PROVIDER`로 `anthropic`, `openai`, `fake` 중에서 고를 수 있습니다.

## 기술 스택
Python, FastAPI, httpx, LLM tool calling, Docker, Render, 카카오톡 봇 API
