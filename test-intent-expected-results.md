# RHEL for Edge Test Intent and Expected Results

**Purpose**: Explain what each test does, define a test intent and expected results.

---

## `ostree.sh` - kickstart

**What it tests**: OSTree commit deployed via kickstart

**Deployment method**: Network boot (PXE) with kickstart, OSTree commit served via HTTP

**Image type**: edge-commit (OSTree repository tarball)

**Use case**: Developer/admin who wants full control over installation using a custom kickstart.

**Test intent**: Verify the end-to-end lifecycle of an OSTree commit: build, install via kickstart (UEFI), upgrade via `rpm-ostree`, and validate post-deployment system health.

**Note**: The test script runs on a host VM (provisioned by Testing Farm) and creates a nested guest VM where the image is installed and validated.

**Test steps**
1. Define blueprint and compose image.
2. Install image on guest VM via kickstart.
3. Verify install: guest VM is reachable via SSH.
4. Compose a second image with updated blueprint content.
5. Run `rpm-ostree upgrade` on guest VM and reboot.
6. Verify upgrade: guest VM is reachable via SSH after reboot.
7. Run `check-ostree.yaml` Ansible playbook from host against guest VM.

**Expected results**
- Compose finishes with `FINISHED` status.
- Guest VM is reachable via SSH after install.
- Guest VM is reachable via SSH after upgrade.
- Ansible playbook exits with code 0.

**Conditional checks enabled** (see [Ansible validation checks](#ansible-validation-checks-check-ostreeyaml))
- `embedded_container`
- `firewall_feature`
- `sysroot_ro`
- `test_custom_dirs_files`

---

## `ostree-ng.sh` - bootable ISO

**What it tests**: OSTree commit deployed via ISO installer

**Deployment method**: ISO boot (BIOS, UEFI) and network boot (PXE) from extracted ISO content via HTTP (if supported)

**Image type**: edge-installer (bootable ISO with embedded OSTree commit)

**Use case**: Edge sites with no network, factory floor deployment (USB stick), air-gapped environments.

**Test intent**: Verify installation from ISO works on BIOS and UEFI (offline), optionally via HTTP boot from extracted ISO content, and that upgrade succeeds.

**Note**: The test script runs on a host VM (provisioned by Testing Farm) and creates nested guest VMs where the image is installed and validated.

**Test steps**
1. Define container blueprint and compose container image with OSTree commit.
2. Define installer blueprint and compose bootable ISO from container image.
3. (if supported) Extract ISO to HTTP server and install via HTTP boot.
4. (if supported) Verify install: HTTP boot guest VM is reachable via SSH.
5. (if supported) Run `check-ostree.yaml` Ansible playbook from host against HTTP boot guest VM.
6. Install image on BIOS guest VM from ISO.
7. Verify install: BIOS guest VM is reachable via SSH.
8. Run `check-ostree.yaml` Ansible playbook from host against BIOS guest VM.
9. Install image on UEFI guest VM from ISO.
10. Verify install: UEFI guest VM is reachable via SSH.
11. Compose a second image with updated blueprint content.
12. Run `rpm-ostree upgrade` on UEFI guest VM and reboot.
13. Verify upgrade: UEFI guest VM is reachable via SSH after reboot.
14. Run `check-ostree.yaml` Ansible playbook from host against UEFI guest VM.

**Expected results**
- Compose finishes with `FINISHED` status.
- (if supported) HTTP boot guest VM is reachable via SSH after install.
- BIOS guest VM is reachable via SSH after install.
- UEFI guest VM is reachable via SSH after install.
- UEFI guest VM is reachable via SSH after upgrade.
- Ansible playbook exits with code 0.

**Conditional checks enabled** (see [Ansible validation checks](#ansible-validation-checks-check-ostreeyaml))
- `embedded_container`
- `sysroot_ro`
- `test_custom_dirs_files`

---

## Ansible validation checks (`check-ostree.yaml`)

All test scripts run the `check-ostree.yaml` Ansible playbook against the deployed guest VM. The playbook validates system health using two groups of checks.

**Always run** (every test script)
- SELinux is in enforcing mode
- Root password is locked
- Deployed ostree commit matches the built commit
- Deployed ostree ref matches the expected ref
- `ostree-remount.service` is active
- Mount points correct: `/var` (`rw`), `/usr` (`ro`)
- No unexpected errors in `dmesg` (filters vary by platform, see Runtime-conditional)
- Podman can run containers (root and non-root)
- Greenboot packages installed, services enabled, `boot_success=1`
- Auto-rollback works when a failing systemd unit is deployed
- No failed systemd units

**Conditional** (controlled by variables passed from each test script)

| Variable | Checks activated | Set by |
|---|---|---|
| `fdo_credential` | FDO onboarding, TPM device, LUKS re-encryption | `ostree-simplified-installer`, `ostree-fdo-*` |
| `embedded_container` | embedded container images present and runnable | `ostree.sh`, `ostree-ng.sh` |
| `ignition` | `hello.service`, `log_trace.conf`, ignition firstboot | `ostree-ignition.sh` |
| `firewall_feature` | firewall zone customizations applied | `ostree.sh` |
| `test_custom_dirs_files` | custom directories, files, and systemd drop-ins | `ostree.sh`, `ostree-ng.sh` |
| `sysroot_ro` | `/sysroot` mounted `ro` (RHEL 9.2+, Fedora 37+) vs `rw` | all scripts (value depends on OS version) |

**Runtime-conditional** (triggered by deployed system state)
- **LVM deployments**: PV format, PV size, sysroot LV size validation
- **Post-upgrade**: `wget` package installation check
- **Post-rollback**: greenboot fallback detection, commit verification, persistent logging, `grubenv` state
- **Distribution/architecture-specific**: different `dmesg` error filters for RHEL/Fedora/CentOS on `x86_64`/`aarch64`
