---
name: lupa-file-search
description: Lupa CLI로 Mac의 파일을 파일명·문서 내용·폴더·날짜·크기로 초고속 검색 (한국어, HWP/PDF/Office/OCR 본문 포함, ~30ms)
trigger: 파일 검색, "찾아줘", "어디 있어", "그 파일", "~한 문서"
keywords: [lupa, 파일검색, 내용검색, 한글, HWP, PDF]
requirements:
  - macOS
  - Lupa 1.0.2+ (App Store) + 폴더 인덱싱 완료
  - lupa-search CLI — 앱에 동봉됨. 경로는 아래 "실행" 순서로 자동 탐색
pitfalls:
  - "개발용 저장소 빌드 경로를 고정으로 쓰지 말 것 — 반드시 App Store 동봉 CLI(/Applications/Lupa.app/Contents/MacOS/lupa-search) 또는 PATH의 lupa-search를 쓴다"
  - "index_not_found → Lupa 앱에서 폴더 등록·인덱싱 먼저. 백업 DB를 복원하지 말 것(낡은 인덱스가 만들어진다)"
  - "0건 → 조건을 하나씩 풀어 재시도 (c: 제거 → 폴더 제거 → 단어 줄이기)"
  - "iCloud 파일은 경로가 나와도 로컬에 안 내려온 경우 열 수 없다 — 검색 결과(snippet·modified)로 답하거나 다운로드 안내"
---

# Lupa File Search

## 실행 — CLI를 이 순서로 찾는다
```bash
for c in /Applications/Lupa.app/Contents/MacOS/lupa-search \
         "$(command -v lupa-search 2>/dev/null)"; do
  [ -x "$c" ] && LUPA="$c" && break
done
"$LUPA" -n 10 "<질의>"
```
인덱스 DB는 CLI가 자동으로 찾는다(앱 그룹 컨테이너). `--db`는 쓰지 않는다.
항상 `-n`을 붙인다. 파일명만 급히 볼 땐 `--names-only`.
종료코드: 0 정상 · 2 인덱스 없음 · 64 인자 오류 · 1 검색 실패.

## 자연어 → 질의
| 사용자 말 | 질의 |
|---|---|
| ~에 대한 파일 | `단어` (파일명+내용) |
| 파일명에 ~ | `n:단어` |
| 내용에 ~, ~라고 적힌 | `c:단어` |
| ~폴더 안에서 | `f:폴더명` |
| ~는 빼고 | `-단어` |
| 정확히 ~ | `"구문"` |
| 한글/PDF 파일 | `.hwp` / `.pdf` / `종류:PDF` |
| 10메가 넘는 | `크기:>10mb` |
| 이번 주에 고친 | `날짜:이번주` (오늘·이번달·2026-06) |
| ~로 시작하는 | `계약*` |
조합: `f:심사보고서 n:gcrm c:경영검토 .docx 날짜:이번달`
한국어는 2-gram — `보호정책`으로 `정보보호정책`이 잡히고 조사(`동방을`→`동방`)도 처리.

## 결과 JSON
`total_matches`(capped면 2,000 이상) · `returned` · `elapsed_ms` ·
`results[]{name,path,extension,kind,size_bytes,modified,matched_keywords,score,snippet}`
- `matched_keywords` 큰 순 — 여러 단어 넣으면 전부 맞는 파일이 위
- `snippet`의 «…»가 일치 구간. "문서에 뭐라고 적혔나"는 `c:`로 검색해 snippet으로 답한다
- 사용자에겐 상위 몇 개만 이름·경로·수정일로, 총 건수·ms 한 줄

## 하지 말 것
- find / grep -r / mdfind로 홈을 훑지 않는다. Lupa가 있으면 Lupa.
- 결과 20개 이상 덤프하지 않는다.
- 인덱싱·복원은 CLI로 하지 않는다(읽기 전용). 폴더 등록은 Lupa 앱에서.
