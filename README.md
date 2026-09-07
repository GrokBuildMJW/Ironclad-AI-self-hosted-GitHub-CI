# Self-hosted GitHub CI for Ironclad AI — a beginner how-to

An **example lab**: a handful of mini-PCs run GitHub Actions for a private product repo, so you do not buy GitHub-hosted minutes. The pictures below are the real split we use around [Ironclad AI](https://github.com/GrokBuildMJW/ironclad-ai). Copy the *idea*, not the hostnames, if your LAN looks different.

No IPs, no SSH keys, no registration tokens live in this tree. SSH aliases live in `~/.ssh/config`. That is the point.

![Home lab of small machines on a desk, ethernet running up to a git cloud](docs/images/hero-lab.jpg)

## Why this exists

GitHub Actions on **private** repos burns plan minutes when the job runs on `ubuntu-latest`, `windows-latest`, or `macos-latest`. macOS is billed at ten times Linux. A product with Linux tests, a Windows installer, and a Mac client will empty a Team plan in a busy week.

**Self-hosted runners do not consume those minutes.** You pay electricity and the machines you already own. GitHub is still the forge (git, PRs, checks). The compute stays on your desk.

![Invoices and a credit card on the left, a stack of mini PCs on the right](docs/images/cost-hosted-vs-selfhosted.jpg)

You still pay GitHub for the usual account (and for artifact storage if you keep huge logs). You do **not** pay per pytest.

## The one-sentence architecture

**Write on Linux. Watch it from Windows. Check it on three dedicated runners. Talk to models on two GPU boxes that never run CI.**

![Map of GitHub, three runners, and three non-runner boxes](docs/images/01-lab-map.svg)

| Box (example name) | What it is | GitHub Actions? |
|---|---|---|
| **devlin** | Linux Tiny, Ubuntu Desktop, Cursor + CLIs | **No. Never.** |
| Windows operator PC | Remote Desktop client only | No |
| **arlin** | Linux mini-PC, native Actions listener | Yes — Linux jobs |
| **arwin** | Windows desktop, Windows service | Yes — Windows jobs |
| **macmini** | Apple Silicon Mini, LaunchDaemon | Yes — macOS jobs |
| **ai4090** | RTX 4090, coder LLM on `:8090` | No (CI used to live here; it moved) |
| **spark** | DGX Spark, orchestrator LLM on `:8000` | No |

Related serving pins, not this repo: [Flash-Next on Spark](https://github.com/GrokBuildMJW/Qwen3.8-Flash-Next-NVFP4-SGLang-DGX-Spark), [27B vLLM rollback](https://github.com/GrokBuildMJW/Qwen3.8-27B-NVFP4-vLLM-DGX-Spark), [Qwen3-Coder on 4090](https://github.com/GrokBuildMJW/Qwen3-Coder-30B-A3B-Q4_K_M-llama.cpp-RTX-4090).

## Extreme split: RDP from Windows onto the Linux dev box

This is the part people skip, and the part that keeps you sane.

The Windows machine is **not** where Ironclad lives. It is a keyboard, a screen, and `mstsc`. All writing happens on **devlin** (ThinkCentre Tiny, Ubuntu Desktop, GNOME Remote Login, TCP **3389**, LAN only).

![Windows operator PC RDPs to Linux devlin](docs/images/04-rdp-split.svg)

### Do this once on Windows

1. Put `devlin` in `~/.ssh/config` (hostname + key). SSH is for scripts and `git` if you ever need it; daily work is RDP.
2. Open **Remote Desktop Connection** (`mstsc`).
3. Computer: `devlin` (or whatever the alias resolves to on your LAN). Port 3389 is the default.
4. First prompt is the **device** user (the account Ubuntu Remote Login uses). Then GDM asks for the Linux account.
5. If the session is a black screen, turn NLA off on that `.rdp` file and try again. Do not install VNC. xrdp is fallback only.

Cursor runs **on Linux**, inside that RDP session. You are looking at Linux through Windows. A Windows update cannot trash the toolchain. A failed CI job cannot lock the files you are typing.

**Do not** install a GitHub Actions listener on `devlin`. If you do, you have put the editor and the judge on the same disk.

## What happens when you push

![Laptop to switch to machines, then GitHub dispatches to three runners](docs/images/push-to-lab.jpg)

![Four steps: RDP, git push, GitHub, three runners](docs/images/02-git-push-flow.svg)

1. You are already inside **devlin** via RDP.
2. `git push` to `GrokBuildMJW/ironclad-ai`.
3. A workflow starts. It must **not** say `runs-on: ubuntu-latest`.
4. Idle listeners on **arlin** / **arwin** / **macmini** pick the jobs that match their labels.
5. **spark** and **ai4090** do not appear in this story. They serve models.

## How a job finds a box

![Three columns for Linux, Windows, macOS label matching](docs/images/03-job-routing.svg)

```yaml
# Linux tests and docker-test
runs-on: [self-hosted, Linux, arlin]

# Windows installer / client
runs-on: [self-hosted, arwin]

# macOS client
runs-on: [self-hosted, macOS, macmini]
```

`arlin` also still wears the old `ai4090` label so historical workflows keep landing on the mini-PC, not on the GPU box.

Never two listeners with the same runner id (`TaskAgentSessionConflict`). One native process per box.

## How to set this up (the short version)

### 1. Buy (or reuse) three cheap computers

You do not need a cluster. A used Ryzen mini-PC, a spare Windows desktop, and a Mac mini cover Linux, Windows, and macOS. Keep GPU boxes for models.

### 2. Linux runner (`arlin` in this lab)

- Ubuntu **24.04** with the **6.8** GA kernel. Do not install 26.04 on the runner. Do not copy this mix onto `devlin`.
- AppArmor **off** (`apparmor=0` on the kernel cmdline, units masked) so nested Bubblewrap in `docker-test` works. Do **not** put `"apparmor-profile"` in Docker `daemon.json` — Engine 29.7.x refuses to start.
- Docker Engine for **tests**, not for the listener.
- Native systemd listener (`actions.runner.<org>-<repo>.<name>`), not a compose container with `restart=always`.

### 3. Windows runner (`arwin`)

- Local account. No Entra/Intune.
- Official runner as a **Windows service**. One listener. `RemoteSigned` at LocalMachine so `run:` steps work.
- Keep a legacy label if old workflows still name the previous Windows box.

### 4. macOS runner (`macmini`)

- System **LaunchDaemon**, not a user LaunchAgent, so it comes up with zero GUI login.
- `KeepAlive` + `RunAtLoad`. Prevent sleep on AC power. Display sleep is fine.
- This is the box that used to destroy GitHub-hosted budgets. Owning it is the whole trick.

### 5. Register against the **private product repo**

Create a runner in the GitHub UI for `ironclad-ai`, copy the token to the box, run `config.sh` / `config.cmd` **once**. Do not commit the token. Do not `--replace` unless GitHub says the old id is dead.

### 6. Point every workflow at those labels

Search the repo for `ubuntu-latest`, `windows-latest`, `macos-latest`. Each hit is a bill. Replace with the self-hosted label sets above.

## Rules that keep the bill at zero compute

1. **Runners are not editors.** `devlin` writes. `arlin`/`arwin`/`macmini` check.
2. **GPUs are not runners.** `spark` and `ai4090` serve LLMs. One GPU occupant.
3. **Windows is not the workspace.** RDP is a viewport. The tree lives on Linux.
4. **No IPs in the product repo.** Aliases in SSH config. A naming-gate CI job can fail a PR that pastes a LAN address.
5. **One listener per machine.** Native systemd / Windows service / LaunchDaemon.
6. **Hosted labels are a regression.** `ubuntu-latest` is how you go broke again.

## What this is not

- Not a SaaS. Not a Terraform module.
- Not permission to copy AppArmor-off or the 6.8 kernel pin onto the **dev** Tiny.
- Not a weight mirror. Models are documented in the sibling Spark / 4090 repos.
- Not a promise that GitHub is free. Seats, LFS, and artifact storage still exist. **Job minutes** are what this lab removes.

## License

MIT for the notes and diagrams in this tree.
