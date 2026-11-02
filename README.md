# Automated Server Provisioning with Cloud-Init

Provisions an Ubuntu 24.04 VM with Docker, Git, curl, FFmpeg, a `testpress` user,
and a small HTTP app — all through cloud-init, no manual SSH setup.

## Usage

1. Install [Multipass](https://multipass.run/)
2. `multipass launch 24.04 --name testpress-vm --cloud-init cloud-init.yaml`
3. `multipass info testpress-vm` to get the VM's IP
4. `curl http://<ip>:8080` → `Hello from Testpress!`

## Why systemd instead of `docker run --restart`

I tried the plain `--restart unless-stopped` flag first, but `docker kill` doesn't
actually trigger it — Docker treats that as a manual stop, not a crash, so it won't
auto-restart. Switched to a systemd unit (`Restart=always`) instead. Also makes it
start on boot and gives normal `systemctl status` / `journalctl -u testpress-app`.

## Verifying it

```
curl http://<ip>:8080/            # Hello from Testpress!
curl http://<ip>:8080/health      # ok
multipass exec testpress-vm -- docker ps     # shows "healthy"
multipass restart testpress-vm
curl http://<ip>:8080/            # still works, no manual restart
```

To check re-running cloud-init doesn't duplicate anything:

```
multipass exec testpress-vm -- sudo cloud-init clean --logs
multipass exec testpress-vm -- sudo cloud-init init --local
multipass exec testpress-vm -- sudo cloud-init modules --mode=config
multipass exec testpress-vm -- sudo cloud-init modules --mode=final
```

Comes back clean — the `users` module skips if the user exists, `write_files` just
overwrites with the same content, `docker build` is a no-op on an unchanged
Dockerfile, and `systemctl enable --now` on an already-running service does nothing.

## How cloud-init runs

Four stages on boot: `init-local` (before networking, finds the datasource) →
`init` (users, packages, write_files) → `config` (`runcmd`, etc.) → `final`
(last-mile scripts).

Most modules default to running **once per instance** — cloud-init checks the
instance-id against what it saw last boot, and if it matches, it skips re-running
them (tracked via semaphore files under `/var/lib/cloud/instances/<id>/sem/`).
That's why a plain reboot doesn't redo any of the setup — it's not diffing state,
it just doesn't run again. A few modules (like `scripts-user`) default to `always`
for things that are supposed to rerun every boot.

## Bonus questions

- **First boot detection:** compares the datasource's instance-id against the one
  cached from last boot (`/var/lib/cloud/data/instance-id`). Different id = fresh
  instance, run everything.
- **cloud-init vs Dockerfile:** Dockerfile bakes an immutable image at build time.
  cloud-init configures a live, already-booted VM at boot time, and the same image
  can get different user-data per instance.
- **user-data vs meta-data:** user-data is what you provide (the whole
  `cloud-init.yaml`). meta-data is instance info the platform provides
  (instance-id, hostname) — both come in via a small seed disk with Multipass/NoCloud.
