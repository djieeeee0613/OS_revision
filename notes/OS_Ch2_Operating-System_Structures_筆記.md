# 作業系統 Chapter 2：Operating-System Structures（作業系統結構）筆記

> **教材**：*Operating System Concepts*, 10th ed.（Silberschatz, Galvin, Gagne）
> **投影片**：Mei-Ling Chiang 教授，暨南大學資管系（2026/09/22，共 113 頁）
> **筆記說明**：
> - 英文專有名詞第一次出現時以「**English（中文）**」標示，文末有完整詞彙表。
> - 標示「📌 投影片例子」者出自投影片；標示「💡 補充例子」者是為了幫助理解另外加的比喻或說明，不在投影片內。

---

## 目錄

- [0. 本章目標](#0-本章目標)
- [2.1 作業系統服務](#21-作業系統服務-operating-system-services)
- [2.2 使用者與作業系統的介面](#22-使用者與作業系統的介面-user-and-os-interface)
- [2.3 系統呼叫](#23-系統呼叫-system-calls)
- [2.4 系統服務](#24-系統服務-system-services)
- [2.5 連結器與載入器](#25-連結器與載入器-linkers-and-loaders)
- [2.6 為什麼應用程式綁定特定 OS](#26-為什麼應用程式綁定特定-os)
- [2.7 作業系統的設計與實作](#27-作業系統的設計與實作)
- [2.8 作業系統結構](#28-作業系統結構-operating-system-structure)
- [2.9 建置與啟動作業系統](#29-建置與啟動作業系統-building--booting-an-os)
- [2.10 作業系統除錯](#210-作業系統除錯-operating-system-debugging)
- [重點整理：五種 OS 結構比較](#重點整理五種-os-結構比較)
- [容易混淆的觀念對照](#容易混淆的觀念對照)
- [作業（Homework）](#作業homework)
- [英中詞彙總表](#英中詞彙總表)

---

## 0. 本章目標

| 問題 | 對應章節 |
|---|---|
| OS 提供哪些服務？ | 2.1 |
| OS 與**使用者**之間的介面是什麼？ | 2.2 |
| System call 如何用來提供 OS 服務？OS 與**程式設計師**之間的介面是什麼？ | 2.3 |
| OS 有哪些組織結構的方式？ | 2.8 |
| OS 如何安裝與開機？ | 2.9 |
| 有哪些工具可以監控 OS 效能？ | 2.10 |
| 設計並實作 Linux kernel module | 2.8.4 |

OS 提供程式執行的環境。觀察 OS 有三個 **vantage point（觀點）**：

1. 看它提供的**服務**（services）
2. 看它給使用者與程式設計師的**介面**（interface）
3. 把系統拆成**元件**及其相互連結（components and interconnections）

---

## 2.1 作業系統服務 (Operating-System Services)

- OS 提供**程式執行的環境**，並對程式與使用者提供服務。
- 這些服務是為了**程式設計師的方便**，讓寫程式更容易。

### 服務總覽（Fig. 2.1）

```
            user and other system programs
   ┌──────────────────────────────────────────────┐
   │   GUI   │  touch screen  │  command line     │  ← user interfaces
   ├──────────────────────────────────────────────┤
   │                 system calls                 │
   ├──────────────────────────────────────────────┤
   │ program    I/O      file   communi-  resource  accounting │
   │ execution  ops     systems  cation   allocation           │
   │      error detection        protection and security       │  ← services
   ├──────────────────────────────────────────────┤
   │               operating system               │
   └──────────────────────────────────────────────┘
                      hardware
```

### 第一組：對使用者有幫助的服務

| 服務 | 中文 | 說明 |
|---|---|---|
| **User interface (UI)** | 使用者介面 | 幾乎所有 OS 都有；形式有 CLI、GUI、touch-screen |
| **Program execution** | 程式執行 | 把程式載入記憶體並執行；能正常或**異常（abnormally）** 結束（表示有錯誤） |
| **I/O operations** | 輸入輸出操作 | 執行中的程式可能需要 I/O（檔案或裝置）。為了**效率與保護**，使用者通常不能直接控制 I/O 裝置，所以 OS 必須提供方法 |
| **File-system manipulation** | 檔案系統操作 | 讀寫檔案與目錄、建立、刪除、搜尋、列出檔案資訊、權限管理 |
| **Communications** | 通訊 | process 之間交換資訊，可在同一台電腦或透過網路跨電腦；實作方式有 **shared memory** 或 **message passing** |
| **Error detection** | 錯誤偵測 | 偵測 CPU、記憶體硬體、I/O 裝置或使用者程式中的錯誤，確保運算正確且一致 |

### 第二組：確保系統本身有效率運作的功能（透過資源共享）

| 服務 | 中文 | 說明 |
|---|---|---|
| **Resource allocation** | 資源分配 | 多個使用者或多個工作同時執行時，把資源分配給它們 |
| **Logging**（舊版稱 accounting） | 紀錄／記帳 | 追蹤哪個使用者用了多少、哪種資源；用於**計費**或**累積使用統計** |
| **Protection and security** | 保護與安全 | Protection：確保所有對系統資源的存取都受到控制。Security：防禦外來者，需要 **user authentication（使用者認證）**，通常用密碼；也延伸到保護外部 I/O 裝置免受無效存取 |

> 💡 補充例子：
> - Logging：雲端主機依 CPU 使用時數收費，就是靠 OS 的使用紀錄。
> - Error detection：USB 隨身碟被硬拔，OS 跳出「裝置無法辨識」的訊息。

---

## 2.2 使用者與作業系統的介面 (User and OS Interface)

使用者與 OS 互動的三種基本方式：CLI、GUI、Touch-screen。

### 2.2.1 Command Interpreter（命令直譯器）— CLI

- **CLI（Command-Line Interface，命令列介面）** 讓使用者直接輸入命令。
- 可能包含在 kernel 內，也可能是一個**特殊的程式**，在工作啟動時或使用者登入時執行（互動式系統）。
- 有多種版本可選時，這些直譯器稱為 **shell（殼層）**。
- 主要工作：**取得並執行使用者指定的下一個命令**。
- 層次：Applications → Shell → Kernel → Hardware。

> 📌 投影片例子：macOS 上的 **bash**（Bourne-Again Shell）。

#### 實作命令的兩種方式 ⭐

| | **(1) Built-in（內建）** | **(2) 透過 system program** |
|---|---|---|
| 做法 | 命令直譯器**本身包含**執行該命令的程式碼 | 命令只是**系統程式的檔名**；shell 找到該檔案，載入記憶體執行 |
| 新增命令 | 需要修改 shell | **不需要修改 shell**，加一個新程式檔即可 |
| 使用者 | — | **UNIX** 採用 |

> 📌 投影片例子：輸入 `rm file.txt`
> 1. shell 到磁碟找名為 `rm` 的檔案（`/bin/rm`）
> 2. 把 `rm` 載入 RAM 執行
> 3. 把 `file.txt` 當參數傳給它
>
> 其他同類命令：`/bin/chmod`、`/bin/cp`、`/bin/echo`。shell 本身完全不懂「刪除檔案」怎麼做。

### 2.2.2 Graphical User Interface（圖形使用者介面）— GUI

- 以**滑鼠**為基礎的**視窗與選單**系統。
- 提供使用者友善的 **desktop metaphor（桌面隱喻）** 介面：圖示代表檔案、程式、動作。
- **1973 年由 Xerox PARC 發明**。
- 📌 投影片例子：Mac OS 的 GUI（Fig. 2.4）。

### 2.2.3 Touch-Screen Interface（觸控螢幕介面）

- 觸控裝置上滑鼠不可行或不適合，需要新的介面。
- 操作與選取以 **gesture（手勢）** 為基礎。
- 用 **virtual keyboard（虛擬鍵盤）** 輸入文字。
- 📌 投影片例子：iPhone 觸控螢幕（Fig. 2.3）。

### 2.2.4 Choice of Interface（介面的選擇）

很多系統同時包含 CLI 與 GUI：

| 系統 | 介面 |
|---|---|
| **Microsoft Windows** | 以 GUI 為主，附 CLI「command」shell |
| **Apple Mac OS X** | 「Aqua」GUI，底下是 UNIX kernel，並提供 shell |
| **Unix / Linux** | 以 CLI 為主，GUI 為選用（如 **KDE**、**GNOME**） |

- 使用者介面可以因系統而異，甚至同一系統內因使用者而異。
- 介面通常**獨立於實際的系統結構**，不是 OS 的直接功能。

---

## 2.3 系統呼叫 (System Calls)

### 基本概念

- **System call（系統呼叫）**：讓使用者程式**請求 OS 服務**的方法。
- 是 OS 所提供服務的**程式介面（programming interface）**。
- 通常以 C / C++ 寫成的 routine 提供。
- 程式大多透過高階的 **API（Application Program Interface，應用程式介面）** 來使用，而不是直接呼叫 system call。

回顧 Ch1：發生 system call 時，CPU 從 **user mode** 切到 **kernel mode**，經 interrupt vector → system call handler → system call service routine，完成後返回。

### System call 使用範例（Fig. 2.5）：複製檔案

> 📌 投影片例子：UNIX 的 `cp in.txt out.txt`，背後的 system call 順序：
>
> | 步驟 | 動作 | 涉及的 I/O |
> |---|---|---|
> | 1 | 取得輸入檔名 | 在螢幕顯示提示、接收輸入 |
> | 2 | 取得輸出檔名 | 在螢幕顯示提示、接收輸入 |
> | 3 | 開啟輸入檔 | 檔案不存在 → **abort（中止）** |
> | 4 | 建立輸出檔 | 檔案已存在 → abort |
> | 5 | 迴圈：從輸入檔讀、寫到輸出檔 | 直到讀取失敗（讀到檔尾） |
> | 6 | 關閉輸出檔 | |
> | 7 | 在螢幕顯示完成訊息 | |
> | 8 | 正常結束 | |
>
> 一個看似簡單的複製動作，就用到一大串 system call。

### API（應用程式介面）

三種最常見的 API：

| API | 適用系統 |
|---|---|
| **Win32 API** | Windows |
| **POSIX API** | POSIX 系統（幾乎所有版本的 UNIX、Linux、Mac OS X） |
| **Java API** | Java 虛擬機器（JVM） |

- **POSIX（Portable Operating System Interface，可攜式作業系統介面）**：IEEE Computer Society 制定的一系列標準，用來維持作業系統之間的相容性。

**為什麼用 API 而不直接用 system call？**
1. **Program portability（程式可攜性）**：同一份程式可在支援相同 API 的系統上編譯執行。
2. 實際的 system call 通常**更瑣碎、更難用**。

> 📌 投影片例子：標準 API `read()`（在命令列用 `man read` 查詢）
> ```c
> #include <unistd.h>
> ssize_t read(int fd, void *buf, size_t count);
> ```
> | 部分 | 說明 |
> |---|---|
> | `ssize_t` | **return value（回傳值）**：成功時回傳讀到的 byte 數；`0` 表示檔尾；錯誤回傳 `-1` |
> | `read` | **function name（函式名稱）** |
> | `int fd` | 要讀取的 **file descriptor（檔案描述子）** |
> | `void *buf` | 資料要讀進去的 **buffer** |
> | `size_t count` | 最多讀進 buffer 的 byte 數 |

### RTE 與 System-call Interface

- **RTE（Run-Time Environment，執行時期環境）**：執行某種程式語言寫的應用程式所需的**整套軟體**，包含 compiler 或 interpreter，以及 library、loader 等。
- RTE 提供 **system-call interface（系統呼叫介面）**，作為連結 OS 所提供 system call 的橋樑。
- System-call interface 會**攔截（intercept）** API 中的函式呼叫，再去呼叫 OS 內必要的 system call。

### API – System Call – OS 的關係（Fig. 2.6）

- 每個 system call 通常對應一個**編號（number）**。
- System-call interface 維護一張**依編號索引的表**（system call table）。
- 它呼叫 kernel 中對應的 system call，並回傳狀態與回傳值。
- 呼叫者**不需要知道** system call 如何實作，只要遵守 API、了解 OS 會回傳什麼即可。

```
            user application
                 │ open()            ▲
 user mode       ▼                   │
 ─────────── system call interface ──┴────
 kernel mode     │                   ▲
                 ▼                   │ return
        system call table  ──i──→  open() 的實作
```

> 📌 投影片例子：**Standard C Library**
> C 程式呼叫 `printf("Greetings")` → C 函式庫攔截 → 呼叫 `write()` system call → kernel 執行 → 結果一路回傳給程式。
> `printf()` 是 **library call（函式庫呼叫）**，`write()` 才是 **system call**。

### Linux/x86 的 System Call 實作

- 使用者 process 請求 kernel 提供的服務（例如開檔）時，就發生 system call。
- 在 x86 CPU 上，使用者 process 用 **software interrupt（軟體中斷）** 通知 kernel。
- 使用的中斷編號是 **`0x80`**（位於 **IDT**，Interrupt Descriptor Table，中斷描述表；32 位元 CPU）。
- 流程：標準 C 函式庫把要傳遞的**參數載入 CPU registers**，然後觸發 `int 0x80`。
- 因此 `system_call` 函式其實是一個 **interrupt handler（中斷處理常式）**。

**System Call Invocation in Linux**：

```
        User Mode                 ║            Kernel Mode
                                  ║
 應用程式       libc 包裝函式      ║   system_call:          sys_xyz() {
   ...          xyz() {           ║     ...                   ...
   xyz()  ───►    ...             ║     sys_xyz()   ───►    }
   ...    ◄───    int 0x80  ══════╬═►   ...         ◄───
                  ...             ║   ret_from_sys_call:
                }           ◄═════╬══   ...
                                  ║     iret
```

| 角色 | 英文 | 位置 |
|---|---|---|
| 應用程式中的呼叫 | system call invocation in application program | user mode |
| 包裝函式 | **wrapper routine** in libc standard library | user mode |
| 系統呼叫處理常式 | **system call handler** | kernel mode |
| 系統呼叫服務常式 | **system call service routine** | kernel mode |

> 📌 投影片例子：Ch1 出現過的簡易 shell 程式，其中 `fork()` 會呼叫 C Library 中的 `fork.S`，由它發出真正的 system call。

**System call 的四步驟**（How a system call can be made）：① trap 進入 kernel → ② 查 dispatch table → ③ 執行 service routine → ④ 返回 user program。

### 參數傳遞（Parameter Passing）⭐

Process 與 OS 之間傳遞 **parameter（參數）** 的三種方法：

| 方法 | 說明 | 特點 |
|---|---|---|
| **1. Registers（暫存器）** | 直接把參數放在暫存器 | **最簡單**，但參數的數量或長度受限 |
| **2. Block / Table（區塊／表格）** | 把參數存在記憶體中的一塊區域，再把**該區塊的位址**放在暫存器傳過去 | **Linux 與 Solaris** 採用 |
| **3. Stack（堆疊）** | 程式把參數 **push** 到 stack，OS 再 **pop** 出來 | 不限制參數的數量或長度 |

> 📌 投影片例子（Fig. 2.7）：user program 把參數放在記憶體的表格 X，把 X 的位址載入 register，然後發出 `system call 13`；OS 執行 13 號 system call 的程式碼時，依 register 中的位址到表格 X 取出參數。
>
> 💡 補充例子：方法 1 像直接把三樣東西拿在手上遞過去（手只有兩隻，東西多了拿不了）；方法 2 像把東西放進置物櫃，只把**櫃子號碼**告訴對方。

### 2.3.3 System Call 的類型（六大類）

| 類別 | 中文 | 主要功能 |
|---|---|---|
| **Process control** | 行程控制 | end、abort；load、execute；create / terminate process；get / set process attributes；wait for time；wait event、signal event；allocate / free memory；用於管理 process 間共享資料存取的 **lock（鎖）** |
| **File manipulation** | 檔案操作 | create / delete file；open、close；read、write、reposition；get / set file attributes |
| **Device manipulation** | 裝置操作 | request / release device；read、write、reposition；get / set device attributes；logically attach / detach devices |
| **Information maintenance** | 資訊維護 | get / set time or date；get / set system data；get / set process、file、device attributes；發生錯誤時 **dump memory**；debugger（單步執行） |
| **Communications** | 通訊 | create / delete communication connection；send / receive messages；transfer status information；attach / detach remote devices |
| **Protection** | 保護 | 控制對資源的存取；set / get permission；allow / deny user |

#### Windows 與 Unix 的 System Call 對照 ⭐

| 類別 | Windows | Unix |
|---|---|---|
| **Process Control** | `CreateProcess()` | `fork()` |
| | `ExitProcess()` | `exit()` |
| | `WaitForSingleObject()` | `wait()` |
| **File Manipulation** | `CreateFile()` | `open()` |
| | `ReadFile()` | `read()` |
| | `WriteFile()` | `write()` |
| | `CloseHandle()` | `close()` |
| **Device Manipulation** | `SetConsoleMode()` | `ioctl()` |
| | `ReadConsole()` | `read()` |
| | `WriteConsole()` | `write()` |
| **Information Maintenance** | `GetCurrentProcessID()` | `getpid()` |
| | `SetTimer()` | `alarm()` |
| | `Sleep()` | `sleep()` |
| **Communication** | `CreatePipe()` | `pipe()` |
| | `CreateFileMapping()` | `shmget()` |
| | `MapViewOfFile()` | `mmap()` |
| **Protection** | `SetFileSecurity()` | `chmod()` |
| | `InitializeSecurityDescriptor()` | `umask()` |
| | `SetSecurityDescriptorGroup()` | `chown()` |

注意：Unix 對檔案和裝置都用 `read()` / `write()`——**把裝置當成檔案來操作**。

### Process Control 的兩個例子

Process 與 job control 有很多面向與變化：

| | **Arduino（Fig. 2.9）** | **FreeBSD（Fig. 2.10）** |
|---|---|---|
| 類型 | **Single-tasking（單工）**，**沒有 OS** | **Multitasking（多工）**，Unix 的變種 |
| 啟動 | 開機時啟動 **boot loader** | 使用者登入 → 啟動使用者選擇的 **shell** |
| 執行程式 | boot loader 把 **sketch（Arduino 程式）** 載入記憶體的特定區域執行 | shell 執行 `fork()` 建立 process，再執行 `exec()` 把程式載入該 process |
| 記憶體 | **單一記憶體空間** | 多個 process 同時在記憶體中（kernel 在高位址，下方有 process C、interpreter、process B、process D） |
| 之後 | 一次只跑一個 sketch | shell 可**等待** process 結束，或**繼續**接受使用者命令 |
| 結束 | — | process 以結束碼離開：**0 = 沒有錯誤**，**> 0 = 錯誤碼** |

> 💡 補充例子：在 Linux 執行完一個命令後輸入 `echo $?`，可以看到上一個 process 的結束碼。成功是 0。

### Communication Models（通訊模型）⭐

**IPC（Interprocess Communication，行程間通訊）** 有兩種常見模型：

#### Message-Passing Model（訊息傳遞模型）

- 透過 OS 提供的 IPC 設施交換資訊；訊息**經由 kernel** 轉送（kernel 內有 message queue）。
- 訊息可以在 process 間**直接**交換，或透過共用的 **mailbox（信箱）** **間接**交換。
- 通訊的 process 可以在同一台或**不同機器**上。
- 必須先**開啟連線（connection）**。
- **Naming（命名）**：要知道對方是誰
  - Host name：`get_hostid()`
  - Process name：`get_processid()`
- 相關 system call：open connection、accept connection、wait for connection、read / write messages、close connection。
- **Daemon（常駐程式）**：專門等待連線的系統程式。接收連線的是 **server**，發起的是 **client**。

#### Shared-Memory Model（共享記憶體模型）

- 用 `shared_memory_create()` 與 `shared_memory_attach()` 建立並取得其他 process 擁有的記憶體區域的存取權。
- 一般情況 OS 會阻止 process 存取別人的記憶體，所以需要**兩個以上的 process 同意移除這項保護限制**。
- 之後就能在共享區域讀寫資料來交換資訊。
- 資料的**格式與位置由這些 process 自己決定**，不受 OS 控制。
- Process 要自己負責**不同時寫入同一位置**——需要 **protection** 與 **synchronization（同步）**。

#### 兩種模型比較

| | **Message Passing** | **Shared Memory** |
|---|---|---|
| 資料路徑 | 經過 **kernel** | 直接讀寫共享區域，**不經 kernel** |
| 適合 | 交換**少量**資料 | 大量資料、需要高速 |
| 優點 | 不需要避免衝突；**跨電腦通訊較容易實作** | **速度最快、最方便**（在同一台電腦內以記憶體傳輸速度進行） |
| 缺點 | 較慢（每次都要 system call） | 需要自行處理 **protection 與 synchronization** |

> 💡 補充例子：
> - Message passing 像**寄信**：信交給郵差（kernel）轉送，安全但慢；可以寄到別的城市（別台電腦）。
> - Shared memory 像**辦公室的共用白板**：兩人直接在上面寫和看，很快；但兩人同時寫同一格就會亂掉，要先約好規則。

---

## 2.4 系統服務 (System Services)

**System services（系統服務）**，又稱 **system utilities（系統公用程式）**：

- 為程式的開發與執行提供方便的環境。
- 有些只是 system call 的使用者介面，有些則複雜得多。

| 分類 | 中文 | 內容 |
|---|---|---|
| **File management** | 檔案管理 | 建立、刪除、複製、改名、列印、列出檔案與目錄 |
| **Status information** | 狀態資訊 | 日期時間、可用記憶體、磁碟空間、使用者數量等 |
| **File modification** | 檔案修改 | text editor（文字編輯器） |
| **Programming-language support** | 程式語言支援 | compiler（編譯器）、assembler（組譯器）、debugger（除錯器）、interpreter（直譯器） |
| **Program loading and execution** | 程式載入與執行 | absolute loader、relocatable loader、linkage editor、overlay loader、debugging system |
| **Communications** | 通訊 | 傳送／接收訊息、瀏覽網頁、email、remote log-in（遠端登入）、傳檔 |
| **Background services** | 背景服務 | 見下方 |

**Background services（背景服務）**：
- 在**開機時啟動**。
  - 有些只用於系統啟動，之後就結束。
  - 有些從開機一直執行到關機。
- 持續執行的系統程式 process 稱為 **services、subsystems、daemons**。
- 提供磁碟檢查、行程排程、錯誤監控、列印等功能。

除了系統程式，大多數 OS 還附帶解決常見問題的**應用程式**：web browser、word processor、text formatter、spreadsheet、繪圖與統計分析套件、database system、compiler、games。

**重點**：大多數使用者對 OS 的印象，來自**應用程式與系統程式**，而不是實際的 system call。

> 💡 補充例子：Windows 的「服務」（services.msc 裡那一長串）、Linux 的 `sshd`、`cron`，都是 daemon。名稱結尾的 `d` 就是 daemon 的意思。

---

## 2.5 連結器與載入器 (Linkers and Loaders)

從原始碼到在記憶體中執行的過程：

```
source program          main.c
      │  compiler        gcc -c main.c
      ▼
object file             main.o         ← relocatable object file
      │  linker          gcc -o main main.o -lm
      │  ◄── other object files / libraries
      ▼
executable file         main
      │  loader          ./main
      │  ◄── dynamically linked libraries
      ▼
program in memory
```

| 元件 | 中文 | 工作 |
|---|---|---|
| **Compiler** | 編譯器 | 把原始碼編譯成 **object file（目的檔）** |
| **Relocatable object file** | 可重定位目的檔 | 設計成可以載入到**任何實體記憶體位置**的目的檔 |
| **Linker** | 連結器 | 把多個目的檔合併成單一的 **binary executable file（二進位可執行檔）**；也會把 library 帶進來 |
| **Loader** | 載入器 | 程式平時以二進位可執行檔存在輔助儲存；要執行時由 loader 載入記憶體 |
| **Relocation** | 重定位 | 為程式各部分指定最終位址，並調整程式中的程式碼與資料以符合這些位址 |

**Dynamic linking（動態連結）**：
- 現代通用系統**不會**把 library 連結進可執行檔。
- 而是使用 **DLL（Dynamically Linked Library，動態連結函式庫）**，在**需要時才載入**。
- 使用同一版本 library 的所有程式**共享**同一份（只載入一次）。

**標準格式**：目的檔與可執行檔有標準格式，OS 才知道如何載入並啟動（Linux 用 **ELF**，Executable and Linking Format）。

> 📌 投影片例子：`gcc -c main.c` 產生 `main.o`；`gcc -o main main.o -lm` 連結數學函式庫（`-lm`）產生 `main`；`./main` 時由 loader 載入執行。
> 💡 補充例子：Windows 上程式啟動時跳出「找不到 xxx.dll」，就是 loader 在執行時找不到要動態連結的函式庫。

---

## 2.6 為什麼應用程式綁定特定 OS

- 在某個系統上編譯的 App，通常**無法在其他 OS 上執行**。
- 原因：每個 OS 提供自己**獨特的 system call**，還有自己的檔案格式等等。

**讓 App 能跨多個 OS 執行的三種方法**：

| 方法 | 說明 | 例子 |
|---|---|---|
| **1. 直譯式語言** | 用 interpreted language 寫，只要各 OS 上有該語言的 interpreter | Python、Ruby |
| **2. 虛擬機器** | 語言本身包含一個 VM，App 在 VM 裡執行 | Java |
| **3. 標準語言＋各自編譯** | 用標準語言寫，在每個 OS 上分別編譯 | C |

> 💡 補充例子：Windows 的 `.exe` 拿到 macOS 上不能直接執行；但同一份 `hello.py` 在 Windows、macOS、Linux 只要裝了 Python 就能跑。

---

## 2.7 作業系統的設計與實作

OS 的設計與實作沒有完整的解答，但有些方法已被證明成功。

### 2.7.1 Design Goals（設計目標）

- 先定義**目標與規格**，會受到硬體選擇與系統類型影響（傳統桌機／筆電、行動、分散式、即時）。

| | **User goals（使用者目標）** | **System goals（系統目標）** |
|---|---|---|
| 觀點 | 使用系統的人 | 設計、建造、維護系統的人 |
| 要求 | convenient to use（方便使用）、easy to learn（容易學）、reliable（可靠）、safe（安全）、fast（快速） | easy to design, implement, maintain（容易設計、實作、維護）；flexible（有彈性）、reliable、error-free（無錯誤）、efficient（有效率） |

### 2.7.2 Mechanisms and Policies（機制與策略）⭐

**把 policy 與 mechanism 分開**是設計 OS 非常重要的原則。

| | **Mechanism（機制）** | **Policy（策略）** |
|---|---|---|
| 決定 | **How** to do something（**如何**做） | **What** will be done（要做**什麼**） |

- 好處：日後策略要改變時，有**最大的彈性**——只改 policy，不用動 mechanism。

> 📌 投影片例子：**timer** 是確保 CPU 保護的 mechanism；而「某個使用者的 timer 要設多久」是 policy 的決定。
> 💡 補充例子：紅綠燈的燈號切換裝置是 mechanism；「這個路口紅燈要 60 秒還是 90 秒」是 policy。交通局想調整秒數，不需要換一組紅綠燈。

### 2.7.3 Implementation（實作）

- 早期 OS 用 **assembly language（組合語言）** 寫，現在大多用 **C 或 C++** 等高階語言。
- 實際上通常混用多種語言：
  - 最底層：assembly
  - 主體：C
  - 系統程式：C、C++、腳本語言（PERL、Python、shell script）

**用高階語言寫的優缺點**：

| 優點 | 缺點 |
|---|---|
| 寫得更快 | 速度降低 |
| 程式碼更精簡（compact） | 儲存空間需求增加 |
| 更容易理解與除錯 | |
| 可使用更好的資料結構與演算法 | |
| **容易 port（移植）** 到其他硬體 | |

---

## 2.8 作業系統結構 (Operating-System Structure)

- OS 既大又複雜，**modularity（模組化）** 很重要。
- 常見做法：把 OS 切成小的元件或 **module（模組）**，而不是一整塊。
- 每個模組都應該是系統中定義明確的部分，有仔細定義的**輸入、輸出與功能**。

系統元件互相連結並**融合（meld）** 成 kernel 的方式有五種：

### 2.8.1 Monolithic Structure（單體式結構）

**Monolithic** ＝ 獨塊巨石的、整體的、龐大的。

- 最簡單的組織方式就是**完全沒有結構**。
- 把 kernel 的所有功能放進**單一、靜態的二進位檔**，在**單一 address space（位址空間）** 中執行。
- 一開始是小而簡單、功能有限的系統，後來成長超出原本的範圍。
- OS 被寫成一堆 procedure 的集合，**任何一個都可以在需要時呼叫其他任何一個**。

> 📌 投影片例子：**原始 UNIX**
> 受限於當時的硬體功能，結構有限，由兩個可分離的部分組成：
> 1. **Systems programs**
> 2. **Kernel**：system-call interface 之下、實體硬體之上的所有東西；提供檔案系統、CPU 排程、記憶體管理等功能——**大量功能被合併在同一層**。
>
> 傳統 UNIX 結構（比 simple 複雜但未完全分層）：
> ```
>                  (the users)
>   shells and commands / compilers and interpreters / system libraries
>   ════════ system-call interface to the kernel ════════
>   signals, terminal handling │ file system        │ CPU scheduling
>   character I/O system       │ swapping, block I/O│ page replacement
>   terminal drivers           │ disk & tape drivers│ demand paging, virtual memory
>   ════════ kernel interface to the hardware ════════
>   terminal controllers │ device controllers │ memory controllers
> ```
>
> 📌 投影片例子：**Linux** 是 **monolithic plus modular design**——applications / glibc → system-call interface → (file systems、CPU scheduler、networks (TCP/IP)、memory manager、block devices、character devices) → device drivers → hardware。

| 優點 | 缺點 |
|---|---|
| **效能優勢明顯**：system-call interface 的 overhead 很小，kernel 內部通訊很快 | **難以實作與擴充** |
| | 所有功能擠在一層，一個地方出錯可能拖垮整個系統 |

儘管有這些缺點，monolithic kernel 的**速度與效率**說明了為什麼 UNIX、Linux、Windows 至今仍看得到這種結構。

> 📌 投影片例子：修改 Linux 核心後重新編譯（`make all`、`make modules_install`、`make install`、`reboot`），若設定有誤可能出現 **Booting has failed** 或 **Kernel Panic**，要重開機並在核心選單改選原本的核心。

### 2.8.2 Layered Approach（分層式）

- OS 被切成許多 **layer（層）**，每一層建立在較低層之上。
- **最底層（layer 0）是硬體**；**最高層（layer N）是使用者介面**。
- 每一層**只能使用較低層**的函式與服務。

```
  layer N   user interface
     ⋮
  layer 1
  layer 0   hardware
```

> 📌 投影片例子：Application Programs → (user / kernel mode 分界) → System Services → File System → Memory and I/O Device Management → Processor Scheduling → Hardware。

| 優點 | 缺點 |
|---|---|
| **建構與除錯簡單** | **需要仔細規劃**：必須適當定義每一層的功能，因為一層只能用它下面的層 |
| 簡化除錯與系統驗證（從最底層逐層往上測，出錯一定在當前這層） | **效率較差**：層越多、overhead 越大、越慢。每一層都替 system call 增加負擔 |
| 簡化系統的設計與實作 | |

- **很少 OS 採用純粹的分層式**。
- 現在的趨勢：設計**較少的層、每層較多功能**，保有模組化程式碼的大部分優點，同時避免層的定義與互動問題。

> 💡 補充例子：像公司公文一層一層往上簽。責任清楚、容易找出哪一關出問題；但每多一層就多一道手續，速度變慢。

### 2.8.3 Microkernels（微核心）

- 1980 年代中期，**CMU（卡內基美隆大學）** 開發了 **Mach**，用 microkernel 方法把 kernel 模組化。
- 做法：把所有**非必要的元件從 kernel 移除**，改以**使用者層級程式（user-level program）** 實作，各自在**獨立的位址空間**。
- Microkernel 通常只提供：
  - 最基本的 **process management**
  - 最基本的 **memory management**
  - **communication facility（通訊設施）**
- 主要功能：在 client 程式與各種同樣在 user space 執行的服務之間提供通訊。
- 通訊透過 **message passing**。

```
  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │ Application │   │ File System │   │Device Driver│   user mode
  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
         └──── messages ───┴─── messages ────┘
  ┌─────────────────────────────────────────────────┐
  │ IPC │ memory management │ CPU scheduling        │   kernel mode
  │                  microkernel                    │
  └─────────────────────────────────────────────────┘
                       hardware
```

| 優點 | 缺點 |
|---|---|
| 容易**擴充**（新服務加在 user space，不必改 kernel） | **效能下降**：系統功能的 overhead 增加 |
| 容易把 OS **移植**到新架構 | user space 與 kernel space 之間通訊的負擔（訊息要複製、要切換 process） |
| 更**安全、可靠**（大部分服務以 user process 執行；某個服務掛掉不會拖垮整個 OS） | |

> 📌 投影片例子：
> - **Darwin**（macOS 與 iOS 的 kernel 元件）部分以 Mach kernel 為基礎。
> - **QNX Neutrino** microkernel。
> - **Windows 的演變**：Windows NT（分層式 microkernel 組織）→ Windows NT 4.0（把一些層從 user space 搬到 kernel space 並更緊密整合）→ Windows XP（比較像 monolithic 而不是 microkernel）。原因就是效能。

### 2.8.4 Modules（模組）

- 目前 OS 設計**可能最好的方法**是使用 **LKM（Loadable Kernel Module，可載入核心模組）**。
- Kernel 有一組核心元件，可以在**開機時**或**執行期間**透過模組連結額外的服務。
- 現代 UNIX 實作（Linux、macOS、Solaris）以及 Windows 都很常見。
- Kernel 提供核心服務，其他服務在 kernel 執行時**動態**實作。

> 📌 投影片例子：**Solaris loadable modules**——core Solaris kernel 周圍掛著：scheduling classes、file systems、loadable system calls、executable formats、STREAMS modules、miscellaneous modules、device and bus drivers。

**與其他結構的比較**：

| 比較對象 | 相似處 | 模組方式更好的地方 |
|---|---|---|
| **Layered** | 每個 kernel 區段都有定義明確、受保護的介面 | **更有彈性**：任何模組都可以呼叫其他任何模組 |
| **Microkernel** | 主模組只有核心功能，以及如何載入其他模組、與之通訊的知識 | **更有效率**：模組之間通訊**不需要 message passing** |

#### Linux 的 LKM

- Linux 使用 LKM，主要用來支援 **device driver** 與 **file system**。
- LKM 可以在執行期間**插入（insert）** 或**移除（remove）**。
- 讓 kernel 具有動態與模組化的特性，同時**保有 monolithic 系統的效能優勢**。

**擴充 Linux kernel 的兩種方式** ⭐：

| | **(1) 直接修改 kernel 原始碼** | **(2) 使用 LKM** |
|---|---|---|
| 做法 | 加入程式碼 → 編譯連結安裝（`make all`、`make modules_install`、`make install`）→ 重開機 | 把模組在執行期間直接載入 kernel |
| kernel 大小 | 加入的程式碼**永久**增加 kernel 大小 | 可隨時卸載，**不會永久影響** kernel 大小 |
| 編譯 | **整個 kernel** 都要重新編譯 | 模組可以**單獨編譯** |
| 重開機 | **需要** | **不需要** |

**Linux 模組的特性**：
- Kernel 編譯時要啟用 loadable module 支援。
- 模組是 **ELF**（Executable and Linking Format）object code。
- 執行期間插入／移除的命令：**`insmod`**、**`rmmod`**。
- 模組是 **kernel 的一部分，不是 process**（位於 kernel space）。
- 至少要有 **2 個函式：init 與 exit**。

> 📌 投影片例子：
> ```
> insmod hello.ko     # 裝載 module
> rmmod hello.ko      # 卸載 module
> cat /var/log/messages   # 查看系統資訊
> ```
> 在 `/var/log/messages` 中可看到 `kernel: Hello, world` 與 `kernel: Goodbye, module world`，表示成功裝載與卸載。
>
> 💡 補充例子：對應上面輸出的最小模組大致長這樣（示意用，我沒有在這裡實際編譯測試；實作時以老師給的範例與你的 kernel 版本為準）：
> ```c
> #include <linux/init.h>
> #include <linux/module.h>
> #include <linux/kernel.h>
>
> MODULE_LICENSE("GPL");
>
> static int __init hello_init(void)      /* insmod 時執行 */
> {
>     printk(KERN_INFO "Hello, world\n");
>     return 0;
> }
>
> static void __exit hello_exit(void)     /* rmmod 時執行 */
> {
>     printk(KERN_INFO "Goodbye, module world\n");
> }
>
> module_init(hello_init);
> module_exit(hello_exit);
> ```
> 模組在 kernel space 執行，所以用 `printk()` 而不是 `printf()`，輸出會進系統 log 而不是螢幕。較新的 Ubuntu 通常用 `dmesg` 或 `journalctl -k` 查看，而不是 `/var/log/messages`。

### 2.8.5 Hybrid Systems（混合式系統）

- 很少 OS 採用單一、嚴格定義的結構。
- 大多數 OS **結合不同結構**，成為 hybrid system，以兼顧 **performance（效能）、security（安全）、usability（易用性）**。

| OS | 結構特性 |
|---|---|
| **Linux** | **Monolithic**：整個 OS 在單一 kernel 位址空間，效能非常好。<br>同時 **modular**：新功能可動態加入 kernel |
| **Windows** | 大體上 **monolithic**（效能考量）。<br>保留一些 **microkernel** 的行為：支援以 user-mode process 執行的獨立 **subsystem**（稱為 personalities）。<br>也支援動態 loadable kernel module |

#### macOS 與 iOS

**架構（由上到下）**：

| 層 | 中文 | 說明 |
|---|---|---|
| applications | 應用程式 | |
| **user experience** | 使用者體驗層 | macOS 用 Aqua；iOS 用 Springboard |
| **application frameworks** | 應用程式框架 | Cocoa（macOS）／**Cocoa Touch**（iOS），提供 Objective-C API |
| **core frameworks** | 核心框架 | 圖形、多媒體等 |
| **kernel environment (Darwin)** | 核心環境 | |

**iOS**：
- Apple 給 iPhone、iPad 的行動 OS。
- 以 Mac OS X 為基礎，加上行動裝置的功能。
- **不能直接執行 Mac OS X 應用程式**；CPU 架構也不同（**ARM** vs. Intel）。
- 各層：Cocoa Touch（開發 App 的 Objective-C API）→ Media services（圖形、音訊、視訊）→ Core services（雲端運算、資料庫）→ Core OS（以 Mac OS X kernel 為基礎）。

**Darwin 的結構**（hybrid）：

```
          applications
        library interface
   Mach traps │ BSD (POSIX) system calls
   ─────────────────────────────────────
   scheduling │ IPC │ memory management
   iokit                Mach kernel
   kexts
```

- 是一個分層系統，主要由 **Mach microkernel** 與 **BSD UNIX kernel** 組成。
- 提供**兩套** system-call 介面：Mach traps 與 BSD (POSIX) system calls。
- Kernel 環境提供 **I/O Kit** 開發 device driver，以及動態可載入模組，稱為 **kernel extension（kext，核心擴充）**。
- Darwin 把 Mach、BSD、I/O Kit 與所有 kext 合併在**單一位址空間**。
- 所以這裡的 **Mach 不是純粹的 microkernel**。
- Mach 內部仍有 message passing，但**不需要複製資料**，因為各服務存取的是同一個位址空間——這樣解決了 microkernel 的效能問題。

#### Android

- 由 **Open Handset Alliance（開放手機聯盟）**（主要由 Google 主導）設計，為 Android 智慧型手機與平板開發。**Open source**。
- 與 iOS 類似：是一個**分層的軟體堆疊（layered stack of software）**，提供豐富的框架來開發行動應用。
- 以 **ART（Android RunTime）虛擬機器**為基礎：
  - App 用 **Java ＋ Android API** 開發。
  - Java class 檔 → 編譯成 Java **bytecode** → 轉譯成可在 ART 上執行的檔案。
- Libraries 包含：web browser 框架（**webkit**）、資料庫（**SQLite**）、網路支援如 **SSL（secure sockets）**。

**架構（由上到下）**：

| 層 | 說明 |
|---|---|
| applications | 應用程式 |
| **ART VM** / Android frameworks / **JNI** | 執行環境、框架、Java Native Interface（讓 Java 呼叫原生程式碼） |
| **native libraries** | SQLite、openGL、webkit、surface manager、SSL、media framework |
| **HAL**（Hardware Abstraction Layer） | 硬體抽象層：把實體硬體抽象化，讓 App 開發者不必管不同硬體的差異 |
| **Bionic** | Google 為 Android 開發的標準 C 函式庫 |
| **Linux kernel** | |
| hardware | |

---

## 2.9 建置與啟動作業系統 (Building & Booting an OS)

### 2.9.1 Operating-System Generation（作業系統生成）

- OS 通常設計成能在**一整類機器**上執行，所以必須針對每個特定的電腦環境做設定或生成。這個過程稱為 **SYSGEN（System Generation，系統生成）**。

**SYSGEN 程式取得硬體組態資訊的方式**：
- 讀取指定的檔案
- 詢問系統操作員
- 直接**探測（probe）** 硬體

**需要決定的事項**：
- 使用什麼 CPU？
- 開機磁碟如何格式化？
- 有多少記憶體可用？
- 有哪些裝置可用？
- 需要哪些 OS 選項、使用什麼參數值？

**建立系統的三種方式**：

| 方式 | 做法 | 特性 |
|---|---|---|
| **(a) Recompile** | 修改原始碼後**完全重新編譯** | 最貼合硬體、系統最小；但修改最麻煩 |
| **(b) 選擇預先編譯的模組** | 從 precompiled library 選出需要的模組並**連結**成 OS | 不必重編，生成較快；系統可能較大而通用 |
| **(c) 完全模組化** | 建構完全模組化的系統，在**執行期**選擇模組 | 最有彈性；硬體變更時最容易調整 |

主要差異在於：生成系統的**大小**與**通用性（generality）**，以及硬體組態變更時**修改的難易度**。

**一般情況**：買來的電腦通常已經裝好 OS，但也可以自己建置並安裝其他 OS。

**從零開始生成 OS 的步驟**：
1. 撰寫 OS 原始碼
2. 針對要執行的系統**設定（configure）** OS
3. **編譯（compile）** OS
4. **安裝（install）** OS
5. **啟動（boot）** 電腦及新的 OS

> 📌 投影片例子：**Building and Booting Linux**
>
> | 步驟 | 命令 | 產生 |
> |---|---|---|
> | 1. 下載原始碼 | 從 <http://www.kernel.org> | |
> | 2. 設定 kernel | `make menuconfig` | **`.config`** 設定檔 |
> | 3. 編譯 kernel | `make` | **`vmlinuz`**（kernel image） |
> | 4. 編譯 kernel modules | `make modules` | 依 `.config` 的參數編譯 |
> | 5. 安裝 kernel modules | `make modules_install` | |
> | 6. 安裝新 kernel | `make install` | |

### 2.9.2 System Boot（系統開機）

問題：硬體怎麼知道 kernel 在哪裡、怎麼載入？

- **Booting（開機）**：藉由載入 kernel 來啟動電腦的程序。
- **Bootstrap program／boot loader（啟動程式／開機載入器）**：
  - 一小段程式碼，存在 **ROM 或 EEPROM**。
  - **找到 kernel → 載入記憶體 → 開始執行**。
- Kernel 初始化硬體後，**掛載 root file system（根檔案系統）**。

#### 單階段 vs. 多階段開機

| 類型 | 流程 |
|---|---|
| **單階段** | ROM 中的 bootstrap loader 直接從磁碟載入 kernel image 到 RAM |
| **Multistage boot（多階段開機）** | ① 執行非揮發性韌體（**BIOS**）中的小型初始 boot loader<br>② 它從磁碟固定位置 **boot block（開機區塊）** 載入**第二個 boot loader**<br>③ boot block 中的程式可能已足夠複雜，能把整個 OS 載入記憶體並開始執行；或只是再去載入更完整的 bootstrap 程式<br>④ 最後載入 kernel image |

- PC 採用多階段：簡單的 bootstrap loader 從磁碟取出較複雜的 boot program，再由它載入 kernel。
- **GRUB（Grand Unified Bootloader）**：Linux 系統上的開放原始碼 bootstrap 程式。

> 📌 投影片例子：GRUB 開機選單（Ubuntu 16.04 在 VMware 上），列出多個可選的 kernel：
> ```
> *Ubuntu, with Linux 4.4.21
>  Ubuntu, with Linux 4.4.21 (upstart)
>  Ubuntu, with Linux 4.4.21 (recovery mode)
>  Ubuntu, with Linux 4.4.0-31-generic
>  ...
> ```
> 用上下鍵選擇、Enter 開機、`e` 編輯命令、`c` 進入命令列。自己編譯新 kernel 後，就是在這裡選擇要用哪一個開機；新 kernel 開不起來時，也是回到這裡選舊的。

#### BIOS vs. UEFI

| | **BIOS** | **UEFI（Unified Extensible Firmware Interface，統一可延伸韌體介面）** |
|---|---|---|
| 開機過程 | 多階段 | **單一、完整的 boot manager** |
| 速度 | 較慢 | **較快** |
| 支援 | 較舊 | 對 **64 位元系統**與**大容量磁碟**支援較好 |

現代系統以 UEFI 取代 BIOS。

---

## 2.10 作業系統除錯 (Operating-System Debugging)

**Debugging（除錯）**：找出並修正錯誤（**bug**）。

### 錯誤資訊的來源

| 名詞 | 中文 | 說明 |
|---|---|---|
| **Log file** | 紀錄檔 | OS 產生，包含錯誤資訊 |
| **Core dump** | 核心傾印 | **應用程式**失敗時產生，擷取該 **process 的記憶體** |
| **Crash dump** | 當機傾印 | **OS** 失敗（crash）時產生，包含 **kernel 記憶體** |

> **Kernighan's Law**："Debugging is twice as hard as writing the code in the first place. Therefore, if you write the code as cleverly as possible, you are, by definition, not smart enough to debug it."
> （除錯比寫程式難一倍。所以如果你把程式寫得盡可能巧妙，照定義來說，你就不夠聰明去替它除錯。）

### Performance Tuning（效能調校）

- 除了當機之外，效能調校可以讓系統效能最佳化。
- 目標：移除 **bottleneck（瓶頸）** 來改善效能。
- OS 必須提供計算並顯示系統行為量測值的方法。
- 兩種方法：
  - **Trace listing（追蹤列表）**：記錄活動以供分析。
  - **Profiling（剖析）**：週期性地取樣 instruction pointer，找出統計趨勢（看程式時間都花在哪）。

> 📌 投影片例子：UNIX 的 **`top`** 程式（顯示 load average、Tasks、CPU 使用率、Mem、Swap，以及各 process 的 PID、%CPU、%MEM）；Windows 的 **Task Manager（工作管理員）**。

### Tracing（追蹤）

針對**特定事件**收集資料，例如一次 system call 呼叫所涉及的步驟。

| 工具 | 用途 |
|---|---|
| **`strace`** | 追蹤某個 process 所呼叫的 **system call** |
| **`gdb`** | 原始碼層級的 debugger |
| **`perf`** | Linux 效能工具集 |
| **`tcpdump`** | 收集網路封包 |

> 💡 補充例子：`strace ls` 可以看到 `ls` 這個命令背後實際呼叫了哪些 system call（`openat`、`read`、`write`、`close`…），正好對應 2.3 節的內容。

### BCC（BPF Compiler Collection）

- 要除錯 user-level 與 kernel 程式碼之間的互動，若沒有能同時理解兩者並能**插入探測（instrument）** 的工具組，幾乎不可能。
- **BCC** 是為 Linux 提供追蹤功能的豐富工具組（另見原始的 **DTrace**）。
- 工具涵蓋整個系統各層：Applications、System Libraries、System Call Interface、VFS、File Systems、Sockets、TCP/UDP、Scheduler、Virtual Memory、Device Drivers 等。

> 📌 投影片例子：`disksnoop.py` 追蹤磁碟 I/O 活動
> ```
> TIME(s)          T   BYTES   LAT(ms)
> 1946.29186700    R   8       0.27
> 1946.33965000    R   8       0.26
> 1948.34585000    W   8192    0.96
> 1950.43251000    R   4096    0.56
> 1951.74121000    R   4096    0.35
> ```
> 欄位：時間、類型（R 讀／W 寫）、位元組數、延遲（毫秒）。
> 其他工具如 `opensnoop`（追蹤開檔）、`execsnoop`（追蹤新 process）、`tcpconnect`（追蹤 TCP 連線）、`biolatency`（區塊 I/O 延遲）。

---

## 重點整理：五種 OS 結構比較

| 結構 | 核心概念 | 優點 | 缺點 | 例子 |
|---|---|---|---|---|
| **Monolithic（單體式）** | 所有功能放在單一位址空間的一個二進位檔 | 效能最好：system call overhead 小、kernel 內通訊快 | 難以實作與擴充；一處出錯影響全體 | 原始 UNIX、Linux（主體） |
| **Layered（分層式）** | 切成多層，每層只能用下層 | 建構、除錯、驗證簡單 | 層的定義需仔細規劃；層多則慢 | 很少純粹採用 |
| **Microkernel（微核心）** | kernel 只留最基本功能，其餘搬到 user space，用 message passing 溝通 | 易擴充、易移植、更安全可靠 | 效能差（user–kernel 通訊 overhead） | Mach、QNX、Darwin（部分） |
| **Modules（模組）** | 核心元件 ＋ 可動態載入的模組（LKM） | 有彈性（任何模組可互相呼叫）；有效率（不需 message passing）；不需重開機 | 模組在 kernel space，有 bug 仍可能影響整個 kernel | Linux、Solaris、macOS、Windows |
| **Hybrid（混合式）** | 結合多種結構 | 兼顧效能、安全、易用性 | 結構較複雜 | Linux、Windows、macOS／iOS、Android |

> 💡 表中 Modules 的缺點是補充說明，投影片沒有明列；其餘優缺點出自投影片。

---

## 容易混淆的觀念對照

| 比較 | A | B |
|---|---|---|
| **System call vs. API** | OS kernel 提供的實際入口，因 OS 而異 | 給程式設計師用的函式介面（如 POSIX、Win32），底層再去呼叫 system call |
| **Library call vs. System call** | `printf()`：在 user mode 的函式庫內 | `write()`：進入 kernel mode |
| **System call vs. System service** | 程式請求 OS 服務的介面 | 系統公用程式（如 `ls`、編輯器、compiler），本身會用到 system call |
| **Built-in command vs. System program** | shell 自己含有執行的程式碼 | shell 載入磁碟上同名的程式來執行 |
| **Message passing vs. Shared memory** | 經 kernel；少量資料；易跨電腦 | 不經 kernel；最快；要自己同步 |
| **Linker vs. Loader** | 把目的檔與函式庫合成可執行檔 | 把可執行檔載入記憶體執行 |
| **Static vs. Dynamic linking** | 函式庫直接併入可執行檔 | 執行時才載入，多個程式共享（DLL） |
| **Mechanism vs. Policy** | **如何**做 | 要做**什麼** |
| **User goals vs. System goals** | 好用、好學、可靠、安全、快 | 好設計、好實作、好維護、有彈性、有效率 |
| **Layered vs. Modules** | 只能呼叫下層 | 任何模組可呼叫任何模組 |
| **Microkernel vs. Modules** | 服務在 user space，靠 message passing | 模組在 kernel space，直接呼叫 |
| **修改 kernel 原始碼 vs. LKM** | 整個重編、要重開機、永久變大 | 單獨編譯、不必重開機、可卸載 |
| **Core dump vs. Crash dump** | 應用程式失敗，存 process 記憶體 | OS 失敗，存 kernel 記憶體 |
| **Profiling vs. Tracing** | 週期性取樣找統計趨勢 | 針對特定事件收集詳細資料 |
| **BIOS vs. UEFI** | 多階段開機、較慢 | 單一 boot manager、較快、支援 64 位元與大磁碟 |
| **Arduino vs. FreeBSD** | 單工、無 OS、boot loader 載入 sketch | 多工、shell 用 `fork()`＋`exec()` |

---

## 作業（Homework）

1. **Briefly describe the features of the following operating system structure: microkernel approach, monolithic kernel approach, modular kernel approach, layered approach, and hybrid systems. Discuss their advantages and disadvantages.**
   （簡述下列作業系統結構的特性：微核心、單體式核心、模組化核心、分層式、混合式系統，並討論其優缺點。）
   → 複習本筆記 [2.8](#28-作業系統結構-operating-system-structure) 與 [五種 OS 結構比較](#重點整理五種-os-結構比較)

2. **Briefly describe the features of your smartphone OS.**
   （簡述你的智慧型手機作業系統的特性。）
   → 複習本筆記 [2.8.5](#285-hybrid-systems混合式系統) 中的 iOS／Darwin 或 Android 部分，依你實際使用的手機來寫

3. **What are the two models of interprocess communication? Discuss the strengths and weaknesses of these two approaches.**
   （行程間通訊的兩種模型是什麼？討論這兩種方法的優缺點。）
   → 複習本筆記 [2.3 Communication Models](#communication-models通訊模型)

---

## 英中詞彙總表

### A–D

| English | 中文 | 補充 |
|---|---|---|
| abort | 中止 | 異常結束 |
| address space | 位址空間 | |
| API (Application Program Interface) | 應用程式介面 | |
| ART (Android RunTime) | Android 執行環境 | Android 的 VM |
| assembler | 組譯器 | |
| assembly language | 組合語言 | |
| authentication | 認證 | |
| background service | 背景服務 | |
| BCC (BPF Compiler Collection) | BPF 編譯器集合 | Linux 追蹤工具組 |
| BIOS | 基本輸入輸出系統 | |
| boot block | 開機區塊 | |
| boot loader / bootstrap program | 開機載入器／啟動程式 | |
| booting | 開機 | |
| bottleneck | 瓶頸 | |
| built-in command | 內建命令 | |
| bytecode | 位元組碼 | |
| CLI (Command-Line Interface) | 命令列介面 | |
| command interpreter | 命令直譯器 | 即 shell |
| compiler | 編譯器 | |
| core dump | 核心傾印 | 應用程式失敗 |
| crash dump | 當機傾印 | OS 失敗 |
| daemon | 常駐程式 | |
| debugger / debugging | 除錯器／除錯 | |
| desktop metaphor | 桌面隱喻 | |
| dispatch table | 分派表 | |
| DLL (Dynamically Linked Library) | 動態連結函式庫 | |
| dump | 傾印 | |

### E–L

| English | 中文 | 補充 |
|---|---|---|
| ELF (Executable and Linking Format) | 可執行與連結格式 | |
| error detection | 錯誤偵測 | |
| executable file | 可執行檔 | |
| file descriptor | 檔案描述子 | |
| gesture | 手勢 | |
| GRUB (Grand Unified Bootloader) | — | Linux 的開機載入器 |
| GUI (Graphical User Interface) | 圖形使用者介面 | |
| HAL (Hardware Abstraction Layer) | 硬體抽象層 | |
| hybrid system | 混合式系統 | |
| IDT (Interrupt Descriptor Table) | 中斷描述表 | |
| instrument | 插入探測 | |
| intercept | 攔截 | |
| interaction | 互動 | 投影片譯「互相影響」 |
| interpreter | 直譯器 | |
| IPC (Interprocess Communication) | 行程間通訊 | |
| JNI (Java Native Interface) | Java 原生介面 | |
| JVM (Java Virtual Machine) | Java 虛擬機器 | |
| kext (kernel extension) | 核心擴充 | macOS 的可載入模組 |
| layered approach | 分層式 | |
| library call | 函式庫呼叫 | |
| linker | 連結器 | |
| LKM (Loadable Kernel Module) | 可載入核心模組 | |
| loader | 載入器 | |
| lock | 鎖 | |
| log file / logging | 紀錄檔／紀錄 | |

### M–P

| English | 中文 | 補充 |
|---|---|---|
| mailbox | 信箱 | |
| mechanism | 機制 | 決定 how |
| meld | 融合 | 投影片譯「使合併；使結合」 |
| message passing | 訊息傳遞 | |
| message queue | 訊息佇列 | |
| microkernel | 微核心 | |
| modularity / module | 模組化／模組 | |
| monolithic | 單體式 | 投影片譯「獨塊巨石的、整體的、龐大的」 |
| mount | 掛載 | |
| multistage boot | 多階段開機 | |
| object file | 目的檔 | |
| overhead | 額外負擔 | |
| parameter | 參數 | |
| performance tuning | 效能調校 | |
| policy | 策略 | 決定 what |
| port / portability | 移植／可攜性 | |
| POSIX | 可攜式作業系統介面 | |
| primitives | 基本操作 | |
| probe | 探測 | |
| profiling | 剖析 | |
| protection | 保護 | |

### R–Z

| English | 中文 | 補充 |
|---|---|---|
| reliability | 可靠度 | 投影片譯「可靠程度」 |
| relocatable object file | 可重定位目的檔 | |
| relocation | 重定位 | |
| reposition | 重新定位（讀寫位置） | |
| resource allocation | 資源分配 | |
| root file system | 根檔案系統 | |
| RTE (Run-Time Environment) | 執行時期環境 | |
| service routine | 服務常式 | |
| shared memory | 共享記憶體 | |
| shell | 殼層 | |
| sketch | — | Arduino 程式 |
| software interrupt | 軟體中斷 | |
| subsystem | 子系統 | |
| synchronization | 同步 | 投影片譯「使同步、使協調」 |
| SYSGEN (System Generation) | 系統生成 | |
| system call | 系統呼叫 | |
| system-call interface | 系統呼叫介面 | |
| system service / system utility | 系統服務／系統公用程式 | |
| touch-screen interface | 觸控螢幕介面 | |
| trace listing / tracing | 追蹤列表／追蹤 | |
| UEFI | 統一可延伸韌體介面 | |
| user authentication | 使用者認證 | |
| vantage point | 觀點 | |
| virtual keyboard | 虛擬鍵盤 | |
| wrapper routine | 包裝函式 | |
