面試 Linux Driver（驅動程式）與韌體工程師，考題通常會跨越 **Linux 核心架構、硬體協定、記憶體管理、同步機制以及 C 語言底層優化**。  
以下為您整理出面試最常被問到的核心問題與標準答案，並分為四大模組：

## **一、 Linux 核心與驅動基礎 (Kernel & Driver Basics)**

### **Q1: 請說明 Linux User Space 與 Kernel Space 的區別？兩者如何進行資料傳遞？**

* **答案：**  
  * **User Space（使用者空間）：** 執行一般應用程式，運行在 CPU 的低權限級別（如 ARM 的 EL0 或 x86 的 Ring 3）。無法直接存取硬體。  
  * **Kernel Space（核心空間）：** 執行作業系統核心與驅動程式，運行在高權限級別（如 ARM 的 EL1 或 x86 的 Ring 0）。擁有完整的硬體存取權限。  
  * **資料傳遞：** User space 不能直接存取 Kernel space 的記憶體。必須透過**系統呼叫 (System Call)**，並使用核心提供的 API 進行資料拷貝：  
    * copy\_from\_user()：將 User space 的資料複製到 Kernel space。  
    * copy\_to\_user()：將 Kernel space 的資料複製到 User space。  
    * 對於大數據量，可以使用 mmap() 將核心記憶體空間映射到使用者空間，實現零拷貝 (Zero-copy) 提高效率。

### **Q2: 什麼是 Linux 字元設備 (Character Device) 驅動？它的主要結構是什麼？**

* **答案：**  
  * 字元設備是 Linux 驅動中最基礎的一種，資料以「位元組流 (Byte Stream)」的形式按順序存取（例如：UART、I2C、SPI 裝置）。  
  * **核心結構：** 1\. **設備號：** 由主設備號 (Major number，代表驅動類型) 和次設備號 (Minor number，代表具體設備個體) 組成。  
    2\. **struct file\_operations：** 這是最重要的結構體，裡面註冊了由驅動實現的函數指針，對應 User space 的 VFS（虛擬檔案系統）操作，如 .open, .read, .write, .ioctl, .release。  
    3\. **cdev 結構體：** 代表核心中的字元設備物件，需透過 cdev\_init() 和 cdev\_add() 註冊到系統中。

### **Q3: 請解釋 Linux 的 Linux Device Model (Bus, Device, Driver) 架構？**

* **答案：**  
  Linux 核心為了實現程式碼解耦與重用，採用了 **總線 (Bus)、設備 (Device)、驅動 (Driver)** 架構：  
  * **Bus（總線）：** 如 PCI、USB、I2C、Platform 等。負責管理連接在其上的設備與驅動，並維護兩條鏈結串列（Devices 鏈結與 Drivers 鏈結）。  
  * **Device（設備）：** 代表硬體實體，包含硬體資源（如暫存器位址、中斷號）。現在多透過 **設備樹 (Device Tree, .dts)** 來描述。  
  * **Driver（驅動）：** 代表軟體行為，實現操作硬體的邏輯與方法（如 probe, remove 函數）。  
  * **Match 機制：** 當一個新設備或新驅動註冊到 Bus 時，Bus 會調用 match() 函數（通常比對名稱或相容性字串 compatible）。如果匹配成功，核心就會觸發驅動的 probe() 函數，正式建立綁定並初始化硬體。

## **二、 中斷處理與下半部 (Interrupts & Bottom Halves)**

### **Q4: 什麼是中斷的上半部 (Top Half) 與下半部 (Bottom Half)？為什麼需要這樣設計？**

* **答案：**  
  * **上半部 (Top Half)：** 硬體中斷觸發後立刻執行的程式。  
    * *特點：* 執行時間極短、**不能睡眠/阻塞**。它只做最核心的事（如讀取硬體狀態、清除中斷標誌、接收資料到緩衝區），然後排程下半部並結束。  
  * **下半部 (Bottom Half)：** 處理非緊急但耗時的工作（如資料解析、封包推播）。  
    * *特點：* 由核心非同步排程執行，允許執行較長時間。  
  * **為什麼需要：** 為了維持系統的即時響應。如果中斷處理時間太長，會導致其他硬體中斷被屏蔽（Blocked），造成掉封包或系統卡頓。

### **Q5: Linux 下半部 (Bottom Half) 有哪些實現方式？它們的差別是什麼？**

* **答案：**  
  主要有三種機制：**Softirq、Tasklet、Workqueue**。

| 機制 | 執行上下文 (Context) | 是否允許睡眠/阻塞？ | 併發性 (Concurrency) |
| :---- | :---- | :---- | :---- |
| **Softirq (軟中斷)** | 中斷上下文 (Interrupt) | **否** (絕對不能睡眠) | 極高。相同核心的 Softirq 可以在不同的 CPU 上同時執行。通常保留給核心關鍵任務（如網路、SCSI）。 |
| **Tasklet** | 中斷上下文 (Interrupt) | **否** (絕對不能睡眠) | 高。基於 Softirq 實現，但同一個 Tasklet 絕對不會在多個 CPU 上同時執行，避免了重入問題（編寫較簡單）。 |
| **Workqueue (工作佇列)** | 進程上下文 (Process) | **是** (可以睡眠、拿鎖、I/O) | 靈活。由核心執行緒 (Kernel thread) 來執行，適合需要長時間處理、存取硬體或配置大塊記憶體的任務。 |

## **三、 同步機制與記憶體管理 (Synchronization & Memory)**

### **Q6: Spinlock (自旋鎖) 與 Mutex (互斥鎖) 有什麼區別？中斷處理程式中可以使用 Mutex 嗎？**

* **答案：**  
  * **Spinlock (自旋鎖)：** \* 當拿不到鎖時，執行緒不會睡眠，而是會在原地進行 **忙碌等待 (Busy-looping)**，持續耗費 CPU 資源。  
    * 適用於鎖持有時間極短的場景。  
    * **可以在中斷上下文中使用。**  
  * **Mutex (互斥鎖)：**  
    * 當拿不到鎖時，執行緒會進入 **睡眠 (Sleep)** 狀態，讓出 CPU 給其他進程執行，直到鎖被釋放後被喚醒。  
    * 適用於鎖持有時間較長、或持鎖期間需要進行 I/O、記憶體配置等可能阻塞的操作。  
  * **中斷程式中可以用 Mutex 嗎？** \* **絕對不行。** 因為中斷上下文沒有進程實體，一旦在中斷中睡眠，排程器 (Scheduler) 將無法正確切換回來，會直接導致系統崩潰 (Kernel Panic)。中斷中若需加鎖，必須使用 spinlock\_t（且通常要用 spin\_lock\_irqsave() 關閉本地中斷）。

### **Q7: 請說明 kmalloc(), vmalloc() 與 malloc() 的區別？**

* **答案：**  
  * **malloc()：** User space 的記憶體配置函數。配置的是虛擬記憶體，實際實體記憶體直到第一次寫入觸發「頁錯誤 (Page Fault)」時才會由核心動態分配。  
  * **kmalloc()：** Kernel space 函數。  
    * 配置的實體記憶體與虛擬記憶體在核心中都是 **連續的**。  
    * 因為實體連續，適合用於 **DMA (直接記憶體存取)** 傳輸。  
    * 基於 Slab/Slub 分配器，適合配置較小的記憶體。  
  * **vmalloc()：** Kernel space 函數。  
    * 配置的 **虛擬記憶體是連續的，但實體記憶體不一定連續**。  
    * 透過修改頁表 (Page Table) 來建立映射，效率比 kmalloc() 低，且**不能用於 DMA**。  
    * 適合用來配置非常大的記憶體緩衝區（如載入核心模組時）。

## **四、 硬體協定與除錯 (Hardware Protocols & Debugging)**

### **Q8: 在寫 I2C 或 SPI 驅動時，DMA 模式與 PIO (Polling/Interrupt) 模式有何不同？如何選擇？**

* **答案：**  
  * **PIO 模式 (Polling / Interrupt)：** \* **Polling：** CPU 用迴圈不斷檢查硬體狀態暫存器，直到傳輸完成。極度浪費 CPU 效能。  
    * **Interrupt：** 每傳輸完一個 Byte/Word，硬體發出中斷，CPU 進入中斷填入下一個數據。適合數據量小、偶發性的傳輸。  
  * **DMA 模式 (Direct Memory Access)：**  
    * CPU 設定好來源位址、目的位址和傳輸長度後，將主導權交給 DMA 控制器。  
    * DMA 直接在記憶體與周邊週邊（如 SPI/I2C FIFO）之間搬移資料，**不需要 CPU 介入**。  
    * 傳輸完成後，DMA 控制器只發出一個中斷通知 CPU。  
  * **選擇策略：** \* 大數據傳輸（如音訊 I2S、顯示面板 SPI、儲存裝置）必用 **DMA**，以釋放 CPU 算力。  
    * 小數據傳輸（如讀取感測器暫存器、幾位元組的控制命令）用 **Interrupt/PIO** 即可，因為建立 DMA 映射 (Cache coherence 處理) 的軟體開銷反而可能大於傳輸時間。

### **Q9: 什麼是 Cache Coherency (快取一致性)？驅動工程師在寫 DMA 驅動時如何處理它？**

* **答案：**  
  * **問題成因：** CPU 讀寫資料會經過高速快取 (Cache)，但 DMA 控制器是直接存取主記憶體 (DRAM)。當 CPU 修改了 Cache 卻還沒寫回 DRAM，或者 DMA 修改了 DRAM 但 CPU 仍讀取舊的 Cache 時，就會導致資料不一致。  
  * **處理方法（Linux 核心提供兩類 API）：**  
    1. **一致性 DMA 映射 (Coherent/Consistent DMA mapping)：** 透過 dma\_alloc\_coherent() 配置記憶體。核心會關閉這塊虛擬記憶體的 Cache 屬性（將其設為 Non-cacheable），使得 CPU 和 DMA 都是直接讀寫 DRAM。優點是簡單安全，缺點是 CPU 存取效能較低。  
    2. **流式 DMA 映射 (Streaming DMA mapping)：** 適用於現有的、由 kmalloc 分配的緩衝區。在 DMA 傳輸前後，手動呼叫 dma\_map\_single() / dma\_sync\_single\_for\_cpu() / for\_device()。這會強迫執行 **Cache Clean**（將 Cache 寫回 DRAM）或 **Cache Invalidate**（將 Cache 標記為無效，強迫 CPU 從 DRAM 重新讀取），維持效能與一致性。

### **Q10: 當 Linux 發生 Kernel Panic (例如 Null pointer dereference) 時，你如何除錯？**

* **答案：**  
  1. **分析 Oops / Panic 訊息：** 觀察核心噴出的 call trace 登錄檔。  
  2. **尋找 PC (Program Counter) / LR (Link Register)：** 找出崩潰發生在特定的函數與偏移量（例如：my\_driver\_read+0x24/0x80）。  
  3. **使用 addr2line 工具：** 如果核心編譯時有開啟偵錯資訊 (-g)，可以使用交叉編譯工具鏈的 addr2line \-e vmlinux \<實體位址\>，直接將出錯的記憶體位址轉換為具體的原始碼檔名與行號。  
  4. **使用 objdump 反組譯：** 將驅動程式的反組譯碼拉出來 (objdump \-D my\_driver.o)，對照出錯的組合語言指令（如 str r1, \[r0\]，如果 r0 為 0 即可確認是空指標）。  
  5. **動態工具：** 使用 printk 進行印出偵錯，或利用核心內建的 ftrace, kprobe 追蹤函數呼叫流程。硬體層面則可掛載 **JTAG (如 Lauterbach / OpenOCD)** 進行硬體級斷點偵錯。
