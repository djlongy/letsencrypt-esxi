# letsencrypt-esxi

Let's Encrypt certificate automation for standalone VMware ESXi hosts, packaged as a VIB.

Fork of [kmcbride3/letsencrypt-esxi](https://github.com/kmcbride3/letsencrypt-esxi) (originally [w2c/letsencrypt-esxi](https://github.com/w2c/letsencrypt-esxi)).

**This fork exists to provide a pre-built VIB as a GitHub Release asset**, so automation tools can download it directly without cloning the repo and building via Docker on every run. No code changes from upstream.

- **VIB version:** 1.2.0-beta7
- **Tested on:** ESXi 7.0, ESXi 8.0
- **Challenge types:** HTTP-01, DNS-01
- **License:** GPLv3

## Pre-built VIB Download

```
https://github.com/djlongy/letsencrypt-esxi/releases/latest/download/w2c-letsencrypt-esxi.vib
```

## Quick Start (Manual SSH Install)

```bash
wget -O /tmp/w2c-letsencrypt-esxi.vib https://github.com/djlongy/letsencrypt-esxi/releases/latest/download/w2c-letsencrypt-esxi.vib
esxcli software acceptance set --level=CommunitySupported
esxcli software vib install -v /tmp/w2c-letsencrypt-esxi.vib -f
```

The VIB installs to `/opt/w2c-letsencrypt/` and adds a weekly cron (Sunday 00:00) for automatic renewal.

## DNS-01 with Cloudflare

After install, create `/opt/w2c-letsencrypt/renew.cfg`:

```ini
CHALLENGE_TYPE="dns-01"
DNS_PROVIDER="cloudflare"
CF_API_TOKEN="<your-cloudflare-api-token>"
```

Lock down the file and trigger the first certificate request:

```bash
chmod 0600 /opt/w2c-letsencrypt/renew.cfg
/etc/init.d/w2c-letsencrypt start
```

### Supported DNS Providers

Cloudflare, Route53, DigitalOcean, Namecheap, GoDaddy, PowerDNS, DuckDNS, NS1, Google Cloud DNS, Azure DNS, Manual.

See `renew.cfg.example` in this repo for all configuration options.

## Ansible Automation

We deploy this VIB via an Ansible playbook. The workflow:

1. Downloads the VIB from this GitHub Release to the Ansible control node
2. Uploads to ESXi via SCP (ESXi can't fetch HTTPS GitHub URLs natively)
3. Sets acceptance level to `CommunitySupported` (fails gracefully if Secure Boot blocks it)
4. Installs or upgrades the VIB (handles payload mismatch by remove + reinstall)
5. Writes `renew.cfg` with Cloudflare DNS-01 credentials sourced from HashiCorp Vault
6. Triggers the initial certificate request

## Gotchas and Operational Notes

These were hard-won. Read before deploying.

### 1. Reboot after VIB install may be needed for wget

ESXi's `wget` can fetch HTTPS GitHub URLs, but after initial VIB installation or major changes you may need a reboot before network utilities work reliably. If `wget` fails with TLS errors, reboot the host first.

### 2. Secure Boot blocks CommunitySupported VIBs

If `esxcli software acceptance set --level=CommunitySupported` fails with a "Secure Boot enabled" error, you must disable Secure Boot in the host's BIOS/UEFI settings first. There is no software workaround.

### 3. Payload mismatch on upgrade

When upgrading from an older VIB version, you may hit:

```
unequal values of the 'payloads' attribute
```

Fix: remove the old VIB first, then install the new one.

```bash
esxcli software vib remove -n w2c-letsencrypt-esxi
esxcli software vib install -v /tmp/w2c-letsencrypt-esxi.vib -f
```

### 4. FQDN must be set before requesting a cert

Let's Encrypt issues certs for the ESXi host's FQDN. Set it before running the renewal:

```bash
esxcli system hostname set --fqdn esxi01.example.com
```

### 5. renew.cfg permissions

Must be `0600`. It contains your Cloudflare API token (or other provider credentials).

### 6. Force renewal

If the cert exists but you want to re-issue:

```bash
rm /etc/vmware/ssl/rui.crt
/etc/init.d/w2c-letsencrypt start
```

### 7. Weekly cron

The VIB installs a cron job that runs every Sunday at 00:00 to check for renewal. No manual cron setup is needed.

## Building the VIB Yourself

If you need to rebuild after upstream changes:

```bash
git clone https://github.com/djlongy/letsencrypt-esxi.git
cd letsencrypt-esxi
docker build --platform linux/amd64 -t letsencrypt-esxi -f build/Dockerfile .
cid=$(docker create --platform linux/amd64 letsencrypt-esxi)
docker cp "$cid:/root/letsencrypt-esxi/build/w2c-letsencrypt-esxi.vib" ./w2c-letsencrypt-esxi.vib
docker rm "$cid"
```

The `--platform linux/amd64` flag is required. The `lamw/vibauthor` base image is x86_64 only.

## Uninstall

```bash
esxcli software vib remove -n w2c-letsencrypt-esxi
```

This removes the VIB, undoes system changes, and generates a new self-signed certificate.

## Upstream

For full feature documentation, issue tracking, and contribution guidelines, see the upstream repos:

- [kmcbride3/letsencrypt-esxi](https://github.com/kmcbride3/letsencrypt-esxi)
- [w2c/letsencrypt-esxi](https://github.com/w2c/letsencrypt-esxi)
