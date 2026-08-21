# Lupa File Search — Claude Code plugin

Lets Claude Code find files on your Mac **by name and by document content** — PDF, DOCX/XLSX/PPTX, HWP/HWPX, scanned PDFs (OCR) — with Korean-aware matching, in ~30 ms. Powered by the [Lupa](https://lupa.kr) app's bundled `lupa-search` CLI. Everything runs locally; nothing leaves your Mac.

## Install

1. Install **Lupa** from the Mac App Store (v1.0.2+): https://apps.apple.com/kr/app/lupa/id6789231809
2. In Lupa: register the folders you want searchable (⌘, → 폴더), let it index once.
3. (Optional) Lupa → Settings → **명령줄 도구 설치** to put `lupa-search` on your PATH.
4. Install the plugin — either from your **terminal**:
   ```bash
   claude plugin marketplace add paeyoungpark-web/lupa-claude-plugin
   claude plugin install lupa-file-search@lupa
   ```
   or from **inside a Claude Code session** (slash commands):
   ```
   /plugin marketplace add paeyoungpark-web/lupa-claude-plugin
   /plugin install lupa-file-search@lupa
   ```

Then just ask: *"작년 계약서 중에 위약금 조항 있던 거 찾아줘"*, *"find the screenshot with the error message from last week"*.

## Hermes Agent users

The same skill is packaged for [Hermes Agent](https://github.com/NousResearch/hermes-agent) under `hermes/lupa-file-search`. One line:

```bash
hermes skills install https://raw.githubusercontent.com/paeyoungpark-web/lupa-claude-plugin/main/hermes/lupa-file-search/SKILL.md --category productivity
```

Then just ask Hermes: *"동방 폴더에서 적용성보고서 찾아줘"*.

## What the skill does
- Converts natural-language requests into Lupa query syntax (`n:` name, `c:` content, `f:` folder, `-` exclude, `"exact"`, `.ext`, `종류:`, `크기:`, `날짜:`)
- Calls `lupa-search -n 10 "<query>"` and reads the JSON
- Shows a short ranked table; falls back gracefully when Lupa or the index is missing

## Support
lupa.app.kr@gmail.com · https://lupa.kr/support.html

MIT License.
