# agent-memory 설계 스펙


## 1. Context
- **목적**: 학습·실험. `rohitg00/agentmemory`(TS)의 핵심을 **Rust 단일 바이너리**로 재구현해 본다.
- **원칙**: 기본은 agentmemory를 따른다. 다르게 가는 건 두 경우뿐이다(§9).
  1. LLM 없이는 의미가 없는 부분
  2. 이슈로 확인된 약점이나 과한 복잡도
- **성공 기준**: 세션 1에서 한 작업을 세션 2(Claude Code 또는 Codex)에서 `search_memory`로 찾을 수 있다. 같은 기능이 두 호스트 모두에서 동작한다.
- **레포**: `~/dev/wooto/agent-memory` → `github.com/baleen37/agent-memory` (public, MIT)
- **배포**: `baleen37/baleen-marketplace` (episodic-memory와 같은 패턴)

## 2. 용어
| 용어 | 뜻 | 원본 용어 |
|---|---|---|
| **record** | 검색 대상인 행 하나. event 또는 memory | — |
| **event** | transcript에서 자동으로 남긴 일 하나(프롬프트, 도구 호출, 답변). episodic 축 | observation |
| **memory** | 에이전트가 `add_memory`로 저장한 지식. semantic 축 | memory |
| **session** | 호스트 대화 하나. project와 offset을 가진다 | session |
| **project** | git common-dir 상위 디렉터리의 basename | project |
| **offset** | transcript에서 이미 읽은 바이트 위치 | — |
| **ingest** | 훅이 offset 이후를 읽어 event를 쓰는 일 | observe |
| **index** | 데몬이 record에 terms(FTS)와 vec를 붙이는 일 | — |
| **replace** | 거의 같은 memory를 같은 id에 덮어쓰기 | supersede |
| **expire** | `expires_at`이 지난 memory를 삭제 | auto-forget(TTL) |

- 세션 요약은 만들지 않는다. 원본도 LLM이 없으면 요약이 없다(`summarize.ts:261-267`).
- 원본 4-tier 중 Working만 event로 가져온다. Episodic 요약, Semantic, Procedural은 LLM으로만 만들어지므로 제외한다(원본 `consolidation-pipeline.ts:95-265`).
- 컨텍스트 주입은 하지 않는다. 과거 기록은 에이전트가 `search_memory`로 꺼낸다.

## 3. 런타임
```
세션 A ─ agent-memory mcp ─┐ (stdio ↔ socket 바이트 파이프)
세션 B ─ agent-memory mcp ─┴─ ~/.agent-memory/daemon-{VER}.sock ─ agent-memory daemon
                                                                 ├ MCP JSON-RPC 처리
                                                                 ├ lindera 사전 + 임베딩 모델 (1회 로드)
                                                                 ├ indexer: 1초마다 indexed=0 처리
                                                                 └ memory 쓰기, 검색, expire
훅 ── agent-memory hook ingest → SQLite에 원문만 기록 (lindera·모델 로드 없음)
```
- **서브커맨드 3개**: `hook ingest --agent <claude|codex>` / `mcp` / `daemon`
- **mcp**: 시작할 때 cwd로 project를 계산해 첫 줄 `{"project":"…"}`로 보내고, 이후 stdio와 소켓을 바이트 그대로 잇는다. 자체 메시지 형식은 없다.
- **데몬은 항상 떠 있다고 가정한다.**
  - mcp가 소켓 연결에 실패하면 데몬을 detached로 띄우고 백오프하며 재시도한다. 10초 안에 연결하지 못하면 종료하고, 호스트가 MCP 서버 에러를 보여준다.
  - 첫 기동 때 모델을 내려받는 동안 도구 호출은 "모델 다운로드 중" 에러를 돌려준다.
- **싱글턴**: `daemon-{VER}.lock`에 `flock`을 건다. 락을 못 잡은 쪽은 바로 종료한다.
- **유휴 종료**: 클라이언트 0개 + `indexed=0`인 행이 없는 상태로 10분이 지나면 종료한다.
- **버전 교체**: 소켓과 락이 버전별이라 handshake가 없다.
  - 업그레이드 직후에는 구버전 데몬과 새 데몬이 함께 돈다. 구버전 데몬은 클라이언트가 사라지면 10분 뒤 스스로 종료한다. 그동안 모델이 두 번 로드된다(약 300MB).
  - 두 데몬이 같은 행을 색인해도 결과는 같다. 색인 커밋은 `indexed=0`인 행에만 반영한다.

## 4. 저장 (SQLite, `~/.agent-memory/memory.db`)
```sql
PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL; PRAGMA busy_timeout=500;

sessions(id TEXT PRIMARY KEY, project TEXT NOT NULL, offset INTEGER NOT NULL DEFAULT 0)

records(id INTEGER PRIMARY KEY,
        kind TEXT NOT NULL,              -- 'event' | 'memory'
        session_id TEXT,                 -- memory는 NULL
        project TEXT NOT NULL,
        ts INTEGER NOT NULL,             -- unix ms
        text TEXT NOT NULL,
        expires_at INTEGER,              -- memory TTL, event는 NULL
        indexed INTEGER NOT NULL DEFAULT 0)
CREATE INDEX records_pending ON records(id) WHERE indexed = 0;
CREATE INDEX records_expiry  ON records(expires_at) WHERE expires_at IS NOT NULL;

CREATE VIRTUAL TABLE records_fts USING fts5(terms, content='', contentless_delete=1,  -- SQLite 3.43+ (rusqlite bundled)
       tokenize='porter unicode61 remove_diacritics 2');   -- rowid = records.id

vecs(id INTEGER PRIMARY KEY, project TEXT NOT NULL, v BLOB NOT NULL)   -- 256 x f32, L2 정규화

CREATE TRIGGER records_ad AFTER DELETE ON records BEGIN
  DELETE FROM records_fts WHERE rowid = old.id;
  DELETE FROM vecs WHERE id = old.id;
END;
```
- **쓰기는 모두 `BEGIN IMMEDIATE`로 시작한다.** WAL에서 읽기 트랜잭션을 쓰기로 올리면 busy_timeout을 기다리지 않고 바로 BUSY가 나기 때문이다.
- 0.5초 안에 락을 못 잡으면 그 작업을 포기한다. ingest는 offset을 갱신하지 않으므로 다음 Stop에서 같은 구간을 다시 읽는다.
- **terms**: lindera + lindera-ko-dic(`embed-ko-dic`)으로 한글 구간을 형태소로 나눠 공백으로 이어 붙인 문자열. 영어는 FTS5 porter가 처리한다. 원본 KV처럼 원문 대신 terms만 색인한다.
- **vecs를 따로 두는 이유**: 벡터 스캔이 텍스트 페이지를 읽지 않게 하려는 것이다. 검색 필터에 쓰려고 project를 복사해 둔다.
- **project**: `git rev-parse --git-common-dir`의 상위 디렉터리 basename. git이 아니면 cwd의 basename. 세션의 첫 ingest 때 한 번 계산해 `sessions`에 저장한다. worktree도 같은 project로 묶인다(원본 `_project.ts:5-19`는 toplevel basename이라 갈라진다, agentmemory#515).
- **스키마 버전**: `PRAGMA user_version`은 정수 하나다. 마이그레이션은 추가만 허용한다(컬럼·테이블·인덱스 추가, 기본값 필수). 그래서 구버전 바이너리도 계속 읽고 쓴다.
  - 깨지는 변경이 필요하면 DB 파일 이름을 바꾼다(`memory-2.db`). 새 바이너리가 처음 열 때 옛 파일에서 한 번 복사한다.
- **모델 교체**: 마이그레이션에서 `DELETE FROM vecs; UPDATE records SET indexed = 0;`를 실행한다. 데몬이 백그라운드에서 다시 색인한다.

## 5. 캡처 (훅 2개, 항상 exit 0)
| 이벤트 | 실행 | 동작 |
|---|---|---|
| Stop | async | `hook ingest` |
| SessionEnd | async (Codex는 동기) | `hook ingest`. 중단 등으로 Stop이 뜨지 않은 턴을 회수한다 |

- 도구 호출마다 기록하지 않는다. 두 호스트 모두 훅 입력에 `transcript_path`를 넘긴다(Codex `codex-rs/hooks/src/events/stop.rs:32`).
- 쓰지 않는 이벤트: SessionStart, UserPromptSubmit, PreToolUse, PostToolUse(Failure), PreCompact, Subagent*, Notification, TaskCompleted. 기록할 내용은 transcript에 다 있고, 주입을 하지 않는다.

**ingest** (한 트랜잭션)
1. stdin에서 `session_id`, `transcript_path`, `cwd`를 받는다.
2. `BEGIN IMMEDIATE`. 세션 행이 없으면 project를 계산해 만든다. `old = offset`.
3. `old`부터 마지막 완결 줄까지 읽는다(최대 1MB. 남으면 다음 ingest가 이어서 읽는다).
4. 호스트별 파서(Claude JSONL, Codex rollout JSONL)로 event를 뽑는다. 파싱할 수 없는 줄은 건너뛰고 로그를 남긴다.
5. event마다 text를 만든다.

   | transcript 항목 | text |
   |---|---|
   | 사용자 프롬프트 | `[prompt] <프롬프트>` |
   | 도구 호출 + 결과 | `[<도구명>] <입력 JSON> \| <출력>`. 실패면 `[error][<도구명>] …` |
   | 턴의 마지막 답변 | `[response] <답변>` |

   - **마스킹을 먼저 한다**: `<private>…</private>`는 `[REDACTED]`, 시크릿 16종(정규식 14개 + PEM 개인키 블록 + URL 자격증명, 원본 `privacy.ts:5-31`)은 `[REDACTED_SECRET]`.
   - **그다음 400자로 자른다**(원본 narrative 길이, `compress-synthetic.ts:94`). base64 이미지(`data:image/`, `iVBORw0KGgo`, `/9j/`)는 `[image]`로 바꾼다.
   - **제외**: 서버 이름이 `agent-memory`인 MCP 도구(자기 호출), `toolsearch`, `listmcpresources`, `fetchmcpresource`(원본 `_capture-filter.ts:1-37`).
6. records INSERT(`kind='event'`, `indexed=0`), `UPDATE sessions SET offset = new WHERE id = ? AND offset = old`. 0행이면 ROLLBACK한다(다른 ingest가 먼저 처리함). 아니면 COMMIT.

## 6. 색인과 정리 (데몬)
- **indexer**: 1초마다 `indexed=0`인 행을 32건씩 읽는다.
  1. 트랜잭션 밖에서 lindera로 terms를 만들고 text를 임베딩한다.
  2. `BEGIN IMMEDIATE`로 아직 `indexed=0`인 행에만 records_fts와 vecs를 INSERT하고 `indexed=1`로 바꾼다.
- **임베딩**: fastembed-rs + **EmbeddingGemma-300m Q4**, MRL로 256차원. 다국어 모델이라 한글을 처리한다(원본 로컬 기본 all-MiniLM-L6-v2는 영어 전용, agentmemory#725). 첫 기동 때 `~/.agent-memory/models`에 받고 Gemma 약관 고지를 출력한다.
- **expire**: indexer 루프가 `kind='memory' AND expires_at < now`인 행을 지운다. 트리거가 FTS와 vecs도 지운다. 검색도 만료된 행을 거른다.

## 7. 검색과 저장 (MCP 도구 3개)
| 도구 | 입력 | 출력 |
|---|---|---|
| `search_memory` | `query`, `limit`(기본 10), `all_projects`(기본 false) | `[{id, kind, ts, text}]` |
| `add_memory` | `text`, `ttl_days?` | `{id, replaced}` |
| `delete_memory` | `id` | `{}`. memory가 아니거나 없으면 에러 |

- 이름은 메모리 MCP들(mem0, OpenMemory)의 동사_명사 형태를 따른다.
- 언제 검색하고 언제 저장할지는 **MCP 서버 `instructions`**에 적는다. 별도 스킬은 두지 않는다.

**search_memory**
1. query를 lindera로 나눠 records_fts에서 `bm25()` 상위 50(k1=1.2, b=0.75로 원본과 같음).
2. query를 임베딩해 vecs를 내적으로 스캔해 상위 50. brute-force이고 10만 행에서 약 20~40ms다.
3. 둘 다 project 필터를 건다(`all_projects=false`일 때). 만료된 memory는 거른다.
4. 가중 RRF: K=60, BM25 0.4 / 벡터 0.6(원본 `hybrid-search.ts:20,30-31`).
5. 한 세션에서 최대 3개만 넣고, 모자라면 다시 채운다(원본 `:288-295`). memory는 각각 별도 그룹이다.

**add_memory** (`BEGIN IMMEDIATE` 한 번)
1. text를 마스킹하고 terms를 만든 뒤 임베딩한다.
2. 같은 project의 만료되지 않은 memory 전부와 Jaccard(terms 집합)를 비교한다. memory는 수백 건 수준이라 전체 스캔이 싸다.
3. 0.7을 넘는 것이 있으면 그 id에 덮어쓴다(text, ts, expires_at, FTS, vecs). `replaced=true`.
4. 없으면 INSERT(`kind='memory'`, `indexed=1`). FTS와 vecs도 같은 트랜잭션에 넣는다.

## 8. 플러그인 패키징
```
.claude-plugin/plugin.json      .codex-plugin/plugin.json (mcpServers, hooks: ./hooks/hooks.codex.json)
hooks/hooks.json   → Stop, SessionEnd: "<root>/bin/agent-memory" hook ingest --agent claude (async)
hooks/hooks.codex.json → 같은 2개, --agent codex
  - 경로 변수: Codex는 PLUGIN_ROOT와 CLAUDE_PLUGIN_ROOT를 모두 주입한다(discovery.rs:265). 경로는 항상 따옴표로 감싼다(agentmemory#477)
.mcp.json → 서버 이름 agent-memory, sh -lc 'exec "${PLUGIN_ROOT:-$CLAUDE_PLUGIN_ROOT}/bin/agent-memory" mcp'
bin/agent-memory  (bash 래퍼, 아래 순서로 실행할 바이너리를 정한다)
  1. AGENT_MEMORY_BIN 환경변수가 있으면 그 경로
  2. ~/.agent-memory/bin/agent-memory-v{VER}가 있으면 그것 (VER는 plugin.json 버전)
  3. 없으면 Release에서 받아 sha256을 검증한 뒤 설치한다
```
- 구버전 바이너리는 지우지 않는다. 실행 중인 세션이 쓰고 있을 수 있다.
- Codex Desktop은 플러그인 훅 버그(openai/codex#16430)가 있다. README에 `~/.codex/hooks.json` 수동 병합 방법을 안내한다.

**배포**
1. semantic-release(main)가 버전을 정한다. `scripts/sync-versions.sh`로 `Cargo.toml`과 plugin.json 2개를 맞추고 태그와 Release를 만든다.
2. 태그가 생기면 `release.yml`이 빌드한다. 타깃은 aarch64/x86_64-apple-darwin, x86_64/aarch64-unknown-linux-gnu. ort는 정적 링크한다. 산출물은 `.tar.gz`와 `.sha256`.
3. Release가 publish되면 `on-release.yml`이 `baleen37/baleen-marketplace/.github/actions/notify@v1`(source: agent-memory)를 실행한다. 시크릿은 `BALEEN_RELEASE_APP_ID`, `BALEEN_RELEASE_APP_PRIVATE_KEY`.
4. 마켓플레이스 레포(별도 PR): `sources.json`에 `"agent-memory": {"repo":"baleen37/agent-memory","paths":["./"]}`를 추가하고, 두 marketplace.json에 항목을 추가한다.

## 9. 원본과 다르게 가는 부분
| # | 원본 | 우리 | 이유 |
|---|---|---|---|
| 1 | iii-engine + Node 워커, 훅이 HTTP로 보내고 실패하면 스풀 | 훅은 SQLite에 원문만 쓰고, 데몬이 색인·검색·memory 쓰기 | 별도 엔진과 스풀이 필요 없다. 엔진 RAM 문제(agentmemory#1312)도 피한다 |
| 2 | KV + 메모리 BM25(부팅마다 재빌드), 한글은 어절 단위 + prefix 확장, 동의어 | FTS5 + lindera 형태소 분석. prefix·동의어 없음 | 영속 색인이다. 조사는 형태소 분석이, 의미 유사성은 벡터가 맡는다 |
| 3 | 훅 12개, 도구 호출마다 전송, Stop마다 세션 종료 처리 | 훅 2개 + 턴마다 transcript를 offset부터 ingest | 턴당 프로세스 1개, offset CAS로 멱등. Stop마다 요약이 첫 턴에 고정되는 문제(agentmemory#745)가 없다. 대가로 파서가 2개 필요하다 |
| 4 | observation 필드 다수(type, title, subtitle, facts, narrative, concepts, files, importance, confidence) | event는 `text` 한 줄 | LLM 없이는 대부분 비거나 상수다. 도구명은 text 앞에 남는다 |
| 5 | 턴의 마지막 답변은 raw에만 보관 | 답변도 event로 남겨 색인 | 결론이 답변에 들어 있는 경우가 많다 |
| 6 | SessionStart 주입(기본 꺼짐), PreCompact 주입(기본 켜짐) | 주입 없음 | 과거 기록은 검색으로 꺼낸다. 토큰 비용 문제(agentmemory#143)가 없다 |
| 7 | memory는 type·concepts·files·버전 체인, supersede 후보는 관찰과 섞인 BM25 상위 50, auto-forget에 Jaccard 0.9·180일 규칙 | memory는 text 하나, replace는 같은 project memory 전체와 Jaccard 0.7, expire는 TTL만 | 원본에서 동작하지 않거나 겹치는 부분이다 |
| 8 | project = toplevel basename | git common-dir 기준 | worktree 분리(agentmemory#515) |
| 9 | 도구 54개, 스킬 다수 | 도구 3개(동사_명사), 스킬 없이 server instructions | 직관성. 나머지는 검색으로 대신한다 |
| 10 | 원본 Codex 훅에 SessionEnd 없음 | Codex에도 SessionEnd 등록 | 최신 Codex가 지원한다 |
| 11 | — | 제외: LLM 기능(압축, 요약, 4-tier, reflect), graph 스트림, lessons, 슬롯, 프로필, 이미지 보관, 팀·mesh·viewer | 학습용 MVP 범위 밖이거나 LLM이 필요하다 |

## 10. 에러 처리
- 사용자 설정용 환경변수는 두지 않는다. 내부·테스트용 3개만 둔다: `AGENT_MEMORY_DIR`(데이터 경로), `AGENT_MEMORY_DISABLE`(재귀 방지), `AGENT_MEMORY_BIN`(로컬 바이너리).
- 훅은 항상 exit 0으로 끝난다. 에러는 `~/.agent-memory/logs/`에 기록한다.
- `AGENT_MEMORY_DISABLE=1`이면 훅이 즉시 종료한다.
- 데몬에 연결할 수 없으면 mcp가 종료하고 호스트가 에러를 보여준다. 훅은 데몬과 무관하게 동작한다.

## 11. 검증
- **사전 측정 (구현 첫 단계)**
  - `embed-ko-dic`을 켰을 때 바이너리 크기 증가분. 과하면 모델처럼 첫 실행 때 다운로드한다.
  - Codex가 MCP server `instructions`를 프롬프트에 넣는지 확인한다. 넣지 않으면 `memory` 스킬 1개를 둔다.
- **단위 테스트**: 형태소 분석("검색을"→"검색", "검색추천"→"추천"으로 검색되는지), 마스킹(16종, 마스킹 후 자르기), 자기 호출 제외(`mcp__agent-memory__search_memory`), text 형식(prompt, 도구, error, response), RRF와 세션당 3개, replace(Jaccard 0.7 경계, 같은 id 유지), expire(만료 행이 검색에서 빠지고 삭제되는지), 삭제 트리거(FTS·vecs 정리)
- **통합 테스트**
  - Claude와 Codex 훅 페이로드 픽스처를 stdin으로 넣는다.
  - **transcript 파서**: 실제 Claude·Codex transcript 샘플을 픽스처로 둔다. offset 이어 읽기, 마지막 줄이 미완결인 경우, 1MB 분할, 같은 구간을 동시에 두 번 ingest해도 행이 늘지 않는지(offset CAS) 확인한다.
  - MCP JSON-RPC 왕복(initialize → tools/list → tools/call)을 mcp 파이프를 거쳐 검증한다. 도구가 3개인지 확인한다.
  - **데몬 경쟁**: 클라이언트 3개를 동시에 띄워 같은 버전 데몬이 1개만 뜨는지 확인한다.
  - **indexer**: ingest 직후 `indexed=0` 행이 몇 초 안에 검색되는지, 데몬 두 개가 동시에 색인해도 FTS·vecs가 중복되지 않는지 확인한다.
- **임베딩 테스트**: `#[ignore]`로 분리하고 CI에서는 별도 job으로 돌린다(모델 다운로드 필요).
- **E2E**
  1. `AGENT_MEMORY_BIN=target/debug/agent-memory claude --plugin-dir .`로 세션 1을 돌린다.
  2. 세션 2에서 `search_memory`로 세션 1의 작업이 검색되는지 확인한다.
  3. 같은 시나리오를 Codex로 반복한다.
- **배포 검증**: v0.1.0 Release 후 `AGENT_MEMORY_BIN` 없이 래퍼를 실행해 다운로드 → 체크섬 검증 → 실행까지 되는지 확인한다.

## 다음 단계 (승인 후)
- `me:writing-plans`로 구현 계획을 작성한다. 실행 방식은 그때 고른다.
