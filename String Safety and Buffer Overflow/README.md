---
title: ' GDB & Binary Exploitation'
disqus: hackmd
---

GDB & Binary Exploitation
===

# Table of Contents

[TOC]

# GDB (GNU Debugger)
> [!Important]
>GNU開發工具 (GNU toolchain) 是Linux作業系統、嵌入式系統 (Embedded System)、與自由及開放原始碼軟體 (Free and Open Source Software, FOSS) 社群中，最常見、使用得最廣泛的程式開發工具。GNU開發工具是由GNU計劃 (GNU Project) 中的數個程式開發工具所集合而成，可以完成編譯、組譯和連結等步驟，還有自動化設定、自動化編譯、除錯工具等功能。GNU計劃是1983年時理查史托曼 (Richard Stallman) 提出的，目的是要建立一個由自由軟體組成的作業系統。1992年時，Linux核心以GNU通用公共授權條款 (GPL) 釋出，與GNU計劃組成GNU/Linux作業系統，而成為最多人使用的自由軟體作業系統家族，GNU開發工具也成為這些系統最主要的開發工具。
>Ref: https://www.cc.ntu.edu.tw/chinese/epaper/0020/20120320_2005.html

GDB（GNU Debugger）是GNU計畫中的一個強大除錯工具，用於調試C、C++、Fortran等語言編寫的程式。GDB提供了多種功能來幫助開發者定位和修正代碼中的錯誤。
主要功能
1. 啟動程序：可以在GDB內部啟動被調試的程序。
2. 設置斷點：可以在代碼的指定行、函數或條件下設置斷點，程序運行到斷點處時會暫停。
3. 單步執行：支持逐行或逐指令執行代碼，以觀察程序運行狀況。
4. 檢查變量值：可以查看和修改內存中變量的值。
5. 追蹤調用棧：當程序崩潰時，能夠查看調用棧來定位問題源頭。

![image](https://hackmd.io/_uploads/rk3RA_PQ0.png)
## Install GDB
檢查是否安裝GDB
```command
gdb --version
```
使用package manager安裝
``` command
# 在Debian/Ubuntu上
sudo apt-get install gdb

# 在Fedora上
sudo dnf install gdb

# 在Arch Linux上
sudo pacman -S gdb
```

## Start up GDB
在terminal裡，輸入指令 `gdb (用 -g 參數編譯的執行檔檔名)` ，載入可執行文件
```command
gdb myprogram.exe
```

GDB的TUI（Text User Interface）提供了一個基於文本的UI界面，能夠顯示source code、Register和RAM等訊息，參數`-tui`進入TUI模式
```command
gdb -tui myprogram.exe
```

啟動成功之後，基本上就是視情況需要，下指令操作 gdb 囉!
## GDB Basic Command
● 設置斷點
在特定行或函數處設置斷點：
```command
(gdb) break main              # 在 main 函數處設置斷點
(gdb) break myprogram.c:10    # 在 myprogram.c 文件的第10行設置斷點
(gdb) break myfunction        # 在 myfunction 函數處設置斷點
```
● 查看斷點
列出所有當前設置的斷點：
```command
(gdb) info breakpoints
```
● 運行Process
```command
(gdb) run
```
● 逐行執行
```command
(gdb) step      # 單步執行代碼，進入函數內部
(gdb) next      # 單步執行代碼，但不進入函數內部
(gdb) continue  # 繼續執行直到下一個斷點
```
● 顯示變量的值
```command
(gdb) print var        # 顯示變量 var 的值
(gdb) print myarray[0] # 顯示數組 myarray 的第一個元素
```
● 修改變量的值：
```command
(gdb) set var = 10  # 將變量 var 設置為 10
```
● 堆疊追蹤
查看當前Stack，了解process崩潰時的調用過程
```command
(gdb) backtrace   # 顯示當前Stack
```
● 查看source code
```command
(gdb) list        # 顯示當前代碼行及其周圍的代碼
(gdb) list 10     # 顯示從第10行開始的代碼
```
● 刪除斷點
1. 刪除指定斷點
```command
(gdb) delete 1    # 刪除編號為1的斷點
```
2. 刪除所有斷點：
```command
(gdb) delete      # 刪除所有斷點
```
● 退出GDB
結束調試會話：
```command
(gdb) quit
```

# Binary Exploitation
Binary Exploitation 是通過分析和操縱可執行文件的二進制代碼來發現和利用軟體漏洞，以達到未經授權的操作。這通常包括攻擊者試圖破壞程序的正常執行流程，例如獲得未經授權的訪問權限或執行任意代碼。
## ELF (Executable and Linkable Format)
Executable and Linkable Format
常見的二進制文件格式，主要用於描述可執行文件（executable files）、共享庫（shared libraries）、目標文件（object files）等的結構和內容。它具有可移植性、靈活性和廣泛的應用，是Unix和類Unix系統中廣泛使用的標準格式之一。
![image](https://hackmd.io/_uploads/ryAYHuv7C.png)

## x64
x86-64或AMD64，是一種64位元的微處理器架構。它是x86架構的64位擴展，主要由英特爾和AMD公司開發，旨在提供更大的記憶體空間和更高的性能。
### 8 bytes alignment (8位元對齊)
在x64架構中，大多數資料的存取都要求按照8位元（8 bytes）進行對齊。這意味著數據在記憶體中的地址必須是8的倍數，否則將會引發對齊異常（alignment fault）。
例如，一個 64 位的整數或一個指標在記憶體中的地址應該以 0, 8, 16 等（在十六進制中為 0x00, 0x08, 0x10, ...）結尾。

### Stack 0x10 bytes alignment (Stack對齊)
函數調用時，Stack Pointer 必須按照0x10 bytes（16 bytes）對齊。這意味著每次函數調用時，Stack Pointer必須指向一個地址，該地址的最低4位必須為0。這是為了確保 stack 上的每個框架都按 16 位元組對齊，保證 SIMD（單指令多數據）類型的數據和其他可能需要更嚴格對齊的數據結構能正常工作

### Registers
- RSP - Stack Pointer Register
指向 Stack 頂端
當一個新的data被push Stack時，RSP的值會向下移動；當一個data從Stack中pop時，RSP的值會向上移動。
- RBP - Base Pointer Register
指向 Stack 底端
通常用於存儲當前函數的Stack Frame的基址。
- RIP - Program Counter Register
指向當前執行指令 instruction 位置
它存儲了下一條要執行的指令的地址，也就是程序計數器（Program Counter）的值。當一個指令被執行完畢後，RIP的值會被自動更新為下一條指令的地址，從而實現指令的連續執行。

### Assembly
在x64架構中，一些重要的組合語言（Assembly）指令
#### 1. jmp (Jump)
`jmp`指令用於無條件跳轉到指定的地址。它會將指令執行流程直接轉移到目標地址處，而不受任何條件的限制。
```command
jmp label   ; 無條件跳轉到標記 label 處
jmp address ; 無條件跳轉到指定地址處
```
跳至程式某一地址 A (address) 執行
```command
jmp = mov rip, A
```

#### 2. call
`call` 指令用於調用函數。它將下一條指令的地址（即 call 指令後面的指令的地址）壓入stack，然後跳轉到指定的函數地址。
```command
call function_name ; 調用名為 function_name 的函數
call address       ; 調用指定地址處的函數
```
將 call 完後回來緊接著要執行的下一行指令位置 push 到 stack 上儲存起來，再跳過去執行。
```command
call A = push next_rip
	 mov rip, A
```

#### 3. leave
`leave` 指令用於退出stack frame，並恢復父函數的stack frame。它實際上等效於以下兩條指令的組合：mov rsp, rbp 和 pop rbp。
```
leave ; 退出當前函數的stack框架
```
還原至 caller 的 stack frame
```command
mov rsp, rbp
pop rbp
```

#### 4. ret (return)
`ret` 指令用於從函數中返回到調用者。它將返回地址從stack中彈出，然後跳轉到該地址開始執行。
```command
ret ; 從函數中返回到調用者
```

### Stack Frame
![image](https://hackmd.io/_uploads/Hkcsi_PQ0.png)
- [rbp] = old rbp (caller rbp)
- [rbp + 8] = Return Address

## Common Vulnerabilities
- Buffer Overflow (緩衝區溢位):

當程序寫入超過預定大小的數據到緩衝區時，會覆蓋相鄰的記憶體空間，這可能導致任意代碼執行。

- Format String Vulnerability (格式化字符串漏洞):

當程序不正確地處理輸入的格式化字符串時，攻擊者可以讀取或寫入記憶體中的任意位置。
- Heap Exploitation (堆利用):

通過操縱堆管理器的行為，攻擊者可以覆蓋關鍵數據結構並控制程序流。
- Return-Oriented Programming (ROP):

通過鏈接現有程序中的代碼段（gadgets）來構建惡意payload，以繞過防御機制如DEP（數據執行保護）。

# PWN 
pwn 是駭客文化中的一個術語，通常指徹底擊敗或控制某個目標系統。在計算機安全領域，pwn特別指的是成功利用漏洞來獲取系統的控制權。pwn用於競賽、研究以及實際攻擊中。

## pwn 工具和技術
Pwntools 學習資源: https://github.com/Gallopsled/pwntools-tutorial

#### 1.GDB (GNU Debugger)
用於調試可執行文件，分析代碼行為，設置斷點並查看內存狀態。
#### 2. pwntools
一個用於編寫exploit的Python庫，提供了便利的函數來處理IO、編碼/解碼、格式化輸入等。
#### 3. ROPgadget
用於自動化ROP鏈構建的工具，可以在二進制文件中搜索可用的gadgets。
#### 4. radare2
一個開源的逆向工程框架，提供了二進制分析、調試和利用工具。
