ALL YOUR BASE ARE BELONG TO US, as it were.

SUITE.hdgl is the document to read first — it is itself a valid .hdgl file containing the complete file tree, all six integration seams, the full build procedure, and the two-node test guide. Because it's .hdgl, the running fabric can load and parse it. It is both the manual and a payload.
The suite divides cleanly into two layers:

Native glyph layer — SUITE.hdgl, hdgl_complete.hdgl, hdgl_fabric.hdgl, hdgl_genome.hdgl, hdgl_genome_shell.hdgl, hdgl_peer_discovery.hdgl, hdgl_nic.asm. These contain the emit rules that produce x86-64 assembly. NASM is the only tool. No C compiler touches the boot path.

C layer — everything else. Exists only for the POSIX transport daemon and hosted test harness. On bare metal, hdgl_complete.hdgl emits the equivalent assembly inline. The C files are the scaffolding you can remove once hardware confirms the glyph emit is correct.

The build is three shell commands: bash build.sh for the image, make for the daemon, dd to flash. The two-node test produces GENOME-LOCK within three gossip cycles — about twelve seconds after the second node boots — at which point all three carrier channels are live and the fabric store is accepting payloads.
