# 作業系統 Chapter 1：Introduction（導論）筆記

> **教材**：*Operating System Concepts*, 10th ed.（Silberschatz, Galvin, Gagne）
> **投影片**：Mei-Ling Chiang 教授，暨南大學資管系（2026/09/15，共 143 頁）
> **筆記說明**：
> - 英文專有名詞第一次出現時以「**English（中文）**」標示，文末有完整詞彙表。
> - 標示「📌 投影片例子」者出自投影片；標示「💡 補充例子」者是為了幫助理解另外加的生活比喻或說明，不在投影片內。

---

## 目錄

- [0. 本章要回答的問題](#0-本章要回答的問題)
- [1.1 作業系統在做什麼](#11-作業系統在做什麼-what-operating-systems-do)
- [1.2 電腦系統組織](#12-電腦系統組織-computer-system-organization)
- [1.3 電腦系統架構](#13-電腦系統架構-computer-system-architecture)
- [1.4 作業系統的運作](#14-作業系統的運作-operating-system-operations)
- [1.5 資源管理](#15-資源管理-resource-management)
- [1.6 安全與保護](#16-安全與保護-security-and-protection)
- [1.7 虛擬化](#17-虛擬化-virtualization)
- [1.8 分散式系統](#18-分散式系統-distributed-systems)
- [1.9 核心資料結構](#19-核心資料結構-kernel-data-structures)
- [1.10 運算環境](#110-運算環境-computing-environments)
- [1.11 自由與開放原始碼作業系統](#111-自由與開放原始碼作業系統)
- [重點整理：OS 什麼時候會執行？](#重點整理os-什麼時候會執行)
- [容易混淆的觀念對照](#容易混淆的觀念對照)
- [作業（Homework）](#作業homework)
- [英中詞彙總表](#英中詞彙總表)

---

## 0. 本章要回答的問題

投影片一開始列出整門課的大問題，Chapter 1 先給概觀：

| 問題 | 對應章節 |
|---|---|
| Why OS is needed?（為什麼需要 OS） | 1.1 |
| What is an OS?（OS 是什麼） | 1.1 |
| What does an OS do?（OS 做什麼） | 1.1、1.5 |
| When does an OS run?（OS 何時執行） | 1.2、1.4 |
| What are the types of OS?（OS 的種類） | 1.3、1.4、1.10 |
| How does OS work?（OS 如何運作） | 1.2、1.4 |
| What are OS components / structures?（元件與結構） | 1.5（結構在 Ch2） |

---

## 1.1 作業系統在做什麼 (What Operating Systems Do)

### 為什麼需要 OS

桌機、筆電、平板、手機的硬體各不相同（CPU、記憶體、硬碟、各種連接埠…）。如果沒有 OS，每個應用程式都得自己去控制這些硬體。OS 把硬體包起來，提供一致、方便的使用環境。

### 電腦系統的四個組成（由上到下）

```
        user（使用者）
          ↕
  application programs（應用程式：編譯器、瀏覽器、開發工具…）
          ↕
  operating system（作業系統）
          ↕
  computer hardware（硬體：CPU、memory、I/O devices…）
```

### Shell（殼層）＝ Command Interpreter（命令直譯器）

- 工作：**取得並執行使用者下的下一個命令**（get and execute the next user-specified command）。
- 兩種形式：
  - **GUI**（Graphical User Interface，圖形使用者介面）
  - **CLI**（Command Line Interface，命令列介面）
- 層次關係：Applications → Shell → Operating System → Hardware（一層包一層）。

> 📌 投影片例子：macOS 的 Finder 視窗、iPad 的主畫面是 GUI；Windows 的「命令提示字元」黑底視窗是 CLI。
> 💡 補充例子：在 Linux Terminal 輸入 `ls` 列出檔案，bash 就是一種 shell。

### OS 的定義

**Operating System（作業系統）**：
- 管理電腦硬體的**軟體**。
- 作為使用者與硬體之間的**intermediary（中介者）**。
- **目的**：提供一個環境，讓使用者能以 **convenient（方便）** 且 **efficient（有效率）** 的方式執行程式。

OS 的兩種角色：

| 角色 | 中文 | 說明 |
|---|---|---|
| **Resource allocator** | 資源分配者 | 管理所有資源；當多個請求互相衝突時，決定如何分配才**有效率又公平** |
| **Control program** | 控制程式 | 管理使用者程式的執行，防止錯誤與不當使用電腦 |

> 💡 補充例子：OS 像餐廳經理。桌位、廚師、食材是資源（resource allocator 負責分配）；客人不能自己闖進廚房動刀動火（control program 負責管制）。

**沒有全世界公認的定義**，常見兩種說法：

1. 簡單觀點：廠商賣 OS 時附帶的所有東西都算（但各家差異很大）。
2. 較常見的定義：**OS 是電腦上隨時都在執行的那一個程式，通常稱為 kernel（核心）**。

除了 kernel 之外還有兩類程式：

| 類別 | 中文 | 說明 |
|---|---|---|
| **System programs** | 系統程式 | 和 OS 有關，但不一定是 kernel 的一部分 |
| **Application programs** | 應用程式 | 和系統運作無關的所有程式 |

**現代 OS（通用與行動裝置）包含三部分**：
1. 一直在執行的 **kernel**
2. **Middleware frameworks（中介軟體框架）**：讓應用程式開發更容易，並提供資料庫、多媒體、繪圖等功能
3. 系統執行期間協助管理的 **system programs**

> 📌 投影片例子：**Android 架構**由下到上為 hardware → Linux kernel → Bionic（C 函式庫）/ HAL（硬體抽象層）→ native libraries（SQLite、openGL、webkit、SSL、media framework、surface manager）→ ART VM、Android frameworks、JNI → applications。其中 SQLite、openGL 這些就是 middleware。

### OS 提供的服務（A View of OS Services）

- 最上層：user and other system programs
- **User interfaces**：GUI、touch screen（觸控螢幕）、command line
- **System calls（系統呼叫）**：程式進入 OS 的入口
- **Kernel 內的服務**：

| 服務 | 中文 |
|---|---|
| program execution | 程式執行 |
| I/O operations | 輸入輸出操作 |
| file systems | 檔案系統 |
| communication | 通訊 |
| resource allocation | 資源分配 |
| accounting | 使用紀錄／記帳 |
| error detection | 錯誤偵測 |
| protection and security | 保護與安全 |

### 一句話總結：OS 做四件事

**Management（管理）、Allocation（分配）、Control（控制）、Service（服務）**

---

## 1.2 電腦系統組織 (Computer-System Organization)

### 基本架構

現代電腦由**一或多個 CPU** 與多個 **device controller（裝置控制器）** 組成，透過共用的 **bus（匯流排）** 連接，並共同存取 **shared memory（共享記憶體）**。

```
 disks      mouse keyboard printer    monitor
   |           \     |     /             |
[CPU]  [disk controller]  [USB controller]  [graphics adapter]
  |          |                  |                  |
  +----------+--------bus-------+------------------+
                      |
                   [memory]
```

### 開機流程（Computer Startup）— OS 執行時機 (1)

電腦一開機需要一個起始程式：**bootstrap program（啟動程式／靴帶程式）**。

- 通常存在 **ROM**、**EEPROM** 或 flash 記憶體中（統稱 **firmware，韌體**）。
- 工作：
  1. 初始化系統各部分（CPU 暫存器、裝置控制器、記憶體內容…）
  2. 把 **OS kernel** 從磁碟**載入**到 RAM
  3. 開始執行 OS
- OS 啟動後執行第一個 process（例如 `init`），然後**等待事件發生**。

投影片圖示的三步驟：
1. CPU 先執行 ROM 裡的 bootstrap program
2. bootstrap 到磁碟找到 **kernel image（核心映像檔）**
3. 把 kernel image 載入 RAM，之後 GUI / Shell 也被載入記憶體

> 💡 補充例子：按下電源鍵後看到主機板 logo 的那段時間，就是 firmware 裡的 bootstrap 在跑；看到 Windows / Ubuntu 登入畫面時，kernel 已載入完成，GUI shell 也啟動了。

### 中斷 (Interrupt) — OS 執行時機 (2)

**Interrupt（中斷）**：通知 CPU「有事件需要處理」的機制。**OS 是 interrupt-driven（中斷驅動）的**。

兩種觸發來源：

| 來源 | 方式 | 例子 |
|---|---|---|
| **H/W（硬體）** | 送出硬體訊號給 CPU | 鍵盤按鍵、磁碟讀完資料、計時器到時 |
| **S/W（軟體）** | 執行 **system call**（也叫 monitor call） | 程式要求讀檔 |

**中斷處理流程**：

1. CPU 收到中斷，暫停目前工作。
2. 中斷架構必須**保存被中斷指令的位址**，之後才能回來繼續執行。
3. 透過 **interrupt vector（中斷向量）** 找到對應的處理程式。
   - interrupt vector 是一張表，存放所有服務程式的**位址**。
4. 把控制權轉給 **ISR（Interrupt Service Routine，中斷服務常式）**。
5. ISR 處理完，回到原本被中斷的程式。

Windows 與 UNIX 都採用這種做法。記憶體中相關的 OS 元件：interrupt vector、ISRs、device drivers。

**Instruction cycle with interrupts（含中斷的指令週期）**：CPU 每執行完一道指令，就檢查有沒有中斷；有的話就去處理。

> 💡 補充例子：你在寫作業（執行使用者程式），手機響了（interrupt）。你在書上夾書籤（保存位址），查通訊錄看是誰打來、該怎麼應對（查 interrupt vector），接電話（執行 ISR），講完翻回書籤那頁繼續寫（return）。

#### Interrupt Timeline（單一程式做輸出的時間軸）

- CPU 平時在執行 user program；I/O device 平時 idle（閒置）。
- 發出 **I/O request** → 裝置開始 transferring（傳輸）→ **transfer done** → **interrupt signaled**（發出中斷）→ CPU 切去做 I/O interrupt processing → **interrupt handled** → CPU 回到 user program。

#### Interrupt-driven I/O Cycle（中斷驅動 I/O 循環）7 步驟

| 步驟 | 誰 | 動作 |
|---|---|---|
| 1 | CPU | device driver 啟動 I/O |
| 2 | I/O controller | 開始 I/O |
| 3 | I/O controller | 輸入備妥／輸出完成／發生錯誤 → 產生中斷訊號 |
| 4 | CPU | 收到中斷，把控制權交給 interrupt handler |
| 5 | CPU | interrupt handler 處理資料，從中斷返回 |
| 6 | CPU | 回復執行被中斷的工作 |
| 7 | — | 回到步驟 1（CPU 在指令之間持續檢查中斷） |

#### Intel 處理器事件向量表（Event-vector Table）重點

| 向量編號 | 說明 | 中文 |
|---|---|---|
| 0 | divide error | 除法錯誤（除以零） |
| 3 | breakpoint | 中斷點（除錯用） |
| 6 | invalid opcode | 無效的運算碼 |
| 11 | segment not present | 區段不存在 |
| 12 | stack fault | 堆疊錯誤 |
| 13 | general protection | 一般保護錯誤 |
| 14 | page fault | 分頁錯誤 |
| 0–31 | — | **non-maskable（不可遮罩）**，多為錯誤／例外 |
| 32–255 | maskable interrupts | **可遮罩中斷**，給裝置使用 |

### I/O 結構 (I/O Structure)

**Device controller（裝置控制器）**：
- 內部有 **local buffer（本地緩衝區）** 與一組**特殊用途暫存器（special-purpose registers）**。
- 負責在它控制的**周邊裝置**與**自己的 local buffer** 之間搬資料。

**Device driver（裝置驅動程式）**：OS 為每個 device controller 準備一個，是 OS 與控制器溝通的軟體。

**一次 I/O 的流程**：
1. device driver 把適當的值載入控制器的 registers。
2. 控制器檢查 registers，執行要求的 I/O（如果是 read，就把資料從裝置搬到 local buffer）。
3. 傳輸完成後，控制器發出 **interrupt** 通知 CPU。

#### 兩種 I/O 方式比較 ⭐

| | **Interrupt-driven I/O（中斷驅動 I/O）** | **DMA I/O（Direct Memory Access，直接記憶體存取）** |
|---|---|---|
| CPU 是否介入搬資料 | **要**。CPU 需主動把資料在記憶體與 I/O 模組間搬移 | **不用**。DMA 模組代替 CPU 完成 |
| 硬體需求 | 無額外硬體 | 系統匯流排上多一個 DMA 模組 |
| 中斷次數 | 低速裝置**每個 byte 一次中斷** | **每個 block 只產生一次中斷** |
| CPU 負擔 | CPU 被 I/O 綁住，每次傳輸都要執行許多指令 | 傳輸期間 CPU 可以去做別的事 |
| 資料路徑 | 裝置 → 控制器 buffer →（CPU 經 ISR）→ 記憶體 | 裝置 → 控制器 buffer → **直接**到記憶體 |
| 適合 | 低速裝置（鍵盤、滑鼠） | 高速、大量資料（磁碟、網路卡） |

> 💡 補充例子：
> - Interrupt-driven 像**老闆親自搬貨**：每到一箱，倉管就叫老闆（中斷）來搬進倉庫，老闆沒辦法做別的事。
> - DMA 像**請了搬運工**：老闆只說「把這一車搬到 A 區」，搬運工整車搬完後才回報一次，老闆這段時間可以處理其他事。

#### 現代電腦如何運作（整合圖）

| 編號 | 動作 |
|---|---|
| 1 | CPU 與 memory 之間：instruction execution cycle（指令執行週期） |
| 2 | CPU → device：I/O request |
| 3 | CPU ↔ device：data |
| 4 | device ↔ memory：**DMA**（不經 CPU） |
| 5 | device → CPU：**interrupt** |
| 6 | CPU ↔ memory：data movement（資料搬移） |

### 儲存結構 (Storage Structure)

各種儲存裝置的主要差異：**speed（速度）、cost（成本）、size（容量）、volatility（揮發性）**。

- **Volatile（揮發性）**：斷電後資料消失。
- **Nonvolatile（非揮發性）**：斷電後資料仍在。

#### Storage-Device Hierarchy（儲存裝置階層，Fig. 1.6）

```
   容量小 / 速度快 / 貴
  ┌─────────────────────┐
  │ registers（暫存器）   │ ┐
  │ cache（快取）         │ ├ primary storage（主儲存）  → volatile
  │ main memory（主記憶體）│ ┘
  ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤ ← 揮發 / 非揮發分界
  │ nonvolatile memory    │ ┐ secondary storage（輔助儲存）
  │ hard-disk drives      │ ┘
  │ optical disk（光碟）   │ ┐ tertiary storage（第三級儲存） → nonvolatile
  │ magnetic tapes（磁帶） │ ┘
  └─────────────────────┘
   容量大 / 速度慢 / 便宜
```

**Caching（快取）**：把資訊複製到較快的儲存系統。**主記憶體可視為輔助儲存的 cache**。（詳見 1.5.5）

---

## 1.3 電腦系統架構 (Computer-System Architecture)

依**通用處理器的數量**粗略分三類，不同架構會影響 OS 的設計：

1. **Single-Processor Systems（單處理器系統）**
2. **Multiprocessor Systems（多處理器系統）**
3. **Clustered Systems（叢集系統）**

### Multiprocessor Systems（多處理器系統）

- 有**兩個以上的處理器**，共享 bus，有時也共享 clock（時脈）、memory、周邊裝置。
- 主要優點：**increased throughput（提高產能／吞吐量）**——單位時間內完成更多工作。

| | **SMP：Symmetric Multiprocessing（對稱式多處理）** | **Asymmetric Multiprocessing（非對稱式多處理）** |
|---|---|---|
| 分工 | 每個 CPU 地位**對等（peer）**，都能執行所有工作，包括 OS 功能與使用者 process | 每個處理器被指派**特定工作** |
| 特點 | 多個 process 可同時執行而效能不會明顯下降 | 一個 boss 處理器負責排程、分派工作給 worker 處理器 |
| 使用情況 | **大多數系統採用**；Windows、macOS、Linux、Android、iOS 都支援 | 較少見 |

> 💡 補充例子：SMP 像一間每位櫃員都能辦所有業務的銀行；Asymmetric 像有一位主管專門分派工作，其他人只負責被指派的項目。

#### Multiple chips vs. Multicore

- **Multicore（多核心）**：把多個運算 **core（核心）** 放在**同一顆晶片**上。
  - 比多顆單核晶片**更省電、更有效率**，因為晶片內通訊（on-chip communication）比較快。
  - 對 OS 而言，N 核心的 CPU 看起來就像 **N 個標準處理器**。
- 📌 投影片例子：dual-core（雙核心）設計——兩個核心放在同一顆晶片上。

#### 記憶體存取模型（Memory Access Model）

| 模型 | 中文 | 說明 |
|---|---|---|
| **UMA**（Uniform Memory Access） | 一致記憶體存取 | 任何 CPU 存取任何 RAM 的時間**都相同**；SMP 架構採用 |
| **NUMA**（Non-Uniform Memory Access） | 非一致記憶體存取 | CPU 存取某些部分的記憶體**比較快**（通常是自己本地的記憶體），存取其他部分較慢 |

### Clustered Systems（叢集系統）

- 由**兩個以上獨立的系統（node，節點）** 連結而成。
- 普遍接受的定義：叢集中的電腦**共享儲存裝置**，並透過 **LAN** 或更快的互連（如 InfiniBand）緊密連結。
- 📌 投影片例子：web cluster、game server cluster、computing cluster；**LVS Cluster 架構**——Clients 經 Internet/Intranet 連到 **Load Balancer（負載平衡器）**，再由它分派給後端多台 Real Server，對外看起來像一台 Virtual Server。

**用途一：High availability（高可用性）**
- 即使叢集中一台或多台系統故障，服務仍然持續。
- 提高 **reliability（可靠度）**。
- **Graceful degradation（優雅降級）**：能依照「存活硬體的比例」持續提供服務的能力。
- **Fault-tolerant system（容錯系統）**：任何單一元件故障仍能繼續運作。
  - 📌 投影片例子：HP NonStop（前身 Tandem）同時使用硬體與軟體備援。

> 💡 補充例子：10 台伺服器壞了 2 台，系統還能提供約 80% 的服務量而不是整個掛掉，這就是 graceful degradation。

**叢集結構**

| | **Asymmetric clustering（非對稱叢集）** | **Symmetric clustering（對稱叢集）** |
|---|---|---|
| 做法 | 一台機器處於 **hot-standby（熱待機）** 模式，只負責監控；另一台執行應用程式 | 兩台以上主機都在執行應用程式，並**互相監控** |
| 故障時 | 待機那台接手 | 其他主機分擔 |
| 效率 | 待機機器平時沒在做事 | 硬體利用率較高 |

**用途二：High-performance computing（HPC，高效能運算）**
- 應用程式必須寫成能 **parallelization（平行化）**。
- **DLM（Distributed Lock Manager，分散式鎖管理器）**：提供存取控制與鎖定，避免對共享資料的衝突操作。

**SAN（Storage-Area Network，儲存區域網路）**：讓許多系統連接到同一個**儲存池**（Fig. 1.8）。

---

## 1.4 作業系統的運作 (Operating-System Operations)

### 1.4.1 Multiprogramming 與 Multitasking

#### 三種系統的演進 ⭐

| | **Single tasking（單工）** | **Multiprogramming（多元程式）** | **Multitasking / Time-sharing（多工／分時）** |
|---|---|---|---|
| 記憶體中的程式 | 一次一個 | **同時放多個 process** | 同時放多個 process |
| 切換時機 | 不切換 | 目前的 process **需要等待**（如等 I/O）時才切換 | **時間片用完就切換**，非常頻繁 |
| 目標 | — | 提高 **CPU utilization（CPU 使用率）** | 提供快速的 **response time（回應時間）** |
| 問題 | 單一程式無法讓 CPU 或 I/O 一直忙碌，效率差 | 有效率，但**不保證公平**（某程式可能一直佔著 CPU） | 需要更多 OS 功能支援 |

**Multiprogramming**：把程式組織起來，讓 **CPU 永遠有東西可以執行**。
- **Process（行程）**：執行中的程式（a program in execution）。
- 帶來兩個 OS 設計問題：
  - **Job scheduling（工作排程）**：決定哪些工作載入記憶體。
  - **CPU scheduling（CPU 排程）**：決定記憶體中哪個 process 下一個使用 CPU。

**Multitasking** 是 multiprogramming 的**邏輯延伸**：CPU 在多個 process 之間切換，切換得非常頻繁，讓使用者感覺反應很快。

**Time-sharing（分時）**：
- 把時間切成一段段 **time slice（時間片）**，每個工作一次只能執行一個時間片。
- 時間片結束，目前的工作被擱置，換下一個工作。
- 這樣快速地**輪替（shuffling）**，造成多個工作**同時執行的錯覺（illusion）**。

> 💡 補充例子：
> - Multiprogramming 像廚師煮湯要等 20 分鐘時，先去切菜，不乾等（有等待才換）。
> - Multitasking 像廚師每 10 秒就換一道菜顧一下，每桌客人都覺得自己的菜一直有人在處理。
> - 你邊聽 Spotify、邊用 Chrome、邊開 VS Code，單核心 CPU 其實是輪流做，只是切得太快你感覺不出來。

#### Multiprogramming 需要的 OS 功能

- **Memory management（記憶體管理）**
- **CPU scheduling（CPU 排程）**：從記憶體中已就緒的工作挑下一個執行
- **Concurrency control（並行控制）**：多個工作並行時，必須限制它們互相影響的能力（包括排程、磁碟儲存、記憶體管理各階段）
  - 📌 投影片例子：Appl. #1 與 Appl. #2 同時存取同一個檔案。

#### Multitasking 額外需要的 OS 功能

- 確保合理的 **response time**
- **Virtual memory（虛擬記憶體）**：允許執行**沒有完全載入記憶體**的 process
- **File system and storage management（檔案系統與儲存管理）**
- **Protection（保護）**
- **Process synchronization and communication（行程同步與通訊）**：確保有秩序地執行
- **Deadlock avoidance（死結避免）**

### 中斷、Trap 與 Exception — OS 執行時機 (3)

現代 OS 是 interrupt-driven：沒有 process 要執行、沒有 I/O 要服務、沒有使用者要回應時，OS 就**安靜地等待**事情發生。事件幾乎都是由 **interrupt** 或 **trap** 來通知。

**Trap（陷阱）＝ Exception（例外）**：**軟體產生的中斷**，原因有兩種：
1. **錯誤**：如 divide by zero（除以零）、invalid memory access（無效的記憶體存取）
2. **使用者程式的特定請求**：要求 OS 提供服務，也就是 **system call**

**Exception 處理流程**（📌 投影片：`A = 5/0`）：
1. CPU 執行記憶體中的應用程式
2. 發生錯誤（除以零）
3. CPU 查 interrupt vector
4. 跳到 OS 的 **exception handler（例外處理常式）** 處理（通常是終止該程式）

#### Program Misbehavior（程式行為不當）— 四個例子 ⭐

投影片問：**哪一個會造成 exception？**

**程式 1：除以零**
```c
#include <stdio.h>
int main() {
    int a;
    a = 5/0;
    printf("a=%d", a);
    return 0;
}
```
→ **會**。觸發 divide error（Intel 向量 0）。投影片實測：編譯時有 `division by zero` 警告，執行後 Windows 顯示「prog1.exe 已經停止運作」。

**程式 2：寫入空指標附近的位址**
```c
#include <stdio.h>
int main() {
    char *b;
    b = (char *)0;
    b[10] = 'A';
    printf("b[10]=%c", b[10]);
    return 0;
}
```
→ **會**。`b` 指向位址 0，寫入 `b[10]` 是 invalid memory access，被 OS 攔下並終止程式。

**程式 3：寫入超出配置範圍的位址**
```c
#include <stdio.h>
#include <malloc.h>
int main() {
    char *b;
    b = (char *)malloc(10);
    b[10000] = 'A';
    printf("b[10000]=%c", b[10000]);
    return 0;
}
```
→ 投影片實測**也被終止**。只配置 10 bytes 卻寫到第 10000 個位置，超出範圍。
（💡 補充：這類越界存取是否當掉，要看該位址是否剛好落在這個 process 被允許存取的範圍內；沒當掉不代表程式正確。）

**程式 4：無窮迴圈**
```c
#include <stdio.h>
int main() {
    char *b;
    while (1) {};
    printf("GG");
    return 0;
}
```
→ **不會產生 exception**。程式只是卡住，畫面全黑、永遠印不出 `GG`。這不是錯誤指令，CPU 無法靠 exception 偵測，必須靠 **timer（計時器）** 解決（見 1.4.3）。

**結論**：設計良好的 OS 必須確保一個**不正確或 malicious（惡意）** 的程式，不會讓其他程式或 OS 本身執行錯誤。process 可能出的問題有：
- infinite loop（無窮迴圈）→ 用 **Timer** 解決
- 修改其他 process 的記憶體、修改 OS → 用 **Dual-mode** 與記憶體保護解決

### 1.4.2 Dual-Mode 與 Multimode Operation（雙模式與多模式運作）

**目的**：必須區分「正在執行 OS 的程式碼」還是「使用者的程式碼」。大多數電腦由**硬體**提供至少兩種模式，用一個 **mode bit（模式位元）** 表示目前模式。

| 模式 | mode bit | 別名 | 代表誰執行 |
|---|---|---|---|
| **User mode（使用者模式）** | **1** | — | 代表使用者 |
| **Kernel mode（核心模式）** | **0** | supervisor mode（監督模式）、system mode（系統模式）、privileged mode（特權模式） | 代表 OS |

**模式切換**：

```
              interrupt / trap / system call
   User mode ─────────────────────────────→ Kernel mode
   (bit = 1) ←───────────────────────────── (bit = 0)
                    set user mode
```

- 發生 interrupt（硬體或軟體）或 trap 時，**硬體**切換到 kernel mode 執行 OS 程式碼。
- 處理完成後切回 user mode，把控制權還給使用者程式。
- 作用：保護 OS 不受 **errant（出錯的）** 使用者影響，也保護使用者彼此不互相干擾。

**Privileged instructions（特權指令）**：
- 可能造成危害的指令，**只能在 kernel mode 執行**。
- 例如：切換到 kernel mode 的指令、I/O 控制、timer 管理、interrupt 管理。
- 如果在 user mode 嘗試執行，硬體視為**非法**，並 **trap** 到 OS。

> 💡 補充例子：銀行大廳（user mode）的客人不能自己進金庫；要領錢得到櫃檯申請（system call），由行員（kernel mode）進金庫處理。客人硬闖金庫（執行特權指令）會被保全攔下（trap）。

**Multimode（多模式）**：
- Windows、UNIX、Linux 等現代 OS 都利用 dual-mode 來加強保護。
- 越來越多 CPU 支援超過兩種模式：
  - 支援虛擬化的 CPU 有 **VMM（Virtual Machine Manager）mode** 給 guest VM 使用。
  - 不同 kernel 元件使用不同模式。
  - 📌 投影片例子：**Intel 64 系列 CPU 支援 4 個 privilege level（特權等級），又稱 protection rings（保護環）**，並支援虛擬化。

### System Call（系統呼叫）— OS 執行時機 (4)

**System call**：讓使用者程式**請 OS 代為執行 OS 工作**的方法。

- 通常以 **trap** 的形式跳到 interrupt vector 中的特定位置。
- 可用通用的 trap 指令，有些系統有專用的 `syscall` 指令。
- 執行 system call 時，硬體把它當成 **software interrupt（軟體中斷）** 處理。

**Transition from User to Kernel Mode（Fig. 1.13）**：

```
user mode (bit=1)   user process 執行 → 呼叫 system call        從 system call 返回
───────────────────────────│────────────────────────↑──────────
kernel mode (bit=0)       trap (bit=0) → 執行 system call → return (bit=1)
```

**System call 的執行步驟**（How a system call can be made）：

| 步驟 | 動作 |
|---|---|
| ① | User program 發出 system call，trap 進入 kernel |
| ② | OS 查 **dispatch table（分派表）** |
| ③ | 找到並執行對應的 **service routine（服務常式）** |
| ④ | 執行完畢，返回 user program |

> 📌 投影片例子：一個簡單的 shell 程式
> ```c
> int main() {
>     int pid = 1, i;
>     char c[100];
>
>     while (scanf("%s", c)) {
>         if (strcmp("exit", c) == 0) {
>             exit(0);
>         } else {
>             pid = fork();              // 呼叫 C Library 中的 fork
>             if (pid == 0) {            // 子行程
>                 printf("%s\n", c);
>                 i = execlp(c, c, NULL);
>                 if (i == -1)
>                     printf("Command Error!!\n");
>                 printf("Child terminal.\n");
>                 exit(0);
>             } else {                   // 父行程
>                 printf("Parent had fork a child with PID %d.\n", pid);
>             }
>         }
>     }
> }
> ```
> **說明**：程式不斷讀入使用者輸入的命令。輸入 `exit` 就結束；否則用 `fork()` 建立子行程，子行程用 `execlp()` 把自己換成要執行的命令。`fork()`、`execlp()`、`exit()` 都會透過 C 函式庫發出 system call，進入 kernel mode 由 OS 完成。這正是 shell（command interpreter）的基本原理。

### 1.4.3 Timer（計時器）

**目的**：防止使用者程式霸佔 CPU（例如陷入無窮迴圈），確保 **OS 能維持對 CPU 的控制**。

- 設定一個 timer，在指定時間後**中斷**電腦。
- 在排程 process 之前先設好，以便之後取回控制權，或終止超過配額時間的程式。
- 時間長度可以是固定的或可變的。
- 做法：用允許的執行時間初始化一個 **counter（計數器）**，倒數到 0 就產生中斷。
- 設定 timer 的指令是 **privileged instruction**。

**Timer interrupt 處理流程**：Appl. #1 在 user mode 執行 → timer 發出中斷 → 切到 kernel mode，經 interrupt vector 找到 **timer handler** → OS 決定下一個執行誰 → 切回 user mode 執行 Appl. #2。

> 💡 補充例子：辯論賽的計時鈴。不管講者講到哪，時間到主持人就打斷換人。前面「程式 4 無窮迴圈」就是靠這個機制，OS 才有機會把 CPU 拿回來分給別的程式。

---

## 1.5 資源管理 (Resource Management)

OS 的基本元件：

| 節 | 主題 | 中文 | 對應章節 |
|---|---|---|---|
| 1.5.1 | Process Management | 行程管理 | ch3–8 |
| 1.5.2 | Memory Management | 記憶體管理 | ch9–10 |
| 1.5.3 | File-System Management | 檔案系統管理 | ch13–15 |
| 1.5.4 | Mass-Storage Management | 大量儲存管理 | ch11 |
| 1.5.5 | Cache Management | 快取管理 | — |
| 1.5.6 | I/O System Management | I/O 系統管理 | ch12 |

### 1.5.1 Process Management（行程管理）

**Program vs. Process** ⭐

| | **Program（程式）** | **Process（行程）** |
|---|---|---|
| 性質 | **Passive entity（被動實體）** | **Active entity（主動實體）** |
| 存在位置 | 磁碟上的檔案內容 | 載入記憶體、正在執行 |
| 定義 | 指令的集合 | **A process is a program in execution**（執行中的程式） |

> 💡 補充例子：食譜是 program，照著食譜「正在做菜」這件事是 process。同一份食譜可以同時有兩個人各做一份（同一個程式開兩個視窗 = 兩個 process）。

- Process 需要資源來完成工作：CPU time、memory、I/O、files、初始化資料。
- Process 結束時，OS 必須**回收（reclamation）** 可重複使用的資源。

**Thread（執行緒）**：

| | **Single-threaded process（單執行緒行程）** | **Multi-threaded process（多執行緒行程）** |
|---|---|---|
| Program counter（程式計數器） | **一個**，指向下一道要執行的指令 | **多個**，每個 thread 一個 |

- 系統通常有很多 process（有些是 user process，有些是 OS process），在一或多個 CPU 上**並行（concurrently）** 執行。
- 並行的實現：讓 CPU 在各 process / thread 之間 **multiplexing（多工切換）**。

**Process management 的工作**：
- 在 CPU 上排程 process 與 thread
- 建立與刪除 user / system process
- 暫停（suspend）與恢復（resume）process
- 提供 **process synchronization（行程同步）** 機制
- 提供 **process communication（行程通訊）** 機制

### 1.5.2 Memory Management（記憶體管理）

- 為了提高 CPU 使用率與回應速度，要**同時把多個程式放在記憶體**，因此需要記憶體管理。
- 選擇記憶體管理方案時必須考慮**硬體設計**，每種演算法需要各自的硬體支援。
- 常見方案：
  - **Contiguous allocation（連續配置）**
  - **Segmentation（分段）**
  - **Paging（分頁）**

**Main-memory management 的工作**：
- 追蹤記憶體的哪些部分正在被使用、**被誰使用**
- 決定哪些 process（或其部分）與資料要**移入、移出**記憶體
- 依需要**配置（allocate）** 與**釋放（deallocate）** 記憶體空間

兩個經典問題：

| 問題 | 中文 | 說明 |
|---|---|---|
| **Placement problem** | 放置問題 | 新的 process 要放進哪一塊空閒空間？ |
| **Replacement problem** | 置換問題 | 記憶體滿了，要把誰換出去？ |

> 📌 投影片例子：記憶體中有 30K、50K、20K、15K 幾塊空位，P1、P2、P3…Pn 等待載入——該把誰放到哪一塊？

### 1.5.3 File-System Management（檔案系統管理）

- OS 對資訊儲存提供**一致的、邏輯的觀點（uniform, logical view）**。
  - 把儲存裝置的實體特性**抽象化**，定義出邏輯儲存單位：**file（檔案）**。
  - 把檔案 **mapping（對應）** 到實體媒體上，並透過儲存裝置存取。
- 各種媒體特性不同：存取速度、容量、資料傳輸率、存取方式（**sequential 循序** 或 **random 隨機**）。
- 檔案通常組織成 **directories（目錄）**。
- 大多數系統有 **access control（存取控制）**，決定誰能存取什麼。

**重點**：使用者看到的是 **files 與 directories**，而不是底層的 **storage blocks（儲存區塊）**。

**File management 的工作**：
- 建立與刪除檔案、目錄
- 提供操作檔案與目錄的 **primitives（基本操作）**
- 把檔案對應到 mass storage
- 把檔案**備份**到穩定（非揮發性）的儲存媒體

### 1.5.4 Mass-Storage Management（大量儲存管理）

- 現代電腦以 **HDD（硬碟）** 與 **NVM（Non-Volatile Memory，非揮發性記憶體，如 SSD）** 作為主要線上儲存媒體，存放程式與資料。
- 輔助儲存的妥善管理非常重要：**整台電腦的運作速度可能取決於（hinge on）輔助儲存子系統及其演算法**。
- 有些儲存不需要快：**tertiary storage（第三級儲存）**，如光碟、磁帶——仍需由 OS 或應用程式管理。

**Secondary storage management 的工作**：

| 工作 | 中文 |
|---|---|
| Mounting / unmounting | 掛載／卸載 |
| Free-space management | 可用空間管理（方法：**bit-mapping 位元映射**、**linked list 鏈結串列**） |
| Storage allocation | 儲存空間配置 |
| Disk scheduling | 磁碟排程 |
| Partitioning | 分割 |
| Protection | 保護 |

> 💡 補充例子：插入 USB 隨身碟後系統把它「掛上來」變成 E: 槽（mount）；「安全移除硬體」就是 unmount。

### 1.5.5 Caching Management（快取管理）

**Caching** 是電腦中很重要的原則，在**硬體、OS、軟體**各層都會用到。

**原理**：
1. 把使用中的資訊，從較慢的儲存**暫時複製**到較快的儲存。
2. 需要資料時**先檢查 cache**。
3. 有 → 直接從 cache 使用（快）。
4. 沒有 → 把資料複製到 cache 再使用。

> 💡 補充例子：常用的書放桌上（cache），不常用的放書架（main memory），更少用的放倉庫（disk）。要找書先看桌上，沒有才去書架拿，拿來後順手放桌上。

#### 各級儲存的效能（Fig. 1.14）

| Level | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|
| **名稱** | registers | cache | main memory | solid state disk | magnetic disk |
| **典型容量** | < 1 KB | < 16 MB | < 64 GB | < 1 TB | < 10 TB |
| **實作技術** | custom memory (多埠 CMOS) | on-chip / off-chip CMOS SRAM | CMOS SRAM | flash memory | magnetic disk |
| **存取時間 (ns)** | 0.25–0.5 | 0.5–25 | 80–250 | 25,000–50,000 | 5,000,000 |
| **頻寬 (MB/s)** | 20,000–100,000 | 5,000–10,000 | 1,000–5,000 | 500 | 20–150 |
| **由誰管理** | compiler | hardware | **operating system** | **operating system** | **operating system** |
| **後備** | cache | main memory | disk | disk | disk or tape |

- 資料在各層級間的移動可以是 **explicit（明確的）** 或 **implicit（隱含的）**。
- 注意「由誰管理」：**主記憶體以下才由 OS 管理**。

#### H/W Cache vs. S/W Cache

| | **Hardware cache（硬體快取）** | **Software-controlled cache（軟體控制快取）** |
|---|---|---|
| 控制者 | 硬體，**OS 不介入** | **OS** |
| 例子 | instruction cache、data cache；資料從 cache 到 CPU 與 registers 的傳輸 | OS 在主記憶體中維護檔案系統資料的 cache；資料從磁碟到記憶體的傳輸 |
| 設計重點 | — | **Cache management**：cache 大小與 **replacement policy（置換策略）** |

#### Cache Coherency（快取一致性）

問題：同一筆資料有多份副本時，如何保持一致？

| 環境 | 情況 | 處理 |
|---|---|---|
| **Multiprocessor（多處理器）** | 每個 CPU 都有自己的 local cache，可能存有同一個變數 | A 的值在一個 cache 更新後，必須**立刻反映**到所有存有 A 的 cache。通常是**硬體**議題 |
| **Distributed（分散式）** | 同一檔案的多個 **replica（複本）** 存在不同電腦 | 一處更新後，其他複本要**盡快更新** |

多處理器的簡單解法：共享記憶體改變時，通知其他處理器「你的 cache 副本已經 invalid（失效）」，請它們改回共享記憶體讀取。

> 💡 補充例子：兩人各自下載同一份 Google 文件離線編輯，若不同步，雙方手上的內容就不一致。

### 1.5.6 I/O System Management（I/O 系統管理）

**目的**：對使用者**隱藏特定硬體裝置的 peculiarities（特殊性）**。

**I/O subsystem 的組成**：

1. 記憶體管理元件，包含：

| 技術 | 中文 | 說明 |
|---|---|---|
| **Buffering** | 緩衝 | 資料傳輸過程中**暫時存放** |
| **Caching** | 快取 | 把部分資料放在較快的儲存以提升**效能** |
| **Spooling** | 排存／假脫機 | OS 協調並行輸出的方式；spool 是一個 buffer，替**不能接受交錯資料流**的裝置（如印表機）保存輸出 |

2. 通用的 **device-driver interface（裝置驅動程式介面）**
3. 各特定硬體的 **drivers**

> 💡 補充例子：三個人同時按列印，如果直接送到印表機，三份文件會一行一行交錯印在一起。Spooling 讓每份文件先完整存進佇列，印表機再一份一份印。

---

## 1.6 安全與保護 (Security and Protection)

| | **Protection（保護）** | **Security（安全）** |
|---|---|---|
| 定義 | 控制 process 或使用者**存取系統資源**的機制 | **防禦**系統受到外部與內部的**攻擊** |
| 重點 | 資源的 **authorization（授權）**：CPU、memory、disk、files… | denial-of-service（阻斷服務）、worms（蠕蟲）、viruses（病毒）、identity theft（身分盜用）、theft of service（服務竊取） |
| 章節 | Chapter 17 | Chapter 16 |

Protection 機制必須能：
- **指定（specify）** 要實施的控制
- **強制執行（enforce）** 這些控制
- **區分（distinguish）** 授權與未授權的使用

**識別使用者**——系統必須能分辨所有使用者，才能決定誰可以做什麼：

| 名詞 | 中文 | 說明 |
|---|---|---|
| **User ID（UID）／Security ID（SID）** | 使用者識別碼 | 每位使用者一組名稱與編號；會關聯到該使用者的所有 process，用來做存取控制 |
| **Group ID（GID）** | 群組識別碼 | 定義一群使用者並統一管理；也關聯到每個 process、thread |
| **Privilege escalation** | 權限提升 | 讓使用者切換到權限更高的 **effective ID（有效識別碼）** |

> 💡 補充例子：
> - Protection：Linux 檔案權限 `rwxr-x---` 規定擁有者、群組、其他人各能做什麼。
> - Privilege escalation：在 Ubuntu 用 `sudo` 暫時以管理員權限執行指令。
> - Security：防火牆擋掉來自外部的 DoS 攻擊。

---

## 1.7 虛擬化 (Virtualization)

**Virtualization（虛擬化）**：把**單一電腦的硬體**（CPU、memory、disk drives、network interface cards…）**抽象**成多個不同的執行環境，造成「每個環境都在自己私有電腦上執行」的**錯覺**。

→ 一台實體機器可以**同時執行多個 OS**，每個 OS 在自己的 **VM（Virtual Machine，虛擬機器）** 中。

```
(a) 沒有 VM                 (b) 三台虛擬機器
 processes                  processes | processes | processes
 ─────────                  ──────────┼───────────┼──────────
  kernel                     kernel   |  kernel   |  kernel
 ─────────                    VM1     |   VM2     |   VM3
 hardware                   ─────────────────────────────────
                              virtual machine manager (VMM)
                            ─────────────────────────────────
                                       hardware
```

### 重要名詞

| 名詞 | 中文 | 說明 |
|---|---|---|
| **VMM**（Virtual Machine Manager）＝ **Hypervisor** | 虛擬機器管理員 | 執行 guest OS、管理其資源使用、保護各 guest 互不干擾 |
| **Host OS** | 主機作業系統 | 實體機器上原本的 OS |
| **Guest OS** | 客體作業系統 | 在 VM 裡執行的 OS |

### Hypervisor 的兩種類型 ⭐

| | **Type 1 hypervisor** | **Type 2 hypervisor** |
|---|---|---|
| 位置 | **直接跑在硬體上**（VMM 本身就是 host，natively） | 作為**應用程式**跑在 host OS 上 |
| 例子 | VMware ESX、Citrix XenServer | **VMware Workstation**、**VirtualBox** |
| 常見場合 | 資料中心、伺服器 | 個人電腦、開發測試 |

> 📌 投影片例子（VMware Workstation）：硬體（CPU、memory、I/O devices）上是 host OS（Linux），其上是 virtualization layer，再上面同時跑 FreeBSD、Windows NT、Windows XP 三個 guest OS，各有自己的 virtual CPU、virtual memory、virtual devices。
> 📌 投影片例子：**KVM**（Linux Kernel-based Virtual Machine）——虛擬化功能做在 Linux kernel 裡（KVM Driver），搭配 Qemu 處理 guest 的 I/O。

### Emulation（模擬）

- 用**軟體模擬電腦硬體**，通常用在**來源 CPU 與目標 CPU 類型不同**的情況。
- 一般是**最慢**的方法（每道指令都要翻譯）。

| | **Virtualization** | **Emulation** |
|---|---|---|
| CPU 類型 | guest 與 host 相同 | 來源與目標**不同** |
| 速度 | 較快 | 最慢 |

> 📌 投影片例子：
> - Apple 從 IBM Power CPU 換成 Intel x86 時，**Rosetta** 模擬器讓為 IBM CPU 編譯的程式能在 Intel CPU 上跑。
> - **Android emulator**：模擬一般行動裝置的所有軟硬體功能（除了無法真的撥電話），不需實體手機就能開發、測試 Android App。

### 虛擬化的用途

- 筆電／桌機同時跑多個 OS，用來探索或執行為其他 OS 寫的應用程式（📌 例：Apple 筆電以 macOS 為 host、Windows 為 guest）
- 不需多台電腦就能為多種 OS **開發、測試、除錯**應用程式
- 在**資料中心**內執行與管理運算環境

詳見 Chapter 18。

---

## 1.8 分散式系統 (Distributed Systems)

**Distributed system（分散式系統）**：一群**實體上分開**、可能是 **heterogeneous（異質的）** 的電腦系統，透過**網路**連結，讓使用者能存取系統所維護的各種資源。

**優點**：共享資源可以提高
- computational speed（運算速度）
- functionality（功能性）
- data availability（資料可用性）
- reliability（可靠度）

需要的基礎：network protocol（網路協定）、network adapter（網路介面卡）、device driver。

| | **Network Operating System（網路作業系統）** | **Distributed Operating System（分散式作業系統）** |
|---|---|---|
| 特性 | 提供跨網路系統間的功能；通訊機制讓不同電腦上的 process **交換訊息** | 提供**自主性較低（less autonomous）** 的環境 |
| 使用者感受 | 知道有多台電腦 | 各電腦溝通緊密到產生「**只有單一 OS 控制整個網路**」的錯覺 |
| 相關技術 | 訊息傳遞 | process migration、data/file migration（行程／資料／檔案遷移） |

詳見 Chapter 19。

---

## 1.9 核心資料結構 (Kernel Data Structures)

OS 實作的核心，是 OS 中使用的基本資料結構。

### 串列類

| 結構 | 中文 | 說明 |
|---|---|---|
| **Array** | 陣列 | 每個元素可直接存取 |
| **Singly linked list** | 單向鏈結串列 | 每個節點指向**下一個** |
| **Doubly linked list** | 雙向鏈結串列 | 每個節點指向**前一個與下一個** |
| **Circular linked list** | 環狀鏈結串列 | 最後一個節點指回**第一個** |

### Stack 與 Queue

| | **Stack（堆疊）** | **Queue（佇列）** |
|---|---|---|
| 原則 | **LIFO**（Last In, First Out，後進先出） | **FIFO**（First In, First Out，先進先出） |
| 操作 | **push**（放入）／**pop**（取出） | 從尾端加入、從前端移除 |
| 💡 補充例子 | 疊盤子；函式呼叫時 OS 用 stack 存放參數、區域變數、返回位址 | 排隊結帳；印表機的列印工作、等待 CPU 的 process |

### Tree（樹）

- 用來**階層式**表示資料的結構。
- **Binary search tree（二元搜尋樹）**：left_child ≤ right_child。
  - 搜尋的 worst case 為 **O(n)**（樹退化成一條鏈時）。
- **Balanced binary search tree（平衡二元搜尋樹）**：
  - n 個項目**最多 lg n 層**。
  - worst case 為 **O(lg n)**。
  - 📌 投影片例子：**Linux 在 CPU 排程中使用 red-black tree（紅黑樹）**。

### Hash Function 與 Hash Map（雜湊函式與雜湊表）

- Hash function 把資料當輸入，計算後回傳一個數值，用它當索引直接找到資料。
- 用途之一是實作 **hash map**（key → value 的對應）。OS 中廣泛使用。
- 搜尋效能可達 **O(1)**。
- **Hash collision（雜湊碰撞）**：不同的輸入算出相同的輸出。

> 📌 投影片例子：`H(X) = X % 8`
> - X = 6 → H(6) = 6 % 8 = **6**
> - X = 14 → H(14) = 14 % 8 = **6**
> - 兩個不同的 X 得到同一個位置 → **hash collision**。常見解法是在該位置掛一條 linked list。
>
> ⚠️ 補充：投影片寫「Search worst-case performance is O(1)」。依課本的說法，hash 的優勢是查找可以**快到 O(1)**；但若碰撞嚴重（全部擠在同一格），最差情況會退化到 O(n)。考試時建議以老師上課的說法為準，並可向老師確認。

### Bitmap（位元圖）

- 用 **n 個二進位數字**表示 **n 個項目的狀態**。
- 優點：非常省空間。

> 📌 投影片例子 1：`001011101`，以 0 表示可用、1 表示不可用
>
> | 位置 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
> |---|---|---|---|---|---|---|---|---|---|
> | bit | 0 | 0 | 1 | 0 | 1 | 1 | 1 | 0 | 1 |
>
> → 資源 **0、1、3、7 可用**（available）；2、4、5、6、8 不可用。
>
> 📌 投影片例子 2：磁碟區塊的 bitmap（n 個 disk block）
> - `bit[i] = 0` → block[i] **free（空閒）**
> - `bit[i] = 1` → block[i] **occupied（已佔用）**
>
> 這就是 1.5.4 提到的 free-space management 中的 bit-mapping。

---

## 1.10 運算環境 (Computing Environments)

### 1.10.1 Traditional Computing（傳統運算）

- **Stand-alone（獨立運作）** 的通用機器。
- 但界線已 **blurred（模糊）**，因為大多數系統都與其他系統互連（Internet）。
- **Portal（入口網站）**：提供以 web 存取內部系統。
- **Network computer／thin client（精簡型用戶端）**：像 web 終端機。
- 行動電腦透過**無線網路**互連。
- 網路變得 **ubiquitous（無所不在）**——連家用系統都用 **firewall（防火牆）** 防禦網路攻擊。

### 1.10.2 Mobile Computing（行動運算）

- 指在手持智慧型手機、平板等裝置上的運算。
- 與傳統筆電的功能差異：多了額外的硬體功能，如 **GPS**、**gyroscope（陀螺儀）**。
- 因此有新型態的 App，如 **augmented reality（AR，擴增實境）**。
- 連線方式：**IEEE 802.11 無線網路（Wi-Fi）** 或 **cellular data network（行動數據網路）**。
- 主流：**Apple iOS**、**Google Android**。

### 1.10.3 Client-Server Computing（主從式運算）

**Server（伺服器）** 回應 **client（用戶端）** 產生的請求。

| 類型 | 中文 | 說明 | 💡 補充例子 |
|---|---|---|---|
| **Compute-server system** | 運算伺服器系統 | 提供介面讓 client 請求服務（如資料庫） | 網站後端的資料庫伺服器 |
| **File-server system** | 檔案伺服器系統 | 提供檔案系統介面讓 client 存取檔案 | 學校的網路硬碟、NAS |

### 1.10.4 Peer-to-Peer Computing（P2P，對等式運算）

- 分散式系統的另一種結構，**不區分 client 與 server**。
- 所有節點都是 **peer（對等節點）**，可以當 client、server，或兩者皆是。
- 節點必須先加入 P2P 網路，兩種找服務的方式：
  1. 向網路上的**集中式查詢服務（centralized lookup service）** 註冊自己的服務
  2. **廣播（broadcast）** 服務請求，並透過 **discovery protocol（探索協定）** 回應他人的請求
- 📌 投影片例子：Napster（集中式查詢）、Gnutella（廣播）、**VoIP（Voice over IP）** 如 Skype。

### 1.10.5 Cloud Computing（雲端運算）

> 雲端運算是一種基於網際網路的運算方式，使用者透過網際網路**隨選租用**遠端廠商所提供的資源、平台與應用系統。

- 透過網路把**運算、儲存，甚至應用程式**當成**服務**提供。
- 是 **virtualization 的邏輯延伸**（以虛擬化為基礎）。
- 📌 投影片例子：**Amazon EC2** 有數千台伺服器、數百萬個 VM、PB 等級的儲存，使用者**依用量付費**。

**依部署方式分類**：

| 類型 | 中文 | 說明 |
|---|---|---|
| **Public cloud** | 公有雲 | 透過 Internet 提供給任何願意付費的人 |
| **Private cloud** | 私有雲 | 公司自己營運、給公司自己用 |
| **Hybrid cloud** | 混合雲 | 同時包含公有與私有雲的元件 |

**依服務模式分類** ⭐：

| 類型 | 全名 | 中文 | 說明 | 📌 投影片例子 | 💡 補充例子 |
|---|---|---|---|---|---|
| **SaaS** | Software as a Service | 軟體即服務 | 透過 Internet 提供一或多個**應用程式** | 文書處理、試算表 | Google Docs、Gmail |
| **PaaS** | Platform as a Service | 平台即服務 | 透過 Internet 提供現成可用的**軟體堆疊** | 資料庫伺服器 | Heroku、Google App Engine |
| **IaaS** | Infrastructure as a Service | 基礎設施即服務 | 透過 Internet 提供**伺服器或儲存** | 備份用的儲存空間 | Amazon EC2、S3 |

> 💡 補充例子（吃披薩比喻）：IaaS 是租一間有烤箱的廚房，食材和做法自己來；PaaS 是連麵團醬料都備好，你只管放料；SaaS 是直接外送到家，打開就吃。

**雲端環境的組成**：傳統 OS ＋ VMM ＋ 雲端管理工具（📌 例：VMware vCloud Director）。
- 連上 Internet 需要 firewall 等安全措施。
- **Load balancer（負載平衡器）** 把流量分散到多個應用程式。

### 1.10.6 Real-Time Embedded Systems（即時嵌入式系統）

**Embedded system（嵌入式系統）**：電腦硬體與軟體的組合（可能再加上機械或其他零件），**設計來執行特定工作**。

- 差異很大（vary considerably）：
  - 有些有 OS（通用電腦跑標準 OS ＋ 特殊用途應用程式，功能有限）
  - 有些**沒有 OS** 也能執行工作
- 嵌入式系統幾乎都跑 **embedded real-time OS（嵌入式即時作業系統）**。

> 📌 投影片例子（嵌入式系統清單節錄）：防鎖死煞車（ABS）、自動對焦相機、ATM、自動收費系統、數位相機、指紋辨識器、家用保全、維生系統、印表機、路由器、洗衣機、遊戲主機、機器人、自動駕駛腳踏車／汽車、無人機（drone）、iRobot 掃地機器人。

**Real-time system（即時系統）**：

- 當處理器的運作或資料流有**嚴格的時間要求（rigid time requirements）** 時使用。
- 常作為專用應用中的**控制裝置**。
- 有**明確、固定的時間限制（well-defined, fixed time constraints）**。
- 處理**必須在限制時間內完成，否則系統就算失敗**。
- 只有在時間限制內回傳**正確結果**，才算運作正確。

> 📌 投影片例子：科學實驗控制、醫學影像系統、工業控制系統、汽車引擎、家電控制器、武器系統。
> 💡 補充例子：汽車安全氣囊必須在撞擊後幾十毫秒內彈出。晚 1 秒才算出「應該彈出」，就算答案正確也沒有意義。

詳見 Chapter 5。

---

## 1.11 自由與開放原始碼作業系統

### Open-source vs. Closed-source

- **Open-source OS（開放原始碼作業系統）**：以**原始碼**形式提供。
- **Closed-source（封閉原始碼）**：只提供編譯後的**二進位檔（binary）**。

### 三個容易混淆的名詞 ⭐

| 名詞 | 中文 | 重點 |
|---|---|---|
| **Freeware** | 免費軟體 | **不用付錢**就能取得，但通常有其他限制；使用者**沒有**使用、複製、研究、修改、散佈的自由；原始碼不一定公開 |
| **Free Software** | 自由軟體 | 重點是**自由（freedom）** 而不是免費；使用者擁有四種自由 |
| **Open Source** | 開放原始碼 | 原始碼可取得 |

> 💡 補充：Free 在這裡是「自由」的意思，不是「免費」。常見的說法是 *free as in free speech, not free beer*。

### Richard Stallman 與 FSF

- **Richard Stallman**：1953 年生於美國，1971 年受聘於 MIT 人工智慧實驗室（AI Laboratory）。
- **1984 年**發起 **GNU 計劃**：創造一套完全自由、相容於 Unix 的作業系統 GNU（*GNU's Not Unix!*）。
- **1985 年**成立 **FSF（Free Software Foundation，自由軟體基金會）**，鼓勵自由交換軟體原始碼與自由使用軟體。
- GNU 的成果：`bash`、`ispell`、`gcc`、`grep` 等。

### 自由軟體的四種自由

| 自由 | 內容 |
|---|---|
| **自由之零** | 不論目的為何，有**使用**該軟體的自由 |
| **自由之一** | 有**研究**該軟體如何運作的自由，並可**修改**以符合自身需求（前提：取得原始碼） |
| **自由之二** | 有**重新散佈**該軟體的自由 |
| **自由之三** | 有**改善並再利用**該軟體的自由，並可發表修訂版本讓整個社群受惠（前提：取得原始碼） |

使用者具有以上四種權利，該軟體才能稱為「自由軟體」。

### GPL（GNU General Public License，GNU 通用公共授權）

- 自由軟體常用的授權條款。
- 規定：原始碼必須**隨二進位檔一起散佈**；對原始碼做的任何修改，也必須以**同樣的 GPL** 釋出。
- 精神：**自由、分享、互惠**——你免費取得原始碼，修改後基於公平互惠也必須公開成果。
- 📌 投影片例子：GNU/Linux、BSD UNIX（包含 Mac OS X 的核心）等。

---

## 重點整理：OS 什麼時候會執行？

這是投影片貫穿全章的主線，共四個時機（加上 timer）：

| # | 時機 | 觸發來源 | 流程 | 例子 |
|---|---|---|---|---|
| **(1)** | **System booting（系統開機）** | 電源開啟 | ROM 的 bootstrap → 載入 kernel 到 RAM → 啟動 GUI / Shell | 按電源鍵 |
| **(2)** | **Interrupt occurs（發生中斷）** | **硬體**裝置 | interrupt vector → ISR → device driver | 鍵盤輸入、磁碟讀取完成 |
| **(3)** | **Exception occurs（發生例外）** | 程式**錯誤** | interrupt vector → exception handler | `a = 5/0`、存取非法記憶體 |
| **(4)** | **System call occurs（發生系統呼叫）** | 程式**主動請求** | trap → system call handler → service routine | `fork()`、讀寫檔案 |
| (+) | **Timer interrupt（計時器中斷）** | 計時器到時 | interrupt vector → timer handler → 切換 process | 時間片用完、無窮迴圈 |

共同點：**每一次都會從 user mode 切到 kernel mode，由 OS 接手，處理完再切回 user mode。**

---

## 容易混淆的觀念對照

| 比較 | A | B |
|---|---|---|
| **Interrupt vs. Trap** | Interrupt：通常由**硬體**產生，與目前執行的指令無關（非同步） | Trap / Exception：由**軟體**產生，因錯誤或 system call 而起 |
| **Program vs. Process** | 被動，磁碟上的檔案 | 主動，執行中的程式 |
| **Multiprogramming vs. Multitasking** | 等待時才切換，目標是 CPU 使用率 | 時間片到就切換，目標是回應時間 |
| **Multiprocessor vs. Clustered** | 多個 CPU 在**同一台機器**內，共享 bus／memory | **多台獨立機器**（node）透過網路連結，共享儲存 |
| **SMP vs. Asymmetric MP** | 每個 CPU 對等，做所有工作 | boss 分派工作給 worker |
| **UMA vs. NUMA** | 存取任何記憶體時間相同 | 存取本地記憶體較快 |
| **Interrupt-driven I/O vs. DMA** | CPU 親自搬資料，每 byte 一次中斷 | 控制器直接搬到記憶體，每 block 一次中斷 |
| **Protection vs. Security** | 控制**內部**資源存取的機制 | 防禦內外部**攻擊** |
| **Virtualization vs. Emulation** | 同類型 CPU，較快 | 不同類型 CPU，用軟體模擬，最慢 |
| **Type 1 vs. Type 2 hypervisor** | 直接跑在硬體上（ESX、XenServer） | 跑在 host OS 上（VirtualBox、VMware Workstation） |
| **Client-Server vs. P2P** | 角色固定：server 提供、client 請求 | 每個節點既是 client 也是 server |
| **Network OS vs. Distributed OS** | 使用者知道有多台電腦 | 讓使用者以為只有單一 OS |
| **Freeware vs. Free Software** | 免費，但無修改散佈自由 | 有四種自由，重點不在價格 |
| **Buffering vs. Caching vs. Spooling** | 傳輸時暫存 | 為了**效能**放到較快的儲存 | 替不能交錯輸出的裝置排隊保存輸出 |

---

## 作業（Homework）

### 書面練習（Written exercises）

- 課本 p.53：**EX 1.5、1.6、1.8、1.9**
- **1.12**：How do clustered systems differ from multiprocessor systems? What is required for two machines belonging to a cluster to cooperate to provide a highly available service?
  （叢集系統與多處理器系統有何不同？叢集中的兩台機器要合作提供高可用性服務，需要什麼？）
  → 複習本筆記 [1.3](#13-電腦系統架構-computer-system-architecture)
- **1.14**：What is the purpose of interrupts? How does an interrupt differ from a trap? Can traps be generated intentionally by a user program? If so, for what purpose?
  （中斷的目的是什麼？interrupt 與 trap 有何不同？使用者程式可以故意產生 trap 嗎？如果可以，目的是什麼？）
  → 複習本筆記 [1.2 中斷](#中斷-interrupt--os-執行時機-2) 與 [1.4](#中斷trap-與-exception--os-執行時機-3)
- 針對以下各類作業系統，描述其**基本特性（essential properties）**，並討論其**興起的原因**：
  - multiprogramming system
  - multitasking system
  - multiprocessor system
  - clustered system
  - real-time system
  - embedded system

### 實作：虛擬化軟體與 Linux 安裝

可用軟體：VMware Workstation Pro 26H1、**VirtualBox 7.2**、**Ubuntu Linux 24.04 / 26.04**

| 配分 | 項目 |
|---|---|
| **80%** | **Virtualization software & Linux installation** |
| ├ 20% | 帳號（username）：**學號** |
| ├ 20% | 電腦名稱（computer's name）：**學號** |
| ├ 10% | 網路要通（5% 開 browser 測試；5% 開 Terminal 下 command 測試） |
| ├ 20% | 重點螢幕截圖（Screenshot） |
| └ 10% | 列出參考文獻資料來源（YouTube 或網頁文件） |
| **10%** | **Write / Compile / Run a program**：寫個小程式來執行（Screenshot、demo） |
| **10%** | **Demo to TA**，展示使用方式 |

參考資料：vi / vim 程式編輯器 — <https://linux.vbird.org/linux_basic/centos7/0310vi.php>

> 💡 這份作業剛好把本章的觀念實際做一遍：VirtualBox 是 **Type 2 hypervisor**，你的 Windows / macOS 是 **host OS**，裝進去的 Ubuntu 是 **guest OS**；在 Terminal 下指令用的是 **CLI shell**。

---

## 英中詞彙總表

### A–C

| English | 中文 | 補充 |
|---|---|---|
| access control | 存取控制 | |
| accounting | 記帳／使用紀錄 | |
| active entity | 主動實體 | 指 process |
| allocation / deallocation | 配置／釋放 | |
| application program | 應用程式 | |
| asymmetric clustering | 非對稱叢集 | 一台 hot-standby |
| asymmetric multiprocessing | 非對稱式多處理 | boss–worker |
| augmented reality (AR) | 擴增實境 | |
| authorization | 授權 | |
| autonomous | 自主的 | |
| availability | 可用性 | 投影片譯「可利用性」 |
| balanced binary search tree | 平衡二元搜尋樹 | |
| binary search tree | 二元搜尋樹 | |
| bitmap | 位元圖 | |
| blurred | 模糊的 | |
| bootstrap program | 啟動程式 | 存於 ROM / EEPROM |
| buffer / buffering | 緩衝區／緩衝 | |
| bus | 匯流排 | |
| cache / caching | 快取 | |
| cache coherency | 快取一致性 | 投影片譯「統一；連貫性」 |
| CLI (Command Line Interface) | 命令列介面 | |
| client-server | 主從式 | |
| cloud computing | 雲端運算 | |
| clustered system | 叢集系統 | |
| command interpreter | 命令直譯器 | 即 shell |
| concurrency / concurrent | 並行（同時發生） | |
| contiguous allocation | 連續配置 | |
| control program | 控制程式 | |
| core | 核心（CPU 運算核心） | 注意與 kernel 區分 |
| CPU scheduling | CPU 排程 | |
| CPU utilization | CPU 使用率 | |

### D–H

| English | 中文 | 補充 |
|---|---|---|
| deadlock | 死結 | |
| denial-of-service (DoS) | 阻斷服務 | |
| device controller | 裝置控制器 | 硬體 |
| device driver | 裝置驅動程式 | 軟體 |
| directory | 目錄 | |
| dispatch table | 分派表 | |
| distributed system | 分散式系統 | |
| DLM (Distributed Lock Manager) | 分散式鎖管理器 | |
| DMA (Direct Memory Access) | 直接記憶體存取 | |
| dual-mode operation | 雙模式運作 | |
| embedded system | 嵌入式系統 | |
| emulation / emulator | 模擬／模擬器 | |
| errant | 出錯的 | 投影片譯「走入歧途的」 |
| exception | 例外 | 同 trap |
| exploration | 探究 | |
| fault-tolerant | 容錯 | |
| FIFO (First In, First Out) | 先進先出 | queue |
| firewall | 防火牆 | |
| firmware | 韌體 | |
| free software | 自由軟體 | |
| freeware | 免費軟體 | |
| GPL (General Public License) | 通用公共授權 | |
| graceful degradation | 優雅降級 | |
| guest OS | 客體作業系統 | |
| GUI (Graphical User Interface) | 圖形使用者介面 | |
| gyroscope | 陀螺儀 | |
| hash collision | 雜湊碰撞 | |
| hash function / hash map | 雜湊函式／雜湊表 | |
| heterogeneous | 異質的 | |
| high availability | 高可用性 | |
| hinge on | 取決於 | |
| host OS | 主機作業系統 | |
| hot-standby | 熱待機 | |
| HPC (High-Performance Computing) | 高效能運算 | |
| hypervisor | 虛擬機器管理程式 | 同 VMM |

### I–P

| English | 中文 | 補充 |
|---|---|---|
| IaaS | 基礎設施即服務 | |
| illusion | 錯覺 | |
| intermediary | 中介者 | |
| interrupt | 中斷 | |
| interrupt-driven | 中斷驅動 | |
| interrupt vector | 中斷向量 | |
| ISR (Interrupt Service Routine) | 中斷服務常式 | |
| job scheduling | 工作排程 | |
| kernel | 核心 | OS 的核心程式 |
| kernel mode | 核心模式 | mode bit = 0 |
| LIFO (Last In, First Out) | 後進先出 | stack |
| linked list | 鏈結串列 | |
| load balancer | 負載平衡器 | |
| malicious | 惡意的 | |
| maskable interrupt | 可遮罩中斷 | |
| mass storage | 大量儲存 | |
| middleware | 中介軟體 | |
| mode bit | 模式位元 | |
| mounting / unmounting | 掛載／卸載 | |
| multicore | 多核心 | |
| multiprocessor system | 多處理器系統 | |
| multiprogramming | 多元程式 | |
| multitasking | 多工 | |
| nonvolatile | 非揮發性 | |
| NUMA | 非一致記憶體存取 | |
| NVM (Non-Volatile Memory) | 非揮發性記憶體 | |
| open-source | 開放原始碼 | |
| PaaS | 平台即服務 | |
| paging | 分頁 | |
| parallelization | 平行化 | |
| partitioning | 分割 | |
| passive entity | 被動實體 | 指 program |
| peculiarities | 特殊性 | 投影片譯「奇特、特性」 |
| peer-to-peer (P2P) | 對等式 | |
| peripheral device | 周邊裝置 | |
| placement problem | 放置問題 | |
| primitives | 基本操作 | |
| privilege escalation | 權限提升 | 投影片譯 escalation「逐步擴大」 |
| privileged instruction | 特權指令 | |
| privileged mode | 特權模式 | 同 kernel mode |
| process | 行程 | |
| program counter | 程式計數器 | |
| protection | 保護 | |
| protection rings | 保護環 | |

### Q–Z

| English | 中文 | 補充 |
|---|---|---|
| queue | 佇列 | |
| real-time system | 即時系統 | |
| reclamation | 回收利用 | |
| red-black tree | 紅黑樹 | |
| register | 暫存器 | |
| reliability | 可靠度 | |
| replacement problem / policy | 置換問題／策略 | |
| replica | 複本 | |
| resource allocator | 資源分配者 | |
| response time | 回應時間 | |
| SaaS | 軟體即服務 | |
| SAN (Storage-Area Network) | 儲存區域網路 | |
| secondary storage | 輔助儲存 | |
| security | 安全 | |
| segmentation | 分段 | |
| sequential / random access | 循序／隨機存取 | |
| shell | 殼層 | |
| shuffling | 輪替 | 投影片譯「洗牌」 |
| SMP (Symmetric Multiprocessing) | 對稱式多處理 | |
| spooling | 排存 | |
| stack | 堆疊 | |
| stand-alone | 獨立運作 | |
| supervisor mode | 監督模式 | 同 kernel mode |
| surviving | 存活的 | |
| symmetric clustering | 對稱叢集 | |
| synchronization | 同步 | |
| system call | 系統呼叫 | 也稱 monitor call |
| system program | 系統程式 | |
| tertiary storage | 第三級儲存 | 光碟、磁帶 |
| thin client | 精簡型用戶端 | |
| thread | 執行緒 | |
| throughput | 產能／吞吐量 | |
| time slice | 時間片 | |
| time-sharing | 分時 | |
| timer | 計時器 | |
| trap | 陷阱 | 軟體產生的中斷 |
| ubiquitous | 無所不在的 | |
| UMA | 一致記憶體存取 | |
| user mode | 使用者模式 | mode bit = 1 |
| virtual machine (VM) | 虛擬機器 | |
| virtual memory | 虛擬記憶體 | |
| virtualization | 虛擬化 | |
| VMM (Virtual Machine Manager) | 虛擬機器管理員 | |
| VoIP (Voice over IP) | 網路語音 | |
| volatile / volatility | 揮發性 | |
