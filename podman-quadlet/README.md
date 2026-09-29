# Forget "podman generate systemd": How Quadlet really works

**All Systems Go! 2026 · Berlin**
Kritik Sachdeva · IBM

📄 [Slides (PDF)](slides/podman-quadlet-asg2026.pdf)

## Abstract

`podman generate systemd` turns a running container into a unit file. That unit is a snapshot, so changing one flag means stopping, removing, re-running, regenerating, reinstalling and reloading. The command you ran and the unit on disk drift apart.

Quadlet replaces that workflow. It is a **systemd generator**: you write a small declarative file, and at `daemon-reload` time it is compiled into a real systemd service. This talk explains how that compilation works, how dependencies become systemd edges, and how to debug it when something breaks.

## Agenda

1. What problem Quadlet solves
2. Groundwork: systemd, generators, Podman
3. What Quadlet is
4. Dependencies, on a real stack
5. Debugging when compilation fails

## Key ideas

### Quadlet is a generator

Generators are small executables that turn config systemd does not understand into unit files before any unit is loaded. You already run several on every boot (`systemd-fstab-generator`, `systemd-getty-generator`, `systemd-cryptsetup-generator`). Their output is transient: it is written to tmpfs, rebuilt on every reload and gone on reboot.

```
webserver.container  →  podman-user-generator  →  webserver.service  →  systemd starts it
  (you write this)       (on daemon-reload)         (in tmpfs)            (podman run, supervised)
```

| Manager | Generator | Your files live in |
|---------|-----------|--------------------|
| System (root) | `/usr/lib/systemd/system-generators/podman-system-generator` | `/etc/containers/systemd/` |
| User (rootless) | `/usr/lib/systemd/user-generators/podman-user-generator` | `~/.config/containers/systemd/` |

### Source and artifact

- `*.container` is **source**. You write it and keep it in version control.
- `*.service` is an **artifact**. It is rebuilt in tmpfs on every reload. Never edit it.

```console
$ systemctl --user show web.service -p FragmentPath
FragmentPath=/run/user/1000/systemd/generator/web.service
```

Nothing watches your files. Editing a `.container` does nothing until `systemctl --user daemon-reload`.

### Every `[Container]` key is a flag

| Quadlet key | `podman run` flag |
|-------------|-------------------|
| `Image=nginx:alpine` | `nginx:alpine` |
| `PublishPort=8080:80` | `-p 8080:80` |
| `Volume=webdata:/data` | `-v webdata:/data` |
| `Network=webnet.network` | `--network webnet` |

The four file types are `.container`, `.volume`, `.network` and `.pod`.

## Demo stack

Three files in `~/.config/containers/systemd/`:

```ini
# webcontent.volume
[Volume]
VolumeName=webcontent
```

```ini
# webnet.network
[Network]
NetworkName=webnet
```

```ini
# webserver.container
[Unit]
Description=Demo web server

[Container]
Image=nginx:alpine
Network=webnet.network
Volume=webcontent.volume:/usr/share/nginx/html
PublishPort=8080:80

[Install]
WantedBy=default.target
```

## Demos

### Demo 1: start one unit, get three

🎬 [demos/demo1-dependencies.mov](demos/demo1-dependencies.mov)

```bash
systemctl --user daemon-reload
systemctl --user start webserver
curl -s localhost:8080 | head -1
systemctl --user list-dependencies webserver
systemctl --user stop webnet-network.service
```

You never started the volume or the network. `Requires=` pulled them in, and stopping the network propagates the stop back to the web server.

```
default.target
  └─ Wants= → webserver.service
                 ├─ Requires= · After= → webcontent-volume.service
                 └─ Requires= · After= → webnet-network.service
```

You wrote two lines, and Quadlet wrote the graph.

**`Requires=` and `After=` are separate axes.** `Requires=` says what must be there. `After=` says when you may start. With only `Requires=`, your app can race its database on boot. Quadlet emits both.

**You cannot `enable` a generated unit.** It lives in tmpfs, so there is nothing on disk to symlink. Put `WantedBy=` in the `[Install]` section of the source file instead.

### Demo 2: breaking it on purpose

🎬 [demos/demo2-debugging-failures.mov](demos/demo2-debugging-failures.mov)

Two failures that look alike from the outside:

**Fault 1: it never compiled** (`Network=does-not-exist.network` gives `Failed: unit not found.`)

| Step | Command | What it tells you |
|------|---------|-------------------|
| 01 | `systemctl --user status webserver` | No such unit. systemd never saw one. |
| 02 | `systemctl --user cat webserver` | Nothing to show. Compilation produced no file. |
| 03 | `journalctl -b -t quadlet-generator` | The generator already logged the error this boot. |
| 04 | `/usr/lib/systemd/user-generators/podman-user-generator --user --dryrun` | Re-run it by hand to see the failing key. |
| 05 | Fix the source, then `daemon-reload` | The unit exists again. |

**Fault 2: it compiled, then died** (`PublishPort=80:80` gives `Job failed. See journal.`)

| Step | Command | What it tells you |
|------|---------|-------------------|
| 06 | `systemctl --user status webserver` | The unit exists and failed, so compilation was fine. |
| 07 | `journalctl --user -u webserver -n 50` | `cannot bind to port 80: permission denied` |
| 08 | `podman ps -a` · `podman inspect` | Podman has the last word, not systemd. |
| 09 | `PublishPort=8080:80`, then `daemon-reload` | Rootless containers cannot bind below port 1024. |

## Requirements

- Podman 4.4+ (Quadlet ships with Podman)
- systemd with a user session; enable lingering for boot-time rootless services (`loginctl enable-linger $USER`)

## Takeaways

- `.container` is source. `.service` is output.
- References become systemd edges, not just Podman flags.
- When it fails, debug the generator before the container.

> Quadlet is not magic. It is a generator. Once you can see what it generates, you can reason about it.
