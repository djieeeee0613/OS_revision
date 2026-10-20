# 作業系統 Chapter 4：Threads & Concurrency（執行緒與並行）筆記

> **教材**：*Operating System Concepts*, 10th ed.（Silberschatz, Galvin, Gagne）
> **投影片**：Mei-Ling Chiang 教授，暨南大學資管系（2026/10/06，共 108 頁）
> **筆記說明**：
> - 英文專有名詞第一次出現時以「**English（中文）**」標示，文末有完整詞彙表。
> - 標示「📌 投影片例子」者出自投影片；標示「💡 補充例子」者是為了幫助理解另外加的比喻或說明，不在投影片內。
> - 標示「⚠️」者是投影片中疑似筆誤或需要留意的地方。

---

## 目錄

- [0. 本章目標](#0-本章目標)
- [4.1 概觀](#41-概觀-overview)
- [4.2 多核心程式設計](#42-多核心程式設計-multicore-programming)
- [4.3 多執行緒模型](#43-多執行緒模型-multithreading-models)
- [4.4 執行緒函式庫](#44-執行緒函式庫-thread-libraries)
- [4.5 隱式執行緒](#45-隱式執行緒-implicit-threading)
- [4.6 執行緒相關議題](#46-執行緒相關議題-threading-issues)
- [4.7 作業系統實例](#47-作業系統實例)
- [重點整理](#重點整理)
- [容易混淆的觀念對照](#容易混淆的觀念對照)
- [作業（Homework）](#作業homework)
- [英中詞彙總表](#英中詞彙總表)

---

## 0. 本章目標

**引言**：Process 由它使用的資源與它執行到的位置來定義，扮演四種角色：

| 角色 | 中文 |
|---|---|
| Unit of resource allocation | 資源分配的單位 |
| Memory context | 記憶體內容 |
| Program that is being executed | 正在執行的程式 |
| Unit of activity | 活動的單位 |

**問題**：如果你的程式想**同時做不只一件事**怎麼辦？

Chapter 3 的 process 模型假設一個 process 只有**單一 thread of control（控制執行緒）**。現代 OS 讓一個 process 可以包含**多個** thread，因此能同時執行多項工作。

**本章目標**：
- 辨識 thread 的基本組成，並比較 thread 與 process。
- 描述設計 multithreaded 應用程式的**好處與挑戰**。
- 說明 **implicit threading** 的各種方法：thread pool、fork-join、Grand Central Dispatch。
- 描述 Windows 與 Linux 如何表示 thread。
- 用 Pthreads、Java、Windows threading API 設計 multithreaded 應用程式。

---

## 4.1 概觀 (Overview)

### Thread 是什麼

**Thread（執行緒）**：**CPU 使用的基本單位**（a basic unit of CPU utilization）。

| Thread **自己擁有**的 | Thread 與同 process 其他 thread **共享**的 |
|---|---|
| **Thread ID**（執行緒識別碼） | **Code section**（程式碼區段） |
| **Program counter**（程式計數器） | **Data section**（資料區段，含全域變數） |
| **Register set**（暫存器組） | 其他 OS 資源：**open files**（已開啟檔案）、**signals**（訊號） |
| **Stack**（堆疊） | |

- 傳統的 process 只有單一 thread of control。
- Process 若有多個 thread，就能同時做多件事。

**Single-threaded vs. Multithreaded Process（Fig. 4.1）** ⭐

```
   single-threaded process              multithreaded process
 ┌───────────────────────┐      ┌─────────────────────────────────┐
 │  code │ data │ files  │      │     code  │  data  │  files     │  ← 共享
 ├───────────────────────┤      ├──────────┬──────────┬───────────┤
 │ registers │ PC │ stack│      │registers │registers │ registers │
 ├───────────────────────┤      │  stack   │  stack   │   stack   │  ← 各自擁有
 │                       │      │   PC     │   PC     │    PC     │
 │       thread  ⌇       │      ├──────────┼──────────┼───────────┤
 │                       │      │    ⌇     │    ⌇     │     ⌇     │
 └───────────────────────┘      └──────────┴──────────┴───────────┘
```

> 💡 補充例子：Process 像一間餐廳，thread 像裡面的廚師。廚房、食材、菜單（code、data、files）大家共用；但每位廚師有自己的雙手、自己正在做到食譜的第幾步（PC）、自己的工作檯面（stack）。

### 4.1.1 Motivation（動機）

現代電腦與行動裝置上的軟體**大多是 multithreaded**。一個應用程式通常實作成一個獨立的 process，內含數個 thread。

> 📌 投影片例子：
> - **Web browser（瀏覽器）**：一個 thread 顯示圖片或文字，另一個 thread 從網路擷取資料。
> - **Word processor（文書處理器）**：一個 thread 顯示圖形、一個 thread 回應使用者按鍵、一個 thread 在背景做拼字與文法檢查。
> - 在多核心系統上，應用程式可以把數個 **CPU-intensive（CPU 密集）** 的工作平行執行。

#### Web Server 的三種設計 ⭐

某些情況下，單一應用程式需要執行數個相似的工作。

**(1) Single-threaded process**——一次只能服務一個 client：
```c
void main() {
    while (1) {
        receive();
        disk read;
        send();
    }
}
```

**(2) Multiple processes**——每來一個請求就 `fork()` 一個新 process 去服務：
```c
void main() {
    while (1) {
        receive();
        pid = fork();
        if (pid == 0) {
            disk read;
            send();
            exit(0);
        }
    }
}
```

**(3) Multithreaded process**——每來一個請求就建立一個新 thread：
```c
void fun() {
    disk read;
    send();
    thread_exit(0);
}

void main() {
    while (1) {
        receive();
        createthread(fun);
    }
}
```

| 設計 | 優點 | 缺點 |
|---|---|---|
| Single-threaded | 簡單 | 一次只服務一個 client，其他人要等 |
| Multiple processes | 可同時服務多個 client | **Process 建立耗時又耗資源**（要複製整個位址空間） |
| **Multithreaded** | 可同時服務多個 client；建立成本低 | 需要處理同步問題 |

**結論**：如果新 process 要做的事和既有 process **相同**，讓一個 process 包含多個 thread 通常**更有效率**。

#### 其他動機

- 在 multithreaded process 中，一個 thread 被 block 而等待時，同一個 task 的**另一個 thread 仍可執行**。
  - 多個 thread 合作可帶來更高的 **throughput（產能）** 與更好的效能。
  - 需要共享 buffer 的應用（如 **producer–consumer**）因 thread 而受益。

> 📌 投影片例子：**Buffering problem 的兩種解法**
>
> | | Two-Process Solution | Two-Thread Solution |
> |---|---|---|
> | 結構 | producer process ＋ consumer process | **一個 process 內有 2 個 thread** |
> | Buffer 的實作 | 需要 **shared memory** 或 **message passing**（要靠 IPC） | Buffer、`in`、`out` 本來就在同一個位址空間，**直接共享** |

- **大多數 OS kernel 也是 multithreaded**。
  - 📌 投影片例子：Linux 開機時會建立數個 **kernel thread（核心執行緒）**，各自負責特定工作，如管理裝置、管理記憶體、處理中斷。
- **OS 執行時機 (5)：kernel thread**——延續 Ch1 的主線（開機、interrupt、exception、system call），第五個時機是 kernel thread 在 kernel mode 中執行。

### 4.1.2 Benefits（好處）⭐

| 好處 | 中文 | 說明 |
|---|---|---|
| **Responsiveness** | 回應性 | 互動式應用程式即使有一部分被 block 或在做耗時的運算，仍能繼續執行，對使用者的回應更快 |
| **Resource sharing** | 資源共享 | Thread 預設就共享所屬 process 的記憶體與資源（process 之間要共享則必須靠 shared memory 或 message passing） |
| **Economy** | 經濟 | 建立 thread 與 thread 之間的 context switch 都比 process **便宜** |
| **Scalability** | 可擴展性 | 可利用多處理器架構，讓 thread **平行**執行 |

> 💡 補充例子（Responsiveness）：按下「下載」後，如果下載和畫面更新是同一個 thread，整個視窗會凍結到下載完；用另一個 thread 下載，你還能繼續點其他按鈕、看進度條。

#### `fork()` vs. `clone()`——process 與 thread 在記憶體中的差異

| | `fork()` 建立 **process** | `clone()` 建立 **thread**（Linux） |
|---|---|---|
| PCB | 新增一個 PCB2 | 新增一個 PCB2 |
| 記憶體 | **複製**出 process 2，有自己獨立的位址空間 | **仍在 process 1 的位址空間內**，不複製 |
| 全域變數 | 各有一份，互不影響 | **共用同一份** |

> 📌 投影片例子：`clone()` 建立 thread
> ```c
> int sum;
>
> main()
> {
>     sum = 5;
>     clone(runner, …);
>     …
> }
>
> runner()
> {
>     sum = 3;
>     …
> }
> ```
> `main` 和 `runner` 是同一個 process 內的兩個 thread，各有自己的 pc、sp（各自一個 PCB），但 **`sum` 是同一個變數**——`runner` 把它改成 3，`main` 看到的也是 3。
> 所以 thread 又稱為 **lightweight process（輕量級行程）**。

---

## 4.2 多核心程式設計 (Multicore Programming)

Multithreaded programming 提供更有效使用多個運算核心的機制，並改善 **concurrency（並行）**。

### Concurrency vs. Parallelism ⭐

| | **Concurrency（並行）** | **Parallelism（平行）** |
|---|---|---|
| 定義 | 讓所有工作都能**有進展**（make progress），支援一個以上的工作 | 能**同時（simultaneously）** 執行一個以上的工作 |
| 單核心 | 可以——thread 的執行隨時間**交錯（interleaved）** | **不行** |
| 多核心 | 可以 | 可以——系統把不同 thread 分配到不同 core |
| 本質 | 靠 CPU scheduler 快速切換，製造平行的**錯覺** | 真的同時在跑 |

```
 單核心（concurrent, 不是 parallel）：
   core:  T1  T2  T3  T4  T1  T2  T3  T4  T1 ...        → 時間

 雙核心（parallel）：
   core 0:  T1  T3  T1  T3  T1 ...
   core 1:  T2  T4  T2  T4  T2 ...                      → 時間
```

- 單處理器／單核心時，process 是 **concurrently 執行，但不是 in parallel**。
- **有 concurrency 不一定有 parallelism；有 parallelism 一定有 concurrency。**

> 📌 投影片例子：四組工作（各拆成 B_0、B_1、B_2、B_3… 等小段）。一顆 CPU 時，四組輪流各執行一小段，總時間長；兩顆 CPU 時，每個時間點有兩組同時執行，總時間明顯縮短。
> 💡 補充例子：一個人邊煮湯邊切菜，是在兩件事之間來回切換（concurrency）；兩個人一個煮湯一個切菜，是真的同時進行（parallelism）。

### 4.2.1 Programming Challenges（程式設計的挑戰）

多核心系統對兩種人都帶來壓力：
- **OS 設計者**：要寫出能利用多核心平行執行的排程演算法。
- **應用程式設計師**：要修改現有程式、設計 multithreaded 程式。

**五大挑戰**：

| 挑戰 | 中文 | 說明 |
|---|---|---|
| **Dividing activities** | 劃分活動 | 找出哪些工作可以**平行**執行 |
| **Balance** | 平衡 | 確保各工作的**工作量相當** |
| **Data splitting** | 資料分割 | 各工作存取的資料也要切開，才能在不同 core 上執行 |
| **Data dependency** | 資料相依 | 若工作之間的資料有相依性，執行必須**同步** |
| **Testing and debugging** | 測試與除錯 | 平行程式有很多種可能的執行路徑，比單執行緒難測很多 |

### 4.2.2 Types of Parallelism（平行的類型）

| | **Data parallelism（資料平行）** | **Task parallelism（任務平行）** |
|---|---|---|
| 分散的是 | **資料**：把同一份資料的子集合分到多個 core | **工作（thread）**：把不同的 thread 分到多個 core |
| 各 core 做的事 | **相同的運算** | **各自獨特的運算** |

```
 data parallelism：   [ data₀ | data₁ | data₂ | data₃ ]
                         ↓       ↓       ↓       ↓
                       core 0  core 1  core 2  core 3      （資料切開，做同樣的事）

 task parallelism：   [           data             ]
                        ↙      ↓       ↓      ↘
                     core 0  core 1  core 2  core 3        （同一份資料，做不同的事）
```

> 📌 投影片例子：
> - **Data parallelism**：雙核心系統上兩個 thread 平行加總 `int array[N]`——thread A 在 core 0 加總 `array[0]..[N/2-1]`，thread B 在 core 1 加總 `array[N/2]..[N-1]`。
> - **Task parallelism**：thread A 做加總、thread B 做相乘；或 producer thread 與 consumer thread。

---

## 4.3 多執行緒模型 (Multithreading Models)

### User Thread 與 Kernel Thread

| | **User thread（使用者執行緒）** | **Kernel thread（核心執行緒）** |
|---|---|---|
| 位置 | User space，在 kernel **之上** | Kernel space |
| 由誰管理 | **Thread library（執行緒函式庫）**，**不需 kernel 支援** | **OS 直接**支援與管理 |
| 建立與管理速度 | **較快**（不需要 kernel 介入） | 較慢 |

- 幾乎所有現代 OS 都支援 kernel thread：**Windows、Linux、macOS**。

**User-level thread library 的底層可以是什麼**（投影片三張圖）：

| 情況 | Thread library 把 user-level thread 多工在什麼之上 |
|---|---|
| (1) Kernel **不支援** kernel thread | 一個 **OS process** |
| (2) Kernel 支援 kernel thread | **一個** kernel thread |
| (3) Kernel 支援 kernel thread | **一組** kernel thread |

**判斷一個模型好壞的兩個問題**：
1. Thread 能否在多核心系統上**平行**執行？
2. 一個 thread 呼叫 **blocking system call** 時，整個 process 是否還能繼續執行？

### 三種模型 ⭐

User thread 與 kernel thread 之間必須有對應關係，常見三種：

#### 4.3.1 Many-to-One Model（多對一）

把**許多** user-level thread 對應到**一個** kernel thread。

```
  user threads:    ⌇  ⌇  ⌇  ⌇
                    \ |  | /
  kernel thread:       ⌇
```

- Thread 管理由 user space 的 thread library 完成，所以**有效率**。
- 只要一個 thread 呼叫 blocking system call，**整個 process 都會被 block**（因為同一時間只有一個 thread 能存取 kernel）。
- 多個 thread **無法**在多核心上平行執行，無法利用多個處理核心。
- **現在很少系統使用**。
- 📌 投影片例子：**Green threads**——Solaris 系統的 thread library。

#### 4.3.2 One-to-One Model（一對一）

把**每個** user thread 對應到**一個** kernel thread。

```
  user threads:    ⌇  ⌇  ⌇  ⌇
                   |  |  |  |
  kernel threads:  ⌇  ⌇  ⌇  ⌇
```

- 比 many-to-one 有**更多 concurrency**：一個 thread 呼叫 blocking system call 時，另一個 thread 可以繼續執行。
- 允許多個 thread 在多處理器上**平行**執行。
- 缺點：建立 user thread 就要建立對應的 kernel thread，**大量 kernel thread 可能拖累系統效能**。
- 📌 投影片例子：**Linux**、**Windows 家族**。

#### 4.3.3 Many-to-Many Model（多對多）

把許多 user-level thread 多工到**數量較少或相等**的 kernel thread。

```
  user threads:    ⌇  ⌇  ⌇  ⌇            |user threads| ≥ |kernel threads|
                    \ | / | /
  kernel threads:    ⌇  ⌇  ⌇
```

- 開發者可以建立任意多個 user thread，對應的 kernel thread 可在多處理器上平行執行。
- 一個 thread 執行 blocking system call 時，kernel 可以排程另一個 thread 執行。

#### Two-Level Model（二階層模型）

Many-to-many 的一個**變種（variation）**：
- 仍然把許多 user-level thread 多工到較少或相等數量的 kernel thread。
- **同時**允許某個 user-level thread **綁定（bound）** 到一個 kernel thread。
- 看起來最有彈性，但**很難實作**。

#### 三種模型比較

| | **Many-to-One** | **One-to-One** | **Many-to-Many** |
|---|---|---|---|
| 可建立的 thread 數 | 想建多少就多少 | 要小心不要建太多 | 想建多少就多少 |
| 多核心上能否 parallel | ❌（kernel 一次只能排一個 kernel thread） | ✅ | ✅ |
| 一個 thread block 時 | **整個 process block** | 其他 thread 可繼續 | 其他 thread 可繼續 |
| 缺點 | 無法利用多核心 | kernel thread 數量多，有負擔 | 難以實作 |
| 例子 | Green threads | Linux、Windows | — |

**趨勢**：隨著處理核心數量增加，**限制 kernel thread 數量已不那麼重要**，所以**現在大多數 OS 使用 one-to-one model**。

> 💡 補充例子：把 kernel thread 想成收銀檯、user thread 想成客人。
> - Many-to-one：只有一個收銀檯，一位客人卡住（block），後面全部都動不了。
> - One-to-one：每位客人一個專屬收銀檯，最順，但開太多收銀檯成本很高。
> - Many-to-many：開少數幾個收銀檯讓大家輪流用。

---

## 4.4 執行緒函式庫 (Thread Libraries)

**Thread library（執行緒函式庫）**：提供程式設計師建立與管理 thread 的 **API**。

**兩種主要實作方式**：

| | **(1) User-level library** | **(2) Kernel-level library** |
|---|---|---|
| 位置 | 完全在 **user space**，沒有 kernel 支援 | 由 OS 直接支援 |
| 程式碼與資料結構 | 在 user space | 在 **kernel space** |
| 呼叫函式庫的函式 | 只是 user space 中的 **local function call**，**不是** system call | 會產生對 kernel 的 **system call** |

**目前三個主要的 thread library**：

| Library | 說明 |
|---|---|
| **POSIX Pthreads** | POSIX 標準的執行緒延伸；可以實作成 user-level 或 kernel-level library |
| **Windows** | Windows 系統上的 **kernel-level** library |
| **Java** | Java thread API 讓 Java 程式直接建立與管理 thread；通常用**主機系統**上的 thread library 實作（Windows 上用 Windows API；UNIX、Linux、macOS 上用 Pthreads API） |

以下三個例子都做同一件事：**用另一個 thread 計算 sum = 1 + 2 + … + N**。

### 4.4.1 Pthreads

- **Pthreads** 指的是 POSIX 標準（**IEEE 1003.1c**），定義了 thread 建立與同步的 API。
- 是 **specification（規格）**，不是 **implementation（實作）**：API 規定 thread library 的行為，怎麼實作由函式庫開發者決定。
- 常見於 UNIX 類系統：**Linux、macOS**。

> 📌 投影片例子（Fig. 4.11）：使用 Pthread API 的 multithreaded C 程式
> ```c
> /**
>  * 編譯：gcc thrd.c -lpthread      （compile & link with pthread lib.）
>  * 執行：a.out <number>
>  */
> #include <pthread.h>
> #include <stdio.h>
>
> int sum;                       /* this data is shared by the thread(s) */
>
> void *runner(void *param);     /* the thread */
>
> main(int argc, char *argv[])
> {
>     pthread_t tid;             /* the thread identifier */
>     pthread_attr_t attr;       /* set of attributes for the thread */
>
>     /* set the default attributes of the thread */
>     pthread_attr_init(&attr);
>
>     /* create the thread */
>     pthread_create(&tid, &attr, runner, argv[1]);
>
>     /* now wait for the thread to exit */
>     pthread_join(tid, NULL);
>
>     printf("sum = %d\n", sum);
> }
>
> /* The thread will execute in this function */
> void *runner(void *param)
> {
>     int i, upper = atoi(param);
>     sum = 0;
>
>     for (i = 1; i <= upper; i++)
>         sum += i;
>
>     pthread_exit(0);
> }
> ```

**重點函式**：

| 函式 | 作用 |
|---|---|
| `pthread_attr_init(&attr)` | 設定 thread 的預設屬性 |
| `pthread_create(&tid, &attr, runner, argv[1])` | 建立 thread。第三個參數是 **thread entry function（進入點函式）**，第四個是傳給它的 **argument（參數）** |
| `pthread_join(tid, NULL)` | **等待**該 thread 結束（類似 process 的 `wait()`） |
| `pthread_exit(0)` | Thread 結束自己 |

**執行過程**：程式啟動時只有 **main thread**。呼叫 `pthread_create()` 後多了 **runner thread**，兩者共享全域變數 `sum`。Main thread 在 `pthread_join()` 等待；runner 算完並結束後，main 才印出 `sum`。

> 例如執行 `a.out 10`，輸出 `sum = 55`。

- 若 thread 實作有 kernel 支援（例如 **Linux 的 one-to-one mapping**），`pthread_create()` 會讓 OS 多建立一個 PCB（PCB2），有自己的 pc、sp，但仍在 process 1 的位址空間內。

> 📌 投影片例子（Fig. 4.12）：等待 10 個 thread
> ```c
> #define NUM_THREADS 10
>
> /* an array of threads to be joined upon */
> pthread_t workers[NUM_THREADS];
>
> for (int i = 0; i < NUM_THREADS; i++)
>     pthread_join(workers[i], NULL);
> ```

### 4.4.2 Windows Threads

用 Windows thread library 建立 thread 的技巧與 Pthreads 類似。

> 📌 投影片例子（Fig. 4.13）：使用 Windows API 的 multithreaded C 程式
> ```c
> #include <stdio.h>
> #include <windows.h>
>
> DWORD Sum;    /* data is shared by the thread(s) */
>
> /* the thread runs in this separate function */
> DWORD WINAPI Summation(LPVOID Param)
> {
>     DWORD Upper = *(DWORD *)Param;
>
>     for (DWORD i = 0; i <= Upper; i++)
>         Sum += i;
>     return 0;
> }
>
> int main(int argc, char *argv[])
> {
>     DWORD ThreadId;
>     HANDLE ThreadHandle;
>     int Param;
>
>     Param = atoi(argv[1]);
>
>     // create the thread
>     ThreadHandle = CreateThread(NULL, 0, Summation, &Param, 0, &ThreadId);
>
>     // now wait for the thread to finish
>     WaitForSingleObject(ThreadHandle, INFINITE);
>
>     // close the thread handle
>     CloseHandle(ThreadHandle);
>
>     printf("sum = %d\n", Sum);
> }
> ```

| 動作 | Pthreads | Windows |
|---|---|---|
| 建立 thread | `pthread_create()` | `CreateThread()` |
| 等待 thread 結束 | `pthread_join()` | `WaitForSingleObject()` |
| Thread 結束 | `pthread_exit()` | `return` |
| 共享資料 | 全域變數 `sum` | 全域變數 `Sum` |

### 4.4.3 Java Threads

- Java 在**語言層級**支援 thread 的建立與管理。
- Java thread 由 **JVM** 管理，通常用底層 OS 提供的 thread 模型實作。
- JVM 規格**沒有規定** Java thread 如何對應到底層 OS，由各 JVM 實作決定。層次：Compiled Java program → JVM（Java threads）→ Thread library → Kernel threads。

**建立 Java thread 的兩種方式**：
1. **繼承（extend）`Thread` 類別**
2. 定義一個**實作 `Runnable` 介面**的類別 ← **標準做法**

> 📌 投影片例子：三個基本動作
> ```java
> // 實作 Runnable 介面
> class Task implements Runnable
> {
>     public void run() {
>         System.out.println("I am a thread.");
>     }
> }
>
> // 建立 thread
> Thread worker = new Thread(new Task());
> worker.start();
>
> // 等待 thread
> try {
>     worker.join();
> }
> catch (InterruptedException ie) { }
> ```
> 注意：呼叫的是 **`start()`** 而不是 `run()`。`start()` 會配置記憶體、在 JVM 中初始化新 thread，再由新 thread 去執行 `run()`。

> 📌 投影片例子 (1)：**繼承 `Thread` 類別**
> ```java
> public class Summation extends Thread
> {
>     public Summation(int n) {
>         upper = n;
>     }
>
>     public void run() {
>         int sum = 0;
>
>         if (upper > 0) {
>             for (int i = 1; i <= upper; i++)
>                 sum += i;
>         }
>         System.out.println("The summation of " + upper + " is " + sum);
>     }
>
>     private int upper;
> }
>
> public class ThreadTester
> {
>     public static void main(String[] args) {
>         if (args.length > 0) {
>             if (Integer.parseInt(args[0]) < 0)
>                 throw new IllegalArgumentException(args[0] + " must be non-negative.");
>             else {
>                 Summation summationThread = new Summation(Integer.parseInt(args[0]));
>                 summationThread.start();
>             }
>         }
>         else
>             System.err.println("Usage: Summation <integer value>");
>     }
> }
> ```

> 📌 投影片例子 (2)：**實作 `Runnable` 介面**（編譯：`javac Driver.java`）
> ```java
> class Sum
> {
>     private int sum;
>
>     public int getSum() {
>         return sum;
>     }
>
>     public void setSum(int sum) {
>         this.sum = sum;
>     }
> }
>
> class Summation implements Runnable
> {
>     private int upper;
>     private Sum sumValue;
>
>     public Summation(int upper, Sum sumValue) {
>         if (upper < 0)
>             throw new IllegalArgumentException();
>         this.upper = upper;
>         this.sumValue = sumValue;
>     }
>
>     public void run() {
>         int sum = 0;
>         for (int i = 0; i <= upper; i++)
>             sum += i;
>         sumValue.setSum(sum);
>     }
> }
>
> public class Driver
> {
>     public static void main(String[] args) {
>         if (args.length != 1) {
>             System.err.println("Usage Driver <integer>");
>             System.exit(0);
>         }
>
>         // create the object to be shared
>         Sum sumObject = new Sum();
>         int upper = Integer.parseInt(args[0]);
>
>         Thread thrd = new Thread(new Summation(upper, sumObject));
>         thrd.start();
>         try {
>             thrd.join();
>         } catch (InterruptedException ie) { }
>
>         System.out.println("The sum of " + upper + " is " + sumObject.getSum());
>     }
> }
> ```
> **為什麼要多一個 `Sum` 物件？** Java 沒有全域變數，所以把一個 `Sum` 物件的參考同時交給 main thread 和 `Summation`，用它來**共享結果**。
> ⚠️ 投影片最後一行寫的是 `sumObject.get()`，但 `Sum` 類別定義的方法是 **`getSum()`**，照投影片原樣會編譯失敗，上面已改正。

#### Java Executor Framework

除了明確建立 thread，Java 也允許圍繞 **`Executor` 介面**來建立 thread：

```java
public interface Executor
{
    void execute(Runnable command);
}

// 使用方式（示意）
Executor service = new Executor;
service.execute(new Task());
```

好處是把「thread 的建立」與「thread 的執行」分開。搭配 **`Callable`** 與 **`Future`** 還能**取得回傳值**（`Runnable` 的 `run()` 不能回傳結果）：

> 📌 投影片例子：
> ```java
> import java.util.concurrent.*;
>
> class Summation implements Callable<Integer>
> {
>     private int upper;
>
>     public Summation(int upper) {
>         this.upper = upper;
>     }
>
>     /* The thread will execute in this method */
>     public Integer call() {
>         int sum = 0;
>         for (int i = 1; i <= upper; i++)
>             sum += i;
>
>         return new Integer(sum);
>     }
> }
>
> public class Driver
> {
>     public static void main(String[] args) {
>         int upper = Integer.parseInt(args[0]);
>
>         ExecutorService pool = Executors.newSingleThreadExecutor();
>         Future<Integer> result = pool.submit(new Summation(upper));
>
>         try {
>             System.out.println("sum = " + result.get());
>         } catch (InterruptedException | ExecutionException ie) { }
>     }
> }
> ```
> `submit()` 交出工作後立刻拿到一個 `Future`；`result.get()` 會**等到**結果算出來才返回。這樣就不需要前一個例子的 `Sum` 共享物件，也不用自己呼叫 `join()`。

---

## 4.5 隱式執行緒 (Implicit Threading)

**Implicit**（內含的、隱式的）：
- Thread 數量增加後，用 **explicit thread（明確建立的執行緒）** 很難確保程式正確，所以這種做法越來越普遍。
- Thread 的建立與管理交給 **compiler 與 run-time library**，而不是程式設計師。
- 程式設計師只需要指出**哪些工作可以平行執行**。

五種方法：**Thread Pools、Fork-Join、OpenMP、Grand Central Dispatch、Intel TBB**。

### 4.5.1 Thread Pools（執行緒池）

**問題**：multithreaded server 每收到一個請求就建立一個新 thread。
- 建立 thread 需要**時間**。
- 沒有上限的 thread 可能**耗盡系統資源**。

**Thread pool 的做法**：
1. Process 啟動時就**先建立好一批 thread** 放進 pool，讓它們坐著等工作。
2. Server 收到請求時，從 pool **喚醒**一個 thread，把請求交給它。
3. Thread 完成服務後**回到 pool**，等下一個工作。

**好處**：

| 好處 | 說明 |
|---|---|
| 較快 | 用現成的 thread 服務請求，比等新 thread 建立好更快 |
| 有上限 | **限制**任一時間存在的 thread 數量 |
| 分離 | 把「要執行的工作」與「建立工作的機制」分開，可以採用不同的執行策略（例如讓工作**週期性**執行） |

> 📌 投影片例子：**Windows thread pool API**
> ```c
> DWORD WINAPI PoolFunction(AVOID Param) {
>     /*
>      * this function runs as a separate thread.
>      */
> }
>
> QueueUserWorkItem(&PoolFunction, NULL, 0);
> ```
> Java 也提供 thread pool 工具（如前面的 `Executors`）。
>
> 💡 補充例子：餐廳不會每來一桌客人才去招募一位服務生，而是固定雇用幾位，有客人就派一位，服務完回來待命。客人太多時就排隊等，不會無限制加人。

### 4.5.2 Fork-Join

**Fork-Join parallelism**：多個 thread（工作）被 **fork（分出）**，之後再 **join（合併）**。

```
                    ┌─ fork ─► [ task ] ─ join ─┐
   main thread ─────┤                           ├────► main thread
                    └─ fork ─► [ task ] ─ join ─┘
```

**通用演算法**（divide-and-conquer，分治法）：

```
Task(problem)
    if problem is small enough
        solve the problem directly
    else
        subtask1 = fork(new Task(subset of problem))
        subtask2 = fork(new Task(subset of problem))

        result1 = join(subtask1)
        result2 = join(subtask2)

        return combined results
```

每個 task 可以再 fork 出子 task，形成樹狀結構；子 task 完成後一層層 join 回來。

> 📌 投影片例子：**Java 的 Fork-Join**——加總陣列
> ```java
> ForkJoinPool pool = new ForkJoinPool();
> // array contains the integers to be summed
> int[] array = new int[SIZE];
>
> SumTask task = new SumTask(0, SIZE - 1, array);
> int sum = pool.invoke(task);
> ```
> ```java
> import java.util.concurrent.*;
>
> public class SumTask extends RecursiveTask<Integer>
> {
>     static final int THRESHOLD = 1000;
>
>     private int begin;
>     private int end;
>     private int[] array;
>
>     public SumTask(int begin, int end, int[] array) {
>         this.begin = begin;
>         this.end = end;
>         this.array = array;
>     }
>
>     protected Integer compute() {
>         if (end - begin < THRESHOLD) {
>             int sum = 0;
>             for (int i = begin; i <= end; i++)
>                 sum += array[i];
>
>             return sum;
>         }
>         else {
>             int mid = (begin + end) / 2;
>
>             SumTask leftTask = new SumTask(begin, mid, array);
>             SumTask rightTask = new SumTask(mid + 1, end, array);
>
>             leftTask.fork();
>             rightTask.fork();
>
>             return rightTask.join() + leftTask.join();
>         }
>     }
> }
> ```
> **說明**：範圍小於 `THRESHOLD`（1000）就直接加總；否則從中間切成左右兩半，各 fork 一個子 task，再把兩邊 join 回來的結果相加。

| 類別 | 說明 |
|---|---|
| **`ForkJoinTask`** | 抽象基底類別 |
| **`RecursiveTask`** | 繼承 `ForkJoinTask`；**會回傳結果**（透過 `compute()` 的回傳值） |
| **`RecursiveAction`** | 繼承 `ForkJoinTask`；**不回傳結果** |

### 4.5.3 OpenMP

- 一組 **compiler directive（編譯器指示）** 加上 API，支援 **C、C++、FORTRAN**。
- 支援**共享記憶體環境**中的平行程式設計。
- 標示出 **parallel region（平行區域）**——可以平行執行的程式碼區塊。
- 在 Linux、Windows、Mac OS X 的多種開放原始碼與商業 compiler 上都可用。

| 指示 | 作用 |
|---|---|
| `#pragma omp parallel` | 建立與**核心數量一樣多**的 thread，各自執行該區塊 |
| `#pragma omp parallel for` | 把 **for 迴圈**的各次迭代分給多個 thread 平行執行 |

> 📌 投影片例子：
> ```c
> #include <omp.h>
> #include <stdio.h>
>
> int main(int argc, char *argv[])
> {
>     /* sequential code */
>
>     #pragma omp parallel
>     {
>         printf("I am a parallel region.");
>     }
>
>     /* sequential code */
>
>     return 0;
> }
> ```
> 在四核心機器上，這行訊息會被印出四次。
>
> ```c
> #pragma omp parallel for
> for (i = 0; i < N; i++) {
>     c[i] = a[i] + b[i];
> }
> ```
> 兩個陣列相加的迴圈被自動拆給多個 thread 平行執行。
>
> 💡 補充：用 gcc 編譯要加 `-fopenmp`（例如 `gcc -fopenmp prog.c`），否則 `#pragma` 會被忽略，程式照樣能跑但只有單一 thread。

### 4.5.4 Grand Central Dispatch（GCD）

- Apple **Mac OS X 與 iOS** 的技術。
- 是 C、C++ 語言的延伸，加上 API 與 run-time library。
- 讓開發者標示出可以平行執行的區段，由 GCD **管理大部分 threading 細節**。
- **Block（區塊）**：用 `^{ }` 包起來的一段獨立工作，例如 `^{ printf("I am a block"); }`
- Block 被放進 **dispatch queue（分派佇列）**；從佇列取出時，分配給 thread pool 中可用的 thread。

**兩種 dispatch queue**：

| 類型 | 取出方式 | 說明 |
|---|---|---|
| **Serial（循序）** | 依 **FIFO** 順序，**一次一個** | 每個 process 有自己的 serial queue，稱為 **main queue（主佇列）**；程式設計師可在程式中建立額外的 serial queue |
| **Concurrent（並行）** | 依 FIFO 順序，但**一次可以取出多個** | 有三個**全系統**的佇列，優先權分為 **low、default、high** |

> 📌 投影片例子：
> ```c
> dispatch_queue_t queue = dispatch_get_global_queue
>     (DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
>
> dispatch_async(queue, ^{ printf("I am a block."); });
> ```
> 取得預設優先權的全域 concurrent queue，再把一個 block 非同步地丟進去執行。

### 4.5.5 Intel Threading Building Blocks（TBB）

- 設計平行 **C++** 程式的 **template library（樣板函式庫）**。

> 📌 投影片例子：
> ```cpp
> // 循序版本
> for (int i = 0; i < n; i++) {
>     apply(v[i]);
> }
>
> // TBB 的 parallel_for 版本
> parallel_for (size_t(0), n, [=](size_t i) { apply(v[i]); });
> ```
> 開發者只要指出迴圈範圍和每次要做的事，TBB 會自動把迴圈切成數塊並建立工作平行執行。

### 五種 Implicit Threading 比較

| 方法 | 平台／語言 | 核心概念 |
|---|---|---|
| **Thread pool** | 通用（Windows API、Java） | 預先建立一批 thread 重複使用 |
| **Fork-Join** | Java | 分治法：fork 出子工作，再 join 結果 |
| **OpenMP** | C、C++、FORTRAN | 用 `#pragma` 標示平行區域 |
| **GCD** | Apple macOS、iOS | 把 block 丟進 dispatch queue |
| **Intel TBB** | C++ | 樣板函式庫，如 `parallel_for` |

---

## 4.6 執行緒相關議題 (Threading Issues)

### 4.6.1 `fork()` 與 `exec()` System Calls

在 multithreaded 程式中，這兩個 system call 的語意改變了。

**`fork()`**：如果程式中的一個 thread 呼叫 `fork()`，新 process 要複製**所有** thread，還是只有呼叫的那一個？
- 有些 UNIX 系統提供**兩種版本**的 `fork()`：
  1. 新 process **複製所有 thread**。
  2. 新 process 是 **single-threaded**（只複製呼叫 `fork()` 的那個 thread）。

**`exec()`**：如果一個 thread 呼叫 `exec()`？
- `exec()` 參數指定的程式會**取代整個 process——包括所有 thread**。

> 💡 補充：該用哪個版本的 `fork()`？如果 fork 之後馬上要呼叫 `exec()`，複製所有 thread 是浪費（反正整個會被換掉），只複製呼叫者就好；如果 fork 之後不呼叫 `exec()`，新 process 就應該複製所有 thread。

### 4.6.2 Signal Handling（訊號處理）

**Signal（訊號）**：UNIX 系統中用來**通知 process 某個特定事件發生了**。

所有 signal 都遵循同樣的模式：
1. 某個特定事件發生，**產生（generate）** signal。
2. 產生的 signal 被**傳遞（deliver）** 給 process。
3. 傳遞後，signal 必須被**處理（handle）**。

| | **Synchronous signal（同步訊號）** | **Asynchronous signal（非同步訊號）** |
|---|---|---|
| 來源 | **執行中的程式自己**的動作造成（即 exception） | 執行中 process **外部**的事件產生 |
| 例子 | 非法記憶體存取、除以零 | 按 **Ctrl-C** 終止 process、timer 到期 |
| 傳遞對象 | 傳給**造成該 signal 的同一個 process** | 通常送給另一個 process |

**每個 signal 可由兩種 handler 之一處理**：

| Handler | 中文 | 說明 |
|---|---|---|
| **Default signal handler** | 預設訊號處理常式 | Kernel 處理 signal 時執行 |
| **User-defined signal handler** | 使用者自訂訊號處理常式 | 可以**覆蓋（override）** 預設行為 |

> 📌 投影片例子：自訂 signal handler 與傳送 signal
> ```c
> #include <stdio.h>
> #include <signal.h>
>
> void sighup();
> void sigint();
> void sigquit();
>
> int main(int argc, char *argv[]) {
>     int pid;
>
>     if ((pid = fork()) == 0) {
>         printf("this is child process for receiving signal\n");
>         signal(SIGHUP, sighup);       /* 註冊 user-defined signal handlers */
>         signal(SIGINT, sigint);
>         signal(SIGQUIT, sigquit);
>         for (;;);                     /* 無窮迴圈，等待 signal */
>     } else {
>         printf("this is parent process for sending signal\n");
>         printf("\nPARENT: sending SIGHUP\n\n");
>         kill(pid, SIGHUP);            /* sending signals to process */
>         printf("\nPARENT: sending SIGINT\n\n");
>         kill(pid, SIGINT);
>         printf("\nPARENT: sending SIGQUIT\n\n");
>         kill(pid, SIGQUIT);
>         kill(pid, SIGHUP);
>         kill(pid, SIGINT);
>     }
> }
>
> void sighup() {
>     printf("CHILD: I have received a SIGHUP\n");
> }
>
> void sigint() {
>     printf("CHILD: I have received a SIGINT\n");
> }
>
> void sigquit() {
>     printf("My DADDY has Killed me!!!\n");
> }
> ```
> **說明**：`signal(訊號, 函式)` 把某個 signal 的處理方式換成自己的函式；`kill(pid, 訊號)` 把 signal 送給指定的 process（名字叫 kill，但作用是「傳送訊號」，不一定是殺掉）。Child 註冊三個 handler 後空轉等待，parent 依序送出訊號。
> 💡 補充：實際輸出的順序與次數取決於兩個 process 的執行時機——parent 可能在 child 還沒註冊好 handler 之前就送出 signal，這時會套用預設行為（SIGHUP 的預設動作是終止 process）。

**Multithreaded 環境中 signal 要送給誰？** 四種選擇：

| 選擇 | 適用例子 |
|---|---|
| 1. 送給 signal **所適用的那個 thread** | Synchronous signal（誰除以零就送給誰） |
| 2. 送給 process 中的**每一個 thread** | Ctrl-C |
| 3. 送給 process 中的**某些 thread** | Pthreads 提供 `pthread_kill(threadid, signal)` |
| 4. 指定**一個特定 thread** 接收該 process 的所有 signal | Solaris 2 |

### 4.6.3 Thread Cancellation（執行緒取消）

**Thread cancellation**：在 thread **完成之前就終止它**。被取消的 thread 稱為 **target thread（目標執行緒）**。

| | **Asynchronous cancellation（非同步取消）** | **Deferred cancellation（延遲取消）** |
|---|---|---|
| 做法 | 一個 thread **立刻**終止 target thread | Target thread **定期檢查**自己是否該終止 |
| 安全性 | 危險：可能正在更新共享資料、資源可能沒釋放 | 安全：thread 可以在**能安全取消的時間點**才結束 |

- Pthreads 把這種可以安全取消的點稱為 **cancellation point（取消點）**。

> 📌 投影片例子：Pthread 建立並取消 thread
> ```c
> pthread_t tid;
>
> /* create the thread */
> pthread_create(&tid, 0, worker, NULL);
>
> . . .
>
> /* cancel the thread */
> pthread_cancel(tid);
>
> /* wait for the thread to terminate */
> pthread_join(tid, NULL);
> ```

呼叫 `pthread_cancel()` 只是**提出取消的請求**，實際是否取消取決於 thread 的狀態：

| Mode | State | Type |
|---|---|---|
| **Off** | Disabled | – |
| **Deferred** | Enabled | Deferred |
| **Asynchronous** | Enabled | Asynchronous |

- 若 thread **停用（disabled）** 取消，請求會保持 **pending（擱置）**，直到 thread 啟用為止。
- **預設類型是 deferred**。
  - 只有當 thread 到達 **cancellation point** 時才會取消，例如呼叫 **`pthread_testcancel()`**。
  - 然後會呼叫 **cleanup handler（清理處理常式）**，讓 thread 在終止前釋放資源。
- 在 **Linux** 系統，thread cancellation 是透過 **signal** 處理的。

> 📌 投影片例子：**Java 的 thread cancellation**
> Deferred cancellation 使用 `interrupt()` 方法，它會設定 thread 的 **interrupted status（中斷狀態）**：
> ```java
> Thread worker;
>
> . . .
>
> /* set the interruption status of the thread */
> worker.interrupt()
> ```
> Thread 自己檢查是否被中斷：
> ```java
> while (!Thread.currentThread().isInterrupted()) {
>     . . .
> }
> ```
>
> 💡 補充例子：Asynchronous 像直接拔掉正在寫入的隨身碟，檔案可能壞掉。Deferred 像按「安全移除」——等它把手上的東西寫完、收拾好才真的停止。瀏覽器載入網頁時按「停止」，就是在取消負責載入的 thread。

### 4.6.4 Thread-Local Storage（TLS，執行緒區域儲存）

- Thread 共享 process 的資料，但有些情況下**每個 thread 需要某些資料的自己一份副本**，這就是 **TLS**。
- 特性：**跨函式呼叫都可見**（visible across function invocations）。
- 大多數 thread library（包括 Windows 與 Pthreads）都提供某種 TLS 支援；Java 也有。

| | **Local variable（區域變數）** | **TLS** | **Global variable（全域變數）** |
|---|---|---|---|
| 可見範圍 | 只在**單次函式呼叫**期間 | **跨函式呼叫**都可見 | 整個程式 |
| 每個 thread 一份？ | 是（在各自的 stack 上） | **是** | 否（所有 thread 共享） |

> 💡 補充例子：交易處理系統中，每個 thread 處理一筆交易，需要各自記住「我正在處理的交易編號」。這個值要在該 thread 呼叫的許多函式中都能讀到（所以不能是區域變數），又不能被其他 thread 看到或改到（所以不能是全域變數）。C 語言中的 `errno` 也是典型例子。

### 4.6.5 Scheduler Activations（排程器啟動）

- 處理的是 **kernel 與 thread library 之間的溝通**。
- **Many-to-many** 與 **two-level** 模型都需要這種溝通，才能維持配置給應用程式的 kernel thread 數量適當。
- 通常在 user thread 與 kernel thread 之間使用一個中間資料結構——**LWP（Lightweight Process，輕量級行程）**。

```
   user thread      ⌇
                    │
                 ┌─────┐
                 │ LWP │      ← 介於 user-level 與 kernel-level thread 之間
                 └─────┘
                    │
   kernel thread    ⌇
```

- 對 user-thread library 而言，LWP 看起來像一個 **virtual processor（虛擬處理器）**，應用程式可以把 user thread 排到上面執行。
- 每個 LWP 都**附屬於一個 kernel thread**。

**Upcall（向上呼叫）**：
- Scheduler activation 提供 **upcall**——一種**從 kernel 到 thread library** 的溝通機制（方向與 system call 相反）。
- Upcall 由 thread library 中的 **upcall handler** 處理。
- 這種溝通讓應用程式能維持**正確數量的 kernel thread**。

> 💡 補充例子：某個 user thread 即將 block 時，kernel 發 upcall 通知 thread library「這個 thread 要停了」，並配一個新的 LWP 給它，讓 library 可以安排其他 user thread 上去執行，不浪費 CPU。

---

## 4.7 作業系統實例

### 4.7.1 Windows Threads

- Windows 實作 Windows API。
- Windows 應用程式以獨立的 process 執行，每個 process 可包含一或多個 thread。
- 採用 **one-to-one mapping**：每個 user-level thread 對應一個 kernel thread。

**每個 thread 包含**：
- **Thread ID**
- **Register set**：代表處理器的狀態
- 分開的 **user stack** 與 **kernel stack**：分別在 thread 執行於 user mode 與 kernel mode 時使用
- **Private data storage area（私有資料儲存區）**：供 run-time library 與 DLL 使用

Register set、stacks、private storage area 合稱為 thread 的 **context**。

**Thread 的主要資料結構** ⭐：

| 結構 | 全名 | 位置 | 內容 |
|---|---|---|---|
| **ETHREAD** | Executive thread block（執行體執行緒區塊） | **Kernel space** | thread start address（起始位址）、指向 parent process 的指標、指向 KTHREAD 的指標 |
| **KTHREAD** | Kernel thread block（核心執行緒區塊） | **Kernel space** | scheduling and synchronization information（排程與同步資訊）、**kernel stack**、指向 TEB 的指標 |
| **TEB** | Thread environment block（執行緒環境區塊） | **User space** | thread identifier、**user stack**、**thread-local storage** |

```
   kernel space                                    │  user space
   ETHREAD ─────────► KTHREAD ─────────────────────┼──► TEB
   · thread start       · scheduling &             │    · thread identifier
     address              synchronization info     │    · user stack
   · pointer to         · kernel stack             │    · thread-local storage
     parent process                                │
```

### 4.7.2 Linux Threads

- Linux **不區分 process 與 thread**。
  - Linux 通常用 **task（任務）** 這個詞來指程式中的控制流程，而不說 process 或 thread。
- Thread 的建立透過 **`clone()`** system call。
- `clone()` 讓 child task **共享** parent task 的位址空間。
  - 傳入的 **flag（旗標）** 決定 parent 與 child 之間共享多少。

| Flag | 意義 |
|---|---|
| **`CLONE_FS`** | 共享**檔案系統資訊** |
| **`CLONE_VM`** | 共享**同一個記憶體空間** |
| **`CLONE_SIGHAND`** | 共享 **signal handler** |
| **`CLONE_FILES`** | 共享**已開啟的檔案** |

> 💡 補充：這四個 flag 全部帶上，新 task 幾乎什麼都和 parent 共享，行為就等於一個 **thread**；一個 flag 都不帶，什麼都不共享，行為就類似 **`fork()`** 建立的 process。所以在 Linux 中 process 與 thread 只是「共享程度」不同的 task。這也是 Ch3 提到的 `task_struct` 同時用來表示兩者的原因。

---

## 重點整理

### Thread 擁有什麼、共享什麼

| 各 thread **自己有** | 同一 process 的 thread **共享** |
|---|---|
| Thread ID | Code（程式碼） |
| Program counter | Data（全域變數） |
| Register set | Heap |
| Stack（區域變數） | Open files |
| TLS | Signals |

### Process vs. Thread

| | **Process** | **Thread** |
|---|---|---|
| 位址空間 | 各自獨立 | 與同 process 的 thread 共享 |
| 建立成本 | 高（要複製位址空間） | 低 |
| Context switch | 慢 | 快 |
| 彼此溝通 | 需要 IPC（shared memory、message passing） | 直接讀寫共享變數 |
| 隔離性 | 好：一個當掉不影響其他 | 差：一個 thread 出錯可能拖垮整個 process |
| UNIX 建立方式 | `fork()` | `pthread_create()`／`clone()` |

### OS 執行的五個時機（Ch1 到 Ch4 的主線）

(1) System booting　(2) Interrupt　(3) Exception　(4) System call　(5) **Kernel thread**

---

## 容易混淆的觀念對照

| 比較 | A | B |
|---|---|---|
| **Process vs. Thread** | 資源擁有的單位；獨立位址空間 | CPU 使用的單位；共享位址空間 |
| **Concurrency vs. Parallelism** | 都有進展（可以靠切換）；單核心可達成 | 真的同時執行；需要多核心 |
| **Data vs. Task parallelism** | 切**資料**，各 core 做相同運算 | 切**工作**，各 core 做不同運算 |
| **User vs. Kernel thread** | Thread library 管理；快；kernel 不知道 | OS 管理；慢一些；可被 kernel 排程 |
| **Many-to-one vs. One-to-one** | 一個 block 全部 block；不能平行 | 互不影響；可平行；但 kernel thread 多 |
| **Many-to-many vs. Two-level** | 多對多 | 多對多＋允許某個 user thread 綁定一個 kernel thread |
| **User-level vs. Kernel-level library** | 呼叫是 local function call | 呼叫會產生 system call |
| **Explicit vs. Implicit threading** | 程式設計師自己建立與管理 thread | Compiler／run-time library 代勞 |
| **Extend `Thread` vs. Implement `Runnable`** | 繼承類別 | 實作介面（標準做法） |
| **`Runnable` vs. `Callable`** | `run()` 不能回傳結果 | `call()` 可以回傳結果（搭配 `Future`） |
| **`start()` vs. `run()`** | 建立新 thread 再執行 `run()` | 直接呼叫只是一般函式呼叫，不會產生新 thread |
| **`RecursiveTask` vs. `RecursiveAction`** | 回傳結果 | 不回傳結果 |
| **Serial vs. Concurrent dispatch queue** | 一次取一個 | 一次可取多個 |
| **Synchronous vs. Asynchronous signal** | 自己的動作造成（除以零） | 外部事件造成（Ctrl-C） |
| **Asynchronous vs. Deferred cancellation** | 立刻終止，不安全 | 到取消點才終止，安全（預設） |
| **Local variable vs. TLS** | 只在單次函式呼叫內 | 跨函式呼叫，每個 thread 一份 |
| **System call vs. Upcall** | User → kernel | Kernel → thread library |
| **`fork()` vs. `clone()`** | 複製出獨立的 process | 依 flag 決定共享多少，可建立 thread |
| **ETHREAD／KTHREAD vs. TEB** | Kernel space | User space |

---

## 作業（Homework）

### 課本習題（10th edition）

**Ex. 4.4、4.5、4.6、4.9、4.11、4.17、4.19**（投影片上 4.11 有特別圈起來）

### Ex. 4.19：手寫或程式實作

| 做法 | 最高得分 |
|---|---|
| **手寫** | 40 分 |
| **程式實作** | 50 分 |

- 請印出**執行結果畫面**（使用自己的**帳號**和**虛擬機**）。
- 請交**原始程式**。

> 📌 投影片附的 Ex. 4.19 程式：
> ```c
> #include <pthread.h>
> #include <stdio.h>
>
> int value = 0;
> void *runner(void *param);    /* the thread */
>
> int main(int argc, char *argv[])
> {
>     pid_t pid;
>     pthread_t tid;
>     pthread_attr_t attr;
>
>     pid = fork();
>
>     if (pid == 0) {    /* child process */
>         pthread_attr_init(&attr);
>         pthread_create(&tid, &attr, runner, NULL);
>         pthread_join(tid, NULL);
>         printf("CHILD: value = %d", value);     /* LINE C */
>     }
>     else if (pid > 0) {    /* parent process */
>         wait(NULL);
>         printf("PARENT: value = %d", value);    /* LINE P */
>     }
> }
>
> void *runner(void *param) {
>     value = 5;
>     pthread_exit(0);
> }
> ```
> 題目問的是 LINE C 與 LINE P 各會輸出什麼。

→ 這題同時考 process 與 thread 對資料的處理方式。複習本筆記 [4.1.2](#forkvs-clone--process-與-thread-在記憶體中的差異) 的 `fork()` vs. `clone()` 對照、[4.4.1](#441-pthreads) 的 Pthreads 範例，以及 Ch3 筆記的 `fork()`。選擇程式實作的話，編譯時記得加 `-lpthread`。

---

## 英中詞彙總表

### A–F

| English | 中文 | 補充 |
|---|---|---|
| asynchronous cancellation | 非同步取消 | 立刻終止 |
| asynchronous signal | 非同步訊號 | 外部事件產生 |
| balance | 平衡 | |
| block (GCD) | 區塊 | `^{ }` 包起來的工作 |
| blocking system call | 阻塞式系統呼叫 | |
| bound | 綁定 | |
| Callable | — | Java 介面，可回傳結果 |
| cancellation point | 取消點 | |
| cleanup handler | 清理處理常式 | |
| compiler directive | 編譯器指示 | 如 `#pragma` |
| concurrency / concurrent | 並行 | 投影片譯「同時發生」 |
| confer | 給予 | |
| context | 內容 | |
| CPU-intensive | CPU 密集 | |
| data dependency | 資料相依 | |
| data parallelism | 資料平行 | |
| data splitting | 資料分割 | |
| default signal handler | 預設訊號處理常式 | |
| deferred cancellation | 延遲取消 | 預設類型 |
| dispatch queue | 分派佇列 | |
| dividing activities | 劃分活動 | |
| economy | 經濟 | |
| ETHREAD | 執行體執行緒區塊 | Windows，kernel space |
| Executor | 執行器 | Java 介面 |
| explicit threading | 顯式執行緒 | |
| fork-join | 分叉–合併 | |
| Future | — | Java，代表尚未完成的結果 |

### G–P

| English | 中文 | 補充 |
|---|---|---|
| GCD (Grand Central Dispatch) | — | Apple 的技術 |
| green threads | 綠色執行緒 | many-to-one 的例子 |
| implicit threading | 隱式執行緒 | 投影片譯「內含的」 |
| interleaved | 交錯 | |
| intervention | 介入 | |
| join | 合併／等待結束 | |
| kernel thread | 核心執行緒 | |
| KTHREAD | 核心執行緒區塊 | Windows，kernel space |
| LWP (Lightweight Process) | 輕量級行程 | |
| main queue | 主佇列 | GCD 的 serial queue |
| main thread | 主執行緒 | |
| many-to-many model | 多對多模型 | |
| many-to-one model | 多對一模型 | |
| multicore | 多核心 | |
| multiplex | 多工 | |
| multithreaded | 多執行緒的 | |
| one-to-one model | 一對一模型 | |
| OpenMP | — | 平行程式設計 API |
| parallel region | 平行區域 | |
| parallelism / parallel | 平行 | |
| pending | 擱置 | |
| Pthreads | POSIX 執行緒 | IEEE 1003.1c |

### R–Z

| English | 中文 | 補充 |
|---|---|---|
| register set | 暫存器組 | |
| resource sharing | 資源共享 | |
| responsiveness | 回應性 | |
| Runnable | — | Java 介面 |
| scalability | 可擴展性 | |
| scheduler activation | 排程器啟動 | |
| serial queue | 循序佇列 | |
| signal | 訊號 | |
| signal handler | 訊號處理常式 | |
| single-threaded | 單執行緒的 | |
| specification vs. implementation | 規格 vs. 實作 | |
| synchronous signal | 同步訊號 | 即 exception |
| target thread | 目標執行緒 | |
| task | 任務 | Linux 對 process／thread 的統稱 |
| task parallelism | 任務平行 | |
| TBB (Threading Building Blocks) | 執行緒建構區塊 | Intel 的 C++ 樣板函式庫 |
| TEB | 執行緒環境區塊 | Windows，user space |
| template library | 樣板函式庫 | |
| thread | 執行緒 | |
| thread cancellation | 執行緒取消 | |
| thread ID | 執行緒識別碼 | |
| thread library | 執行緒函式庫 | |
| thread of control | 控制執行緒 | |
| thread pool | 執行緒池 | |
| throughput | 產能 | |
| TLS (Thread-Local Storage) | 執行緒區域儲存 | |
| two-level model | 二階層模型 | |
| upcall | 向上呼叫 | kernel → thread library |
| upcall handler | 向上呼叫處理常式 | |
| user thread | 使用者執行緒 | |
| user-defined signal handler | 使用者自訂訊號處理常式 | |
| variation | 變種 | |
| virtual processor | 虛擬處理器 | |
