# ChampSim

ChampSim is a trace-based simulator for microarchitecture research.

Useful trace links:
- Traces (access requires an LDAP ID): https://drive.google.com/drive/folders/1zYchkn-M1auZp_l5wRkIzAcDYPkNtW3H?usp=sharing
- ChampSim wiki: https://champsim.github.io/ChampSim/master/index.html

## Clone the repository

```bash
git clone https://github.com/AnkitB018/microarchitecture-performance-engineering.git
```

## Compile

To compile ChampSim, specify three parameters: the L1D prefetcher, the L2C replacement policy, and the binary name.

For example, `./build_champsim.sh no lru baseline` builds a single-core processor with a hashed perceptron branch predictor, no L1D data prefetcher, and the LRU replacement policy for the L2C.

You can give the binary any name. Custom names help distinguish binaries that use the same configuration.

```bash
./build_champsim.sh ${L1D_PREFETCHER} ${L2C_REPLACEMENT} ${BINARY_NAME}

./build_champsim.sh no lru baseline
```

## Run simulation

```bash
./bin/[BINARY] -warmup_instructions [N_WARM] -simulation_instructions [N_SIM] -traces [TRACE_DIR]/[TRACE]
./bin/hashed_perceptron-no-no-no-no-no-no-no-lru-lru-lru-lru-lru-lru-lru-lru-1core-baseline -warmup_instructions 25000000 -simulation_instructions 25000000 -traces ../traces/trace1.champsimtrace.xz
```

Where:
- `${BINARY}`: ChampSim binary compiled by `build_champsim.sh` (for example, `hashed_perceptron-no-no-no-no-no-no-no-lru-lru-lru-lru-lru-lru-lru-lru-1core-baseline`)
- `${N_WARM}`: number of instructions for the warmup period (25 million)
- `${N_SIM}`: number of instructions for the detailed simulation (25 million)
- `${TRACE_DIR}`: directory containing the trace (for example, `../traces/`)
- `${TRACE}`: name of the trace (for example, `trace1.champsimtrace.xz`)

## Evaluate the simulation

ChampSim measures IPC (instructions per cycle) as its main performance metric. It also prints other useful metrics at the end of each simulation.

## Install GCC 7 on Ubuntu

```bash
sudo apt update
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
vim /etc/apt/sources.list
```

Update the last line with:

```bash
deb [arch=amd64] http://archive.ubuntu.com/ubuntu focal main universe
```

Then run:

```bash
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get install gcc-7
sudo apt-get install g++-7
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-7 0
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 0
```

If GCC and G++ are already present in `/usr/bin`, configure the alternatives using:

```bash
sudo update-alternatives --config g++
sudo update-alternatives --config gcc
```

---

# Implementation Overview

This project explores prefetching and cache replacement techniques in ChampSim. The main implementations are described below.

## 1. IP-Stride Prefetcher (`ip_stride.l1d_pref`)

The IP-stride prefetcher learns memory-access patterns separately for each load instruction.

- Maintains 24 fully associative instruction trackers, storing the instruction PC, previous cache-line address, previous stride, and LRU information.
- Calculates the stride using consecutive cache-line addresses.
- When the current stride matches the previous stride, it treats the pattern as stable and prefetches the next three cache lines.
- Stops prefetching at a 4 KB page boundary.
- When the MSHR occupancy is high, prefetched lines are directed to L2 instead of L1 to reduce pressure on the L1 cache.

This approach works well for regular, constant-stride access patterns but cannot effectively capture changing or non-uniform stride sequences.

## 2. Signature Path Prefetcher (SPP) (`spp.l1d_pref`)

SPP extends stride prediction to handle non-uniform access patterns by using recent stride history.

- **IP Table (IPT):** Maintains 64 direct-mapped entries containing per-instruction access history, including the previous offset, stride, confidence, and signature.
- **Stride detection:** First checks whether an instruction follows a stable stride. If its stride confidence reaches the threshold, it uses conventional stride-based prefetching.
- **Signature generation:** For non-uniform patterns, each instruction maintains a 12-bit rolling signature. The signature is updated as:

  ```cpp
  sig = ((sig << 3) ^ (stride & 0x7F)) & 0xFFF;
  ```

  Each update incorporates the latest stride while retaining a compressed representation of recent stride history (approximately four stride updates).

- **Pattern Table (PT):** A 4096-entry table indexed by the signature. Each entry stores the stride predicted to follow that signature and a confidence counter. The table is trained using actual demand accesses.
- **Lookahead prediction:** Starting from the current signature, SPP looks up the predicted stride, issues a prefetch, and generates a speculative next signature using that predicted stride. This signature is then used to look up the next prediction. Repeating this process allows SPP to follow sequences of different strides.
- **Adaptive degree:** Adjusts the prefetch degree based on useful-prefetch accuracy measured over windows of 256 issued prefetches. Higher accuracy permits more aggressive lookahead, while lower accuracy reduces the degree to limit bandwidth usage and cache pollution.

Unlike IP-stride, SPP can learn and predict recurring sequences of different strides.

## 3. Dynamic I-RRIP Replacement Policy (`dyn_irrip.l2c_repl`)

Dynamic I-RRIP combines instruction-aware cache replacement with set dueling to adapt the L2 replacement policy to the workload.

- **RRIP:** Uses a 2-bit Re-Reference Prediction Value (RRPV), where lower values indicate likely near-future reuse and RRPV 3 indicates a line is a good eviction candidate.
- **Instruction-aware insertion:** Under I-RRIP, instruction lines are inserted with RRPV 1, while data lines are inserted with RRPV 2. Hits promote lines to RRPV 0. During eviction, data lines at RRPV 3 are preferred; otherwise, any eligible line can be evicted.
- **Set dueling:** Selects every 32nd set as a leader set. These leader sets always use standard RRIP, while the remaining follower sets use either RRIP or I-RRIP depending on the workload.
- **Decision logic:** Tracks instruction and total accesses/misses in leader sets. At the end of each epoch of 4096 leader accesses, it calculates the instruction-traffic ratio. If the ratio exceeds the configured threshold (0.25), follower sets use I-RRIP; otherwise, they use standard RRIP.
- **Prefetch awareness:** Prefetched lines are inserted with RRPV 3, and a prefetch hit does not promote them. Prefetches are also excluded from the set-dueling counters so that prefetch traffic does not distort the workload's instruction-traffic measurement.

This allows the replacement policy to favour instruction lines when instruction traffic is significant, while limiting the impact of speculative prefetched lines on cache contents.
