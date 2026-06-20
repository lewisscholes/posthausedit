# CLAUDE.md — The Post Haus (read this first, every session)

This is the working brief for the Post Haus app. Read it before changing anything.
Its #1 job is to stop changes that silently undo previous fixes.

## What this is
The Post Haus — a TikTok-Shop affiliate content service. Creators log in, upload raw
clips + attachments; the editor (Aaron) edits them; finished videos go to the creator's
"To post" side; the creator downloads and posts to TikTok themselves and tags the product.
Tagline: "Creators create, we edit."

## Where everything lives (hosting)
- **Front-end / the whole app:** `client/index.html` — ONE large single file (~500KB+). Served by **GitHub Pages** from this repo (`lewisscholes/posthausedit`).
- **Domain:** posthausedit.com -> GitHub Pages via the `CNAME` file + DNS. Root `index.html` just redirects to `/client/`.
- **Data + video storage:** **Supabase** (tables like `ed_videos`, `ed_send_jobs`, `ed_video_downloads`, `temp_files`; private storage buckets via signed URLs). The app uses the public **anon key** and RLS is currently open.
- **Heavy work:** a Windows "workhorse" laptop runs Python workers (render + send). The app only QUEUES jobs; the workers do the actual ffmpeg rendering and TikTok delivery.
- One app, two roles: **admins** log in with a workspace code (see the Bank/editor), **creators** with an invite code (see upload / To post).

## The data is NOT in the code
Clients' uploaded clips and ready-to-post videos live in **Supabase**, not in this repo.
Editing/deploying `index.html` cannot delete or move them. Never do a destructive DB
migration to achieve a UI change.

## NON-NEGOTIABLE RULES (these are why fixes keep reverting)
1. **Never regenerate or re-upload the whole file.** Edit surgically and commit small **diffs**. Whole-file replacement silently wipes earlier fixes (it has erased the home-screen icon twice). If you must rewrite a big chunk, diff it against `git` first and confirm nothing else changed.
2. **ASCII-only inside `<script>`.** Non-ASCII chars (em dash, curly quotes, `£`, `·`, `…`, box-drawing, emoji) can cause a **silent parse failure** on GitHub Pages and break the whole app. Use `--` for em dash, `...` for ellipsis, `String.fromCharCode(...)` / `String.fromCodePoint(...)` for symbols. (HTML/CSS outside `<script>` can have a few, but don't add new ones.)
3. **Do not touch the render pipeline** unless that's the explicit task: `edit_state`, overlays, crop/zoom, trims, and the worker's rendering. The editor preview (`mteOvParts` / `mteSmoothBox`) and the render worker must stay in LOCKSTEP — change one, change the other, or burned videos stop matching the preview.
4. **Secrets stay server-side.** Never put the Supabase `service_role` key, the Anthropic key, or the PostPeer key in `client/index.html`. Only the public anon key belongs there.

## The Bank (admin video bank) — status model
Buckets in `bankBuckets()`: a clip stays in **New** (uploaded, even while being edited) and
only moves to **Posted** once scheduled/sent to the creator. `downloaded` = the creator
saved it. The `edited` bucket is intentionally empty/hidden ("no Edited limbo").
Current admin tabs (keep these keys; labels only): **To do** (`new`) · **Sent to Post** (`posted`) · **Downloaded** (`downloaded`).

## Keep these (they've been reverted before)
- **Home-screen icon:** `<link rel="apple-touch-icon" sizes="180x180" href="posthaus-icon-180.png">` and the 512 `rel="icon"` in `<head>`. Files `posthaus-icon-180.png` / `posthaus-icon-512.png` live in `client/`. Re-check these survive after any big change.
- **Brand:** red `#D7342C`, cream `#F4ECDC`, warm black `#1A1512`. (Some old azure `--blue:#0ea5e9` may linger in CSS — don't reintroduce blue UI.)

## Workflow (do this every change)
1. `git pull` first (get the latest — avoids clobbering anyone else).
2. Make the smallest edit that does the job.
3. **Verify before deploy:** extract the inline `<script>` and run `node --check`; confirm no NEW non-ASCII was added to the script.
4. Review the `git diff` — it should contain ONLY your intended change.
5. Commit with a clear message and `git push`. GitHub Pages redeploys in ~60-90s (atomic swap; the old version serves until the new one is ready, no downtime).
6. If anything looks wrong live, `git revert` + push restores the previous version in ~90s.

## Quick script syntax check
```bash
python3 - << 'EOF'
import re
s=open('client/index.html',encoding='utf-8').read()
js="\n;\n".join(re.findall(r'<script(?![^>]*\bsrc=)[^>]*>(.*?)</script>', s, re.S))
open('/tmp/_appcheck.js','w').write(js)
print("non-ASCII in script:", sorted(set(re.findall(r'[^\x00-\x7F]', js))))
EOF
node --check /tmp/_appcheck.js && echo "JS OK"
```
