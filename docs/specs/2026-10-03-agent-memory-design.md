# agent-memory 설계 스펙


## 1. Context
- **목적**: 학습·실험. `rohitg00/agentmemory`(TS)의 핵심을 **Rust 단일 바이너리**로 재구현해 본다.
- **원칙**: 기본은 agentmemory를 따른다. 다르게 가는 건 두 경우뿐이다(§8).
  1. LLM 없이는 의미가 없는 부분
  2. 이슈로 확인된 약점이나 과한 복잡도
- **성공 기준**: 세션 1에서 한 작업을 세션 2(Claude Code 또는 Codex)에서 `search_memory`로 찾을 수 있다. 같은 기능이 두 호스트 모두에서 동작한다.
- **레포**: `~/dev/wooto/agent-memory` → `github.com/baleen37/agent-memory` (public, MIT)
- **배포**: `baleen37/baleen-marketplace` (episodic-memory와 같은 패턴)

## 2. 개념: 두 축
| 축 | 테이블 | 내용 | 만드는 주체 |
|---|---|---|---|
| **episodic** (무슨 일이 있었나) | `observations` | 프롬프트, 도구 호출, 턴의 마지막 답변. 1건당 1행 | 훅, 자동 |
| **semantic** (앞으로도 알아야 할 것) | `memories` | 사실, 패턴, 결정 | 에이전트가 `add_memory`로 |

- `sessions`는 두 축을 묶는 그릇이다(에이전트, 프로젝트, transcript 위치).
- **세션 요약은 만들지 않는다.** 원본도 LLM이 없으면 요약이 없다(`summarize.ts:261-267`).
- 원본의 4-tier 중 Working만 observations로 가져온다. Episodic 요약, Semantic(`KV.semantic`), Procedural(`KV.procedural`)은 LLM으로만 만들어지므로 제외한다(원본 `consolidation-pipeline.ts:95-265`).
- **컨텍스트 주입은 하지 않는다.** 과거 기록은 에이전트가 `search_memory`로 꺼낸다.

## 3. 런타임
```
세션 A ─ agent-memory mcp (얇은 stdio 프록시) ┐
세션 B ─ agent-memory mcp                      ├─ unix socket ─ agent-memory daemon (1개)
                                                               ├ add / search / delete 처리
                                                               ├ 임베딩 모델 로드
                                                               └ index_queue 처리
훅 ──── agent-memory hook <event> → SQLite에 직접 기록 (데몬을 쓰지 않음)
```
- **쓰기 경로는 두 개다.**
  - 훅: episodic(observations)을 SQLite에 직접 쓴다. 데몬이 죽어도 기록은 남는다.
  - 데몬: semantic(memories)의 추가와 삭제. mcp의 모든 요청은 데몬을 거친다.
- **데몬은 항상 떠 있다고 가정한다.** BM25만으로 답하는 대체 경로는 두지 않는다.
  - mcp가 소켓 연결에 실패하면 데몬을 detached로 띄우고 백오프하며 재시도한다. 10초 안에 연결하지 못하면 도구 호출에 에러를 돌려준다.
- **싱글턴**: `~/.agent-memory/daemon.lock`에 `flock`을 건다. 락을 못 잡은 쪽은 바로 종료한다.
- **유휴 종료**: 클라이언트 0개 + 큐가 빈 상태로 10분이 지나면 종료한다.
- **버전 교체**: 업그레이드 직후에는 구버전 세션과 새 세션이 함께 돈다.
  1. 연결할 때 handshake로 `{app_version, proto}`를 교환한다.
  2. 데몬보다 높은 app_version의 클라이언트는 shutdown을 요청한다. 데몬은 처리 중인 요청을 마치고 락을 푼 뒤 종료하고, 클라이언트가 새 데몬을 띄운다.
  3. 데몬보다 낮은 app_version의 클라이언트는 "세션을 재시작하세요" 에러를 돌려준다. 데몬을 종료시키지 않는다(핑퐁 방지).
- **서브커맨드**: `hook <event>` / `mcp` / `daemon` / `search` / `add` / `status`

## 4. 저장 (SQLite, `~/.agent-memory/memory.db`)
- WAL 모드, `busy_timeout=500`. `AGENT_MEMORY_DIR`로 경로를 바꿀 수 있다.
  - 0.5초 안에 락을 못 잡으면 해당 작업을 포기한다. ingest는 offset을 갱신하지 않으므로 다음 Stop에서 같은 구간을 다시 읽는다(멱등).
- `sessions(id, agent[claude|codex], project, cwd, transcript_path, transcript_offset, started_at, ended_at)`
- `observations(id "obs_<ts36>_<rand>", session_id, seq, ts, type, title, subtitle, narrative, files_json, UNIQUE(session_id, seq))`, `INDEX(session_id, seq)`
- `memories(id, project NULL 허용, type[pattern|preference|architecture|bug|workflow|fact], title(content 앞 80자), content, concepts_json, files_json, session_ids_json, version, parent_id, supersedes_json, is_latest, forget_after, created_at, updated_at)` (원본 `types.ts:125-148`에서 LLM 전용 필드 `strength`·`relatedIds`·`sourceObservationIds`·이미지 필드를 뺐다)
- `meta(key, value)`: 마지막 auto-forget 실행 시각 등
- `index_queue(ref_kind, ref_id)`: 임베딩 대기열
- `embeddings(ref_kind, ref_id, model_id, dim, vec BLOB)`
- **FTS5 색인**: `tokenize='porter unicode61 remove_diacritics 2'`.
  - 색인하는 것은 원문이 아니라 미리 나눈 `terms` 컬럼이다. 관찰은 title, subtitle, narrative, files, type을, memory는 title, content, concepts, files를 이어 붙인다(원본 `search-index.ts:259-268`, `memory-utils.ts:11-27`).
  - **형태소 분석**: lindera + lindera-ko-dic(`embed-ko-dic`). 한글 구간을 형태소로 나눠 공백으로 이어 붙인다. 영어는 FTS5 porter가 처리한다.
  - 형태소 분석은 행을 쓰는 프로세스(Stop·SessionEnd 훅, 데몬)가 **같은 트랜잭션 안에서** 한다. 그래서 BM25 색인은 항상 최신이다.
  - `is_latest=0`인 memory는 FTS·벡터 색인에서 지운다(원본 `remember.ts:193-196`).
- **project**: `git rev-parse --git-common-dir`의 상위 디렉터리 basename. git이 아니면 cwd의 basename. worktree도 같은 프로젝트로 묶인다(원본 `_project.ts:5-19`는 toplevel basename이라 worktree가 갈라진다, agentmemory#515).
- **스키마 버전**: `PRAGMA user_version = major*1000 + minor`.
  - 같은 major 안에서는 추가만 하는 마이그레이션(컬럼·테이블 추가, 기본값 필수)만 허용한다. 구버전 바이너리도 계속 읽고 쓸 수 있다.
  - 마이그레이션은 새 바이너리가 처음 열 때 트랜잭션 하나로 실행한다.
  - 모르는 major를 만나면 쓰지 않는다. 훅은 exit 0, 데몬은 시작하지 않고 에러를 남긴다.
- **모델 변경**: 데몬은 현재 모델과 `model_id`가 다른 행을 백그라운드에서 재임베딩한다. 끝날 때까지 벡터 검색은 같은 `model_id`끼리만 비교한다.

## 5. 캡처 (훅 2개 + transcript, 항상 exit 0)
- **원칙**: 도구 호출마다 기록하지 않는다. 턴이 끝날 때 transcript를 **지난번에 읽은 위치(offset) 이후만** 읽어서 기록한다.
  - 두 호스트 모두 훅 입력에 `transcript_path`를 넘긴다(Codex `codex-rs/hooks/src/events/stop.rs:32`).
  - 두 호스트 모두 `"async": true`를 지원한다. 단, Codex는 SessionEnd의 async를 무시하고 동기로 실행한다.

| 이벤트 | 실행 | 동작 |
|---|---|---|
| Stop | async | `ingest` |
| SessionEnd | async (Codex는 동기) | 남은 부분을 `ingest`하고 `ended_at`을 기록한다. |

- 에이전트를 기다리게 하는 훅이 없다.
- **쓰지 않는 이벤트**: SessionStart, UserPromptSubmit, PostToolUse(Failure), PreToolUse, PreCompact, Subagent*, Notification, TaskCompleted. 기록할 내용은 transcript에 다 들어 있고, 주입을 하지 않으므로 SessionStart도 필요 없다.

**ingest 파이프라인** (Stop, SessionEnd 공통)
1. 세션 행이 없으면 만든다(`INSERT OR IGNORE`: id, agent, project, cwd, transcript_path, started_at=첫 ingest 시각). `sessions.transcript_offset`부터 끝까지 읽는다. 마지막 줄이 완결되지 않았으면 그 앞까지만 처리한다.
2. 호스트별 파서(Claude JSONL, Codex rollout JSONL)로 이벤트를 뽑아 관찰로 만든다.

   | transcript 이벤트 | type | title | narrative |
   |---|---|---|---|
   | 사용자 프롬프트 | `conversation` | `prompt` | 프롬프트 |
   | 도구 호출 + 결과 | 도구명 규칙(아래), 실패면 `error` | 도구명 | `input \| output` |
   | 턴의 마지막 답변 | `conversation` | `response` | 답변 |

   - 원본은 프롬프트와 도구를 같은 방식으로 기록하고(`observe.ts:153-160`, `compress-synthetic.ts:8-13`), 답변은 raw에만 둔다(`observe.ts:162`). 우리는 답변도 관찰로 남겨 검색되게 한다. 결론이 답변에 들어 있는 경우가 많다(§8).
   - 제외: 서버 이름이 `agent-memory`인 MCP 도구(자기 호출), `toolsearch`, `listmcpresources`, `fetchmcpresource`(원본 `_capture-filter.ts:1-37`).
3. **마스킹**: `<private>…</private>`는 `[REDACTED]`, 시크릿 16종(정규식 14개 + PEM 개인키 블록 + URL 자격증명, 원본 `privacy.ts:5-31`)은 `[REDACTED_SECRET]`로 바꾼다. 원본과 달리 마스킹을 자르기보다 먼저 한다(잘린 경계에 걸린 비밀값 방지).
4. **규칙 압축** (원본 `compress-synthetic.ts:75-99`)
   - 도구 출력은 8000자에서 자르고 `\n[...truncated]`를 붙인다(원본 `_capture-filter.ts:56-73`). base64 이미지(`data:image/`, `iVBORw0KGgo`, `/9j/`)는 `[image data extracted]`로 바꾼다(원본 `observe.ts:27`).
   - type: 도구명 단어 매칭. web_fetch → search → command_run → file_edit → file_write → file_read → subagent, 그 외 other(원본 `compress-synthetic.ts:30-37`)
   - title 80자 / subtitle: 입력 120자 / narrative 400자(빈 부분 생략)
   - files: 입력의 `file_path, filepath, path, filePath, file, notebook_path` 키 값. 검색 도구면 `path`는 제외, 512자 이상은 버린다.
5. **트랜잭션 하나로** observations INSERT(`(session_id, seq)` UNIQUE로 멱등), FTS 갱신, `transcript_offset` 갱신, `index_queue` 등록을 처리한다.
6. 마지막 auto-forget이 1시간보다 오래됐으면 이어서 실행한다.
7. **파싱 실패 시**: 해당 줄은 건너뛰고 로그를 남긴다. 파서가 통째로 실패하면 훅 입력의 `last_assistant_message`만이라도 `response` 관찰로 기록한다. transcript 형식은 공식 계약이 아니므로 실제 샘플을 픽스처로 둔다.

**auto-forget** (원본 `auto-forget.ts`, 1시간 주기, Stop ingest가 실행)
- `forget_after`가 지난 memory를 삭제한다. `add_memory`의 `ttlDays`로 설정한다(원본 `remember.ts:182-184`).
- 같은 concept를 공유하는 최신 memory끼리 Jaccard > 0.9이면 오래된 쪽을 `is_latest=0`으로 바꾼다(원본 `auto-forget.ts:88-120`). Jaccard 토큰은 lindera로 나눈다.
- 원본의 "180일 지나고 importance ≤ 2인 관찰 삭제"는 넣지 않는다. LLM 없이는 importance가 늘 5라 동작하지 않는다.

## 6. 검색 (데몬)
- **BM25**: FTS5 `bm25()`, k1=1.2, b=0.75(원본과 같음). 검색어도 lindera로 나눈다.
- **벡터**: fastembed-rs + **EmbeddingGemma-300m Q4**, MRL로 256차원. 다국어 모델이라 한글을 처리한다(원본 로컬 기본 모델 all-MiniLM-L6-v2는 영어 전용, agentmemory#725).
  - 대상: 관찰 `title + " " + narrative`(원본 `observe.ts:390-394`), memory `title + " " + content`(원본 `remember.ts:215`).
  - f32 BLOB에 저장하고 brute-force cosine으로 계산한다. sqlite-vec은 쓰지 않는다.
- **결합**: 가중 RRF, K=60, BM25 0.4 / 벡터 0.6(원본 `hybrid-search.ts:20,30-31`). 한 세션에서 최대 3개만 넣고, 결과가 limit보다 적으면 다시 채운다(`:288-295`).
- **대체(supersede)**: `add_memory` 때 실행한다(원본 `remember.ts:95-196`).
  - 후보: **memories만** 대상으로 BM25 상위 50개. 같은 project이거나 project가 없는 memory만 비교한다. 원본은 관찰과 색인을 공유해서 후보 50개를 관찰이 차지한다.
  - Jaccard(lindera 토큰) > 0.7이면 version+1, parent_id와 supersedes를 연결하고, 이전 기록은 is_latest=0으로 바꾼 뒤 색인에서 지운다.
  - 0.4~0.7이면 저장하고 `similarTo`로 비슷한 기록을 알려준다.

## 7. MCP 도구 (3개)
| 도구 | 입력 | 동작 |
|---|---|---|
| `search_memory` | `query`, `limit`(기본 10), `format`(`compact`·`full`, 기본 `full`) | observations와 memories를 하이브리드 검색한다. compact는 id·type·title만 준다 |
| `add_memory` | `content`, `type`, `concepts?`, `files?`, `ttlDays?` | memory를 저장하고 supersede를 판단한다. 결과에 `superseded`·`similarTo`를 준다 |
| `delete_memory` | `id` | memory를 삭제하고 색인과 임베딩도 지운다. 없는 id면 에러를 준다 |

- 이름은 메모리 MCP들(mem0, OpenMemory)의 동사_명사 형태를 따른다. 원본 이름(`memory_recall`·`memory_save`·`memory_governance_delete`)과 다르다(§8).
- 원본의 `memory_smart_search`, `memory_sessions`, `memory_timeline`, `memory_file_history`는 넣지 않는다. `search_memory` 하나로 대신한다.

## 8. 원본과 다르게 가는 부분
| # | 원본 | 우리 | 이유 |
|---|---|---|---|
| 1 | iii-engine + Node 워커, 훅이 HTTP로 보내고 실패하면 스풀 | 훅은 SQLite에 직접, 데몬은 memories 쓰기·검색·임베딩 | 별도 엔진과 스풀이 필요 없다. 엔진 RAM 문제(agentmemory#1312)도 피한다 |
| 2 | KV + 메모리 BM25(부팅마다 재빌드), 한글은 어절 단위 + prefix 확장, 동의어 | SQLite FTS5 + lindera 형태소 분석. prefix·동의어 없음 | 영속 색인이고 조사 처리는 형태소 분석이 맡는다. 의미 유사성은 벡터가 맡는다 |
| 3 | 훅 12개, 도구 호출마다 전송, Stop마다 세션 종료 처리 | 훅 3개 + 턴마다 transcript를 offset부터 ingest | 턴당 프로세스 1개, 멱등이 단순하다. Stop마다 요약이 첫 턴에 고정되는 문제(agentmemory#745)가 없다. 대가로 transcript 파서가 2개 필요하다 |
| 4 | 턴의 마지막 답변은 raw에만 보관 | 답변도 관찰로 남겨 색인 | 결론이 답변에 들어 있는 경우가 많다 |
| 5 | SessionStart 주입(기본 꺼짐), PreCompact 주입(기본 켜짐, 1500토큰) | 주입 없음, SessionStart·PreCompact 훅 없음 | 과거 기록은 검색으로 꺼낸다. 토큰 비용 문제(agentmemory#143)가 없다 |
| 6 | supersede 후보가 관찰과 섞임, auto-forget 180일 규칙 | memories만 후보, 180일 규칙 없음 | 원본에서 동작하지 않는 부분이다 |
| 7 | project = toplevel basename | git common-dir 기준 | worktree 분리(agentmemory#515) |
| 8 | 도구 54개, 원본 이름 | 3개, 동사_명사 이름 | 직관성. 나머지는 검색으로 대신한다 |
| 9 | 원본 Codex 훅에 SessionEnd 없음 | Codex에도 SessionEnd 등록 | 최신 Codex가 지원한다 |
| 10 | — | 제외: LLM 기능(압축, 요약, 4-tier, reflect), graph 스트림, lessons, 슬롯, 프로필, 이미지 보관, 팀·mesh·viewer | 학습용 MVP 범위 밖이거나 LLM이 필요하다 |

## 9. 플러그인 패키징
```
.claude-plugin/plugin.json      .codex-plugin/plugin.json (skills, mcpServers, hooks: ./hooks/hooks.codex.json)
hooks/hooks.json   hooks/hooks.codex.json
  - 두 파일 모두 Stop(async), SessionEnd(async) 2개만 등록한다
  - statusMessage는 두지 않는다 (원본 Codex는 SessionStart·UserPromptSubmit에 있지만 우리는 그 훅을 등록하지 않는다)
  - 경로 변수: Codex는 PLUGIN_ROOT와 CLAUDE_PLUGIN_ROOT를 모두 주입한다(discovery.rs:265). 경로는 항상 따옴표로 감싼다(agentmemory#477)
.mcp.json → 서버 이름 agent-memory, sh -lc 'exec "${PLUGIN_ROOT:-$CLAUDE_PLUGIN_ROOT}/bin/agent-memory" mcp'
bin/agent-memory  (bash 래퍼, 아래 순서로 실행할 바이너리를 정한다)
  1. AGENT_MEMORY_BIN 환경변수가 있으면 그 경로
  2. ~/.agent-memory/bin/agent-memory-v{VER}가 있으면 그것 (VER는 plugin.json 버전)
  3. 없으면 Release에서 받아 sha256을 검증한 뒤 설치한다
skills/: recall, remember, forget
  - 원본 SKILL.md를 가져와 도구 이름을 search_memory, add_memory, delete_memory로 바꾼다
```
- 구버전 바이너리는 지우지 않는다. 실행 중인 세션이 쓰고 있을 수 있다. `status`가 설치된 버전과 용량을 보여준다.
- Codex Desktop은 플러그인 훅 버그(openai/codex#16430)가 있다. README에 `~/.codex/hooks.json` 수동 병합 방법을 안내한다.

## 10. 배포
1. **릴리스 버전 결정**: semantic-release(main 브랜치)가 다음 버전을 정한다. `scripts/sync-versions.sh`로 `Cargo.toml`과 plugin.json 2개의 버전을 맞추고 태그와 Release를 만든다.
2. **바이너리 빌드**: 태그가 생기면 `release.yml`이 빌드한다.
   - 타깃: aarch64/x86_64-apple-darwin, x86_64/aarch64-unknown-linux-gnu
   - ort는 정적 링크한다. 산출물은 `.tar.gz`와 `.sha256`이다.
3. **마켓플레이스 반영**: Release가 publish되면 `on-release.yml`이 `baleen37/baleen-marketplace/.github/actions/notify@v1`(source: agent-memory)를 실행한다.
   - 필요한 시크릿: `BALEEN_RELEASE_APP_ID`, `BALEEN_RELEASE_APP_PRIVATE_KEY`
4. **마켓플레이스 레포 변경** (별도 PR): `sources.json`에 `"agent-memory": {"repo":"baleen37/agent-memory","paths":["./"]}`를 추가하고, 두 marketplace.json에 항목을 추가한다.
5. **모델 다운로드**: 데몬이 처음 뜰 때 `~/.agent-memory/models`에 받는다. 이때 Gemma 약관 고지를 출력한다.

## 11. 에러 처리
- 사용자 설정용 환경변수는 두지 않는다. 내부·테스트용 3개만 둔다: `AGENT_MEMORY_DIR`(데이터 경로), `AGENT_MEMORY_DISABLE`(재귀 방지), `AGENT_MEMORY_BIN`(로컬 바이너리).
- 훅은 항상 exit 0으로 끝난다. 에러는 `~/.agent-memory/logs/`에 기록한다.
- `AGENT_MEMORY_DISABLE=1`이면 모든 훅이 즉시 종료한다.
- 데몬에 연결할 수 없으면 mcp 도구는 에러를 돌려준다(§3). 훅은 데몬과 무관하게 동작한다.

## 12. 검증
- **사전 측정 (구현 첫 단계)**
  - `embed-ko-dic`을 켰을 때 바이너리 크기 증가분과 사전 로드 시간. Stop 훅(async)에서 견딜 만한지 본다. 과하면 모델처럼 첫 실행 때 다운로드한다.
- **단위 테스트**: 형태소 분석("검색을"→"검색", "검색추천"→"추천"으로 검색되는지), 마스킹(16종, 마스킹 후 자르기), 제외 도구(`mcp__agent-memory__search_memory` 등 자기 호출), 규칙 압축(type·files 규칙), RRF와 세션당 3개, supersede(memories만 후보, 대체된 기록이 검색에서 빠지는지), auto-forget(TTL, concept 공유 Jaccard), ingest가 세션 행을 처음 한 번만 만드는지
- **통합 테스트**
  - Claude와 Codex 훅 페이로드 픽스처를 stdin으로 넣는다.
  - **transcript 파서**: 실제 Claude·Codex transcript 샘플을 픽스처로 둔다. offset 이어 읽기, 마지막 줄이 미완결인 경우, 같은 구간을 두 번 ingest해도 행이 늘지 않는지(멱등) 확인한다.
  - MCP JSON-RPC 왕복(initialize → tools/list → tools/call)을 검증한다. 도구가 3개인지 확인한다.
  - **데몬 경쟁**: 클라이언트 3개를 동시에 띄워 데몬 1개만 뜨는지 확인한다.
  - **버전 교체**: v1 데몬에 v2 클라이언트가 붙으면 교체되는지, v1 클라이언트가 v2 데몬에 붙으면 에러를 받고 데몬이 유지되는지, user_version major가 높은 DB에 구버전 훅이 쓰지 않는지 확인한다.
- **임베딩 테스트**: `#[ignore]`로 분리하고 CI에서는 별도 job으로 돌린다(모델 다운로드 필요).
- **E2E**
  1. `AGENT_MEMORY_BIN=target/debug/agent-memory claude --plugin-dir .`로 세션 1을 돌린다.
  2. 세션 2에서 `search_memory`로 세션 1의 작업이 검색되는지 확인한다.
  3. 같은 시나리오를 Codex로 반복한다.
- **배포 검증**: v0.1.0 Release 후 `AGENT_MEMORY_BIN` 없이 래퍼를 실행해 다운로드 → 체크섬 검증 → 실행까지 되는지 확인한다.

## 다음 단계 (승인 후)
- `me:writing-plans`로 구현 계획을 작성한다. 실행 방식은 그때 고른다.
