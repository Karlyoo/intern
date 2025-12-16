# ISAC Xapp

✅ action item: from chei-chun 
-
- trace the code in both oai and Flexric 
- the newest version of FlexRIC for ISAC function .——> it needs the private repository,requires Eurecom Gitlab account.
  
REF:

[bubble ran xapp-programing](https://bubbleran.com/docs/devops-guide/flexric/developers-guide/xapp/xapp-programming/usr)

[bubble ran xapp-train-lab10 : isac](https://bubbleran.com/docs/user-guide/xapp-training/lab10)

[behining the ISAC](https://docs.google.com/presentation/d/1N7rEccMr3gYtHP5ud3Wo_UydceQRh-IQ/edit?slide=id.g33d3e013c19_0_27#slide=id.g33d3e013c19_0_27)

## Flow between oai-gnb and flexRIC 
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
```
:~/flexric/build/examples/ric$ ./nearRT-RIC
```
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
I use the monitor xapp to familiar with the subscription and indication part.

The Monitor xApp (xapp_gtp_mac_rlc_pdcp_moni) is the standard reference application in FlexRIC. It serves as a "dashboard" for the RAN, simultaneously subscribing to multiple protocol layers (MAC, RLC, PDCP, GTP) to gather real-time statistics.
```mermaid
sequenceDiagram
    autonumber
    %% Define Participants
    participant xApp as Monitor xApp<br>(Client)
    participant RIC as FlexRIC<br>(Server)
    participant Agent as E2 Agent<br>(gNB)

    %% --- Phase 1: Subscription Setup ---
    Note over xApp, Agent: 🟢 Phase 1: Subscription Setup (One-time Handshake)
    
    xApp->>RIC: E42 Subscription Request<br/>(Target: MAC Layer, SM ID=142, Interval=10ms)
    
    Note right of RIC: RIC validates ID &<br/>translates message
    
    RIC->>Agent: E2AP Subscription Request<br/>(Forward to gNB)
    
    Agent-->>Agent: Internal Config: Start Timer<br/>(Bind to MAC data source)
    
    Agent-->>RIC: E2AP Subscription Response<br/>(Success / Failure)
    
    RIC-->>xApp: E42 Subscription Response<br/>(Subscription Handle ID = 1)

    %% --- Phase 2: Indication Loop ---
    Note over xApp, Agent: 🔵 Phase 2: Indication Loop (Continuous Reporting)
    
    loop Every 10ms (or Event Trigger)
        Agent-->>Agent: 📸 Data Snapshot<br/>(Capture MAC Throughput/Buffer)
        
        Agent->>RIC: E2AP Indication Message<br/>(Encoded ASN.1 Data)
        
        RIC->>xApp: E42 Indication Message<br/>(Forward via IPC/Socket)
        
        activate xApp
        xApp-->>xApp: ⚡ Trigger my_callback()<br/>Parse Data -> Log -> Calculate Latency
        deactivate xApp
    end

    %% --- Phase 3: Termination ---
    Note over xApp, Agent: 🔴 Phase 3: Termination (Cleanup)
    xApp->>RIC: Subscription Delete Request
    RIC->>Agent: Stop Reporting
    Agent-->>Agent: Stop Timer
```
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
