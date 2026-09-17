---
name: lupa-file-search
description: Mac의 파일을 파일명·문서 내용·폴더·날짜·크기 조각으로 찾을 때 Lupa CLI(lupa-search)로 검색한다. 사용자가 "찾아줘", "어디 있어", "그 파일", "~한 문서", "스크린샷/계약서/보고서 찾아" 등 파일 탐색을 요청하면 find/grep/mdfind 대신 반드시 이 스킬을 쓴다. HWP·PDF·DOCX·XLSX·PPTX·스캔 PDF(OCR) 내용까지 한국어로 검색되며 30ms급으로 빠르다.
---

# Lupa File Search — Claude Code 스킬

Lupa는 이 Mac의 문서를 파일명과 **내용**까지 미리 인덱싱해 둔 로컬 검색 엔진이다.
`lupa-search` CLI는 앱과 동일한 엔진·랭킹을 쓰고 **JSON**을 낸다. 완전 로컬 — 네트워크 없음.

## 0. 도구 선택 — MCP가 있으면 MCP부터

- **`lupa_search`·`lupa_read` 도구가 보이면 그걸 쓴다.** 셸 CLI와 같은 엔진이고 결과도 같다.
  질의 문법(아래 3절)과 결과 필드는 그대로 적용된다.
- MCP 도구가 없을 때만 아래 CLI를 셸로 실행한다.
- 셸로 실행한 `lupa-search`가 **종료코드 133**(또는 SIGTRAP·`Trace/BPT trap`)으로 죽으면
  에이전트의 셸이 샌드박스(`sandbox-exec`) 안에 있는 것이다 — Aside·Codex 등이 그렇다.
  App Store 샌드박스로 서명된 CLI는 그 안에서 기동이 막힌다. **다시 시도하거나 파일을
  복사해 실행하지 말 것** — 전부 같은 이유로 막힌다. 사용자에게 이렇게 안내하고 멈춘다:
  "이 에이전트에서는 Lupa를 MCP 서버로 등록해야 합니다. Lupa 앱 설정 ▸ 일반 ▸
  '○○에 연결…'(2.0.1+), 또는 에이전트의 MCP 설정에 Command
  `npx -y github:paeyoungpark-web/lupa-mcp`를 추가한 뒤 새 세션을 여세요."

## 1. CLI 찾기 (이 순서로)

```bash
for c in /Applications/Lupa.app/Contents/MacOS/lupa-search \
         "$HOME/Applications/Lupa.app/Contents/MacOS/lupa-search" \
         "$(command -v lupa-search 2>/dev/null)"; do
  [ -x "$c" ] && LUPA="$c" && break
done
```

- App Store판 1.0.2+ 는 앱 번들에 CLI가 동봉돼 있다 (첫 번째 경로).
- 하나도 없으면: "Lupa 미설치 — App Store https://apps.apple.com/kr/app/lupa/id6789231809" 안내 후 종료.
- 인덱스 DB는 CLI가 자동으로 찾는다(앱 그룹 컨테이너). `--db`는 평소 불필요.

## 2. 호출 규약

```bash
"$LUPA" -n 10 "<질의>"
```

- **항상 `-n`(limit)을 명시**한다 — 기본 20은 컨텍스트를 낭비한다. 보통 5~10.
- 파일명만 필요하고 빠르게 훑을 땐 `--names-only`.
- 종료코드: `0` 정상 / `2` 인덱스 없음(앱에서 폴더 등록·인덱싱 필요) / `3` `--content` 본문 없음 / `64` 인자 오류 / `1` 검색 실패.
- 알 수 없는 플래그는 64로 거부되니, 검색어에 `-`로 시작하는 단어가 있으면 따옴표 안에 넣는다.

## 3. 검색 문법 (자연어 → 질의)

| 사용자가 말하면 | 질의 |
|---|---|
| "~에 대한 파일", "~라는 문서" | `단어` (파일명+내용) |
| "파일명에 ~가 들어간" | `n:단어` |
| "내용에 ~가 있는", "~라고 적힌" | `c:단어` |
| "~폴더 안에서" | `f:폴더명` (여러 개면 AND) |
| "~는 빼고" | `-단어` |
| "정확히 ~" | `"정확 구문"` |
| "한글 파일", "PDF" | `.hwp` / `.pdf` / `종류:PDF` |
| "10메가 넘는" | `크기:>10mb` |
| "이번 주에 고친" | `날짜:이번주` (`오늘`·`이번달`·`2026-06`) |
| "~로 시작하는" | `계약*` |

조건은 전부 조합 가능: `f:심사보고서 n:gcrm c:경영검토 .docx 날짜:이번달`
한국어는 2-gram이라 `보호정책`으로 `정보보호정책`이 잡히고, 조사 붙은 `동방을`도 `동방`을 찾는다.
더 많은 변환 예: `references/natural-language-to-lupa-query.md`

## 4. 결과 읽기

```json
{ "query": "...", "total_matches": 3390, "total_matches_capped": true,
  "returned": 10, "elapsed_ms": 42.1,
  "results": [ { "name", "path", "extension", "kind", "size_bytes",
                 "modified", "matched_keywords", "score", "snippet" } ] }
```

- 결과는 `matched_keywords`(맞춘 키워드 수) 큰 순으로 묶여 있다 — 여러 단어를 넣었을 때 **전부 맞는 파일이 위**.
- `total_matches_capped: true`면 총계가 아니라 "2,000 이상"으로 읽는다.
- `snippet`의 `«`…`»`가 일치 구간. 파일명만 맞으면 빈 문자열.
- 사용자에게는 **상위 몇 개만** 이름·경로·수정일(·스니펫) 표로 보여주고, 총 건수와 응답 시간을 한 줄로 덧붙인다.

## 5. 전형적 흐름

1. 요청을 질의로 변환 → 실행 (`-n 10`).
2. 0건이면: 조건을 하나씩 풀어 재시도(`c:` 제거 → 폴더 제거 → 단어 줄이기). 그래도 0건이면 "해당 폴더가 Lupa에 등록됐는지" 안내.
3. 너무 많으면(수백 건): `f:`·`.ext`·`날짜:`로 좁히는 질의를 제안하거나 바로 적용.
4. 찾은 파일로 후속 작업(열기·복사·요약)은 `path`를 그대로 쓴다 — Lupa가 주는 경로는 NFC 정규화된 실제 경로라 그대로 신뢰해도 된다.
5. **본문이 필요하면**(요약·발췌·질문 답변) 먼저 `--content`로 받는다 — HWP·HWPX·스캔 PDF처럼 Read 도구가 못 읽는 파일도 된다:

   ```bash
   "$LUPA" --content "<path>" --max-chars 6000
   ```

   - `content`·`total_chars`·`truncated`가 온다. 잘렸고 더 필요하면 `--max-chars`를 올린다(`0` = 전부). 문맥창을 아끼려면 기본값부터.
   - CLI는 **파일을 열지 않는다** — 인덱싱 때 추출한 텍스트를 돌려준다. 그래서 `modified` 이후 파일이 바뀌었으면 옛 본문일 수 있고, 종료코드 `3`(`content_not_indexed`)이면 그 파일은 본문이 인덱스에 없는 것이다. 그땐 Read 도구로 `path`를 직접 연다(txt·md·pdf 등 읽히는 포맷에 한함).
   - 종료코드 `64` "알 수 없는 옵션: --content"면 Lupa가 1.0.2 이하다 — Read 도구로 대체하고 사용자에게 업데이트를 안내.

## 6. 하지 말 것

- `find`/`grep -r`/`mdfind`로 홈 폴더를 훑지 않는다 — 느리고 한글 내용 검색이 안 된다. Lupa가 있으면 Lupa.
- 결과 20개 이상을 그대로 덤프하지 않는다.
- 인덱싱은 CLI로 못 한다 — 읽기 전용. 폴더 등록·인덱싱은 Lupa 앱(⌘, → 폴더)에서.
