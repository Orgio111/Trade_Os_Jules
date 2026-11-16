# Trade_agent OS v2 — Бүрэн Автомат, Өөрөө Суралцдаг Trading OS-ийн Нарийвчилсан Төлөвлөгөө

Энэхүү баримт бичиг нь 48 зургийн бүх санаа, `Trade_agent` (Python, Rust, Go, TimesFM 2.5, Hermes, Ollama, Docker, Telegram) стек болон хэт хөнгөн, хурдан өөрөө суралцах шийдлүүдийг **давхцалгүйгээр**, яг тоо ширхэг, бүтэц, холболт болон ажиллуулах дараалалтай нь нэгтгэсэн эцсийн инженерийн зураглал юм.

---

## 1. Системийн Нийт Бүрэлдэхүүн (Тоо ширхэг ба Нөөцийн Хуваарилалт)

Системийг **Локал хурдан цөм (`< 1 GB RAM`)** болон **Үнэгүй Cloud / Colab T4 (`0 локал ачаалал`)** гэж хоёр хувааснаар локал компьютер эсвэл $5 VPS огт гацахгүй.

| Бүрэлдэхүүн | Тоо ширхэг | Технологи / Хэрэгсэл | Үндсэн үүрэг ба Нөөц |
| :--- | :--- | :--- | :--- |
| **Микросервис** | **6 сервис** | Go, Rust, Python, DuckDB/Drive, Mini-Hermes, Telegram | Өгөгдөл, эрсдэл/гүйцэтгэл, сургалт, судалгаа, хадгалалт, дохио |
| **Шууд арилжааны AI** | **4 загвар** | [ONNX Runtime](https://onnxruntime.ai/docs/) (LightGBM) + [Vowpal Wabbit](https://vowpalwabbit.org/tutorials/contextual_bandits.html) + [TimesFM 2.5](https://github.com/google-research/timesfm/blob/master/timesfm-forecasting/SKILL.md) + [River](https://riverml.xyz/) | Микросекунд/миллисекунд шийдвэр (**LLM огт оролцохгүй**) |
| **Судалгааны AI Роль** | **15 роль** | **1 LLM = 15 Prompt** (OpenRouter / NVIDIA NIM / Ollama `cron`) | Тусдаа 15 процесс биш, ганц модель prompt-оор ээлжилж ажиллана |
| **Токен & Контекст хяналт** | **2 хөдөлгүүр** | [`tiktoken-rs`](https://github.com/zurawiki/tiktoken-rs) + [`tokenizers`](https://github.com/huggingface/tokenizers) | 0 ms локал тоолуур, pre-flight тайралт, 75% rolling compaction |
| **Өгөгдлийн эх сурвалж** | **6 төрөл** | WebSocket, OpenBB, MT5, News/Calendar, Social, On-chain | API-гүй JavaScript сайтыг Rust [`obscura`](https://github.com/h4ckf0r0day/obscura)-аар татна |
| **Database бүтэц** | **2 давхарга** | Локал **DuckDB / Parquet** (`<24h`) + **Google Drive** (`>=24h`) | Hot-оос Cold руу цаг/өдрөөр архивлагдах 6 хүснэгтийн схем |
| **Validation Gate** | **6 шат** | `VectorBT` + 2-Stage Early Pruning (Colab T4) | 5,000–50,000 санаанаас 3–5 шилдгийг шүүх |
| **Хатуу эрсдэлийн дүрэм** | **8 дүрэм** | Rust (AI-аас бүрэн тусгаарлагдсан) | Position size, exposure, daily loss, kill switch, stale check |
| **Мэдэгдэл ба Буцаалт** | **8 триггер** | Telegram Bot + Health Monitor | Drift, slippage, rejection, алдагдал, автомат rollback |

---

## 2. 6 Микросервис, Мини-Hermes ба `obscura` / `Tauri v2` Бүтэц

Бүх сервис нэг `docker-compose.yml` эсвэл Rust workspace дотор ажиллах боловч **Live Execution (Rust)** болон **AI Research** хоёр хэзээ ч шууд эрх солилцохгүй.

1. **`svc-data-hub` (Go — Port `8081`) + `obscura` Sidecar:**
   - Биржүүдийн WebSocket (`Hyperliquid`, `MT5`, `Binance/Bybit`, `Alpaca`)-аас OHLCV, tick, orderbook, spread татна.
   - API-гүй эсвэл JavaScript-ээр ачаалдаг сайт (TradingView, ForexFactory, CoinGlass) унших шаардлагатай үед Rust workspace дотор `tokio::spawn`-аар хамт асдаг (auto-restart supervisor-той) **[`obscura`](https://github.com/h4ckf0r0day/obscura)** headless браузерийг локал gRPC/Unix socket-оор дуудаж JSON дата авна. Бүх дата API-аар ирдэг бол `obscura`-г унтрааж нөөц хэмнэнэ.
   - **TimesFM 2.5 (Candle-Close горим):** [TimesFM 2.5 (`200M`)](https://github.com/google-research/timesfm/blob/master/timesfm-forecasting/SKILL.md) нь ~800 MB диск, ~1.5 GB RAM / ~1 GB VRAM шаардаж, нэг таамгийг **~86–450 ms**-д гаргадаг. Иймээс tick бүрт биш, зөвхөн **`1m / 5m / 15m / 1h` лаа хаагдах бүрт** ганц удаа дуудаж гарсан чиглэл/магадлалыг feature болгон хавсаргана.
   - **Clock Sync & Data Quality:** Бүх эх сурвалжийг UTC миллисекундэд синхрончилж, 5 секундээс илүү саатсан өгөгдөлд `stale_flag = true` тавина.
2. **`svc-risk-exec` (Rust — Port `8082`):**
   - **AI-аас бүрэн тусгаарлагдсан цөм.** Дотроо [ONNX Runtime (`onnxruntime` crate)](https://docs.rs/onnxruntime) агуулж, `.onnx` загварыг `< 1 ms`-д уншина. Шинэ загвар ирэхэд **atomic pointer swap**-аар тасалдалгүй **hot-swap** хийнэ.
   - Захиалга бүрийг 8 хатуу дүрмээр шалгаад `Submitted → Accepted → Partially Filled → Filled → Closed` төлөвөөр гүйцэтгэнэ.
3. **`svc-trainer` (Python — Port `8083` + OpenRouter Codegen):**
   - Шууд арилжаанд оролцдог **Vowpal Wabbit** (стратеги сонгогч), **River** (ADWIN drift илрүүлэгч), **LightGBM (`init_model` / `refit`)** болон **Shadow Gate**-ийг хариуцна.
   - **OpenRouter Pre-Train Codegen:** Жинхэнэ сургалт эхлэхээс өмнө feature schema болон алдааны сургамжийг **OpenRouter** руу илгээж `train_candidate.py` үүсгүүлнэ. Үүссэн кодыг шууд ажиллуулахгүйгээр эхлээд **`ast.parse()` + 100 ширхэг хиймэл мөр дээр 5 секундийн dry-run + `.onnx` экспорт шалгалт** давсан тохиолдолд л сургалт руу оруулна.
4. **`strategy_studio` & Мини-Hermes (Rust Orchestrator + VS Code Colab T4):**
   - **Reverse-Engineered Мини-Hermes (Rust):** [`NousResearch/hermes-agent`](https://github.com/nousresearch/hermes-agent)-ийн хүнд Python daemon-ийг бүтнээр нь авалгүй, зөвхөн **4 цөм хэсгийг Rust дээр (~250–350 мөр, ~10–25 MB RAM)** бичнэ: (1) `reqwest` + `serde_json` OpenRouter/Ollama tool-calling loop, (2) Persistent Memory (`sqlx`/`rusqlite`), (3) `tokio-cron-scheduler`, (4) Telegram + MCP клиент.
   - **Startup + Гар идэвхжүүлэлт:** Trading OS асахад идэвхтэй стратеги цөөн эсвэл >24 цаг болсон бол `strategy_studio` background-аар автоматаар ажиллана. Мөн хүссэн үедээ **CLI** (`trade-os strategy generate`), **Telegram** (`/new_strategy`), эсвэл **HTTP/UI товч** (`POST /api/strategy/generate`)-аар гараар идэвхжүүлнэ.
   - **Zero-Copy VS Code → Colab T4:** Албан ёсны **[Google Colab VS Code Extension](https://marketplace.visualstudio.com/items?itemName=Google.colab)**-оор локал `.ipynb` дээр `Select Kernel → Colab → T4 GPU` холбож, Drive/Git-ээр синк хийгдсэн локал `.py` кодуудыг `%run /content/drive/MyDrive/Trade_agent/...` гэж copy-долгүй ажиллуулна.
5. **`svc-memory` (Hot DuckDB/Parquet `<24h` + Cold Google Drive `>=24h` + MLflow):**
   - PostgreSQL-ийн raw файлыг Drive sync дээр шууд тавьж эвдэхгүйн тулд **Hot/Cold** хуваана (хүснэгтийн бүтцийг 4-р хэсгээс харна уу).
6. **`svc-monitor-alert` & Optional `Tauri v2` Desktop UI:**
   - Telegram руу дохио явуулж, ноцтой үед Kill Switch / Rollback асаана. Хэрэв танд график харах, `strategy_studio` гараар асаах, Kill Switch дарах дэлгэцийн програм хэрэгтэй бол Electron-ий оронд Rust **[`Tauri v2`](https://v2.tauri.app/)** (~30–60 MB RAM) ашиглана.

---

## 3. Токен Хэмнэлт ба Context Window Удирдлага (`tiktoken-rs`)

Мини-Hermes агент нь OpenRouter болон бусад LLM API-ийн зардлыг **50–80% бууруулж**, контекст дүүрэхээс хамгаалахын тулд хүсэлт илгээх бүрийн өмнө **[`tiktoken-rs`](https://github.com/zurawiki/tiktoken-rs)** (`0.12.1`) ашиглан локал дээр 0 ms-д хэмжилт хийнэ (open-weight моделд [`huggingface/tokenizers`](https://github.com/huggingface/tokenizers), харин Gemini/Claude-д `tiktoken-rs` + **10–15% safety buffer** ашиглана).

**Токен зарцуулалт бууруулах 4 шүүлтүүр:**
- **Pre-flight Truncation:** `obscura`-аар татсан урт мэдээ эсвэл `trade_memory`-ийн 50 арилжааны логийг илгээхийн өмнө `tiktoken-rs`-ээр тоолж, `max_budget = 2,000 токен`-оос хэтэрсэн илүү хэсгийг автоматаар тайрч эсвэл хураангуйлна.
- **RAG (`trade_memory`) оновчлол:** Мэдээ болон түүхийг **300–500 токеноор** жижиглэж хадгалснаар бүтэн баримт биш, зөвхөн хамгийн хамааралтай 2–3 хэсгийг (~1,000 токен) LLM рүү явуулна.
- **Хоосон алдаа ба Retry зогсоох:** Context window дүүрч API алдаа заан олон дахин retry хийж токен үрэхээс урьдчилан сэргийлнэ.
- **Формат шахалт (Minification):** Илүү хоосон зай, давтагдсан JSON түлхүүрүүдийг хасаж компакт CSV/товч бүтцээр илгээж токен зардлыг **30–50%** шууд багасгана.

**Context Window дүүргэхгүй барьж шинэчлэх 4 арга:**
1. **Tool-ийн урт үр дүнг 1 мөрөөр солих:** Агент том хүснэгт эсвэл вэб хуудсыг уншиж дүгнэлтээ гаргасны дараа тэр 4,000 токены түүхий текстийг түүх дотроос устгаж, оронд нь `"[Уншсан: 5,000 мөр шалгав, шилдэг Sharpe=1.82]"` гэсэн ганц мөр үлдээнэ.
2. **Sliding Window (Гулсах цонх):** `System Prompt` болон үндсэн даалгаврыг байнга эхэнд нь хадгалаад, зөвхөн **хамгийн сүүлийн 4–6 мессежийг** л үлдээж хуучныг нь тасдаж хаяна.
3. **Rolling Compaction (75% босго дээр шинэчлэх):** `tiktoken-rs`-ээр хэмжихэд контекстийн **70–75%** дүүрмэгц, хуучин хэсгийг нь ганц дуудалтаар 5–10 өгүүлбэр болгон хураангуйлаад (`Summary`), тэр хураангуйгаар **шинэ цэвэр контекст** эхлүүлнэ.
4. **Санах ой руу буулгаад 0-лэх (`State Offload`):** Шалгарсан стратеги, алдааны дүгнэлтийг `DuckDB` / `trade_memory` руу хадгалмагц тухайн сессийн яриаг **бүрэн тэглэнэ** — дараагийн цикл эхлэхэд санах ойгоос зөвхөн хэрэгтэй хэсгийг нь татаж авна.

---

## 4. Өгөгдлийн Сангийн Схем (Hot DuckDB `<24h` + Cold Google Drive `>=24h`)

- **Hot давхарга (`0–24 цаг`, Локал SSD):** Шууд явж буй арилжаа, live үнэ, `River`/`Bandit`-ийн датаг локал **DuckDB** (`live_state.duckdb`) болон `.parquet`/`.jsonl` файлд миллисекундэд бичнэ.
- **Cold/Archive давхарга (`>= 24 цаг` ба хуучин дата, Google Drive):** Цаг тутам эсвэл 24 цаг тутам хасагдсан хуучин дата болон урсгалын хуулбарыг `.parquet` руу шахаж (`rclone` / Drive for Desktop) `MyDrive/TradeOS/archive/YYYY-MM-DD.parquet` руу хадгална. 24+ цагийн моделиуд (`LightGBM`, `VectorBT`, `LoRA`) Colab дотроос Drive-ийн `.parquet`-ийг шууд уншиж сургана.

**6 Үндсэн Хүснэгт (DuckDB / Parquet / `pgvector` схем):**
1. **`market_candles`:** `symbol`, `timeframe`, `ts_utc`, `open`, `high`, `low`, `close`, `volume`, `spread`, `atr_14`, `timesfm_pred`, `regime_tag`.
2. **`trade_memory`:** `trade_id`, `strategy_id`, `strategy_version`, `symbol`, `timeframe`, `entry_time`, `exit_time`, `entry_price`, `exit_price`, `position_size`, `slippage`, `commission`, `pnl_net`, `mae`, `mfe`, `exit_reason`, `hypothesis`, `market_context` (JSON), `decision`, `tags`, `context_embedding` (300–500 токен chunk-тай вектор).
3. **`experiments`:** `experiment_id`, `strategy_template`, `params_json`, `dataset_range`, `is_sharpe`, `oos_sharpe`, `wf_consistency`, `max_dd`, `profit_factor`, `total_trades`, `status`.
4. **`strategy_registry`:** `version_id`, `git_commit_sha`, `onnx_model_path`, `onnx_sha256`, `parent_version`, `status` (`candidate | shadow | paper | live | rolled_back`), `deployed_at`.
5. **`orders_audit`:** `order_id`, `strategy_version`, `symbol`, `side`, `qty`, `expected_price`, `fill_price`, `slippage_bps`, `lifecycle_state`, `risk_check_reason`.
6. **`system_alerts`:** `alert_id`, `severity`, `category`, `message`, `resolved`.

---

## 5. 15 AI Агент ба Роль (1 LLM = 15 Prompt)

Эдгээр 15 роль нь тусдаа 15 процесс биш, **Мини-Hermes** дотроос нэг LLM (OpenRouter / Ollama) руу тусгай prompt-оор ээлжилж дуудагдана:

- **Ажиглалт ба Мэдээлэл (4 роль):** `1. Scout / Scanner` (эзлэхүүн, гажилт), `2. News & Catalyst` (макро хуанли, мэдээ), `3. Sentiment` (Bullish/Bearish/Hype оноо), `4. Macro / Regime` (Dalio-ийн 4 квадрант + хэлбэлзлийн дэглэм).
- **Стратеги Судалгаа (7 роль):** `5. Trend Following`, `6. Mean Reversion`, `7. Volatility & Breakout`, `8. Statistical / Quant` (TimesFM 2.5 + cross-asset), `9. Session Analysis` (Asia/London/NY), `10. Event-Driven`, `11. Exit Strategy` (trailing/ATR/partial exit).
- **Код, Шалгалт, Багц (4 роль):** `12. Coding Agent`, `13. Independent Critic / QA` (look-ahead, leakage, overfitting шүүх), `14. Portfolio & Sizing` (корреляци + Kelly/ATR), `15. Decision Orchestrator` (`BUY / SELL / WAIT / SKIP`).

---

## 6. 5,000–50,000 Стратеги (2-Stage Early Pruning) ба 6-Шатлалт Validation Gate

Live агентуудад саад болохгүйн тулд шүүлтийг **Colab T4** эсвэл хязгаарлагдмал Docker worker (`cpus: "1.0"`, `nice -n 19`) дээр хийнэ.
- **2-Stage Early Pruning:** `VectorBT`-ээр эхлээд **сүүлийн 3 сарын өгөгдөл** дээр 5,000–50,000 хослолыг 10–30 секундэд гүйлгэж муу 90%-ийг шууд хаяна; үлдсэн **шилдэг 100–500 стратегийг** л доорх 6 шатаар гүн шалгаснаар тооцооллын нөөцийг **90% хэмнэнэ**.

| Шат | Нэр ба Өгөгдлийн хуваалт | Заавал тэнцэх тоон босго | Татгалзах (Reject) нөхцөл |
| :--- | :--- | :--- | :--- |
| **1** | **In-Sample Backtest** (`60%`) | `Profit Factor >= 1.35`, `Trades >= 200`, бүх зардал (commission + spread + slippage + financing) хассан цэвэр өгөөж эерэг | Зөвхөн 0-зардалтай үед ашигтай |
| **2** | **Out-of-Sample (OOS)** (`20%`) | OOS Sharpe >= In-Sample Sharpe-ийн **60%**, цэвэр өгөөж эерэг | OOS дээр огцом унах |
| **3** | **Walk-Forward** (`5` цонх `T1/V1..T5/V5`) | 5 цонхны **>= 4 нь** ашигтай | Ганц үед л ажиллах |
| **4** | **Parameter Sensitivity Heatmap** | `±10–20%` өөрчлөхөд **өргөн тогтвортой бүс (plateau)** үүсгэх | Ганц хурц цэг (sharp peak) |
| **5** | **Bias, Distribution & Monte Carlo** (`20%`) | `1,000` симуляцид `95%` нь Max DD хязгаараас бага; Leakage = `0`; High Win% биш **Sharpe > Buy & Hold** | Skewness сөрөг, Tail Risk (VaR/CVaR) өндөр |
| **6** | **Paper Trading** (`4–12` долоо хоног) | Live slippage/fill нь backtest-тэй **±15%** дотор таарах | Гүйцэтгэл backtest-ээс зөрөх |

**5 Сонгодог Seed Template:** *Warren Buffett* (`ROE > 15%`, `D/E < 0.5`, `FCF > 0`, `Margin of Safety 20–40%`), *Peter Lynch* (`PEG < 1`, `CAGR > 15%`), *Ray Dalio* (All-Weather + 4 макро квадрант), *George Soros* (`SMA 50` + `Volume` + `Sentiment` + divergence exit), *Jim Simons* (`MA(20) ±2%` mean reversion + cross-asset ranking).

---

## 7. Шууд Арилжааны AI болон 3 Хурдтай Өөрөө Суралцах Давталт

**Шууд арилжааны замд (Live Execution) зөвхөн хөнгөн 4 ML загвар ажиллана (LLM оролцохгүй):**

1. **Хурд 1 — Микросекунд / Арилжаа бүр (Live Inference & Online Learning):**
   - **LightGBM (`ONNX Runtime` — Rust дотор, `< 1 ms`):** Индикатор, orderbook болон TimesFM-ийн дохиог шүүнэ.
   - **Vowpal Wabbit (`Contextual Bandit`, `~1 ms`):** Тухайн мөчийн контекстэд баталгаажсан стратегиудаас сонгож, хаагдсан арилжааны `net_pnl / risk` утгаар шууд суралцана.
   - **TimesFM 2.5 (`~86–450 ms`, `1m+` лаан дээр):** Лаа хаагдах бүрт дараагийн чиглэл/хэлбэлзлийн feature гаргана.
   - **River (`ADWIN Drift Detector`, `< 1 ms`):** Live үр дүн муудаж дэглэм солигдмогц `DRIFT_ALERT` өгнө.
2. **Хурд 2 — 50 арилжаа тутам / 00:00 UTC (Incremental Micro-Training):**
   - Python дээр [LightGBM](https://lightgbm.readthedocs.io/en/latest/pythonapi/lightgbm.Booster.html)-ийг `init_model` эсвэл `refit`-ээр 1–2 секундэд шинэчлэн `.onnx` болгож **Shadow Gate** (дараагийн 20–30 дохион дээр зэрэгцэж шалгах)-аар тэнцвэл Rust руу **hot-swap** хийнэ.
3. **Хурд 3 — Долоо хоног / Сар тутам (Deep Research & LoRA):**
   - Мини-Hermes нь `trade_memory`-ээс алдаа шүүж OpenRouter-аар шинэ код үүсгэн 6-шатлалт Validation Gate рүү оруулна. Сард 1 удаа Colab T4 дээр [Unsloth](https://github.com/unslothai/unsloth) (LoRA/QLoRA)-аар адаптер шинэчилж болно.

---

## 8. Independent Risk Engine (8 Хатуу Дүрэм ба Буцаалт)

Rust цөм дотор AI-ийн халдаж чадахгүй 8 дүрэм хатуу ажиллана:
- `max_position_size_risk`: **1.0%** | `max_exposure_per_instrument`: **20%** | `max_sector_exposure`: **40%** | `max_correlation_limit`: **0.70**
- `daily_loss_limit`: **3.0%** | `max_drawdown_limit`: **10.0%** | `max_order_price_deviation`: **1.0%**
- `require_fresh_data`: **true** (`> 5s` бол блоклоно) + **Duplicate Order Cooldown** (`30s`) + **Kill Switch**.

**Автомат Буцаалт (Rollback) & Telegram 8 триггер:** Өдрийн 3% алдагдал, `>2×` drawdown хурд, `>2×` slippage, `Order Rejected`, `Data Disconnected`, `River ADWIN Drift`, эрсдэлийн зөрчил, `/kill` команд — аль нь ч тохиолдвол одоогийн хувилбарыг унтрааж өмнөх тогтвортой хувилбар руу буцаана.

---

## 9. Нээлттэй Эхийн Сангууд ба 3 Шатлалт Асаалт

- **Backtest & Судалгаа:** `polakowo/vectorbt`, `mementum/backtrader`, `microsoft/qlib`, `QuantConnect/Lean`, `OpenBB`.
- **Агент, Токен, Браузер, MCP & Skills:** `NousResearch/hermes-agent`, `zurawiki/tiktoken-rs`, `huggingface/tokenizers`, `obscura`, `Tauri v2`, `zt2201/finmem`, `tradesdontlie/tradingview-mcp`, `Jakeschincariol/day-trader-skill`, `tradermonty/claude-trading-skills`, `agiprolabs/claude-trading-skills`.

**Хэрэгжүүлэх 3 үе шат:**
- **Phase 1 (1–2 долоо хоног):** `Docker Compose` + `Go Data Hub` + `DuckDB (<24h) / Google Drive Parquet (>=24h)` + `Rust Risk Engine` + `Telegram` + Paper Trading.
- **Phase 2 (3–6 долоо хоног):** `VS Code Colab` дээр `VectorBT` 2-stage pruning + 6-шатлалт Gate + `LightGBM init_model → ONNX` + `Vowpal Wabbit` + `River` + `Shadow Gate` + `Rollback`.
- **Phase 3 (7+ долоо хоног):** Rust Мини-Hermes (`tiktoken-rs` + context compaction) + `OpenRouter Pre-Train Codegen (Dry-Run)` + `strategy_studio` (auto + manual) + `TimesFM 2.5` + `obscura` / `Tauri v2`.
