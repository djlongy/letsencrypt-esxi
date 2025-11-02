# Copilot Instructions for letsencrypt-esxi

## Project Overview
This is a VMware ESXi VIB (vSphere Installation Bundle) that automates Let's Encrypt certificate management on standalone ESXi servers. The solution handles both HTTP-01 and DNS-01 ACME challenges with automatic renewal via cron jobs.

**Critical Compatibility Requirement**: All code must be fully compatible with ESXi 6.5 and above. Scripts and Python code must only use functionality available in the limited ESXi shell environment - no external dependencies or modern shell features that aren't present in the minimal ESXi BusyBox environment.

## Architecture & Key Components

### Core Scripts
- **`renew.sh`** - Main certificate lifecycle manager with config backup/restore logic, firewall management, and ACME integration
- **`w2c-letsencrypt`** - ESXi service init script that handles VIB installation/removal and calls `renew.sh`
- **`acme_tiny.py`** - Modified ACME client with DNS-01 challenge support via external DNS API framework

### Configuration System
- **`renew.cfg.example`** - Template with comprehensive provider settings and validation patterns
- **`renew.cfg`** - User config file with backup persistence across ESXi upgrades at `/etc/w2c-letsencrypt/`
- Config backup rotation: `.bak` (latest) → `.bak.old` (previous) with timestamp-based updates

### DNS Provider Framework
- **`dnsapi/dns_api.sh`** - Universal DNS provider interface with standardized command structure: `<command> <domain> [txt_value]`
- **`dnsapi/dns_*.sh`** - Provider-specific implementations (Cloudflare, manual, etc.)
- TXT value calculation uses same SHA256+base64 encoding as `acme_tiny.py`

### VIB Packaging
- **`build/`** - Docker-based VIB creation using `lamw/vibauthor` container
- **`build/build.sh`** - Orchestrates Docker build and artifact extraction
- Output: `.vib` file and offline bundle for ESXi installation

## Critical Development Patterns

### ESXi-Specific Integrations
```sh
# HTTP proxy routing for ACME challenges
echo "/.well-known/acme-challenge local 8120 redirect allow" >> /etc/vmware/rhttpproxy/endpoints.conf

# Firewall ruleset management with state restoration
esxcli network firewall ruleset set -e true -r webAccess
trap cleanup_firewall EXIT INT TERM

# Certificate installation and service restart
cp -p "$LOCALDIR/$KEY" "$VMWARE_KEY"
for s in /etc/init.d/*; do
  if [ -x "$s" ] && grep -q "ssl_reset" "$s" 2>/dev/null; then
    "$s" ssl_reset 2>/dev/null || true
  fi
done
```

### Challenge Type Detection & Automation
- **HTTP-01**: Requires inbound firewall access, starts Python HTTP server on port 8120
- **DNS-01**: Requires outbound firewall access for ACME and DNS provider APIs, prevents automated renewal with manual providers in cron context
- Automated run detection: checks TTY, TERM, PPID for cron/background execution
- Both challenge types require `ssl_reset` service restart after certificate installation

### Configuration Persistence Logic
Uses sophisticated backup rotation to survive VIB upgrades:
- Main config: `/opt/w2c-letsencrypt/renew.cfg` (ephemeral)
- Persistent backup: `/etc/w2c-letsencrypt/renew.cfg.bak` (survives upgrades)
- ESXi-compatible timestamp comparison using `ls -l` instead of `-nt` operator (BusyBox limitation)

### Error Handling & Validation
- Certificate domain mismatch detection via SAN comparison
- Let's Encrypt issuer validation before renewal checks
- Fallback to self-signed certificates on ACME failures
- DNS propagation wait with configurable timeouts
- Automatic key generation for fresh installs (account key, domain key, CSR)
- ESXi-compatible SSL service restart using static file analysis instead of service execution

## Development Workflows

### Local Testing
```sh
# Build VIB locally
./build/build.sh

# Test certificate renewal (dry run with staging)
DIRECTORY_URL="https://acme-staging-v02.api.letsencrypt.org/directory" ./renew.sh

# Force renewal for testing
rm /etc/vmware/ssl/rui.crt && /etc/init.d/w2c-letsencrypt start
```

### DNS Provider Development
1. Implement in `dnsapi/dns_<provider>.sh` following the standardized interface
2. Support commands: `add`, `rm`, `info`, `list`, `test`
3. Use environment variables for credentials (exported from `renew.cfg`)
4. Calculate TXT values via `calculate_txt_value()` in `dns_api.sh`

### Configuration Updates
- Always update `renew.cfg.example` as the canonical reference
- Include validation patterns and comprehensive provider examples
- Test config persistence across VIB upgrade/removal cycles

## Key Files for Common Tasks
- **Certificate logic**: `renew.sh` lines 110-130 (renewal checks), 290-320 (certificate installation and service restart)
- **Firewall management**: `renew.sh` lines 145-180 (challenge-specific rules), cleanup function
- **DNS provider interface**: `dnsapi/dns_api.sh` lines 1-100 (core framework)
- **VIB packaging**: `build/create_vib.sh` and `build/Dockerfile`
- **ESXi service integration**: `w2c-letsencrypt` (init script patterns)
- **Key generation**: `renew.sh` lines 258-275 (automatic key/CSR generation)

## Testing Considerations
- **ESXi Compatibility**: Must work on ESXi 6.5+ with limited BusyBox shell - avoid bash-specific syntax, modern utilities, or Python features not in ESXi's minimal environment
- ESXi-specific: hostname FQDN requirements, firewall ruleset names, service restart patterns
- ACME staging environment for certificate testing without rate limits
- DNS propagation timing varies by provider (default 30s, configurable)
- VIB acceptance level must be "community" for installation

## ESXi Environment Constraints
- **Shell**: Use POSIX-compliant `/bin/sh` syntax only (no bash arrays, `[[`, process substitution)
- **BusyBox Limitations**: Avoid `-nt`/`-ot` file test operators - use `ls -l` timestamp comparison instead
- **Python**: Limited to Python 2.7/3.x features available in ESXi - no pip packages, minimal standard library
- **Utilities**: BusyBox versions of common tools with reduced feature sets (grep, sed, awk, etc.)
- **Paths**: Use absolute paths and verify tool availability before use (`which`, `command -v`)
- **OpenSSL**: Core ESXi OpenSSL for cryptographic operations (certificate parsing, key generation)
- **Command Availability**: Check for `readlink -f`, `pidof`, `python3` availability and provide fallbacks
- **SSL Reset Syntax**: Use static file analysis for service detection: `grep -q "ssl_reset" "$s"` instead of executing services
