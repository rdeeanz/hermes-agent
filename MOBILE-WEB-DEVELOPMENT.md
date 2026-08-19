# Hermes Agent — Analisa Codebase & Strategi Pengembangan Mobile Web

> Dokumen ini hasil analisa langsung terhadap source code di repository ini.
> Setiap klaim teknis dirujuk ke file dan baris yang bisa diverifikasi.
>
> Tanggal analisa: 19 Agustus 2026 · Commit: `13ce0c5c` · Versi: `hermes-agent 0.20.4`

---

## Daftar Isi

1. [Ringkasan Eksekutif (Jawaban Singkat)](#1-ringkasan-eksekutif-jawaban-singkat)
2. [Codebase Ini Apa?](#2-codebase-ini-apa)
3. [Tech Stack Lengkap](#3-tech-stack-lengkap)
4. [Arsitektur Sistem](#4-arsitektur-sistem)
5. [Cara Menjalankan](#5-cara-menjalankan)
6. [Jawaban Jujur: Bisa Dideploy di VPS & Dibuka di Browser Smartphone?](#6-jawaban-jujur-bisa-dideploy-di-vps--dibuka-di-browser-smartphone)
7. [Analisa Gap Detail](#7-analisa-gap-detail)
8. [Strategi & Konsep Teknis Pengembangan](#8-strategi--konsep-teknis-pengembangan)
9. [Panduan Deploy VPS (Fase 0 — Bisa Dikerjakan Hari Ini)](#9-panduan-deploy-vps-fase-0--bisa-dikerjakan-hari-ini)
10. [Risiko, Trade-off, dan Catatan Keamanan](#10-risiko-trade-off-dan-catatan-keamanan)
11. [Roadmap & Estimasi Effort](#11-roadmap--estimasi-effort)

---

## 1. Ringkasan Eksekutif (Jawaban Singkat)

**Pertanyaan: Apakah codebase ini bisa dideploy di VPS dengan tampilan web dan dibuka di browser smartphone?**

**Jawaban jujur: YA, BISA — dan lebih siap dari yang Anda mungkin duga.** Tapi ada satu gap besar yang menghalangi tujuan spesifik Anda ("UI/UX seperti aplikasi Claude di smartphone").

Rinciannya:

| Kebutuhan Anda | Status | Bukti |
|---|---|---|
| Ada tampilan web (bukan cuma terminal) | ✅ **SUDAH ADA** | `web/` — SPA React 19 penuh dengan 19 halaman |
| Bisa dideploy di VPS | ✅ **SUDAH ADA** | `hermes dashboard --host 0.0.0.0`, `docker-compose.yml` |
| Aman diekspos ke internet | ✅ **SUDAH ADA** | Auth gate wajib, fail-closed (`web_server.py:641`) |
| Bisa dibuka di browser smartphone | ✅ **SUDAH BISA** | Viewport meta, drawer mobile, safe-area, keyboard inset |
| Semua fitur & pengaturan diakses via browser | ✅ **SUDAH ADA** | 18 halaman non-chat: Config, Keys, Cron, Skills, MCP, Channels, dll. |
| Chat dengan UI/UX seperti app Claude | ❌ **BELUM** | Tab Chat = emulator terminal xterm.js, bukan chat bubble |
| Terpasang seperti app (PWA / add to home screen) | ❌ **BELUM** | Tidak ada `manifest.json`, tidak ada service worker |
| HTTPS bawaan | ❌ **BELUM** | Tidak ada opsi `--ssl-certfile`; wajib reverse proxy |

**Kesimpulan strategis:** Anda **tidak perlu membangun web UI dari nol**. Yang perlu dikerjakan adalah **3 pekerjaan terarah**:

1. **Deploy + hardening** (0.5–1 hari) → langsung bisa dipakai dari HP hari ini.
2. **Ganti tab Chat** dari mirror terminal xterm.js menjadi **native chat UI** — dan komponennya **sudah ada di repo ini** (`apps/desktop/src/components/assistant-ui/`) yang sudah bicara ke backend yang sama via WebSocket JSON-RPC. Ini pekerjaan **porting**, bukan pekerjaan bikin baru.
3. **Lapisi PWA** (2–3 hari) → bisa "Add to Home Screen", fullscreen, terasa seperti app native.

---

## 2. Codebase Ini Apa?

### Identitas

**Hermes Agent** — AI agent otonom yang bisa memperbaiki dirinya sendiri (*self-improving*), dibuat oleh **[Nous Research](https://nousresearch.com)**.

- Lisensi: **MIT** (`LICENSE`)
- Repo upstream: `github.com/NousResearch/hermes-agent`
- Versi Python package: `0.20.4` (`pyproject.toml:4`)
- Skala: **±9.700 file**, `run_agent.py` sendiri 419 KB, `hermes_state.py` 590 KB, `cli.py` 967 KB
- Test suite: **3.104 file test Python** (`tests/`) + suite Vitest untuk JS/TS

Ini **bukan project kecil atau eksperimen**. Ini produk matang dengan CI, security policy (`SECURITY.md`), dokumentasi multi-bahasa (Inggris, Mandarin, Urdu, Spanyol), dan riwayat hardening keamanan yang terdokumentasi di komentar kode.

### Apa yang Dilakukannya

Hermes Agent adalah **agent loop** yang memanggil LLM secara berulang dengan akses ke tool-tool nyata (shell, file, browser, HTTP, dsb). Yang membedakannya dari agent lain:

| Kemampuan | Penjelasan |
|---|---|
| **Learning loop tertutup** | Agent membuat *skill* sendiri setelah menyelesaikan tugas kompleks, lalu menyempurnakannya saat dipakai. Skill = memori prosedural berbentuk file Markdown + script. |
| **Memori persisten** | Profil user yang makin dalam antar sesi. Pencarian full-text SQLite FTS5 atas seluruh percakapan lampau (`hermes_state_search.py`, 114 KB). Integrasi [Honcho](https://github.com/plastic-labs/honcho) untuk *dialectic user modeling*. |
| **Multi-platform messaging** | Satu proses gateway melayani **22 platform**: Telegram, Discord, Slack, WhatsApp, Signal, Matrix, Teams, Email, IRC, LINE, Feishu, DingTalk, WeCom, Google Chat, Mattermost, SMS, Home Assistant, ntfy, dll. (`plugins/platforms/`) |
| **Cron scheduler bawaan** | Tugas terjadwal dalam bahasa natural, hasilnya dikirim ke platform mana pun (`cron/`). |
| **Subagent & paralelisasi** | Spawn subagent terisolasi untuk workstream paralel (`delegate_task`). |
| **7 terminal backend** | Local, Docker, SSH, Singularity, Modal, Daytona, Vercel Sandbox. Modal & Daytona mendukung *serverless hibernation* — biaya nyaris nol saat idle. |
| **Provider-agnostic** | Nous Portal, OpenRouter, OpenAI, Anthropic, Azure, GitHub Copilot, endpoint sendiri. Ganti model tanpa ubah kode. |
| **124 tool** | `tools/` berisi 124 modul Python tool, diorganisir lewat sistem *toolset* (`toolsets.py`). |
| **MCP (Model Context Protocol)** | Bisa jadi MCP client maupun MCP server (`mcp_serve.py`). |
| **Research-ready** | Batch trajectory generation + kompresi trajectory untuk melatih model tool-calling generasi berikutnya (`batch_runner.py`, `trajectory_compressor.py`). |

### Surface / Antarmuka yang Tersedia

Ini bagian **paling penting** untuk pertanyaan Anda. Hermes punya **empat** antarmuka terpisah:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  1. CLI / TUI          `hermes`                                          │
│     Terminal UI (Ink/React di terminal) — ui-tui/                        │
│                                                                          │
│  2. Messaging Gateway  `hermes gateway start`                            │
│     22 platform chat (Telegram, Discord, WhatsApp, ...) — gateway/        │
│                                                                          │
│  3. Web Dashboard      `hermes dashboard`      ◄── INI YANG ANDA BUTUH    │
│     SPA React 19 + FastAPI, default 127.0.0.1:9119 — web/                 │
│                                                                          │
│  4. Desktop App        Electron                                          │
│     Native chat UI (assistant-ui) via JSON-RPC — apps/desktop/            │
│     ◄── INI SUMBER KOMPONEN UNTUK MEMPERBAIKI GAP CHAT                   │
└──────────────────────────────────────────────────────────────────────────┘
```

Plus **`hermes serve`** — backend headless (JSON-RPC/WebSocket, tanpa SPA) yang dipakai desktop app dan client remote (`hermes_cli/subcommands/dashboard.py`).

---

## 3. Tech Stack Lengkap

### 3.1 Backend — Python

| Komponen | Teknologi | Versi | Catatan |
|---|---|---|---|
| Runtime | **Python** | `>=3.11,<3.14` | Batas atas load-bearing: 3.14 belum ada wheel cp314 untuk pydantic-core (`pyproject.toml:14`) |
| Package manager | **uv** (Astral) | — | Rust-based; dibundel oleh installer |
| Web framework | **FastAPI** | `0.133.1` | Extra `web` (`pyproject.toml:334`) |
| ASGI server | **uvicorn[standard]** | `0.41.0` | Dipanggil via `uvicorn.Server` API langsung, bukan `uvicorn.run()` |
| ASGI toolkit | **Starlette** | `1.3.1` | |
| Validasi | **Pydantic** | `2.13.4` | Dinaikkan dari 2.12.5 karena segfault pydantic-core di thread non-main |
| LLM SDK | **openai** | `2.24.0` | Client utama (OpenAI-compatible); `anthropic==0.87.0` sebagai extra |
| Terminal UI | **prompt_toolkit** + **rich** | `3.0.52` / `14.3.3` | |
| CLI parsing | **fire** + **argparse** | `0.7.1` | |
| HTTP client | **httpx[socks]** | `0.28.1` | |
| Scheduler | **croniter** | `6.0.0` | |
| Database | **SQLite** + **FTS5** | built-in | Ekstensi C custom untuk tokenisasi CJK: `native/fts5_cjk/fts5_cjk.c` |
| Crypto | **cryptography** | `50.0.0` | Floor keamanan: CVE-2026-69247 |
| Auth token | **PyJWT[crypto]** | `2.13.0` | |
| Upload | **python-multipart** | `0.0.32` | Untuk file manager dashboard |
| Proses | **psutil** | `7.2.2` | Cross-platform, termasuk Windows |
| WebSocket | **websockets** | `15.0.1` | |
| Config | **PyYAML** + **ruamel.yaml** | `6.0.3` / `0.18.17` | ruamel untuk round-trip yang mempertahankan komentar |

**Catatan penting soal dependency policy:** semua direct dependency **di-pin exact** (`==X.Y.Z`), bukan range. Alasannya didokumentasikan di `pyproject.toml:18-30` — respons terhadap worm *Mini Shai-Hulud* yang menyerang `mistralai 2.4.6` di PyPI pada 12 Mei 2026. Ini indikator kuat kualitas maintenance.

### 3.2 Frontend Web Dashboard — `web/`

| Komponen | Teknologi | Versi |
|---|---|---|
| Framework | **React** | `19.2.7` |
| Bahasa | **TypeScript** | `6.0.3` |
| Build tool | **Vite** | `8.2.0` |
| Styling | **Tailwind CSS** | `4.3.3` (via `@tailwindcss/vite`) |
| Design system | **@nous-research/ui** | `0.18.2` (paket privat Nous) |
| Routing | **react-router** | `8.3.0` |
| Ikon | **lucide-react** | `0.577.0` |
| Animasi | **motion** (Framer) + **gsap** | `12.42.2` / `3.15.0` |
| Terminal emulator | **@xterm/xterm** + addon fit/webgl/unicode11/web-links | `6.0.0` |
| Charting | **@observablehq/plot** | `0.6.17` |
| 3D | **@react-three/fiber** + **three** | `9.6.1` / `0.180.0` |
| QR code | **qrcode** | `1.5.4` (untuk pairing) |
| Testing | **Vitest** | `4.1.10` |
| Utility CSS | **clsx**, **tailwind-merge**, **class-variance-authority** | |

**Build output:** `web/vite.config.ts:87` → `outDir: "../hermes_cli/web_dist"`, yang persis dibaca oleh `hermes_cli/web_server.py:138` (`WEB_DIST`, bisa dioverride dengan env `HERMES_WEB_DIST`).

### 3.3 Terminal UI — `ui-tui/`

React yang dirender ke terminal (kemungkinan besar **Ink** — ada `types/hermes-ink.d.ts`), di-bundle dengan **esbuild**, dijalankan sebagai `node ui-tui/dist/entry.js`. Komponen: model picker, skills hub, plugins hub, pet sprite, widget grid, todo panel, billing overlay, streaming markdown.

### 3.4 Desktop App — `apps/desktop/`

| Komponen | Teknologi | Versi |
|---|---|---|
| Shell | **Electron** | `40.10.2` |
| Renderer | **React + Vite** | sama seperti web |
| Chat UI | **assistant-ui** | `apps/desktop/src/components/assistant-ui/` |
| State | **nanostores** + `@nanostores/react` | |
| Packaging | **electron-builder** | DMG, MSI, NSIS, AppImage, deb, rpm |
| E2E test | **Playwright** | |
| Haptics | **web-haptics** | `apps/desktop/src/components/haptics-provider.tsx` |

### 3.5 Shared / Monorepo

**npm workspaces** (`package.json:6-12`): `apps/*`, `ui-tui`, `ui-tui/packages/*`, `web`, `tests-js`.

Paket bersama **`@hermes/shared`** (`apps/shared/src/`) berisi kode yang dipakai web + desktop:
- `json-rpc-gateway.ts` — client JSON-RPC/WebSocket ke backend
- `websocket-url.ts` — resolusi URL WS + ticket OAuth
- `backend-scope.ts`, `cron-trigger-controller.ts`, `skill-scaffold.ts`, `skin.ts`, `billing-*.ts`

Ini penting: **web dan desktop sudah berbagi transport layer**. Itu yang membuat porting chat UI jadi realistis.

### 3.6 Infrastruktur & Tooling

| Kategori | Teknologi |
|---|---|
| Container | **Docker** (`Dockerfile` 25 KB, multi-stage) + **s6-overlay** sebagai PID 1 |
| Orkestrasi lokal | **docker-compose** (`docker-compose.yml`, `docker-compose.windows.yml`) |
| Reproducible build | **Nix flake** (`flake.nix`, `nix/`) |
| Node runtime | **Node.js >= 22.22.0** (`.nvmrc` = `26`) |
| Lint JS | **ESLint 9** + typescript-eslint + perfectionist + react-hooks |
| Format | **Prettier** |
| Lint Dockerfile | **hadolint** (`.hadolint.yaml`) |
| CI | **GitHub Actions** (`.github/`) |
| Code review | **CodeRabbit** (`.coderabbit.yaml`) |
| Env dev | **direnv** (`.envrc`) |
| Protokol agent | **ACP adapter** (`acp_adapter/`) |

---

## 4. Arsitektur Sistem

### 4.1 Peta Utuh

```
                        ╔═══════════════════════════════════════════╗
                        ║        HERMES CORE (Python)               ║
                        ║                                           ║
  ┌── CLI/TUI ─────────►║  run_agent.py    — agent loop (419 KB)    ║
  │   `hermes`          ║  agent/          — conversation, prompts   ║
  │                     ║  toolsets.py     — 124 tool di tools/      ║
  ├── Gateway ─────────►║  hermes_state.py — SQLite state (590 KB)   ║
  │   22 platform       ║  hermes_state_search.py — FTS5 search      ║
  │                     ║  cron/           — scheduler               ║
  ├── Dashboard ───────►║  skills/         — memori prosedural       ║
  │   :9119 (web)       ║  plugins/        — 21 kategori plugin      ║
  │                     ║  mcp_serve.py    — MCP server              ║
  └── Desktop ─────────►║                                            ║
      (Electron)        ╚═══════════════════════════════════════════╝
```

### 4.2 Web Dashboard — Cara Kerja Detail

Ini bagian yang perlu Anda pahami paling dalam.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  BROWSER (Chrome/Safari di HP atau desktop)                              │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  SPA React 19  (web/src/)                                          │  │
│  │                                                                    │  │
│  │  App.tsx  ── sidebar nav (drawer di mobile) + <Routes>              │  │
│  │     │                                                              │  │
│  │     ├─ 18 halaman "settings/management"  ─── REST ──► /api/*        │  │
│  │     │  (Config, Keys, Cron, Skills, MCP, Channels, Files, ...)      │  │
│  │     │  → arsitektur BAGUS untuk mobile, sudah responsif             │  │
│  │     │                                                              │  │
│  │     └─ ChatPage.tsx  ─── WebSocket ──► /api/pty                     │  │
│  │        xterm.js Terminal (WebGL)                                    │  │
│  │        → INI MASALAHNYA untuk mobile UX                             │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ HTTP + WebSocket
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  FastAPI  (hermes_cli/web_server.py — 19.157 baris)                      │
│                                                                          │
│  Middleware chain:                                                       │
│    • host_header_middleware   — anti DNS-rebinding (GHSA-ppp5-vxwm-4cf7)  │
│    • auth_middleware          — mode loopback: token sesi ephemeral       │
│    • gated_auth_middleware    — mode non-loopback: cookie OAuth/password  │
│                                                                          │
│  Router (hermes_cli/web_routers/):                                       │
│    cron.py · git.py · mcp.py · profiles.py · sessions.py                  │
│    skills.py · tools.py                                                  │
│                                                                          │
│  Auth (hermes_cli/dashboard_auth/):                                      │
│    routes.py · middleware.py · cookies.py · login_page.py                │
│    token_auth.py · ws_tickets.py · audit.py · prefix.py                   │
│                                                                          │
│  Provider auth (plugins/dashboard_auth/):                                │
│    basic (username + scrypt) · nous (OAuth Portal) · self_hosted · drain  │
│                                                                          │
│  mount_spa()  — serve web_dist/, inject token + base-path                │
│                 honor X-Forwarded-Prefix (reverse proxy subpath)          │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
        ┌───────────────────────┴────────────────────────┐
        ▼                                                ▼
┌──────────────────────┐                    ┌────────────────────────────┐
│  /api/pty  (WS)      │                    │  /api/ws  (WS JSON-RPC)    │
│                      │                    │                            │
│  POSIX PTY           │                    │  tui_gateway/server.py     │
│    │                 │                    │                            │
│    ▼                 │                    │  Method: prompt.submit,    │
│  node ui-tui/        │                    │   session.*, config.*,     │
│    dist/entry.js     │                    │   tools.*, slash.*         │
│    │                 │                    │                            │
│    ▼                 │                    │  Event stream:             │
│  tui_gateway         │                    │   message.delta            │
│    │                 │                    │   thinking.delta           │
│    ▼                 │                    │   tool.start/progress/     │
│  AIAgent             │                    │        complete            │
│                      │                    │   approval.request         │
│  ◄── dipakai         │                    │                            │
│      ChatPage web    │                    │  ◄── dipakai Desktop app   │
│      (xterm mirror)  │                    │      (native chat UI)      │
└──────────────────────┘                    └────────────────────────────┘
```

**Observasi kunci dari diagram di atas:** ada **dua jalur berbeda** untuk chat ke agent yang sama.

- **Web dashboard** memakai `/api/pty` → PTY → TUI → xterm.js. Artinya browser hanya jadi *emulator terminal*.
- **Desktop app** memakai `/api/ws` → JSON-RPC dengan event stream terstruktur → React chat UI dengan bubble, markdown, tool card.

Jalur kedua itulah yang harus dibawa ke web untuk mendapat UX seperti app Claude. **Backend-nya sudah ada dan sudah berjalan** — tidak perlu dibangun.

### 4.3 Halaman Dashboard yang Sudah Ada (19 halaman + tab plugin dinamis)

Dari `web/src/pages/` dan tabel nav di `web/src/App.tsx:139-220`:

| Route | Halaman | Fungsi |
|---|---|---|
| `/chat` | ChatPage | Chat dengan agent (**via xterm — gap mobile**) |
| `/sessions` | SessionsPage | Riwayat & pencarian sesi |
| `/files` | FilesPage | File manager (browse, upload, download, delete) |
| `/analytics` | AnalyticsPage | Token & cost analytics |
| `/models` | ModelsPage | Pilih provider & model |
| `/logs` | LogsPage | Log viewer |
| `/cron` | CronPage | Penjadwalan tugas |
| `/skills` | SkillsPage | Kelola skill + Skills Hub |
| `/plugins` | PluginsPage | Kelola plugin |
| `/mcp` | McpPage | Konfigurasi MCP server |
| `/channels` | ChannelsPage | Platform messaging (Telegram, Discord, dll.) |
| `/webhooks` | WebhooksPage | Konfigurasi webhook |
| `/pairing` | PairingPage | Pairing DM (dengan QR code) |
| `/profiles` | ProfilesPage | Multi-profil agent |
| `/profile-builder` | ProfileBuilderPage | Wizard bikin profil |
| `/config` | ConfigPage | Seluruh config.yaml |
| `/env` | EnvPage | API keys / environment variables |
| `/system` | SystemPage | Status sistem, update, diagnostik |
| `/docs` | DocsPage | Dokumentasi in-app |
| *(dinamis)* | Plugin tabs | Tab dari plugin (mis. Kanban) |

**Ini artinya: "semua fitur dan pengaturan Hermes via browser" sudah 95% terpenuhi.** Yang kurang cuma kualitas UX pada tab Chat, dan lapisan PWA.

---

## 5. Cara Menjalankan

### 5.1 Instalasi Cepat (cara resmi)

**Linux / macOS / WSL2 / Termux:**
```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
source ~/.bashrc
hermes
```

**Windows (PowerShell, native — tanpa WSL):**
```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

Installer mengurus: `uv`, Python 3.11, Node.js, ripgrep, ffmpeg, dan (di Windows) portable Git Bash (MinGit ke `%LOCALAPPDATA%\hermes\git`).

### 5.2 Dari Source (yang relevan untuk VPS)

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# 1. Python environment (uv membaca requires-python & uv.lock)
uv sync --extra all          # atau: uv sync --extra web  (minimal untuk dashboard)

# 2. Node dependencies (npm workspaces)
npm ci

# 3. Build frontend dashboard → hermes_cli/web_dist/
npm run build --workspace web

# 4. Setup wizard
./hermes setup               # atau: ./hermes setup --portal  (OAuth Nous Portal)

# 5. Jalankan
./hermes                     # TUI
./hermes dashboard           # Web UI di http://127.0.0.1:9119
./hermes gateway start       # Messaging gateway
```

Di Termux/Android pakai extra khusus: `uv sync --extra termux` (extra `all` menarik dependency voice yang tidak kompatibel Android).

### 5.3 Perintah CLI Utama

```bash
hermes                    # Interactive TUI
hermes model              # Pilih provider & model
hermes tools              # Konfigurasi tool yang aktif
hermes config set/get     # Set/get nilai config
hermes dashboard          # Web UI (port 9119)
hermes dashboard --stop   # Stop dashboard yang berjalan
hermes dashboard --status # List proses dashboard
hermes serve              # Backend headless (JSON-RPC/WS, tanpa SPA)
hermes gateway setup      # Wizard platform messaging
hermes gateway start      # Jalankan gateway
hermes setup              # Wizard lengkap
hermes doctor             # Diagnostik
hermes update             # Update ke versi terbaru
hermes portal info        # Cek status Nous Portal
```

### 5.4 Flag `hermes dashboard`

Dari `hermes_cli/subcommands/dashboard.py`:

| Flag | Default | Fungsi |
|---|---|---|
| `--port N` | `9119` | Port (`0` = auto-assign OS) |
| `--host H` | `127.0.0.1` | Interface bind |
| `--no-open` | off | Jangan buka browser otomatis (**wajib di VPS headless**) |
| `--skip-build` | off | Jangan build frontend, pakai `web_dist` yang ada |
| `--isolated` | off | Server khusus per-profil (default: satu server machine-level) |
| `--insecure` | off | **DEPRECATED / NO-OP** — tidak lagi mem-bypass auth |
| `--stop` | — | Stop semua proses dashboard, lalu exit |
| `--status` | — | List proses dashboard yang listening, lalu exit |

**Penting:** `hermes dashboard` akan **otomatis menjalankan `npm install` + `vite build`** jika `web_dist` belum ada atau stale (`hermes_cli/main.py:6280` `_build_web_ui`, diserialisasi dengan `flock` agar boot paralel tidak saling starve). Di VPS kecil ini bisa jadi masalah — lihat [§10.3](#103-build-frontend-di-vps-kecil-bisa-oom).

### 5.5 Docker

```bash
HERMES_UID=$(id -u) HERMES_GID=$(id -g) docker compose up -d
```

`docker-compose.yml` mendefinisikan dua service:
- **`gateway`** — `command: ["gateway", "run"]`
- **`dashboard`** — `command: ["dashboard", "--host", "127.0.0.1", "--no-open"]`

Keduanya `network_mode: host`, volume `~/.hermes:/opt/data`, entrypoint `/init` (s6-overlay PID 1 yang menjalankan `cont-init.d` scripts sebelum service mana pun start).

Komentar di `docker-compose.yml:14-18` secara eksplisit menyarankan **jangan** pakai `--insecure --host 0.0.0.0`, dan pakai SSH tunnel atau reverse proxy ber-auth. Nasihat itu sekarang sebagian sudah usang: `--insecure` sudah jadi no-op dan auth gate otomatis aktif pada bind non-loopback — jadi `--host 0.0.0.0` **dengan** auth provider terkonfigurasi kini merupakan jalur yang didukung.

---

## 6. Jawaban Jujur: Bisa Dideploy di VPS & Dibuka di Browser Smartphone?

### 6.1 YA — dan berikut bukti bahwa infrastrukturnya sudah ada

#### (a) Bind non-loopback sudah didukung secara resmi

`hermes_cli/web_server.py:641-660`:

```python
def should_require_auth(host: str, allow_public: bool = False) -> bool:
    """Return True iff the dashboard auth gate must be active.

    Truth table:
      host == loopback        → False (no auth — local-only, trusted operator)
      host != loopback        → True  (gate engages — OAuth or password required)
    """
    return host not in _LOOPBACK_HOST_VALUES
```

Jadi `hermes dashboard --host 0.0.0.0` **berjalan**, dan otomatis mengaktifkan auth gate.

#### (b) Auth gate bersifat WAJIB dan fail-closed

`hermes_cli/web_server.py:18862-18937` — jika bind non-loopback tapi **tidak ada** auth provider terdaftar, server **menolak start** dengan `SystemExit` dan pesan yang memberi instruksi perbaikan. Tidak ada escape hatch.

Konteksnya didokumentasikan di kode: hardening Juni 2026 sebagai respons kampanye **`hermes-0day`** MCP-persistence, yang mengeksploitasi dashboard publik tanpa auth. `--insecure` sengaja dijadikan no-op:

```python
# ``--insecure`` no longer disables the auth gate (June 2026 hardening:
# the hermes-0day MCP-persistence campaign abused unauthenticated public
# dashboards).
```

Ini kabar **bagus** untuk Anda: berarti mengekspos dashboard ke internet adalah use case yang sudah dipikirkan dan diamankan, bukan hack.

#### (c) Dua metode auth siap pakai

**1. Password (scrypt) — paling praktis untuk VPS pribadi.**
Plugin `plugins/dashboard_auth/basic/`. Config di `hermes_cli/config_defaults.py:1515`:

```yaml
dashboard:
  basic_auth:
    username: ""            # kosong → plugin no-op
    password_hash: ""       # scrypt$... (disarankan — tanpa plaintext at rest)
    password: ""            # fallback plaintext (di-hash in-memory saat load)
    secret: ""              # kunci penandatangan token; kosong → random per-proses
    session_ttl_seconds: 0  # 0 → default plugin (12 jam)
```

Generate hash (`plugins/dashboard_auth/basic/__init__.py:115`):
```bash
python -c "from plugins.dashboard_auth.basic import hash_password; print(hash_password('PASSWORD-ANDA'))"
```
Parameter: `scrypt` dengan salt acak, format `scrypt$n$r$p$<salt_b64>$<dk_b64>`, verifikasi constant-time.

**2. OAuth via Nous Portal:** `hermes dashboard register`.

Ada juga provider `self_hosted` dan `drain` (service credential non-interaktif).

#### (d) Halaman login sudah mobile-friendly

`hermes_cli/dashboard_auth/login_page.py:40` — `<meta name="viewport" content="width=device-width, initial-scale=1">`, form dengan `max-width: 26rem`, POST JSON ke `/auth/password-login`. Sudah enak di HP.

#### (e) Frontend sudah dirancang sadar-mobile — ini yang paling meyakinkan

**Viewport meta lengkap** (`web/index.html:6-9`):
```html
<meta name="viewport"
      content="width=device-width, initial-scale=1.0,
               viewport-fit=cover,
               interactive-widget=resizes-content" />
```
`viewport-fit=cover` = dukungan notch/Dynamic Island. `interactive-widget=resizes-content` = penanganan keyboard virtual yang benar. Ini **bukan** viewport tag default hasil scaffold — ini pilihan sadar untuk mobile.

**Layout mobile nyata** (`web/src/App.tsx`):
- `web/src/App.tsx:396` — `const isMobile = useBelowBreakpoint(1024)`
- `web/src/App.tsx:504` — `matchMedia("(min-width: 1024px)")` untuk auto-close drawer
- Sidebar jadi **drawer overlay** di mobile dengan `mobileOpen` state, `body.style.overflow = "hidden"` saat terbuka, dan handler Escape
- `web/src/App.tsx:770` — `pb-[calc(2rem+env(safe-area-inset-bottom,0px))]` → **safe-area inset** untuk home indicator iPhone
- `h-dvh max-h-dvh` → **dynamic viewport height** (bukan `100vh` yang bermasalah di mobile Safari)
- Padding responsif berlapis: `px-3 sm:px-6`, `pt-2 sm:pt-4 lg:pt-6`

**Bottom sheet untuk dropdown di layar sempit:**
- `web/src/components/ThemeSwitcher.tsx:33-34` — `useBelowBreakpoint(640)` → `useMobileSheet`
- `web/src/components/LanguageSwitcher.tsx:35-36` — pola sama

**Penanganan keyboard virtual yang serius:**
- `web/src/lib/keyboard-inset.ts` — `computeKeyboardInset()` mengukur area yang ditutupi keyboard via `visualViewport`, termasuk `offsetTop` (kasus iOS yang menggeser visual viewport)
- `web/src/lib/pty-mobile-input.ts` — `normalizePtyMobileInput()`, `shouldTreatInputAsMobileReplacement()` untuk menangani autocorrect/IME mobile yang mengirim multi-karakter
- `web/src/lib/pty-scroll.ts` — pinning viewport ke bawah saat streaming

**Jumlah utility responsif Tailwind di `web/src/`:** 177× `sm:`, 61× `lg:`, 8× `md:`, 5× `xl:`.

Ada juga **test unit khusus mobile**: `web/src/lib/keyboard-inset.test.ts` menguji kasus keyboard iOS. Dan di `tests-js/assistant-ui-tap-compat.test.ts` ada test kompatibilitas *tap*.

#### (f) Dukungan reverse proxy sudah dibangun

`hermes_cli/web_server.py:17285-17290` (`mount_spa`):
> When served behind a path-prefix reverse proxy (e.g. `mission-control.tilos.com/hermes/*` -> local Caddy -> :9119), the proxy injects `X-Forwarded-Prefix: /hermes` on every request. We rewrite the served `index.html` so absolute asset URLs (`/assets/...`) and the SPA's runtime `__HERMES_BASE_PATH__` honour that prefix without rebuilding the bundle.

Jadi **subpath hosting** (`domain.com/hermes/`) sudah didukung tanpa rebuild.

`proxy_headers=True` diaktifkan otomatis saat auth gate aktif (`hermes_cli/web_server.py:18987`), sehingga `X-Forwarded-Proto` dipakai untuk memutuskan flag `Secure` pada cookie. Ada juga override `dashboard.public_url` (env `HERMES_DASHBOARD_PUBLIC_URL`) untuk proxy yang tidak meneruskan header `X-Forwarded-*` dengan benar.

#### (g) WebSocket sudah disiapkan untuk kondisi jaringan mobile/tunnel

`hermes_cli/web_server.py:18990-18995`:
```python
ws_ping_interval=None if _is_loopback else 20.0,
ws_ping_timeout=None if _is_loopback else 20.0,
```
Keepalive ping 20s **hanya** untuk bind non-loopback, karena "Non-loopback binds sit behind a Cloudflare Tunnel (idle timeout ~100s) where half-open IS a real failure mode". Loopback mematikannya agar event-loop stall tidak memutus koneksi lokal yang sehat.

Ada juga logika reconnect di sisi klien: `web/src/lib/pty-reconnect.ts` (`shouldReconnectPtyOnPageResume`) dan `pty-resume-loading.ts` — persis kebutuhan mobile di mana tab di-background dan koneksi mati.

#### (h) Pengamanan tambahan yang sudah ada

| Proteksi | Implementasi |
|---|---|
| DNS rebinding | `_is_accepted_host()` (`web_server.py:663`) + `host_header_middleware`; referensi GHSA-ppp5-vxwm-4cf7 |
| WS Host/Origin guard | `_ws_host_origin_is_allowed()` / `_ws_host_origin_reason()` (`web_server.py:15848+`) |
| WS peer gate | `_ws_client_is_allowed()` (`web_server.py:15805`) — fail-closed saat peer tak teridentifikasi |
| WS ticket single-use | `hermes_cli/dashboard_auth/ws_tickets.py` — untuk `/api/pty` & `/api/ws` di mode gated |
| Public path allowlist | `hermes_cli/dashboard_auth/public_paths.py` — terpusat, minimal, dengan kriteria eksplisit ("aman di-curl siapa pun") |
| Audit log | `hermes_cli/dashboard_auth/audit.py` |
| Cookie handling | `hermes_cli/dashboard_auth/cookies.py` — flag `Secure` mengikuti `X-Forwarded-Proto` |

### 6.2 TAPI — tiga hal yang BELUM ada

#### ❌ Gap #1 (KRITIS untuk tujuan Anda): Tab Chat adalah emulator terminal, bukan chat UI

Ini gap yang paling penting. Baca header file `web/src/pages/ChatPage.tsx:1-17`:

```
/**
 * ChatPage — embeds `hermes --tui` inside the dashboard.
 *
 *   <div host> (dashboard chrome)
 *     └─ <div wrapper> (rounded, dark bg, padded — the "terminal window" look)
 *         └─ @xterm/xterm Terminal (WebGL renderer, Unicode 11 widths)
 *              │ onData      keystrokes → WebSocket → PTY master
 *              │ onResize    terminal resize → `\x1b[RESIZE:cols;rows]`
 *              │ write(data) PTY output bytes → VT100 parser
 *              ▼
 *     WebSocket /api/pty?token=<session>
 *          ▼
 *     FastAPI pty_ws  (hermes_cli/web_server.py)
 *          ▼
 *     POSIX PTY → `node ui-tui/dist/entry.js` → tui_gateway + AIAgent
 */
```

Artinya: saat Anda buka `/chat` di HP, yang Anda lihat adalah **terminal Linux yang dirender dalam canvas WebGL**, menjalankan TUI Hermes. `ChatPage.tsx` panjangnya **1.954 baris**, sebagian besar untuk menjembatani kesenjangan terminal↔mobile.

**Mengapa ini tidak akan pernah terasa seperti app Claude, seberapa pun dipoles:**

| Aspek | Terminal via xterm.js | Yang diharapkan dari app Claude |
|---|---|---|
| Layout teks | Grid karakter monospace berukuran tetap (cols × rows) | Teks yang reflow mengikuti lebar layar |
| Struktur pesan | Aliran teks datar dengan ANSI escape | Bubble terpisah, jelas siapa berbicara |
| Markdown | Diterjemahkan ke ANSI oleh TUI — tabel jebol di layar sempit | Rendering HTML asli, tabel bisa di-scroll |
| Blok kode | Teks ANSI biasa | Syntax highlight + tombol copy |
| Input | Keystroke dikirim byte-per-byte ke PTY | `<textarea>` native: autocorrect, dikte, paste, undo |
| Seleksi teks | Seleksi xterm custom, bukan seleksi native | Seleksi & copy native OS |
| Tap target | Tidak ada — semuanya karakter | Tombol ≥44×44pt sesuai HIG |
| Tool call | Baris teks | Card yang bisa dilipat dengan status |
| Gambar/lampiran | Tidak bisa inline | Preview inline |
| Scroll | Scrollback xterm | Scroll momentum native |
| Zoom | Merusak grid | Reflow normal |
| Screen reader | Praktis tidak bisa diakses | Semantik ARIA |
| Rotasi layar | Perlu resize + reflow PTY | Otomatis |

Tim Hermes sudah bekerja keras memitigasi ini (`pty-mobile-input.ts`, `keyboard-inset.ts`, `pty-composition.ts`, `pty-keyboard-shortcuts.ts`, `pty-scroll.ts`, `pty-resume-sanitizer.ts`) — tapi itu **memoles emulator terminal**, bukan mengubahnya jadi chat UI. Batas atas kualitasnya adalah "terminal yang dapat dipakai di HP", bukan "app chat yang nyaman".

#### ✅ Tapi ada kabar sangat baik: solusinya sudah ada di repo ini

**Desktop app tidak memakai jalur PTY untuk chat.** Buktinya:

1. `apps/desktop/src/hermes.ts:1` mengimpor `JsonRpcGatewayClient` dari `@hermes/shared` — client **WebSocket JSON-RPC**, bukan PTY.
2. xterm di desktop **hanya** dipakai di `apps/desktop/src/app/right-sidebar/terminal/` — panel terminal terpisah sebagai *tool*, bukan untuk chat.
3. Chat UI-nya ada di `apps/desktop/src/components/assistant-ui/` dan `apps/desktop/src/app/chat/`, dengan komponen: `markdown-text.tsx`, `thread/`, `tool/`, `artifact-card.tsx`, `ansi-text.tsx`, `embeds/`, `message-render-boundary.tsx`, `inline-preview-directive.tsx`.
4. Backend-nya sudah menyediakan **event stream terstruktur** yang persis dibutuhkan chat UI. Dari `apps/shared/src/json-rpc-gateway.ts:1-24`:

```typescript
export type GatewayEventName =
  | 'gateway.ready'      | 'session.info'      | 'session.usage'
  | 'message.start'      | 'message.delta'     | 'message.interim'
  | 'message.complete'   | 'thinking.delta'    | 'reasoning.delta'
  | 'reasoning.available'| 'status.update'
  | 'tool.start'         | 'tool.progress'     | 'tool.complete'
  | 'tool.generating'    | 'clarify.request'   | 'approval.request'
  | 'sudo.request'       | 'secret.request'    | 'background.complete'
  | 'error'              | 'skin.changed'
```

5. Method JSON-RPC yang tersedia di `tui_gateway/` sudah lengkap:
   `prompt.submit`, `prompt.background`, `session.create/list/history/resume/interrupt/steer/redirect/branch/compress/undo/title/delete/usage/context_breakdown`, `config.get/set/show/yaml`, `tools.list/configure/show`, `slash.compress`.

**Kesimpulan gap #1:** ini pekerjaan **porting renderer**, bukan membangun sistem baru. Backend streaming, protokol, dan komponen React chat-nya semua sudah ada dan sudah teruji di produksi (desktop app v0.17.0).

#### ❌ Gap #2: Tidak ada PWA

Hasil pencarian di `web/src`, `web/index.html`, `web/public`: **nol** kemunculan `manifest.json`, `serviceWorker`, `apple-mobile-web-app-*`, atau `display: standalone`.

`web/public/` hanya berisi `favicon.ico` + folder font. Bandingkan `apps/desktop/public/` yang **sudah punya** `apple-touch-icon.png` — asetnya ada, cuma belum dipakai di web.

Konsekuensinya di HP:
- Tidak bisa "Add to Home Screen" sebagai app sungguhan (masih tampil address bar browser)
- Tidak ada icon di home screen
- Tidak ada splash screen
- Tidak ada standalone display mode
- Tidak ada offline shell — kehilangan sinyal sekejap = halaman putih
- Tidak ada theme-color untuk status bar
- Tidak ada push notification

#### ❌ Gap #3: Tidak ada TLS bawaan

Pencarian `ssl_certfile` / `ssl_keyfile` / `--ssl` di `web_server.py` dan `subcommands/dashboard.py`: **nol hasil**. `uvicorn.Config` dipanggil tanpa parameter SSL (`web_server.py:18980-18999`).

Ini **bukan cacat desain** — memang praktik yang benar adalah terminasi TLS di reverse proxy. Tapi berarti Anda **wajib** menyiapkan Caddy/nginx/Cloudflare Tunnel. Dan ini bukan opsional: tanpa HTTPS, Service Worker (untuk PWA) tidak akan jalan, cookie `Secure` tidak akan diset, dan password Anda melintas polos.

---

## 7. Analisa Gap Detail

Ringkasan seluruh gap, diurutkan berdasarkan dampak terhadap tujuan Anda:

| # | Gap | Dampak | Effort | Prioritas |
|---|---|---|---|---|
| 1 | Chat = xterm terminal, bukan native chat UI | 🔴 Kritis — inti dari "UX seperti app Claude" | Besar (2–4 minggu) | **P0** |
| 2 | Tidak ada PWA (manifest, SW, ikon) | 🟠 Tinggi — "terasa seperti app" | Kecil (2–3 hari) | **P0** |
| 3 | Tidak ada TLS bawaan | 🟠 Tinggi — blocker untuk PWA & keamanan | Kecil (2–4 jam, di proxy) | **P0** |
| 4 | Nav mobile masih pola desktop (drawer 19 item) | 🟡 Sedang — navigasi terasa canggung di HP | Sedang (3–5 hari) | P1 |
| 5 | Build frontend berat untuk VPS kecil | 🟡 Sedang — risiko OOM saat deploy/update | Kecil (workaround ada) | P1 |
| 6 | Tidak ada 2FA/TOTP | 🟡 Sedang — dashboard ini pegang API keys & shell | Sedang | P1 |
| 7 | Tidak ada push notification | 🟢 Rendah — nice to have | Sedang | P2 |
| 8 | Halaman padat tabel belum dioptimalkan mobile | 🟢 Rendah — sudah bisa dipakai | Sedang | P2 |
| 9 | Tidak ada unit systemd bawaan | 🟢 Rendah — mudah dibuat manual | Kecil | P2 |

Catatan gap #9: kode sudah **mengenali** systemd — `hermes_cli/main.py:8150` mendefinisikan `_DASHBOARD_SYSTEMD_UNIT = "hermes-dashboard.service"` dan `_restart_managed_dashboard_service()` akan me-restart unit itu (bukan kill PID mentah) saat update. Jadi ada *konvensi* nama unit, hanya file unit-nya tidak disertakan. Kalau Anda menamai unit Anda persis `hermes-dashboard.service`, integrasi update jadi lebih baik.

---

## 8. Strategi & Konsep Teknis Pengembangan

Strateginya berlapis. Setiap fase memberi nilai yang bisa langsung dipakai, dan tidak ada fase yang menghalangi fase berikutnya.

```
FASE 0  Deploy & Hardening              0.5–1 hari    ► Bisa dipakai dari HP HARI INI
FASE 1  PWA Shell                       2–3 hari      ► Terasa seperti app
FASE 2  Native Chat UI (JSON-RPC)       2–4 minggu    ► UX seperti app Claude  ◄ INTI
FASE 3  Navigasi & Layout Mobile        3–5 hari      ► Navigasi terasa native
FASE 4  Hardening & Operasional         3–5 hari      ► Siap produksi jangka panjang
FASE 5  Push Notification & Offline     1–2 minggu    ► Opsional
```

---

### FASE 0 — Deploy & Hardening (0.5–1 hari)

**Tujuan:** dashboard bisa dibuka aman dari browser HP, dengan HTTPS, tanpa mengubah kode sama sekali.

**Konsep teknis:** Hermes bind ke loopback saja; reverse proxy yang menangani TLS dan menghadap internet. Ini lebih aman daripada `--host 0.0.0.0` langsung, karena tidak ada jalur ke Hermes yang melewati proxy.

```
Internet ──HTTPS──► Caddy :443 ──HTTP──► Hermes :9119 (127.0.0.1)
           (Let's Encrypt)      (loopback saja)
```

**Dua pilihan arsitektur:**

| Pilihan | Bind Hermes | Auth | Kapan dipakai |
|---|---|---|---|
| **A. Loopback + reverse proxy** ✅ disarankan | `127.0.0.1` | Auth gate Hermes **tidak** aktif (loopback) → proxy yang harus meng-auth | Paling aman: tidak ada port Hermes terbuka |
| **B. `0.0.0.0` + auth gate Hermes** | `0.0.0.0` | Auth gate Hermes **wajib** aktif (password/OAuth) | Kalau tidak mau kelola auth di proxy |

**Penting untuk pilihan A:** karena bind loopback membuat `should_require_auth()` mengembalikan `False`, Hermes **tidak akan** meminta login. Jadi proxy Anda **wajib** melakukan autentikasi (basic auth Caddy, forward-auth, atau Cloudflare Access). Kalau tidak, siapa pun yang tahu domain Anda punya akses penuh ke shell dan API keys.

**Untuk pilihan B (aktifkan auth gate Hermes) — paling seimbang:**

Bisa juga dikombinasikan: bind `0.0.0.0` **tapi** firewall hanya izinkan proxy, sehingga dapat dua lapis. Namun bentuk paling bersih adalah: bind loopback + auth di Hermes tidak aktif + auth di proxy; atau bind `0.0.0.0` di-firewall + auth di Hermes aktif.

Detail langkah lengkap ada di [§9](#9-panduan-deploy-vps-fase-0--bisa-dikerjakan-hari-ini).

**Hasil Fase 0:** Anda sudah bisa buka `https://hermes.domain-anda.com` di Safari/Chrome HP, login, dan mengakses **18 halaman pengaturan** dengan layout responsif yang sudah ada. Chat sudah bisa dipakai — cuma masih berupa terminal.

---

### FASE 1 — PWA Shell (2–3 hari)

**Tujuan:** Hermes bisa di-"Add to Home Screen", buka fullscreen tanpa address bar, punya ikon dan splash screen, dan tidak blank saat sinyal hilang sekejap.

Ini **effort terkecil dengan dampak persepsi terbesar**. Sebagian besar orang mengira "seperti app native" berarti standalone display + ikon home screen — dan itu murni pekerjaan PWA, bukan chat UI.

#### 1.1 Web App Manifest

Buat `web/public/manifest.webmanifest`:

```json
{
  "name": "Hermes Agent",
  "short_name": "Hermes",
  "description": "Self-improving AI agent — Nous Research",
  "start_url": "/chat",
  "scope": "/",
  "display": "standalone",
  "display_override": ["window-controls-overlay", "standalone"],
  "orientation": "any",
  "background_color": "#0a0a0a",
  "theme_color": "#0a0a0a",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-maskable-512.png", "sizes": "512x512",
      "type": "image/png", "purpose": "maskable" }
  ],
  "shortcuts": [
    { "name": "Chat",    "url": "/chat",    "icons": [{ "src": "/icons/chat-96.png",    "sizes": "96x96" }] },
    { "name": "Cron",    "url": "/cron",    "icons": [{ "src": "/icons/cron-96.png",    "sizes": "96x96" }] },
    { "name": "Sessions","url": "/sessions","icons": [{ "src": "/icons/sessions-96.png","sizes": "96x96" }] }
  ]
}
```

Sumber ikon: `apps/desktop/assets/icon.png` dan `apps/desktop/public/apple-touch-icon.png` sudah ada di repo — cukup di-resize.

**Catatan kritis soal `theme_color`:** dashboard punya sistem tema (`web/src/themes/presets.ts`). Manifest bersifat statis, jadi warna di manifest hanya dipakai untuk splash screen. Untuk status bar yang mengikuti tema aktif, update `<meta name="theme-color">` secara dinamis dari `ThemeProvider` (`web/src/themes/context.tsx`).

#### 1.2 Tag `<head>` tambahan

Di `web/index.html`, tambahkan setelah viewport meta yang sudah ada:

```html
<link rel="manifest" href="/manifest.webmanifest" />
<meta name="theme-color" content="#0a0a0a" />
<link rel="apple-touch-icon" href="/icons/apple-touch-icon-180.png" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<meta name="apple-mobile-web-app-title" content="Hermes" />
<meta name="mobile-web-app-capable" content="yes" />
```

`black-translucent` dipilih karena `viewport-fit=cover` sudah ada di viewport tag — kombinasi ini yang membuat konten mengalir sampai bawah notch dengan benar, dan `env(safe-area-inset-*)` yang sudah dipakai di `App.tsx:770` langsung bekerja.

#### 1.3 Service Worker

Gunakan **`vite-plugin-pwa`** (wrapper Workbox untuk Vite) supaya tidak menulis SW manual:

```bash
npm install -D vite-plugin-pwa --workspace web
```

Di `web/vite.config.ts`:

```typescript
import { VitePWA } from "vite-plugin-pwa";

plugins: [
  react(),
  tailwindcss(),
  hermesDevToken(),
  VitePWA({
    registerType: "prompt",        // JANGAN autoUpdate — lihat catatan di bawah
    manifest: false,               // kita kelola manifest.webmanifest sendiri
    workbox: {
      globPatterns: ["**/*.{js,css,html,woff2}"],
      // Naikkan dari default 2 MiB: bundle dashboard punya chunk besar
      // (xterm + WebGL, three.js, observable plot).
      maximumFileSizeToCacheInBytes: 8 * 1024 * 1024,
      navigateFallback: "/index.html",
      navigateFallbackDenylist: [
        /^\/api\//,          // REST — jangan pernah dilayani dari cache
        /^\/auth\//,         // login/OAuth — harus selalu ke jaringan
        /^\/p\//,            // route ber-prefix profil
      ],
      runtimeCaching: [
        {
          // Font: immutable, aman di-cache lama
          urlPattern: /\/fonts?.*\.woff2$/,
          handler: "CacheFirst",
          options: {
            cacheName: "hermes-fonts",
            expiration: { maxEntries: 32, maxAgeSeconds: 60 * 60 * 24 * 365 },
          },
        },
        {
          // JANGAN cache /api/* — isinya config, API key, state agent.
          urlPattern: /^\/api\//,
          handler: "NetworkOnly",
        },
      ],
    },
  }),
]
```

**Empat aturan keras untuk Service Worker di aplikasi ini** — ini yang paling gampang salah:

1. **Jangan pernah cache `/api/*`.** Endpoint ini mengembalikan API key, config, dan state agent. Menyimpannya di Cache Storage berarti menaruh secret di penyimpanan browser yang persisten. `NetworkOnly`, tanpa kecuali.

2. **Jangan pernah cache `/auth/*`.** Menyajikan halaman login atau callback OAuth dari cache akan merusak alur login dengan cara yang membingungkan.

3. **Jangan cache `index.html` secara agresif.** `mount_spa()` **meng-inject token sesi dan base-path ke `index.html` saat runtime** (`web_server.py:17311-17322`). HTML yang di-cache akan membawa **token kedaluwarsa** → semua panggilan `/api/*` 401 dan user melihat aplikasi rusak tanpa penjelasan. Gunakan `NetworkFirst` untuk dokumen navigasi, dan pastikan Anda memahami bahwa `navigateFallback` hanya boleh dipakai sebagai jaring saat benar-benar offline.

4. **Pakai `registerType: "prompt"`, bukan `"autoUpdate"`.** Backend Python dan frontend SPA harus versi sepadan. Kalau SW otomatis meng-update aset di belakang sesi aktif, Anda bisa mendapat SPA baru yang bicara ke backend lama (atau sebaliknya). Codebase ini sudah punya konsep *code skew* (`gateway/code_skew.py`) — hormati itu. Tampilkan prompt "Versi baru tersedia — reload".

#### 1.4 Offline shell

Buat halaman fallback offline sederhana yang menjelaskan bahwa Hermes butuh koneksi ke VPS (agent-nya berjalan di server, bukan di HP — jadi offline mode yang sesungguhnya tidak masuk akal). Yang berguna: shell app tetap muncul dengan pesan jelas dan tombol retry, alih-alih halaman error browser.

#### 1.5 Verifikasi

- Lighthouse → kategori PWA, target *installable*
- iOS Safari: Share → Add to Home Screen → buka → cek tidak ada address bar, cek safe area
- Android Chrome: cek prompt install, cek maskable icon tidak terpotong
- Airplane mode → cek shell muncul, bukan error browser
- **Cek ulang bahwa login masih berfungsi setelah SW aktif** — ini regresi paling umum

---

### FASE 2 — Native Chat UI via JSON-RPC (2–4 minggu) ◄ INTI PEKERJAAN

**Tujuan:** ganti tab Chat dari mirror terminal xterm.js menjadi chat UI React sungguhan dengan bubble pesan, markdown ter-render, tool card, dan `<textarea>` native — persis seperti app Claude.

**Konsep teknis inti:** pindahkan tab Chat dari `/api/pty` (PTY + xterm) ke `/api/ws` (JSON-RPC + event stream), lalu render event stream itu dengan komponen React. **Backend-nya sudah ada dan sudah dipakai produksi oleh desktop app** — Anda tidak membangun protokol baru.

```
SEBELUM (web dashboard hari ini):
  Browser ──WS /api/pty──► PTY ──► node ui-tui/dist/entry.js ──► tui_gateway ──► AIAgent
          ◄── byte ANSI ──                                                      
          xterm.js mem-parse VT100 → canvas WebGL
          ❌ tidak ada struktur, tidak ada semantik, grid tetap

SESUDAH (target — sama seperti desktop app hari ini):
  Browser ──WS /api/ws──► tui_gateway/server.py ──► AIAgent
          ◄── JSON-RPC ──
          message.delta / thinking.delta / tool.start / tool.complete / ...
          React merender: bubble, markdown, tool card, textarea
          ✅ terstruktur, semantik, reflow bebas
```

#### 2.1 Mengapa ini realistis (bukan rewrite)

| Yang dibutuhkan | Status | Lokasi |
|---|---|---|
| Backend event stream terstruktur | ✅ **Sudah ada** | `tui_gateway/server.py`, endpoint `/api/ws` |
| Protokol JSON-RPC terdefinisi | ✅ **Sudah ada** | `apps/shared/src/json-rpc-gateway.ts` |
| Client WebSocket JSON-RPC | ✅ **Sudah ada, shared** | `JsonRpcGatewayClient` di `@hermes/shared` |
| Resolusi URL WS + ticket OAuth | ✅ **Sudah ada, shared** | `apps/shared/src/websocket-url.ts` |
| Komponen chat React | ✅ **Sudah ada** | `apps/desktop/src/components/assistant-ui/` |
| Renderer markdown + artifact | ✅ **Sudah ada** | `markdown-text.tsx`, `artifact-card.tsx` |
| Tool call card | ✅ **Sudah ada** | `assistant-ui/tool/` |
| Thread/message list | ✅ **Sudah ada** | `assistant-ui/thread/` |
| Manajemen sesi | ✅ **Sudah ada** | method `session.*` |
| WS ticket auth di server | ✅ **Sudah ada** | `dashboard_auth/ws_tickets.py` |
| Web sudah punya komponen sesi | ✅ **Sudah ada** | `web/src/components/ChatSessionList.tsx`, `ChatSidebar.tsx` |
| **Adapter Electron IPC → browser** | ❌ **Perlu dibuat** | ← ini pekerjaan utamanya |

Yang perlu dikerjakan pada dasarnya: **membuat komponen chat desktop bisa hidup di browser tanpa Electron.**

#### 2.2 Hambatan utama: coupling Electron

Hasil pengukuran: **297 pemakaian** `window.hermes` / `ipcRenderer` tersebar di **98 file** di `apps/desktop/src/`.

`window.hermes` adalah bridge preload Electron yang menyediakan hal-hal yang tidak ada di browser (`apps/desktop/src/global.d.ts:21-57`):
- `getConnection(profile)` → `HermesConnection` (baseUrl, mode local/remote, authMode)
- `getGatewayWsUrl(profile)` → URL WS + ticket
- `openSessionWindow(sessionId)` → buka window OS baru
- `revalidateConnection()`, `touchBackend()`, `getAgentRoster()`, `getProfileRoutes()`
- Fitur khusus desktop: window controls, tray, native find-in-page, pet overlay

**Strategi: buat shim `window.hermes` untuk browser.** Sebagian besar dari 297 pemakaian itu jatuh ke beberapa method saja; banyak sisanya adalah fitur desktop-only yang bisa jadi no-op.

```typescript
// web/src/lib/hermes-browser-bridge.ts
//
// Menyediakan permukaan window.hermes yang sama seperti bridge preload
// Electron, tapi diimplementasikan dengan primitif browser. Ini memungkinkan
// komponen assistant-ui dari apps/desktop dipakai tanpa modifikasi.
//
// Aturan: method yang murni desktop (window controls, tray, session window)
// menjadi no-op yang mengembalikan { ok: false, error: 'unsupported' } —
// JANGAN throw, karena banyak call site tidak menangani exception.

export function installBrowserHermesBridge(): void {
  const bridge = {
    // ── Koneksi ──────────────────────────────────────────────────────
    // Di browser, backend SELALU origin yang sama (SPA dilayani FastAPI
    // yang sama). Tidak ada mode remote/SSH, tidak ada pool profil.
    getConnection: async (profile?: string | null) => ({
      baseUrl: window.location.origin + (window.__HERMES_BASE_PATH__ ?? ''),
      mode: 'local' as const,
      authMode: window.__HERMES_AUTH_REQUIRED__ ? ('oauth' as const)
                                                : ('token' as const),
      isFullscreen: false,
      profile: profile ?? null,
    }),

    // ── URL WebSocket ────────────────────────────────────────────────
    // Mode gated: mint ticket sekali-pakai lewat REST, karena browser tidak
    // bisa menyetel header Authorization pada handshake WebSocket.
    // Mode loopback: pakai token sesi yang di-inject mount_spa ke index.html.
    getGatewayWsUrl: async (profile?: string | null) => {
      const proto = location.protocol === 'https:' ? 'wss:' : 'ws:';
      const base = `${proto}//${location.host}${window.__HERMES_BASE_PATH__ ?? ''}`;
      if (window.__HERMES_AUTH_REQUIRED__) {
        const res = await fetch('/api/ws-ticket', { credentials: 'same-origin' });
        if (!res.ok) {
          return { ok: false as const, error: 'ws ticket failed',
                   needsOauthLogin: res.status === 401 };
        }
        const { ticket } = await res.json();
        return { ok: true as const, wsUrl: `${base}/api/ws?ticket=${ticket}` };
      }
      const token = window.__HERMES_SESSION_TOKEN__ ?? '';
      return { ok: true as const, wsUrl: `${base}/api/ws?token=${token}` };
    },

    // ── Desktop-only: no-op yang aman ────────────────────────────────
    // Buka di tab browser, bukan window OS.
    openSessionWindow: async (sessionId: string) => {
      window.open(`/chat?session=${encodeURIComponent(sessionId)}`, '_blank');
      return { ok: true as const };
    },
    revalidateConnection: async () => ({ ok: true, rebuilt: false }),
    touchBackend:         async () => ({ ok: true }),
    getAgentRoster:       async () => ({ agents: [] }),
    getProfileRoutes:     async () => [],
    // ...method desktop-only lain → { ok: false, error: 'unsupported' }
  };

  (window as unknown as { hermes: typeof bridge }).hermes = bridge;
}
```

**Catatan penting soal ticket WebSocket:** browser **tidak bisa** menyetel header kustom pada handshake WebSocket — itu batasan API `WebSocket`. Karena itu server sudah menyediakan mekanisme ticket sekali-pakai (`hermes_cli/dashboard_auth/ws_tickets.py`) dan jalur `?token=` untuk mode loopback. Cek nama endpoint mint ticket yang sebenarnya di `dashboard_auth/routes.py` sebelum implementasi — jangan asumsikan `/api/ws-ticket`.

#### 2.3 Cara memindahkan komponen — tiga opsi

| Opsi | Cara | Kelebihan | Kekurangan |
|---|---|---|---|
| **A. Ekstrak ke `apps/shared` (atau paket `apps/chat-ui` baru)** ✅ disarankan | Pindahkan `assistant-ui/` ke paket workspace bersama; desktop & web sama-sama mengimpor | Satu sumber kebenaran, tidak ada drift, perbaikan bug mengalir ke keduanya | Refactor awal paling besar; harus memisahkan dependensi Electron dulu |
| **B. Copy ke `web/src/components/chat/`** | Duplikasi komponen, sesuaikan untuk web | Tercepat sampai prototipe jalan | Dua salinan akan menyimpang; setiap fix harus dikerjakan dua kali. **Utang teknis yang mahal.** |
| **C. Build renderer desktop sebagai target web kedua** | Tambah Vite build mode "web" di `apps/desktop`, serve dari FastAPI | Reuse maksimal, satu codebase UI | Renderer desktop punya banyak fitur desktop-only (pet overlay, tray, HUD, starmap) yang tidak masuk akal di HP; bundle jadi besar |

**Rekomendasi: Opsi A**, dilakukan bertahap:

1. Buat paket workspace baru `apps/chat-ui` (tambahkan ke `workspaces` di `package.json` root).
2. Pindahkan komponen assistant-ui yang **murni presentasional** dulu — yang tidak menyentuh `window.hermes`: `markdown-text.tsx`, `ansi-text.tsx`, `artifact-card.tsx`, `tool/`, `message-render-boundary.tsx`.
3. Untuk komponen yang menyentuh `window.hermes`, ubah jadi menerima dependensi via **props atau React context**, bukan mengakses global. Ini yang membuatnya portabel.
4. Desktop mengimpor dari `apps/chat-ui` dan menyuntikkan bridge Electron-nya. Web mengimpor paket yang sama dan menyuntikkan bridge browser dari §2.2.
5. Verifikasi tidak ada regresi di desktop — `npm run check --workspace apps/desktop` menjalankan typecheck + lint + test UI + test platform.

#### 2.4 Arsitektur ChatPage baru

```typescript
// web/src/pages/ChatPage.tsx  (versi baru — menggantikan xterm)

import { JsonRpcGatewayClient } from "@hermes/shared";
import { Thread, Composer, MessageList } from "@hermes/chat-ui";

/**
 * ChatPage — native chat UI di atas /api/ws (JSON-RPC).
 *
 *   Browser
 *     └─ JsonRpcGatewayClient  ──WS──► /api/ws ──► tui_gateway ──► AIAgent
 *          │
 *          ├─ request:  prompt.submit, session.create, session.interrupt, ...
 *          └─ event:    message.delta, thinking.delta, tool.start, ...
 *                          │
 *                          ▼
 *                    reducer  →  React state  →  <MessageList>
 *
 * Menggantikan mirror PTY/xterm. Semua penanganan khusus mobile
 * (keyboard-inset, pty-mobile-input, pty-composition) menjadi tidak
 * relevan: <textarea> native menangani IME, autocorrect, dan dikte
 * sendiri, dan `interactive-widget=resizes-content` di index.html
 * mengurus keyboard.
 */
```

**Pemetaan event → UI:**

| Event JSON-RPC | Rendering |
|---|---|
| `message.start` | Buat bubble assistant baru (kosong, dengan indikator streaming) |
| `message.delta` | Append token ke bubble aktif; markdown di-render progresif |
| `message.interim` | Update draf sementara |
| `message.complete` | Finalisasi bubble; render markdown penuh + tombol copy |
| `thinking.delta` / `reasoning.delta` | Blok "Thinking" yang bisa dilipat (default tertutup di mobile) |
| `reasoning.available` | Tampilkan toggle reasoning |
| `tool.start` | Tool card, status *running*, spinner |
| `tool.progress` | Update isi/progress card |
| `tool.complete` | Card jadi *done*; output bisa dilipat |
| `tool.generating` | Indikator "menyiapkan pemanggilan tool" |
| `approval.request` | **Bottom sheet modal** dengan tombol Approve/Deny besar (≥44pt) |
| `clarify.request` | Kartu pertanyaan inline dengan input |
| `sudo.request` / `secret.request` | Sheet input aman |
| `status.update` | Strip status halus (bukan bubble) |
| `session.usage` | Chip token/biaya di header |
| `background.complete` | Notifikasi toast |
| `error` | Bubble error dengan tombol retry |

**Catatan UX khusus mobile untuk `approval.request`:** Hermes bisa meminta persetujuan sebelum menjalankan perintah shell. Di terminal ini muncul sebagai prompt teks. Di mobile, ini **harus** jadi bottom sheet dengan tombol besar dan teks perintah yang bisa di-scroll — inilah salah satu keuntungan UX terbesar dari pindah ke chat UI native, karena approval yang salah tekan di terminal HP adalah risiko nyata.

**Composer (input):**

```
┌─────────────────────────────────────────────────┐
│  ┌───────────────────────────────────────────┐  │
│  │ <textarea> auto-grow, max 40vh            │  │  ← native: IME,
│  │ enterKeyHint="send"                       │  │    autocorrect,
│  │ autocapitalize="sentences"                │  │    dikte, paste
│  └───────────────────────────────────────────┘  │
│  [📎]  [/ slash]              [◼ stop] [➤ kirim] │  ← target ≥44×44pt
└─────────────────────────────────────────────────┘
   ↑ pb-[env(safe-area-inset-bottom)]
   ↑ posisi mengikuti visualViewport saat keyboard muncul
```

Komponen yang sudah ada dan bisa dipakai langsung: `web/src/components/SlashPopover.tsx` (autocomplete slash command), `web/src/components/Markdown.tsx`, `web/src/components/ChatSessionList.tsx`, `web/src/components/ChatSidebar.tsx`.

#### 2.5 Rencana migrasi bertahap (jangan big-bang)

Ini penting: **jangan hapus jalur xterm dulu.**

```
Langkah 1  Feature flag: dashboard.chat_ui = "pty" | "native"   (default "pty")
Langkah 2  Bangun ChatPageNative.tsx paralel dengan ChatPage.tsx yang ada
Langkah 3  Route /chat  → pty (lama)
           Route /chat2 → native (baru)   ← untuk dites berdampingan
Langkah 4  Uji: streaming, tool call, approval, interrupt, resume sesi,
           multi-profil, reconnect setelah tab di-background di HP
Langkah 5  Balik default ke "native"; simpan "pty" sebagai escape hatch
Langkah 6  Setelah stabil beberapa rilis, pertimbangkan pindahkan xterm
           ke panel "Terminal" terpisah (seperti desktop app) — bukan hapus.
           Terminal tetap berguna sebagai tool, hanya bukan sebagai chat.
```

**Kenapa pertahankan xterm sebagai panel terpisah:** desktop app melakukan tepat itu (`apps/desktop/src/app/right-sidebar/terminal/`). Terminal adalah tool yang sah; masalahnya hanya ketika ia dipakai *sebagai* antarmuka chat.

**Risiko utama Fase 2 dan mitigasinya:**

| Risiko | Mitigasi |
|---|---|
| Merusak desktop app saat refactor komponen bersama | Jalankan `npm run check --workspace apps/desktop` di setiap langkah; suite-nya termasuk test UI + platform + parity (`hermes-parity.test.ts`) |
| Ada event yang tidak tertangani → pesan hilang tanpa jejak | Terapkan reducer *exhaustive*; log event tak dikenal ke console dan tampilkan bubble fallback, jangan diam-diam drop |
| Perbedaan perilaku halus vs TUI (mis. urutan interrupt, seed sesi) | Jalankan berdampingan di `/chat` vs `/chat2` dengan sesi yang sama |
| Bundle jadi lebih besar | Sebenarnya bisa **lebih kecil**: xterm + WebGL bisa jadi lazy-load setelah bukan lagi jalur chat utama |
| Kompleksitas reconnect di jaringan mobile | `apps/shared` sudah punya `reconnect-backoff`; `JsonRpcGatewayClient` sudah punya `ConnectionState`. Reuse, jangan tulis ulang. |

---

### FASE 3 — Navigasi & Layout Mobile (3–5 hari)

**Tujuan:** navigasi terasa native di HP, bukan sidebar desktop yang dipaksa jadi drawer.

Kondisi saat ini: **18 item nav** dalam satu drawer vertikal (`web/src/App.tsx:139-220`). Di desktop ini wajar. Di HP, drawer 18 item butuh scroll dan tidak ada hierarki.

#### 3.1 Bottom tab bar + overflow

Pola yang dipakai app Claude, ChatGPT, dan hampir semua app chat: **4–5 destinasi utama di bottom bar**, sisanya di "More".

```
┌─────────────────────────────────────┐
│  ☰  Hermes            ⋯   [avatar]  │ ← header ramping, judul kontekstual
├─────────────────────────────────────┤
│                                     │
│         KONTEN                      │
│         (scroll)                    │
│                                     │
├─────────────────────────────────────┤
│  💬      🕐      ⚡      ⚙️      ⋯  │ ← bottom tab bar
│ Chat  Sessions  Cron  Settings  More│   + safe-area-inset-bottom
└─────────────────────────────────────┘
```

Pengelompokan yang disarankan:

| Tab | Isi |
|---|---|
| **Chat** | `/chat` |
| **Sessions** | `/sessions` |
| **Cron** | `/cron` |
| **Settings** | Hub berisi: Models, Config, Keys, Profiles |
| **More** | Skills, Plugins, MCP, Channels, Webhooks, Pairing, Files, Logs, Analytics, System, Docs, tab plugin |

Implementasi: pertahankan tabel nav yang sudah ada sebagai satu-satunya sumber kebenaran; tambahkan field `mobileGroup` pada tiap entri, lalu render `<BottomTabBar>` saat `isMobile` dan `<Sidebar>` saat desktop. Jangan bikin tabel nav kedua — itu akan menyimpang.

Bottom bar **wajib** memakai `pb-[env(safe-area-inset-bottom,0px)]`, konsisten dengan pola yang sudah dipakai di `App.tsx:770`.

#### 3.2 Halaman padat tabel → card di mobile

Halaman seperti Sessions, Cron, Logs, Files, MCP, Channels kemungkinan besar memakai tabel. Tabel dengan >3 kolom tidak bisa dipakai di layar 390px.

Pola: di bawah breakpoint `sm`, ubah setiap baris tabel jadi **card**:

```
Desktop:  │ Nama        │ Jadwal      │ Terakhir  │ Status │ Aksi │
Mobile:   ┌──────────────────────────────────────┐
          │ Nama tugas                    ● aktif │
          │ Setiap hari 09:00                     │
          │ Terakhir: 2 jam lalu                  │
          │                       [Edit]  [Hapus] │
          └──────────────────────────────────────┘
```

Sisi baik: pola ini **sudah dipakai** di beberapa tempat (`ThemeSwitcher`/`LanguageSwitcher` sudah punya mobile sheet). Jadi ada preseden internal yang bisa diikuti, bukan pola asing.

#### 3.3 Detail sentuh & gestur

| Item | Aksi |
|---|---|
| Tap target | Minimal 44×44pt (iOS HIG) / 48×48dp (Material) untuk semua tombol |
| Swipe | Swipe dari kiri untuk buka drawer; swipe pada item sesi untuk hapus |
| Pull-to-refresh | Pada Sessions, Logs, Cron |
| Haptics | Repo sudah punya `web-haptics` di desktop (`haptics-provider.tsx`) — bisa dipakai di web untuk kirim pesan, approve, error |
| Long-press | Menu konteks pada pesan (copy, retry, branch) |
| Momentum scroll | `-webkit-overflow-scrolling: touch` pada kontainer scroll |
| Anti double-tap-zoom | `touch-action: manipulation` pada tombol |
| Font input ≥16px | Mencegah iOS Safari auto-zoom saat fokus input |

---

### FASE 4 — Hardening & Operasional (3–5 hari)

Dashboard ini punya akses ke **shell, filesystem, dan API keys** di VPS Anda. Mengeksposnya ke internet berarti panel itu **adalah** permukaan serangan. Perlakukan seperti panel admin produksi.

#### 4.1 Autentikasi berlapis

| Lapis | Cara | Prioritas |
|---|---|---|
| Auth Hermes | Password scrypt atau OAuth Nous Portal | **Wajib** |
| Set `dashboard.basic_auth.secret` | 32+ byte acak → sesi bertahan restart & multi-worker | **Wajib** |
| `session_ttl_seconds` | Turunkan dari default 12 jam (mis. 8 jam) | Disarankan |
| Cloudflare Access / forward-auth | Auth di edge, sebelum request menyentuh VPS | Sangat disarankan |
| **2FA / TOTP** | **Belum ada** — perlu dikembangkan sebagai `DashboardAuthProvider` plugin baru | Disarankan |
| Rate limit login | Di proxy (Caddy/nginx) atau fail2ban | Disarankan |
| IP allowlist | Kalau IP Anda cukup stabil | Opsional |

Soal 2FA: arsitektur plugin auth sudah ada (`plugins/dashboard_auth/` dengan interface `DashboardAuthProvider`, dan `supports_password` sebagai capability flag di `login_page.py:485`). Jadi menambahkan provider TOTP adalah **extension point yang sudah disediakan**, bukan hack — tapi tetap perlu ditulis.

#### 4.2 Alternatif paling aman: jangan ekspos ke internet sama sekali

Kalau yang Anda butuhkan hanya akses dari HP **Anda sendiri**, ini lebih aman daripada dashboard publik:

| Metode | Cara kerja | Trade-off |
|---|---|---|
| **Tailscale** ✅ paling direkomendasikan | Mesh VPN WireGuard. Install di VPS + HP. Akses `http://hermes-vps:9119` di jaringan privat. Bind Hermes ke IP Tailscale. | Perlu app Tailscale di HP. **Zero permukaan publik.** Ada Tailscale Serve untuk HTTPS otomatis di dalam tailnet. |
| **Cloudflare Tunnel** | `cloudflared` bikin koneksi keluar; tidak ada port terbuka. Bisa digabung Cloudflare Access (Google/GitHub SSO + 2FA). | Trafik lewat Cloudflare. Kode sudah menyebut Cloudflare Tunnel (idle timeout ~100s sudah diakomodasi di setting ws ping). |
| **WireGuard sendiri** | VPN yang Anda kelola penuh | Setup manual lebih ribet |
| **SSH tunnel** | `ssh -L 9119:localhost:9119` | Di HP butuh app SSH (Termius); tidak praktis untuk pemakaian harian |

Kode Hermes sendiri menyarankan Tailscale/SSH tunnel di beberapa pesan error (`web_server.py:18857`, `18894`) — jadi ini jalur yang memang diantisipasi.

**Kombinasi terbaik untuk kenyamanan + keamanan:** Tailscale + Tailscale Serve (HTTPS di tailnet) + auth Hermes tetap aktif. Anda dapat HTTPS (jadi PWA bisa jalan), tidak ada permukaan publik, dan tetap ada password sebagai lapis kedua.

#### 4.3 Operasional

**Unit systemd** — namai persis `hermes-dashboard.service` agar `hermes update` bisa me-restart-nya dengan benar (`hermes_cli/main.py:8150`):

```ini
[Unit]
Description=Hermes Agent Dashboard
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=hermes
WorkingDirectory=/opt/hermes-agent
Environment=HERMES_WEB_DIST=/opt/hermes-agent/hermes_cli/web_dist
ExecStart=/opt/hermes-agent/.venv/bin/python -m hermes_cli.main dashboard \
          --host 127.0.0.1 --port 9119 --no-open --skip-build
Restart=always
RestartSec=5

# Hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=read-only
ReadWritePaths=/home/hermes/.hermes

[Install]
WantedBy=multi-user.target
```

Catatan: `ProtectSystem`/`ProtectHome` mungkin perlu dilonggarkan tergantung tool apa yang Anda izinkan agent jalankan — agent ini memang dirancang untuk menyentuh filesystem. Sesuaikan, jangan copy-paste buta.

Lainnya:
- **Backup**: `~/.hermes/` berisi seluruh state, sesi, dan skill. Ada `hermes_cli/backup.py` (81 KB) — pakai itu. Jadwalkan lewat cron Hermes sendiri.
- **Monitoring**: `/api/health` dan `/api/status` sudah public-allowlisted khusus untuk uptime probe (`dashboard_auth/public_paths.py`) — pakai untuk Uptime Kuma / Better Stack.
- **Log**: `~/.hermes/logs/`; halaman Logs sudah ada di dashboard.
- **Resource**: `gateway/memory_monitor.py`, `gateway/disk_status.py`, `gateway/scale_to_zero.py` sudah ada. Dashboard juga melaporkan telemetri memori & disk via `/api/status` (`web/src/lib/api.ts:1904-1922`).

---

### FASE 5 — Push Notification & Offline (opsional, 1–2 minggu)

Berguna kalau Anda ingin tahu saat tugas panjang atau cron selesai, tanpa membuka app.

**Konsep:** Web Push (VAPID) + Push API.

```
Agent selesai tugas
   → tui_gateway emit background.complete
   → backend Python kirim Web Push (VAPID) ke endpoint tersimpan
   → Service Worker terima 'push' event
   → showNotification()  → muncul di lock screen HP
   → tap → buka /chat?session=<id>
```

Yang perlu dibuat:
1. Sepasang kunci VAPID; simpan di `.env`
2. Endpoint `POST /api/push/subscribe` untuk menyimpan subscription per user
3. Push sender di Python (`pywebpush`) yang dipicu dari event `background.complete` / cron
4. Handler `push` + `notificationclick` di Service Worker
5. UI izin notifikasi (jangan minta saat load pertama — minta saat user mengaktifkan fiturnya)

**Catatan iOS penting:** Web Push di iOS Safari **hanya** bekerja untuk PWA yang sudah di-**Add to Home Screen** (iOS 16.4+). Jadi Fase 1 adalah prasyarat mutlak. Ini alasan tambahan mengerjakan PWA lebih dulu.

Alternatif yang jauh lebih murah: Hermes **sudah** punya gateway ke 22 platform messaging, termasuk **ntfy** (`plugins/platforms/ntfy`) yang memang dibuat untuk push notification, dan Telegram. Anda bisa dapat notifikasi HP **hari ini, tanpa menulis kode apa pun**, dengan mengonfigurasi cron delivery ke ntfy atau Telegram. **Coba ini dulu sebelum membangun Web Push.**

---

## 9. Panduan Deploy VPS (Fase 0 — Bisa Dikerjakan Hari Ini)

Target: VPS Ubuntu 22.04/24.04, 2 vCPU / 2 GB RAM (1 GB bisa, tapi lihat catatan build di §10.3), domain sudah mengarah ke IP VPS.

### Langkah 1 — Persiapan sistem

```bash
sudo adduser --disabled-password --gecos "" hermes
sudo apt update && sudo apt install -y curl git ripgrep ffmpeg

# Node.js 22+ (wajib: engines.node >= 22.22.0)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### Langkah 2 — Install Hermes

```bash
sudo -iu hermes
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
source ~/.bashrc
hermes doctor        # verifikasi instalasi
```

### Langkah 3 — Setup agent

```bash
hermes setup                  # wizard: provider, model, API key
# atau, kalau pakai Nous Portal:
hermes setup --portal
```

### Langkah 4 — Konfigurasi auth dashboard

Ini langkah paling penting. Jangan dilewati.

```bash
# Generate hash scrypt (JANGAN simpan plaintext di config)
cd ~/.hermes  # atau direktori instalasi
python -c "from plugins.dashboard_auth.basic import hash_password; print(hash_password('PASSWORD-KUAT-ANDA'))"

# Generate secret penandatangan sesi
openssl rand -base64 32
```

Edit `~/.hermes/config.yaml`:

```yaml
dashboard:
  basic_auth:
    username: "admin"
    password_hash: "scrypt$16384$8$1$<salt>$<hash>"   # dari perintah di atas
    secret: "<hasil openssl rand -base64 32>"          # sesi bertahan restart
    session_ttl_seconds: 28800                         # 8 jam
  public_url: "https://hermes.domain-anda.com"         # penting untuk OAuth & cookie
```

Verifikasi provider terdaftar:
```bash
hermes plugins list | grep -i basic     # pastikan tidak ada di plugins.disabled
```

### Langkah 5 — Build frontend

```bash
cd /path/ke/hermes-agent
npm ci
npm run build --workspace web     # → hermes_cli/web_dist/
```

Kalau VPS Anda kecil, lihat [§10.3](#103-build-frontend-di-vps-kecil-bisa-oom) — build di laptop lalu rsync `web_dist/` ke VPS.

### Langkah 6 — Jalankan sebagai service

Pasang unit systemd dari [§4.3](#43-operasional) (namai `hermes-dashboard.service`):

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hermes-dashboard
sudo systemctl status hermes-dashboard
curl -s localhost:9119/api/health     # harus 200
```

Kalau juga ingin messaging gateway, buat `hermes-gateway.service` serupa dengan `ExecStart=... gateway run`.

### Langkah 7 — Reverse proxy + TLS (Caddy)

Caddy dipilih karena Let's Encrypt otomatis, tanpa konfigurasi tambahan.

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy
```

`/etc/caddy/Caddyfile`:

```caddyfile
hermes.domain-anda.com {
    encode zstd gzip

    # WebSocket (/api/pty, /api/ws) melewati reverse_proxy Caddy tanpa
    # konfigurasi khusus — Caddy meneruskan header Upgrade secara otomatis.
    reverse_proxy 127.0.0.1:9119 {
        header_up X-Forwarded-Proto {scheme}
        header_up X-Forwarded-Host  {host}
        # Agent turn bisa berjalan lama. Naikkan timeout, jangan biarkan
        # proxy memutus di tengah tugas panjang.
        transport http {
            read_timeout   1h
            write_timeout  1h
        }
    }

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options    "nosniff"
        X-Frame-Options           "DENY"
        Referrer-Policy           "strict-origin-when-cross-origin"
    }

    log {
        output file /var/log/caddy/hermes.log
    }
}
```

```bash
sudo systemctl reload caddy
```

**Kalau Anda pakai bind loopback (pilihan A di §Fase 0)**, auth gate Hermes **tidak** aktif — tambahkan auth di Caddy:

```caddyfile
    basic_auth {
        admin <hash-bcrypt-dari: caddy hash-password>
    }
```

Atau lebih baik: pakai Cloudflare Access / forward-auth di depannya.

**Kalau Anda pakai bind `0.0.0.0` (pilihan B)** — auth gate Hermes aktif, tapi **firewall port 9119** agar hanya proxy yang bisa mengaksesnya:

```bash
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 80,443/tcp
sudo ufw enable
# 9119 TIDAK dibuka — hanya diakses dari localhost oleh Caddy
```

### Langkah 8 — Verifikasi dari HP

1. Buka `https://hermes.domain-anda.com` di Safari/Chrome HP
2. Halaman login muncul (sudah responsif) → login
3. Cek drawer nav (tap ikon hamburger)
4. Cek beberapa halaman: Config, Cron, Sessions, Keys
5. Cek tab Chat — akan berupa terminal; **ini yang diperbaiki di Fase 2**
6. Cek safe area di iPhone ber-notch (tidak ada konten terpotong)
7. Cek keyboard virtual tidak menutupi input

### Opsional: hosting di subpath

Kalau ingin `https://domain-anda.com/hermes/` (bukan subdomain):

```caddyfile
domain-anda.com {
    handle_path /hermes/* {
        reverse_proxy 127.0.0.1:9119 {
            header_up X-Forwarded-Prefix "/hermes"
            header_up X-Forwarded-Proto  {scheme}
            header_up X-Forwarded-Host   {host}
        }
    }
}
```

`mount_spa()` akan menulis ulang `index.html` agar URL aset dan `__HERMES_BASE_PATH__` menghormati prefix itu — **tanpa perlu rebuild bundle** (`web_server.py:17285-17290`).

---

## 10. Risiko, Trade-off, dan Catatan Keamanan

### 10.1 Ini panel yang sangat berkuasa — perlakukan sesuai

Dashboard ini memberi akses ke:
- **Halaman Keys** (`/env`) — semua API key dalam bentuk yang bisa dibaca/diedit
- **Halaman Files** (`/files`) — browse, upload, download, delete di filesystem VPS
- **Chat** — agent bisa menjalankan perintah shell arbitrer
- **Halaman Config** — seluruh perilaku agent, termasuk pengaturan approval
- **Halaman MCP** — menambah MCP server (jalur *persistence* yang dieksploitasi kampanye `hermes-0day`)

Yang berhasil menembus dashboard ini **memiliki VPS Anda dan semua API key Anda**. Karena itu: password kuat + HTTPS + idealnya Tailscale/Cloudflare Access, dan pertimbangkan serius membangun provider TOTP.

### 10.2 Mode loopback tidak punya auth — jangan sampai keliru

`should_require_auth()` mengembalikan `False` untuk `127.0.0.1`. Jadi dashboard yang bind loopback **tidak meminta login sama sekali** — asumsinya operator lokal sudah dipercaya.

Konsekuensi yang mudah terlewat: kalau Anda memasang reverse proxy ke dashboard loopback **tanpa** auth di proxy, Anda baru saja mengekspos dashboard tanpa autentikasi apa pun ke internet — dan Hermes tidak akan memperingatkan, karena dari sudut pandangnya ia hanya bind ke loopback.

**Aturannya: kalau bind loopback, auth WAJIB ada di proxy. Kalau auth ada di Hermes, bind non-loopback (dan firewall port-nya).** Jangan sampai jatuh di antara keduanya.

### 10.3 Build frontend di VPS kecil bisa OOM

`hermes dashboard` akan otomatis menjalankan `npm install` + `vite build` bila `web_dist` stale (`hermes_cli/main.py:6280`). Bundle dashboard ini besar: React 19 + three.js + xterm + WebGL + Observable Plot. Vite build bisa memakai >1 GB RAM.

Di VPS 1 GB, ini bisa OOM-kill di tengah build — dan yang lebih buruk, bisa terjadi **saat `hermes update`**, membuat dashboard mati di waktu yang tidak Anda duga.

**Mitigasi (pilih satu):**

| Cara | Perintah |
|---|---|
| **Prebuild di mesin lain** ✅ disarankan | Build di laptop/CI → `rsync -a hermes_cli/web_dist/ vps:/opt/hermes-agent/hermes_cli/web_dist/` → jalankan dengan `--skip-build` |
| Override lokasi dist | Set `HERMES_WEB_DIST=/path/ke/dist` (`web_server.py:138`) |
| Tambah swap | `fallocate -l 2G /swapfile && mkswap /swapfile && swapon /swapfile` |
| Pakai image Docker | Image resmi sudah membawa dist yang ter-build |

Selalu sertakan `--skip-build` di unit systemd produksi. Build adalah langkah deploy, bukan langkah boot.

### 10.4 Sesi WebSocket panjang vs proxy

Agent turn bisa berjalan lama (menit, bahkan lebih dengan subagent). Kode sudah menangani ini di sisi Hermes:
- `ws_ping_interval=20.0` untuk bind non-loopback, agar tetap di bawah idle timeout Cloudflare Tunnel (~100s)
- Komentar di `web_server.py:18965-18979` menjelaskan bahwa event-loop stall bisa mencapai **226 detik** (issue #53773)

Yang perlu **Anda** lakukan: naikkan timeout di reverse proxy Anda (contoh Caddy di §9 Langkah 7 sudah menyetel 1 jam). Default nginx (60s) akan memutus turn panjang.

### 10.5 Code skew antara backend dan frontend

Repo punya `gateway/code_skew.py` — artinya ketidaksepadanan versi backend/frontend adalah masalah yang sudah dikenal. Setelah `hermes update`, **selalu rebuild frontend** dan restart dashboard. Kalau Anda menambahkan Service Worker (Fase 1), risiko ini naik — karena itu `registerType: "prompt"`, bukan `"autoUpdate"`.

### 10.6 Jangan lupa jalur yang lebih sederhana

Sebelum berinvestasi berminggu-minggu di Fase 2, tanyakan: **apakah Anda benar-benar butuh web chat, atau butuh chat dari HP?**

Hermes **sudah** punya gateway ke 22 platform. Chat via **Telegram** di HP memberi Anda:
- UI chat native yang sudah sempurna (bubble, markdown, gambar, voice note, notifikasi)
- Push notification gratis
- Transkripsi voice memo (sudah ada di Hermes)
- Kontinuitas percakapan lintas platform
- Nol pekerjaan pengembangan

Kombinasi yang mungkin **paling praktis**, dan yang saya sarankan Anda evaluasi dulu:

```
Chat harian     →  Telegram di HP        (UX native sempurna, 0 hari kerja)
Setting & admin →  Web dashboard di HP   (sudah responsif, Fase 0 + 1 saja)
```

Ini memberi 90% dari tujuan Anda dengan sekitar **3–4 hari kerja**, bukan 1–2 bulan.

Fase 2 (native chat UI) tetap layak dikerjakan kalau Anda spesifik ingin: chat dan pengaturan dalam satu antarmuka, kontrol penuh atas UX chat, tidak bergantung pihak ketiga, atau memang ingin mengkontribusikan perbaikan ini ke upstream. Itu keputusan Anda — tapi keputusannya sebaiknya dibuat dengan tahu bahwa ada jalur 3 hari yang mendekati hasil yang sama.

---

## 11. Roadmap & Estimasi Effort

### Ringkasan

| Fase | Pekerjaan | Effort | Hasil |
|---|---|---|---|
| **0** | Deploy VPS + TLS + auth | **0.5–1 hari** | Dashboard bisa dipakai dari browser HP, aman |
| **1** | PWA (manifest, SW, ikon) | **2–3 hari** | Add to Home Screen, fullscreen, terasa seperti app |
| **2** | Native chat UI via JSON-RPC | **2–4 minggu** | UX chat seperti app Claude ◄ inti |
| **3** | Navigasi & layout mobile | **3–5 hari** | Bottom tab bar, card layout, gestur |
| **4** | Hardening & operasional | **3–5 hari** | Siap produksi: TOTP, monitoring, backup, systemd |
| **5** | Push notification | **1–2 minggu** | Notifikasi native (opsional — coba ntfy/Telegram dulu) |

**Jalur minimum untuk "bisa dipakai dari HP":** Fase 0 → **1 hari**
**Jalur pragmatis (rekomendasi saya):** Fase 0 + 1 + Telegram untuk chat → **3–4 hari**
**Jalur lengkap sesuai visi Anda:** Fase 0 → 1 → 2 → 3 → 4 → **6–9 minggu**

### Urutan eksekusi yang disarankan

```
Minggu 1    Fase 0  Deploy + TLS + auth          ► pakai dari HP
            Fase 1  PWA                          ► terasa seperti app
            ── Evaluasi di sini: coba Telegram untuk chat harian.
               Kalau sudah cukup, Anda selesai. Kalau tidak, lanjut. ──

Minggu 2–3  Fase 2a Ekstrak assistant-ui ke paket workspace bersama
                    Bikin browser bridge (shim window.hermes)
                    Verifikasi desktop app tidak regresi

Minggu 4–5  Fase 2b ChatPageNative di /chat2, di belakang feature flag
                    Petakan seluruh event stream ke komponen UI
                    Uji: streaming, tool, approval, interrupt, reconnect

Minggu 6    Fase 3  Bottom tab bar, card layout, gestur
                    Balik default chat ke native

Minggu 7    Fase 4  TOTP provider, monitoring, backup terjadwal, systemd
                    ── Rilis ──

Nanti       Fase 5  Web Push (hanya kalau ntfy/Telegram terbukti tidak cukup)
```

### Definition of Done

**Fase 0**
- [ ] `https://domain/` melayani dashboard dengan sertifikat valid
- [ ] Login diperlukan; password tersimpan sebagai hash scrypt, bukan plaintext
- [ ] `dashboard.basic_auth.secret` diset (sesi bertahan restart)
- [ ] Port 9119 tidak dapat dijangkau dari internet
- [ ] Dashboard berjalan sebagai `hermes-dashboard.service`, restart otomatis
- [ ] Timeout proxy ≥ 1 jam (turn panjang tidak terputus)
- [ ] `--skip-build` dipakai; dist di-prebuild
- [ ] Diverifikasi di Safari iOS **dan** Chrome Android

**Fase 1**
- [ ] Lighthouse: PWA installable
- [ ] Add to Home Screen berfungsi di iOS & Android; ikon tampil benar
- [ ] Mode standalone: tidak ada address bar; safe area benar
- [ ] `/api/*` dan `/auth/*` **tidak pernah** dilayani dari cache — diverifikasi
- [ ] Login masih berfungsi setelah SW aktif (regresi paling umum)
- [ ] Offline → shell + pesan jelas, bukan error browser
- [ ] Update memunculkan prompt reload, bukan auto-swap

**Fase 2**
- [ ] `/chat` merender bubble pesan, bukan grid terminal
- [ ] Markdown, blok kode dengan syntax highlight, tombol copy
- [ ] Streaming token mulus; ada indikator thinking
- [ ] Tool call jadi card yang bisa dilipat, dengan status
- [ ] `approval.request` jadi bottom sheet dengan tombol ≥44pt
- [ ] Input: `<textarea>` native — autocorrect, dikte, paste, undo berfungsi
- [ ] Seleksi & copy teks native berfungsi
- [ ] Interrupt / steer / redirect berfungsi
- [ ] Resume sesi + histori berfungsi
- [ ] Reconnect setelah tab di-background di HP berfungsi
- [ ] **Desktop app tidak regresi** (`npm run check --workspace apps/desktop` hijau)
- [ ] Jalur xterm masih tersedia di belakang flag / sebagai panel terminal terpisah

---

## Lampiran A — Peta Referensi File Kunci

| Kebutuhan | File | Baris kunci |
|---|---|---|
| Server dashboard (FastAPI) | `hermes_cli/web_server.py` | 19.157 baris total |
| Kebijakan auth gate | `hermes_cli/web_server.py` | `641` `should_require_auth()` |
| Guard Host header (anti DNS-rebind) | `hermes_cli/web_server.py` | `663` `_is_accepted_host()` |
| Lokasi dist frontend | `hermes_cli/web_server.py` | `138` `WEB_DIST` |
| Mount SPA + prefix proxy | `hermes_cli/web_server.py` | `17278` `mount_spa()` |
| Startup server + uvicorn | `hermes_cli/web_server.py` | `18798` `start_server()` |
| Guard peer WebSocket | `hermes_cli/web_server.py` | `15805` `_ws_client_is_allowed()` |
| Framework auth dashboard | `hermes_cli/dashboard_auth/` | `middleware.py`, `routes.py`, `ws_tickets.py` |
| Halaman login (mobile-friendly) | `hermes_cli/dashboard_auth/login_page.py` | `40` viewport meta |
| Allowlist path publik | `hermes_cli/dashboard_auth/public_paths.py` | seluruh file |
| Provider password scrypt | `plugins/dashboard_auth/basic/__init__.py` | `115` `hash_password()` |
| Default config dashboard | `hermes_cli/config_defaults.py` | `1515` blok `basic_auth` |
| Flag CLI dashboard | `hermes_cli/subcommands/dashboard.py` | seluruh file |
| Auto-build frontend | `hermes_cli/main.py` | `6280` `_build_web_ui()` |
| Konvensi unit systemd | `hermes_cli/main.py` | `8150` `_DASHBOARD_SYSTEMD_UNIT` |
| Shell SPA + tabel nav | `web/src/App.tsx` | `139-220` nav, `396` `isMobile`, `770` safe-area |
| Viewport meta | `web/index.html` | `6-9` |
| **Chat via xterm (gap #1)** | `web/src/pages/ChatPage.tsx` | `1-17` header arsitektur |
| Penanganan keyboard mobile | `web/src/lib/keyboard-inset.ts` | `computeKeyboardInset()` |
| Input mobile untuk PTY | `web/src/lib/pty-mobile-input.ts` | seluruh file |
| Reconnect PTY | `web/src/lib/pty-reconnect.ts` | seluruh file |
| Client REST | `web/src/lib/api.ts` | 2.600+ baris |
| Output build Vite | `web/vite.config.ts` | `87` `outDir` |
| **Protokol JSON-RPC (solusi gap #1)** | `apps/shared/src/json-rpc-gateway.ts` | `1-24` `GatewayEventName` |
| Resolusi URL WS + ticket | `apps/shared/src/websocket-url.ts` | seluruh file |
| **Chat UI native (sumber porting)** | `apps/desktop/src/components/assistant-ui/` | seluruh direktori |
| SDK desktop (JSON-RPC) | `apps/desktop/src/hermes.ts` | `1` import `JsonRpcGatewayClient` |
| Bridge Electron (yang perlu di-shim) | `apps/desktop/src/global.d.ts` | `21-57` |
| Backend JSON-RPC | `tui_gateway/server.py` | method `prompt.*`, `session.*`, `config.*` |
| Docker compose | `docker-compose.yml` | `63-76` service dashboard |

## Lampiran B — Ringkasan Verdict per Pertanyaan

**"Codebase apa ini?"**
Hermes Agent — AI agent otonom yang bisa memperbaiki diri, buatan Nous Research. MIT license, ±9.700 file, matang dan aktif dimaintain.

**"Tech stack apa saja?"**
Backend: Python 3.11–3.13, FastAPI, uvicorn, Pydantic 2, SQLite+FTS5, OpenAI SDK.
Frontend: React 19, TypeScript 6, Vite 8, Tailwind 4, xterm.js.
Desktop: Electron 40 + React (assistant-ui).
TUI: React di terminal (Ink) via esbuild.
Infra: Docker + s6-overlay, Nix flake, npm workspaces, GitHub Actions.

**"Bagaimana cara menjalankannya?"**
`curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash` lalu `hermes setup` dan `hermes` (TUI) / `hermes dashboard` (web) / `hermes gateway start` (messaging). Dari source: `uv sync --extra all && npm ci && npm run build --workspace web`.

**"Apakah bisa dideploy di VPS dengan tampilan web dan dibuka di browser smartphone?"**
**Ya, bisa — hari ini, dengan sekitar 1 hari kerja dan tanpa mengubah kode.** Web dashboard React sudah ada dengan 19 halaman, sudah responsif dengan penanganan mobile yang serius (safe-area, dynamic viewport, keyboard inset, drawer), dan sudah punya auth gate wajib yang fail-closed untuk bind publik. Yang perlu Anda tambahkan hanya reverse proxy + TLS.

**"Kalau belum, bagaimana strateginya agar UX-nya seperti app Claude di smartphone?"**
Tiga pekerjaan, diurut berdasarkan rasio dampak/effort:
1. **PWA** (2–3 hari) — manifest + service worker + ikon → bisa di-install, fullscreen, terasa native. Effort kecil, dampak persepsi besar.
2. **Native chat UI** (2–4 minggu) — pindahkan tab Chat dari mirror terminal xterm.js ke JSON-RPC + komponen assistant-ui. Ini gap sebenarnya. Kabar baiknya: **backend, protokol, dan komponen React-nya semua sudah ada di repo ini** (dipakai desktop app), jadi ini porting, bukan membangun baru. Pekerjaan utamanya adalah membuat shim `window.hermes` untuk browser dan mengekstrak komponen ke paket workspace bersama.
3. **Navigasi mobile** (3–5 hari) — bottom tab bar menggantikan drawer 19 item, tabel jadi card, tap target 44pt, gestur.

Dan satu catatan jujur: sebelum berinvestasi 1–2 bulan di poin 2, coba dulu **Telegram** untuk chat harian (sudah didukung, nol kode) + web dashboard untuk pengaturan. Kombinasi itu memberi sekitar 90% tujuan Anda dalam 3–4 hari. Poin 2 tetap layak kalau Anda memang ingin semuanya dalam satu antarmuka.
