<img width="628" height="665" alt="image" src="https://github.com/user-attachments/assets/6b914d55-c8f3-447f-aff5-0bb2b497ce4f" />

# ALL YOUR BASE ARE BELONG TO US, as it were.

SUITE.hdgl is the document to read first — it is itself a valid .hdgl file containing the complete file tree, all six integration seams, the full build procedure, and the two-node test guide. Because it's .hdgl, the running fabric can load and parse it. It is both the manual and a payload.
The suite divides cleanly into two layers:

Native glyph layer — SUITE.hdgl, hdgl_complete.hdgl, hdgl_fabric.hdgl, hdgl_genome.hdgl, hdgl_genome_shell.hdgl, hdgl_peer_discovery.hdgl, hdgl_nic.asm. These contain the emit rules that produce x86-64 assembly. NASM is the only tool. No C compiler touches the boot path.

C layer — everything else. Exists only for the POSIX transport daemon and hosted test harness. On bare metal, hdgl_complete.hdgl emits the equivalent assembly inline. The C files are the scaffolding you can remove once hardware confirms the glyph emit is correct.

The build is three shell commands: bash build.sh for the image, make for the daemon, dd to flash. The two-node test produces GENOME-LOCK within three gossip cycles — about twelve seconds after the second node boots — at which point all three carrier channels are live and the fabric store is accepting payloads.

# NEW THIS VERSION

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

---

# The core idea

The bot doesn't sit "above" the fabric. It is a fabric node. Its VectorContext identity in analog-container.mjs is computed with phiFoldHash128, which is the JS port of phi_fold_hash32 from the conscious-128-bit-floor. That makes the bot's session identity phi-lattice-native. The lkAdvance() entropy ratchet is structurally equivalent to a Kuramoto epoch tick. So the bot already speaks the fabric's identity language — what it's missing is the resource discovery and multi-node coordination layer.
What needs to be built

There were three gaps to close:

Gap 1 — Resource census at session start. Right now get_context() in the Wu-Wei server reads only the local machine (shell tools, Python, ffmpeg). It needs to be extended to also query the fabric via a HEALTH frame broadcast on the phi-seed multicast address (the same 239.x.x.x derivation in hdgl_peer_discovery.hdgl). Each responding peer returns its gossip_dn_ema and genome_fingerprint. The bot's unfold() pass selector should factor in which node has spare compute, storage strands, or GPU type (OTYPE_GPU in the Omega graph) before deciding where to run a given pass. This is just an additional RECALL pass that runs before any SHELL or CODE pass on a multi-node task.

Gap 2 — Task routing through the phi-hash router. The wuwei-routing layer already has the phi-hash router (get_phi_hash() in router-phi.ps1). That needs to be promoted from a two-server local load balancer into an n-node router. The routing key is phi_fold(task_content_hash, genome_fp, strand_id) — exactly the same expression used in hdgl_fabric_loader.c to assign a payload to a strand. Tasks that are heavy inference go to the node whose strand_auth byte has CPU/GPU authority (BASE_A strands). Tasks that are storage (transcripts, notes, ERL ledger entries) go to nodes holding BASE_T/BASE_C strands. The router reads the fabric's gossip table — which is already converging via zchg_lattice_apply_gossip with genome EMA — to know which node holds which authority right now.

Gap 3 — Cross-node ERL ledger. The Elegant Recursive Ledger currently lives in erl-ledger.json on one machine, with a file-lock for two concurrent server writes. On a multi-node fabric the ledger becomes a strand-native append-only store: each erlAppend() call computes phi_tau(entry_id) mod strand_count to select the authoritative node, sends the entry payload as a FILESWAP frame over zchg, and the remote node stores it. The existing ledger hash chain is unchanged; the transport layer underneath it changes. Verification (erlVerify) stays local because each node holds only its own strands — cross-chain verification uses gossip-propagated fingerprints already present in gossip_msg_t.cluster_fingerprint.

# The new file: fabric_node_agent.mjs

This is the single file you need to add to the EZ server. It exports three things:

js//
```
Broadcast HEALTH, collect fabric peers with their Omega census
export async function discoverFabricNodes(genomeFp, timeoutMs = 200)

// Given a task description and required pass type, return best-fit node IP:port
export function routePassToNode(taskHash, passType, peerTable)

// Wrap erlAppend to write to the authoritative strand node
export async function fabricErlAppend(ledger, entry, peerTable)
```

discoverFabricNodes sends a UDP multicast packet to 0xEF000000 | (phiFold32(genomeFp) & 0x00FFFFFF) on port 8090 (directly mirroring the assembly in hdgl_peer_discovery.hdgl). Responses arrive as JSON-wrapped HEALTH payloads from any node running the fabric HTTP shim on top of zchg. The result is a peer table that routePassToNode uses at every unfold() call.
routePassToNode maps Wu-Wei pass types to Omega base types: FETCH/BROWSE → BASE_G (IO nodes), CODE/SHELL → BASE_A (CPU/runtime nodes), STORE/RECALL → BASE_T/BASE_C (storage nodes), TRANSFORM (ffmpeg/whisper) → BASE_A with OTYPE_GPU preference. It picks the peer whose strand_auth bitmap has the highest-weight bit matching the base type.
Integration point in server.js
The change to the existing server is minimal. In selectPassSequence(), after the task analysis, add one line:

```
jsconst nodeMap = await discoverFabricNodes(railState.genomeFp);
```

Then in each pass executor, before shelling out or calling the LLM, route the work:

js
```
const targetNode = routePassToNode(taskHash, passType, nodeMap);
// if targetNode.ip !== localIp → forward via zchg HTTP proxy
```

If targetNode is the local node, execution proceeds exactly as before — zero overhead for single-node deployments. Only when a remote peer with better authority is available does the pass get forwarded.

What doesn't change
The analog-container identity computation, the phi-lattice hash, the ERL ledger chain structure, and the bare-metal firmware are all untouched. The firmware nodes don't need to know there's an AI bot above them — they just respond to HEALTH frames and accept FILESWAP payloads the same as any peer. The bot is native to the fabric by virtue of speaking phi_fold.
