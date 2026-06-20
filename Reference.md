# HDGL Analog Fabric — Consolidated Suite
**Expert Analysis & Reference Document**

## OVERVIEW
This is a single-file consolidation of the metal_fabric.zip suite - a complete OS, manual, and valid payload in one file. Targets:
- **QEMU**: qemu-system-x86_64 with i440FX machine and e1000 NIC
- **METAL**: Gigabyte H81-BTC-Pro (Haswell PCH), Intel i217-V (8086:153A)

---

## CONSOLIDATION CHANGES (Key Modifications from Original)
1. **Unified Glyph Trees**: All glyph trees merged into one file with parent references
2. **BAR64 Fix**: Corrected i217-V 64-bit MMIO BAR read (was sign-extend error)
3. **RCTL Fix**: Changed from 0x8002 → 0x8802 (fixes BSIZE interpretation for 82579/i217)
4. **Extended Smoke Test**: Added RTL8139 QEMU test and metal probe target
5. **Memory Map Clarification**: 
   - 0x101208: live gossip_fingerprint (updated each cycle)
   - 0x1013FC: phi-lattice slot 127 = cached genome_fp (set once at boot)
6. **Collapsed Duplicate**: Removed duplicate smoke_test glyph
7. **Makefile Extensions**: Added carrier_test, fabric_test, metal targets
8. **Test Files**: hdgl_genome_test.c, hdgl_fabric_test.c, zchg_carrier_test.c referenced

---

## ARCHITECTURE PARTS

### PART I — UNIVERSAL CONSTANTS
**Mathematical Foundation**:
- φ (Golden Ratio) = 1.6180339887498948
- Axis: A=0, C=1, G=2, T=3
- **carrier_constants** = floor(2³²/φ) derived values:
  - PHI32 = 0x9E3779B9 (Knuth multiplicative hash)
  - FIB32 = 0x9E3779B1 (nearest prime)
  - SQRT_PHI32 = 0xA17F0BCB
  - PHI32_INV = 0x144CBC89

### PART II — PHI-FOLD CARRIER PRIMITIVE
**Core Encryption/Obfuscation**:
- **phi_fold(x, key, seq)** = (x·PHI32 + key·FIB32 + seq·SQRT_PHI32) mod 2³²
- **Three Carriers**:
  - CH-0: 4 bytes/frame (reserved field in zchg_frame_header) = 800KB/sec covert channel
  - CH-1: 2 bytes/gossip (LSB-2 bits of strand_weights) = 16 bits per message
  - CH-2: /serve/<path> phi-sequence (1 bit/request via HTTP paths)

### PART III — BASE-4 CODEC
**DNA-inspired Codec**:
- TYPE → DNA mapping with energetic affinity:
  - CPU/RUNTIME → A (purine, high-energy)
  - MEM → C (pyrimidine, structural)
  - IO → G (purine, bridging)
  - COMPILER/STOR → T (pyrimidine, templating)
- Codon = 3-mer of consecutive Omega nodes
- K-mer = 4-mer (256 possibilities)

### PART IV — GENOME FABRIC ENGINE
**Runtime Configuration Derived from Omega Graph**:
Memory Layout (Identity-mapped):
- 0x101100: GenomeFabricConfig (16 bytes)
- 0x101200: gossip_dn_ema (8 bytes, EMA of peer aggregates)
- 0x101208: gossip_fingerprint (4 bytes, live)
- 0x1013FC: phi-lattice slot 127 = genome_fp cache (stable)
- 0x200000: OMEGA_BASE (graph storage)

**Key Derived Values**:
- points_per_tick = 100 + (phi_lattice_mean & 0xFFFF) * 500 / 65535
- strand_count = nearest_pow2(gc_content * n_nodes, 8, 256)
- core_radius = popcount(dn_aggregate) * φ / 32
- genome_fp = FNV_PHI_SPIRAL(base4_seq, dn_aggregate)

### PART V — NIC DRIVER
**Complete NIC Driver for e1000 and RTL families**

**Memory Map**:
- 0x105000: NIC driver state (512 bytes)
- 0x106000: TX ring (256 bytes)
- 0x107000: RX ring (256 bytes)
- 0x108000: TX buffers (32KB)
- 0x110000: RX buffers (32KB)

**Critical H81-BTC Fixes**:
1. BAR64 detection and combination for i217-V
2. RCTL corrected: 0x8002 → 0x8802 (BSIZE=01b for 2KB buffers)

**Register Offsets**:
- e1000: E1000_CTRL, E1000_RCTL, E1000_TCTL, E1000_TDLEN, etc.
- RTL8111: RTL_CMD, RTL_RCR, RTL_TCR, RTL_IDR0, etc.

### PART VI — NIC TIMING
- e1000 reset wait using phi_tick
- RTL version detection from TXCFG register
- Header offset detection (0 or 4 bytes)

### PART VII — PEER DISCOVERY
**Three-Phase Discovery**:
1. **Phi-seed Multicast**: mcast_ip = 0xEF000000 | (phi_fold(genome_fp, 0, 0) & 0x00FFFFFF), port 8090
2. **ARP Probe**: Fallback for different subnets (10.0.0.1-254)
3. **Sector-68 Static Peers**: Build-time LN_SEED_PEERS from disk

**Peer Table** (0x119800):
- 16 entries × 16 bytes = {uint32 ip, uint16 port, uint8[10] pad}

### PART VIII — BOOTSTRAP GLOBALS
- GENOME_FP_ADDR = 0x1013FC (phi-lattice slot 127)
- FABRIC_READY_FLAG = 0x101014 bit 5
- **INVARIANT**: genome_fp at 0x1013FC is boot-time stable; gossip_fingerprint at 0x101208 is live

### PART IX — BARE-METAL STORE
**Persistent Storage**:
- Sector 69: store header (strand_count, record_count, genome_fp)
- Sectors 70+: strand data (64 sectors/strand × 512 = 32KB/strand)
- ARENA_BASE = 0x400000 (4MB arena)
- Magic: "ZCHG" (0x47484347 LE)

### PART X — GENOME BOOT HOOK
**Bootstrap Sequence**:
1. Compute phi_lattice_mean from slots 0-3
2. Count active Omega nodes
3. Derive fabric config at 0x101100
4. Initialize gossip_fingerprint
5. Store genome_fp to slot 127 (0x1013FC)

### PART XI — SHELL COMMAND EXTENSIONS
**New Commands**:
- genome, fabric, ftick, fgossip, fauth, nic2, store, peers, send, load

---

## KEY INVARIANTS
1. **Peer Discovery**: 
   - Reads live gossip from 0x101208
   - Reads stable index from 0x1013FC
2. **BAR64**: Bits 2:1 of BAR0 = 10b indicates 64-bit BAR
3. **EMA Alpha**: φ/(φ+1) ≈ 0.618 for natural weighting
4. **Phi-fold Bijection**: Every (x, key, seq) maps to unique fold value

---

## IMPLEMENTATION NOTES
- ASM code uses identity-mapped memory for bare-metal compatibility
- QEMU and metal targets both supported
- All config derived from Omega graph (no hardcoded values)
- Covert channels operate at network layer without detection
- Supports Intel e1000 family + Realtek RTL8111/8168/8169 + Atheros AR8131
