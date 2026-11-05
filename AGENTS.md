# AGENTS.md — pod-crabbox-k3s-gpu

The **NVIDIA-GPU k3s lease image** for the Crabbox `containerd` provider: the
`pod-crabbox-k3s-gpu` image (`charly.yml`) and the `nvidia-gpu-lease` candy
(`candy/nvidia-gpu-lease/`) it composes on top of `opencharly/pod-crabbox-k3s`.
This repo owns exactly the GPU delta — the NVIDIA userspace, the toolkit's CDI
refresh made runnable inside a lease, and the device-plugin manifest — plus the
probes that repeat the inherited k3s/lease contract at this image's boundary. The
lease contract, systemd, the inner engine and k3s belong to the base repos; a
change to any of them lands there, not here.

Canonical files:

- `charly.yml` — the image: `base: ghcr.io/opencharly/pod-crabbox-k3s` at a
  pinned tag, `entrypoint: [/sbin/init]`, `user: root`.
- `candy/nvidia-gpu-lease/charly.yml` — the layer requirement, the refresh-unit
  drop-in, the device-plugin manifest and the `check:` steps.
- `.github/workflows/image.yml` — the image build; it triggers on
  `tag-on-merge` **completion** (`workflow_run`), never on `push: tags`
  (`opencharly/.github#277`: tag-on-merge mints tags through the REST API, which
  emits no push event).
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-image:image` — the `candy:` image entry (base, user, entrypoint).
- `/charly-image:layer` — the candy body: `distro:` packages, `require:`, plan
  step verbs, and the `check:` surface.
- `/charly-distros:nvidia` — the GPU runtime layer this image composes: the
  toolkit, the distro package arms, and what a CDI spec does and does not prove.
- `/charly-check:check` — the check verbs, the ADE requirement, and the R10 gate.
- `/charly-internals:git-workflow` — before any git/PR action.

## What is unique here

- **The image tag IS the git tag.** `.github/workflows/image.yml` passes the
  newest `v*` tag (minus its `v`) to `charly box build --tag`.
- **The base is a pinned OCI artifact.** `base:
  ghcr.io/opencharly/pod-crabbox-k3s:<tag>` pulls the published k3s image.
  Advancing the base is a one-line tag bump, produced first by the k3s repo
  (producer-first).
- **The GPU userspace is NOT re-declared here.** `require:
  '@github.com/opencharly/layer-nvidia:<tag>'` is the one implementation of
  `nvidia-container-toolkit` + `nvidia-utils` + the Vulkan ICD wiring. Do not copy
  its package list or its plan steps into this repo (R2).
- **The package already ENABLES its refresh units.** `nvidia-container-toolkit`
  installs `nvidia-cdi-refresh.path` / `.service` AND their
  `multi-user.target.wants` vendor links. Do not add an `enable` step: that is the
  package's own job, and re-linking it here would be a second copy.
- **A lease must clear the vendor `ExecCondition`.** The vendor unit gates on
  `grep -qE "/(nvidia|nvidia-current)[.]ko" /lib/modules/%v/modules.dep || [ -e
  /dev/dxg ]`. A container runs the host kernel and carries no
  `/lib/modules/<release>` tree, so the condition can never hold and systemd
  SKIPS the unit — leaving no `/var/run/cdi/nvidia.yaml` at all. The drop-in
  `20-crabbox-lease.conf` resets the list with an empty `ExecCondition=`
  (`systemd.unit(5)`); do not replace it with a copied unit.
- **The CDI spec is refreshed, never snapshotted.** A one-shot
  `nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml` is the form
  CONTAINERD.md §14.2 rejects: the driver version appears throughout the spec, so
  a driver upgrade breaks every CDI mount. Only `/var/run/cdi` is regenerated.
- **The device plugin is a k3s manifest, not a `kubectl apply`.** The DaemonSet
  lives under `/var/lib/rancher/k3s/server/manifests/`, which k3s reconciles at
  startup. The three env vars are the row's and each is a real v0.20.1 flag:
  `DEVICE_LIST_STRATEGY` (`--device-list-strategy`), `NVIDIA_DRIVER_ROOT`
  (`--nvidia-driver-root`) and `NVIDIA_CDI_HOOK_PATH` (`--nvidia-cdi-hook-path`).
- **`cdi-cri` is why a workload needs no `runtimeClassName`.** The plugin passes
  the allocated device as a CRI `CDIDevices` id and containerd resolves it from
  `/var/run/cdi`; the lease's inner containerd keeps `crun` as its default
  handler.

## Build / validate / test

- `charly box validate` at the repo root.
- `charly box generate pod-crabbox-k3s-gpu` — inspect the emitted Containerfile,
  the label set, and the baked `ai.opencharly.description` plan.
- `charly check box pod-crabbox-k3s-gpu` — the build-scope `check:` steps.
- The PR image build builds and publishes nothing; the merge-time
  `workflow_run` path publishes
  `ghcr.io/opencharly/pod-crabbox-k3s-gpu:<git-tag>`.
- `nvidia-utils` is the long pole: it is a ~1 GB installed userland. That is the
  price of `nvidia-smi` and NVML; do not drop it to make the build smaller — the
  refresh unit's `ExecStart` runs `nvidia-smi -L` and the plugin loads
  `libnvidia-ml.so.1`.

## Modify this repo

- Keep the base tag in step: a base-contract, k3s, systemd or inner-engine fix
  lands in its owning repo first, then this repo's `base:` tag advances.
- Every claim in the plan needs a deterministic `check:`. A toolkit that merely
  installs without a device-level probe is not proof.
- `charly check box` runs only build-context `check:` steps. A claim that needs a
  running cluster — the CDI spec generated at boot, `nvidia.com/gpu` advertised,
  a workload pod exec'ing `nvidia-smi` — belongs in a deploy-context bed, not
  here, and the image description must not claim it as build-proven.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly`. Do not restate its rules here.
