# ISAC Xapp
ref:https://bubbleran.com/docs/devops-guide/flexric/developers-guide/xapp/xapp-programming/usr

https://bubbleran.com/docs/user-guide/xapp-training/lab10

```mermaid
graph LR
    %% --- 設定樣式 (讓圖比較緊湊) ---
    classDef hardware fill:#fff9c4,stroke:#fbc02d,stroke-width:5px;
    classDef software fill:#e1f5fe,stroke:#0277bd,stroke-width:5px;
    classDef highlight fill:#c8e6c9,stroke:#2e7d32,stroke-width:5px;
    classDef signal fill:#ffcdd2,stroke:#c62828,stroke-width:5px;

    %% --- 1. 硬體端 (Hardware) ---
    subgraph HW_Layer ["Physical & Hardware"]
        direction LR
        Wave(("Radio Waves")):::signal
        Antenna["Antenna"]:::hardware
        USRP["USRP (ADC)"]:::hardware
    end

    %% --- 2. 軟體端 (OAI gNB) ---
    subgraph OAI_Layer ["OAI gNB Stack"]
        direction TB
        PHY["PHY Layer (L1)<br>(FFT/CSI)"]:::software
        MAC["MAC Layer (L2)"]:::software
        Agent["E2 Agent"]:::highlight
    end

    %% --- 3. 控制端 (RIC) ---
    subgraph RIC_Layer ["RIC & xApp"]
        direction LR
        RIC["FlexRIC"]:::software
        xApp["ISAC xApp"]:::highlight
    end

    %% --- 連線流向 ---
    %% 硬體流
    Wave --> Antenna
    Antenna --> USRP
    USRP == "I/Q Samples" ==> PHY

    %% 軟體處理流
    PHY --> MAC
    
    %% 關鍵攔截點 (JCAS)
    PHY -. "Tap: CSI/Radar" .-> Agent
    MAC -. "Tap: Stats" .-> Agent

    %% E2 傳輸
    Agent == "E2AP (Port 36421)" ==> RIC
    RIC == "E42 Protocol" ==> xApp
```


```mermaid

    graph TD
    %% --- Nodes Definition ---
    UE(User Equipment<br/>OAI nr-uesoftmodem)
    RF((RF Simulator))
    
    subgraph RAN_Layer [Radio Access Network]
        gNB(gNB Base Station<br/>OAI nr-softmodem)
        Agent[E2 Agent<br/>Fix: Port 36421]
    end

    subgraph RIC_Layer [RIC Platform]
        RIC(FlexRIC<br/>Near-RT RIC)
        DB[(SDL Database<br/>SQLite)]
    end

    subgraph App_Layer [xApps]
        Monitor[Monitor xApp<br/>KPM/MAC Stats]
        ISAC[Custom ISAC xApp<br/>BubbleRAN Code]
    end

    %% --- Connections ---
    UE <==>|Traffic Generation| RF
    RF <==>|Radio Link| gNB
    
    gNB --- Agent
    Agent <==>|E2 Interface<br/>SCTP 36421| RIC
    
    RIC <-->|E42 Protocol| Monitor
    RIC <-->|E42 Protocol| ISAC
    RIC --- DB

    %% --- Styling ---
    style Agent fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style RF fill:#ffffcc,stroke:#aaaa00
    style Monitor fill:#e1f5fe
    style ISAC fill:#e1f5fe
```

##  FlexRIC
confirm the message can go through from gnb to flexRIC
```
[iApp]: nearRT-RIC IP Address = 127.0.0.1, PORT = 36422
[NEAR-RIC]: Initializing Task Manager with 2 threads 
[E2AP]: E2 SETUP-REQUEST rx from PLMN   1. 1 Node ID 3584 RAN type ngran_gNB
[NEAR-RIC]: Accepting RAN function ID 2 with def = ORAN-E2SM-KPM 
[NEAR-RIC]: Accepting RAN function ID 3 with def = ORAN-E2SM-RC 
[NEAR-RIC]: Accepting RAN function ID 142 with def = MAC_STATS_V0 
[NEAR-RIC]: Accepting RAN function ID 143 with def = RLC_STATS_V0 
[NEAR-RIC]: Accepting RAN function ID 144 with def = PDCP_STATS_V0 
[NEAR-RIC]: Accepting RAN function ID 145 with def = SLICE_STATS_V0 
[NEAR-RIC]: Accepting RAN function ID 146 with def = TC_STATS_V0 
[NEAR-RIC]: Accepting RAN function ID 148 with def = GTP_STATS_V0 
[iApp]: E42 SETUP-REQUEST rx
[iApp]: E42 SETUP-RESPONSE tx
[iApp]: SUBSCRIPTION-REQUEST RAN_FUNC_ID 142 RIC_REQ_ID 1 tx 
[iApp]: SUBSCRIPTION-REQUEST RAN_FUNC_ID 143 RIC_REQ_ID 2 tx 
[iApp]: SUBSCRIPTION-REQUEST RAN_FUNC_ID 144 RIC_REQ_ID 3 tx 
[iApp]: SUBSCRIPTION-REQUEST RAN_FUNC_ID 148 RIC_REQ_ID 4 tx 
[NEAR-RIC]: SUBSCRIPTION DELETE REQUEST tx RAN FUNC ID 142 RIC_REQ_ID 1021 

```

## Monitor Xapp
check the status and calculate the latency . 
```
 Registered node 0 ran func id = 146 
 Registered node 0 ran func id = 148 
 [xApp]: E42 RIC SUBSCRIPTION REQUEST tx RAN_FUNC_ID 142 RIC_REQ_ID 1 
[xApp]: SUBSCRIPTION RESPONSE rx
[xApp]: Successfully subscribed to RAN_FUNC_ID 142 
[xApp]: E42 RIC SUBSCRIPTION REQUEST tx RAN_FUNC_ID 143 RIC_REQ_ID 2 
[xApp]: SUBSCRIPTION RESPONSE rx
[xApp]: Successfully subscribed to RAN_FUNC_ID 143 
[xApp]: E42 RIC SUBSCRIPTION REQUEST tx RAN_FUNC_ID 144 RIC_REQ_ID 3 
[xApp]: SUBSCRIPTION RESPONSE rx
[xApp]: Successfully subscribed to RAN_FUNC_ID 144 
MAC ind_msg latency = 378 μs
```

## ISAC Xapp
From bubble RAN　lab10, we can get a code for ISAC Xapp:
```
## xapp_isac_srs.c
#include "../include/src/xApp/e42_xapp_api.h"
#include "../include/src/util/alg_ds/alg/defer.h"
#include "../include/src/util/time_now_us.h"
#include "../include/src/sm/isac_sm/isac_sm_id.h"
#include <poll.h>
#include <stdio.h>

void cb(sm_ag_if_rd_t const *rd, global_e2_node_id_t const *n)
{
  assert(n != NULL);
  assert(rd != NULL);
  assert(rd->type == INDICATION_MSG_AGENT_IF_ANS_V0);
  assert(rd->ind.type == ISAC_STATS_V0);

  isac_ind_data_t const* ind = &rd->ind.isac;
  isac_ind_msg_t const* msg = ind->msg; // needed for flexible array member
  printf("Timestamp %ld latency %ld \n", msg->tstamp, time_now_us() - msg->tstamp);
  printf("len_srs_iq %lu \n", msg->len_srs_iq);

//  int16_t const* srs_iq = msg->srs_iq;
  for(size_t i = 0; i < msg->len_srs_iq; ++i){
//    printf("re %d im %d noise_re %d noise_im %d \n",srs_iq[4*i], srs_iq[4*i+1], srs_iq[4*i+2], srs_iq[4*i+3]);
  }
}

static
isac_sub_data_t gen_isac_sub_data(void)
{
  isac_sub_data_t dst = {0};
  dst.et.type = APERIODIC_ISAC_EVENT;
  dst.et.aper = SRS_ISAC_EVENT_TRIGGER_APER;

  return dst;
}

int main(int argc, char *argv[])
{
  assert(argc == 2);

  //Init the xApp
  init_xapp_api(argv[1]);
  poll(NULL, 0, 1000);

  e2_node_arr_xapp_t arr = e2_nodes_xapp_api();
  defer({ free_e2_node_arr_xapp(&arr); });

  assert(arr.len > 0);
  sm_ans_xapp_t* hndl = calloc(arr.len, sizeof(sm_ans_xapp_t));
  defer({ free(hndl); });

  // Generate RAN CONTROL Subscription
  isac_sub_data_t isac_sub = gen_isac_sub_data();
  defer({ free_isac_sub_data(&isac_sub); });

  // Retrieve information about the E2 Nodes in the callback func (cb)
  for(size_t i = 0; i < arr.len; ++i){
    hndl[i] = report_sm_xapp_api(&arr.n[i].id, SM_ISAC_ID, &isac_sub, cb);
    assert(hndl[i].success == true);
  }

  poll(NULL, 0, 2000);

  for(size_t i = 0; i < arr.len; ++i){
    rm_report_sm_xapp_api(hndl[i].u.handle);
  }

  // stop the xApp
  while(try_stop_xapp_api() == false)
    poll(NULL, 0, 1000);

  return 0;
}
```
However the public FlexRIC doesn't have`isac_sm` and it require physical equipment to get correct data.

I try to fix `kpm_sm` to familiar with the xapp deployment workflow. 
```
#include "xapp_infra.h"
#include "lib/flexric.h"
#include "sm/kpm_sm/kpm_sm_id.h" // Switched to KPM

// This is the Callback; it will be called when the RIC receives data
void my_callback(sm_ag_if_rd_t const* rd)
{
    // Check if it is KPM data
    if (rd->type == INDICATION_MSG_AGENT_IF_ANS_V0 && rd->ind.type == KPM_STATS_V3_0) {
        printf("[Mock ISAC] Data received! (It's KPM, but the flow is correct)\n");
        
        // Here we simulate reading ISAC data
        // In real ISAC code, you would access: msg->distance, msg->angle
        // Here we are just proving that the callback was triggered
        
        // Simply print a timestamp to prove it is alive
        int64_t now = time_now_us();
        printf(" >> Timestamp: %ld \n", now);
    }
}

int main(int argc, char* argv[])
{
    // 1. Initialization (Exactly the same as Lab 10)
    fr_args_t args = init_fr_args(argc, argv);
    init_xapp_api(&args);
    sleep(1);

    // 2. Connection Check
    e2_node_connected_t* nodes = get_e2_node_connected();
    if (nodes->len > 0) {
        printf("[Mock ISAC] Connected to gNB, preparing to subscribe...\n");

        // 3. Subscription Setup (Switched ISAC ID to KPM ID)
        // In the future when you have ISAC code, just change this to SM_ISAC_ID
        const int sm_id = SM_KPM_ID; 

        inter_xapp_e2_sub_req_t req = {0};
        req.net.id = nodes->n[0].id;
        
        // KPM requires a simple Action Definition (this part is slightly more complex than ISAC, but we use default values)
        sm_io_ag_ran_func_def_t func_def = {0};

        // Send subscription!
        subscribe_xapp_sm(sm_id, &req, &func_def, my_callback);
        printf("[Mock ISAC] Subscription request sent!\n");
    } else {
        printf("[Mock ISAC] Error: No gNB found!\n");
    }

    // 4. Wait for data (Exactly the same as Lab 10)
    while(1) {
        sleep(1);
    }

    return 0;
}
```
