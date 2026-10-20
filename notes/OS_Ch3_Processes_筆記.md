# 作業系統 Chapter 3：Processes（行程）筆記

> **教材**：*Operating System Concepts*, 10th ed.（Silberschatz, Galvin, Gagne）
> **投影片**：Mei-Ling Chiang 教授，暨南大學資管系（2026/09/29，共 107 頁）
> **筆記說明**：
> - 英文專有名詞第一次出現時以「**English（中文）**」標示，文末有完整詞彙表。
> - 標示「📌 投影片例子」者出自投影片；標示「💡 補充例子」者是為了幫助理解另外加的比喻或說明，不在投影片內。
> - 標示「⚠️」者是投影片中疑似筆誤或需要留意的地方。

---

## 目錄

- [0. 本章目標](#0-本章目標)
- [3.1 行程概念](#31-行程概念-process-concept)
- [3.2 行程排程](#32-行程排程-process-scheduling)
- [3.3 行程的操作](#33-行程的操作-operations-on-processes)
- [3.4 行程間通訊](#34-行程間通訊-interprocess-communication)
- [3.5 共享記憶體系統的 IPC](#35-共享記憶體系統的-ipc)
- [3.6 訊息傳遞系統的 IPC](#36-訊息傳遞系統的-ipc)
- [3.7 IPC 系統實例](#37-ipc-系統實例)
- [3.8 主從式系統的通訊](#38-主從式系統的通訊)
- [重點整理](#重點整理)
- [容易混淆的觀念對照](#容易混淆的觀念對照)
- [作業（Homework）](#作業homework)
- [英中詞彙總表](#英中詞彙總表)

---

## 0. 本章目標

- 介紹 **process（行程）** 的概念——**執行中的程式**，是所有運算的基礎。
- 描述 process 的各種特性：**排程（scheduling）、建立（creation）、終止（termination）**。
- 探討以 **shared memory** 與 **message passing** 進行的行程間通訊。
- 描述 **client-server** 系統中的通訊。

---

## 3.1 行程概念 (Process Concept)

**Process：a program in execution（執行中的程式）**。Process 需要資源：CPU time、memory、files、I/O devices。

### 從程式到行程

```
 program (原始碼)  ──compile & link──►  executable image  ──load into memory──►  process
                                         (可執行映像檔)                          (在 RAM 中)
```

**Executable image（可執行映像檔）** 的組成：

| 部分 | 中文 | 內容 |
|---|---|---|
| **Header** | 標頭 | entry point、執行檔大小、text section 的大小與起始位址、data section 的大小與起始位址 |
| **Text section** | 文字區段（程式碼區段） | **指令**；通常是**唯讀（read-only）**、可執行 |
| **Data section** | 資料區段 | 可讀寫（read/write） |

- **Entry point（進入點）**：text segment 中開始執行的位址。

> 📌 投影片例子：
> ```c
> #include <stdio.h>
> int a = 3;          // 全域變數 → data section
> void main()
> {                   // main 的指令 → text section
>     a = 5;
>     printf("a=%d", a);
> }
> ```

**Loading（載入）**：用二進位檔（executable image）中的區段初始化記憶體，並**額外加上**：
- **Stack（堆疊）**：stack frame、區域變數
- **Heap（堆積）**：動態配置的資料（例如 `malloc`）

所以一個 process（執行中的程式）**至少有 4 個區段**：**text、data、stack、heap**。

### 3.1.1 The Process

Process **不只是程式碼**（text section），還包含：

| 組成 | 中文 | 內容 |
|---|---|---|
| **Current activity** | 目前的活動 | **program counter（程式計數器）** 的值、處理器 **registers（暫存器）** 的內容 |
| **Process stack** | 行程堆疊 | 暫時性資料：函式**參數（parameters）**、**返回位址（return address）**、**區域變數（local variables）** |
| **Data section** | 資料區段 | **全域變數（global variables）** |
| **Heap** | 堆積 | 執行期間**動態配置**的記憶體 |

#### Process in Memory（Fig. 3.1）⭐

```
  max ┌──────────┐
      │  stack   │   ↓ 往低位址成長
      │    ↓     │
      │          │
      │    ↑     │
      │   heap   │   ↑ 往高位址成長
      ├──────────┤
      │   data   │
      ├──────────┤
      │   text   │
    0 └──────────┘
```

> 📌 投影片例子：各變數放在哪裡？
> ```c
> int a = 3;                    // a：全域變數 → data
> main()
> {
>     int ret;                  // ret：區域變數 → stack
>     char *ptr;                // ptr 這個指標本身：區域變數 → stack
>
>     ptr = malloc(1000);       // 配置的 1000 bytes → heap
>     ptr[10] = 'b';
>     ret = func(a, 5);         // 呼叫時在 stack 上建立新的 stack frame
>     printf("%d", ret);
>     return;
> }
>
> int func(int p1, int p2)      // p1, p2：參數 → stack
> {
>     int sum;                  // sum：區域變數 → stack
>     sum = p1 + p2;
>     return sum;
> }
> ```
> 呼叫 `func(a, 5)` 時，stack 上依序放入 **parameters**（3 和 5）、**return address**（回到 `main` 的哪一行）、**local variables**（`sum`），**SP（Stack Pointer，堆疊指標）** 隨之移動；函式返回時這個 frame 被移除。

**Process 是 active entity（主動實體）**：有一個 program counter 指出下一道要執行的指令，以及一組相關的資源。CPU 中的 `sp`、`pc`、`base`、`limit` 等暫存器記錄目前執行中 process 的狀態。

**其他重點**：
- 程式透過 GUI 滑鼠點擊、命令列輸入名稱等方式開始執行。
- **兩個 process 可以對應同一個程式**，但被視為兩個獨立的執行序列：text section 相同，但 **data、heap、stack 不同**。
- Process 執行時也可以 **spawn（衍生）** 出許多 process。
- Process 本身可以是**其他程式碼的執行環境**。

> 📌 投影片例子：**JVM（Java Virtual Machine）** 以一個 process 的形式執行，負責解譯載入的 Java 程式碼並代為採取動作。例如執行編譯好的 `Program.class`。層次：Java Program → JVM → OS。
> 💡 補充例子：同時開兩個記事本視窗，就是同一個程式 `notepad.exe` 的兩個 process，各自編輯不同的內容。

### 3.1.2 Process State（行程狀態）⭐

Process 執行時會改變**狀態**：

| 狀態 | 中文 | 說明 |
|---|---|---|
| **New** | 新建 | Process 正在被建立 |
| **Running** | 執行 | 指令正在被執行 |
| **Waiting** | 等待 | 等待某個事件發生（如 I/O 完成、收到 signal） |
| **Ready** | 就緒 | 等待被分配給處理器 |
| **Terminated** | 終止 | 執行完畢 |

**狀態轉換圖（Fig. 3.2）**：

```
   new ──admitted──►  ready  ◄──────interrupt───────  running ──exit──► terminated
                        │  ────scheduler dispatch────►   │
                        ▲                                │
         I/O or event   │                                │  I/O or event wait
          completion    └──────────  waiting  ◄──────────┘
```

| 轉換 | 觸發原因 |
|---|---|
| new → ready | **admitted**：被系統接受 |
| ready → running | **scheduler dispatch**：排程器選中並分派 CPU |
| running → ready | **interrupt**：被中斷（如時間片用完） |
| running → waiting | **I/O or event wait**：要等 I/O 或事件 |
| waiting → ready | **I/O or event completion**：等的事情完成了 |
| running → terminated | **exit**：執行結束 |

> ⭐ 注意：**waiting 不會直接回到 running**，一定先回 ready 再等排程。任一時刻每個 CPU core 上只有一個 process 是 running，但可以有很多個 ready 和 waiting。
>
> 💡 補充例子：去醫院看病——掛號（new）→ 候診區等叫號（ready）→ 進診間看診（running）→ 醫生叫你去抽血等報告（waiting）→ 報告出來後回候診區（ready）→ 再進診間（running）→ 看完離開（terminated）。

### 3.1.3 Process Control Block（PCB，行程控制區塊）⭐

- OS 用一個 **PCB** 代表每個 process，又稱 **task control block**。
- PCB 是一個**資料結構**，存放與特定 process 相關的資訊。

| 欄位 | 中文 | 內容 |
|---|---|---|
| **Process state** | 行程狀態 | new、ready、running、waiting、halted… |
| **Program counter** | 程式計數器 | 這個 process 下一道要執行的指令位址 |
| **CPU registers** | CPU 暫存器 | 發生中斷時必須連同 program counter 一起保存，之後才能正確繼續執行 |
| **CPU-scheduling information** | CPU 排程資訊 | priority（優先權）、指向排程佇列的指標、排程參數 |
| **Memory-management information** | 記憶體管理資訊 | base 與 limit 暫存器、page table、segment table… |
| **Accounting information** | 記帳資訊 | 已用時間、時間限制、帳號、process 編號… |
| **I/O status information** | I/O 狀態資訊 | 配置給此 process 的 I/O 裝置清單、已開啟檔案清單… |

> 💡 補充例子：PCB 像醫院裡每位病人的病歷。醫生換看下一位病人前，把目前病人的狀況記在病歷上；之後再輪到他時，翻開病歷就能從上次的地方繼續。

### 3.1.4 Threads（執行緒）

- 傳統上，一個 process 是執行**單一 thread of execution（執行緒）** 的程式，一次只能做一件事。
- 大多數現代 OS 擴充了 process 的概念，允許一個 process 有**多個執行緒**，一次能做多件事。
- 做法：每個 process 有**多個 program counter**，多個位置可以同時執行。
- 支援 thread 的系統中，**PCB 會擴充**以包含每個 thread 的資訊。
- 詳見 Chapter 4。

> 📌 投影片例子：**Linux 的 process 表示法**——C 結構 `task_struct`
> ```c
> pid_t pid;                    /* process identifier（行程識別碼） */
> long state;                   /* state of the process（狀態） */
> struct sched_entity se;       /* scheduling information（排程資訊） */
> struct task_struct *parent;   /* this process's parent（父行程） */
> struct list_head children;    /* this process's children（子行程串列） */
> struct files_struct *files;   /* list of open files（已開啟檔案） */
> struct mm_struct *mm;         /* address space of this process（位址空間） */
> ```
> 這就是 Linux 中 PCB 的實作。

---

## 3.2 行程排程 (Process Scheduling)

| | 目標 |
|---|---|
| **Multiprogramming** | 隨時都有 process 在執行，**最大化 CPU 使用率** |
| **Time sharing** | 讓 CPU 在 process 之間頻繁切換，使用者能在程式執行時與之**互動** |

**Process scheduler（行程排程器）**：從可用的 process 中選一個，放到某個 core 上執行。

### 3.2.1 Scheduling Queues（排程佇列）

| 佇列 | 中文 | 說明 |
|---|---|---|
| **Ready queue** | 就緒佇列 | Process 進入系統後放在這裡，已就緒、等待在 CPU core 上執行 |
| **Wait queue** | 等待佇列 | 等待某事件發生（如 I/O 完成）的 process 放在這裡 |

**實作（Fig. 3.4）**：佇列通常用 **linked list** 實作。queue header 有 `head` 與 `tail` 指標，把各個 **PCB** 串起來。

> 📌 投影片例子：ready queue：head → PCB₇ → PCB₂；wait queue：head → PCB₃ → PCB₁₄ → PCB₆。

**Queueing diagram（佇列圖，Fig. 3.5）**：

```
            ┌────────────────────────────────────────────────┐
            ▼                                                │
       ready queue ───dispatch──► CPU ──────────────────────►  (結束離開)
            ▲                      │
            ├── I/O ◄── I/O wait queue ◄── I/O request ──────┤
            ├───────────────────────── time slice expired ◄──┤
            ├── child terminates ◄── child termination ◄── create child process
            │                        wait queue              │
            └── interrupt occurs ◄── interrupt ◄── wait for an interrupt
                                     wait queue
```

Process 被分派到 CPU 後，可能發生四件事：

| 事件 | 結果 |
|---|---|
| 發出 **I/O request** | 進入 I/O wait queue，I/O 完成後回 ready queue |
| **Time slice expired**（時間片用完） | 直接回 ready queue |
| **Create child process** 並等它結束 | 進入 child termination wait queue |
| **Wait for an interrupt** | 進入 interrupt wait queue |

### 3.2.2 CPU Scheduling（CPU 排程）

- Process 在生命週期中會在各個排程佇列間**遷移（migrate）**。
- **CPU scheduler（CPU 排程器）**：
  - 從 ready queue 選一個 process，分配 CPU core 給它。
  - **執行非常頻繁**（例如至少每 100 毫秒一次），所以**必須很快**。

**Swapping（置換）**——一種中間形式的排程：

- 把 process 從記憶體移出（也就退出對 CPU 的競爭），藉此**降低 degree of multiprogramming（多元程式度）**，即記憶體中 process 的數量。
- **Swap out**：把 process 從記憶體移到磁碟，保存目前狀態。
- **Swap in**：之後從磁碟搬回記憶體，恢復狀態。
- 通常只在記憶體被 **overcommitted（過度配置）** 而必須釋放時才需要。

> 📌 投影片例子：Process A 佔用的記憶體被 swap out 到磁碟的 4–7 號區塊；Process B 從磁碟的 17–19 號區塊 swap in 回主記憶體。

### 3.2.3 Context Switch（內容切換／上下文切換）⭐

- CPU 切換到另一個 process 時，系統必須**保存舊 process 的狀態**，並**載入新 process 先前保存的狀態**。這個動作叫 **context switch**。
- **Context（內容）** 記錄在 process 的 **PCB** 中：CPU 暫存器的值、process state、記憶體管理資訊。
- Context-switch 的時間是**純粹的 overhead（額外負擔）**：切換期間系統**不做任何有用的工作**。
  - 典型速度是**幾微秒（microseconds）**。
- 切換時間高度取決於**硬體支援**。

**CPU 從 P0 切換到 P1 的過程**：

```
   Process P0            Operating System             Process P1
   executing                                            idle
       │      ── interrupt or system call ──►
       ▼           save state into PCB0
     idle                   ⋮
                   reload state from PCB1  ──────────► executing
                                                           │
              ◄── interrupt or system call ──              ▼
                   save state into PCB1                  idle
                            ⋮
   executing ◄──── reload state from PCB0
```

> 💡 補充例子：你同時寫兩科作業。換科目時要把目前的課本收起來、記下寫到第幾題（save），再拿出另一科課本、翻到上次的地方（reload）。這段時間兩科都沒有進度，換得越頻繁浪費越多。

**補充背景：CPU 如何執行指令**
- 📌 投影片例子：8086 微處理器的處理器模型，以及 **SIC（Simplified Instructional Computer）**——CPU 內有暫存器 A、X、L、**PC**（program counter）、SW，以及 CU（控制單元）與 ALU（算術邏輯單元）。
- 執行每道指令的三步驟：**1. Fetch（擷取）→ 2. Decode（解碼）→ 3. Execution（執行）**。
- 這些暫存器的內容就是 context switch 時要保存與恢復的東西：OS 為每個 process 維護一個 PCB（存 pc、sp…），切換時把 CPU 暫存器存進舊 PCB，再從新 PCB 載入。

---

## 3.3 行程的操作 (Operations on Processes)

### 3.3.1 Process Creation（行程建立）

**OS 建立 process 的步驟**（對應狀態 new → ready）：

1. **配置記憶體**
2. 把程式的 **executable image 載入**記憶體
3. 設定 **PCB**
   - 管理與執行中程式相關的資料結構
   - 設定 CPU context（PC、SP、registers）
   - **PC 設為 entry point**
4. 把新 process 放入 **ready queue**（把它的 PCB 串到 ready queue）

#### Process Tree（行程樹）

- Process 執行時可以建立多個新 process。
- 建立者稱為 **parent process（父行程）**，被建立的是 **child process（子行程）**；子行程又可以建立其他 process，形成**樹狀結構**。
- Process 通常以 **pid（process identifier，行程識別碼）** 來識別與管理。

> 📌 投影片例子：典型 Linux 系統的 process tree（Fig. 3.7）
> ```
>                    systemd (pid=1)
>            ┌────────────┼────────────┐
>      logind (8415)  python (2808)  sshd (3028)
>            │                          │
>       bash (8416)                 sshd (3610)
>        ┌───┴───┐                      │
>    ps (9298) vim (9204)           tcsh (4005)
> ```
> `systemd`（pid = 1）是所有使用者 process 的根。
> 💡 補充例子：在 Linux 輸入 `pstree` 或 `ps -ef` 就能看到自己系統的 process tree。

#### Parent 與 Child 的關係

| 面向 | 可能的選擇 |
|---|---|
| **Resource sharing（資源共享）** | ① parent 與 children 共享**所有**資源<br>② children 共享 parent 資源的**子集合**<br>③ parent 與 child **不共享**資源 |
| **Execution（執行）** | ① parent 與 children **並行（concurrently）** 執行<br>② parent **等待**部分或全部 children 結束 |
| **Address space（位址空間）** | ① child 是 parent 的**複製品（duplicate）**<br>② child 載入一個**新程式** |

#### UNIX：`fork()` 與 `exec()` ⭐

| System call | 作用 |
|---|---|
| **`fork()`** | 建立一個新 process，是 parent 的**複製品** |
| **`exec()`** | 在 `fork()` 之後使用，把 process 的記憶體空間**換成一個新程式** |
| **`wait()`** | parent 等待 child 結束 |

**`fork()` 的回傳值**——分辨自己是誰的關鍵：

| 回傳值 | 意義 |
|---|---|
| `< 0` | fork 失敗 |
| `== 0` | 我是 **child** |
| `> 0` | 我是 **parent**，回傳值是 child 的 pid |

> 📌 投影片例子（Fig. 3.8）：用 `fork()` 建立獨立的 process
> ```c
> #include <sys/types.h>
> #include <stdio.h>
> #include <unistd.h>
>
> int main()
> {
>     pid_t pid;
>
>     pid = fork();                    /* fork a child process */
>
>     if (pid < 0) {                   /* error occurred */
>         printf("Fork Failed");
>         exit(-1);
>     }
>     else if (pid == 0) {             /* child process */
>         execlp("/bin/ls", "ls", NULL);
>     }
>     else {                           /* parent process */
>         wait(NULL);                  /* wait for child to complete */
>         printf("child complete");
>     }
>     return 0;
> }
> ```
> Child 執行的程式碼與 parent **相同**，差別只在 `fork()` 對 child 回傳 0。
>
> **執行流程（Fig. 3.9）**：
> ```
>                         ┌─ parent (pid > 0) ── wait() ─────────────► parent resumes
>   parent ── fork() ─────┤                         ▲
>                         └─ child (pid = 0) ── exec() ── exit() ──────┘
> ```

**從記憶體的角度看**（投影片的三張圖）：

| 階段 | 記憶體中的情況 |
|---|---|
| 1. 呼叫 `fork()` | OS 建立 **PCB2**，並把 process 1 的內容**複製**成 process 2。兩者程式碼相同 |
| 2. `fork()` 返回 | 兩個 process 都從 `fork()` **之後的下一行**繼續執行，各有自己的 pc、sp。Parent 走 `else` 去 `wait()`；child 走 `if (pid == 0)` |
| 3. Child 呼叫 `execve("a")` | Process 2 的記憶體內容被**換成程式 "a"**，從頭開始執行。Process 1 不受影響 |

> 💡 補充例子：`fork()` 像細胞分裂——分裂後有兩個一模一樣的細胞，都記得分裂前的一切，只有「我是原本的還是新的」這件事不同。`exec()` 則像新細胞把自己的 DNA 整個換掉，變成另一種細胞。
>
> 💡 補充：`exec()` 成功後**不會返回**，因為原本的程式碼已經被換掉了。所以寫在 `execlp()` 後面的程式碼，只有在 `exec` **失敗**時才會執行到。

#### Windows：`CreateProcess()`

> 📌 投影片例子（Fig. 3.10）：用 Win32 API 建立獨立的 process
> ```c
> #include <windows.h>
> #include <stdio.h>
>
> int main(VOID)
> {
>     STARTUPINFO si;
>     PROCESS_INFORMATION pi;
>
>     // Allocate memory
>     ZeroMemory(&si, sizeof(si));
>     si.cb = sizeof(si);
>     ZeroMemory(&pi, sizeof(pi));
>
>     // Create child process
>     if (!CreateProcess(NULL,                              // No module name (use command line)
>             "C:\\WINDOWS\\system32\\mspaint.exe",         // Command line
>             NULL,          // Process handle not inheritable
>             NULL,          // Thread handle not inheritable
>             FALSE,         // Set handle inheritance to FALSE
>             0,             // No creation flags
>             NULL,          // Use parent's environment block
>             NULL,          // Use parent's starting directory
>             &si,           // Pointer to STARTUPINFO structure
>             &pi))          // Pointer to PROCESS_INFORMATION structure
>     {
>         printf("CreateProcess failed\n");
>         return -1;
>     }
>
>     // parent will wait for the child to complete
>     WaitForSingleObject(pi.hProcess, INFINITE);
>     printf("Child Complete");
>
>     // Close process and thread handles
>     CloseHandle(pi.hProcess);
>     CloseHandle(pi.hThread);
> }
> ```
> 這個程式會開啟小畫家（mspaint.exe），等它關閉後印出 "Child Complete"。

| | **UNIX** | **Windows** |
|---|---|---|
| 建立 | `fork()`：先複製 parent | `CreateProcess()`：建立時就**直接載入指定程式** |
| 載入新程式 | 另外呼叫 `exec()` | 已包含在 `CreateProcess()` 中 |
| 參數 | `fork()` 不需要參數 | 至少 10 個參數 |
| 等待 | `wait()` | `WaitForSingleObject()` |

### 3.3.2 Process Termination（行程終止）

**Process 自己結束**（`exit()` system call）：
- 執行完最後一道敘述，要求 OS 刪除自己。
- Child 可以回傳一個**狀態值（status value）** 給 parent（parent 透過 `wait()` 取得）。
- Process 的資源由 OS **回收（deallocate）**。

**Parent 終止 children**（例如 Windows 的 `TerminateProcess()`），原因可能是：
- Child **超用**了被配置的資源。
- 指派給 child 的工作**不再需要**。
- Parent 要結束，而有些 OS 不允許 parent 終止後 child 繼續存在。
  - 所有 children 都被終止——**cascading termination（連鎖終止／層疊式終止）**。

#### UNIX／Linux 的終止

```c
pid_t pid;
int status;
pid = wait(&status);    // 等待 child 終止，回傳該 child 的 pid
```

| 名詞 | 中文 | 定義 | 處理方式 |
|---|---|---|---|
| **Zombie process** | 殭屍行程 | Process **已經終止**，但 parent **還沒呼叫 `wait()`** | Parent 呼叫 `wait()` 後，zombie 的 pid 與 process table 項目才被釋放 |
| **Orphan process** | 孤兒行程 | **Parent 先終止了**，留下還在執行的 children | 把 **`init` process** 指定為孤兒的新 parent；`init` 會**定期呼叫 `wait()`** |

> 💡 補充例子：
> - Zombie：學生考完交卷離開了（已終止），但老師還沒登記成績（parent 沒 `wait()`），所以座位表上還留著他的名字。
> - Orphan：父母先離開了，孩子由 `init`（現代 Linux 是 `systemd`）收養。
>
> 為什麼需要 zombie 這個狀態？因為 child 的結束狀態要留著等 parent 來讀取。

---

## 3.4 行程間通訊 (Interprocess Communication)

並行執行的 process 分兩種：

| 類型 | 中文 | 定義 |
|---|---|---|
| **Independent process** | 獨立行程 | **不會**影響其他 process，也不受影響 |
| **Cooperating process** | 合作行程 | **會**影響其他 process 或受其影響；任何與其他 process **共享資料**的 process 都是 |

**需要 cooperating process 的原因**：

| 原因 | 中文 | 說明 |
|---|---|---|
| **Information sharing** | 資訊共享 | 多個應用程式可能需要同一份資訊 |
| **Computation speedup** | 加速運算 | 把工作拆成子工作平行執行（需要多個處理核心） |
| **Modularity** | 模組化 | 把系統功能分成獨立的 process |

> 📌 投影片例子：
> - **Producer–consumer**：producer process → buffer → consumer process。
> - **Parallel computing（平行運算）**：兩個 CPU core 各處理 `int array[N]` 的一半。

Cooperating process 需要 **IPC（Interprocess Communication，行程間通訊）** 機制來交換資料。**兩種基本模型**：

| | **(1) Shared memory（共享記憶體）** | **(2) Message passing（訊息傳遞）** |
|---|---|---|
| 做法 | 建立一塊由合作 process 共享的記憶體區域，透過**讀寫**該區域交換資訊 | 透過在合作 process 之間**交換訊息**來通訊 |
| 是否經過 kernel | 只有**建立**共享區域時需要 system call；建立後所有存取都是一般的記憶體存取，**不需 kernel 協助** | 通常用 **system call** 實作，每次都需要 **kernel 介入** |
| 速度 | **較快**（以記憶體傳輸速度進行） | 較慢 |
| 適合 | 同一台電腦內、大量資料 | **少量資料**（不需避免衝突） |
| 分散式系統 | 較難 | **較容易實作**（跨電腦通訊） |
| 注意事項 | 需要自行處理 **protection 與 synchronization** | — |

---

## 3.5 共享記憶體系統的 IPC

- 要通訊的 process 必須**建立一塊共享記憶體區域**。
- 共享記憶體區域通常位於**建立它的 process 的位址空間**中。
- 其他想用它通訊的 process 必須把它 **attach（附加）** 到自己的位址空間。
- 使用類似 `shared_memory_create()` 與 `shared_memory_attach()` 的 system call。
- 一般情況 OS 會阻止 process 存取別人的記憶體，所以需要**兩個以上的 process 同意移除這項保護限制**。
- 資料的**格式與位置由這些 process 決定**，不受 OS 控制。
- Process 也要負責確保**不會同時寫入同一位置**——需要 protection 與 synchronization（同步問題詳見 Chapter 6、7）。

### Producer–Consumer Problem（生產者–消費者問題）⭐

合作 process 的**典範（paradigm）**：**producer（生產者）** process 產生資訊，由 **consumer（消費者）** process 消耗。

> 📌 投影片例子：web server（producer）產生網頁內容，web browser（consumer）讀取。

兩者之間需要一個 **buffer（緩衝區）**：

| 類型 | 中文 | 說明 |
|---|---|---|
| **Unbounded buffer** | 無限緩衝區 | 大小沒有限制。Producer 永遠可以放；consumer 可能要等 |
| **Bounded buffer** | 有限緩衝區 | 固定大小。**滿了 producer 要等；空了 consumer 要等** |

Buffer 可以由 OS 透過 message-passing 設施提供，也可以由程式設計師用 shared memory 自行撰寫。

### Bounded-Buffer 的 Shared-Memory 解法

**共享資料**——用**環狀陣列（circular array）** 實作：

```c
#define BUFFER_SIZE 10

typedef struct {
    ...
} item;

item buffer[BUFFER_SIZE];
int in = 0;      // 下一個空位（next free position）
int out = 0;     // 第一個有資料的位置（first full position）
```

| 狀態 | 判斷條件 |
|---|---|
| **Empty（空）** | `in == out` |
| **Full（滿）** | `((in + 1) % BUFFER_SIZE) == out` |

**Producer process**：
```c
item next_produced;

while (true) {
    /* produce an item in next_produced */

    while (((in + 1) % BUFFER_SIZE) == out)
        ;   /* do nothing —— buffer 滿了，空轉等待 */

    buffer[in] = next_produced;
    in = (in + 1) % BUFFER_SIZE;
}
```

**Consumer process**：
```c
item next_consumed;

while (true) {
    while (in == out)
        ;   /* do nothing —— buffer 空了，空轉等待 */

    next_consumed = buffer[out];
    out = (out + 1) % BUFFER_SIZE;

    /* consume the item in next_consumed */
}
```

> 📌 投影片例子：逐步追蹤（BUFFER_SIZE = 10，格子編號 0–9）
>
> | 動作 | in | out | 項目數 | 狀態 |
> |---|---|---|---|---|
> | 初始 | 0 | 0 | 0 | **empty**（in == out） |
> | 生產 1 個 | 1 | 0 | 1 | |
> | 再生產 1 個 | 2 | 0 | 2 | |
> | 持續生產到第 9 個 | 9 | 0 | 9 | **full**（(9+1) % 10 == 0） |
> | 消費 1 個 | 9 | 1 | 8 | |
> | 再消費 1 個 | 9 | 2 | 7 | |
> | 生產 1 個（放在格子 9，in 繞回 0） | 0 | 2 | 8 | |
> | 再生產 1 個 | 1 | 2 | 9 | **full**（(1+1) % 10 == 2） |

**投影片的提問：這個解法正確，但 buffer 最多只能放 BUFFER_SIZE − 1 個項目。為什麼？**

因為只靠 `in` 和 `out` 兩個變數來判斷空與滿。如果允許 10 格全部放滿，`in` 會繞一圈追上 `out`，變成 `in == out`——**跟「空」的條件一模一樣，無法分辨**。所以必須故意留一格不用，讓「滿」的條件變成「`in` 的下一格是 `out`」。

（💡 補充：若想用滿全部 BUFFER_SIZE 格，可以另外加一個計數變數 `count` 記錄目前項目數；但兩個 process 同時修改 `count` 會產生同步問題，這是後面章節討論 race condition 的經典例子。）

---

## 3.6 訊息傳遞系統的 IPC

- Shared memory 的做法要求 process 共享一塊記憶體，且存取與操作的程式碼要由**程式設計師明確撰寫**。
- 另一種方式：由 **OS 提供** message-passing 設施讓合作 process 通訊。
- Message passing 讓 process **不需共享同一位址空間**就能通訊並同步彼此的動作。
  - 對位於**不同電腦**上的 process 特別有用。
  - 📌 投影片例子：Internet 聊天程式。

**至少提供兩個操作**：
- **`send(message)`**——訊息大小可以固定或可變
- **`receive(message)`**

若 P 與 Q 要通訊，需要：
1. 在兩者間建立 **communication link（通訊鏈結）**
2. 用 send / receive 交換訊息

**邏輯上實作 link 與 send/receive 的三個面向**：

| 面向 | 選擇 | 章節 |
|---|---|---|
| Naming | **Direct** 或 **indirect** communication | 3.6.1 |
| Synchronization | **Synchronous** 或 **asynchronous** communication | 3.6.2 |
| Buffering | **Automatic** 或 **explicit** buffering | 3.6.3 |

> 📌 投影片例子：用 message passing 解 producer–consumer 問題
> ```c
> /* producer */                        /* consumer */
> message next_produced;                message next_consumed;
>
> while (true) {                        while (true) {
>     /* produce an item in                 receive(&next_consumed);
>        next_produced */                   /* consume the item in
>     send(next_produced);                     next_consumed */
> }                                     }
> ```
> 比 shared-memory 版本簡單很多——buffer 的管理、滿了要等、空了要等，全部由 OS 處理。

### 3.6.1 Naming（命名）

要通訊的 process 必須有辦法**指稱對方**。

#### Direct Communication（直接通訊）

Process 必須**明確指名**對方：
- `send(P, message)`——送訊息給 process P
- `receive(Q, message)`——從 process Q 接收訊息

**Link 的特性**：
- 每一對想通訊的 process 之間**自動**建立 link。
- 一條 link **剛好**關聯兩個 process。
- 每對 process 之間**剛好只有一條** link。

**定址方式**：

| | Sender | Receiver |
|---|---|---|
| **Symmetric（對稱）** | `send(P, message)` | `receive(Q, message)`——指名從 Q 收 |
| **Asymmetric（非對稱）** | `send(P, message)` | `receive(id, message)`——從**任何** process 收，`id` 會被設為發送者 |

**缺點**：模組化程度有限。**改變某個 process 的識別碼**，就得檢查並修改所有其他提到舊識別碼的 process 定義——識別碼被**寫死（hard-coded）** 在程式裡。

#### Indirect Communication（間接通訊）

- 訊息送到 **mailbox（信箱）**（也稱 **port，埠**），並從中接收。
- 每個 mailbox 有**唯一的 id**。
- Process 只有在**共享同一個 mailbox** 時才能通訊。
- `send(A, message)`——送訊息到 mailbox A
- `receive(A, message)`——從 mailbox A 接收訊息

**Link 的特性**：
- 只有當一對 process 共享 mailbox 時才建立 link。
- 一條 link 可以關聯**兩個以上**的 process。
- 每對 process 之間可以有**多條** link，每條對應一個 mailbox。

**Mailbox sharing 的問題**：
> P1、P2、P3 共享 mailbox A。P1 送出訊息，P2 和 P3 都執行 receive——**誰收到？**

解法：
1. 一條 link 最多只關聯**兩個** process。
2. 同一時間最多只允許**一個** process 執行 receive。
3. 讓系統**任意選擇**接收者，並通知 sender 是誰收到的。

**Mailbox 的擁有者**：

| 擁有者 | 說明 |
|---|---|
| **Process** | **Owner**：只能從 mailbox **接收**訊息。**User**：只能**傳送**訊息到 mailbox |
| **OS** | OS 必須提供機制讓 process：建立新 mailbox、透過 mailbox 傳送與接收、刪除 mailbox |

| | **Direct** | **Indirect** |
|---|---|---|
| 指定對象 | 對方的 process 名稱 | mailbox 的 id |
| 一條 link 關聯 | 剛好 2 個 process | 可以 2 個以上 |
| 每對 process 的 link 數 | 剛好 1 條 | 可以多條 |

> 💡 補充例子：Direct 像打電話給特定的人——你必須知道他的號碼，他換號碼你就找不到。Indirect 像寄信到公司的客服信箱——你不需要知道是哪位客服處理，換人了也沒關係。

### 3.6.2 Synchronization（同步）

Message passing 可以是 **blocking（阻塞）** 或 **nonblocking（非阻塞）**。

| | **Blocking（＝ synchronous，同步）** | **Nonblocking（＝ asynchronous，非同步）** |
|---|---|---|
| **send** | Sender **被擋住**，直到訊息被 receiver 或 mailbox 收到 | Sender 送出訊息後**繼續執行** |
| **receive** | Receiver **被擋住**，直到有訊息可收 | Receiver 立刻取得一則有效訊息，或得到 **null** |

- 當 `send()` 與 `receive()` **都是 blocking** 時，sender 與 receiver 之間形成 **rendezvous（會合）**。

> 💡 補充例子：Blocking send 像打電話，要等對方接起來才能講；nonblocking send 像傳 LINE，送出後就去做別的事。Blocking receive 像守在信箱前等信；nonblocking receive 像經過時看一眼，沒信就走。

### 3.6.3 Buffering（緩衝）

附屬於 link 的**訊息佇列**，有三種實作方式：

| 類型 | 容量 | Sender 的行為 | 分類 |
|---|---|---|---|
| **Zero capacity（零容量）** | 佇列長度 = 0 | **必須 block**，直到 recipient 收下訊息 | **No buffering** |
| **Bounded capacity（有限容量）** | 有限長度 n | 佇列**滿了**才要等 | **Automatic buffering** |
| **Unbounded capacity（無限容量）** | 無限長度 | **永遠不會 block** | **Automatic buffering** |

---

## 3.7 IPC 系統實例

### 3.7.1 POSIX Shared Memory

| 步驟 | 函式 | 說明 |
|---|---|---|
| 1. 建立共享記憶體區段 | `shm_fd = shm_open(name, O_CREAT \| O_RDWR, 0666);` | 也用來開啟既有的區段 |
| 2. 設定物件大小 | `ftruncate(shm_fd, 4096);` | |
| 3. 建立 **memory-mapped file（記憶體映射檔）** | `ptr = mmap(0, SIZE, PROT_READ \| PROT_WRITE, MAP_SHARED, shm_fd, 0);` | 把共享記憶體物件映射到位址空間 |
| 4. 當成一般記憶體存取 | `sprintf(ptr, "%s", message_0);` | 用 `mmap()` 回傳的指標 |
| 5. 從系統移除 | `shm_unlink(name);` | |

> ⚠️ 投影片第 75 頁的說明把旗標寫成 `O_RDRW`，應為 **`O_RDWR`**（read-write）；後面第 76 頁的完整程式碼是正確的。

> 📌 投影片例子（Fig. 3.16）：**Producer**——寫入共享記憶體
> ```c
> #include <stdio.h>
> #include <stdlib.h>
> #include <string.h>
> #include <fcntl.h>
> #include <sys/shm.h>
> #include <sys/stat.h>
> #include <sys/mman.h>
>
> int main()
> {
>     const int SIZE = 4096;
>     const char *name = "OS";            // name of shared memory object
>     const char *message_0 = "Hello";
>     const char *message_1 = "World!";
>     int fd;                             // shared memory file descriptor
>     char *ptr;                          // pointer to shared memory object
>
>     /* create the shared memory object */
>     fd = shm_open(name, O_CREAT | O_RDWR, 0666);
>
>     /* configure the size of the shared memory object */
>     ftruncate(fd, SIZE);
>
>     /* now map the shared memory object in the address space of the process */
>     ptr = (char *) mmap(0, SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
>     if (ptr == MAP_FAILED) {
>         printf("Map failed\n");
>         return -1;
>     }
>
>     /* write to the shared memory object */
>     sprintf(ptr, "%s", message_0);
>     ptr += strlen(message_0);
>     sprintf(ptr, "%s", message_1);
>     ptr += strlen(message_1);
>
>     return 0;
> }
> ```
>
> 📌 投影片例子（Fig. 3.17）：**Consumer**——讀取共享記憶體
> ```c
> #include <stdio.h>
> #include <stdlib.h>
> #include <fcntl.h>
> #include <sys/shm.h>
> #include <sys/stat.h>
> #include <sys/mman.h>
>
> int main()
> {
>     const char *name = "OS";            // name of shared memory object
>     const int SIZE = 4096;
>     int fd;                             // shared memory file descriptor
>     char *ptr;                          // pointer to shared memory object
>     int i;
>
>     /* open the shared memory object */
>     fd = shm_open(name, O_RDONLY, 0666);
>     if (fd == -1) {
>         printf("shared memory failed\n");
>         exit(-1);
>     }
>
>     /* now map the shared memory object in the address space of the process */
>     ptr = (char *) mmap(0, SIZE, PROT_READ, MAP_SHARED, fd, 0);
>     if (ptr == MAP_FAILED) {
>         printf("Map failed\n");
>         exit(-1);
>     }
>
>     /* now read from the shared memory object */
>     printf("%s", (char *)ptr);
>
>     /* remove the shared memory object */
>     if (shm_unlink(name) == -1) {
>         printf("Error removing %s\n", name);
>         exit(-1);
>     }
>     return 0;
> }
> ```
> **重點**：兩個程式用**同一個名稱 `"OS"`** 找到同一塊共享記憶體。Producer 寫入 "Hello" 和 "World!"，consumer 讀出後印出 `HelloWorld!`，然後移除該區段。
>
> 💡 補充：在 Linux 編譯這類程式，較舊的 glibc 需要加 `-lrt`（例如 `gcc producer.c -o producer -lrt`）；程式中用到 `ftruncate()` 另需 `#include <unistd.h>`。

### 3.7.2 Mach Message Passing

- Mach kernel 支援建立與銷毀多個 **task**（類似 process），每個 task 有多個 thread。
- Mach 中大部分通訊都透過**訊息**進行。訊息送到 mailbox 並從中接收，mailbox 在 Mach 中稱為 **port（埠）**。
- 每個 task 建立時，同時建立兩個 mailbox：
  - **Kernel mailbox**：kernel 用來與 task 通訊。
  - **Notify mailbox**：kernel 用來通知事件發生。

**訊息傳輸只需三個 system call**：

| System call | 作用 |
|---|---|
| `msg_send()` | 傳送訊息 |
| `msg_receive()` | 接收訊息 |
| `msg_rpc()` | 傳送一則訊息，並等待 receiver **剛好一則**回覆 |
| `port_allocate()` | 建立新 mailbox 並配置其訊息佇列的空間 |

> 📌 投影片例子：Mach 訊息的結構與收發
> ```c
> #include <mach/mach.h>
>
> struct message {
>     mach_msg_header_t header;
>     int data;
> };
>
> mach_port_t client;
> mach_port_t server;
> ```
> **Client 端**——送出訊息：
> ```c
> struct message message;
>
> // construct the header
> message.header.msgh_size = sizeof(message);
> message.header.msgh_remote_port = server;
> message.header.msgh_local_port = client;
>
> // send the message
> mach_msg(&message.header,        // message header
>          MACH_SEND_MSG,          // sending a message
>          sizeof(message),        // size of message sent
>          0,                      // maximum size of received message - unnecessary
>          MACH_PORT_NULL,         // name of receive port - unnecessary
>          MACH_MSG_TIMEOUT_NONE,  // no time outs
>          MACH_PORT_NULL);        // no notify port
> ```
> **Server 端**——接收訊息：
> ```c
> struct message message;
>
> // receive the message
> mach_msg(&message.header,        // message header
>          MACH_RCV_MSG,           // receiving a message
>          0,                      // size of message sent
>          sizeof(message),        // maximum size of received message
>          server,                 // name of receive port
>          MACH_MSG_TIMEOUT_NONE,  // no time outs
>          MACH_PORT_NULL);        // no notify port
> ```
> 同一個函式 `mach_msg()` 靠第二個參數（`MACH_SEND_MSG` / `MACH_RCV_MSG`）決定是送還是收。
> ⚠️ 投影片 server 端 `MACH_RCV_MSG` 那行的註解寫的是 "sending a message"，是沿用課本的筆誤，實際意思是**接收**。

**Send 與 receive 很有彈性**：

- Mailbox **沒滿**：訊息被複製進去，sending thread 繼續執行。
- Mailbox **滿了**，sending thread 有四個選擇：
  1. **無限期等待**，直到 mailbox 有空間。
  2. **最多等 n 毫秒**。
  3. **完全不等**，立刻返回。
  4. **暫時快取**一則訊息（OS 只多保留一則）。
- 沒有訊息可收時，receiving thread 可以：等待、最多等 n 毫秒、或不等。

**效能問題與解法**：
- Mach 特別為**分散式系統**設計，但也適用於處理核心較少的系統。
- 訊息系統的主要問題是**效能差**，因為要把訊息從 sender 的 port **複製**到 receiver 的 port。
- Mach 用**虛擬記憶體管理技術**避免複製：把含有 sender 訊息的位址空間**映射（map）** 到 receiver 的位址空間，訊息本身從未真正被複製。
  - 只適用於**系統內部（intrasystem）** 的訊息。
  - 相關技術：**copy-on-write**、shared memory。

### 3.7.3 Windows — ALPC

- Windows 支援多種操作環境，稱為 **subsystem（子系統）**。
- 應用程式透過 message-passing 機制與 subsystem 通訊；應用程式可視為 Windows subsystem server 的 **client**。
- Windows 的 message-passing 設施稱為 **ALPC（Advanced Local Procedure Call，進階本地程序呼叫）**。
- **只能用於同一系統上的 process 之間**。
- 用 **port**（類似 mailbox）建立並維護通訊管道。

**通訊流程（Fig. 3.19）**：

| 步驟 | 動作 |
|---|---|
| 1 | Client 開啟 subsystem 的 **connection port** 物件的 **handle（控制代碼）** |
| 2 | Client 送出 **connection request（連線請求）** |
| 3 | Server 建立**兩個私有的 communication port**（client communication port 與 server communication port），並把其中一個的 handle 回傳給 client |
| 4 | Client 與 server 用各自的 port handle 傳送訊息或 **callback**，並監聽回覆 |

圖中還有 **shared section object（> 256 bytes）**：
（💡 補充：較小的訊息直接透過 port 的訊息佇列複製；較大的訊息則透過 section object 這塊共享記憶體傳遞，避免複製大量資料。）

### 3.7.4 Pipes（管道）

**Pipe** 像一條**導管（conduit）**，讓兩個 process 通訊（源自 UNIX）。設計 pipe 時要考慮四個問題：

1. **Unidirectional（單向）** 還是 **bidirectional（雙向）** 通訊？
2. 若是雙向，是 **half duplex（半雙工**：同一時間資料只能往一個方向）還是 **full duplex（全雙工**：資料可同時雙向傳輸）？
3. 通訊的 process 之間是否必須有 **parent–child 關係**？
4. 能否**跨網路**通訊？

#### 3.7.4.1 Ordinary Pipes（普通管道）

- **單向**通訊；**需要 parent–child 關係**。
- UNIX：用 `read()` 與 `write()` system call 存取。
- Windows：稱為 **anonymous pipe（匿名管道）**，用 `ReadFile()` 與 `WriteFile()` 存取。
- `pipe(fd)` 建立後：**`fd[0]` 是 read end（讀取端）**，**`fd[1]` 是 write end（寫入端）**。

> 📌 投影片例子（Fig. 3.21、3.22）：UNIX 的 ordinary pipe
> ```c
> #include <stdio.h>
> #include <unistd.h>
> #include <sys/types.h>
> #include <string.h>
>
> #define BUFFER_SIZE 25
> #define READ_END    0
> #define WRITE_END   1
>
> int main(void)
> {
>     char write_msg[BUFFER_SIZE] = "Greetings";
>     char read_msg[BUFFER_SIZE];
>     pid_t pid;
>     int fd[2];
>
>     /* create the pipe */
>     if (pipe(fd) == -1) {
>         fprintf(stderr, "Pipe failed");
>         return 1;
>     }
>
>     /* fork a child process */
>     pid = fork();
>
>     if (pid < 0) {
>         fprintf(stderr, "Fork failed");
>         return 1;
>     }
>
>     if (pid > 0) {   /* parent process */
>         /* close the unused end of the pipe */
>         close(fd[READ_END]);
>
>         /* write to the pipe */
>         write(fd[WRITE_END], write_msg, strlen(write_msg) + 1);
>
>         /* close the write end of the pipe */
>         close(fd[WRITE_END]);
>     }
>     else {           /* child process */
>         /* close the unused end of the pipe */
>         close(fd[WRITE_END]);
>
>         /* read from the pipe */
>         read(fd[READ_END], read_msg, BUFFER_SIZE);
>         printf("child read %s\n", read_msg);
>
>         /* close the read end of the pipe */
>         close(fd[READ_END]);
>     }
>     return 0;
> }
> ```
> **流程**：先 `pipe()` 再 `fork()`，所以 child **繼承**了 pipe 的兩端。Parent 關掉讀取端、寫入 "Greetings"；child 關掉寫入端、讀出後印出 `child read Greetings`。
> **為什麼要先關掉不用的那一端？** 因為 pipe 是單向的，各自只用一端；也讓讀取方在寫入方關閉後能偵測到 end-of-file。
> ⚠️ 投影片 child 最後一行的註解寫 "close the write end"，實際關閉的是 **read end**（`fd[READ_END]`）。
>
> 💡 補充例子：shell 裡的 `ls | grep txt` 中的 `|` 就是 pipe——`ls` 的輸出接到 `grep` 的輸入。

#### 3.7.4.2 Named Pipes（具名管道）

- **雙向**通訊；**不需要 parent–child 關係**。
- 建立後多個 process 都可以使用；通訊的 process 結束後它仍然存在。

| | **UNIX**（稱為 **FIFO**） | **Windows** |
|---|---|---|
| 雙工 | **Half-duplex** | **Full-duplex** |
| 位置 | 通訊的 process 必須在**同一台機器** | **不必**在同一台機器 |

#### Ordinary vs. Named Pipes

| | **Ordinary pipe** | **Named pipe** |
|---|---|---|
| 方向 | 單向 | 雙向 |
| Parent–child 關係 | **需要** | **不需要** |
| 存在時間 | 通訊的 process 結束就消失 | 持續存在，直到被明確刪除 |

---

## 3.8 主從式系統的通訊

Shared memory 與 message passing 也能用於 client-server 系統。另外還有兩種策略：**Sockets** 與 **RPC**。

### 3.8.1 Sockets（通訊端）

- **Socket** 定義為**通訊的端點（endpoint for communication）**。
- 一對透過網路通訊的 process 使用**一對 socket**，每個 process 一個。
- Socket 由 **IP address** 加上 **port number（埠號）** 來識別。
- 通訊發生在**一對 socket 之間**。
- 在主機間傳送的封包，依據**目的埠號**送到正確的 process。

> 📌 投影片例子：
> - socket `161.25.19.8:1625` 指的是主機 `161.25.19.8` 上的 port `1625`。
> - Host X（`146.86.5.20`）的 socket `146.86.5.20:1625` ↔ web server（`161.25.19.8`）的 socket `161.25.19.8:80`。
>
> 💡 補充：IP address 像大樓地址，port number 像門牌房號。Server 通常監聽固定的 well-known port（1024 以下，如 HTTP 是 80、SSH 是 22）；client 的 port 由系統指派一個大於 1024 的號碼。

> 📌 投影片例子：Java 的 **Date Server**
> ```java
> import java.net.*;
> import java.io.*;
>
> public class DateServer
> {
>     public static void main(String[] args) {
>         try {
>             ServerSocket sock = new ServerSocket(6013);
>
>             // now listen for connections
>             while (true) {
>                 Socket client = sock.accept();
>                 // we have a connection
>
>                 PrintWriter pout = new
>                     PrintWriter(client.getOutputStream(), true);
>
>                 // write the Date to the socket
>                 pout.println(new java.util.Date().toString());
>
>                 // close the socket and resume listening for more connections
>                 client.close();
>             }
>         }
>         catch (IOException ioe) {
>             System.err.println(ioe);
>         }
>     }
> }
> ```
> **Date Client**：
> ```java
> import java.net.*;
> import java.io.*;
>
> public class DateClient
> {
>     public static void main(String[] args) {
>         try {
>             // this could be changed to an IP name or address other than the localhost
>             Socket sock = new Socket("127.0.0.1", 6013);
>
>             InputStream in = sock.getInputStream();
>             BufferedReader bin = new
>                 BufferedReader(new InputStreamReader(in));
>
>             String line;
>             while ((line = bin.readLine()) != null)
>                 System.out.println(line);
>
>             sock.close();
>         }
>         catch (IOException ioe) {
>             System.err.println(ioe);
>         }
>     }
> }
> ```
> **流程**：Server 在 port **6013** 監聽，`accept()` 會 block 直到有 client 連線；連上後把目前日期時間寫進 socket 再關閉連線。Client 連到 `127.0.0.1:6013`（**localhost**，本機自己），讀出並印出日期。

### 3.8.2 Remote Procedure Calls（RPC，遠端程序呼叫）

- **RPC** 把網路系統上 process 之間的**程序呼叫**抽象化。
- RPC 的語意讓 client 呼叫**遠端主機上的程序**，就像呼叫**本地程序**一樣。
- **Stub（存根）**：client 端代表 server 上實際程序的 **proxy（代理）**。
- **Client-side stub**：找到 server，並把參數 **marshal（封送／打包）**。
- **Server-side stub**：收到訊息，**unmarshal**（解開打包的參數），然後在 server 上執行該程序。

**架構**：

```
     Client process                        Server process
   ┌─────────────────┐                   ┌─────────────────┐
   │ Client routines │                   │ Server routines │
   │      ↓  ↑       │                   │      ↓  ↑       │
   │   Client stub   │                   │   Server stub   │
   └──────┬──▲───────┘                   └──────┬──▲───────┘
   ┌──────▼──┴───────┐   ──────────►     ┌──────▼──┴───────┐
   │ network routines│   ◄──────────     │ network routines│
   └─────────────────┘                   └─────────────────┘
       Local kernel                         Remote kernel
```

**RPC 的執行過程（Fig. 3.29）**——client 如何找到 server 的 port：

| 步驟 | Client 端 | 訊息內容 | Server 端 |
|---|---|---|---|
| 1 | User 呼叫 kernel，要送 RPC 訊息給程序 X | | |
| 2 | Kernel 送訊息給 **matchmaker（媒合者）** 查詢 port | From: client／To: server／Port: **matchmaker**／Re: address for RPC X | Matchmaker 收到訊息，查詢答案 |
| 3 | Kernel 把 port P 放進使用者的 RPC 訊息 | From: server／To: client／Port: kernel／Re: RPC X／Port: **P** | Matchmaker 回覆 client port P |
| 4 | Kernel 送出 RPC | From: client／To: server／Port: **port P**／〈contents〉 | 監聽 port P 的 **daemon** 收到訊息 |
| 5 | Kernel 收到回覆，傳給 user | From: RPC／Port: P／To: client／Port: kernel／〈output〉 | Daemon 處理請求並送出輸出 |

**Matchmaker**（也叫 rendezvous daemon）：在固定的 RPC port 上提供查詢服務，告訴 client 某個 RPC 目前在哪個 port。

> 💡 補充例子：RPC 像請秘書（client stub）幫你打電話給另一家公司的秘書（server stub），由對方秘書轉告他的老闆（server routine）去做事，再把結果傳回來。你只要像平常交代事情一樣說一聲，不必自己處理撥號、轉接、留言等細節。Marshalling 就是把你的口頭交代整理成對方看得懂的標準格式。

| | **Socket** | **RPC** |
|---|---|---|
| 抽象層級 | 低階：交換**無結構的位元組串流** | 高階：像**呼叫函式**一樣 |
| 資料格式 | 由應用程式自己定義與解析 | Stub 自動 marshal／unmarshal |
| 程式設計師負擔 | 較大 | 較小 |

---

## 重點整理

### Process 的記憶體配置

| 區段 | 存放 | 大小 |
|---|---|---|
| **text** | 程式碼（指令） | 固定 |
| **data** | 全域變數 | 固定 |
| **heap** | 動態配置（`malloc`） | 執行時變動，往上長 |
| **stack** | 函式參數、返回位址、區域變數 | 執行時變動，往下長 |

### 五種狀態與六種轉換

new →(admitted)→ **ready** ⇄(dispatch / interrupt)⇄ **running** →(exit)→ terminated
running →(I/O or event wait)→ **waiting** →(I/O or event completion)→ ready

### IPC 方式總覽

| 方式 | 模型 | 範圍 | 特點 |
|---|---|---|---|
| **POSIX shared memory** | Shared memory | 同一台機器 | 最快；要自己同步 |
| **Mach ports** | Message passing | 同機／分散式 | 用 VM 映射避免複製 |
| **Windows ALPC** | Message passing | **只能同一台機器** | 用 port 與 handle |
| **Ordinary pipe** | Message passing | 同一台機器，需 parent–child | 單向 |
| **Named pipe (FIFO)** | Message passing | UNIX 同機；Windows 可跨機 | 雙向，不需親屬關係 |
| **Socket** | Message passing | **可跨網路** | IP ＋ port；低階 |
| **RPC** | Message passing | **可跨網路** | 像呼叫本地函式；高階 |

---

## 容易混淆的觀念對照

| 比較 | A | B |
|---|---|---|
| **Program vs. Process** | 被動；磁碟上的可執行檔 | 主動；載入記憶體執行中，多了 stack、heap、PC、registers |
| **Stack vs. Heap** | 自動管理；區域變數、參數、返回位址 | 手動配置（`malloc`）；動態資料 |
| **Data vs. Stack** | 全域變數，整個執行期間都在 | 區域變數，函式返回就消失 |
| **Ready vs. Waiting** | 萬事俱備，**只缺 CPU** | 在等某件事（如 I/O），**給 CPU 也不能跑** |
| **Ready queue vs. Wait queue** | 等 CPU 的 process | 等事件的 process |
| **CPU scheduling vs. Swapping** | 從 ready queue 選誰用 CPU；很頻繁 | 把 process 整個搬出／搬入記憶體；記憶體不足時才做 |
| **`fork()` vs. `exec()`** | 複製出一個新 process | 把現有 process 的內容換成新程式（不產生新 process） |
| **`fork()` 回傳 0 vs. > 0** | 我是 child | 我是 parent，值為 child 的 pid |
| **Zombie vs. Orphan** | Child 已死，parent 還沒 `wait()` | Parent 已死，child 還活著 |
| **Independent vs. Cooperating** | 不影響也不受影響 | 會互相影響（共享資料） |
| **Shared memory vs. Message passing** | 快；不經 kernel；自己同步 | 慢；每次經 kernel；適合少量資料與跨電腦 |
| **Unbounded vs. Bounded buffer** | Producer 不用等 | 滿了 producer 等、空了 consumer 等 |
| **Direct vs. Indirect** | 指名對方 process | 透過 mailbox／port |
| **Blocking vs. Nonblocking** | 同步；等到完成才繼續 | 非同步；立刻返回 |
| **Zero vs. Bounded vs. Unbounded capacity** | Sender 一定 block | 滿了才 block | 永不 block |
| **Ordinary vs. Named pipe** | 單向；需 parent–child | 雙向；不需親屬關係 |
| **Half vs. Full duplex** | 同一時間只能單方向 | 可同時雙向 |
| **Socket vs. RPC** | 低階位元組串流 | 高階，像呼叫函式 |

---

## 作業（Homework）

### 課本習題

**[10th edition] Ex. 3.1、3.2、3.11、3.13**（Textbook pp. 154–156、905、907）

| 類型 | 配分 | 題目 |
|---|---|---|
| **手寫** | 75% | 3.1 & 3.2 & 3.13 |
| **程式實作** | 25% | 3.1 & 3.13 |

- 程式實作要繳交**原始程式**與**執行結果畫面截圖**（用自己的**學號**當帳號和機器名稱）。

→ 這幾題都與 `fork()` 有關，複習本筆記 [3.3.1](#331-process-creation行程建立) 的 `fork()` 回傳值與 process 複製的概念，以及 [3.1.1](#311-the-process) 的記憶體區段。

### 程式作業：Implement a simplified shell（實作簡化版 shell）

練習使用 **`fork()`**、**`execlp()`**。

| 等級 | 要求 | 例子 |
|---|---|---|
| **80%** | 允許 **0 個參數** | `ps`、`ls` |
| **90%** | 允許**固定數個參數** | `ls -l`、`ps -aux` |
| **100%** | 允許 **0 ~ 2 個參數** | `ls`、`ls -a`、`ls -a -l` |

**注意事項**：
1. 請印出**執行結果畫面**
2. 請交**原始程式**
3. 請找 **TA Demo**（**沒有 Demo 不給分**）

> 📌 投影片的執行結果範例：
> ```
> $ vi prog3.c
> $ gcc prog3.c
> $ ./a.out
>
>  A simplified shell ... Enter exit to terminate!
>  Joanna:$ ls
> a.out  prog1.c  prog2.c  prog3.c
> child complete
>  Joanna:$ pwd
> /home/joanna/OSDI/ProcessControl
> child complete
>  Joanna:$ date
> Wed Oct 30 15:13:50 GMT 2019
> child complete
>  Joanna:$ l
>
>  Error command! Child exit !child complete
>  Joanna:$ exit
>
>  Joanna:$ Simplified shell terminated !
> ```
> 從範例可以看出程式需要的行為：顯示提示字元 → 讀入命令 → 輸入 `exit` 就結束 → 否則建立 child 去執行 → parent 等 child 結束後印出 `child complete` → 命令不存在時 child 印出錯誤訊息並結束。

→ 參考投影片提供的 Fig. 3.8 程式（本筆記 [3.3.1](#unixfork-與-exec-)）與 Ch1／Ch2 出現過的簡易 shell 範例，以及 Ch2 的 command interpreter 概念。

---

## 英中詞彙總表

### A–D

| English | 中文 | 補充 |
|---|---|---|
| accounting information | 記帳資訊 | |
| address space | 位址空間 | |
| ALPC (Advanced Local Procedure Call) | 進階本地程序呼叫 | Windows 的訊息傳遞設施 |
| anonymous pipe | 匿名管道 | Windows 的 ordinary pipe |
| asynchronous | 非同步 | |
| attach | 附加 | |
| bidirectional | 雙向 | |
| blocking / nonblocking | 阻塞／非阻塞 | |
| bounded buffer | 有限緩衝區 | |
| buffering | 緩衝 | |
| callback | 回呼 | |
| cascading termination | 連鎖終止 | 投影片譯「層疊式的」 |
| child process | 子行程 | |
| communication link | 通訊鏈結 | |
| conduit | 導管 | |
| consumer | 消費者 | |
| context / context switch | 內容／內容切換 | 也譯「上下文切換」 |
| cooperating process | 合作行程 | |
| copy-on-write | 寫入時複製 | |
| CPU scheduler | CPU 排程器 | |
| data section | 資料區段 | 全域變數 |
| deallocate | 回收／釋放 | |
| degree of multiprogramming | 多元程式度 | 記憶體中 process 的數量 |
| direct / indirect communication | 直接／間接通訊 | |
| dispatch | 分派 | |
| duplicate | 複製品 | |

### E–O

| English | 中文 | 補充 |
|---|---|---|
| endpoint | 端點 | |
| entry point | 進入點 | |
| executable image | 可執行映像檔 | |
| fetch / decode / execute | 擷取／解碼／執行 | |
| FIFO | 先進先出 | UNIX 的 named pipe 也叫 FIFO |
| full duplex / half duplex | 全雙工／半雙工 | |
| global variable | 全域變數 | |
| handle | 控制代碼 | |
| header | 標頭 | |
| heap | 堆積 | 動態配置 |
| independent process | 獨立行程 | |
| IPC (Interprocess Communication) | 行程間通訊 | |
| loading | 載入 | |
| local variable | 區域變數 | |
| localhost | 本機 | 127.0.0.1 |
| mailbox | 信箱 | |
| marshal / unmarshal | 封送／解封送 | 打包／解開參數 |
| matchmaker | 媒合者 | |
| memory-mapped file | 記憶體映射檔 | |
| message passing | 訊息傳遞 | |
| migrate | 遷移 | |
| named pipe | 具名管道 | |
| ordinary pipe | 普通管道 | |
| orphan process | 孤兒行程 | |
| overcommitted | 過度配置 | |
| overhead | 額外負擔 | |

### P–Z

| English | 中文 | 補充 |
|---|---|---|
| paradigm | 典範 | 投影片譯「範例」 |
| parallel computing | 平行運算 | |
| parameter | 參數 | |
| parent process | 父行程 | |
| PCB (Process Control Block) | 行程控制區塊 | 也稱 task control block |
| pid (process identifier) | 行程識別碼 | |
| pipe | 管道 | |
| port | 埠 | Mach／Windows 的 mailbox；也指網路埠號 |
| priority | 優先權 | |
| process | 行程 | |
| process scheduler | 行程排程器 | |
| process state | 行程狀態 | |
| producer | 生產者 | |
| program counter (PC) | 程式計數器 | |
| proxy | 代理 | |
| ready queue | 就緒佇列 | |
| register | 暫存器 | |
| rendezvous | 會合 | 投影片譯「約會、集會」 |
| return address | 返回位址 | |
| RPC (Remote Procedure Call) | 遠端程序呼叫 | |
| shared memory | 共享記憶體 | |
| socket | 通訊端 | |
| spawn | 衍生 | 投影片譯「生育」 |
| stack | 堆疊 | |
| stack pointer (SP) | 堆疊指標 | |
| stub | 存根 | |
| subsystem | 子系統 | |
| swap in / swap out | 置入／置出 | |
| swapping | 置換 | |
| synchronization | 同步 | |
| synchronous | 同步的 | |
| task | 任務 | Mach 中類似 process |
| text section | 文字區段 | 程式碼 |
| thread | 執行緒 | |
| time slice | 時間片 | |
| unbounded buffer | 無限緩衝區 | |
| unidirectional | 單向 | |
| wait queue | 等待佇列 | |
| zombie process | 殭屍行程 | |
