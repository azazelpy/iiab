# IIAB as an offline medical library (en)

> Spanish version: [ES-MEDICO.md](ES-MEDICO.md)

Our own working branch on top of `iiab/iiab`. It does **not** aim to contribute upstream: it keeps
one concrete installation alive — a medical library in Spanish and English, no hotspot, behind the
home router.

## Installation

- Debian 13 (trixie) VM, 4 vCPU, 8 GB RAM, ~190 GB disk, hostname `iiab.lan`.
- Kiwix on port **3000**, nginx in front, content under `/library/zims`.
- Profile: [`vars/local_vars_es_medico.yml`](../vars/local_vars_es_medico.yml).

```bash
sudo install -m 644 vars/local_vars_es_medico.yml /etc/iiab/local_vars.yml
sudo iiab -f          # -f = fast: no interactive prompts
```

## Content

IIAB does **not** download the ZIM files: they are seeded by hand from a NAS, where each one is
checked against the official `.sha256` from `download.kiwix.org` before being copied.

```bash
# on the VM — root needs its key authorised on the NAS
sudo ssh-keygen -q -t ed25519 -N "" -f /root/.ssh/id_ed25519      # once
sudo cat /root/.ssh/id_ed25519.pub                                 # authorise on the NAS

# NOTE: ZIMs belong in /library/zims/CONTENT/, not in /library/zims/
sudo rsync -a --partial --info=progress2 --include='*.zim' \
  --exclude='*' -e "ssh -i /root/.ssh/id_ed25519" \
  root@<nas>:/path/to/zim/medicine/ /library/zims/content/

# IIAB builds the index itself: it scans content/ via get_zim_list()
sudo iiab-make-kiwix-lib -f
curl -s "localhost:3000/kiwix/catalog/v2/entries?count=50" | grep -oE "<totalResults>[0-9]+"
```

**The index is not rebuilt automatically.** `kiwix-serve` starts with
`--library /library/zims/library.xml`; copying `.zim` files does not update that XML. And
`iiab-make-kiwix-lib` scans **`/library/zims/content/`** only (`content = path + "/content/"` in
`iiab_lib.py:get_zim_list`), so a ZIM sitting in the root of `/library/zims` stays invisible even if
you index it by hand with `kiwix-manage`. `iiab` rewrites `library.xml` on every run: anything seeded
outside `content/` gets rolled back.

**`rsync` cannot copy remote→remote.** Attempting it from a third machine returns
`both source and destination cannot be remote files` — and if the command is piped into `tail`, it
exits with rc=0 and looks like it worked. The copy has to originate on one of the two ends; that is
why the VM holds its own key to the NAS and `rsync` runs **on the VM**.

Corpus: mdwiki (full medical encyclopedia, 10.75 GB), MedlinePlus ES/EN, WikiMed ES/EN, Medical
Library (field and emergency medicine), Military Medicine (26 manuals), Gutenberg class R, Wikibooks
(the *First Aid* wikibook), LibreTexts Medicine, clinical guides, clinical Q&A, NHS medicines.

## Rules that cost us time

1. **IIAB does not uninstall.** A `_install: False` on anything already listed in
   `/etc/iiab/iiab_state.yml` aborts the play with `DISALLOWED ... WHEN 'X_installed' is defined`.
   Leave `_install: True` and turn the service off with `_enabled: False`.
2. **`dnsmasq` and `hostapd` default to `enabled: True`.** On a LAN VM, dnsmasq can compete with the
   router's DHCP. They are disabled explicitly here.
3. **`remoteit` is a third-party tunnel** and ships enabled in the default install: keep it off.
   Check `tailscale` the same way if the VM should not join the tailnet.
4. **The `get_curl` module can hang without a timeout** while downloading a service's payload
   (observed with Matomo: 25 min, 0 bytes, no network socket). If a `TASK` writes nothing to the log
   for ~10 min, kill the module and re-run `sudo iiab` — it is resumable.
5. **`local_vars.yml` must be in place before the installer runs.** If it exists, IIAB uses it as is
   and asks nothing (`EXISTING /etc/iiab/local_vars.yml is being used`).

## Maintenance

The fork is watched by a nightly job that compares our profile against `default_vars.yml` and reviews
role changes before they reach production.

Remotes:

| Remote | Target | Role |
|---|---|---|
| `forgejo` | self-hosted instance (private) | **source of truth**, where work happens |
| `origin` | `github.com/azazelpy/iiab` (public) | publication |
| `upstream` | `github.com/iiab/iiab` | read-only |
