# 作業系統 Chapter 5：CPU Scheduling（CPU 排程）筆記

> **教材**：*Operating System Concepts*, 10th ed.（Silberschatz, Galvin, Gagne）
> **投影片**：Mei-Ling Chiang 教授，暨南大學資管系（2026/10/20，共 123 頁）
> **筆記說明**：
> - 英文專有名詞第一次出現時以「**English（中文）**」標示，文末有完整詞彙表。
> - 標示「📌 投影片例子」者出自投影片；標示「💡 補充例子」者是為了幫助理解另外加的比喻或說明，不在投影片內。
> - 標示「⚠️」者是投影片中疑似筆誤或需要留意的地方。
> - 時間單位若沒有特別說明，都是**毫秒（ms）**。

---

## 目錄

- [0. 本章目標](#0-本章目標)
- [5.1 基本概念](#51-基本概念-basic-concepts)
- [5.2 排程準則](#52-排程準則-scheduling-criteria)
- [5.3 排程演算法](#53-排程演算法-scheduling-algorithms)
- [5.4 執行緒排程](#54-執行緒排程-thread-scheduling)
- [5.5 多處理器排程](#55-多處理器排程-multiple-processor-scheduling)
- [5.6 即時 CPU 排程](#56-即時-cpu-排程-real-time-cpu-scheduling)
- [5.7 作業系統實例](#57-作業系統實例-operating-system-examples)
- [5.8 演算法評估](#58-演算法評估-algorithm-evaluation)
- [重點整理](#重點整理)
- [容易混淆的觀念對照](#容易混淆的觀念對照)
- [作業（Homework）](#作業homework)
- [英中詞彙總表](#英中詞彙總表)

---

## 0. 本章目標

- 介紹 **process scheduling（行程排程）**，它是 multiprogrammed OS 的基礎。
- 描述各種 **CPU-scheduling algorithm（CPU 排程演算法）**。
- 討論為特定系統挑選排程演算法時的**評估準則（evaluation criteria）**。
- 檢視幾個 OS 實際使用的排程演算法。

> 💡 補充：Ch3 講過 process 在 ready queue 裡等 CPU，也講過 CPU scheduler 會「從 ready queue 選一個」。這一章回答的就是：**要選哪一個？怎麼選才好？**

---

## 5.1 基本概念 (Basic Concepts)

- Multiprogramming 的目的是取得**最大的 CPU 使用率**。
- 幾乎所有電腦資源在使用前都要經過排程。
- **CPU scheduling 是 OS 設計的核心**。

### 5.1.1 CPU–I/O Burst Cycle（CPU–I/O 爆發週期）

Process 的執行是由 **CPU execution（CPU 執行）** 與 **I/O wait（I/O 等待）** 交替組成的**循環**（Fig. 5.1）。

```
   load store
   add store          ┐
   read from file     ┘ CPU burst
   ── wait for I/O ──   I/O burst
   store increment    ┐
   index              │ CPU burst
   write to file      ┘
   ── wait for I/O ──   I/O burst
   load store
   add store          ┐ CPU burst
   read from file     ┘
   ── wait for I/O ──   I/O burst
          ⋮
```

| 名詞 | 中文 | 說明 |
|---|---|---|
| **CPU burst** | CPU 爆發 | 一段連續使用 CPU 的時間 |
| **I/O burst** | I/O 爆發 | 一段等待 I/O 完成的時間 |

**CPU burst 的分布（Fig. 5.2）**：橫軸是 burst 長度（毫秒），縱軸是出現頻率。曲線顯示**大量的短 burst**、**少量的長 burst**。

| 類型 | 中文 | 特徵 |
|---|---|---|
| **I/O-bound program** | I/O 密集型程式 | 有**很多短的** CPU burst |
| **CPU-bound program** | CPU 密集型程式 | 可能有**少數幾個長的** CPU burst |

> 💡 補充例子：文書編輯器是 I/O-bound——大部分時間在等你按鍵，按一下只需要極短的 CPU 時間處理。影片轉檔是 CPU-bound——一直在算，很少等 I/O。

### 5.1.2 CPU Scheduler（CPU 排程器）

- 每當 CPU **閒置（idle）**，OS 必須從 **ready queue** 中**選出**一個 process 來執行。
- 這個選擇工作由 **CPU scheduler** 執行。
- Ready queue 可以用 **FIFO queue**、**priority queue（優先權佇列）**、**tree（樹）** 或 **unordered linked list（無序鏈結串列）** 實作。

> 💡 補充：Ready queue 不一定是「先進先出」的佇列。它裡面放的是各 process 的 PCB，用什麼資料結構、怎麼挑，取決於排程演算法。

### 5.1.3 Preemptive & Nonpreemptive Scheduling（可搶占與不可搶占排程）⭐

CPU 排程決策可能在 process 發生下列情況時進行：

| 編號 | 狀態轉換 | 對應 Ch3 狀態圖 |
|---|---|---|
| **(0)** | new → ready | admitted |
| **(1)** | **running → waiting** | I/O or event wait |
| **(2)** | running → ready | interrupt |
| **(3)** | waiting → ready | I/O or event completion |
| **(4)** | **terminates** | exit |

- 如果只在 **(1) 與 (4)** 這兩種情況才排程，稱為 **nonpreemptive（不可搶占）** 或 **cooperative（合作式）**。
- 其他情況也會排程，就是 **preemptive（可搶占）**。

> 📌 投影片例子：在 process state diagram 上標出排程點——preemptive 是 (0, 1, 2, 3, 4) 全部；nonpreemptive 只有圈起來的 (1) 和 (4)。
>
> 為什麼 (1) 和 (4) 一定要排程？因為這兩種情況下目前的 process **自己放棄了 CPU**（去等 I/O 或結束了），CPU 空出來，非選下一個不可。其他情況是「目前的 process 還能跑，要不要把它換掉」，只有 preemptive 才會這樣做。

#### Nonpreemptive Scheduling（不可搶占排程）

- CPU 一旦分配給某個 process，該 process 就一直持有 CPU，直到它**終止**或**切換到 waiting 狀態**才釋放。
- **幾乎所有現代 OS 都使用 preemptive 排程演算法**：Windows、macOS、Linux、UNIX…

#### Preemptive Scheduling（可搶占排程）的問題

| 問題 | 說明 | 處理方式 |
|---|---|---|
| **Race condition（競爭條件）** | 存取共享資料時，一個 process 改到一半被搶走，另一個 process 讀到不一致的資料 | 需要協調共享資料存取的機制（Chapter 6） |
| **影響 kernel 設計：在 kernel mode 被搶占** | Process 執行 system call 期間被搶占，kernel 資料結構可能不一致 | Kernel 在處理 system call 期間**不搶占**該 process，確保 kernel 結構一致 |
| **影響 kernel 設計：重要 OS 活動中發生中斷** | 被中斷影響的程式碼必須防止同時被使用 | 進入時**關閉中斷**（disable INTs），離開時**重新開啟**（reenable INTs） |

> 💡 補充例子（race condition）：兩個 process 共用一個計數器 `count = 5`。A 讀出 5，正要寫回 6 時被搶占；B 也讀出 5、寫回 6。A 回來後寫回 6。兩次加一，結果卻只多了 1。

### 5.1.4 Dispatcher（分派器）

**Dispatcher**：把 CPU core 的控制權交給 CPU scheduler 所選出之 process 的**模組**。

**Dispatcher 的工作**：
1. **Switching context**——切換內容（context switch）
2. **Switching to user mode**——切換到 user mode
3. 跳到 user program 中**適當的位置**，讓程式繼續執行

**Dispatch latency（分派延遲）**：dispatcher **停止一個 process 並開始執行另一個**所花的時間。

**Dispatcher 的角色（Fig. 5.3）**：

```
   P0 executing
        │                    ┐
   save state into PCB0      │
        │                    │ dispatch latency
   restore state from PCB1   │
        │                    ┘
   P1 executing
```

| | **CPU scheduler** | **Dispatcher** |
|---|---|---|
| 工作 | **決定**下一個執行誰 | **執行**這個決定（實際切換過去） |
| 💡 補充比喻 | 教練決定換誰上場 | 場邊工作人員把人換上去 |

Dispatcher 每次切換都會被呼叫，所以必須**越快越好**。

---

## 5.2 排程準則 (Scheduling Criteria)

用來比較 CPU 排程演算法的準則：

| 準則 | 中文 | 定義 | 目標 |
|---|---|---|---|
| **CPU utilization** | CPU 使用率 | CPU 有多忙。例如 40%（輕負載）到 90%（重負載） | **最大化** |
| **Throughput** | 產能 | **單位時間內完成的 process 數**。例如長 process 每小時 1 個；短交易每秒 10 個 | **最大化** |
| **Turnaround time** | 回復時間／周轉時間 | 執行一個 process 所需的時間，即從**送出（submission）到完成（completion）** 的間隔 | **最小化** |
| **Waiting time** | 等待時間 | process 在 **ready queue 中等待**的總時間 | **最小化** |
| **Response time** | 回應時間 | 從送出請求到**產生第一個回應**的時間。指的是「開始回應」所需的時間，而不是輸出完整回應的時間 | **最小化** |

**最佳化目標（Optimization Criteria）**：
- Max CPU utilization、Max throughput
- Min turnaround time、Min waiting time、Min response time
- Min 最大回應時間（minimize the max response time）
- Min 回應時間的變異數（minimize the variance in the response time）

**本章計算時的常用公式** ⭐（💡 補充整理）：

```
Turnaround time = Completion time − Arrival time
Waiting time    = Turnaround time − Burst time
                = Completion time − Arrival time − Burst time
```

> 📌 投影片也寫了同樣的公式：**Waiting time = completion time − arrival time − execution time**。
>
> 💡 補充：排程演算法其實**不影響**一個 process 執行或做 I/O 的時間長短，它只影響 process **在 ready queue 裡等多久**。所以本章最常用 **average waiting time（平均等待時間）** 來比較演算法。

---

## 5.3 排程演算法 (Scheduling Algorithms)

為了方便說明，本章的例子每個 process **只考慮一個 CPU burst**（單位毫秒），並以**平均等待時間**作為比較標準。

### 各演算法是否可搶占 ⭐

| 演算法 | Preemptive | Non-preemptive |
|---|---|---|
| **FCFS** | | ✓ |
| **SJF** | ✓（稱為 **SRTF**） | ✓ |
| **Priority** | ✓ | ✓ |
| **RR** | ✓ | |
| **Multilevel queue** | ✓ | |
| **Multilevel feedback queue** | ✓ | |

> ⚠️ 投影片第 18 頁的目錄把 5.3.3 寫成 Priority、5.3.4 寫成 Round-Robin，但後面內頁的標題（以及 10th edition 課本）是 **5.3.3 Round-Robin、5.3.4 Priority**。本筆記依內頁順序。

### 5.3.1 First-Come, First-Served（FCFS，先到先服務）

**誰先到 ready queue，誰就先用 CPU**。用 FIFO queue 就能實作。

> 📌 投影片例子 1：三個 process 依 P1、P2、P3 的順序到達（皆在時間 0）
>
> | Process | Burst Time |
> |---|---|
> | P1 | 24 |
> | P2 | 3 |
> | P3 | 3 |
>
> ```
> Gantt chart:
> |            P1            | P2 | P3 |
> 0                         24   27   30
> ```
> - Waiting time：P1 = 0，P2 = 24，P3 = 27
> - **平均等待時間 = (0 + 24 + 27) / 3 = 17**
>
> 📌 投影片例子 2：同樣三個 process，到達順序改成 P2、P3、P1
> ```
> Gantt chart:
> | P2 | P3 |            P1            |
> 0    3    6                         30
> ```
> - Waiting time：P1 = 6，P2 = 0，P3 = 3
> - **平均等待時間 = (6 + 0 + 3) / 3 = 3**——比上一種情況好很多

**FCFS 的特性**：
- 平均等待時間通常**不是最佳的**，而且當各 process 的 CPU burst 長度差異很大時，**變動也很大**。
- **Nonpreemptive**：對 time-sharing 系統來說很麻煩（一個 process 可能霸佔 CPU 很久）。

**Convoy effect（護送效應）** ⭐：
- 短的 process 排在長的 process 後面。
- 所有其他 process 都在等那個大 process 離開 CPU。
- 結果使 **CPU 與裝置的使用率都降低**。

> 📌 投影片例子：假設有一個 CPU-bound process 和許多 I/O-bound process。
> 1. CPU-bound process 佔著 CPU；I/O-bound process 都做完 I/O 回到 ready queue 排在它後面 → **I/O 裝置閒置**。
> 2. CPU-bound process 終於去做 I/O；I/O-bound process 很快地用完各自短短的 CPU burst，又都去排 I/O，排在 CPU-bound process 後面 → **CPU 閒置**。
> 3. 如此反覆，CPU 和裝置輪流閒著。
>
> 💡 補充例子：超市只有一個收銀台，前面的人推了滿滿一車，後面只買一瓶水的人全部卡住。

### 5.3.2 Shortest-Job-First（SJF，最短工作優先）

- 更精確的名稱是 **shortest-next-CPU-burst algorithm（最短下一次 CPU burst 演算法）**。
- 每個 process 都關聯到它**下一次 CPU burst 的長度**；選擇**下一次 CPU burst 最短**的 process。
  - 長度相同時，依 **FCFS** 順序。
- 看的是 process **下一次** CPU burst 的長度，**不是**它的總長度。

**兩種版本**：

| | **Nonpreemptive SJF** | **Preemptive SJF ＝ SRTF** |
|---|---|---|
| 規則 | CPU 一旦給了某個 process，在它完成這次 CPU burst 之前**不能被搶占** | 新到達的 process 的 CPU burst 長度**小於**目前執行中 process 的**剩餘時間**，就搶占 |
| 全名 | — | **Shortest-Remaining-Time-First（最短剩餘時間優先）** |

> 📌 投影片例子 1：Nonpreemptive SJF（全部在時間 0 到達）
>
> | Process | Burst Time |
> |---|---|
> | P1 | 6 |
> | P2 | 8 |
> | P3 | 7 |
> | P4 | 3 |
>
> ```
> | P4 |   P1   |    P3    |    P2    |
> 0    3        9         16         24
> ```
> 平均等待時間 = (3 + 16 + 9 + 0) / 4 = **7**
>
> （💡 補充：同樣這四個 process 用 FCFS 的話，平均等待時間是 (0 + 6 + 14 + 21) / 4 = 10.25。）

> 📌 投影片例子 2：Nonpreemptive SJF，**加入不同的到達時間**
>
> | Process | Arrival Time | Burst Time |
> |---|---|---|
> | P1 | 0 | 7 |
> | P2 | 2 | 4 |
> | P3 | 4 | 1 |
> | P4 | 5 | 4 |
>
> ```
> |      P1      | P3 |   P2   |   P4   |
> 0              7    8       12       16
> ```
> - 時間 0 只有 P1，先執行；nonpreemptive，所以 P1 跑完整個 burst（到 7）。
> - 時間 7：P2（4）、P3（1）、P4（4）都到了，選最短的 P3。
> - 時間 8：P2 與 P4 一樣長（4），依 FCFS 選先到的 P2。
>
> | Process | 等待時間 |
> |---|---|
> | P1 | 0 |
> | P2 | 8 − 2 = 6 |
> | P3 | 7 − 4 = 3 |
> | P4 | 12 − 5 = 7 |
>
> 平均等待時間 = (0 + 6 + 3 + 7) / 4 = **4**

> 📌 投影片例子 3：**SRTF**（preemptive SJF）
>
> | Process | Arrival Time | Burst Time |
> |---|---|---|
> | P1 | 0 | 8 |
> | P2 | 1 | 4 |
> | P3 | 2 | 9 |
> | P4 | 3 | 5 |
>
> ```
> | P1 |   P2   |    P4    |      P1      |       P3       |
> 0    1        5         10             17               26
> ```
> - 時間 1：P2 到達（4）< P1 剩下的 7 → **搶占**，換 P2。
> - 時間 5：P2 完成；剩 P1（7）、P3（9）、P4（5）→ 選 P4。
> - 時間 10：P4 完成；P1（7）< P3（9）→ 選 P1。
>
> 平均等待時間 = [(10 − 1) + (1 − 1) + (17 − 2) + (5 − 3)] / 4 = 26 / 4 = **6.5 ms**
>
> （P1 從時間 1 停到 10 才繼續，所以等了 10 − 1 = 9。）

> 📌 投影片例子 4：**Preemptive SJF**，與例子 2 同一組資料
>
> | Process | Arrival Time | Burst Time |
> |---|---|---|
> | P1 | 0 | 7 |
> | P2 | 2 | 4 |
> | P3 | 4 | 1 |
> | P4 | 5 | 4 |
>
> ```
> |  P1  |  P2  | P3 |  P2  |    P4    |      P1      |
> 0      2      4    5      7         11             16
> ```
> - 時間 2：P2 到達（4）< P1 剩下的 5 → 搶占。
> - 時間 4：P3 到達（1）< P2 剩下的 2 → 搶占。
> - 時間 5：P3 完成；P2 剩 2、P4 為 4、P1 剩 5 → P2。
> - 時間 7：P2 完成；P4（4）< P1（5）→ P4。
> - 時間 11：P4 完成 → P1 跑完。
>
> 用公式 **Waiting = Completion − Arrival − Burst**：
>
> | Process | Completion | Arrival | Burst | Waiting |
> |---|---|---|---|---|
> | P1 | 16 | 0 | 7 | **9** |
> | P2 | 7 | 2 | 4 | **1** |
> | P3 | 5 | 4 | 1 | **0** |
> | P4 | 11 | 5 | 4 | **2** |
>
> 平均等待時間 = (9 + 1 + 0 + 2) / 4 = **3**（比 nonpreemptive 版本的 4 更好）

**SJF 的優缺點**：
- SJF **可證明是最佳的（provably optimal）**：對一組給定的 process，它的**平均等待時間最小**。
- 常用於 **long-term scheduling（長期排程）**。
- **無法直接用在 short-term scheduling（短期排程，即 CPU 排程）**——因為無法事先知道下一次 CPU burst 有多長。
- 只能**預測（predict）** 長度：選**預測的 CPU burst 最短**的 process，來近似 SJF。

> 💡 補充：為什麼 SJF 最佳？把短的工作移到長的前面，短工作少等的時間比長工作多等的時間還多，所以總等待時間一定下降。

#### 預測下一次 CPU burst 的長度 ⭐

用過去 CPU burst 實際長度的 **exponential average（指數平均）** 來預測：

```
τ(n+1) = α · t(n) + (1 − α) · τ(n)        0 ≤ α ≤ 1
```

| 符號 | 意義 |
|---|---|
| **t(n)** | 第 n 次 CPU burst 的**實際**長度 |
| **τ(n)** | 對第 n 次 CPU burst 的**預測**值（代表過去的歷史） |
| **τ(n+1)** | 對下一次 CPU burst 的**預測**值 |
| **α** | 權重：決定最近一次實際值與過去歷史各佔多少 |

> 📌 投影片例子：α = 1/2，τ0 = 10
>
> | i | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
> |---|---|---|---|---|---|---|---|---|
> | 實際 CPU burst t(i) | 6 | 4 | 6 | 4 | 13 | 13 | 13 | … |
> | 預測 τ(i) | 10 | 8 | 6 | 6 | 5 | 9 | 11 | 12 |
>
> 計算過程：
> - τ1 = ½ × 6 + ½ × 10 = **8**
> - τ2 = ½ × 4 + ½ × 8 = **6**
> - τ3 = ½ × 6 + ½ × 6 = **6**
> - τ4 = ½ × 4 + ½ × 6 = **5**
> - τ5 = ½ × 13 + ½ × 5 = **9**
> - τ6 = ½ × 13 + ½ × 9 = **11**
> - τ7 = ½ × 13 + ½ × 11 = **12**
>
> 實際值從 4 跳到 13 之後，預測值逐步追上去（5 → 9 → 11 → 12）。

**α 的兩個極端**：

| α | 公式 | 意義 |
|---|---|---|
| **α = 0** | τ(n+1) = τ(n) | **最近的實際值不算數**，預測永遠不變 |
| **α = 1** | τ(n+1) = t(n) | **只看最後一次實際 CPU burst**，過去的歷史不算數 |

**展開公式**：

```
τ(n+1) = α·t(n) + (1−α)·α·t(n−1) + … + (1−α)^j·α·t(n−j) + … + (1−α)^(n+1)·τ0
```

因為 α 和 (1 − α) 都 ≤ 1，**越久以前的 burst 權重越小**。

> 💡 補充例子：像天氣預報結合「昨天的實際溫度」和「之前的預報」。α 越大，越相信最近發生的事。

### 5.3.3 Round-Robin（RR，輪流排程）

- **專為 time-sharing 系統設計**。
- 每個 process 拿到一小段 CPU 時間，稱為 **time quantum（時間配額）** 或 **time slice（時間片）**，通常 **10 到 100 毫秒**。
- Time quantum **到期（expired）** 時，process 就被**搶占**。
- Ready queue 維持為 **FIFO queue**：
  - 新 process 加到**尾端（tail）**
  - 排程**頭端（head）** 的第一個 process 執行
  - Timer 響時，執行中的 process 被搶占並加到尾端

> 📌 投影片例子 1：Time quantum = 4（全部在時間 0 到達）
>
> | Process | Burst Time |
> |---|---|
> | P1 | 24 |
> | P2 | 3 |
> | P3 | 3 |
>
> ```
> | P1 | P2 | P3 | P1 | P1 | P1 | P1 | P1 |
> 0    4    7   10   14   18   22   26   30
> ```
> - 通常 RR 的平均 turnaround 比 SJF **高**，但**回應比較好**。
>
> （💡 補充計算：等待時間 P1 = 30 − 24 = 6、P2 = 4、P3 = 7，平均 17 / 3 ≈ 5.67。和 FCFS 的 17 比，好很多。）

> 📌 投影片例子 2：Time quantum = 20（全部在時間 0 到達）
>
> | Process | Burst Time |
> |---|---|
> | P1 | 53 |
> | P2 | 17 |
> | P3 | 68 |
> | P4 | 24 |
>
> ```
> | P1 | P2 | P3 | P4 | P1 | P3 | P4 | P1 | P3 | P3 |
> 0   20   37   57   77   97  117  121  134  154  162
> ```
> - P2 只要 17，第一輪就做完（20 → 37）。
> - P4 第二輪只剩 4（117 → 121）。
>
> （💡 補充計算：完成時間 P1 = 134、P2 = 37、P3 = 162、P4 = 121；等待時間 = 完成 − burst：P1 = 81、P2 = 20、P3 = 94、P4 = 97，平均 = 292 / 4 = 73。）

**RR 的性質**：
- 若 ready queue 有 **n** 個 process，time quantum 為 **q**：
  - 每個 process 每 **n·q** 個時間單位得到一段 q 的 CPU 時間，也就是每 q 個時間單位得到 **1/n** 的 CPU。
  - 每個 process **最多等待 (n − 1) × q** 個時間單位。
- **RR 的效能取決於 q**：
  - **q → ∞**：RR 變成 **FCFS**。
  - **q 越小**：**context switch 次數越多**。
- Time quantum 相對於 context-switch 時間**必須夠大**，但**也不能太大**。

> 📌 投影片例子：**較小的 time quantum 會增加 context switch**（一個 process 需要 10 個時間單位）
>
> | Quantum | Context switches |
> |---|---|
> | 12 | 0 |
> | 6 | 1 |
> | 1 | 9 |
>
> - q 應該遠大於 context switch 的時間。
> - q 通常 **10 ms 到 100 ms**，context switch **< 10 μs**。

> 📌 投影片例子：**Average turnaround time 隨 time quantum 變化**（P1 = 6、P2 = 3、P3 = 1、P4 = 7）
>
> | Time quantum | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
> |---|---|---|---|---|---|---|---|
> | Avg turnaround（約） | 11.0 | 11.5 | 10.75 | 11.5 | 12.25 | 10.5 | 10.5 |
>
> 重點：turnaround time **不一定**隨 quantum 變大而改善。
> （💡 補充：課本的經驗法則是約 **80%** 的 CPU burst 應該比 time quantum 短。以 q = 6 驗算：P1 0–6、P2 6–9、P3 9–10、P4 10–16、P4 16–17，turnaround 為 6、9、10、17，平均 42 / 4 = 10.5。）

### 5.3.4 Priority Scheduling（優先權排程）

- 每個 process 關聯一個 **priority number（優先權數字，整數）**。
- 選擇**優先權最高**的 process。
- 優先權相同時，依 **FCFS** 順序。

**優先權的定義方式**：

| 類型 | 中文 | 例子 |
|---|---|---|
| **External priority** | 外部優先權 | 由系統外部決定：重要性、政治因素… |
| **Internal priority** | 內部優先權 | 用可量測的數值計算：時間限制、記憶體需求、開啟檔案數… |

- **SJF 其實是 priority 演算法的一種**：優先權就是**預測的下一次 CPU burst 時間**；CPU burst 越長，優先權越低。

**可搶占或不可搶占**：

| | 新到達的 process 優先權**較高**時 |
|---|---|
| **Preemptive priority** | **搶占** CPU |
| **Nonpreemptive priority** | 把新 process 放到 ready queue 的**頭端**（等目前的跑完） |

> 📌 投影片例子 1：Nonpreemptive priority（全部在時間 0 依 P1…P5 順序到達；**數字越小優先權越高**）
>
> | Process | Burst Time | Priority |
> |---|---|---|
> | P1 | 10 | 3 |
> | P2 | 1 | 1 |
> | P3 | 2 | 4 |
> | P4 | 1 | 5 |
> | P5 | 5 | 2 |
>
> ```
> | P2 |   P5   |        P1         | P3 | P4 |
> 0    1        6                  16   18   19
> ```
> 等待時間：P1 = 6、P2 = 0、P3 = 16、P4 = 18、P5 = 1
> 平均等待時間 = 41 / 5 = **8.2 ms**

> 📌 投影片例子 2：**Priority ＋ Round-Robin**（time quantum = 2 ms）
> 規則：執行優先權最高的 process；**優先權相同的用 RR 輪流**。
>
> | Process | Burst Time | Priority |
> |---|---|---|
> | P1 | 4 | 3 |
> | P2 | 5 | 2 |
> | P3 | 8 | 2 |
> | P4 | 7 | 1 |
> | P5 | 3 | 3 |
>
> ```
> |    P4    | P2 | P3 | P2 | P3 | P2 |  P3  | P1 | P5 | P1 | P5 |
> 0          7    9   11   13   15   16     20   22   24   26   27
> ```
> - P4 優先權最高（1），而且是同級唯一的一個，所以一路跑完（0–7）。
> - P2、P3（優先權 2）用 RR 輪流；P2 在 16 做完後，P3 自己跑到 20。
> - P1、P5（優先權 3）最後輪流。
>
> （💡 補充計算：等待時間 P1 = 26 − 4 = 22、P2 = 16 − 5 = 11、P3 = 20 − 8 = 12、P4 = 0、P5 = 27 − 3 = 24，平均 69 / 5 = 13.8。）

**主要問題：Indefinite blocking（無限期阻塞）＝ Starvation（飢餓）** ⭐
- 低優先權的 process **可能永遠不會被執行**。
- 一個 ready 但一直等不到 CPU 的 process，可以視為被 block 住。

**解法：Aging（老化）**
- 對在系統中**等待很久**的 process，**逐步提高**它的優先權。

> 💡 補充例子：醫院急診依傷勢輕重（優先權）看診，扭到腳的人可能一直被更急的病人插隊（starvation）。Aging 就像規定「每等 30 分鐘就往前升一級」，最後一定輪得到。

### 5.3.5 Multilevel Queue Scheduling（多層佇列排程）

- 把 process **分成不同群組**，因為不同群組可能有不同的排程需求。
  - 例如：**foreground（前景，互動式）** process 與 **background（背景，批次）** process。
- 把 ready queue **分割成多個獨立的佇列**：
  - Process **永久（permanently）** 被指派到某一個佇列。
  - **每個佇列有自己的排程演算法**，例如 foreground 用 RR、background 用 FCFS。

> 📌 投影片例子：**每個優先權各有一個佇列（Fig. 5.7）**
> ```
> priority = 0  : T0  T1  T2  T3  T4
> priority = 1  : T5  T6  T7
> priority = 2  : T8  T9  T10 T11
>      ⋮
> priority = n  : Tx  Ty  Tz
> ```

**佇列之間如何排程**（兩種方式）：

| 方式 | 說明 |
|---|---|
| **Fixed-priority preemptive scheduling（固定優先權可搶占）** | 每個佇列對較低優先權的佇列有**絕對優先權**；較高優先權的佇列**全部空了**，較低的才能執行；有新的高優先權 process 進來，就搶占目前的。**可能造成 starvation** |
| **Time slice among the queues（佇列間分配時間片）** | 每個佇列分到一定比例的 CPU 時間，再分給佇列內的 process。例如 **80%** 給 foreground（RR）、**20%** 給 background（FCFS） |

> 📌 投影片例子：四個佇列，由高到低
> ```
> 最高優先權  ── real-time processes
>             ── system processes
>             ── interactive processes
> 最低優先權  ── batch processes
> ```
> 採用 fixed-priority preemptive 時，batch 佇列中的 process 必須等上面三個佇列**全部空了**才能執行。

### 5.3.6 Multilevel Feedback Queue Scheduling（多層回饋佇列排程）⭐

- **最通用（most general）**、也**最複雜**的 CPU 排程演算法。
- Process **可以在各佇列之間移動**（沒有永久固定的優先權）。
  - 依 CPU burst 的特性把 process 分開。
- 用 **aging 防止 starvation**：
  - 用了**太多 CPU 時間** → 移到**較低**優先權的佇列。
  - 在佇列中**等太久** → 移到**較高**優先權的佇列。

> 📌 投影片例子：三個佇列
> ```
>  新工作 ──► [ Q0：quantum = 8  ] ──► 8 ms 內做完就離開
>                    │ 沒做完
>                    ▼
>             [ Q1：quantum = 16 ] ──► 再給 16 ms 內做完就離開
>                    │ 還沒做完
>                    ▼
>             [ Q2：FCFS         ] ──► 做完離開
> ```
> - 新工作進入 **Q0**（Q0 內以 FCFS 服務）。拿到 CPU 時得到 **8 ms**；8 ms 內沒做完就移到 **Q1**。
> - 在 Q1 同樣以 FCFS 服務，再得到 **16 ms**；還是沒做完就被搶占，移到 **Q2**。
> - Q2 用 FCFS。
> - 效果：**CPU burst ≤ 8 ms 的 process 得到最高優先權**——它們很快拿到 CPU、做完 burst 就去做 I/O。
>
> 💡 補充：只有 Q0 空了才會執行 Q1，Q0 和 Q1 都空了才會執行 Q2。所以短的、互動式的 process 自然留在上層，CPU-bound 的長工作沉到下層。

**定義一個 multilevel feedback queue scheduler 的參數**：
1. 佇列的**數量**
2. 每個佇列的**排程演算法**
3. 決定何時把 process **升級（upgrade）** 的方法
4. 決定何時把 process **降級（demote）** 的方法
5. 決定 process 需要服務時**進入哪一個佇列**的方法

| | **Multilevel Queue** | **Multilevel Feedback Queue** |
|---|---|---|
| Process 能否換佇列 | **不能**，永久指派 | **可以**，依行為升降 |
| Starvation | 可能（fixed-priority 時） | 用 aging 防止 |
| 彈性 | 較低 | **最高**，但最複雜 |

---

## 5.4 執行緒排程 (Thread Scheduling)

- 在同時支援 user-level 與 kernel-level thread 的 OS 上，**被 OS 排程的是 kernel-level thread**。
- User-level thread 要在 CPU 上執行，最終必須對應到一個 kernel-level thread。

| | **PCS：Process-Contention Scope（行程競爭範圍）** | **SCS：System-Contention Scope（系統競爭範圍）** |
|---|---|---|
| 誰在排程 | **Thread library** 把 user-level thread 排到可用的 **LWP** 上 | **Kernel** 把 kernel thread 排到可用的 **CPU** 上 |
| 競爭對象 | 同一個 **process** 內的 thread 互相競爭 | **整個系統**中所有 thread 互相競爭 |
| 適用模型 | Many-to-one、many-to-many | 所有模型；**one-to-one 只用 SCS** |

- 採用 **one-to-one** 模型的系統（**Windows、Linux、Solaris**）**只用 SCS** 排程 thread。
- **POSIX Pthread API** 允許在建立 thread 時指定 PCS 或 SCS：
  - **`PTHREAD_SCOPE_PROCESS`**：用 PCS 排程
  - **`PTHREAD_SCOPE_SYSTEM`**：用 SCS 排程
- 可能受 OS 限制：**Linux 與 Mac OS X 只允許 `PTHREAD_SCOPE_SYSTEM`**。

> 📌 投影片例子（Fig. 5.10）：Pthread scheduling API
> ```c
> #include <pthread.h>
> #include <stdio.h>
> #define NUM_THREADS 5
>
> int main(int argc, char *argv[])
> {
>     int i, scope;
>     pthread_t tid[NUM_THREADS];
>     pthread_attr_t attr;
>
>     /* get the default attributes */
>     pthread_attr_init(&attr);
>
>     /* first inquire on the current scope */
>     if (pthread_attr_getscope(&attr, &scope) != 0)
>         fprintf(stderr, "Unable to get scheduling scope\n");
>     else {
>         if (scope == PTHREAD_SCOPE_PROCESS)
>             printf("PTHREAD_SCOPE_PROCESS");
>         else if (scope == PTHREAD_SCOPE_SYSTEM)
>             printf("PTHREAD_SCOPE_SYSTEM");
>         else
>             fprintf(stderr, "Illegal scope value.\n");
>     }
>
>     /* set the scheduling algorithm to PROCESS or SYSTEM */
>     pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM);
>
>     /* create the threads */
>     for (i = 0; i < NUM_THREADS; i++)
>         pthread_create(&tid[i], &attr, runner, NULL);
>
>     /* now join on each thread */
>     for (i = 0; i < NUM_THREADS; i++)
>         pthread_join(tid[i], NULL);
> }
>
> /* Each thread will begin control in this function */
> void *runner(void *param)
> {
>     printf("I am a thread\n");
>     pthread_exit(0);
> }
> ```
> 程式先用 `pthread_attr_getscope()` 查詢目前的競爭範圍，再用 `pthread_attr_setscope()` 設成 SCS，然後建立 5 個 thread。
> ⚠️ 投影片程式中用到的 `scope` 變數沒有宣告，`NUM THREADS`、`pthread t`、`pthread attr t` 也少了底線，上面已補正（`int i, scope;`、`NUM_THREADS`、`pthread_t`、`pthread_attr_t`）。實際編譯時 `runner` 也需要先宣告原型。

---

## 5.5 多處理器排程 (Multiple-Processor Scheduling)

- 有多個 CPU 時，CPU 排程變得**更複雜**。
- 多個 CPU 讓 **load sharing（負載分擔）** 成為可能。

| | **Homogeneous（同質性）** | **Heterogeneous（異質性）** |
|---|---|---|
| 處理器 | 功能**完全相同** | 能力**不同** |
| 排程 | 可以用**任何**可用的處理器執行佇列中的任何 process | 受限於各處理器的能力 |

### 5.5.1 多處理器排程的方法

| | **Asymmetric multiprocessing（非對稱式）** | **Symmetric multiprocessing, SMP（對稱式）** |
|---|---|---|
| 做法 | 所有系統活動由**一個特定處理器**處理，稱為 **master server（主伺服器）**；其他處理器只執行 user code | **每個處理器自己排程（self-scheduling）** |
| 優點 | 只有一個處理器存取系統資料結構，**減少資料共享的需要** | 沒有單一處理器的瓶頸 |
| Ready queue | — | 可以是**共用的 ready queue**，或**每個處理器各自私有的 ready queue** |
| 使用 | — | **幾乎所有現代 OS**：Windows、Linux、macOS、Android、iOS |

**Ready queue 的兩種組織方式**：

```
 (a) 共用 ready queue                 (b) 每個處理器私有 ready queue
     ┌──────────────────┐                core0 ◄── [ T T T ]
     │  T  T  T  T  T   │──► core0        core1 ◄── [ T T ]
     └──────────────────┘──► core1          ⋮
                         ──► …           coren ◄── [ T T T T ]
                         ──► coren
```

> 💡 補充：共用佇列可能有 race condition（兩個 core 同時搶同一個 thread），需要鎖保護；私有佇列沒有這個問題，也比較能利用 cache，是大多數 SMP 系統的做法，但需要 load balancing。

### 5.5.2 Multicore Processors（多核心處理器）

- 近年的趨勢：把多個處理器**核心**放在同一顆實體晶片上。
  - 對 OS 而言，每個核心看起來就像一個獨立的實體處理器。
  - 比每個處理器各自一顆晶片**更快、更省電**。
- **Memory stall（記憶體停滯）**：處理器存取記憶體時，花大量時間**等資料可用**（例如發生 **cache miss**）。

**Multithreaded multicore system（Fig. 5.13）**：
- 每個核心有**多個 hardware thread（硬體執行緒）** 的設計也越來越多。
  - **Multithreaded processor core**：每個核心分配兩個以上的 hardware thread。
  - 利用 memory stall 的空檔：一個 thread 在等記憶體時，**讓另一個 thread 繼續執行**。
  - 對 OS 而言，每個 hardware thread 都是一個 **logical processor（邏輯處理器）**，可以執行一個 software thread。
  - **Dual-threaded、dual-core** 的系統，OS 看到 **4 個 logical processor**。

```
 thread1:       | C | M | C | M | C | M | C |
 thread0:   | C | M | C | M | C | M | C |
            ─────────────────────────────────► time
 C = compute cycle（運算）  M = memory stall cycle（等記憶體）
 一個 thread 在 M 時，核心去執行另一個 thread 的 C
```

- **CMT（Chip-Multithreading，晶片多執行緒）**：每個核心分配多個 hardware thread。**Intel 稱為 hyperthreading（超執行緒）**。
- 📌 投影片例子：**4 核心、每核 2 個 hardware thread** → OS 看到 **8 個 logical processor**（CPU0–CPU7）。

**兩層排程**：
```
 software threads   ⌇ ⌇ ⌇ ⌇
        level 1：OS 決定哪個 software thread 跑在哪個 hardware thread 上
 hardware threads    ⌇   ⌇     （logical processors）
        level 2：核心決定哪個 hardware thread 使用運算單元
 processing core      [core]
```
- 若 OS **知道處理器資源的共享程度**，就能把 software thread 排到**不共享資源**的 logical processor 上。

> 💡 補充例子：一個廚師（核心）同時顧兩鍋菜（hardware thread）。一鍋在等水滾（memory stall）時，就去處理另一鍋。從外面看像有兩個廚師，但其實只有一雙手，所以兩個 hardware thread 的效能不等於兩個真正的核心。

### 5.5.3 Load Balancing（負載平衡）

- **Load balancing**：試著讓 SMP 系統中**所有處理器的工作量平均分配**，才能充分發揮多處理器的好處。
- 通常只有在**每個處理器各有私有 ready queue** 的系統才需要；**大多數 OS 採用這種方式**。
- **共用 run queue** 的系統通常**不需要**：處理器一閒下來就立刻從共用佇列取出可執行的 process。

| 方法 | 中文 | 做法 |
|---|---|---|
| **Push migration** | 推送遷移 | 一個特定的 task **定期檢查**各處理器的負載；發現不平衡時，把 process 從過載的處理器**推**到閒置或較不忙的處理器 |
| **Pull migration** | 拉取遷移 | **閒置的處理器**從忙碌的處理器**拉**一個等待中的 task 過來 |

- Push 與 pull 通常**同時並用**。📌 例：**Linux scheduler** 與 **FreeBSD ULE scheduler**。

### 5.5.4 Processor Affinity（處理器親和性）

- **Processor affinity**（affinity：喜好、傾向）：process 對**目前正在執行它的處理器**有親和性。
- 大多數 SMP 系統會**盡量讓 process 留在同一個處理器上**，以利用該處理器 **cache** 中已有的資料。

> 📌 投影片圖：記憶體中的資料 X 被載入處理器 A 的 cache。若 process 被搬到處理器 B，B 的 cache 中沒有 X，必須重新載入；A 的 cache 中的 X 也要作廢——這就是遷移的代價。

| 類型 | 中文 | 說明 |
|---|---|---|
| **Soft affinity** | 軟親和性 | **盡量**讓 process 留在同一處理器，但**不保證** |
| **Hard affinity** | 硬親和性 | 系統提供 system call，讓 process **指定**自己只能在哪些處理器上執行 |

- 很多系統兩種都提供。📌 例：Linux 提供 **`sched_setaffinity()`** system call 支援 hard affinity。

> 💡 補充：Load balancing 和 processor affinity 互相拉扯——為了平衡而搬移 process，就會失去 cache 的好處。

**NUMA 與 CPU 排程**：
- **NUMA（Non-Uniform Memory Access）** 架構：CPU 存取主記憶體某些部分比其他部分**快**（回顧 Ch1）。
- CPU 排程與記憶體配置演算法要**一起考慮 affinity**。
- 若 OS 是 **NUMA-aware（NUMA 感知）**，會把記憶體配置在**靠近**該 thread 所執行 CPU 的地方。

---

## 5.6 即時 CPU 排程 (Real-Time CPU Scheduling)

即時 OS 的 CPU 排程有一些特殊問題。

| | **Soft real-time system（軟即時系統）** | **Hard real-time system（硬即時系統）** |
|---|---|---|
| 保證 | **不保證**關鍵的即時 process 何時被排程；只保證它比非關鍵的 process **優先** | 工作**必須在 deadline 前**完成；超過 deadline 才服務，**等於沒有服務** |
| 💡 補充例子 | 影音串流偶爾延遲一下還能接受 | 汽車安全氣囊、心律調節器 |

**回顧 Ch1：Real-time 與 embedded system**
- **Embedded system（嵌入式系統）**：硬體、軟體（可能再加上機械或其他零件）的組合，設計來執行**特定工作**。嵌入式系統差異相當大，有些有 OS、有些沒有；幾乎都執行 **embedded real-time OS**。
- **Real-time system**：常作為專用應用中的控制裝置；有**明確、固定的時間限制**；必須在限制時間內回傳正確結果才算運作正確。

### 5.6.1 Minimizing Latency（最小化延遲）

- 即時系統是 **event-driven（事件驅動）** 的：通常在等某個即時事件發生，一發生就必須盡快回應並服務。
- **Event latency（事件延遲）**：從**事件發生**到**被服務**所經過的時間。

```
   event E first occurs                real-time system responds to E
          t0 ◄──────── event latency ────────► t1
   ────────────────────────────────────────────────► Time
```

**影響效能的兩種延遲** ⭐：

| | **Interrupt latency（中斷延遲）** | **Dispatch latency（分派延遲）** |
|---|---|---|
| 定義 | 從**中斷到達 CPU** 到**開始執行服務該中斷的常式（ISR）** 的時間 | Scheduler **把目前的 process 移出 CPU 並切換到另一個** 所需的時間 |
| 包含 | 判斷中斷類型 → context switch → 開始 ISR | **Conflicts** 階段 ＋ **Dispatch** 階段 |
| 要求 | 即時 OS 應該**最小化**；硬即時系統必須**有上限（bounded）** | 保持低 dispatch latency **最有效的方法是提供 preemptive kernel** |

**Dispatch latency 的組成**（投影片圖）：

```
 event                                                    response to event
   │◄───────────────────── response interval ──────────────────────►│
   │◄ interrupt ►│◄────────── dispatch latency ──────────►│◄ real-time ►│
   │ processing  │◄── conflicts ──►│◄──── dispatch ─────►│  process    │
                 ↑                                        │  execution  │
         process made available
```

- **Conflicts（衝突階段）**：(1) 搶占 kernel 中正在執行的任何 process；(2) 讓低優先權 process 釋放高優先權 process 需要的資源。（💡 補充，取自課本對此圖的說明）
- **Dispatch**：把高優先權 process 排上可用的 CPU。

### 5.6.2 Priority-based Scheduling（以優先權為基礎的排程）

- 即時排程器必須支援 **preemptive、priority-based** 排程。
  - 但這**只保證 soft real-time**。
  - 📌 例：Linux、Windows、Solaris 都給即時 process **最高優先權**。
- **Hard real-time** 還必須提供**滿足 deadline** 的能力。

**Periodic process（週期性行程）的特性**：
- 以**固定間隔**需要 CPU。
- 有固定的 **processing time t（處理時間）**、**deadline d（截止期限）**、**period p（週期）**。
- **0 ≤ t ≤ d ≤ p**
- 週期性工作的 **rate（速率）** 是 **1/p**。

```
   |◄──────── p ────────►|◄──────── p ────────►|◄──────── p ────────►|
   |◄──── d ────►|        |◄──── d ────►|        |◄──── d ────►|
   |◄ t ►|               |◄ t ►|               |◄ t ►|
   [████]                [████]                [████]
   period1               period2               period3          → Time
```

**Admission control（准入控制）**：
- Process 可能要向排程器**宣告它的 deadline 需求**。
- 排程器用 **admission-control algorithm**：
  - **接受（admit）** 該 process，並保證它能準時完成；或
  - 若無法保證在 deadline 前服務完，就**拒絕（reject）** 這個請求。

### 5.6.3 Rate-Monotonic Scheduling（RMS，速率單調排程）⭐

- 用**靜態優先權（static priority）** 加上**搶占**來排程週期性工作。
- 優先權與**週期成反比**：
  - **週期越短 → 優先權越高**（週期越短的先做；速率越高的先做）
  - 週期越長 → 優先權越低
  - 理由：越常需要 CPU 的工作，給越高的優先權。
- 假設週期性 process 每次 CPU burst 的處理時間都相同。

> 📌 投影片例子 1：P1：p1 = 50、t1 = 20；P2：p2 = 100、t2 = 35
> - **Admission control**：檢查能否讓這些工作都趕上 deadline。
>   - CPU 使用率：P1 = 20/50 = **0.4**，P2 = 35/100 = **0.35**，總和 0.75。
> - P1 週期較短，**P1 的優先權高於 P2**。
>
> （💡 補充：完整排程如下，兩者都趕上 deadline。）
> ```
> | P1 |   P2   | P1 |P2|  idle  | P1 |   P2   | P1 |P2| idle |
> 0   20       50   70 75       100  120     150  170 175    200
> Deadlines:      P1(50)    P1,P2(100)      P1(150)    P1,P2(200)
> ```
> P2 在 20–50 跑了 30，被 50 到達的 P1 搶占，70–75 跑完剩下的 5，在 deadline 100 之前完成。

**RMS 的限制**：
- RMS 是**最佳的**（在靜態優先權演算法中），但有一個限制：**CPU 使用率有上限**，不一定能把 CPU 資源用滿。
- 排程 N 個 process 的**最壞情況 CPU 使用率上限**：

```
U ≤ N (2^(1/N) − 1)

N = 2 時：2 × (√2 − 1) = 0.828…，約 83%
```

> 📌 投影片例子 2：**RMS 錯過 deadline**——P1：p1 = 50、t1 = 25；P2：p2 = 80、t2 = 35
> - 使用率 = 25/50 + 35/80 = 0.5 + 0.4375 ≈ **94%**，超過 83% 的上限。
> ```
> |   P1   |   P2   |   P1   |P2|
> 0       25       50       75 80  ← P2 的 deadline 80 到了，卻只做了 25 + 5 = 30，還差 5
> ```
> - P1 優先權較高，0–25 先跑；P2 跑 25–50 後被 50 到達的 P1 搶占；P1 跑到 75；P2 只剩 75–80 這 5 ms，**來不及在 80 之前完成 35**。
>
> 💡 補充：使用率超過上限**不代表一定**會錯過 deadline，只是 RMS **不再保證**；低於上限則保證一定排得出來。

### 5.6.4 Earliest-Deadline-First Scheduling（EDF，最早截止期限優先）⭐

- Process 變成可執行時，向排程器**宣告它的 deadline**。
- **動態（dynamically）** 依 deadline 指定優先權：
  - **deadline 越早 → 優先權越高**
  - deadline 越晚 → 優先權越低

> 📌 投影片例子：同一組 P1：p1 = 50、t1 = 25；P2：p2 = 80、t2 = 35（RMS 排不出來的那組）
> ```
> |  P1  |    P2    |  P1  | P2 |  P1  | P2 |   | P1 ...
> 0     25         60     85  100   125  145  150
> ```
> - 0–25：P1（deadline 50）先跑完。
> - 25：只剩 P2（deadline 80）。50 時 P1 的第二期到達，deadline 是 100，**比 P2 的 80 晚**，所以 P2 **不被搶占**，一路跑到 60 完成。
> - 60–85：P1（deadline 100）。80 時 P2 的第二期到達，deadline 160，比 P1 的 100 晚，不搶占。
> - 85–100：P2。100 時 P1 第三期到達，deadline 150 **早於** P2 的 160 → P1 搶占。
> - 100–125：P1；125–145：P2 做完剩下的 20；145–150 閒置；150 起 P1……
> - **所有 deadline 都趕上了**。

**EDF 的特點**：
- 與 RMS 不同，EDF **不要求 process 是週期性的**，也**不要求每次 burst 的 CPU 時間固定**。
- EDF **理論上是最佳的**：能讓每個 process 都趕上 deadline，且 CPU 使用率可達 **100%**。
- 實際上**不可能達到 100%**：process 之間的 context switch 與中斷處理都有成本。

| | **RMS** | **EDF** |
|---|---|---|
| 優先權 | **靜態**：依**週期**（週期短 → 高） | **動態**：依**deadline**（越早 → 高） |
| 是否要求週期性 | 是 | 否 |
| CPU 使用率上限 | N(2^(1/N) − 1)，N = 2 約 83% | 理論上 100% |

### 5.6.5 Proportional Share Scheduling（比例分配排程）

- 把 **T 份（shares）** 分配給所有應用程式。
- 一個應用程式拿到 **N 份**，就保證它得到總處理器時間的 **N / T**（N < T）。
- 📌 例：T = 100 份——A 拿 50 份、B 拿 15 份、C 拿 20 份 → A 得到 50%、B 15%、C 20% 的處理器時間。
- 必須搭配 **admission-control policy**：只有在**剩下的份數足夠**時，才接受要求特定份數的 client。

> 💡 補充延伸：上例已用掉 85 份，若再有 D 要求 30 份，就會被拒絕（只剩 15 份）。

### 5.6.6 POSIX Real-Time Scheduling

- 依據 **POSIX.1b** 標準；POSIX API 提供排程即時 thread 的相關函式。
- 為即時 thread 定義**兩種排程類別**：

| 類別 | 說明 |
|---|---|
| **`SCHED_FIFO`** | 用 FIFO 佇列，以 **FCFS** 排程；**相同優先權的 thread 之間沒有 time-slicing** |
| **`SCHED_RR`** | 和 `SCHED_FIFO` 類似，但**相同優先權的 thread 之間會 time-slicing**（輪流） |

（另外還有 **`SCHED_OTHER`**，由系統自行定義，非即時。）

- 取得與設定排程策略的兩個函式：
  - `pthread_attr_getschedpolicy(pthread_attr_t *attr, int *policy)`
  - `pthread_attr_setschedpolicy(pthread_attr_t *attr, int policy)`

> 📌 投影片例子（Fig. 5.25）：POSIX real-time scheduling API
> ```c
> #include <pthread.h>
> #include <stdio.h>
> #define NUM_THREADS 5
>
> int main(int argc, char *argv[])
> {
>     int i, policy;
>     pthread_t tid[NUM_THREADS];
>     pthread_attr_t attr;
>
>     /* get the default attributes */
>     pthread_attr_init(&attr);
>
>     /* get the current scheduling policy */
>     if (pthread_attr_getschedpolicy(&attr, &policy) != 0)
>         fprintf(stderr, "Unable to get policy.\n");
>     else {
>         if (policy == SCHED_OTHER) printf("SCHED_OTHER\n");
>         else if (policy == SCHED_RR) printf("SCHED_RR\n");
>         else if (policy == SCHED_FIFO) printf("SCHED_FIFO\n");
>     }
>
>     /* set the scheduling policy - FIFO, RR, or OTHER */
>     if (pthread_attr_setschedpolicy(&attr, SCHED_FIFO) != 0)
>         fprintf(stderr, "Unable to set policy.\n");
>
>     /* create the threads */
>     for (i = 0; i < NUM_THREADS; i++)
>         pthread_create(&tid[i], &attr, runner, NULL);
>
>     /* now join on each thread */
>     for (i = 0; i < NUM_THREADS; i++)
>         pthread_join(tid[i], NULL);
> }
>
> /* Each thread will begin control in this function */
> void *runner(void *param)
> {
>     /* do some work ... */
>     pthread_exit(0);
> }
> ```
> ⚠️ 投影片第 89 頁把函式名稱寫成 `pthread_attr_getsched _policy`（多了空格、多了底線），正確名稱是程式碼中的 **`pthread_attr_getschedpolicy`**；`pthread exit` 也應為 `pthread_exit`。

---

## 5.7 作業系統實例 (Operating-System Examples)

### 5.7.1 Linux Scheduling

#### 版本 2.5 之前與 2.5：O(1) scheduler

- 2.5 之前：使用標準 UNIX 排程演算法的變形。
- **2.5 版改成常數時間 O(1) 的排程**：
  - **Preemptive、priority-based**
  - 兩個優先權範圍：**time-sharing** 與 **real-time**
  - **Real-time 範圍 0–99**，**nice value 對應 100–140**
  - 對應到全域優先權，**數值越小優先權越高**
  - **優先權越高，quantum 越大**
  - Task 只要 time slice 還有剩就可執行（**active**）；用完了（**expired**）就要等所有其他 task 用完各自的 slice 才能再執行
  - 所有可執行的 task 記錄在**每個 CPU 一個的 runqueue** 資料結構中：
    - 兩個 priority array：**active** 與 **expired**
    - Task 依優先權建立索引
    - active array 空了，兩個 array **互換**
  - 效果不錯，但**互動式 process 的回應時間差**。

> ⚠️ 投影片第 93 頁寫 nice value 對應「100 to 140」，但第 98 頁說 nice +19 對應到全域優先權 **139**，兩者不一致。依課本，一般 task 的範圍是 **100–139**（nice −20 ~ +19 共 40 級），140 個優先權編號是 0–139。

```
 active array                    expired array
 priority   task lists           priority   task lists
   [0]      ○─○                    [0]      ○─○─○
   [1]      ○─○─○                  [1]      ○
    ⋮                               ⋮
  [140]     ○                     [140]     ○─○
```

#### 版本 2.6.23 之後：CFS ⭐

**CFS（Completely Fair Scheduler，完全公平排程器）**：

- **Scheduling class（排程類別）**：
  - 每個類別有特定的優先權。
  - 排程器選擇**最高優先權類別中、優先權最高的 task**。
  - 不再用固定長度的 quantum，而是依 **CPU 時間的比例**分配。
  - 內建 2 個類別，可以再加：(1) **default**、(2) **real-time**。
- 依 **nice value（−20 到 +19）** 計算 quantum：
  - **數值越低，優先權越高**。
  - 計算 **target latency（目標延遲）**：每個 task 至少應該執行一次的時間間隔。
  - Active task 數量增加時，target latency 可能變大。
- CFS 為每個 task 維護一個 **virtual run time（虛擬執行時間）**，存在變數 **`vruntime`**：
  - 依 task 的優先權有一個 **decay factor（衰減係數）**：**優先權越低，衰減率越高**（vruntime 增加得越快）。
  - **預設（normal）優先權**的 task：virtual run time ＝ 實際執行時間。
- **選擇 vruntime 最小的 task** 作為下一個執行者。

> 📌 投影片例子：**CFS 的效能**
> - 每個可執行的 task 放在一棵 **red-black tree（紅黑樹）** 中——一種平衡二元搜尋樹，**key 是 `vruntime`**。
> - Task 變成可執行時加入樹；不可執行時（例如在等 I/O）從樹中移除。
> - 拿到較少處理時間的 task（`vruntime` 較小）在樹的**左邊**；拿到較多的在右邊。
> - **最左邊的節點** `vruntime` 最小，就是**優先權最高**的 task。
> - 樹是平衡的，找最左節點需要 **O(lg N)**；但 Linux 把這個值快取在變數 **`rb_leftmost`**，所以決定下一個 task 只需要讀取快取值。
>
> 💡 補充：回顧 Ch1 的 kernel data structures——投影片當時就提到「Linux uses red-black tree in CPU scheduling」，指的就是這裡。
>
> 💡 補充例子：CFS 像輪流玩遊戲機，誰玩得最少誰下一個玩。優先權高的人，計時器走得比較慢，所以能玩比較久。

**Real-time 與全域優先權**：
- Real-time 排程依照 **POSIX.1b**；real-time task 有**靜態**優先權。
- Real-time 加上一般 task 對應到同一套全域優先權：

```
 0 ──────────── 99 | 100 ──────────── 139
     real-time      |       normal
 數字越小 → 優先權越高
 nice −20 → 100        nice +19 → 139
```

**Linux 的負載平衡與 NUMA**：
- Linux 支援 load balancing，同時也是 **NUMA-aware**。
- **Scheduling domain（排程網域）**：一組可以互相平衡負載的 CPU 核心。
- 網域依照**共享的資源**（例如 cache）來組織。目標是**避免 thread 在網域之間遷移**。

> 📌 投影片例子：一個 physical processor domain（NUMA node）下有 domain0（core0、core1 共用一個 L2）與 domain1（core2、core3 共用一個 L2），四個核心共用 L3。Thread 盡量在同一個 domain 內移動，才不會失去 L2 cache 中的資料。

### 5.7.2 Windows Scheduling

- Windows 使用 **priority-based preemptive scheduling（以優先權為基礎的可搶占排程）**。
- **優先權最高的 thread 下一個執行**。
- **Dispatcher 就是 scheduler**。
- Thread 一直執行，直到：(1) **block**、(2) **用完 time slice**、(3) **被更高優先權的 thread 搶占**、(4) **終止**。
- Real-time thread 可以搶占非 real-time thread。
- **32 級優先權**：
  - **Variable class（可變類別）：1–15**
  - **Real-time class（即時類別）：16–31**
  - **優先權 0**：memory-management thread
- 每個優先權一個佇列。
- 沒有可執行的 thread 時，執行 **idle thread**。

**Windows 優先權類別（priority class）與相對優先權（relative priority）**：

- Win32 API 定義 process 可屬於的幾種 priority class：
  `REALTIME_PRIORITY_CLASS`、`HIGH_PRIORITY_CLASS`、`ABOVE_NORMAL_PRIORITY_CLASS`、`NORMAL_PRIORITY_CLASS`、`BELOW_NORMAL_PRIORITY_CLASS`、`IDLE_PRIORITY_CLASS`
  - 除了 **REALTIME** 之外都是 **variable**（可變的）。
- 某個 priority class 內的 thread 有一個 **relative priority**：
  `TIME_CRITICAL`、`HIGHEST`、`ABOVE_NORMAL`、`NORMAL`、`BELOW_NORMAL`、`LOWEST`、`IDLE`
- **Priority class ＋ relative priority → 數字優先權**：

> 📌 投影片例子：Windows thread priorities
>
> | Relative priority ＼ Priority class | real-time | high | above normal | normal | below normal | idle priority |
> |---|---|---|---|---|---|---|
> | **time-critical** | 31 | 15 | 15 | 15 | 15 | 15 |
> | **highest** | 26 | 15 | 12 | 10 | 8 | 6 |
> | **above normal** | 25 | 14 | 11 | 9 | 7 | 5 |
> | **normal** | **24** | **13** | **10** | **8** | **6** | **4** |
> | **below normal** | 23 | 12 | 9 | 7 | 5 | 3 |
> | **lowest** | 22 | 11 | 8 | 6 | 4 | 2 |
> | **idle** | 16 | 1 | 1 | 1 | 1 | 1 |
>
> 例如：NORMAL_PRIORITY_CLASS 中 relative priority 為 HIGHEST 的 thread，數字優先權是 **10**。

**優先權的動態調整**：
- **Base priority（基礎優先權）** 是該 class 中的 **NORMAL** 那一格（表中粗體）。
- **Quantum 用完** → 優先權**降低**，但**絕不低於 base**。
- 發生**等待（wait）** 後 → 依等待的對象**提高（boost）** 優先權（例如等鍵盤輸入的 boost 比等磁碟的大）。
- **前景視窗（foreground window）** 的 quantum 給 **3 倍**的加成（3x priority boost）。

> 💡 補充：這些動態調整的目的都是讓**互動式**的 thread 回應更快——你正在操作的視窗、剛等到鍵盤輸入的程式，會優先拿到 CPU。

- **Windows 7** 加入 **UMS（User-Mode Scheduling，使用者模式排程）**：
  - 應用程式**獨立於 kernel** 建立與管理 thread。
  - 對大量 thread 而言**有效率得多**。
  - UMS 排程器來自程式語言函式庫，例如 **C++ Concurrent Runtime（ConcRT）** framework。

### 5.7.3 Solaris Scheduling

- **Priority-based scheduling**。
- **六種類別**：

| 類別 | 縮寫 |
|---|---|
| Time sharing（**預設**） | **TS** |
| Interactive | **IA** |
| Real time | **RT** |
| System | **SYS** |
| Fair Share | **FSS** |
| Fixed priority | **FP**（圖中標 FX） |

- 一個 thread 同一時間只屬於一個類別；每個類別有自己的排程演算法。
- **Time sharing 是 multilevel feedback queue**，使用系統管理員可設定的 **loadable table**。

> 📌 投影片例子：**Solaris 的 time-sharing 與 interactive thread dispatch table**（數字越大優先權越高，節錄）
>
> | Priority | Time quantum | Time quantum expired | Return from sleep |
> |---|---|---|---|
> | 0 | 200 | 0 | 50 |
> | 5 | 200 | 0 | 50 |
> | 10 | 160 | 0 | 51 |
> | 20 | 120 | 10 | 52 |
> | 30 | 80 | 20 | 53 |
> | 40 | 40 | 30 | 55 |
> | 50 | 40 | 40 | 58 |
> | 59 | 20 | 49 | 59 |
>
> - **Time quantum**：該優先權得到的時間片。**優先權越高，quantum 越小**（反向關係）。
> - **Time quantum expired**：用完整個 quantum（沒有 block）後的**新優先權**——被**降級**（CPU-bound 的 thread 往下沉）。
> - **Return from sleep**：從睡眠（例如等完 I/O）回來後的新優先權——被**提升到 50–59**（互動式 thread 往上浮）。
>
> 💡 補充：這正是 5.3.6 的 multilevel feedback queue 實作——用完 quantum 降級、等 I/O 回來升級。

- 排程器把各類別的優先權轉成**每個 thread 的全域優先權**：
  - 全域優先權最高的 thread 下一個執行。
  - Thread 一直執行到 (1) block、(2) 用完 time slice、(3) 被更高優先權的 thread 搶占。
  - 相同優先權的多個 thread 用 **RR** 選擇。

```
 global priority                         scheduling order
  169 ┐ interrupt threads                      first
  160 ┘
  159 ┐ realtime (RT) threads
  100 ┘
   99 ┐ system (SYS) threads
   60 ┘
   59 ┐ fair share (FSS)、fixed priority (FX)、
    0 ┘ timeshare (TS)、interactive (IA) threads   last
```

---

## 5.8 演算法評估 (Algorithm Evaluation)

選擇 CPU 排程演算法的準則可能結合多項量測，例如：
- 在**最大回應時間 300 ms** 的限制下，最大化 CPU 使用率。
- 最大化 throughput，使 turnaround time（平均而言）與總執行時間**成線性比例**。

**四種評估方法**：

| 方法 | 中文 | 一句話 |
|---|---|---|
| **Deterministic modeling** | 決定性模型 | 拿一組固定的工作負載，算出每個演算法的表現 |
| **Queueing models** | 佇列模型 | 用機率分布與數學公式分析 |
| **Simulations** | 模擬 | 寫程式模擬電腦系統 |
| **Implementation** | 實作 | 直接寫進 OS 實際跑 |

### 5.8.1 Deterministic Modeling（決定性模型）

（deterministic：既定性的、決定性的）

- 拿一組**特定、預先決定好的工作負載**，定義每個演算法在這組負載下的效能。
- 優點：**簡單、快速**；給出**確切的數字**，可以直接比較演算法。
- 缺點：需要**確切的輸入數字**，答案也只適用於這些情況；**太特定**，需要太多確切的知識才有用。
- 主要用途：**描述排程演算法**與**提供範例**（本章的例子都是這種）。

> 📌 投影片例子：5 個 process，全部在時間 0 到達
>
> | Process | Burst Time |
> |---|---|
> | P1 | 10 |
> | P2 | 29 |
> | P3 | 3 |
> | P4 | 7 |
> | P5 | 12 |
>
> **FCFS**：
> ```
> |   P1   |           P2            | P3 |  P4  |    P5    |
> 0       10                        39   42     49         61
> ```
> 等待：0、10、39、42、49 → 平均 140 / 5 = **28 ms**
>
> **SJF**（nonpreemptive）：
> ```
> | P3 |  P4  |   P1   |    P5    |           P2            |
> 0    3     10       20         32                        61
> ```
> 等待：P1 = 10、P2 = 32、P3 = 0、P4 = 3、P5 = 20 → 平均 65 / 5 = **13 ms**
>
> **RR**（q = 10）：
> ```
> |   P1   |   P2   | P3 |  P4  |   P5   |   P2   |P5|  P2  |
> 0       10       20   23     30       40       50 52     61
> ```
> 等待：P1 = 0、P2 = 61 − 29 = 32、P3 = 20、P4 = 23、P5 = 52 − 12 = 40 → 平均 115 / 5 = **23 ms**
>
> 結論：這組負載下 **SJF 最好**（13），RR 次之（23），FCFS 最差（28）。

### 5.8.2 Queueing Models（佇列模型）

- 很多系統上執行的 process **每天都不同**，沒有一組固定的 process（與時間）能用於 deterministic modeling。
- 可以決定的是：
  - **CPU 與 I/O burst 的分布**：描述某個 CPU burst 出現機率的數學公式，通常是 **exponential（指數）分布**，用**平均值（mean）** 描述。
  - **Process 到達時間的分布**。
- 這些分布可以實際量測後近似，或直接估計。

**Modeling computer system（電腦系統的建模）**：
- 把電腦系統描述成一個 **network of servers（伺服器網路）**。
  - 每個 server 有一個等待中的 process 佇列。
  - **CPU** 是一個 server，它的佇列是 ready queue；**I/O 系統**也是，它的佇列是 device queue。
- **Queueing-network analysis（佇列網路分析）**：
  - **已知**：arrival rate（到達率）與 service rate（服務率）
  - **計算**：使用率、平均佇列長度、平均等待時間等

**Little's formula（Little 公式）** ⭐：

```
n = λ × W
```

| 符號 | 意義 |
|---|---|
| **n** | 平均**佇列長度**（不含正在被服務的 process） |
| **W** | 佇列中的平均**等待時間** |
| **λ** | 新 process 的平均**到達率** |

- 成立條件：系統處於**穩態（steady state）**，即 **process 到達率 = process 離開率**。
- 知道其中兩個就能算出第三個。

> 💡 補充例子：平均每秒到達 7 個 process（λ = 7），佇列中平均有 14 個 process（n = 14），則平均等待時間 W = n / λ = 14 / 7 = **2 秒**。
>
> 💡 補充比喻：一家拉麵店每分鐘來 2 位客人，每位平均要在門口排 10 分鐘，那門口平均就有 2 × 10 = 20 人在排隊。

- 用來比較排程演算法很有用，但**有限制**：到達與服務的分布常被定義成**不切實際、但數學上容易處理（tractable）** 的形式。

### 5.8.3 Simulations（模擬）

- 模擬能對排程演算法做**更準確**的評估。
- 用軟體寫出電腦系統的模型：
  - **Simulator（模擬器）**：有一個代表**時鐘（clock）** 的變數；時鐘增加時，模擬器修改系統狀態，反映裝置、process 與排程器的活動。
  - **Input data（輸入資料）**
  - **Output of statistics（統計輸出）**

> 📌 投影片例子：模擬器的結構
>
> **輸入**：每筆紀錄包含 `pid`、`arrive-time`、`burst-time`、`priority`
>
> **模擬器**（虛擬碼）：
> ```
> clock = 0
> While not-end-of-input-records
> {
>     if clock == r.arrive-time
>         RunPolicy        // preemptive
>     if clock == r.finish-time
>         RunPolicy        // non-preemptive
>                          // preemptive
>     clock++;
>     ....
> }
> ```
> 每個排程物件：
> ```
> Class schedule
> {
>     checkinthread();
>     checkoutthread();
> }
> ```
> **輸出**：Gantt chart、平均等待時間、平均 turnaround time、context switch 次數、完成所有 process 所需的總時間
>
> 說明：nonpreemptive 演算法只在有 process **完成**時才需要重新決策；preemptive 演算法在有 process **到達**時也要重新決策（看新來的是否應該搶占）。

**驅動模擬器的資料來源**：

| 方式 | 中文 | 說明 |
|---|---|---|
| **Distribution-driven simulation** | 分布驅動模擬 | **隨機產生**資料；分布可以用數學定義，或**依經驗（empirically）** 定義 |
| **Simulation with trace tapes** | 追蹤磁帶模擬 | 監控**真實系統**、記錄實際事件序列，做成 **trace tape**，再拿來驅動模擬 |

> 📌 投影片例子：用 trace tape 評估排程器
> ```
>  actual process execution ──record──► trace tape
>                                        CPU 10
>                                        I/O 213
>                                        CPU 12        ──► simulation (FCFS)   ──► performance statistics for FCFS
>                                        I/O 112       ──► simulation (SJF)    ──► performance statistics for SJF
>                                        CPU 2         ──► simulation (RR, q=14) ► performance statistics for RR
>                                        I/O 147
>                                        CPU 173
>                                          ⋮
> ```
> 同一份真實紀錄餵給不同演算法的模擬器，就能在**完全相同的輸入**下比較它們。

### 5.8.4 Implementation（實作）

- 評估排程演算法**唯一完全準確**的方法：**把它寫進 OS**，看它實際運作得如何。
- 讓真正的演算法在**真實系統、真實運作條件**下接受評估。
- 主要困難：**成本很高**（要修改 OS、測試，還要觀察使用者的反應；使用者也可能改變行為來配合新排程器）。

> 📌 投影片例子：**NCNU IM OS Simulator**——<http://ossim.im.ncnu.edu.tw/>
> 暨大資管系的作業系統線上模擬器（Operating System Online Simulator），可模擬 **CPU Scheduling、Real-Time Scheduling、Deadlock、Page Replacement、Disk Scheduling**，可以註冊帳號或以訪客身分使用。本章作業的「實驗組」會用到它。

### 四種評估方法比較

| 方法 | 準確度 | 成本 | 特點 |
|---|---|---|---|
| Deterministic modeling | 只對特定輸入準確 | **最低** | 簡單快速，適合教學範例 |
| Queueing models | 取決於分布是否貼近現實 | 低 | 數學分析，但假設常不切實際 |
| Simulations | **較準確** | 中到高 | 可用 trace tape 重現真實負載 |
| Implementation | **完全準確** | **最高** | 真實環境測試 |

---

## 重點整理

### 排程演算法總表

| 演算法 | 選誰 | 可搶占？ | 優點 | 缺點 |
|---|---|---|---|---|
| **FCFS** | 先到的 | 否 | 簡單 | 平均等待時間可能很長；**convoy effect** |
| **SJF** | 下一次 CPU burst 最短的 | 兩種皆有 | 平均等待時間**最小（最佳）** | 無法得知下一次 burst 長度，只能預測；長工作可能 starvation |
| **SRTF** | 剩餘時間最短的 | 是 | SJF 的搶占版 | 同上 |
| **Priority** | 優先權最高的 | 兩種皆有 | 彈性 | **Starvation** → 用 **aging** 解決 |
| **RR** | 輪流，每人一個 quantum | 是 | **回應時間好**，適合 time-sharing | 平均 turnaround 通常比 SJF 高；效能取決於 q |
| **Multilevel queue** | 依佇列優先權；各佇列各自的演算法 | 是 | 不同類型 process 用不同策略 | Process 不能換佇列；可能 starvation |
| **Multilevel feedback queue** | 同上，但 process 可升降級 | 是 | **最通用** | **最複雜**，參數多 |
| **RMS** | 週期最短的（靜態） | 是 | 靜態優先權中最佳 | 使用率上限 N(2^(1/N) − 1) |
| **EDF** | deadline 最早的（動態） | 是 | 理論上可達 100% 使用率 | 實際上有 context switch 成本 |

### 計算題 SOP（💡 補充整理）

1. 依到達時間與規則畫出 **Gantt chart**。
   - Nonpreemptive：只在**有 process 完成時**重新選。
   - Preemptive：**有 process 到達時**也要重新比較。
2. 從 Gantt chart 讀出每個 process 的 **completion time**。
3. **Turnaround = Completion − Arrival**
4. **Waiting = Turnaround − Burst**
5. 平均等待時間 = 所有 waiting 的總和 ÷ process 數。

### 本章投影片範例答案速查

| 範例 | 演算法 | 平均等待時間 |
|---|---|---|
| P1 24、P2 3、P3 3（P1 先到） | FCFS | 17 |
| 同上（P2、P3 先到） | FCFS | 3 |
| 6、8、7、3 | SJF | 7 |
| 到達 0/2/4/5，burst 7/4/1/4 | Nonpreemptive SJF | 4 |
| 同上 | Preemptive SJF | 3 |
| 到達 0/1/2/3，burst 8/4/9/5 | SRTF | 6.5 |
| burst 10/1/2/1/5，priority 3/1/4/5/2 | Nonpreemptive priority | 8.2 |
| burst 10/29/3/7/12 | FCFS／SJF／RR(q=10) | 28／13／23 |

---

## 容易混淆的觀念對照

| 比較 | A | B |
|---|---|---|
| **CPU scheduler vs. Dispatcher** | **決定**下一個執行誰 | **執行**切換（context switch、切回 user mode、跳到程式位置） |
| **Preemptive vs. Nonpreemptive** | 在 4 種情況（甚至 5 種）都可能排程，可以把執行中的 process 換掉 | 只在 running → waiting 與 terminate 時排程 |
| **I/O-bound vs. CPU-bound** | 很多短的 CPU burst | 少數長的 CPU burst |
| **Turnaround vs. Waiting vs. Response time** | 送出到**完成** | 在 **ready queue** 等的總時間 | 送出到**第一個回應** |
| **SJF vs. SRTF** | 不可搶占；一次跑完整個 burst | 可搶占；比的是**剩餘**時間 |
| **External vs. Internal priority** | 外部因素（重要性、付費） | 可量測的數值（時間限制、記憶體需求） |
| **Starvation vs. Aging** | 問題：低優先權永遠等不到 | 解法：等越久優先權越高 |
| **Multilevel queue vs. MLFQ** | Process 永久固定在一個佇列 | Process 可依行為升降級 |
| **Fixed-priority vs. Time slice among queues** | 高優先權佇列空了才輪到低的 | 依比例分配 CPU 時間（如 80% / 20%） |
| **PCS vs. SCS** | Thread library 排程；同 process 內競爭 | Kernel 排程；全系統競爭 |
| **Asymmetric vs. Symmetric multiprocessing** | 一個 master 處理系統活動 | 每個處理器自己排程 |
| **Homogeneous vs. Heterogeneous** | 處理器功能相同 | 處理器能力不同 |
| **Common vs. Private ready queue** | 不需 load balancing，但有存取衝突 | 需要 load balancing，cache 利用較好 |
| **Push vs. Pull migration** | 定期檢查，把工作從忙的推給閒的 | 閒的處理器自己去拉工作 |
| **Soft vs. Hard affinity** | 盡量留在同一處理器，不保證 | 用 system call 指定只能在哪些處理器上跑 |
| **Physical core vs. Logical processor** | 實體運算單元 | 每個 hardware thread 對 OS 都是一個 logical processor |
| **Soft vs. Hard real-time** | 只保證優先於非關鍵 process | 必須在 deadline 前完成 |
| **Interrupt latency vs. Dispatch latency** | 中斷到達 → ISR 開始 | 停止一個 process → 開始另一個 |
| **RMS vs. EDF** | 靜態優先權，週期越短越高 | 動態優先權，deadline 越早越高 |
| **`SCHED_FIFO` vs. `SCHED_RR`** | 同優先權不輪流 | 同優先權用 time-slicing 輪流 |
| **O(1) scheduler vs. CFS** | Linux 2.5：active／expired 兩個 array | Linux 2.6.23+：依 vruntime，紅黑樹 |
| **Linux nice vs. Windows priority** | 數字**越小**優先權越高 | 數字**越大**優先權越高 |
| **Distribution-driven vs. Trace tape** | 隨機產生資料 | 用真實系統的紀錄 |

---

## 作業（Homework）

### 課本習題（10th edition）

**Ex 5.3、5.4、5.5、5.17、5.18、5.35**（投影片上 **5.4** 與 **5.35** 有特別圈起來）

- **(Ex. 5.4)** 畫出排程的 Gantt chart，算出每個 process 的 waiting time 與平均 waiting time。
- **(Ex. 5.35)** 畫出即時排程的 Gantt chart。

| 組別 | 繳交方式 |
|---|---|
| **對照組** | 繳交**手寫**作業 |
| **實驗組** | 使用 **OS Simulator**（<http://ossim.im.ncnu.edu.tw/>），用**自己的帳號**，繳交**執行結果螢幕截圖** |

> 📌 投影片附的 **Ex. 5.4** 題目：
> Consider the following set of processes, with the length of the CPU burst time given in milliseconds:
>
> | Process | Burst Time | Priority |
> |---|---|---|
> | P1 | 2 | 2 |
> | P2 | 1 | 1 |
> | P3 | 8 | 4 |
> | P4 | 4 | 2 |
> | P5 | 5 | 3 |
>
> The processes are assumed to have arrived in the order P1, P2, P3, P4, P5, all at time 0.
> - a. Draw four Gantt charts that illustrate the execution of these processes using the following scheduling algorithms: **FCFS, SJF, nonpreemptive priority（a larger priority number implies a higher priority）, and RR（quantum = 2）**.
> - b. What is the turnaround time of each process for each of the scheduling algorithms in part a?
> - c. What is the waiting time of each process for each of these scheduling algorithms?
> - d. Which of the algorithms results in the minimum average waiting time (over all processes)?
>
> ⚠️ 注意這題的 priority 是「**數字越大優先權越高**」，和投影片 5.3.4 例子（數字越小越高）**相反**。

> 📌 投影片附的 **Ex. 5.35** 題目：
> Consider two processes, P1 and P2, where **p1 = 50, t1 = 25, p2 = 75, and t2 = 30**.
> - a. Can these two processes be scheduled using **rate-monotonic scheduling**? Illustrate your answer using a Gantt chart such as the ones in Figure 5.21–Figure 5.24.
> - b. Illustrate the scheduling of these two processes using **earliest-deadline-first (EDF)** scheduling.

→ 複習本筆記 [5.3](#53-排程演算法-scheduling-algorithms) 的各演算法範例與 [計算題 SOP](#計算題-sop補充整理)；Ex. 5.35 參考 [5.6.3 RMS](#563-rate-monotonic-schedulingrms速率單調排程) 與 [5.6.4 EDF](#564-earliest-deadline-first-schedulingedf最早截止期限優先) 的投影片例子（注意這題的 p2 是 75，不是 80）。

### 加分題

**Write a simulation program to implement preemptive CPU scheduling algorithms.**（寫一個模擬程式，實作可搶占的 CPU 排程演算法。）
- 參考 **5.8.3 Simulations** 的投影片（本筆記 [5.8.3](#583-simulations模擬)）。

**注意**：
1. 請印出**執行結果畫面** & 交**原始程式**
2. 請找 **TA Demo**（**沒有 Demo 不給分**）

---

## 英中詞彙總表

### A–D

| English | 中文 | 補充 |
|---|---|---|
| admission control | 准入控制 | 能保證才接受 |
| affinity | 親和性 | 投影片譯「喜好、傾向」 |
| aging | 老化 | 解決 starvation |
| arrival rate | 到達率 | Little's formula 的 λ |
| arrival time | 到達時間 | |
| asymmetric multiprocessing | 非對稱式多處理 | master server |
| average waiting time | 平均等待時間 | 本章主要比較標準 |
| background process | 背景行程 | 批次 |
| base priority | 基礎優先權 | Windows |
| batch process | 批次行程 | |
| boost | 提升 | Windows 的優先權加成 |
| burst time | 爆發時間 | 一次 CPU burst 的長度 |
| cache miss | 快取未命中 | |
| CFS (Completely Fair Scheduler) | 完全公平排程器 | Linux 2.6.23+ |
| CMT (Chip-Multithreading) | 晶片多執行緒 | |
| completion time | 完成時間 | |
| convoy effect | 護送效應 | FCFS 的問題 |
| cooperative | 合作式 | 即 nonpreemptive |
| CPU burst | CPU 爆發 | |
| CPU scheduler | CPU 排程器 | |
| CPU utilization | CPU 使用率 | |
| CPU-bound | CPU 密集型 | 少數長 burst |
| CPU–I/O burst cycle | CPU–I/O 爆發週期 | |
| deadline | 截止期限 | |
| decay factor | 衰減係數 | CFS |
| demote | 降級 | |
| deterministic modeling | 決定性模型 | 投影片譯「既定性的、決定性的」 |
| dispatch latency | 分派延遲 | |
| dispatch table | 分派表 | Solaris |
| dispatcher | 分派器 | |
| distribution-driven simulation | 分布驅動模擬 | |

### E–M

| English | 中文 | 補充 |
|---|---|---|
| EDF (Earliest-Deadline-First) | 最早截止期限優先 | 動態優先權 |
| empirically | 依經驗地 | 投影片譯「憑經驗地」 |
| event latency | 事件延遲 | |
| event-driven | 事件驅動 | |
| exponential average | 指數平均 | 預測 burst 長度 |
| external priority | 外部優先權 | |
| FCFS (First-Come, First-Served) | 先到先服務 | |
| fixed-priority preemptive scheduling | 固定優先權可搶占排程 | multilevel queue 之間 |
| foreground process | 前景行程 | 互動式 |
| Gantt chart | 甘特圖 | |
| hard affinity | 硬親和性 | |
| hard real-time | 硬即時 | |
| hardware thread | 硬體執行緒 | |
| heterogeneous | 異質性 | |
| homogeneous | 同質性 | |
| hyperthreading | 超執行緒 | Intel 的 CMT |
| I/O burst | I/O 爆發 | |
| I/O-bound | I/O 密集型 | 很多短 burst |
| idle thread | 閒置執行緒 | Windows |
| indefinite blocking | 無限期阻塞 | 即 starvation |
| internal priority | 內部優先權 | |
| interrupt latency | 中斷延遲 | |
| Little's formula | Little 公式 | n = λ × W |
| load balancing | 負載平衡 | |
| load sharing | 負載分擔 | |
| logical processor | 邏輯處理器 | |
| long-term scheduling | 長期排程 | |
| master server | 主伺服器 | |
| memory stall | 記憶體停滯 | |
| multicore | 多核心 | |
| multilevel feedback queue | 多層回饋佇列 | 最通用 |
| multilevel queue | 多層佇列 | |

### N–R

| English | 中文 | 補充 |
|---|---|---|
| nice value | nice 值 | Linux，−20 ~ +19，越小越優先 |
| nonpreemptive | 不可搶占 | |
| NUMA-aware | NUMA 感知 | |
| O(1) scheduler | O(1) 排程器 | Linux 2.5 |
| optimal | 最佳的 | |
| PCS (Process-Contention Scope) | 行程競爭範圍 | |
| period | 週期 | |
| periodic | 週期性的 | |
| permanently | 永久地 | |
| preemptive | 可搶占 | |
| priority class | 優先權類別 | Windows |
| priority number | 優先權數字 | |
| priority scheduling | 優先權排程 | |
| processing time | 處理時間 | 週期性工作的 t |
| processor affinity | 處理器親和性 | |
| proportional share scheduling | 比例分配排程 | 投影片譯「成比例的」 |
| pull migration | 拉取遷移 | |
| push migration | 推送遷移 | |
| queueing model | 佇列模型 | |
| race condition | 競爭條件 | |
| rate | 速率 | 1/p |
| rate-monotonic scheduling | 速率單調排程 | 投影片譯「速率、單調的」 |
| ready queue | 就緒佇列 | |
| real-time class | 即時類別 | Windows 16–31 |
| red-black tree | 紅黑樹 | CFS |
| relative priority | 相對優先權 | Windows |
| response time | 回應時間 | |
| Round-Robin (RR) | 輪流排程 | |
| runqueue | 執行佇列 | |

### S–Z

| English | 中文 | 補充 |
|---|---|---|
| `SCHED_FIFO` / `SCHED_RR` | — | POSIX 即時排程類別 |
| scheduling class | 排程類別 | Linux CFS |
| scheduling criteria | 排程準則 | |
| scheduling domain | 排程網域 | Linux |
| SCS (System-Contention Scope) | 系統競爭範圍 | |
| self-scheduling | 自我排程 | SMP |
| service rate | 服務率 | |
| shares | 份數 | proportional share |
| short-term scheduling | 短期排程 | 即 CPU 排程 |
| simulation | 模擬 | |
| SJF (Shortest-Job-First) | 最短工作優先 | |
| SMP (Symmetric Multiprocessing) | 對稱式多處理 | |
| soft affinity | 軟親和性 | |
| soft real-time | 軟即時 | |
| SRTF (Shortest-Remaining-Time-First) | 最短剩餘時間優先 | preemptive SJF |
| starvation | 飢餓 | |
| static priority | 靜態優先權 | |
| target latency | 目標延遲 | CFS |
| throughput | 產能 | |
| time quantum / time slice | 時間配額／時間片 | |
| trace tape | 追蹤磁帶 | |
| tractable | 易處理的 | |
| turnaround time | 回復時間 | 周轉時間 |
| UMS (User-Mode Scheduling) | 使用者模式排程 | Windows 7 |
| upgrade | 升級 | |
| variable class | 可變類別 | Windows 1–15 |
| virtual run time (`vruntime`) | 虛擬執行時間 | CFS |
| waiting time | 等待時間 | |
