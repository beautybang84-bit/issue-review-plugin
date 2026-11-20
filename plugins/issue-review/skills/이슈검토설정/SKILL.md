---
name: 이슈검토설정
description: /세무이슈·/회계이슈의 선택 항목(NotebookLM 노트북, GPT 교차검토, CaseNote 브라우저 조회)과 결과 저장 폴더를 설정하고, 필요한 MCP 연결 상태를 점검한다. "/이슈검토설정"으로 호출.
disable-model-invocation: true
---

# 이슈 검토 설정

모든 소통은 한국어로 한다. 설정 파일은 `~/.claude/issue-review-config.json` 이다.

## 1. 현재 상태 점검

1. 설정 파일이 있으면 Read로 읽어 현재 값을 보여 준다. 없으면 "아직 설정 없음(선택 항목 모두 꺼짐)"이라고 말한다.
2. MCP 연결 상태를 확인한다. `claude mcp list`를 Bash로 실행해 아래 서버가 연결(✓)인지 본다.
   - 필수(세무): `nts-tax`, `korean-law` / 필수(회계): `accountingwiki` / 보조: `taxdoctor`
   - 빠진 서버는 이 플러그인 README의 "MCP 설치" 절 명령을 그대로 보여 준다. 사용자가 원하면 대신 실행한다(법제처 API 키 같은 개인 값은 사용자에게 받는다).
3. 선택 항목 도구가 있는지 확인한다.
   - NotebookLM: `nlm --version` (없으면 `pip install notebooklm-mcp-cli` 후 `nlm login` 안내)
   - GPT 교차검토: `codex --version` (없으면 `npm install -g @openai/codex` 후 `codex login` 안내)
   - CaseNote: Claude in Chrome 확장 연결 여부

## 2. 선택 항목 묻기

AskUserQuestion으로 한 번에 묻는다 (최대 4문항).
- NotebookLM을 쓸지 — 쓰면 실무자료 노트북과 기준서 노트북의 식별자(별칭 또는 ID)를 받는다. 별칭은 `nlm alias set <별칭> <노트북ID>`로 만든다. **노트북 "이름"으로는 질의가 안 되므로 별칭이나 ID를 받는다.** 노트북 목록은 `nlm list notebooks`로 보여 줄 수 있다.
- GPT 교차검토를 쓸지 — OpenAI API 키(선불 크레딧) 또는 ChatGPT 계정으로 `codex login`이 되어 있어야 한다. 비용은 사용자 본인 계정에 청구된다.
- CaseNote 브라우저 조회를 쓸지 — Chrome 확장이 필요하고, 세무 인용 재조사 단계에서만 쓴다.
- 결과 저장 기본 폴더 — 비워 두면 저장할 때마다 묻는다.

## 3. 저장

받은 값으로 설정 파일을 쓴다.

```json
{
  "notebooklm": { "enabled": true, "practice_notebook": "<별칭 또는 ID>", "standards_notebook": "<별칭 또는 ID 또는 빈칸>" },
  "gpt_review": { "enabled": false },
  "casenote":   { "enabled": false },
  "save_root":  "<폴더 경로 또는 빈칸>"
}
```

API 키·비밀번호는 이 파일에 쓰지 않는다. 쓴 내용을 보여 주고 끝낸다.
