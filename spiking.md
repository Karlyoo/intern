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
```
ls -ltr /tmp/spx_fullgrid_*.bin
```
```
-rw-r--r-- 1 root root 61440 十一 26 17:35 /tmp/spx_fullgrid_f0_s0.bin
```
# 3. SpikingRx
```mermaid
graph TD
    %% 定義樣式
    classDef script fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef file fill:#fff9c4,stroke:#fbc02d,stroke-width:2px,stroke-dasharray: 5 5;
    classDef data fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;

    subgraph Step_E [Step E: first Forward Inference]
        direction TB
        E_Env[1.  source .venv]
        E_Run[2. run inference script]:::script
        E_Input(read OAI Dump /tmp/):::data
        E_Out(output src/inference/out/):::file
        
        E_Env --> E_Run
        E_Input --> E_Run
        E_Run -->|" 32x32 -> Forward"| E_Out
    end

    subgraph Step_F [Step F: model Training]
        direction TB
        F_Run[ train_spikingrx.py]:::script
        F_Log( Log: docs/results/train/):::file
        F_Ckpt(save: checkpoints/spikingrx_checkpoint.pth):::data

        F_Run --> F_Log
        F_Run --> F_Ckpt
    end

    subgraph Step_G [Step G: Re-Inference after training]
        direction TB
        G_Run[run inference script]:::script
        G_Out(Update output results):::file

        G_Run --> G_Out
    end

    %% 流程連接
    Step_E --> Step_F
    Step_F --> Step_G

    %% 關鍵數據流：訓練好的權重被 G 步驟讀取
    F_Ckpt -.->|" Checkpoint"| G_Run

    %% 引用標註
    %% Step E [cite: 34-39]
    %% Step F [cite: 45-53]
    %% Step G [cite: 54-57]
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
