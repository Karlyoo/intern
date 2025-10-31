## SRS signal test
### get the SRS signal from OAI gNB-UE
Ref: https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/common/utils/T/DOC/T/howto_new_trace.md

#### 1 - create a new trace
In the `file common/utils/T/T_messages.txt` we add what we want to check. Take the SRS signal for example:
```
ID = GNB_PHY_SRS_CHANNEL_ESTIMATE
    DESC = gNodeB SRS channel estimation in the time domain
    GROUP = ALL:PHY:GRAPHIC:HEAVY:GNB
    FORMAT = int,gNB_ID : int,rnti : int,frame : int,subframe : int,antenna : buffer,chest_t
```

- You create an ID for the new trace.
- You give the format as a list of variables with there types.

Then, in the file `openair1/PHY/NR_ESTIMATION/nr_ul_estimation.c`, in function `nr_srs_channel_estimation()`, we add, at the place where we want to trace.
If you are interested in time-domain channel estimation,
```
T(GNB_PHY_SRS_CHANNEL_ESTIMATE,
             T_INT(gNB->Mod_id),                            // gNB ID
             T_INT(srs_pdu->rnti),                          // RNTI
             T_INT(frame),                                  // Frame
             T_INT(slot),                                   // Slot
             T_INT(ant),                                    // Rx Antenna
             T_INT(p_index),                                // UE Port
             T_BUFFER(srs_estimated_channel_time[ant][p_index], // data you want to collect
                      gNB->frame_parms.ofdm_symbol_size * sizeof(int32_t), // data length
                      "srs_chest_t")                        // name
           );
```
And you need to recompile it,
```
cd ~/openairinterface5g/cmake_targets/
sudo ./build_oai -w SIMU --gNB --build-e2 --ninja
```

and make a trace text:
```
cd ~/openairinterface5g/common/utils/T/tracer/
make
```

And start your connection:


