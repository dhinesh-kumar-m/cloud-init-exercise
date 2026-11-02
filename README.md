# Automated Server Provisioning with Cloud-Init

Provisions an Ubuntu 24.04 VM with Docker, Git, curl, FFmpeg, a `testpress` user,
and a small HTTP app — all through cloud-init, no manual SSH setup.

## Usage

1. Install [Multipass](https://multipass.run/)
2. `multipass launch 24.04 --name testpress-vm --cloud-init cloud-init.yaml`
3. `multipass info testpress-vm` to get the VM's IP
4. `curl http://<ip>:8080` → `Hello from Testpress!`

More to come (verification steps, how cloud-init works) as the setup gets fleshed out.
