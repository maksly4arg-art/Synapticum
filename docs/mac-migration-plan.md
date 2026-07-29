# Mac Studio Migration Plan - Synapticum

Status: ACTIVE (Mac Studio M3 Ultra arrived 2026-06-05). Inventory re-verified vs live C:\ scan 2026-06-06.
Storage decision: single Samsung 990 PRO 4TB in ORICO USB4 (NO stripe for now). 2nd NVMe = photo disk; added to the working set later only if 4TB runs out.

**PIVOT 2026-06-07 (OVERRIDES the storage decision above + every `/Volumes/Synapticum` path below):** Main memory + project now go on the Mac **INTERNAL SSD**, root `~/Synapticum/`. Reason: everything is tiny (Vault 46 MB, all DBs a few MB) so internal is fastest and free; AND the external 990 PRO in ORICO OVERHEATS at idle - the DRIVE itself runs boiling, not the bridge chip (disabling Spotlight did NOT help; the USB4 bridge does not pass NVMe low-power states, so the drive never idles down). External NVMe + DAS DEFERRED: DAS ships too slow to wait on; the external disk will be diagnosed from the Mac later. So everywhere below read `/Volumes/Synapticum/` as `~/Synapticum/`. Move to external only when internal space runs low. First transport is deliberately minimal: install Claude Code + pull Vault (ZIP from GitHub is fine, no git dance) -> Builder boots on Mac and drives the rest.

## Target layout
- Project root = external NVMe, volume name `Synapticum` -> `/Volumes/Synapticum/`
- Mirror the Windows `C:\` root onto the NVMe root:
  `/Volumes/Synapticum/{ClaudeClaw,Vault,Shared,Archivarius,Bob,Alice,Jesse,Eva,Analyst,Joe,Aleks,Finn,Backups}`
- Apps (Claude Desktop, Obsidian, Node, etc.) -> internal SSD (standard).
- telegram-memory -> `/Volumes/Synapticum/telegram-memory`, then `ln -s` from `~/telegram-memory` so the `~`-based load.sh/save.sh keep working.

## From GitHub (clone) vs carry by hand

GitHub (maksly4arg-art) - 10 repos, clone fresh:
  ClaudeClaw, Vault, Archivarius, Analyst, Eva, Jesse, Alice, Joe, Aleks, Finn

NOT in git - COPY MANUALLY (precious, non-recoverable):
- `Bob/`    - entire dir (not a git repo)
- `Shared/` - entire dir (a2a-server, agent-name-filter, video-note-handler, report-engine, vault-briefs, vault-lessons, a2a-autochain, a2a-signal). Wired into 7 agents.
- All `.env` secrets - see bundle
- SQLite memory/state DBs - see bundle
- Chrome `Default` profile + unpacked X-comment extension (live X session)

Transfer method: exFAT USB stick, or LAN (Finder -> Go -> Connect to Server -> `smb://<windows-ip>`). Bundle is tiny if you exclude `node_modules` (rebuilt on Mac).

### Secrets bundle (.env - NEVER in git)
```
C:\ClaudeClaw\.env
C:\Archivarius\.env
C:\Bob\.env
C:\Alice\.env
C:\Jesse\.env
C:\Eva\.env
C:\Analyst\.env
C:\Joe\.env
C:\Shared\.env.moltbook
C:\Shared\.env.x
```
(Aleks/Finn have only templates - no live secrets.)
On Mac: rewrite any `C:\...` paths inside these to `/Volumes/Synapticum/...`

### Memory / state DBs bundle
```
C:\Users\MLpho\telegram-memory\conversations.db   (+ load.sh, save.sh, memory.py)
C:\Archivarius\store\archivarius.db                (1.4 MB - central context store)
C:\ClaudeClaw\store\*.db                           (bob.db, claudeclaw.db, ...) + whole store/
per-agent store DBs (Eva eva.db, Joe joe.db, Jesse jesse.db, ...)
```
NOTE: Jesse also has malformed dupes `Jessestorejesse.db`, `Jessestorepolymarket.db`, `Archivariusstorearchivarius.db` (old path-join bug, missing separators) - IGNORE them, real DBs live in `Jesse/store/`.

### Other C:\ dirs - triage (verified 2026-06-06, not yet phased)
Outside the Synapticum agent set. Decide per dir:
- `C:\Reports\{alice,eva,freddy,jesse,joe}\` - agent report outputs (not git). MIGRATE by hand; reconcile vs `Vault/reports/`.
- `C:\Raw\` (README,_INDEX,_shared,_meta_template) - structured raw/source store. Check what reads it, likely MIGRATE.
- `C:\Максим\` (`.canvas`, Операционная доска) - personal ops board. MIGRATE if still used.
- `C:\Marketplace WB+Ozon\` - SEPARATE business project (WB/Ozon cards, labels, xls). Not Synapticum - migrate only if worked on Mac.
- `C:\Archive\reports\` - old archived reports. LOW priority / skip unless needed.
- `C:\Tenorshare\`, `C:\tmp\`, `$Recycle.Bin`, Windows system dirs - SKIP.

### Drag-folder staging script (run at CUTOVER, agents STOPPED via launcher)
Bundles everything NOT in git into one folder to drag to the Mac. No node_modules -> stays tiny. PowerShell:
```
$dst = "C:\_MAC_MIGRATION"
# precious non-git dirs (exclude node_modules/dist/.git)
robocopy C:\Bob    "$dst\Bob"    /MIR /XD node_modules dist .git /NFL /NDL /NJH /NJS
robocopy C:\Shared "$dst\Shared" /MIR /XD node_modules dist .git /NFL /NDL /NJH /NJS
robocopy C:\Users\MLpho\telegram-memory "$dst\telegram-memory" /MIR /XD node_modules /NFL /NDL /NJH /NJS
# .env secrets + store DBs per agent (stop agents first so SQLite is consistent)
foreach ($a in 'ClaudeClaw','Archivarius','Bob','Alice','Jesse','Eva','Analyst','Joe','Aleks','Finn') {
  New-Item -ItemType Directory -Force "$dst\$a" | Out-Null
  Copy-Item "C:\$a\.env" "$dst\$a\.env" -ErrorAction SilentlyContinue
  if (Test-Path "C:\$a\store") { robocopy "C:\$a\store" "$dst\$a\store" *.db /NFL /NDL /NJH /NJS }
}
Copy-Item C:\Shared\.env.* "$dst\Shared\" -ErrorAction SilentlyContinue
# Claude auto-memory + CLAUDE.md
robocopy "C:\Users\MLpho\.claude" "$dst\.claude" /MIR /XD node_modules /NFL /NDL /NJH /NJS
Copy-Item "C:\Users\MLpho\CLAUDE.md" "$dst\CLAUDE.md" -ErrorAction SilentlyContinue
Write-Host "Bundle ready at $dst - drag to /Volumes/Synapticum/ on the Mac, then distribute per Phase 2-3"
```

## Phases

### Phase 0 - Toolchain (internal SSD, ~30 min)
- [ ] `xcode-select --install`   (git + compilers)
- [ ] Install Homebrew
- [ ] `brew install node@24 gh python@3.13 python-tk ffmpeg`
- [ ] Install Claude Code CLI + `claude` login
- [ ] Install Claude Desktop app + login

### Phase 1 - MEMORY ONLINE FIRST (Builder runs on Mac with full context, then drives the rest)
- [ ] Plug NVMe into ORICO -> Disk Utility -> Erase: APFS, scheme GUID Partition Map, name `Synapticum`
- [ ] `sudo pmset -a disksleep 0`   (never let the working disk sleep)
- [ ] `gh auth login`
- [ ] `git clone .../Vault.git /Volumes/Synapticum/Vault`   (knowledge graph = my context)
- [ ] Carry `.claude` auto-memory files + `~/CLAUDE.md` + project `CLAUDE.md` to Mac
- [ ] Install Obsidian -> open vault `/Volumes/Synapticum/Vault`
- [ ] Launch Claude Code in `/Volumes/Synapticum` -> Builder live with memory

### Phase 2 - Code
- [ ] Clone the remaining 9 repos into `/Volumes/Synapticum/`
- [ ] Copy `Bob/` and `Shared/` (not git) from Windows
- [ ] Copy all `.env`; rewrite `C:\` paths -> `/Volumes/Synapticum/`
- [ ] In each project: `npm install` (REBUILD native modules for arm64 - do NOT copy node_modules)
- [ ] `npx tsc` where needed (Eva, Joe run from `dist/`)

### Phase 3 - Memory DBs + runtime
- [ ] Copy archivarius.db, ClaudeClaw/store DBs, per-agent DBs into matching `store/` dirs
- [ ] Copy telegram-memory (conversations.db + scripts); `ln -s` `~/telegram-memory` -> NVMe
- [ ] Smoke-test: load.sh reads history; agent reads its DB

### Phase 4 - Launcher + CUTOVER
- [ ] Port `launcher.pyw` to macOS:
  - `CREATE_NO_WINDOW` / `creationflags` -> remove (Windows-only)
  - `taskkill /PID /T /F` -> `os.kill` / `kill` (posix process tree)
  - `C:\` paths -> `/Volumes/Synapticum/`
  - tkinter runs on Mac as-is
- [ ] Test ONE agent end-to-end (e.g. Eva) before the rest
- [ ] !!! CLEAN CUTOVER !!! Stop the Windows launcher (ALL agents) BEFORE starting the Mac launcher.
  One Telegram bot token = one getUpdates poller. Both running = 409 conflict, both break.
- [ ] Start Mac launcher -> verify each agent replies in Telegram

### Phase 5 - Peripherals
- [ ] Chrome: install, log into X, load unpacked X-comment extension into Default profile, re-grant `chrome.debugger`
- [ ] Backups: `synapticum_backup.ps1` -> either `brew install --cask powershell` (keep .ps1) or rewrite as bash + launchd
- [ ] Telegram Desktop (monitor agents)
- [ ] Verify ffmpeg (video notes) + curl (Whisper)

## Software install list
| Tool | Why | Install |
|------|-----|---------|
| Xcode CLT | git, compilers | `xcode-select --install` |
| Homebrew | package manager | brew.sh script |
| Node 24 + npm | all TS agents | `brew install node@24` |
| gh | clone 10 private repos | `brew install gh` |
| Python 3.13 + tk | launcher.pyw | `brew install python@3.13 python-tk` |
| ffmpeg | video-note-handler | `brew install ffmpeg` |
| Claude Code CLI | Builder delegation | npm / official installer |
| Claude Desktop | Builder (me) | claude.ai/download |
| Obsidian | the Vault | obsidian.md |
| Telegram Desktop | monitor agents | telegram.org |
| Google Chrome | X-comment extension | google.com/chrome |
| VS Code | build/debug agents | `brew install --cask visual-studio-code` |
| DB Browser SQLite | inspect memory (optional) | `brew install --cask db-browser-for-sqlite` |
| PowerShell (pwsh) | only if keeping .ps1 backup | `brew install --cask powershell` |

## Gotchas (read before cutover)
1. node_modules: rebuild via `npm install`, never copy (Windows binaries crash on arm64).
2. Bot tokens: one poller per bot - clean cutover, never run Win + Mac agents together.
3. Secrets are NOT in git - the .env bundle must be hand-carried.
4. Bob + Shared are NOT git - hand-carry whole dirs.
5. Disk sleep: `pmset disksleep 0`, else agents drop when the NVMe naps.
6. launcher.pyw is Windows-coupled (taskkill, CREATE_NO_WINDOW) - a real port, not a copy.
7. Path sweep is LIGHT: only ~4 hardcoded `C:\` in ClaudeClaw/src; most paths are env-driven. Fix .env paths + launcher.
