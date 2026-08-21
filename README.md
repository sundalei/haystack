# haystack

Ansible provisioning for a two-node Elasticsearch cluster joined over a WireGuard
tunnel, with Kibana behind an Apache reverse proxy and Let's Encrypt TLS.

```
                 internet
                     │
        kibana.sundalei.cloud  es.sundalei.cloud
                     │
                     ▼
        ┌──────────────────────────────┐
        │ es2  31.97.136.158  (RHEL)   │
        │  Apache :443 ──► Kibana :5601│
        │  Elasticsearch (data only)   │
        └──────────────┬───────────────┘
                       │ WireGuard 10.10.0.0/24
                       │ transport :9300 over wg0
        ┌──────────────┴───────────────┐
        │ es1  187.77.1.89  (Debian)   │
        │  Elasticsearch (sole master) │
        │  transport CA lives here     │
        └──────────────────────────────┘
```

Elasticsearch HTTP is bound to `127.0.0.1` on both nodes. Nothing but WireGuard
(51820/udp) and Apache (80/443, es2 only) is reachable from the internet.

## Node roles

| | es1 | es2 |
|---|---|---|
| OS family | Debian (`apt`) | RHEL (`dnf`) |
| public IP | 187.77.1.89 | 31.97.136.158 |
| tunnel IP | 10.10.0.1 | 10.10.0.2 |
| ES roles | all (sole master-eligible) | `data`, `ingest`, `remote_cluster_client` |
| transport CA | **yes** — generates the CA and every node cert | no |
| Kibana / Apache | no | **yes** |
| firewall | ufw | firewalld |

Both distro families are live — every `when: ansible_facts['os_family'] == ...`
branch runs on exactly one host, so neither path is dead code.

Only es1 is master-eligible, deliberately. Two master-eligible nodes would need a
quorum of two, so losing either would stop the cluster; with one, es2 can restart
freely. The trade-off is that es1 is a single point of failure for the cluster,
and es2 is one for the website.

## Requirements

- `ansible-core >= 2.15` — that is when `ansible.builtin.deb822_repository` landed,
  which the elasticsearch role uses on the Debian host
- collections: `ansible.posix`, `community.general` (see `requirements.yml`)
- SSH access as `root` to both hosts (see `inventory/hosts.yml`)
- a `.vault_pass` file in the repo root (gitignored — never commit it)

```bash
conda env create -f environment.yml
conda activate haystack
```

`environment.yml` installs the `ansible` package rather than bare `ansible-core`,
which bundles both collections — so no galaxy step is needed. If you are working
from some other environment:

```bash
ansible-galaxy collection install -r requirements.yml
```

## DNS

`kibana_domain` and `es_domain` (in `group_vars/all/main.yml`) must resolve to
**es2** — `31.97.136.158` — *before* the apache role runs. certbot validates over
HTTP on port 80 against whatever the name resolves to; a stale A record fails at
`certbot certonly` with an unhelpful challenge error, after Apache is installed.

```bash
dig +short kibana.sundalei.cloud es.sundalei.cloud
```

## Secrets

`group_vars/all/vault.yml` is Ansible Vault encrypted and holds:

| variable | used by |
|---|---|
| `admin_password` | `common` — must be a **crypt hash**, not plaintext |
| `admin_ssh_public_key` | `common` |
| `vault_elastic_password` | `elasticsearch` — seeds `bootstrap.password`, then used for API calls |
| `vault_kibana_system_password` | `elasticsearch` (sets it) and `kibana` (uses it) |
| `vault_kibana_encryption_keys` | `kibana` — dict with `encryptedSavedObjects`, `reporting`, `security` |

```bash
ansible-vault view group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
```

`ansible.cfg` points `vault_password_file` at `.vault_pass`, so no `--ask-vault-pass`
is needed.

## Running

```bash
ansible-playbook site.yml
```

**Run against both hosts together.** `certs.yml` reads `hostvars[ca_host]` to
distribute certs, `wg0.conf.j2` reads each peer's public key, and the health gate
in `service.yml` waits for `number_of_nodes == 2`. `--limit` breaks all three.

Useful tags:

```bash
ansible-playbook site.yml --tags wireguard
ansible-playbook site.yml --tags es
ansible-playbook site.yml --tags kibana,apache
```

`--check` is of limited use here: `command` tasks don't support check mode, so
they report `skipped` regardless of real state.

### The idempotency check

Run twice. The second run should report `changed=0` on both hosts, with one known
exception: `common : Update apt cache (Debian)` on es1 re-runs whenever
`cache_valid_time` (1h) has expired. Anything else reporting `changed` on a
converged run is a bug.

## How the cluster bootstraps

This is the part that is easy to break, and it is worth understanding before
editing the elasticsearch role.

`cluster.initial_master_nodes` tells a node to **form a new cluster**. Any node
that has it and can satisfy its own quorum will bootstrap independently — and two
independently bootstrapped clusters have different UUIDs and can never be merged,
regardless of matching `cluster.name`. Recovery is destroying one node's
`path.data` and letting it rejoin.

So the template renders that setting only when **both** conditions hold:

```jinja
{% if es_cert_authority | default(false) and not es_bootstrapped %}
cluster.initial_master_nodes: ["{{ es_node_name }}"]
{% endif %}
```

- `es_cert_authority` — only es1 ever bootstraps. Every other node is a joiner and
  finds the cluster through `discovery.seed_hosts`.
- `es_bootstrapped` — set from the presence of `/etc/elasticsearch/.cluster_bootstrapped`,
  which `service.yml` writes **only after** `_cluster/health` reports the full node
  count. Gating on `status=green` alone is not enough: a single-node cluster reports
  green quite happily.

Discovery is not a network search. Nodes connect to the explicit addresses in
`discovery.seed_hosts`; `cluster.name` is validated during the handshake, not used
to find anything.

The packages also run security auto-configuration at install time, which writes its
own `initial_master_nodes` and PKCS#12 certs. Explicit `xpack.security.*.ssl.*`
settings in our template suppress it on a fresh host, but on a host where it already
ran, the leftovers (`http.p12`, `transport.p12`, `http_ca.crt`, and three
`*.secure_password` keystore entries) should be cleared.

## Rebuilding a node

If a node ends up in its own cluster, or you reinstall the VPS, use `reset-node.yml`
— it is deliberately not imported by `site.yml`:

```bash
ansible-playbook reset-node.yml -l es2 -e es_confirm_wipe=es2
ansible-playbook site.yml
```

`es_confirm_wipe` must equal the host being wiped, so a mistyped `--limit` cannot
take out the wrong box. It clears `path.data`, the bootstrap marker, and the
auto-configuration leftovers (`http.p12`, `transport.p12`, `http_ca.crt`, and the
three `*.secure_password` keystore entries), then recreates an empty data directory
with the right ownership. Your own `node.crt`, `node.key` and `ca.crt` are left
alone.

The playbook leaves the node **stopped**, on purpose. Starting it before `site.yml`
has rewritten `elasticsearch.yml` means it bootstraps from the stale config and you
are back where you started.

Wiping **es1** additionally requires `-e es_confirm_ca_wipe=true`. es1 is the only
bootstrap node, so resetting it destroys the cluster and every other node has to be
reset too. The CA itself (`/root/es-ca/`, `/root/es-certs.zip`) is never touched by
the playbook and survives — but losing that directory means regenerating every node
certificate, so it is the one thing on these boxes worth backing up.

## Checks

```bash
# cluster membership -- expect two rows, only es1 with 'm'
curl -s -u elastic:<pw> 'http://127.0.0.1:9200/_cat/nodes?v'

# same cluster UUID on both nodes, or they are not actually one cluster
curl -s -u elastic:<pw> 'http://127.0.0.1:9200/?pretty' | grep cluster_uuid

# kibana_system credential really works
curl -s -u kibana_system:<pw> 'http://127.0.0.1:9200/_security/_authenticate?pretty'

# public entry point -- 302 to the Kibana login is the healthy answer
curl -sI https://kibana.sundalei.cloud | head -1

# firewalls
ansible es1 -a 'ufw status verbose'
ansible es2 -a 'firewall-cmd --list-rich-rules'
```

## Linting

```bash
ansible-lint
```

The repo passes at the `production` profile. Keep it that way — and note that lint
cannot see every idempotency bug: `file` with `state: touch`, or a `command`
without `changed_when`, are both legal constructs that report `changed` forever.

## Known gaps

- `expose_elasticsearch: true` publishes the entire Elasticsearch API to the
  internet, protected only by the `elastic` password. Consider scoping the vhost
  or setting it to `false`.
- SELinux is disabled on es2, which now terminates public TLS.
- Neither node is redundant: es1 is the only master, es2 is the only web tier.
  That is inherent to two nodes and fine for this setup — just do not mistake it
  for high availability.
- `kibana.yml` is rendered with the `kibana_system` password in plaintext. Kibana
  ships a keystore (`kibana-keystore add elasticsearch.password`) that would keep
  it out of the config file.
- `roles/elasticsearch/tasks/config.yml` does not ensure `/var/lib/elasticsearch`
  exists; only `reset-node.yml` recreates it. A data directory removed by any other
  means will fail at startup until the reset playbook is run.
