# ZIT - Zetabyte Information Tracker
**Version:** 0.3.0  
**Purpose:** Line-granular version control and package management for xFlowOS  
**License:** GPL-3.0  
**Author:** Shaun Lloyd (0@xflowos.au)

---
<img width="600" alt="Melt example" src="https://github.com/xflowos/zit/blob/main/zit-demo.gif">

---
## Standing on the Shoulders of Giants
ZIT is built on battle-tested tools. We don't reinvent wheels.

### [OpenZFS](https://openzfs.org) - Storage Foundation
**What it is:** Production-grade filesystem with Copy-on-Write, snapshots, compression, and deduplication.  
**Why ZIT uses it:**
- **Snapshots** – Every commit/package version is a ZFS snapshot (instant, zero-copy)
- **Clones** – Writable branches without duplicating data (Copy-on-Write)
- **Rollback** – Instant recovery from bad state  

**Without ZFS:** Core magic is lost for now. Portable backends (IPFS, S3, etc.) planned as alternative providers.


---
### rsync and the MAVLink Ecosystem – Giants of Sync and Messaging
Key tools we interoperate with:

rsync – Legendary delta-transfer algorithm and tool
MAVLink Protocol – Lightweight hybrid pub/sub + point-to-point messaging standard
MAVProxy & pymavlink – Battle-tested ground station, proxy, and Python tools

Giants behind them:

Andrew "Tridge" Tridgell (OAM) – Creator/maintainer of rsync (with Paul Mackerras), Samba; creator and lead maintainer of MAVProxy; original author of pymavlink; ArduPilot lead developer (fixed-wing/VTOL)
Lorenz Meier – Creator of the MAVLink protocol (2009), PX4, Pixhawk, QGroundControl

Why ZIT stands directly on these shoulders:

zit-sync (planned v0.5.0) uses full rsync compatibility for bandwidth-efficient remote deltas
Direct interop with MAVProxy/pymavlink to leverage MAVLink’s proven hybrid pub/sub model for lightweight metadata broadcasts, package announcements, and P2P coordination over Tailscale meshes
These tools existed long before xFlowOS. MAVLink, in particular, worked flawlessly for me in real-world, high-stakes environments where reliability was everything. There may be theoretically “better” protocols on paper, but none I trust more. Trust is the foundation.

Huge respect to Andrew "Tridge" Tridgell – true Aussie GNU/CCC-style hacker legend – and Lorenz Meier for building these foundational, interoperable tools that have powered critical systems for decades. ZIT and zit-sync wouldn't be possible without them. 🇦🇺🇨🇭🤘


### Tailscale - The SDN Magic for P2P Distribution
What it is: Zero-config, secure mesh VPN built on WireGuard – making any devices talk directly as if on the same LAN, anywhere.
Founders/Hackers: Avery Pennarun (CEO), David Crawshaw, David Carney – true networking wizards turning NAT hell into seamless connectivity.
Why ZIT (and xFlowOS) loves it:

zit-sync (v0.5.0+) uses Tailscale for P2P distribution, team package caching, and secure remote sync – nodes discover and connect directly, no port forwarding nightmares.
ZIT is the datastore sitting directly on Tailscale meshes, enabling distributed xFlowOS fleets to feel like one big, intelligent local network.
Critically: Their huge free tier (3 users, 100 devices – perfect for personal/homelab/open-source projects) makes this accessible to all hackers without barriers.

Echoing Sun Microsystems legend Scott G. McNealy's iconic line: "The network is the computer."
Tailscale makes that vision not just possible, but real, easy, and usable today – turning your entire distributed fleet into one cohesive, intelligent system.
Massive props to the Tailscale team for building this game-changer and keeping a generous free plan alive. xFlowOS wouldn't be the same without standing on these shoulders. 🤘
(Note: For fully self-hosted control server, the excellent community OSS Headscale is compatible and recommended for larger/privacy-focused setups.)

"In xFlowOS, intelligence isn't centralized — it's distributed.
ZIT stores the data. Tailscale connects the nodes.
The network itself thinks, lives, and knows.
But ultimately — you are the intelligence.
We just connect your devices and let the fleet become an extension of your mind."

"xFlowOS doesn't centralize intelligence — it distributes it.
Every node thinks. The whole fleet knows."

"xFlowOS turns your devices into a single distributed brain.
The network doesn't just carry data — it thinks with it."

---

### [Charm.sh](https://charm.sh) - Beautiful Terminal UIs
**What it is:** A suite of Go libraries that make building delightful, modern terminal applications feel effortless – Bubble Tea, Lip Gloss, Huh, Glamour, Log, and more.  

**Key Innovators:**
- **Chris Waldon** – Creator and lead maintainer of Bubble Tea (Elm-inspired TUI framework for Go), Lip Gloss, and the core Charm vision
- **Maas Lalani** – Creator of Huh (beautiful, powerful forms/prompts), major contributor, and tutorial wizard
- **bashbunni** – The charismatic YouTube/Twitch face of Charm TV – hosting epic demos, tutorials, and making the terminal glamorous for everyone
- **Mike Harris** – Major contributor across the ecosystem
- The entire Charm team and open-source contributors

**Why ZIT stands directly on these shoulders (just like we do with MAVLink tools):**
- **Bubble Tea** – Elm Architecture port to Go; enables reactive, state-managed TUIs (progress bars, spinners, selection lists, live diffs)
- **Lip Gloss** – Declarative styling for colors, borders, layouts
- **Huh** – Intuitive, accessible forms and interactive prompts (game-changer for user input)
- **Log** – Structured, pretty logging

Without Charm, ZIT’s interface would look like it’s from 1995 and be 10× harder to build.  

Massive LOVE and respect to Chris Waldon for the foundational wizardry, Maas Lalani for Huh and killer tutorials, **bashbunni** for being the infectious Charm TV host that gets everyone hyped on glamorous CLIs (you're crushing it, legend ❤️ you made everything make sense for me peace & love), Mike Harris, and the whole Charm crew for bringing Elm-inspired elegance and pure joy to the terminal. True hacker artistry. ❤️

**Without Charm:** ZIT would look like it’s from 1995 and be 100× harder to build.

---

## What is ZIT?
**ZIT is the datastore provider for xFlowOS.**  
It manages:
- **Source code** – Project repositories (like Git, but line-granular)
- **Files**, **blobs**, **blocks** (images, ISOs, etc.)

Using:
- **ZFS snapshots** – Instant, zero-copy versioning
- **Line-granular content addressing** – Maximum deduplication
- **DataStore indexing** – Fast queries and metadata
- **Beautiful TUI** – Charm.sh for a delightful experience

---
## Current Status
**⚠️ Work in Progress – v0.3.0**  
ZIT currently implements:
- ✅ Basic VCS operations (init, add, status, diff, branch, ls)
- ✅ Line-granular content addressing
- ✅ SQLite storage backend
- ✅ Beautiful Charm.sh TUI
- ✅ Multi-platform builds (Linux, macOS, Windows)

**Not yet implemented:**
- ❌ ZFS snapshot integration (planned v0.4.0)
- ❌ Commit/tag support (planned v0.4.0)
- ❌ Remote sync (planned v0.5.0)
- ❌ Merge operations (planned v0.6.0)

Installation and usage instructions describe the planned architecture. PRs very welcome :)

---
## Roadmap
### v0.3.0 - Core VCS Features
- [x] Line-granular tracking
- [x] ZFS snapshot integration
- [x] Beautiful TUI (Charm.sh)
- [ ] Commit messages and metadata
- [ ] Branch merging (line-level)
- [ ] Tag support

### v0.4.0 - Package Management
- [ ] Automatic ZFS snapshots on install
- [ ] Package dedup statistics
- [ ] Version switching via clones

### v0.5.0 - Remote Sync
- [ ] ZFS send/receive over network
- [ ] TailScale P2P distribution
- [ ] Team package caching

### v0.6.0 - Advanced Features
- [ ] Line-level merge tool
- [ ] Interactive rebase
- [ ] Conflict resolution UI
- [ ] Git interop (import/export)

### v1.0.0 - Production Ready
- [ ] Full stability guarantees
- [ ] Complete documentation
- [ ] Performance benchmarks
- [ ] Migration tooling

---
## Credits
**Standing on the shoulders of:**
1. **[OpenZFS](https://openzfs.org)** – The foundation (snapshots, clones, dedup, compression)
2. **Andrew "Tridge" Tridgell** & **Lorenz Meier** – rsync, MAVProxy, pymavlink, MAVLink protocol
3. **Tailscale Team** (Avery Pennarun, David Crawshaw, David Carney) – Secure mesh networking magic
4. **[Charm](https://charm.sh)** – Beautiful terminal UIs (Bubble Tea, Lip Gloss, Huh, Log)
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) – Fast, secure hashing

**ZIT exists because these tools are excellent.**

---
## Author
**Shaun Lloyd**  
Email: 0@xflowos.au  
Web: https://xflowos.au  
Part of the **xFlowOS ecosystem**.

---
## Next Steps
**Want to contribute?**  
I'm just one hacker (actually a chef, not even a pro dev)! Priorities:
- [ ] Peer review
- [ ] Failure
- [ ] More Peer review
- [ ] Benchmarks
- [ ] Test suite
- [ ] Real-world production failure → recovery → stability testing  

(This is a single-hacker WIP. Storing data in xFlowOS is no joke – real incidents, metrics, and battle-testing required.)

**Want to learn more about xFlowOS?**  
https://xflowos.au

---
*Zetabyte Information Tracker – Because ZFS makes everything better.*
