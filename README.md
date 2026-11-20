# 세무·회계 이슈 검토 플러그인 (issue-review)

Claude Code에서 세무·회계 이슈를 입력하면, 여러 검색 agent를 동시에 돌려 **원문 근거(기준서 문단·질의회신·예규·심판례·판례·조문)**를 모으고, **인용이 실제로 있는지·취지가 맞는지 검증**한 뒤 결론을 정리해 주는 플러그인입니다.

| 명령 | 하는 일 |
|---|---|
| `/세무이슈 <질문>` | 국세법령정보시스템 + 국가법령정보센터 본조사, TaxDoctor 누락 점검, 인용 원문 검증(재조사 루프) |
| `/회계이슈 <질문>` | 회계기준원 + 회계위키 + 웹 조사, 회계위키로 문단·질의회신 인용 점검 |
| `/메일회신 <제목·보낸 사람>` | 거래처 질의 메일을 읽고 회계·세무를 판단해 위 조사·인용 검증을 거친 회신 초안 작성 (보내지 않음, 확인 후 메일함 초안까지만) |
| `/이슈검토설정` | 선택 항목(NotebookLM·GPT 교차검토·CaseNote)과 저장 폴더 설정, MCP 연결 점검 |

> 결과는 실무 참고용입니다. 최종 판단은 담당 세무사·회계사가 원문을 확인해 내립니다.

---

## 1. 설치

> 처음 설치하는 분은 **[설치설명서.md](설치설명서.md)**를 따라 하세요. 준비물 설치, 법제처 API 키 신청, NotebookLM 연결, OpenAI 유료 키 구매까지 단계별로 적혀 있습니다. 아래는 요약입니다.

### 1-1. 플러그인 설치 (Claude Code 안에서)

GitHub 저장소로 받는 경우:
```
/plugin marketplace add beautybang84-bit/issue-review-plugin
/plugin install issue-review@issue-review-market
```

폴더(압축 파일)로 받은 경우: 압축을 푼 폴더 경로로
```
/plugin marketplace add C:\경로\공유용_이슈검토플러그인
/plugin install issue-review@issue-review-market
```

설치하면 **회계위키**와 **TaxDoctor** MCP는 자동으로 연결됩니다(주소만 쓰는 공개 서버).

### 1-2. MCP 설치 — 세무 본조사용 2종 (터미널에서)

**국세법령정보시스템 (nts-tax)** — 파이썬 필요
```bash
git clone https://github.com/taxwoong/nts-tax-mcp.git
cd nts-tax-mcp
pip install -r requirements.txt
claude mcp add --scope user nts-tax -e MCP_TRANSPORT=stdio -- python "<클론한 폴더 전체 경로>/server.py"
```

**국가법령정보센터 (korean-law)** — Node.js 필요, 법제처 Open API 인증키(OC) 필요
1. https://open.law.go.kr 에서 회원가입 후 Open API 사용 신청 → 본인 OC 값(보통 가입 이메일 아이디) 확보
2. 설치·등록
```bash
npm install -g korean-law-mcp
claude mcp add --scope user korean-law -e LAW_OC=<본인 OC> -- korean-law-mcp
```
(명령 이름이 안 잡히면 `-- node "<npm 전역 폴더>/korean-law-mcp/build/index.js"` 형태로 등록)

설치 후 `claude mcp list`에서 `nts-tax`, `korean-law`, `accountingwiki`, `taxdoctor`가 연결(✓)인지 확인합니다. 빠진 서버가 있어도 agent가 웹 검색으로 대신하지만 정확도가 떨어집니다.

### 1-3. 선택 항목 (필요한 사람만)

| 항목 | 쓰는 곳 | 준비 |
|---|---|---|
| **NotebookLM** | `/회계이슈`에서 본인이 모은 실무자료·기준서 노트북 조회 | `pip install notebooklm-mcp-cli` → `nlm login` → `nlm alias set <별칭> <노트북ID>` |
| **GPT 교차검토** | 결론을 다른 모델(GPT)로 한 번 더 점검 | `npm install -g @openai/codex` → `codex login` (본인 OpenAI API 키 또는 ChatGPT 계정, **비용은 본인 부담**) |
| **CaseNote** | 세무 인용이 끝까지 확인 안 될 때 후속 판례 확인 | Claude in Chrome 확장 연결 |

준비가 끝나면 Claude Code에서 `/이슈검토설정`을 실행해 켜면 됩니다. 설정은 `~/.claude/issue-review-config.json`에 저장되고, API 키는 이 파일에 쓰지 않습니다.

---

## 2. 사용 예

```
/메일회신 ○○전자 김과장 법인세 질의
/세무이슈 특수관계 없는 법인에 전환사채를 발행한 뒤 전환된 경우, 미상각 사채발행비는 전환일에 손금 추인되는가?
/회계이슈 일반기업회계기준 적용 회사가 판매장려금을 매출 차감으로 처리해야 하는지
```

결과는 「결론 / 검토 내용 / 근거 목록(검증 결과 포함) / 상충·한계 / 추가 참고의견」 순서로 나옵니다. "파일로 저장해 줘"라고 하면 설정한 저장 폴더(없으면 물어봄)에 `.md`로 저장합니다.

## 3. 구성

```
plugins/issue-review/
├── .claude-plugin/plugin.json
├── .mcp.json                 # 회계위키·TaxDoctor (자동 연결)
├── skills/
│   ├── 세무이슈/SKILL.md
│   ├── 회계이슈/SKILL.md
│   ├── 메일회신/SKILL.md
│   └── 이슈검토설정/SKILL.md
└── agents/
    ├── nts-tax-search.md     # 국세법령정보시스템·지방세
    ├── korean-law-search.md  # 법령 조문·판례
    ├── citation-verify.md    # 인용 원문 검증
    ├── kasb-search.md        # 회계기준원
    ├── web-search.md         # 웹 (참고)
    ├── notebooklm-search.md  # [선택] NotebookLM
    ├── gpt-review.md         # [선택] GPT 교차검토
    └── casenote-search.md    # [선택] CaseNote
```

## 4. 유의사항

- GPT 교차검토로 보낼 때는 거래처 상호·개인정보를 가린 뒤 보냅니다. 그래도 사무소 정보보호 방침을 먼저 확인하세요.
- 회계위키·TaxDoctor·CaseNote는 외부 서비스입니다. 질의 내용에 거래처 식별 정보를 넣지 않는 것을 권합니다.
- 각 출처의 저작권은 원 기관에 있습니다. NotebookLM 노트북은 본인 계정 자료만 씁니다.

## 5. 업데이트

새 버전이 올라오면 Claude Code에서:
```
/plugin marketplace update issue-review-market
```
