# 作業系統 Chapter 8：Deadlocks（死結）筆記

> **教材**：*Operating System Concepts*, 10th ed.（Silberschatz, Galvin, Gagne）
> **投影片**：Mei-Ling Chiang 教授，暨南大學資管系（2026/11/10，共 78 頁）
> **筆記說明**：
> - 英文專有名詞第一次出現時以「**English（中文）**」標示，文末有完整詞彙表。
> - 標示「📌 投影片例子」者出自投影片；標示「💡 補充例子」者是為了幫助理解另外加的比喻或說明，不在投影片內。
> - 標示「⚠️」者是投影片中疑似筆誤或需要留意的地方。
> - 投影片用 **thread（T）** 表示執行單位，在本章中把它想成 process 也完全適用。

---

## 目錄

- [0. 本章要回答的問題](#0-本章要回答的問題)
- [什麼是 Deadlock](#什麼是-deadlock)
- [8.1 系統模型](#81-系統模型-system-model)
- [8.2 多執行緒程式中的死結](#82-多執行緒程式中的死結-deadlock-in-multithreaded-applications)
- [8.3 死結的特徵](#83-死結的特徵-deadlock-characterization)
- [8.4 處理死結的方法](#84-處理死結的方法-methods-for-handling-deadlocks)
- [8.5 死結預防](#85-死結預防-deadlock-prevention)
- [8.6 死結避免](#86-死結避免-deadlock-avoidance)
- [8.7 死結偵測](#87-死結偵測-deadlock-detection)
- [8.8 從死結中復原](#88-從死結中復原-recovery-from-deadlock)
- [重點整理](#重點整理)
- [容易混淆的觀念對照](#容易混淆的觀念對照)
- [作業（Homework）](#作業homework)
- [英中詞彙總表](#英中詞彙總表)

---

## 0. 本章要回答的問題

| 問題 | 中文 | 對應章節 |
|---|---|---|
| What is a deadlock problem? | 什麼是死結問題？ | 8.1–8.3 |
| How to **prevent** a deadlock? | 如何**預防**死結？（**主動積極策略**） | 8.5 |
| How to **avoid** a deadlock? | 如何**避免**死結？（**消極策略**） | 8.6 |
| How to **detect** a deadlock? | 如何**偵測**死結？ | 8.7 |
| How to **recover** from a deadlock? | 如何從死結中**復原**？ | 8.8 |

**本章目標（Chapter Objectives）**：
- 說明使用 **mutex lock（互斥鎖）** 時死結如何發生。
- 定義描述死結特徵的 **4 個必要條件（necessary conditions）**。
- 在 **resource-allocation graph（資源配置圖）** 中辨識死結。
- 評估 4 種**預防**死結的方法。
- 應用 **banker's algorithm（銀行家演算法）** 來避免死結。
- 應用 **deadlock detection algorithm（死結偵測演算法）**。
- 評估從死結中**復原**的方法。

> 📌 投影片附的介紹影片：<https://www.youtube.com/watch?v=UVWtwx_izh0>

---

## 什麼是 Deadlock

> 📌 投影片例子：**兩台卡車在窄路上迎面相遇**——Neither truck can proceed（兩台都過不去）。每台都佔著對方需要的路，又都在等對方讓開。

> 📌 投影片例子：**Bridge Crossing Example（過橋例子）**
> 一座窄橋，同一時間只能單向通行。
> - 橋的每一段都可以看成一個**資源（resource）**。
> - 兩邊的車同時開上橋、在中間卡住，就發生死結。
> - 死結發生時，只要**其中一台車倒車**就能解開——相當於**搶占資源並回復（preempt resources and rollback）**。
> - 有時候**好幾台車**都得倒車。
> - 可能發生 **starvation（飢餓）**：例如某一邊的車一直被要求倒車，永遠過不了橋。

### Deadlock 的定義 ⭐

> **A situation in which every process in a set of processes is waiting for an event that can be caused only by another process in the set.**
>
> 一組 process 中的**每一個** process 都在等待一個事件，而這個事件**只能由同一組中的另一個 process 引發**。

換一種說法（The Deadlock Problem）：
> 一組**被 block 的 process**，每一個都**持有（holding）** 某個資源，同時又在**等待取得（waiting to acquire）** 同一組中其他 process 所持有的資源。

> 📌 投影片例子 1：系統有 **3 台 CD-RW 燒錄機**。P1、P2、P3 各**持有一台**，又各自**再要求一台**。三台都被佔走了，誰也拿不到第二台，大家都在等別人放手。
>
> 📌 投影片例子 2：兩個 **semaphore A、B**，初始值都是 1
> ```
>         P0              P1
>      wait(A);        wait(B);
>      wait(B);        wait(A);
> ```
> 若執行順序是 P0 `wait(A)` → P1 `wait(B)` → P0 `wait(B)`（被擋）→ P1 `wait(A)`（被擋），兩個都會永遠等下去。
>
> 💡 補充：`wait()` 就是取得（acquire）的意思，semaphore 為 1 時拿得到並把它變成 0；為 0 時就要等。

> 💡 補充例子：兩個小朋友畫畫，一個拿著紅筆想要藍筆，另一個拿著藍筆想要紅筆，兩人都不肯先放下手上的筆。

---

## 8.1 系統模型 (System Model)

- 系統由**有限數量的資源**組成，分配給多個互相競爭的 thread 使用。
  - 實體資源：**CPU cycles、files、I/O devices**（例如 network interface、DVD drive）
  - 邏輯資源：**mutex locks、semaphores**
- 資源被分成數種 **type（類型）**，又稱 **class（類別）**；每種類型由若干個**相同的 instance（實例）** 組成。
  - 💡 補充例子：系統有 2 個 CPU，則「CPU」這個 resource type 有 2 個 instance。Thread 要一個 CPU 時，**任何一個 instance 都能滿足**它。

**Thread 使用資源的三個步驟**：

| 步驟 | 中文 | 說明 |
|---|---|---|
| **Request** | 請求 | 請求資源；如果無法立刻取得，就**必須等待** |
| **Use** | 使用 | 使用該資源 |
| **Release** | 釋放 | 用完後釋放 |

> 📌 投影片例子：request 與 release 的 system call
>
> | 資源 | Request | Release |
> |---|---|---|
> | 檔案 | `open()` | `close()` |
> | 裝置 | `request()` | `release()` |
> | 記憶體 | `allocate()` | `free()` |

---

## 8.2 多執行緒程式中的死結 (Deadlock in Multithreaded Applications)

開發多執行緒程式的人必須時時注意死結的可能性。

> 📌 投影片例子：建立並初始化兩個 mutex lock
> ```c
> pthread_mutex_t first_mutex;
> pthread_mutex_t second_mutex;
>
> pthread_mutex_init(&first_mutex, NULL);
> pthread_mutex_init(&second_mutex, NULL);
> ```
>
> 📌 投影片例子（Fig. 8.1）：Deadlock example
> ```c
> /* thread one runs in this function */
> void *do_work_one(void *param)
> {
>     pthread_mutex_lock(&first_mutex);
>     pthread_mutex_lock(&second_mutex);
>     /** Do some work */
>     pthread_mutex_unlock(&second_mutex);
>     pthread_mutex_unlock(&first_mutex);
>     pthread_exit(0);
> }
>
> /* thread two runs in this function */
> void *do_work_two(void *param)
> {
>     pthread_mutex_lock(&second_mutex);
>     pthread_mutex_lock(&first_mutex);
>     /** Do some work */
>     pthread_mutex_unlock(&first_mutex);
>     pthread_mutex_unlock(&second_mutex);
>     pthread_exit(0);
> }
> ```
> ⚠️ 投影片程式中的 `first mutex`、`second mutex` 少了底線，應為 `first_mutex`、`second_mutex`（上面已補正）。

**死結如何發生**：

| 時間 | thread_one | thread_two |
|---|---|---|
| 1 | 取得 `first_mutex` ✔ | |
| 2 | | 取得 `second_mutex` ✔ |
| 3 | 要 `second_mutex` → **等待**（被 thread_two 持有） | |
| 4 | | 要 `first_mutex` → **等待**（被 thread_one 持有） |
| 5 | 永遠等下去 | 永遠等下去 |

**用 resource allocation graph 表示**：

```
   ┌─────────────┐                 ┌──────────────┐
   │ first_mutex │ ◄───────┐       │ second_mutex │
   └──────┬──────┘         │       └──────┬───────┘
          │ 持有            │  請求    ▲     │ 持有
          ▼                │         │     ▼
     thread_one ───────────┼── 請求 ─┘  thread_two
                           │                │
                           └────────────────┘
   thread_one 持有 first、等 second；thread_two 持有 second、等 first → 形成環
```

> 💡 補充：這個程式**不一定每次都會死結**——要剛好在 thread_one 拿到第一把鎖、還沒拿第二把之前，thread_two 搶先拿到 second_mutex 才會發生。所以死結的 bug 很難測試和重現。
> 💡 補充：最簡單的修法是讓兩個 thread **用相同順序**取鎖（都先拿 `first_mutex` 再拿 `second_mutex`），這就是 8.5.4 的 lock ordering 觀念。

---

## 8.3 死結的特徵 (Deadlock Characterization)

（characterization：特徵）

- 死結發生時，一組中的每個 process 都被 block，等待只有組內另一個 process 才能引發的事件。
- 在死結中，process **永遠無法執行完畢**，系統資源也被**綁住（tied up）**，使其他工作無法開始。

### 8.3.1 Necessary Conditions（必要條件）⭐⭐

死結只有在下列**四個條件同時成立（hold simultaneously）** 時才可能發生：

| 條件 | 中文 | 定義 | 💡 補充例子（過橋） |
|---|---|---|---|
| **Mutual exclusion** | 互斥 | 同一時間**只有一個** thread 能使用該資源 | 一段橋面一次只能停一台車 |
| **Hold and wait** | 持有並等待 | 一個 thread **至少持有一個資源**，同時在**等待**取得其他 thread 持有的額外資源 | 車子佔著一段橋，又在等前面那段 |
| **No preemption** | 不可搶占 | 資源只能由持有它的 thread 在完成工作後**自願釋放**，不能被強制拿走 | 不能把別人的車吊走 |
| **Circular wait** | 循環等待 | 存在一組等待中的 thread {T0, T1, …, Tn}：T0 等 T1 持有的資源、T1 等 T2 的、…、Tn−1 等 Tn 的、**Tn 又等 T0 的** | 兩邊的車互相等對方先退 |

```
 Circular wait：
   T0 ──等──► T1 ──等──► T2 ──等──► … ──等──► Tn
   ▲                                          │
   └──────────────────等──────────────────────┘
```

> 💡 補充：四個條件**缺一不可**。所以只要讓**任一個條件不成立**，死結就不可能發生——這就是 8.5 deadlock prevention 的出發點。
> 💡 補充：課本也提到這四個條件並非完全獨立，circular wait 其實隱含了 hold and wait。

### 8.3.2 Resource-Allocation Graph（資源配置圖）⭐

用 **system resource-allocation graph** 可以更精確地描述死結。它是一個有向圖，由**頂點集合 V** 與**邊集合 E** 組成。

**頂點（Vertices）分兩種**：

| 種類 | 集合 | 圖示 |
|---|---|---|
| **Thread** | T = {T1, T2, …, Tn}：系統中所有活動中的 thread | **圓形** ○ |
| **Resource type** | R = {R1, R2, …, Rm}：系統中所有資源類型 | **方形** □，裡面的每個**點 •** 代表一個 instance |

**邊（Edges）分兩種**：

| 種類 | 中文 | 方向 | 意義 |
|---|---|---|---|
| **Request edge** | 請求邊 | **Ti → Rj** | Ti **已請求**並正在等待 Rj 的一個 instance（指向方框） |
| **Assignment edge** | 分配邊 | **Rj → Ti** | Rj 的一個 instance **已分配給** Ti（從方框裡的某個點出發） |

```
  Thread：        ( Ti )
  Resource type   ┌───────┐
  with 4 instances│ • • • •│
                  └───────┘
  Ti requests Rj：( Ti ) ──────►┌─────┐
                                │  •  │ Rj
                                └─────┘
  Ti is holding Rj：( Ti ) ◄────┌─────┐
                                │  •  │ Rj
                                └─────┘
```

> 📌 投影片例子（Fig. 8.4）：一般的 resource-allocation graph
> - R1、R3 各 1 個 instance；R2 有 2 個；R4 有 3 個。
> - 邊：T1 → R1（請求）、R1 → T2（分配）、T2 → R3（請求）、R3 → T3（分配）、R2 → T1、R2 → T2（分配）
>
> | Thread | 持有 | 等待 |
> |---|---|---|
> | T1 | R2 的一個 instance | R1 |
> | T2 | R1、R2 的一個 instance | R3 |
> | T3 | R3 | （無） |
>
> 圖中**沒有 cycle**，所以**沒有死結**。T3 可以做完並釋放 R3 → T2 拿到 R3 做完，釋放 R1、R2 → T1 拿到 R1 做完。

> 📌 投影片例子（Fig. 8.5）：**Resource allocation graph with a deadlock**
> 在 Fig. 8.4 的基礎上，T3 **再請求 R2**（加上 T3 → R2）。此時 R2 的兩個 instance 都已分配出去。
> 出現**兩個 cycle**：
> - T1 → R1 → T2 → R3 → T3 → R2 → T1
> - T2 → R3 → T3 → R2 → T2
>
> T1、T2、T3 全部卡住——**死結**。
> 投影片標語：**Deadlock implies cycle!!!（deadlock → cycle）**

> 📌 投影片例子（Fig. 8.6）：**Resource allocation graph with a cycle but no deadlock**
> - R1、R2 各有 2 個 instance。
> - R1 分配給 T2、T3；R2 分配給 T1、T4；T1 請求 R1；T3 請求 R2。
> - **有一個 cycle**：T1 → R1 → T3 → R2 → T1
> - 但**沒有死結**：T4 沒有在等任何東西，可以做完並**釋放 R2 的一個 instance**，T3 就拿得到 R2；T2 做完也會釋放 R1 給 T1。cycle 因此被打破。
>
> 投影片標語：**A cycle does not imply deadlock!!!（但 deadlock → cycle 仍成立）**

### Deadlocks 與 Cycles 的關係 ⭐⭐

| 圖的情況 | 結論 |
|---|---|
| **沒有 cycle** | **一定沒有**死結 |
| 有 cycle，且**每種資源只有一個 instance** | **一定有**死結 |
| 有 cycle，且**資源有多個 instance** | **可能**有死結（cycle 不一定代表死結） |

（imply：暗指、暗示、意味）

> 💡 補充（邏輯關係）：
> - **deadlock ⇒ cycle**（有死結一定有 cycle）
> - **cycle ⇏ deadlock**（有 cycle 不一定有死結），除非每種資源都只有 1 個 instance
> - 等價說法：**no cycle ⇒ no deadlock**

---

## 8.4 處理死結的方法 (Methods for Handling Deadlocks)

三種方法：

| # | 方法 | 對應技術 |
|---|---|---|
| 1 | 用某種**協定（protocol）** 確保系統**永遠不會進入**死結狀態 | **Deadlock prevention**（預防）或 **deadlock avoidance**（避免） |
| 2 | **允許**系統進入死結狀態，**偵測**到之後再**復原** | **Deadlock detection**（偵測）＋ **deadlock recovery**（復原） |
| 3 | **完全忽略**這個問題，假裝系統中不會發生死結 | **大多數 OS 採用，包括 Linux 與 Windows** |

> 💡 補充：第 3 種方法常被戲稱為 **ostrich algorithm（鴕鳥演算法）**——把頭埋進沙裡。因為死結很少發生，而預防、避免、偵測都有額外成本，所以一般 OS 把處理死結的責任交給應用程式開發者。

### Deadlock Prevention 與 Deadlock Avoidance 的差別 ⭐

| | **Deadlock prevention（預防）** | **Deadlock avoidance（避免）** |
|---|---|---|
| 做法 | 確保 4 個必要條件中**至少一個不成立** | OS **事先**得知每個 thread 在生命週期中會請求、使用哪些資源 |
| 手段 | **限制請求資源的方式**（constraining how requests can be made） | 每次請求時**決定**能否立刻滿足，或必須**延後** |
| 需要的資訊 | 不需要事先資訊 | 需要**額外的事先資訊**（例如最大需求量） |
| 投影片說法 | **主動積極策略** | **消極策略** |

> 💡 補充例子：
> - Prevention 像**交通法規**：直接規定「這條路只能單向通行」，從根本讓對撞不可能發生。
> - Avoidance 像**塔台管制**：飛機要起飛前先報告航線，塔台判斷放行之後是否仍然安全，不安全就先等一下。
> - Detection ＋ recovery 像**事故處理**：先讓大家照常開，真的卡住了再派拖吊車處理。

---

## 8.5 死結預防 (Deadlock Prevention)

**Restrain（限制）** 請求資源的方式，確保 **4 個條件不會同時成立**。

### 8.5.1 Mutual Exclusion（互斥）

- **Nonsharable（不可共享）** 的資源**一定要**互斥。
- **Sharable（可共享）** 的資源**不需要**互斥。
  - 例如 **read-only files（唯讀檔案）**：多個 thread 同時讀也沒問題，不會為它等待。
- **結論**：一般而言，**無法靠否定 mutual exclusion 來預防死結**，因為有些資源**本質上（intrinsically）就是不可共享的**（例如 mutex lock、印表機）。

### 8.5.2 Hold and Wait（持有並等待）

必須保證：thread **請求資源時，手上沒有持有任何其他資源**。

| Protocol | 做法 |
|---|---|
| **Protocol 1** | 每個 thread 在**開始執行之前**，就請求並取得它需要的**所有**資源 |
| **Protocol 2** | Thread **只有在手上沒有資源時**才能請求資源。要請求額外資源之前，必須先**釋放目前持有的所有資源** |

**缺點**：

| 缺點 | 說明 |
|---|---|
| **Low resource utilization（資源使用率低）** | 資源可能被配置了很久卻沒有用到 |
| **Starvation possible（可能飢餓）** | 需要多個熱門資源的 thread 可能一直湊不齊而無限期等待 |

> 📌 投影片例子：一個 thread 要 **把 DVD 光碟機的資料複製到磁碟檔案 → 排序磁碟檔案 → 把結果印到印表機**
> ```
>   DVD drive ──► Disk file ──► printer
> ```
> **Protocol 1**：執行前一次取得 3 個資源
> ```
>   取得 DVD drive、disk file、printer
>     從 DVD 讀入，寫到磁碟
>     排序資料，印到印表機
> ```
> → 印表機在整個執行期間都被佔著，但其實**只有最後才用到**（使用率低）。
>
> **Protocol 2**：分段取得
> ```
>   取得 DVD drive 和 disk file
>     從 DVD 讀入，寫到磁碟
>   釋放 DVD drive 和 disk file
>   取得 disk file 和 printer
>     把磁碟檔案印到印表機
> ```
> → 印表機只在需要時才佔用；但中間釋放了 disk file，**不保證**之後拿得回來（資料可能被改動，也可能一直等不到）。

### 8.5.3 No Preemption（不可搶占）

- 適合用在**狀態容易保存與還原**的資源，例如 **CPU registers**、**database transactions（資料庫交易）**。
- 一般**無法**用在 **mutex lock、semaphore** 這類資源。

**Protocol 1：自己請求不到，就交出手上所有資源**
- 如果一個 thread 持有某些資源，又請求另一個**無法立即配置**的資源，那它目前持有的**所有資源都被搶占**（**implicitly released，隱含地釋放**）。
- 這個 thread 只有在能**重新取得舊資源以及正在請求的新資源**時，才會重新開始。

> 📌 投影片例子（圖）：T2 持有 R1、R2，請求 R3；但 R3 被 T3 持有，無法立即配置。
> ```
>   R1 ─┐
>        ├──► T2 ──請求──► R3 ──► T3
>   R2 ─┘
>   → T2 持有的 R1、R2 被搶占（打叉）
>   → T2 要等到能同時拿回 R1、R2 並取得 R3 時才重新開始
> ```

**Protocol 2：從「正在等待的 thread」那裡搶**
- 如果請求的資源已配置給**另一個正在等待額外資源的 thread**，就從那個等待中的 thread **搶占**這些資源，配置給請求者。
- 被搶的 thread 只有在**取得它請求的新資源**，並**收回等待期間被搶走的資源**時，才能重新開始。

> 📌 投影片例子（圖）：T1 請求 R1，但 R1 被 T2 持有；T2 持有 R1、R2，正在等 T3 持有的 R3。
> ```
>   T1 ──請求──► R1 ──► T2 ──請求──► R3 ──► T3
>                       ▲
>               R2 ─────┘
>   → T2 自己也在等，所以把 R1 從 T2 搶過來（打叉）給 T1
> ```
>
> 💡 補充：兩個 protocol 的差別——Protocol 1 是「請求者自己交出資源」；Protocol 2 是「請求者去搶**正在等待**的人的資源」。不會去搶**正在執行**的 thread 的資源。

### 8.5.4 Circular Wait（循環等待）⭐

- 對所有資源類型**強加（impose）** 一個**全序（total ordering）**：
  - 定義函數 **F: R → N**，替每種資源編號（**resource numbering**）。
  - F 應該依照資源**正常的使用順序**來定義。

| Protocol | 做法 |
|---|---|
| **Protocol 1** | 每個 thread 只能依照編號**遞增（increasing order of enumeration）** 的順序請求資源 |
| **Protocol 2** | Thread 請求 Rj 時，必須**已經釋放**所有滿足 **F(Ri) ≥ F(Rj)** 的資源 Ri |

（enumeration：列舉）

> 📌 投影片例子：
> ```
> F(DVD drive)  = 1
> F(disk drive) = 5
> F(printer)    = 12
> ```
> **Protocol 1**：依序請求 R1 → R5 → R12 ✔（編號遞增）
>
> **Protocol 2**：已持有 R1、R5、R12，現在想要 R6
> ```
>   R1   R5   R12   R6
>             ✘
> ```
> 因為 F(R12) = 12 ≥ F(R6) = 6，所以**必須先釋放 R12** 才能請求 R6。
>
> **為什麼這樣就不會有 circular wait？**（投影片右側的環狀圖）
> 假設存在一個環：T1 等 R1（被 T2 持有）、T2 等 R2（被 T3 持有）、…、Tn 等 Rn（被 T1 持有）。
> 依規則，持有 Ri 的 thread 只能再請求編號更大的資源，所以 F(R1) < F(R2) < … < F(Rn) < F(R1)，**F(R1) < F(R1) 不可能成立**——矛盾，所以不會有環。

**預防死結可能的副作用（Possible Side Effects）**：
- **Low device utilization**（裝置使用率低）
- **Reduced system throughput**（系統產能降低）

### Lock Ordering 也不一定安全

> 📌 投影片例子（Fig. 8.7）：**Deadlock example with lock ordering**（轉帳）
> ```c
> void transaction(Account from, Account to, double amount)
> {
>     mutex lock1, lock2;
>     lock1 = get_lock(from);
>     lock2 = get_lock(to);
>
>     acquire(lock1);
>         acquire(lock2);
>             withdraw(from, amount);
>             deposit(to, amount);
>         release(lock2);
>     release(lock1);
> }
> ```
> - 一個 thread：`transaction(account_A, account_B, 25);`  → A 轉 25 給 B，先鎖 A 再鎖 B
> - 另一個 thread：`transaction(account_B, account_A, 50);` → B 轉 50 給 A，先鎖 B 再鎖 A
>
> ```
>              T1（A → B）
>          ┌──持有──┘  └──等待──┐
>          ▼                    ▼
>       帳戶 A                帳戶 B
>          │                    │
>          └──等待──┐  ┌──持有──┘
>              T2（B → A）
> ```
> 兩個 thread **同時**執行時，T1 拿到 A 的鎖、T2 拿到 B 的鎖，接著各自等對方的鎖——**deadlock**。
>
> **重點**：程式碼本身「看起來」有固定的取鎖順序（先 from 再 to），但**鎖的順序取決於參數**，所以不同呼叫之間的順序還是可能相反。
>
> 💡 補充：常見的修法是改成**依帳戶編號**決定取鎖順序，例如永遠先鎖編號小的帳戶。這樣不論誰轉給誰，取鎖順序都一致。

> 📌 投影片附的暨大資管學生作品（死結影片）：
> - <https://www.youtube.com/watch?v=wdEqnTKr9ZM&t=30s>
> - <https://www.youtube.com/watch?v=-MA5UJVzQdk>

---

## 8.6 死結避免 (Deadlock Avoidance)

- 要求 OS 擁有**事前資訊（a priori information）**：知道每個 thread 將如何請求系統資源。
- **最簡單也最有用的模型**：每個 thread **宣告**它對每種資源**可能需要的最大數量（maximum number）**。
- Deadlock-avoidance algorithm **動態地**檢查 **resource-allocation state（資源配置狀態）**，確保**永遠不會出現 circular-wait**。

**Resource-allocation state 由三者定義**：
1. **可用**資源的數量（available）
2. **已配置**資源的數量（allocated）
3. 各 thread 的**最大需求（maximum demands）**

### 8.6.1 Safe State（安全狀態）⭐⭐

- Thread 請求可用的資源時，系統必須判斷：**立刻配置**之後，系統是否仍處於 **safe state**。
- **Safe state**：系統能以**某種順序**把資源配置給每個 thread（最多到它的最大需求），並且仍能避免死結。
- 系統處於 safe state **若且唯若（iff）** 存在一個 **safe sequence（安全序列）** ⟨T1, T2, …, Tn⟩，使所有 thread 都能完成。

#### Safe Sequence（安全序列）

序列 ⟨T1, T2, …, Tn⟩ 是目前配置狀態的 safe sequence，條件是：

> 對每個 Ti，**Ti 還可能提出的請求**，都能被「**目前可用的資源 ＋ 所有排在它前面的 Tj（j < i）所持有的資源**」滿足。

- 如果 Ti 需要的資源**目前不夠**，Ti 可以**等到所有 Tj 都完成**。
- Tj 完成後，Ti 就能取得所需資源、執行、歸還資源並結束。
- Ti 結束後，Ti+1 就能取得它需要的資源，依此類推。
- **如果不存在這樣的序列**，系統就是 **unsafe（不安全）** ⇒ **可能**導致死結。

#### Safe、Unsafe、Deadlock 的關係

```
 ┌──────────────────────────────────┐
 │  unsafe            ┌───────────┐ │
 │                    │ deadlock  │ │
 │                    └───────────┘ │
 │ ──────────────────────────────── │
 │  safe                            │
 └──────────────────────────────────┘
```

| 狀態 | 結論 |
|---|---|
| **Safe** | ⇒ **一定沒有**死結 |
| **Unsafe** | ⇒ **可能**有死結（死結狀態是 unsafe 的一個**子集合**） |
| **Avoidance 的目標** | ⇒ 確保系統**永遠不會進入 unsafe 狀態** |

> 💡 補充：**unsafe 不等於 deadlock**。Unsafe 只表示 OS 無法**保證**避免死結——如果 thread 剛好沒有用到它宣告的最大需求，可能還是能順利結束。但 avoidance 採取保守的做法，連 unsafe 都不讓它發生。

> 📌 投影片例子：**Safe State Example**——系統有 **12 台 tape drive（磁帶機）**
>
> | Thread | Max Needs（最大需求） | Current Needs（目前持有） |
> |---|---|---|
> | T0 | 10 | 5 |
> | T1 | 4 | 2 |
> | T2 | 9 | 2 |
>
> ⚠️ 投影片表頭寫「Current Needs」，旁邊註明是 **Currently holding**——指的是**目前已持有的數量**，不是還需要的數量。
>
> **步驟 1：算出目前可用**：12 − (5 + 2 + 2) = **3**
>
> **步驟 2：算出每個 thread 還需要多少（Max − 持有）**：T0 = 5、T1 = 2、T2 = 7
>
> **步驟 3：找 safe sequence**
>
> | 順序 | Thread | 還需要 | 可用（執行前） | 能否滿足 | 完成後歸還 → 可用 |
> |---|---|---|---|---|---|
> | 1 | T1 | 2 | 3 | 2 ≤ 3 ✔ | 3 + 2 = **5** |
> | 2 | T0 | 5 | 5 | 5 ≤ 5 ✔ | 5 + 5 = **10** |
> | 3 | T2 | 7 | 10 | 7 ≤ 10 ✔ | 10 + 2 = **12** |
>
> ⟨T1, T0, T2⟩ 是 safe sequence → **safe state**。
>
> **如果 T2 再請求 1 台並被核准**：T2 持有 3 台，可用剩 **2** 台。
>
> | Thread | 還需要 | 可用 2 能否滿足 |
> |---|---|---|
> | T1 | 2 | ✔ → 做完後可用變 4 |
> | T0 | 5 | 4 不夠 ✘ |
> | T2 | 6 | 4 不夠 ✘ |
>
> T1 做完後只有 4 台，T0、T2 都不夠——**沒有 safe sequence → unsafe state**。
>
> **結論**：當初如果讓 T2 **先等**，直到其他 thread 做完並釋放資源，就能避免死結。

**Deadlock avoidance 的運作方式**：
- 確保系統**永遠維持在 safe state**。
  - 每當 thread 請求**目前可用**的資源，系統都要決定：立刻配置，還是讓 thread 等待。
  - **只有在配置後系統仍是 safe 時，才核准請求**。
- 缺點：**資源使用率可能比沒有 avoidance 的系統低**——因為即使請求的資源**目前可用**，thread 仍可能要等。

**兩種 avoidance 演算法**：

| 系統類型 | 演算法 |
|---|---|
| 每種資源**只有一個 instance** | **Resource-Allocation Graph Algorithm** |
| 每種資源有**多個 instance** | **Banker's Algorithm** |

### 8.6.2 Resource-Allocation Graph Algorithm（資源配置圖演算法）

- 適用：每種資源**只有一個 instance** 的系統。
- 資源必須**事先宣告（claimed a priori）**。
- 新增第三種邊：**Claim edge（宣告邊）** Ti → Rj，用**虛線**表示：Ti **將來某個時候可能**會請求 Rj。

**邊的轉換**：

```
           Ti 請求 Rj                   Rj 配置給 Ti
 claim edge ─────────► request edge ─────────────► assignment edge
 Ti - - -► Rj          Ti ──────► Rj               Rj ──────► Ti
      ▲                                                 │
      └──────────────── Ti 釋放 Rj ────────────────────┘
```

**核准規則**：Ti 對 Rj 的請求，**只有在把 request edge Ti → Rj 轉成 assignment edge Rj → Ti 之後不會形成任何 cycle（含 claim edge）時**，才能核准。
- 方法：**假裝核准**，用 **cycle-detection algorithm（環偵測演算法）** 檢查：
  - **沒有 cycle** ⇒ system **safe**
  - **有 cycle** ⇒ system **unsafe**

> 📌 投影片例子（Fig. 8.9、8.10）：
> ```
>              ┌────┐
>     ┌────────│ R1 │◄───────┐
>     ▼        └────┘        │
>   ( T1 )                 ( T2 )
>     ┆                      │ 請求（紅色）
>     ┆ claim（虛線）          ▼
>     └┄┄┄┄┄┄┄►┌────┐◄───────┘
>              │ R2 │
>              └────┘
> ```
> - R1 已分配給 T1；T2 請求 R1（在等）。
> - T1 將來可能請求 R2（claim edge，虛線）。
> - 現在 **T2 請求 R2**——R2 目前是空的，**應該核准嗎？**
>
> 假裝核准，把 T2 → R2 改成 R2 → T2：
> ```
>   T1 ┄┄► R2 ──► T2 ──► R1 ──► T1     ← 形成 cycle
> ```
> 有 cycle ⇒ **unsafe**。如果之後 T1 真的請求 R2，就會死結。
> 所以 **T2 → R2 的請求不能核准**，T2 必須等待（即使 R2 目前是空的）。

### 8.6.3 Banker's Algorithm（銀行家演算法）⭐⭐⭐

- 適用：每種資源有**多個 instance** 的系統。
- 新 thread 進入系統時，必須**宣告**它對每種資源**可能需要的最大數量**（每個 thread 必須事先宣告最大使用量）。
- Thread 請求一組資源時，系統必須判斷配置後是否仍處於 safe state：
  - **Safe** → 配置資源
  - **Unsafe** → thread 必須**等待**，直到其他 thread 釋放足夠的資源

> 💡 補充：名稱的由來——銀行家借錢給客戶時，要確保即使所有客戶都借到他們的額度上限，銀行仍能依某個順序滿足每個人，而不會發生錢不夠、大家都還不了的情況。

#### 資料結構

設 **n = thread 數量**，**m = 資源類型數量**：

| 名稱 | 大小 | 意義 |
|---|---|---|
| **Available** | [1..m] 向量 | `Available[j] = k`：資源類型 Rj 目前有 **k 個可用** |
| **Max** | [1..n, 1..m] 矩陣 | `Max[i][j] = k`：Ti **最多**可能要求 k 個 Rj（**max demand**） |
| **Allocation** | [1..n, 1..m] 矩陣 | `Allocation[i][j] = k`：Ti **目前已分配到** k 個 Rj |
| **Need** | [1..n, 1..m] 矩陣 | `Need[i][j] = k`：Ti **還可能需要** k 個 Rj（**remaining need**） |

```
Need[i][j] = Max[i][j] − Allocation[i][j]
```

**記號**：Needi、Allocationi、Requesti 指矩陣的**第 i 列**（一個向量）。向量比較 **X ≤ Y** 表示**每一個分量**都 X[j] ≤ Y[j]。
（💡 補充例子：(1, 0, 2) ≤ (3, 3, 2) 成立；(0, 2, 0) ≤ (2, 1, 0) 不成立，因為 2 > 1。）

#### 8.6.3.1 Safety Algorithm（安全演算法）

**用途**：檢查系統是否處於 safe state。

```
1. Work = Available
   Finish[i] = false，for i = 0, 1, …, n−1

2. 找一個 i，同時滿足：
     a. Finish[i] == false
     b. Need_i ≤ Work            ← 向量比較
   找不到就跳到步驟 4

3. Work = Work + Allocation_i     ← Ti 做完，釋放它持有的資源
   Finish[i] = true
   回到步驟 2

4. 如果所有 i 都是 Finish[i] == true → safe state
```

> 💡 補充：Work 代表「目前手上能動用的資源」。每找到一個能做完的 thread，就把它持有的資源收回來，讓 Work 變大，再去找下一個。

#### 8.6.3.2 Resource-Request Algorithm（資源請求演算法）

令 **Requesti** 為 thread Ti 的請求向量：

```
1. 如果 Request_i ≤ Need_i → 到步驟 2
   否則 → 錯誤（error）：thread 超過了它宣告的最大需求

2. 如果 Request_i ≤ Available → 到步驟 3
   否則 → Ti 必須等待（資源不夠）

3. 假裝（pretend）把資源配置給 Ti：
     Available    = Available    − Request_i
     Allocation_i = Allocation_i + Request_i
     Need_i       = Need_i       − Request_i

4. 執行 safety algorithm：
     safe   → 真的把資源配置給 Ti
     unsafe → Ti 必須等待 Request_i，並把配置狀態還原（restore）
```

#### 完整例子 ⭐

> 📌 投影片例子：**5 個 thread T0–T4；3 種資源 A（10 個）、B（5 個）、C（7 個）**
>
> 目前的 snapshot：
>
> | Thread | Allocation (A B C) | Max (A B C) | Need = Max − Allocation (A B C) |
> |---|---|---|---|
> | T0 | 0 1 0 | 7 5 3 | **7 4 3** |
> | T1 | 2 0 0 | 3 2 2 | **1 2 2** |
> | T2 | 3 0 2 | 9 0 2 | **6 0 0** |
> | T3 | 2 1 1 | 2 2 2 | **0 1 1** |
> | T4 | 0 0 2 | 4 3 3 | **4 3 1** |
>
> **Available = (3, 3, 2)**
> （💡 驗算：已配置總和 = (7, 2, 5)，總量 (10, 5, 7) − (7, 2, 5) = (3, 3, 2) ✔）

**問題 1：目前是 safe state 嗎？**

> 用 safety algorithm，Work 從 (3, 3, 2) 開始：
>
> | 步驟 | 選中 | Need | ≤ Work？ | Work + Allocation = 新 Work |
> |---|---|---|---|---|
> | 1 | T1 | (1, 2, 2) | ≤ (3, 3, 2) ✔ | (3, 3, 2) + (2, 0, 0) = **(5, 3, 2)** |
> | 2 | T3 | (0, 1, 1) | ≤ (5, 3, 2) ✔ | (5, 3, 2) + (2, 1, 1) = **(7, 4, 3)** |
> | 3 | T4 | (4, 3, 1) | ≤ (7, 4, 3) ✔ | (7, 4, 3) + (0, 0, 2) = **(7, 4, 5)** |
> | 4 | T2 | (6, 0, 0) | ≤ (7, 4, 5) ✔ | (7, 4, 5) + (3, 0, 2) = **(10, 4, 7)** |
> | 5 | T0 | (7, 4, 3) | ≤ (10, 4, 7) ✔ | (10, 4, 7) + (0, 1, 0) = **(10, 5, 7)** |
>
> 序列 **⟨T1, T3, T4, T2, T0⟩** 滿足安全條件 → **safe state**。
> （最後的 Work 等於系統總資源 (10, 5, 7)，可以拿來驗算。）
>
> 💡 補充：**safe sequence 不唯一**。投影片的程式執行範例找到的是 ⟨T1, T3, T0, T2, T4⟩（印成 `<1,3,0,2,4>`）：(3,3,2) → T1 → (5,3,2) → T3 → (7,4,3) → T0（Need (7,4,3) ≤ (7,4,3)）→ (7,5,3) → T2 → (10,5,5) → T4 → (10,5,7)，同樣成立。只要**找得到一個**就是 safe。

**問題 2：T1 請求 Request1 = (1, 0, 2)，能核准嗎？**

> 1. **Request1 ≤ Need1**？(1, 0, 2) ≤ (1, 2, 2) → **true**
> 2. **Request1 ≤ Available**？(1, 0, 2) ≤ (3, 3, 2) → **true**
> 3. **假裝配置**：
>
> | Thread | Allocation | Need | Available |
> |---|---|---|---|
> | T0 | 0 1 0 | 7 4 3 | **2 3 0** |
> | T1 | **3 0 2** | **0 2 0** | |
> | T2 | 3 0 2 | 6 0 0 | |
> | T3 | 2 1 1 | 0 1 1 | |
> | T4 | 0 0 2 | 4 3 1 | |
>
> 4. **執行 safety algorithm**，Work 從 (2, 3, 0) 開始：
>
> | 步驟 | 選中 | Need | ≤ Work？ | 新 Work |
> |---|---|---|---|---|
> | 1 | T1 | (0, 2, 0) | ≤ (2, 3, 0) ✔ | (2, 3, 0) + (3, 0, 2) = **(5, 3, 2)** |
> | 2 | T3 | (0, 1, 1) | ≤ (5, 3, 2) ✔ | + (2, 1, 1) = **(7, 4, 3)** |
> | 3 | T4 | (4, 3, 1) | ≤ (7, 4, 3) ✔ | + (0, 0, 2) = **(7, 4, 5)** |
> | 4 | T0 | (7, 4, 3) | ≤ (7, 4, 5) ✔ | + (0, 1, 0) = **(7, 5, 5)** |
> | 5 | T2 | (6, 0, 0) | ≤ (7, 5, 5) ✔ | + (3, 0, 2) = **(10, 5, 7)** |
>
> ⟨T1, T3, T4, T0, T2⟩ 是 safe sequence → **safe，核准請求**。

**問題 3：（在問題 2 核准後）T4 請求 Request4 = (3, 3, 0)，能核准嗎？**

> - Request4 ≤ Need4？(3, 3, 0) ≤ (4, 3, 1) → true
> - Request4 ≤ Available？(3, 3, 0) ≤ (2, 3, 0) → **false**（A：3 > 2）
>
> → **不能核准**：資源目前不夠，**T4 必須等待**。（根本不用跑 safety algorithm。）

**問題 4：（在問題 2 核准後）T0 請求 Request0 = (0, 2, 0)，能核准嗎？**

> 1. Request0 ≤ Need0？(0, 2, 0) ≤ (7, 4, 3) → true
> 2. Request0 ≤ Available？(0, 2, 0) ≤ (2, 3, 0) → true
> 3. 假裝配置：
>
> | Thread | Allocation | Need | Available |
> |---|---|---|---|
> | T0 | **0 3 0** | **7 2 3** | **2 1 0** |
> | T1 | 3 0 2 | 0 2 0 | |
> | T2 | 3 0 2 | 6 0 0 | |
> | T3 | 2 1 1 | 0 1 1 | |
> | T4 | 0 0 2 | 4 3 1 | |
>
> 4. 執行 safety algorithm，Work = (2, 1, 0)：
>
> | Thread | Need | ≤ (2, 1, 0)？ | 原因 |
> |---|---|---|---|
> | T0 | (7, 2, 3) | ✘ | A、B、C 都不夠 |
> | T1 | (0, 2, 0) | ✘ | B：2 > 1 |
> | T2 | (6, 0, 0) | ✘ | A：6 > 2 |
> | T3 | (0, 1, 1) | ✘ | C：1 > 0 |
> | T4 | (4, 3, 1) | ✘ | 都不夠 |
>
> **一個都找不到** → **unsafe** → **不能核准**，T0 必須等待，並把配置狀態還原。
> （即使 (0, 2, 0) 目前其實是「拿得出來」的。）
>
> ⚠️ 投影片第 56 頁右側留著一欄從 Avail (2, 1, 0) 開始累加 T1、T3、T4、T0 的計算，但 T1 的 Need (0, 2, 0) 並**不 ≤** (2, 1, 0)，那欄看起來是從前一頁沿用的，**不要照著算**。投影片程式的執行範例（第 77 頁）也顯示 Process 0–4 全部 "not done"、結果為 **Unsafe**，與上面一致。

> 📌 投影片的程式執行範例（第 74–78 頁，Homework #2 選項二的參考輸出）依序示範了：
> 1. 初始狀態 → safety algorithm 找到 `<1,3,0,2,4>` → Safe
> 2. Process 1 請求 (1, 0, 2) → 假裝核准 → Safe → **Grant**
> 3. Process 4 請求 (3, 3, 0) → `Request > Available! Process must wait!`
> 4. Process 0 請求 (0, 2, 0) → 假裝核准 → 全部 not done → **Unsafe, Do not grant! Restore the allocation state!**

---

## 8.7 死結偵測 (Deadlock Detection)

如果系統**既不做 deadlock prevention、也不做 deadlock avoidance**，允許系統進入死結狀態，那系統必須提供：

| 需要 | 說明 |
|---|---|
| **Detection algorithm（偵測演算法）** | 檢查系統是否發生死結 |
| **Recovery scheme（復原機制）** | 從死結中復原 |

**成本（overhead）**：
- 維護必要資訊、執行偵測演算法的**執行期成本（run-time costs）**
- 從死結復原時**本身就會造成的損失（potential losses inherent in recovering）**

**兩種偵測演算法**：

| 系統類型 | 演算法 |
|---|---|
| 每種資源**只有一個 instance** | **Wait-for graph** |
| 每種資源有**多個 instance** | 類似 Banker's Algorithm 的演算法 |

### 8.7.1 Single Instance for Each Resource Type

- 使用 resource-allocation graph 的一種**變形（variant）**：**wait-for graph（等待圖）**。
- 系統要維護 wait-for graph：
  - **節點只有 thread**（把資源節點拿掉）。
  - **Ti → Tj**：Ti 在**等** Tj（Ti 請求的資源被 Tj 持有）。
- **定期（periodically）** 執行演算法，在圖中**搜尋 cycle**：
  - **有 cycle ⇒ 有 deadlock**

**從 resource-allocation graph 轉成 wait-for graph**：

```
 若 RAG 中有  Ti ──請求──► Rq ──分配──► Tj
 則 wait-for graph 中有  Ti ──────► Tj
```

> 📌 投影片例子：
>
> **Resource-allocation graph** 的邊：
>
> | 請求（Ti → R） | 分配（R → Tj） | 轉成 wait-for 邊 |
> |---|---|---|
> | T1 → R1 | R1 → T2 | **T1 → T2** |
> | T2 → R3 | R3 → T5 | **T2 → T5** |
> | T2 → R4 | R4 → T3 | **T2 → T3** |
> | T2 → R5 | R5 → T4 | **T2 → T4** |
> | T3 → R5 | R5 → T4 | **T3 → T4** |
> | T4 → R2 | R2 → T1 | **T4 → T1** |
>
> **Wait-for graph**：
> ```
>                 T5
>                 ▲
>                 │
>   T1 ──────────► T2 ──────────► T3
>   ▲              │              │
>   │              ▼              │
>   └──────────── T4 ◄────────────┘
> ```
> 有 cycle：**T1 → T2 → T4 → T1**，以及 **T1 → T2 → T3 → T4 → T1** → **有死結**（T1、T2、T3、T4）。
>
> 💡 補充：T5 不在任何 cycle 上，沒有在等任何人，它可以正常做完。

> 💡 補充：搜尋 cycle 的演算法需要 **O(n²)** 次操作（n 是圖中的節點數），這也是為什麼不會每時每刻都執行。

### 8.7.2 Several Instances of a Resource Type

**使用的資料結構**：

| 名稱 | 大小 | 意義 |
|---|---|---|
| **Available** | 長度 m 的向量 | 每種資源**可用**的數量 |
| **Allocation** | n × m 矩陣 | 每個 thread **目前已分配**的各種資源數量 |
| **Request** | n × m 矩陣 | 每個 thread **目前的請求**。`Request[i][j] == k` 表示 Ti 正在**再要求** k 個 Rj |

**偵測演算法（Detection Algorithm）**：

```
1. Work = Available
   對每個 i = 0, 1, …, n−1：
     如果 Allocation_i ≠ 0 → Finish[i] = false
     否則                  → Finish[i] = true    ← 沒拿資源的不可能卡住別人

2. 找一個 i，同時滿足：
     a. Finish[i] == false
     b. Request_i ≤ Work
   找不到就跳到步驟 4

3. Work = Work + Allocation_i
   Finish[i] = true
   回到步驟 2

4. 如果有某個 i 的 Finish[i] == false → 系統處於 deadlocked state
   而且所有 Finish[i] == false 的 Ti 都是 deadlocked
```

**偵測演算法 vs. Safety algorithm 的差別** ⭐（💡 補充整理）：

| | **Safety algorithm（avoidance）** | **Detection algorithm** |
|---|---|---|
| 比較的對象 | **Need**（未來最多還可能要的） | **Request**（**目前**真的在要的） |
| Finish 初始值 | 全部 false | Allocation 為 0 的直接設為 true |
| 結論 | 有 false → **unsafe**（可能死結） | 有 false → **已經死結**，並指出是哪些 thread |
| 態度 | 保守：假設每個 thread 都會要到最大量 | 樂觀：假設拿到目前請求的就能做完 |

> 💡 補充：偵測演算法「樂觀」的意思是——如果 Ti 目前的請求能被滿足，就假設它會做完並歸還資源。如果之後它又提出新請求而卡住，下次執行偵測演算法時就會被抓到。

#### 偵測演算法例子

> 📌 投影片例子 1：**5 個 thread T0–T4；3 種資源 A（7 個）、B（2 個）、C（6 個）**
>
> | Thread | Allocation (A B C) | Request (A B C) |
> |---|---|---|
> | T0 | 0 1 0 | 0 0 0 |
> | T1 | 2 0 0 | 2 0 2 |
> | T2 | 3 0 3 | 0 0 0 |
> | T3 | 2 1 1 | 1 0 0 |
> | T4 | 0 0 2 | 0 0 2 |
>
> **Available = (0, 0, 0)**
>
> | 步驟 | 選中 | Request | ≤ Work？ | 新 Work |
> |---|---|---|---|---|
> | 1 | T0 | (0, 0, 0) | ≤ (0, 0, 0) ✔ | + (0, 1, 0) = **(0, 1, 0)** |
> | 2 | T2 | (0, 0, 0) | ✔ | + (3, 0, 3) = **(3, 1, 3)** |
> | 3 | T3 | (1, 0, 0) | ≤ (3, 1, 3) ✔ | + (2, 1, 1) = **(5, 2, 4)** |
> | 4 | T1 | (2, 0, 2) | ≤ (5, 2, 4) ✔ | + (2, 0, 0) = **(7, 2, 4)** |
> | 5 | T4 | (0, 0, 2) | ≤ (7, 2, 4) ✔ | + (0, 0, 2) = **(7, 2, 6)** |
>
> 序列 **⟨T0, T2, T3, T1, T4⟩** 使所有 Finish[i] = true → **不是死結狀態**。
> （最後 Work = (7, 2, 6) 等於系統總資源 ✔）
>
> 📌 投影片第 64 頁另有一個變形版本：C 共 **13** 個、T3 的 Allocation 為 (2, 1, **8**)。計算過程為 (0,0,0) → T0 → (0,1,0) → T2 → (3,1,3) → T3 → (5,2,11) → T1 → (7,2,11) → T4 → (7,2,13)，同樣得到 ⟨T0, T2, T3, T1, T4⟩，不是死結。

> 📌 投影片例子 2：**接續例子 1，T2 再多請求 1 個 C**——Request2 由 (0, 0, 0) 變成 **(0, 0, 1)**
>
> | Thread | Allocation | Request |
> |---|---|---|
> | T0 | 0 1 0 | 0 0 0 |
> | T1 | 2 0 0 | 2 0 2 |
> | T2 | 3 0 3 | 0 0 **1** |
> | T3 | 2 1 1 | 1 0 0 |
> | T4 | 0 0 2 | 0 0 2 |
>
> Available = (0, 0, 0)
>
> | 步驟 | 結果 |
> |---|---|
> | 1 | T0 的 Request (0, 0, 0) ≤ (0, 0, 0) ✔ → Work = **(0, 1, 0)** |
> | 2 | T1 要 (2, 0, 2) ✘、T2 要 (0, 0, 1) ✘、T3 要 (1, 0, 0) ✘、T4 要 (0, 0, 2) ✘ → **找不到** |
>
> 回收 T0 的資源後只有 (0, 1, 0)，**沒有其他 thread 的請求能被滿足**。
> → **Deadlock exists**，由 **T1、T2、T3、T4** 組成。

### 8.7.3 Detection-Algorithm Usage（偵測演算法的使用時機）

- 配置給死結 thread 的資源，在死結被打破之前都會**閒置**；而且捲入死結循環的 thread **數量可能越來越多**。
- **何時、多久**執行一次偵測演算法，取決於：
  1. **死結多常發生？**
  2. **死結發生時會影響多少 thread？**

| 執行頻率 | 優點 | 缺點 |
|---|---|---|
| **每次有請求時都執行** | 能立刻找出是**哪個請求**造成死結 | **成本很高**（computation overhead 大） |
| **每次請求無法立即被滿足時執行** | 只在可能出問題時才檢查 | 仍然相當頻繁 |
| **較低頻率的間隔**（例如每小時一次，或 CPU 使用率低於 40% 時） | 成本低 | 圖中可能有很多 cycle，**很難判斷是哪個 thread「造成」死結** |

（💡 補充：表中的優缺點與「每小時一次、CPU 使用率低於 40%」取自課本說明；投影片只列出三種執行時機。）

---

## 8.8 從死結中復原 (Recovery from Deadlock)

兩大類方法：

| 方法 | 內容 |
|---|---|
| **Process and Thread Termination（終止 process／thread）** | 終止所有死結的 process；或一次終止一個，直到死結循環消除 |
| **Resource Preemption（搶占資源）** | 選擇 victim、rollback、處理 starvation |

### 8.8.1 Process and Thread Termination（終止行程與執行緒）

**兩種方法**：

| 方法 | 優點 | 缺點（💡 補充） |
|---|---|---|
| **Abort all deadlocked processes**（終止**所有**死結的 process） | 一定能打破死結，簡單 | **代價很大**：這些 process 可能已經算了很久，結果全部作廢，之後要重算 |
| **Abort one process at a time until the deadlock cycle is eliminated**（**一次終止一個**，直到死結循環消除） | 損失較小 | **overhead 大**：每終止一個都要**重新執行偵測演算法**，確認死結是否解除 |

**一次終止一個時，要依什麼順序選？** 考慮因素：
1. **Priority**：process 的優先權
2. process **已經計算了多久**，以及**還要多久**才能完成
3. process **已經使用**了多少資源（以及是哪些類型，例如是否容易搶占）
4. process **還需要**多少資源才能完成
5. 總共**需要終止多少個** process

> 💡 補充：原則是選擇「**終止代價最小**」的 process。例如優先選優先權低、剛開始執行、佔用資源多（釋放後最能解套）的那個。
> 💡 補充：終止 process 也有風險——如果它正在更新檔案或印表，中途終止可能讓資料處於不一致的狀態。

### 8.8.2 Resource Preemption（資源搶占）

- 從某些 process 身上**搶占**資源，交給其他 process，直到**死結循環被打破**。
  - 從**部分死結的 process** 搶占資源。

**需要處理的 3 個問題**：

| 問題 | 中文 | 處理方式 |
|---|---|---|
| **Selecting a victim** | 選擇犧牲者 | 搶誰的、搶哪些資源？目標是**最小化成本（minimize cost）**——成本可以包括持有的資源數、已經執行的時間等 |
| **Rollback** | 回復 | 被搶了資源的 process 無法正常繼續，必須**回復到某個安全狀態，再從那裡重新開始**。最簡單的是 **total rollback**（整個終止重來）（💡 補充） |
| **Starvation** | 飢餓 | 同一個 process 可能**每次都被選為 victim**。解法：把**被 rollback 的次數**納入成本因素（cost factor） |

> 💡 補充例子：回到過橋的例子——
> - Termination 像把卡住的車直接**拖走**。
> - Resource preemption ＋ rollback 像請其中一台車**倒車**到橋頭，讓出路面後再重新上橋。
> - Starvation 就是每次都叫同一台車倒車；解法是記錄「這台車已經倒過幾次」，倒過越多次越不該再選它。

---

## 重點整理

### 處理死結的方法總覽

| 方法 | 核心想法 | 需要事先資訊？ | 成本／缺點 | 常見程度 |
|---|---|---|---|---|
| **Ignore（忽略）** | 假裝不會發生 | 否 | 真的發生時要靠人處理 | **最常見**（Linux、Windows） |
| **Prevention（預防）** | 讓 4 個必要條件中至少一個**不成立** | 否 | 裝置使用率低、throughput 降低 | — |
| **Avoidance（避免）** | 每次配置前檢查是否仍 **safe** | **是**（每個 thread 的最大需求） | 資源可用也可能要等；需事先知道需求 | — |
| **Detection ＋ Recovery（偵測與復原）** | 允許發生，定期檢查，發生後處理 | 否 | 偵測的執行期成本＋復原的損失 | — |

### 四個必要條件與對應的預防方法

| 條件 | 預防方法 | 可行性 |
|---|---|---|
| **Mutual exclusion** | 讓資源可共享 | **通常不可行**：有些資源本質上不可共享 |
| **Hold and wait** | 執行前一次拿齊（P1）；或沒資源時才能要（P2） | 可行，但使用率低、可能 starvation |
| **No preemption** | 要不到就交出全部（P1）；搶正在等待者的資源（P2） | 只適用於狀態易保存的資源（CPU registers、DB transactions） |
| **Circular wait** | 資源編號，依遞增順序請求 | **最實用**（lock ordering） |

### 四種演算法對照

| 用途 | 單一 instance | 多個 instance |
|---|---|---|
| **Avoidance（避免）** | Resource-allocation graph algorithm（claim edge ＋ 檢查 cycle） | **Banker's algorithm**（safety ＋ resource-request） |
| **Detection（偵測）** | **Wait-for graph**（找 cycle） | Detection algorithm（類似 banker's，用 Request） |

### Banker's Algorithm 解題 SOP（💡 補充整理）

1. 算出 **Need = Max − Allocation**。
2. 如果題目沒給 Available，用 **總資源 − 所有 Allocation 的總和**算出來。
3. **判斷 safe**：Work = Available，反覆找 **Need ≤ Work** 的 thread，Work += 它的 Allocation，記下順序。全部找得到 → safe，並寫出 safe sequence。
4. **判斷能否核准請求**：
   - Request ≤ Need？否 → error
   - Request ≤ Available？否 → 必須等待
   - 假裝配置（Available −、Allocation +、Need −），再做一次步驟 3
   - safe → 核准；unsafe → 等待並還原
5. 檢查：最後的 Work 應該等於系統的總資源。

### 投影片範例答案速查

| 範例 | 結果 |
|---|---|
| 12 台 tape drive，T0/T1/T2 = 10/5、4/2、9/2 | safe，⟨T1, T0, T2⟩ |
| 同上，T2 再要 1 台 | unsafe |
| Banker 初始狀態（Available 3 3 2） | safe，⟨T1, T3, T4, T2, T0⟩ |
| T1 請求 (1, 0, 2) | safe，⟨T1, T3, T4, T0, T2⟩ → **核准** |
| 接著 T4 請求 (3, 3, 0) | Request > Available → **等待** |
| 接著 T0 請求 (0, 2, 0) | unsafe → **等待** |
| 偵測例子 1（Available 0 0 0） | 沒有死結，⟨T0, T2, T3, T1, T4⟩ |
| 偵測例子 2（T2 多要 1 個 C） | **死結**：T1、T2、T3、T4 |

---

## 容易混淆的觀念對照

| 比較 | A | B |
|---|---|---|
| **Deadlock vs. Starvation** | 一組 thread **互相等待**，全部永遠無法前進 | 某個 thread **一直輪不到**，但系統其他部分仍在運作 |
| **Deadlock prevention vs. avoidance** | 從根本讓某個必要條件**不可能成立**；不需事先資訊 | 每次配置前**動態檢查**是否 safe；需要最大需求等事先資訊 |
| **Avoidance vs. Detection** | 事前：**不讓**系統進入 unsafe | 事後：**允許**死結發生，再找出並處理 |
| **Safe vs. Unsafe vs. Deadlock** | 保證沒有死結 | 可能死結 | 已經死結（unsafe 的子集合） |
| **Cycle vs. Deadlock** | 單一 instance 時 cycle ⇔ deadlock | 多個 instance 時 cycle 只是「可能」 |
| **Request edge vs. Assignment edge vs. Claim edge** | Ti → Rj，正在等 | Rj → Ti，已分配 | Ti ┄► Rj（虛線），將來可能要 |
| **Resource-allocation graph vs. Wait-for graph** | 有 thread 與 resource 兩種節點 | 只有 thread 節點；Ti → Tj 表示 Ti 在等 Tj |
| **RAG algorithm vs. Banker's algorithm** | Avoidance，單一 instance | Avoidance，多個 instance |
| **Wait-for graph vs. Detection algorithm** | Detection，單一 instance | Detection，多個 instance |
| **Need vs. Request** | Max − Allocation，**未來最多**還要的（avoidance 用） | **目前**實際在要的（detection 用） |
| **Max vs. Allocation** | 宣告的最大需求 | 目前已拿到的 |
| **Request > Need vs. Request > Available** | **錯誤**：超過宣告的最大需求 | **等待**：資源目前不夠 |
| **Hold-and-wait Protocol 1 vs. 2** | 執行前一次拿齊所有資源 | 沒有資源時才能請求；要新的先放掉舊的 |
| **No-preemption Protocol 1 vs. 2** | 自己請求不到 → 交出持有的全部 | 去搶**正在等待**的 thread 的資源 |
| **Circular-wait Protocol 1 vs. 2** | 只能依編號遞增請求 | 要 Rj 前先釋放所有 F(Ri) ≥ F(Rj) 的 Ri |
| **Abort all vs. Abort one at a time** | 代價大但簡單 | 損失小，但每次都要重跑偵測 |
| **Termination vs. Resource preemption** | 直接終止 process | 搶資源＋rollback，要處理 victim 選擇與 starvation |

---

## 作業（Homework）

### Homework #1

**Textbook（10th edition）：Ex 8.28**

| 組別 | 繳交方式 |
|---|---|
| **對照組** | 繳交**手寫**作業 |
| **實驗組** | 使用 **OS Simulator**（<http://ossim.im.ncnu.edu.tw/>），用**自己的帳號**，繳交**執行結果螢幕截圖** |

> 📌 投影片附的 **Ex. 8.28** 題目：
> Consider the following snapshot of a system:
>
> | Thread | Allocation (A B C D) | Max (A B C D) |
> |---|---|---|
> | T0 | 3 1 4 1 | 6 4 7 3 |
> | T1 | 2 1 0 2 | 4 2 3 2 |
> | T2 | 2 4 1 3 | 2 5 3 3 |
> | T3 | 4 1 1 0 | 6 3 3 2 |
> | T4 | 2 2 2 1 | 5 6 7 5 |
>
> **Available = (2, 2, 2, 4)**
>
> Answer the following questions using the banker's algorithm:
> - a. Illustrate that the system is in a safe state by demonstrating an order in which the threads may complete.
> - b. If a request from thread T4 arrives for (2, 2, 2, 4), can the request be granted immediately?
> - c. If a request from thread T2 arrives for (0, 1, 1, 0), can the request be granted immediately?
> - d. If a request from thread T3 arrives for (2, 2, 1, 2), can the request be granted immediately?

→ 照本筆記的 [Banker's Algorithm 解題 SOP](#bankers-algorithm-解題-sop補充整理) 與 [8.6.3 的完整例子](#完整例子-) 一步一步做。這題有 **4 種資源（A B C D）**，比投影片例子多一種。

### Homework #2（2 擇 1）

**選項一：製作 deadlock 影片（100%）**
1. 至多 **2 人一組**。
2. 上傳至 **moodle**。
3. 請說明影片內容（例如符合 deadlock 的哪些**必要條件**，一共 **4 個條件**）。

→ 可參考 [8.3.1 的四個必要條件](#831-necessary-conditions必要條件) 與投影片附的學長姐作品影片。

**選項二：Implementation of the banker's algorithm（100%）**
- 10th edition 測試資料二選一：
  - 使用 **ex. 8.28**（textbook p. 931）作為輸入測試資料
  - 或使用 **ppt #51–#56**（textbook pp. 336–337）作為輸入測試資料
- **任何程式語言**皆可，請上傳**程式原始檔**、**執行結果截圖**至 moodle。
- 需要向 **OS 助教 demo**。
- 註：**抄襲者、被抄襲者一律以 0 分計算**。
- 註：原始程式的執行結果與「執行結果截圖」**不符者，一律以 0 分計算**。

> 📌 投影片第 74–78 頁附了**執行結果範例**（使用 ppt #51–#56 的資料），流程是：
> 1. 印出初始狀態（Available、Allocation、Max、Need）
> 2. 執行 safety algorithm，印出 safe sequence
> 3. 讓使用者輸入「哪個 process」與「各種資源的請求量」
> 4. 假裝配置 → 印出新狀態 → 執行 safety algorithm
> 5. Safe 就核准；Request > Available 就要求等待；Unsafe 就不核准並還原
> 6. 詢問是否繼續（`Yes=y or Y`）
>
> 本筆記 [8.6.3 的完整例子](#完整例子-) 的問題 1–4 正好對應這份輸出，可以拿來核對自己的程式結果。

---

## 英中詞彙總表

### A–D

| English | 中文 | 補充 |
|---|---|---|
| a priori | 事前的 | 事先得知 |
| abort | 終止 | |
| acquire | 取得 | 取得鎖或資源 |
| allocation | 配置 | Banker's 的 Allocation 矩陣 |
| assignment edge | 分配邊 | Rj → Ti |
| available | 可用的 | Banker's 的 Available 向量 |
| avoidance | 避免 | 消極策略 |
| banker's algorithm | 銀行家演算法 | 多 instance 的 avoidance |
| characterization | 特徵 | |
| circular wait | 循環等待 | 必要條件之一 |
| claim edge | 宣告邊 | 虛線，Ti ┄► Rj |
| cost factor | 成本因素 | |
| cycle | 環 | |
| cycle-detection algorithm | 環偵測演算法 | |
| database transaction | 資料庫交易 | 狀態容易保存，可搶占 |
| deadlock | 死結 | |
| deadlock avoidance | 死結避免 | |
| deadlock detection | 死結偵測 | |
| deadlock prevention | 死結預防 | |
| deadlock recovery | 死結復原 | |
| deadlocked state | 死結狀態 | |
| detection algorithm | 偵測演算法 | |

### E–N

| English | 中文 | 補充 |
|---|---|---|
| edge | 邊 | |
| enumeration | 列舉 | |
| finish | 完成 | 演算法中的 Finish 陣列 |
| hold and wait | 持有並等待 | 必要條件之一 |
| imply | 暗指、意味 | deadlock implies cycle |
| impose | 強加於 | |
| implicitly | 隱含地 | 投影片譯「暗示地」 |
| instance | 實例 | RAG 方框中的點 |
| intrinsically | 本質上 | |
| lock ordering | 取鎖順序 | |
| max demand | 最大需求 | Banker's 的 Max 矩陣 |
| mutex lock | 互斥鎖 | |
| mutual exclusion | 互斥 | 必要條件之一 |
| necessary condition | 必要條件 | 共 4 個 |
| need | 需求 | Max − Allocation |
| no preemption | 不可搶占 | 必要條件之一 |
| nonsharable | 不可共享的 | |

### O–R

| English | 中文 | 補充 |
|---|---|---|
| ostrich algorithm | 鴕鳥演算法 | 忽略死結（補充名詞） |
| overhead | 額外負擔 | |
| periodically | 定期地 | |
| preempt / preemption | 搶占 | |
| pretend | 假裝 | 先假裝配置再檢查 |
| protocol | 協定 | |
| read-only file | 唯讀檔案 | 可共享資源的例子 |
| recovery scheme | 復原機制 | |
| release | 釋放 | |
| request | 請求 | 也是 detection 的 Request 矩陣 |
| request edge | 請求邊 | Ti → Rj |
| resource | 資源 | |
| resource numbering | 資源編號 | F: R → N |
| resource preemption | 資源搶占 | |
| resource type | 資源類型 | 也稱 class |
| resource-allocation graph (RAG) | 資源配置圖 | |
| resource-allocation state | 資源配置狀態 | |
| resource-request algorithm | 資源請求演算法 | |
| restore | 還原 | unsafe 時還原配置狀態 |
| restrain | 限制 | |
| rollback | 回復 | |

### S–Z

| English | 中文 | 補充 |
|---|---|---|
| safe sequence | 安全序列 | |
| safe state | 安全狀態 | |
| safety algorithm | 安全演算法 | |
| semaphore | 號誌 | |
| sharable | 可共享的 | |
| snapshot | 快照 | 某時刻的系統狀態 |
| starvation | 飢餓 | |
| system model | 系統模型 | |
| termination | 終止 | |
| throughput | 產能 | |
| total ordering | 全序 | 所有資源排出先後順序 |
| unsafe state | 不安全狀態 | |
| utilization | 使用率 | |
| variant | 變形 | |
| victim | 犧牲者 | |
| wait-for graph | 等待圖 | 單 instance 的 detection |
| work | 工作向量 | 演算法中的 Work |
