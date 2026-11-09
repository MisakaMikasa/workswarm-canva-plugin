---
name: canva
version: 1.0.2
description: "Canva 可画连接器使用规则（必读）：调用任何 mcp_canva_ 工具前，先用 skill_tool 读取本技能完整内容。核心规则：⓪用户给出 canva.com 设计链接时，禁止用浏览器打开（浏览器未登录 Canva）；从链接 canva.com/design/<设计ID>/… 中取出设计ID（DA 开头），直接调用 get-design、get-design-pages 等 Canva 工具；canva.link 短链先用 resolve-shortlink；①merge-designs、commit-editing-transaction 每次调用前都要先向用户说明具体操作并获得明确同意（删页不可恢复）；②用户要求修改现有设计时，不得新建设计；③编辑工具不能在空白页上新增文字——遇到空白页先告诉用户并给出选项（删除空白页/重新生成），等用户选择；④结果与预期不符时停下来汇报，不要擅自尝试替代方案；⑤create-design、generate-image 只返回任务 ID，需自行轮询到完成并给出链接。Canva rules: read this skill via skill_tool before any Canva tool call."
tags: [canva, design, poster, presentation, export]
---

# Canva connector

The Canva tools come from Canva's official MCP server and appear with the
connector prefix (for example `mcp_canva_create-design`). Below, tools are
named without the prefix.

Reply in the user's language. Always end a Canva task by giving the user the
Canva link (edit/view URL) or the export download URL.

## 0. Ground rules (override everything below)

0. **Canva links go to the Canva tools, never the browser.** The browser and
   browser sub-agents are *not* logged in to Canva and only reach a login
   wall; the Canva tools already act as the user's account. For a link like
   `https://www.canva.com/design/DAHaLNzVsL4/FyfxSUutAYnK-CUU4J1FkQ/edit`, the
   design ID is the segment after `/design/` (here `DAHaLNzVsL4`); call
   `get-design` / `get-design-pages` with it. For `canva.link/...` short links
   call `resolve-shortlink` first. Never ask the user to "log in to the
   browser" for Canva work.
1. **Approval before permanent changes.** Before *every* `merge-designs` or
   `commit-editing-transaction` call, tell the user exactly what will happen
   (which pages are added, removed or reordered; which text/images change) and
   wait for an explicit yes. A general "fix it" is not approval for a specific
   page deletion.
2. **Respect "edit, don't create".** If the user asks to change an existing
   design, do not call `create-design`, `generate-design`, `copy-design` or
   `import-design-from-url` to produce a replacement unless they agree to it.
   Each of those leaves an extra design in their Canva account.
3. **Stop and report when evidence says no.** If a check contradicts your plan
   (e.g. `get-design-content` is empty, pages have `is_empty: true`, a tool
   returns an error), stop, tell the user what you found, and offer options.
   Do not try a chain of workarounds on your own.
4. **Know the editing limit.** Editing tools can only change elements that
   already exist on a page. They cannot add new text boxes to a blank page.
5. **Don't leave transactions open.** Commit (after approval) or cancel.

### Blank pages after generation

`create-design` sometimes returns pages with no content. After every
generation, check `get-design-pages`; if any page has `is_empty: true`:

- Tell the user which pages are blank.
- Offer: (a) remove the blank pages with `merge-designs` (after approval);
  (b) regenerate with a shorter, single-language brief and an explicit page
  count; (c) duplicate a filled page with `merge-designs` and then edit its
  text.
- Wait for their choice. Do not insert pages from other freshly generated
  designs without checking that those pages actually have content.

## 1. There is no Canva widget here — poll async jobs yourself

Several tool descriptions say "the UI widget polls … do not poll yourself".
That widget does **not** exist in WorkSwarm. Always poll:

| Started by | Poll with | Pass |
|---|---|---|
| `create-design` | `get-create-design-async-job` | `job_id` + latest `continuation_token` |
| `generate-image` | `get-generate-image-job` | `jobId` |
| `separate-image-layers` | `get-separate-image-layers-job` | `job_id` + latest `continuation_token` |

- Wait `polling_policy.wait_seconds` between polls (default ~5 s for image
  jobs). Never poll faster; it does not speed anything up.
- Tell the user briefly that the design is being generated while you wait.
- Stop on success or failure. On failure, explain `failureType` /
  `failureMessage`; do not silently start a new job.
- For `generate-image` success, include the link with the text
  "Open generated image".
- If a completed `create-design` result has `design.print.is_printable: true`,
  you may mention the Canva Print link (`design.print.print_url`).

## 2. Pick the right creation tool

- **Editable design** (poster, flyer, presentation, doc, social post, resume,
  infographic, website): `create-design`. Write a detailed brief: purpose,
  audience, exact text, tone, colours, size/format, page count (keep
  presentations to 1–5 slides unless asked).
- **A standalone picture** (image, photo, illustration, icon, sticker):
  `generate-image`.
- **On-brand request** (their brand kit, brand colours/fonts/logo, or a brand
  template): `create-design` cannot apply brands. Use `list-brand-kits` /
  `search-brand-templates`, then the legacy flow `generate-design` →
  let the user pick a candidate → `create-design-from-candidate`, or
  `create-design-from-brand-template`.
- **From a public URL** (PDF, PPTX, DOCX, webpage): `import-design-from-url`.
- Ignore internal tools: `get-design-thumbnails`, `track-event-lifecycle`,
  `get-design-candidates` (except when polling a legacy brand job).

## 3. Finding and reading designs

- `search-designs` for the user's designs; `search-folders` /
  `list-folder-items` for folders. Follow `continuation` tokens only if needed.
- A canva.link shortlink must go through `resolve-shortlink` first.
- Read text with `get-design-content`, slide notes with `get-presenter-notes`,
  metadata with `get-design`, page previews with `get-design-pages`.

## 4. Editing — always preview, always ask before saving

1. `start-editing-transaction` → returns `transaction_id` and editable elements.
2. `perform-editing-operations` with the changes (text, images, title, layout).
3. Show the user a preview (`get-design-thumbnail` for the changed pages) and a
   short summary of what changed.
4. Only after the user explicitly approves: `commit-editing-transaction`.
   If they decline: `cancel-editing-transaction`.

Never commit without approval. Never leave a transaction open: commit or
cancel before finishing the task.

`merge-designs` (add/remove/reorder pages, combine designs) also needs
explicit approval of the exact operation first — deleted pages cannot be
recovered. `resize-design` and `copy-design` create new designs and leave the
original unchanged, so they are safe.

## 5. Exporting

1. `get-export-formats` for the design first.
2. `export-design` only with a format (and pages) it supports.
3. Give the user the download URL. Mention it expires; re-export if needed.

## 6. Uploading files

- File already at a public HTTPS URL: `upload-asset-from-url` (images/videos)
  or `import-design-from-url` (documents). Never re-host a private file on a
  third-party site to obtain a URL.
- Local file or generated file: `create-upload-url`, then send **one** HTTP
  POST of the raw file bytes to that URL with header
  `Content-Type: application/octet-stream` (no multipart, no base64). For
  example with the shell tool:

  ```bash
  curl -sS -X POST -H "Content-Type: application/octet-stream" \
       --data-binary @"<local file path>" "<upload_url>"
  ```

  The response contains the media/resource IDs to pass to other tools. Each
  upload URL is single-use and short-lived; on failure request a new one.

## 7. Account, scopes and errors

- `get-logged-in-account-details` shows which Canva account is connected.
- "Missing scopes" or 401/403 errors: ask the user to disconnect and
  reconnect the Canva connector, which signs in again in the browser.
- Some features need a paid plan (resize: Canva Pro; brand templates and
  autofill: Canva Enterprise). If a call fails for plan reasons, say so plainly.
- For "how do I … in Canva" questions, use the `help` tool and relay its
  answer as written.
