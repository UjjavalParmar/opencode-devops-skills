---
name: network-debug
description: Debug Linux / Kubernetes networking at L3-L4 — connectivity, NetworkPolicy, kube-proxy / iptables / nftables, conntrack, MTU, packet loss, NAT, security groups. Use for "can't reach", timeouts, RST, intermittent drops. For DNS use dns-debug; for HTTP layer / Ingress use ingress-debug.
---

# Sockets & routes (host)
```
ss -tnp state established '( sport = :<P> or dport = :<P> )'
ss -tln
ip -br addr
ip route get <DST-IP>
ip neigh
```

# Reachability ladder
1. ICMP: `ping -c3 <DST>` (may be blocked — not authoritative)
2. TCP: `nc -vz -w3 <DST> <PORT>` or `curl -v telnet://<DST>:<PORT>`
3. Path: `mtr -rwzbc 50 <DST>` ; `traceroute -T -p <PORT> <DST>`

# Packet capture (least-invasive)
```
tcpdump -ni any -s0 -c 200 'host <DST> and port <PORT>' -w /tmp/cap.pcap
```
Read with `tshark -r /tmp/cap.pcap` or copy out.

# Kubernetes path
- Pod→Pod: `kubectl exec <SRC> -n <NS> -- nc -vz <DST-IP> <PORT>`
- Service: `kubectl get endpointslices -l kubernetes.io/service-name=<SVC> -n <NS>` — empty ⇒ selector mismatch
- NetworkPolicy: `kubectl get networkpolicy -A` ; default-deny isolates pod from all unless explicit allow
- kube-proxy mode: `kubectl -n kube-system logs ds/kube-proxy | head` ; iptables: `iptables-save | grep <SVC-IP>` ; ipvs: `ipvsadm -Ln`
- CNI: `kubectl -n kube-system logs ds/<cni> --tail=200`

# Symptom → likely cause
- **Connection refused** → nothing listening; wrong port; pod not Ready
- **Connection timeout** → SG/NetPol/firewall drop, or wrong route
- **RST mid-stream** → idle timeout (NLB 350s, ALB 60s default), conntrack drop
- **Intermittent drop** → conntrack table full: `sysctl net.netfilter.nf_conntrack_count nf_conntrack_max`
- **Works small, fails large** → MTU/PMTUD: `ping -M do -s 1472 <DST>`; lower MTU on overlay (VXLAN ~1450)

# AWS specifics
- SG: `aws ec2 describe-security-groups --group-ids <SG>`
- NACL (stateless, both directions)
- Reachability Analyzer: `aws ec2 create-network-insights-path ...`
- VPC Flow Logs filtered by `srcaddr/dstaddr/action=REJECT`

# Verify
Same `nc -vz` or `curl -v` that failed, now passes from the original source.
