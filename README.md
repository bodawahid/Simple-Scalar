
# Dynamic Branch Prediction Simulation

## Problem Definition

This simulation explores the performance of various **dynamic branch prediction schemes** using the `sim-outorder` simulator from the **SimpleScalar toolset**. The goal is to develop a deeper understanding of how different prediction strategies impact processor performance, particularly in relation to control hazards in pipelines.

The processor model used in this assignment mimics a simple, in-order CPU with the following configuration:

- One instruction per cycle: fetch, decode, issue, execute, and commit.
- One functional unit of each type.
- Caches and TLBs configured as:
  - 8KB, 2-way set-associative L1 instruction and data caches (32-byte blocks)
  - 256KB, direct-mapped L2 unified cache (32-byte blocks)
  - 8-way, 128-entry data TLB
  - 4-way, 64-entry instruction TLB
  - LRU block replacement policy
  - 4KB page size

### Branch Prediction Strategies Evaluated

Three prediction schemes are tested and compared:

1. **Static Prediction**:
   - Backward branches are predicted as taken.
   - Forward branches are predicted as not taken.

2. **2-bit Dynamic Branch Prediction**:
   - Uses saturating counters for each branch instruction to dynamically predict future behavior based on past outcomes.

3. **Correlating Branch Prediction**:
   - Employs a global history register and a pattern history table (PHT) to improve prediction by correlating outcomes of recent branches.

### Objective

The simulation aims to analyze and compare these prediction strategies with respect to:

- **Branch frequency and characterization** (forward/backward, taken/not taken)
- **Prediction accuracy** across schemes
- **Impact of history bits** on predictor performance
- **Differences in instruction fetch vs commit counts**
- **Usefulness of traditional static prediction schemes**

Applications from the SPEC95 benchmark suite are used as input, each precompiled for the simulator.

## Problems Encountered and Solutions

### 🛠 Problem 1: Static Branch Prediction Not Available

**Issue:**  
The simulator did not provide a built-in static prediction mode where backward branches are predicted as taken and forward branches as not taken.

**Solution:**  
To implement this manually, I modified the `bpred_lookup()` function inside `bpred.c`. I added a condition to check the predictor class (`BPredStatic`) and used the branch target (`btarget`) compared to the branch address (`baddr`) to determine the direction of the branch.

If the branch was backward (`btarget < baddr`), it was predicted as taken; otherwise, as not taken. The prediction result was also assigned to the `dir_update_ptr` fields for accurate tracking.

This custom logic allowed static prediction to work as expected when selected in the simulator configuration.

```c
if (!(MD_OP_FLAGS(op) & F_CTRL))
    return 0;
    pred->lookups++;
  if (btarget < baddr) {
    dir_update_ptr->dir.bimod = TRUE;
    dir_update_ptr->dir.twolev = TRUE;
    dir_update_ptr->dir.meta = TRUE;
    return btarget;  // Predict taken
  } else {
    dir_update_ptr->dir.bimod = FALSE;
    dir_update_ptr->dir.twolev = FALSE;
    dir_update_ptr->dir.meta = FALSE;
    return baddr + sizeof(md_inst_t);  // Predict not taken
  }
}
```
### 🛠 Problem 2: Branch Direction Statistics Not Displayed

**Issue:**  
The simulator did not display detailed statistics for forward and backward branches—specifically, how many were taken or not taken during both execution and commit stages. These metrics were essential for analyzing branch behavior in relation to prediction strategies.

**Solution:**  
To collect and report these statistics, the following steps were taken inside the file `sim-outorder.c` :

#### 1. **Counters Defined**

Eight new counters were added to track forward/backward branches and their taken/not taken states, both during execution and commit:

```c
static counter_t forward_branches_taken;
static counter_t backward_branches_taken;
static counter_t forward_branches_not_taken; 
static counter_t backward_branches_not_taken; 
static counter_t committed_forward_taken;
static counter_t committed_forward_not_taken;
static counter_t committed_backward_taken; 
static counter_t committed_backward_not_taken;
```
### 2. Counters Initialized

All counters were initialized in `lsq_init()`:

```c
forward_branches_taken = 0;
forward_branches_not_taken = 0; 
backward_branches_taken = 0; 
backward_branches_not_taken = 0; 
committed_forward_taken = 0; 
committed_forward_not_taken = 0; 
committed_backward_taken = 0; 
committed_backward_not_taken = 0;
```
### 3. Counting During Execution

by comparing `target_PC` to `regs.regs_PC` to detect forward vs. backward branches.

```c
if (MD_OP_FLAGS(op) & F_CTRL) {
  sim_total_branches++;
  if (target_PC > regs.regs_PC) {
    if (br_taken)
     forward_branches_taken++;
    else
     forward_branches_not_taken++;
  } else if (target_PC < regs.regs_PC) {
    if (br_taken)
     backward_branches_taken++;
    else
     backward_branches_not_taken++;
  }
}
```
### 4. Counting During Commit

Similarly, committed branch directions were recorded when instructions were finalized:

```c
if (MD_OP_FLAGS(op) & F_CTRL) {
  sim_num_branches++;
  if (target_PC > regs.regs_PC) {
    if (br_taken)
     committed_forward_taken++;
    else
     committed_forward_not_taken++;
  } else if (target_PC < regs.regs_PC) {
    if (br_taken)
     committed_backward_taken++;
    else
     committed_backward_not_taken++;
  }
}
```
### 5. Registering Statistics

Finally, the counters were registered for output in the statistics report using `stat_reg_counter()`:

```c
stat_reg_counter(sdb, "forward_taken", "number of forward branches that were taken", &forward_branches_taken, 0, NULL);
stat_reg_counter(sdb, "forward_not_taken", "number of forward branches that were not taken", &forward_branches_not_taken, 0, NULL);
stat_reg_counter(sdb, "backward_taken", "number of backward branches that were taken", &backward_branches_taken, 0, NULL);
stat_reg_counter(sdb, "backward_not_taken", "number of backward branches that were not taken", &backward_branches_not_taken, 0, NULL);
stat_reg_counter(sdb, "committed_forward_taken", "number of committed forward branches that were taken", &committed_forward_taken, 0, NULL);
stat_reg_counter(sdb, "committed_forward_not_taken", "number of committed forward branches that were not taken", &committed_forward_not_taken, 0, NULL);
stat_reg_counter(sdb, "committed_backward_taken", "number of committed backward branches that were taken", &committed_backward_taken, 0, NULL);
stat_reg_counter(sdb, "committed_backward_not_taken", "number of committed backward branches that were not taken", &committed_backward_not_taken, 0, NULL);
```
