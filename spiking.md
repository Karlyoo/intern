# OAI → Full-grid Dump → SpikingRx
## 1.dump
We need to take out the FFT data from OAI, add the code in `nr_dlsch_demodulation` ,this function will generate a OAI Full-grid dump.
```
#include <stdio.h>
#include <stdint.h>
// --- SpikingRx full-grid dump guard ---
static int spx_last_frame = -1;
static int spx_last_slot = -1;
// =====================================================
// === SpikingRx FULL-GRID DUMP (14 symbols × 2048 SC) ==
// =====================================================
{
 static uint32_t spx_dump_idx = 0; // 全域 index，自動遞增
 int frame = proc->frame_rx;
 int slot = proc->nr_slot_rx;
 char fname[256];
 snprintf(fname, sizeof(fname),
 "/tmp/spx_fullgrid_idx%06u.bin", spx_dump_idx);
 FILE *f = fopen(fname, "wb");
 if (f) {
 // ----- header -----
 uint16_t header[8];
 header[0] = (uint16_t)frame;
 header[1] = (uint16_t)slot;
 header[2] = 0; // start_symbol (unused)
 header[3] = fp->symbols_per_slot; // usually 14
 header[4] = fp->ofdm_symbol_size; // usually 2048
 header[5] = 0; // rx_ant=0
 header[6] = 0; // cw=0
 header[7] = 0; // reserved
 fwrite(header, sizeof(uint16_t), 8, f);
 int ofdm = fp->ofdm_symbol_size;
 int nsym = fp->symbols_per_slot;
 // dump 14 symbols × 2048 complex16
 for (int sym = 0; sym < nsym; sym++) {
 fwrite(&rxdataF[0][sym * ofdm], sizeof(c16_t), ofdm, f);
 }
 fclose(f);
 printf("FULLGRID_IDX=%u frame=%d slot=%d → %s\n",
 spx_dump_idx, frame, slot, fname);
 }
 spx_dump_idx++; // 每次 dump 完自動 +1
}
```
## 2.compile OAI（gNB + UE）and run
```
#clean the old one
cd ~/openairinterface5g/cmake_targets/ran_build/build
rm -rf CMakeCache.txt CMakeFiles
```
```
#rebuild 
cd ~/openairinterface5g/cmake_targets
sudo ./build_oai --gNB --nrUE -w SIMU
```
```
# run the oai-gnb 
cd ~/openairinterface5g
source oaienv
cd cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --rfsim --sa -E
```
```
# run the oai-nrue 
cd ~/openairinterface5g
source oaienv
cd cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 -E --rfsim --sa --uicc0.imsi 001010000000001 --rfsimulator.serveraddr 127.0.0.1

```
* before run gnb and ue, you need to open cn5g to get the full log. the link below shows how to config your cn5g and gnb in the same machine.
* https://github.com/bmw-ece-ntust/L-release-near-RT-RIC-/blob/master/gNB_UE_5gCN.md

After you finich  running, you can use the command to check the dump file. You will see something like this:
```sh
ls -ltr /tmp/spx_fullgrid_*.bin
# -rw-r--r-- 1 root root 61440 十一 26 17:35 /tmp/spx_fullgrid_f0_s0.bin
```
# 3. SpikingRx
```mermaid
graph TD
    %% 定義樣式
    classDef script fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef file fill:#fff9c4,stroke:#fbc02d,stroke-width:2px,stroke-dasharray: 5 5;
    classDef data fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;

    subgraph Step_1 [Step 1: first Forward Inference]
        direction TB
        1_Env[1.  source .venv]
        1_Run[2. run inference script]:::script
        1_Input(read OAI Dump /tmp/):::data
        1_Out(output src/inference/out/):::file
        
        1_Env --> 1_Run
        1_Input --> 1_Run
        1_Run -->|" 32x32 -> Forward"| 1_Out
    end

    subgraph Step_2 [Step 2: model Training]
        direction TB
        2_Run[ train_spikingrx.py]:::script
        2_Log( Log: docs/results/train/):::file
        2_Ckpt(save: checkpoints/spikingrx_checkpoint.pth):::data

        2_Run --> 2_Log
        2_Run --> 2_Ckpt
    end

    subgraph Step_3 [Step 3: Re-Inference after training]
        direction TB
        3_Run[run inference script]:::script
        3_Out(Update output results):::file

        3_Run --> 3_Out
    end

    %% 流程連接
    Step_1 --> Step_2
    Step_2 --> Step_3

    %% 關鍵數據流：訓練好的權重被 G 步驟讀取
    2_Ckpt -.->|" Checkpoint"| 3_Run

    %% 引用標註
    %% Step 1 [cite: 34-39]
    %% Step 2 [cite: 45-53]
    %% Step 3 [cite: 54-57]
```

```
#1.env
cd ~/SpikingRx-on-OAI
source .venv/bin/activate
```
```
# 2. inference（read OAI dump automatically 、Convert format（32x32），make forward inferences ）
python src/inference/run_spikingrx_on_oai_dump.py
```

```
# 3.training
cd ~/SpikingRx-on-OAI
python src/train/train_spikingrx.py
#checkpointmodel will be in：
#checkpoints/spikingrx_checkpoint.pth
```
```
# 4.inference again
cd ~/SpikingRx-on-OAI
python src/inference/run_spikingrx_on_oai_dump.py
```






# 如果要和oai比較
新的完整工作流程：

1.執行 OAI：sudo ./nr-uesoftmodem ...
當 UE 接收到 PDSCH，nr_dlsch_demodulation.c 中的程式碼會執行，產生 /tmp/spx_fullgrid_f...bin 檔案。

2.執行 Python：python3 run_spikingrx_on_oai_dump.py
找到最新的 spx_fullgrid_...bin 檔案，進行模型推論。
推論完成後，它會產生一個新的 LLR 檔案 /tmp/spx_llrs_f...bin。

3.OAI 繼續執行：
OAI 的處理流程繼續往下走，呼叫到 nr_dlsch_decoding 函式。
偵測到 /tmp/spx_llrs_f...bin 檔案的存在，於是讀取它，並用模型產生的 LLR 來進行 LDPC 解碼。

遇到時間問題：
目前的簡易流程：

OAI 存一個 IQ 檔。
Python 腳本處理最新的 IQ 檔，存一個 LLR 檔。
OAI 讀取對應的 LLR 檔。
這裡的「最新」和「對應」就是問題所在。如果 Python 處理得不夠快，或者 OAI 產生了多個 PDSCH 的 IQ 檔，整個流程就會錯亂。

解決方案：使用唯一的識別碼進行標記
要解決這個問題，我們必須為每一次 PDSCH 的傳輸嘗試建立一個唯一的標記。在 5G NR 中，最適合的唯一識別碼組合是：

Frame (訊框號)
Slot (時槽號)
HARQ Process ID (HARQ 流程 ID)
HARQ ID 至關重要，因為在同一個 slot 中，可能會有多個 HARQ process 在運作；而且對於同一個 HARQ process，還可能會有多次重傳 (retransmission)。

我們需要將這個唯一的 (frame, slot, harq_pid) 標記應用到整個鏈路的所有檔名中。
