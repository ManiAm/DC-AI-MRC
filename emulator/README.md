# SRv6 MRC Emulator

SRv6 fabric emulator and per-EV (Entropy Value) MRC behavior layer for AI-cluster networking research.

Originally sourced from the [srv6-mrc-emulator](https://github.com/segmentrouting/srv6-mrc-emulator) project by [Bruce McDougall](https://www.linkedin.com/in/bruce-mcdougall/). He built the emulator independently to give engineers a hands-on way to explore MRC's multiplane packet-spraying architecture using containerized SONiC switches.

Licensed under the [Apache License 2.0](LICENSE).

> **Prerequisites:** This README covers the emulator's internals — its code layout and how its MRC control plane is implemented. For foundational concepts (what MRC is, how EVs and SRv6 uSID routing work, and how to install and run the emulator), see the [main README](../README.md). For the full MRC protocol deep dive, see the [MRC Primer](../docs/MRC_Primer.md).

## Credits

Bruce McDougall is a Principal Architect at Cisco Systems specializing in Segment Routing and SRv6. He has presented SRv6 deployment use-cases at conferences such as NANOG and Cisco Live, and holds a patent on Segment Routing in NFV networks.

## Emulator Layout

```
emulator/
├── srv6_mrc/           Python package: topology constants, runtime libs
│   ├── topo.py            fabric dimensions + addressing helpers (reads topo.yaml)
│   ├── topology.py        typed Topology accessor
│   ├── runner.py           spray sender/receiver core
│   ├── encap.py            SRv6 outer-packet builder (raw-socket, no SRH)
│   ├── policy.py           per-plane / per-EV scheduling policies
│   │                       (round_robin, hash5tuple, weighted, ev_spray, health_aware_mrc)
│   ├── reorder.py          reorder-distance histogram + FlowStats schema
│   ├── netem.py            tc netem helpers (run via nsenter)
│   ├── report.py           JSON + ascii summary writer
│   ├── cli/
│   │   ├── spray.py        userspace SRv6 packet generator        (CLI: spray)
│   │   ├── routes.py       static SRv6 route management           (CLI: routes)
│   │   └── srctl.py        kubectl-shaped lab CLI                  (CLI: srctl)
│   └── mrc/
│       ├── run.py           scenario orchestrator                   (CLI: run-scenario)
│       ├── daemon.py        per-host MRC daemon
│       ├── scenario.py      scenario YAML schema + executor
│       ├── agent.py         SenderMrcAgent + ReceiverMrcAgent
│       ├── transport.py     MrcTransport ABC + Srv6RawTransport
│       ├── ev_state.py      per-(tenant, plane, path) EV state machine
│       ├── probe.py         PROBE / PROBE_REPLY / LOSS_REPORT wire format
│       ├── probe_clock.py   per-EV in-flight probe bookkeeping
│       ├── loss_window.py   per-EV receiver-side loss accounting
│       └── loss_compute.py  per-EV SentWindowRing on the sender
├── generators/
│   └── fabric.py           parameterized generator: reads topo.yaml,
│                            writes topology.clab.yaml + config/
├── topologies/
│   ├── 4p-4x8/             default topology (48 switches + 16 hosts)
│   ├── 4p-8x16/            larger topology  (96 switches + 32 hosts)
│   └── 2p-4x8/             smallest topology (24 switches + 16 hosts)
│       ├── topo.yaml           single source of truth for this variant
│       ├── topology.clab.yaml  containerlab topology (generated)
│       ├── config/             per-node SONiC + FRR configs (generated)
│       ├── scenarios/          MRC traffic scenario YAMLs
│       └── routes/             route-spec YAMLs for `routes apply`
├── host-image/
│   └── Dockerfile          alpine + scapy + pip-installed srv6_mrc
├── scripts/
│   └── config.sh           push config_db.json + frr.conf into containers
├── tests/                  unittest mirror of srv6_mrc/ layout
├── docs/                   design + runbook documentation
└── results/                scenario JSON output (gitignored)
```

---

## MRC Architecture

This section explains how the emulator's MRC control plane works under the hood — how it detects path failures and shifts traffic in real time.

### Stateless Probes

MRC monitors every path through the fabric by sending lightweight **probe packets** along each [EV](../README.md#entropy-values-evs). The emulator uses a **stateless** probe design: each probe is a [uSID](../README.md#srv6-usid-and-endpoint-actions)-encapsulated packet with a 6-slot micro-SID list that routes the probe out through the fabric to the peer host, then loops it back to the sender — all through the kernel's IPv6 forwarding plane, with no userland processing on the peer.

The round trip for a probe on EV `P0:S0` from `host00` toward `host01`:

```text
host00 → leaf00 → spine00 → leaf01 → host01 (kernel forwards, no userland)
       → leaf01 → spine00 → leaf00 (End.DT6 decap) → host00
```

The sender's MRC daemon receives the returning probe and records a success for that EV. If probes stop coming back, the EV is unhealthy.

**Key design points:**

- The peer host's kernel forwards the probe back without any application involvement — no timing-sensitive correlation, no match tables.
- Each EV has its own /128 address on the sender (e.g., `cccc::100` for plane 0 / spine 0), so the daemon knows which EV a returning probe belongs to by looking at the inner destination address.
- Probes are sent every 200 ms per EV by default.

### EV Health Model

Each EV's health is tracked with a **sliding-window** model:

1. **Window**: over the last 5 probe intervals (1 second by default), count how many probes were sent vs. received for each EV.
2. **Demotion**: if `received / sent` drops below 50% (the fail threshold), the EV is marked `ASSUMED_BAD` and its spray weight drops to 0 — traffic shifts to the remaining healthy EVs.
3. **Recovery**: once a demoted EV's ratio rises above 90% for 5 consecutive windows, it returns to `GOOD` and regains its spray weight.

This two-signal approach (probes for path liveness, loss reports for data-path quality) gives MRC fast fault detection without false positives. Probes are described above; loss reports — per-EV packet-loss feedback sent from the receiver back to the sender — are covered in the [MRC Daemon](#mrc-daemon-process-model) section below.

### MRC Daemon Process Model

The emulator runs **one MRC daemon per source host**, separate from the data-sending processes. This separation avoids GIL (Global Interpreter Lock) contention in Python — the GIL prevents multiple threads from executing Python code simultaneously, so the daemon runs as a separate process. The daemon handles probes and health tracking, while sender processes focus on high-throughput packet generation.

```text
src_host (e.g., yellow-host00)
├── MRC Daemon (one per host)
│   ├── Probe emitter — sends one probe per EV every 200 ms
│   ├── Dispatcher — receives returning probes + loss reports from peers
│   ├── EV health table — per-EV sliding-window state machine
│   └── Snapshot publisher — writes health to /dev/shm every 200 ms
│
├── Data sender (dst_id=1) — reads health snapshot, picks EVs by weight
├── Data sender (dst_id=2)
└── ...

dst_host (e.g., yellow-host01)
└── Receiver
    ├── Data RX — collects per-flow stats
    └── Loss reporter — sends per-EV loss reports back to the sender
```

**How senders consume health state:** The daemon writes a JSON snapshot to `/dev/shm/srv6-mrc/` every 200 ms containing each EV's state and weight. Data sender processes read this file and build a weighted distribution — healthy EVs get equal weight, demoted EVs get zero. This file-based approach is simple, debuggable (`cat` + `jq` to inspect), and introduces at most 200 ms of staleness — well within MRC's detection thresholds.

**Lifecycle:** The scenario orchestrator (`run-scenario`) starts daemons before senders and stops senders before daemons. Cleanup uses `docker exec pkill` to reach inside containers, since a local `Popen.terminate()` cannot cross the `docker exec` boundary.
