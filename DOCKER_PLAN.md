# Feature Plan: Official Docker Support for MPD

> **Status:** plan only — nothing described here is implemented yet.
> **Audience:** the agent/developer implementing the feature.
> **Housekeeping:** this file is a working document. Delete it (or move the
> user-facing parts into `doc/user.rst`) before opening the upstream PR to
> `MusicPlayerDaemon/MPD`.

## 1. Goal

Let people build and run MPD from this source tree as a container, with:

1. A multi-stage `Dockerfile` that compiles MPD from the checked-out source
   and produces a small, non-root runtime image.
2. A default configuration that works in a container with no host audio
   device at all (HTTP stream output), plus documented recipes for ALSA,
   PulseAudio, PipeWire, and Snapcast.
3. A Compose example for the common "music library + persistent state" setup.
4. A GitHub Actions workflow that builds the image for `linux/amd64` and
   `linux/arm64`, smoke-tests it, and (only when opted in) publishes it to
   GHCR.
5. User documentation in the Sphinx manual and a `NEWS` entry.

### Non-goals

- No changes to MPD's C++ code or Meson build logic. Everything must work with
  the existing build system. If a code change turns out to be needed, stop and
  raise it as a separate proposal.
- No Alpine/musl image (see §9).
- No publishing to Docker Hub. GHCR only, and only when opted in.
- No `linux/arm/v7` in the first iteration; it can come later (see §9).
- No web UI, `mpc`, or other clients bundled in the image.

## 2. Facts from the codebase that shape the design

Verify these before relying on them. File references are relative to the repo
root.

| Fact | Where | Consequence |
|---|---|---|
| Needs a C++23 compiler (GCC ≥ 14 or clang 19) and Meson ≥ 1.2 | `meson.build` (project(), GCC version check), `doc/user.rst` "Compiling from source" | Debian **trixie** is the oldest Debian with GCC 14 by default. Use `debian:trixie-slim`. |
| `doc/user.rst` already lists the full trixie `apt install` dependency set | `doc/user.rst` ~L70–110 | Use it as the starting point for the build-stage package list. Keep the two lists in sync, or cross-reference them. |
| Config file search: `$XDG_CONFIG_HOME/mpd/mpd.conf`, `~/.mpdconf`, `~/.mpd/mpd.conf`, then `$prefix/$sysconfdir/mpd.conf` | `src/CommandLine.cxx` ~L420, `meson.build` L133 | Pass the config path explicitly on the command line (`/etc/mpd.conf`) so the container never depends on `$HOME` or prefix. |
| With no `log_file` and no systemd, log output goes to **syslog or `/dev/null`** unless `--stderr` is passed | `src/LogInit.cxx` `log_init()` / `setup_log_output()` | The container **must** start MPD with `--no-daemon --stderr`, or `docker logs` will be empty. |
| MPD handles SIGINT/SIGTERM (clean shutdown, writes state file) and SIGHUP through its own handlers | `src/unix/SignalHandlers.cxx` | `docker stop` shuts down cleanly. Children still need reaping when MPD is PID 1 (the `pipe` output and others spawn processes), so run under `tini`. |
| If io_uring setup fails, MPD logs it at info level and continues | `src/event/Thread.cxx` ~L60–77 | Docker's default seccomp profile blocks io_uring on newer Engine versions. That is harmless; document it and don't loosen seccomp. |
| If real-time scheduling fails, MPD logs it and continues | `src/event/Thread.cxx`, `src/output/Thread.cxx` ~L445 | Optional `--cap-add SYS_NICE --ulimit rtprio=40`, the same limit as `systemd/system/mpd.service.in`. |
| `zeroconf=auto` resolves to avahi only if D-Bus is found | `src/zeroconf/meson.build` | Keep `dbus` enabled at build time so avahi compiles in. Disable `udisks`, since it's useless in a container. |
| Meson always recurses into `doc/` and installs `AUTHORS`, `COPYING`, `NEWS`, `README.md`, `mpd.svg` | `meson.build` L672–685 | `.dockerignore` must **not** exclude these files or `doc/`. `android/` and `win32/` are only entered on those platforms, so they can be excluded. |
| The `systemd/` subdir is only entered when libsystemd is found | `meson.build` L669 | Build with `-Dsystemd=disabled`: no unit files get installed, and there's no `sd_notify` or journal detection. |
| Version string comes from `vcs_tag()` (git) and falls back to the project version | `meson.build` L114, `src/GitVersion.cxx` | `.git` will be excluded from the build context, so `mpd --version` reports `0.25` without a git suffix. That's fine. The exact commit goes into the OCI labels instead. |
| Default ports: protocol 6600; httpd output 8000 (default); snapcast output 1704 | `src/Listen.cxx` L30, `doc/plugins.rst` | `EXPOSE 6600 8000`. Document 1704 for Snapcast. |
| CI uses `--wrap-mode nofallback` with distro packages | `.github/workflows/build.yml` | Do the same in the image for reproducible builds with no downloads during `meson setup`. |
| `build.yml` has a `paths-ignore` list | `.github/workflows/build.yml` | Add `docker/**` to it so Docker-only changes don't trigger the full native CI matrix. |
| Dependabot already tracks `github-actions` | `.github/dependabot.yml` | Add a `docker` ecosystem entry for `/docker` so base-image digests stay current. |

## 3. Deliverables (file layout)

Keep everything self-contained under `docker/` so the upstream diff is easy to
review, and leave the repo root untouched.

```
docker/
  Dockerfile                  # multi-stage build (build → runtime)
  Dockerfile.dockerignore     # BuildKit per-Dockerfile ignore file (context = repo root)
  mpd.conf                    # container-oriented default config → /etc/mpd.conf
  compose.yaml                # example Compose deployment
  README.md                   # short pointer to the manual section + quick start
.github/workflows/build_docker.yml   # CI build, smoke test, optional publish
.github/workflows/build.yml          # (edit) add 'docker/**' to both paths-ignore lists
.github/dependabot.yml               # (edit) add docker ecosystem for /docker
doc/user.rst                         # (edit) new "Running in Docker" section under Installation
NEWS                                 # (edit) one line under "ver 0.25 (not yet released)"
```

Build invocation (the context is the repo root; the Dockerfile lives in `docker/`):

```sh
docker build -f docker/Dockerfile -t mpd .
```

`Dockerfile.dockerignore` next to the Dockerfile is honoured by BuildKit, which
is the default builder since Docker Engine 23. That avoids adding a root-level
`.dockerignore`. Note in `docker/README.md` that the legacy builder is not
supported.

## 4. Dockerfile design

### 4.1 Stages

1. **`build`** (`FROM debian:trixie-slim AS build`)
   - `apt-get install --no-install-recommends` the toolchain (`g++`, `meson`,
     `ninja-build`, `pkgconf`) and the `-dev` packages from §4.2.
   - Use BuildKit cache mounts for `/var/cache/apt` and `/var/lib/apt/lists`
     (`RUN --mount=type=cache,...`). Optionally mount a ccache directory too.
     The build must still work without a warm cache.
   - `COPY . /src`, then:
     ```sh
     meson setup /src/output/docker /src \
       --buildtype=release -Db_ndebug=true \
       --prefix=/usr/local --sysconfdir=/etc \
       --wrap-mode=nofallback \
       -Ddocumentation=disabled -Dtest=false \
       -Dsystemd=disabled -Dudisks=disabled \
       -Dzeroconf=avahi -Ddbus=enabled \
       -Dpipewire=enabled -Dpulse=enabled -Dalsa=enabled \
       -Dio_uring=enabled \
       $MPD_MESON_ARGS
     meson compile -C /src/output/docker
     DESTDIR=/install meson install -C /src/output/docker --strip
     ```
   - `ARG MPD_MESON_ARGS=""` lets users add or override features without
     editing the Dockerfile. Document it.
   - Features not listed above are left on `auto`, so everything whose `-dev`
     package is installed gets built. Set an explicit `-D...=enabled` only for
     features the default `mpd.conf` or the documented recipes depend on, so a
     missing package fails the build instead of silently dropping a plugin.
   - **Resolve runtime packages automatically.** Don't hand-maintain a list
     of runtime library package names; trixie's `t64` renames make that
     brittle. In the build stage, write `/install/runtime-packages.txt`:
     `ldd /install/usr/local/bin/mpd` → the resolved `.so` paths → `dpkg -S`
     → a unique list of package names. Watch for the usr-merge pitfall: `ldd`
     may print `/lib/...` while dpkg recorded `/usr/lib/...`, or the other
     way round. Try the path as printed, its `realpath`, and the
     `/usr`-prefixed or unprefixed variant. **Fail the build** if any library
     can't be mapped to a package. Put this logic in a small shell snippet
     inside the Dockerfile, or in `docker/resolve-runtime-deps.sh` if it gets
     longer than ~15 lines.

2. **`runtime`** (`FROM debian:trixie-slim`, the final/default stage)
   - `apt-get install --no-install-recommends` the packages from
     `runtime-packages.txt`, plus these extras:
     - `tini`, the init process for PID 1
     - `ca-certificates`, needed for HTTPS streams (curl input) and Qobuz
     - `libpipewire-0.3-modules` and `libspa-0.2-modules`. PipeWire clients
       `dlopen` protocol modules that `ldd` can't see. Verify the PipeWire
       recipe works; if these packages turn out to be unnecessary, drop them.
     - `tzdata` is optional. Skip it unless log timestamps turn out wrong.
   - Then `rm -rf /var/lib/apt/lists/*`.
   - `COPY --from=build /install/usr/local /usr/local`
   - Create the user and group `mpd` with a fixed UID/GID, set by
     `ARG MPD_UID=1000` and `ARG MPD_GID=1000`. Add it to the `audio`
     group. The group helps only when host and container GIDs match; §6
     documents `--group-add` for the general case.
   - Create `/var/lib/mpd/{playlists}` and `/run/mpd`, owned by `mpd:mpd`,
     and `/music`, left empty as a mount point.
   - `COPY docker/mpd.conf /etc/mpd.conf`
   - `VOLUME ["/var/lib/mpd"]`. Don't declare `/music` as a volume; it's
     always a user bind mount, and declaring it would create stray anonymous
     volumes.
   - `EXPOSE 6600 8000`
   - `USER mpd`
   - `ENTRYPOINT ["/usr/bin/tini", "--"]`
   - `CMD ["mpd", "--no-daemon", "--stderr", "/etc/mpd.conf"]`
     - `--no-daemon`: stay in the foreground.
     - `--stderr`: log to stdout/stderr. Required; see §2.
     - An explicit config path, so no search-path surprises.
     - `docker run mpd mpd --version` must keep working, which is why this is
       `CMD` rather than `ENTRYPOINT`.
   - `HEALTHCHECK` using bash's `/dev/tcp`, so no extra packages are needed:
     connect to `127.0.0.1:6600` and check that the greeting starts with
     `OK MPD`. Interval 30s, timeout 5s, start-period 30s, since the first
     database scan of a large library can delay startup. Document that users
     who bind only a Unix socket must override or disable it.
   - OCI labels:
     - `org.opencontainers.image.title="Music Player Daemon"`
     - `org.opencontainers.image.source` (repo URL, passed as a build arg by CI)
     - `org.opencontainers.image.licenses="GPL-2.0-or-later"` (matches the
       SPDX headers in `src/`)
     - `org.opencontainers.image.revision` and
       `org.opencontainers.image.version` (build args, set by
       `docker/metadata-action` in CI)

### 4.2 Build-stage package list

Start from the trixie list in `doc/user.rst`, then:

- **Drop** `libsystemd-dev` (it appears twice there), `libgtest-dev`, and
  `libsidplay2-dev libsidutils-dev libresid-builder-dev`. The sidplay
  packages are optional; include them only if they're still in trixie and
  build cleanly.
- **Keep** `libdbus-1-dev` and `libavahi-client-dev` (for zeroconf),
  `libpipewire-0.3-dev`, `libpulse-dev`, `libasound2-dev`, `libcurl4-gnutls-dev`,
  `libsqlite3-dev` (stickers), `libicu-dev`, `libpcre2-dev`, `libfmt-dev`,
  `nlohmann-json3-dev`, `libsoxr-dev`, and
  `libsamplerate0-dev`, plus all the decoder and encoder libraries.
- **Add** `liburing-dev` and `zlib1g-dev` if they're missing from the manual's list.

If a package in that list doesn't exist in trixie, drop it and note the
reason in a comment. If you fix `doc/user.rst` along the way, keep that fix in
its own commit.

### 4.3 Image size

Measure the final image and record the size in `docker/README.md`. Don't
chase size at the cost of features. The goal is "full-featured MPD", and
people who want a minimal build can use `MPD_MESON_ARGS`.

## 5. Default `docker/mpd.conf`

This is a container-oriented config. Every path is absolute and falls under
the two mount points. Sketch:

```conf
# Music Player Daemon — default configuration for the Docker image.
# Mount your music at /music (read-only is fine) and persist /var/lib/mpd.
# See https://mpd.readthedocs.io/en/stable/user.html for all options.

music_directory     "/music"
playlist_directory  "/var/lib/mpd/playlists"
db_file             "/var/lib/mpd/database"
state_file          "/var/lib/mpd/state"
sticker_file        "/var/lib/mpd/sticker.sql"

# No log_file / pid_file / user: the container runtime handles these.
# Logging goes to stdout because the image starts mpd with --stderr.

bind_to_address     "any"
port                "6600"

auto_update         "yes"
zeroconf_enabled    "no"      # needs host networking + D-Bus; see the manual

# Works without any audio device: listen at http://<host>:8000/
audio_output {
    type            "httpd"
    name            "HTTP Stream"
    encoder         "vorbis"
    port            "8000"
    quality         "5.0"
    format          "44100:16:2"
    always_on       "yes"
    tags            "yes"
}

# --- Uncomment ONE of the following for local playback. ---
# Each needs extra `docker run` flags; see the "Running in Docker" manual section.
#audio_output { type "alsa"     name "ALSA" }
#audio_output { type "pulse"    name "PulseAudio" }
#audio_output { type "pipewire" name "PipeWire" }
#audio_output { type "snapcast" name "Snapcast" }
```

Notes for the implementer:

- Write the commented-out blocks in the multi-line style used by
  `doc/mpdconf.example`. The one-liners above are only shorthand.
- Check that the `vorbis` encoder exists in the build
  (`mpd --version` lists the encoders). If it doesn't, use `lame`.
- Run `mpd --no-daemon --stderr /etc/mpd.conf` once inside the container and
  make sure it starts with **no warnings** about deprecated or unknown
  options.
- `auto_update` depends on inotify. It works on Linux bind mounts but not
  through Docker Desktop's file sharing on macOS or Windows. Say so in the
  docs.

## 6. Audio output recipes (for the docs and `compose.yaml` comments)

Each recipe gives the `mpd.conf` output block, the `docker run` flags, and
the Compose equivalent. Test each one before documenting it. If a recipe
can't be tested in CI, mark it as manually verified and note on which host.

| Output | Container flags | Notes |
|---|---|---|
| **httpd** (default) | `-p 8000:8000` | No host audio needed. |
| **ALSA** | `--device /dev/snd --group-add $(getent group audio \| cut -d: -f3)` | The host's `audio` GID usually isn't 1000 or the container's `audio` GID. `--group-add` with the numeric host GID fixes that. |
| **PulseAudio** | `-v $XDG_RUNTIME_DIR/pulse/native:/run/pulse/native -e PULSE_SERVER=unix:/run/pulse/native --user $(id -u):$(id -g)` | The UID must match the host user that owns the socket. Also mount `~/.config/pulse/cookie` read-only if the server needs it. Works with `pipewire-pulse` too. |
| **PipeWire** | `-v $XDG_RUNTIME_DIR/pipewire-0:/run/user/1000/pipewire-0 -e XDG_RUNTIME_DIR=/run/user/1000 --user $(id -u):$(id -g)` | Adjust `1000` to the host UID. Needs the `*-modules` packages from §4.1. |
| **Snapcast** | `-p 1704:1704` | MPD acts as the Snapcast server. Clients connect directly. |
| **FIFO → snapserver sidecar** | shared volume holding the FIFO | Optional. Document only if tested. |

Also document:

- **Running with `--user`**: the image doesn't depend on the `mpd` user name,
  so any UID works as long as it can write to `/var/lib/mpd`. Named volumes
  are initialised with `mpd:mpd` ownership. With bind mounts, the user must
  `chown` the directory.
- **Real-time priority** (optional): `--cap-add SYS_NICE --ulimit rtprio=40`.
- **io_uring**: an "io_uring could not be initialised" info log line under
  the default seccomp profile is expected and harmless. Don't recommend
  `--security-opt seccomp=unconfined`.
- **Zeroconf**: needs `--network host` and
  `-v /run/dbus/system_bus_socket:/run/dbus/system_bus_socket`, with
  avahi-daemon running on the host. Set `zeroconf_enabled "yes"`.
- **Security**: the MPD protocol is unauthenticated by default. The Compose
  example publishes ports on `127.0.0.1` and explains how to open them up,
  with `password` / `host_permissions` as options (see `doc/user.rst`
  "Permissions and Passwords"; netmasks in `host_permissions` are new in 0.25).
- **Custom config**: bind-mount a file over `/etc/mpd.conf` with `:ro`.
- **Reload**: `docker kill -s HUP <ctr>` reopens log files and flushes caches.
  That matches MPD's existing SIGHUP behaviour and is *not* a config reload;
  say so plainly.

## 7. `docker/compose.yaml`

```yaml
services:
  mpd:
    build:
      context: ..
      dockerfile: docker/Dockerfile
    image: mpd:local
    init: false            # tini is already the image ENTRYPOINT
    restart: unless-stopped
    ports:
      - "127.0.0.1:6600:6600"
      - "127.0.0.1:8000:8000"
    volumes:
      - ${MUSIC_DIR:-./music}:/music:ro
      - mpd-data:/var/lib/mpd
      # - ./mpd.conf:/etc/mpd.conf:ro
    # --- ALSA ---
    # devices: ["/dev/snd:/dev/snd"]
    # group_add: ["<host audio gid>"]
    # --- PulseAudio / PipeWire: see the manual ---
volumes:
  mpd-data:
```

Validate it with `docker compose -f docker/compose.yaml config`, then bring it
up for real in §8.

## 8. CI: `.github/workflows/build_docker.yml`

Follow the conventions in `build.yml`: `permissions: contents: read` at the
top level, and the same `actions/checkout` major version.

- **Triggers:** `push` and `pull_request` on `master` and `v0.24.x` (same as
  `build.yml`), `workflow_dispatch`, and `push` of tags `v*`. Use a `paths`
  filter: `docker/**`, `src/**`, `meson.build`, `meson_options.txt`,
  `subprojects/**`, `doc/meson.build`, `.github/workflows/build_docker.yml`.
- **Job `build`**: a matrix over native runners, so there's no QEMU and the
  C++ build stays fast.
  - `linux/amd64` on `ubuntu-24.04`
  - `linux/arm64` on `ubuntu-24.04-arm`

  Steps: `docker/setup-buildx-action`, `docker/metadata-action`, and
  `docker/build-push-action` with `load: true` and the GHA cache
  (`cache-from/to: type=gha,scope=${{ matrix.platform }}`). Then run the
  **smoke test**:
  1. `docker run --rm IMAGE mpd --version`: exits 0 and the output lists the
     `httpd` output plus the `alsa`, `pulse`, and `pipewire` outputs.
  2. Start the container detached, with an empty `/music` tmpfs or dir
     mount. Wait until `docker inspect` reports `healthy`, with a timeout of
     about 60s.
  3. Talk to `127.0.0.1:6600`: send `status\nclose\n`, expect `OK MPD` and
     then `OK`.
  4. Send `outputs\nclose\n` and expect the `HTTP Stream` output to be
     listed with `outputenabled: 1`. Don't curl port 8000: the httpd output
     only opens its listener once playback has started, and CI has no music
     to play. `always_on` just keeps it open after playback stops.
  5. `docker stop` exits within the grace period, and `docker logs` shows
     MPD's shutdown messages, proving that signals and logging work.
  6. On failure, print `docker logs`.
- **Job `publish`**: runs only when all three hold:
  - `github.event_name != 'pull_request'`
  - `vars.DOCKER_PUBLISH == 'true'`, a repository variable, so forks and
    upstream opt in explicitly and nobody pushes images by accident
  - the ref is `master` or a `v*` tag

  Use `permissions: packages: write`. Push the per-arch images by digest,
  then merge them into a multi-arch manifest with
  `docker buildx imagetools create`; this is the standard
  `docker/build-push-action` "distribute build across multiple runners"
  pattern. Registry: `ghcr.io/${{ github.repository_owner }}/mpd`.
  Tags from `metadata-action`: `edge` for `master`, `X.Y.Z` / `X.Y` for tags,
  and `sha-<short>`.
- **`build.yml` edit**: add `'docker/**'` to both `paths-ignore` lists.
- **`dependabot.yml` edit**: add
  `package-ecosystem: "docker"` with `directory: "/docker"` and a weekly schedule.
- Pin the base image by digest (`debian:trixie-slim@sha256:...`) so Dependabot
  can bump it. If upstream prefers a floating tag, it's easy to switch.

## 9. Decisions and rejected alternatives

- **Debian trixie-slim, not Alpine.** MPD needs GCC 14 and C++23, and the
  project's docs and CI already target Debian and Ubuntu package names. With
  musl there'd be locale/ICU differences and an untested libc for a
  real-time audio app. Not worth it.
- **Debian, not Ubuntu 24.04.** Ubuntu 24.04 needs the `g++-14` package
  rather than the default compiler, and its `libfmt`, `meson`, etc. are
  older. Trixie is what `doc/user.rst` already documents.
- **Everything in `docker/`, not a root-level `Dockerfile`.** Easier upstream
  review, and it doesn't clutter the root. The cost is typing
  `-f docker/Dockerfile .`, which is documented.
- **Native arm64 runners, not QEMU.** Compiling MPD under emulation takes
  far too long. `arm/v7` (older Raspberry Pis) would need QEMU or a
  cross-compile, so it's deferred to a follow-up.
- **No entrypoint script, no env-var → config templating.** `mpd.conf` is the
  interface, and a bind-mounted file is simpler and more transparent than a
  shell templating layer. Revisit only if users ask.
- **`tini` in the image instead of relying on `docker run --init`.** Reaping
  child processes shouldn't depend on the user remembering a flag.
- **Fixed UID/GID 1000 by default.** It matches the typical first desktop
  user, which makes the PulseAudio and PipeWire recipes work unchanged for
  most people. Build args can change it, and `--user` works at runtime.

## 10. Documentation changes

1. `doc/user.rst`: add **"Running in Docker"** under *Installation*, after
   "Installing on Android" and before "Compiling from source". Cover:
   building the image, a quick start with `docker run`, volumes and ports,
   the Compose example, the audio recipes from §6, `MPD_MESON_ARGS`, and the
   caveats (inotify on Docker Desktop, io_uring log line, security). Use the
   file's existing RST conventions (`:program:`, `:file:`,
   `.. code-block:: none`).
2. `docker/README.md`: 20–40 lines. Quick start, a link to the manual
   section, the image size from §4.3, and "requires BuildKit".
3. `NEWS`: under `ver 0.25 (not yet released)`, add a line in the file's
   existing style, e.g. a new `* build` bullet with
   `  - Dockerfile and container image`. Match whatever bullet grouping
   upstream uses at merge time.
4. Build the docs locally
   (`meson setup -Ddocumentation=enabled ...` or `sphinx-build doc out`) and
   confirm the new section renders without warnings.

## 11. Implementation order (checklist)

Commit in small, reviewable steps. Suggested commits:

- [ ] **1. Dockerfile + dockerignore + mpd.conf.** `docker build -f
      docker/Dockerfile .` succeeds on amd64. The container starts, logs to
      `docker logs`, answers on 6600, and serves the stream on 8000.
- [ ] **2. Runtime dependency resolution** is automatic and fails loudly
      (test it by deliberately removing a runtime package).
- [ ] **3. Non-root user, volumes, healthcheck, labels, tini.** Verify
      `docker stop` gives a clean shutdown and that the state file is written
      and restored across restarts (queue and volume survive).
- [ ] **4. compose.yaml.** `docker compose -f docker/compose.yaml up` works
      with a sample music dir. The database builds and `auto_update` picks up
      a newly added file (on Linux).
- [ ] **5. Audio recipes** verified manually on a Linux host: ALSA, and
      PulseAudio or PipeWire, whichever the host runs. Record what was tested.
- [ ] **6. CI workflow** green on a PR in the fork, both architectures. The
      publish job is skipped when `DOCKER_PUBLISH` isn't set, then set it
      once to verify a push to GHCR and a multi-arch manifest
      (`docker buildx imagetools inspect`).
- [ ] **7. build.yml / dependabot.yml edits.**
- [ ] **8. Docs + NEWS.** The Sphinx build is clean.
- [ ] **9. Cleanup.** Remove this `DOCKER_PLAN.md` (or reduce it to nothing
      upstream would object to) before the upstream PR.

## 12. Acceptance criteria

- `docker build -f docker/Dockerfile -t mpd .` succeeds from a clean clone on
  amd64 and arm64 with no network access needed during `meson setup`
  (`nofallback`).
- `docker run --rm mpd mpd --version` shows the expected plugins.
- `docker run -d -p 6600:6600 -p 8000:8000 -v "$PWD/music:/music:ro" mpd`:
  - becomes `healthy`
  - `mpc -h 127.0.0.1 update && mpc -h 127.0.0.1 listall` lists the files
  - playback over the httpd stream is audible in a browser or VLC
  - `docker stop` completes in under 10s, and restarting with the same
    `/var/lib/mpd` volume restores the queue and position
- The container runs as non-root by default (`docker exec <ctr> id`).
- `docker logs` shows MPD's log output, with nothing going to syslog or
  `/dev/null`.
- CI: both architecture builds and smoke tests pass on PRs, and publishing
  happens only when opted in.
- No changes under `src/`, and `meson.build` / `meson_options.txt` are
  untouched.

## 13. Open questions for the maintainer (not blockers)

1. Should the published image live under the fork owner
   (`ghcr.io/eguthriewastaken/mpd`) until upstream adopts it? The workflow
   uses `github.repository_owner`, so it follows whoever runs it.
2. Would upstream prefer a root-level `Dockerfile` for discoverability? It's
   a trivial move later.
3. Is `arm/v7` wanted in the first iteration (Raspberry Pi 2/3 on a 32-bit
   OS)? If so, add a QEMU job and accept the long build time.
