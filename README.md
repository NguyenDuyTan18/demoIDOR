# One sandbox: four attackers and one server

This is one CyberRangeCZ Sandbox Definition. Create **one pool of size 1** from this repository root. The single sandbox contains four Kali attacker VMs, one Debian ServiceDesk server VM, and a separate gateway router.

| Node | IP | HTTP |
| --- | --- | --- |
| attacker1 | `10.80.0.11` | `http://10.80.0.11/` |
| attacker2 | `10.80.0.12` | `http://10.80.0.12/` |
| attacker3 | `10.80.0.13` | `http://10.80.0.13/` |
| attacker4 | `10.80.0.14` | `http://10.80.0.14/` |
| attacker1 ServiceDesk | `10.80.0.11:8080` | Independent team instance |
| attacker2 ServiceDesk | `10.80.0.12:8080` | Independent team instance |
| attacker3 ServiceDesk | `10.80.0.13:8080` | Independent team instance |
| attacker4 ServiceDesk | `10.80.0.14:8080` | Independent team instance |
| server | `10.80.0.20` | ServiceDesk on port 80 |
| gateway | `10.80.0.1` | Routing only |

**Traffic policy:** Each attacker can ping and open HTTP on the other three attackers. The server can ping and open HTTP on all four attackers. New connections from any attacker to `10.80.0.20` are blocked, including ping and HTTP. The server does not forward packets. The shared ServiceDesk remains on the server at `http://127.0.0.1/`; each attacker also runs a separate ServiceDesk instance on port 8080 with its own local SQLite database. Attacker VMs can reach one another, so these are separate datasets but not network-isolated from other teams inside the sandbox. HTTP access among the permitted machines uses port 80 without adding `:80` to the URL.

## Deploy

1. Put this directory at the root of a Git repository accessible to CyberRangeCZ. `topology.yml` and `provisioning/` must be at the repository root.
2. Create one Sandbox Definition from that Git URL and revision. Create one pool with **size 1**, then Allocate it. The provisioning playbook configures all five hosts and fails if the required connectivity policy does not work.
3. Import `training-definition.json`, review and release it, then create one cloud Training Instance and assign the pool. Open a run to see the five hosts in the topology. The attacker Linux login is `user-access / ClientLab-Only-2026!`.

The image and flavor names in `topology.yml` must exist in your cloud. The platform runner needs Ansible, the `user-access` role, and `ansible.posix` from `provisioning/requirements.yml`. The five host IPs and gateway subnet must not conflict with the platform's management network.

Each attacker VM runs an independent ServiceDesk instance on port 8080 with its own database. Teams log in with username `alice` and a team-specific password. Supply four distinct passwords (at least 16 characters) through the secure deployment environment variables `CYBERRANGE_WEBDESK_ATTACKER1_PASSWORD` through `CYBERRANGE_WEBDESK_ATTACKER4_PASSWORD`, or through an encrypted Ansible Vault/platform secret store. The shared ServiceDesk on `server` uses its own `CYBERRANGE_WEBDESK_SERVER_PASSWORD` if provided; otherwise, it uses team 1's password (the server is isolated from attacker VMs by the network policy). Do not commit passwords to Git. The playbook validates all four team passwords before it changes any hosts, without printing the values.

## Verify inside the sandbox

On each attacker, its own ServiceDesk instance is available at `http://127.0.0.1:8080/` and reports a host-specific sandbox identity at `/health`. The four team instances use separate local databases. On attacker1, these must succeed:

```bash
ping -c 1 10.80.0.12
ping -c 1 10.80.0.13
ping -c 1 10.80.0.14
curl --noproxy '*' -fsS http://10.80.0.12/health
curl --noproxy '*' -fsS http://10.80.0.13/health
curl --noproxy '*' -fsS http://10.80.0.14/health
```

These must fail from every attacker:

```bash
ping -c 1 -W 2 10.80.0.20
curl --noproxy '*' --connect-timeout 3 http://10.80.0.20/
```

On the server, `curl --noproxy '*' http://127.0.0.1/health` must succeed, and `sudo nft list table inet idor_server_acl` must show the rule blocking new attacker connections. The server can also ping and curl all four attacker addresses.

The server ACL has its own nftables table and systemd unit; it does not replace other firewall tables. The provisioning playbook tests all permitted and blocked paths before it finishes. A private `10.x.x.x` URL is accessible only from the lab network or an approved tunnel.

## Local source checks

```bash
python3 -m pip install -r requirements-dev.txt
python3 -m unittest discover -s tests -v
```
