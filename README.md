ALL YOUR BASE ARE BELONG TO US, as it were.

SUITE.hdgl is the document to read first — it is itself a valid .hdgl file containing the complete file tree, all six integration seams, the full build procedure, and the two-node test guide. Because it's .hdgl, the running fabric can load and parse it. It is both the manual and a payload.
The suite divides cleanly into two layers:

Native glyph layer — SUITE.hdgl, hdgl_complete.hdgl, hdgl_fabric.hdgl, hdgl_genome.hdgl, hdgl_genome_shell.hdgl, hdgl_peer_discovery.hdgl, hdgl_nic.asm. These contain the emit rules that produce x86-64 assembly. NASM is the only tool. No C compiler touches the boot path.

C layer — everything else. Exists only for the POSIX transport daemon and hosted test harness. On bare metal, hdgl_complete.hdgl emits the equivalent assembly inline. The C files are the scaffolding you can remove once hardware confirms the glyph emit is correct.

The build is three shell commands: bash build.sh for the image, make for the daemon, dd to flash. The two-node test produces GENOME-LOCK within three gossip cycles — about twelve seconds after the second node boots — at which point all three carrier channels are live and the fabric store is accepting payloads.

NEW THIS VERSION

What the file contains
1462 lines, zero third-party dependencies. Everything either re-implements a primitive from your C layer or delegates to Node builtins.

Primitives (phiFold32, phiUnfold32, phiTauStrand, contentHash64, fabricIdentity64, buildGenomeFp) are exact JS ports of zc_fold/zc_unfold from zchg_carrier.h and the FNV-phi spiral from hdgl_fabric_loader.c. Same constants, same accumulator pattern, verified round-trip.

Discovery (discoverFabricNodes) runs three phases in sequence: UDP multicast to 239.x.x.x where x.x.x derives from phi_fold(genome_fp, 0, 0) & 0x00FFFFFF (mirrors hdgl_peer_discovery.hdgl Phase 1 exactly), static peers from FABRIC_PEERS env (mirrors sector-68), then HTTP /health poll for any stubs that responded to multicast but didn't send a full JSON gossip payload.

Routing (routePassToNode) maps the Wu-Wei pass type to an Omega base type (A/C/G/T), scores every peer by the sum of strand weights on strands where strandIndex % 4 === base, adds a GPU bonus for TRANSFORM passes, and uses a phi-hash of the task content as tiebreaker — same distribution property as coord-proxy.js.

Remote execution (fabricForwardPass) POSTs a JSON-RPC tools/call to the remote node's MCP endpoint. The remote node needs no modification — this is the same protocol @modelcontextprotocol/sdk already speaks.

ERL routing (fabricErlAppend) writes locally first (chain integrity guaranteed), computes phi_tau(identity) mod 8 to find the authoritative strand, then ships a FILESWAP payload to the remote node's /fabric/store endpoint. Non-fatal if the remote is unreachable — local copy is the fallback.

Integration into server.js is three lines:

```
jsimport { attachFabricHandlers, patchSelectPassSequence,
         patchRunUnfold } from './fabric_node_agent.mjs';

// After httpServer is created:
attachFabricHandlers(httpServer);

// Replace the two functions:
const selectPassSequence = patchSelectPassSequence(originalSelectPassSequence);
```

attachFabricHandlers mounts /health, /gossip, /fabric/store, /fabric/query, /fabric/peers, /fabric/status and starts the 30-second gossip tick. The existing handler tree is untouched — fabric endpoints intercept before it. On a single-node deployment the routing always falls back to local with zero overhead.
