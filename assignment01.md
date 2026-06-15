---


---

<blockquote>
<h1 id="srt210-assignment-1">SRT210 Assignment 1</h1>
</blockquote>
<p><strong>Student Name:</strong> Arian Nili<br>
<strong>Student ID:</strong> 173189226<br>
<strong>Date:</strong> 06/15/2026<br>
<strong>Environment:</strong> VirtualBox 7.2.8, Windows Server 2022, Ubuntu 26.04 LTS</p>
<hr>
<h2 id="windows-network-topology">1. Windows Network Topology</h2>

<table>
<thead>
<tr>
<th>Segment</th>
<th>Subnet</th>
<th>Mask</th>
<th>Usable addresses</th>
</tr>
</thead>
<tbody>
<tr>
<td>Gateway ↔ Relay</td>
<td><code>192.168.76.0/29</code></td>
<td>255.255.255.248</td>
<td><code>.1</code> – <code>.6</code></td>
</tr>
<tr>
<td>Relay ↔ Node</td>
<td><code>192.168.76.80/28</code></td>
<td>255.255.255.240</td>
<td><code>.81</code> – <code>.94</code></td>
</tr>
</tbody>
</table>
<table>
<thead>
<tr>
<th>VM</th>
<th>Interface role</th>
<th>IP address</th>
<th>How assigned</th>
</tr>
</thead>
<tbody>
<tr>
<td>windows-GW</td>
<td>NAT (to internet)</td>
<td><code>10.0.2.15</code></td>
<td>VirtualBox DHCP</td>
</tr>
<tr>
<td></td>
<td>Internal (to Relay)</td>
<td><code>192.168.76.1/29</code></td>
<td>Static (manual)</td>
</tr>
<tr>
<td>windows-Relay</td>
<td>Internal (to Gateway)</td>
<td><code>192.168.76.2/29</code></td>
<td>DHCP from GW (MAC reservation)</td>
</tr>
<tr>
<td></td>
<td>Bridged (to Node)</td>
<td><code>192.168.76.81/28</code></td>
<td>Static</td>
</tr>
<tr>
<td>windows-Node</td>
<td>Bridged (to Relay)</td>
<td><code>192.168.76.82/28</code></td>
<td>DHCP from Relay (MAC reserv.)</td>
</tr>
</tbody>
</table><p><strong>Physical connections:</strong></p>
<ul>
<li>Relay ↔ Node: Realtek USB GbE Family Controller adapter connected by a physical CAT6 cable.</li>
<li>Host‑only adapters (Adapter 4 on GW) used for SSH from the K1001 PC.</li>
</ul>
<hr>
<h2 id="windows-server-2022-routing-lab">2. Windows Server 2022 Routing Lab</h2>
<h3 id="virtual-machine-creation">2.1 Virtual Machine Creation</h3>

<table>
<thead>
<tr>
<th>VM</th>
<th>OS Edition</th>
<th>RAM</th>
<th>CPUs</th>
<th>Disk</th>
<th>Cloned from</th>
</tr>
</thead>
<tbody>
<tr>
<td>windows-GW</td>
<td>Server 2022 Standard Evaluation (Core)</td>
<td>4 GB</td>
<td>2</td>
<td>50 GB</td>
<td>Fresh install</td>
</tr>
<tr>
<td>windows-Relay</td>
<td>Server 2022 Standard Evaluation (Core)</td>
<td>4 GB</td>
<td>2</td>
<td>50 GB</td>
<td>Full clone of GW, <strong>generate new MACs</strong></td>
</tr>
<tr>
<td>windows-Node</td>
<td>Server 2022 Standard Evaluation (Desktop Exp.)</td>
<td>8 GB</td>
<td>2</td>
<td>50 GB</td>
<td>Fresh install</td>
</tr>
</tbody>
</table><p><strong>Critical clone step:</strong> After cloning windows‑Relay, right‑click → Clone → “Generate new MAC addresses for all network adapters”. Otherwise MAC conflicts break networking.</p>
<h3 id="virtual-network-adapter-configuration-all-vms-powered-off">2.2 Virtual Network Adapter Configuration (all VMs powered off)</h3>
<h4 id="windows‑gw-gateway">windows‑GW (Gateway)</h4>

<table>
<thead>
<tr>
<th>Adapter</th>
<th>Attached to</th>
<th>Name / Promiscuous</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>NAT</td>
<td>-</td>
<td>internet out</td>
</tr>
<tr>
<td>2</td>
<td>Internal Network</td>
<td><code>intnet</code></td>
<td>to Relay</td>
</tr>
<tr>
<td>3</td>
<td>Disabled</td>
<td>-</td>
<td>-</td>
</tr>
<tr>
<td>4</td>
<td>Host‑Only Adapter</td>
<td>VirtualBox Host‑Only</td>
<td>for SSH from K1001 PC</td>
</tr>
</tbody>
</table><h4 id="windows‑relay">windows‑Relay</h4>

<table>
<thead>
<tr>
<th>Adapter</th>
<th>Attached to</th>
<th>Name / Promiscuous</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>Internal Network</td>
<td><code>intnet</code></td>
<td>to Gateway</td>
</tr>
<tr>
<td>2</td>
<td>Bridged</td>
<td>Realtek USB GbE Family Controller, Allow All</td>
<td>to Node (physical cable)</td>
</tr>
</tbody>
</table><h4 id="windows‑node">windows‑Node</h4>

<table>
<thead>
<tr>
<th>Adapter</th>
<th>Attached to</th>
<th>Name / Promiscuous</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>Bridged</td>
<td>Realtek USB GbE Family Controller #2, Allow All</td>
<td>to Relay (physical cable)</td>
</tr>
</tbody>
</table><blockquote>
<p><strong>Promiscuous Mode = Allow All</strong> is mandatory on bridged adapters of Relay and Node because the Relay acts as a router and must forward frames not destined to its own MAC.</p>
</blockquote>
<hr>
<h3 id="gateway-windows‑gw-configuration">2.3 Gateway (windows‑GW) Configuration</h3>
<p>All commands run as Administrator in PowerShell.</p>
<h4 id="step-gw‑1-–-identify-interfaces">Step GW‑1 – Identify interfaces</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Get<span class="token operator">-</span>NetIPConfiguration
</code></pre>
<ul>
<li>NAT Adapter shows <code>10.0.2.15</code> (internet)</li>
<li>Internal adapter shows <code>169.254.x.x</code> (no DHCP yet) – note its name, e.g. <code>"Ethernet 2"</code></li>
</ul>
<h4 id="step-gw‑2-–-internet-confirmation">Step GW‑2 – Internet confirmation</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">ping 1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1
</code></pre>
<p>Must get replies.</p>
<h4 id="step-gw‑3-–-static-ip-on-internal-interface">Step GW‑3 – Static IP on Internal interface</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">New<span class="token operator">-</span>NetIPAddress <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet 2"</span> 
<span class="token operator">-</span>IPAddress 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>1 <span class="token operator">-</span>PrefixLength 29
</code></pre>
<p>Verify: <code>Get-NetIPConfiguration -InterfaceAlias "Ethernet 2"</code></p>
<h4 id="step-gw‑4-–-enable-ip-forwarding-registry">Step GW‑4 – Enable IP forwarding (registry)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set-ItemProperty</span> <span class="token operator">-</span>Path <span class="token string">"HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\
Parameters"</span> <span class="token operator">-</span>Name IPEnableRouter <span class="token operator">-</span>Value 1
</code></pre>
<p><strong>Staged – requires reboot later.</strong></p>
<h4 id="step-gw‑5-–-add-return-route-to-node-network-most-commonly-missed">Step GW‑5 – Add return route to Node network (most commonly missed)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">New<span class="token operator">-</span>NetRoute <span class="token operator">-</span>DestinationPrefix 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>80<span class="token operator">/</span>28 
<span class="token operator">-</span>NextHop 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>2 <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet 2"</span>
</code></pre>
<p>Without this, replies from internet to Node are dropped at Gateway.</p>
<h4 id="step-gw‑6-–-install-rras-routing-role">Step GW‑6 – Install RRAS (routing role)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Install<span class="token operator">-</span>WindowsFeature <span class="token operator">-</span>Name Routing <span class="token operator">-</span>IncludeManagementTools
Install<span class="token operator">-</span>RemoteAccess <span class="token operator">-</span>VpnType RoutingOnly   <span class="token comment"># May require reboot first – see Troubleshooting</span>
</code></pre>
<h4 id="step-gw‑7-–-reboot">Step GW‑7 – Reboot</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Restart-Computer</span>
</code></pre>
<p>After reboot, verify forwarding:</p>
<pre class=" language-powershell"><code class="prism  language-powershell">Get<span class="token operator">-</span>NetIPInterface <span class="token punctuation">|</span> <span class="token function">Select-Object</span> InterfaceAlias<span class="token punctuation">,</span> AddressFamily<span class="token punctuation">,</span> Forwarding
</code></pre>
<p>Both IPv4 entries must show <code>Enabled</code>.</p>
<h4 id="step-gw‑8-–-create-nat-translation">Step GW‑8 – Create NAT translation</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">New<span class="token operator">-</span>NetNat <span class="token operator">-</span>Name LabNAT <span class="token operator">-</span>InternalIPInterfaceAddressPrefix 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>0<span class="token operator">/</span>24
Get<span class="token operator">-</span>NetNat   <span class="token comment"># should show Active = True</span>
</code></pre>
<h4 id="step-gw‑9-–-firewall-rules-allow-essential-traffic">Step GW‑9 – Firewall rules (allow essential traffic)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token comment"># ICMP only from internal network (Lab 4 requirement)</span>
New<span class="token operator">-</span>NetFirewallRule <span class="token operator">-</span>Name <span class="token string">"Allow-ICMPv4-Internal"</span> <span class="token operator">-</span>DisplayName <span class="token string">"Allow ICMPv4 from Internal Only"</span> <span class="token operator">-</span>Protocol ICMPv4 <span class="token operator">-</span>IcmpType 8 <span class="token operator">-</span>Direction Inbound <span class="token operator">-</span>Action Allow <span class="token operator">-</span>RemoteAddress 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>0<span class="token operator">/</span>24
<span class="token comment"># DHCP server inbound</span>
New<span class="token operator">-</span>NetFirewallRule <span class="token operator">-</span>Name <span class="token string">"Allow-DHCP-In"</span> <span class="token operator">-</span>DisplayName <span class="token string">"Allow DHCP Inbound"</span> <span class="token operator">-</span>Protocol UDP <span class="token operator">-</span>LocalPort 67 <span class="token operator">-</span>Direction Inbound <span class="token operator">-</span>Action Allow
<span class="token comment"># SSH from anywhere (school PC)</span>
New<span class="token operator">-</span>NetFirewallRule <span class="token operator">-</span>Name <span class="token string">"Allow-SSH-In"</span> <span class="token operator">-</span>DisplayName <span class="token string">"Allow SSH Inbound"</span> <span class="token operator">-</span>Protocol TCP <span class="token operator">-</span>LocalPort 22 <span class="token operator">-</span>Direction Inbound <span class="token operator">-</span>Action Allow
<span class="token comment"># Full internal trust (Relay can talk freely)</span>
New<span class="token operator">-</span>NetFirewallRule <span class="token operator">-</span>Name <span class="token string">"Allow-Internal-In"</span> <span class="token operator">-</span>DisplayName <span class="token string">"Allow Internal Network"</span> <span class="token operator">-</span>Direction Inbound <span class="token operator">-</span>Action Allow <span class="token operator">-</span>RemoteAddress 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>0<span class="token operator">/</span>24
</code></pre>
<h4 id="step-gw‑10-–-set-default-inbound-policy-to-block-lab-4-strict-mode">Step GW‑10 – Set default inbound policy to Block (Lab 4 strict mode)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set</span><span class="token operator">-</span>NetFirewallProfile <span class="token operator">-</span>Profile Domain<span class="token punctuation">,</span>Public<span class="token punctuation">,</span>Private <span class="token operator">-</span>DefaultInboundAction Block
</code></pre>
<h4 id="step-gw‑11-–-install-dhcp-server-role">Step GW‑11 – Install DHCP Server role</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Install<span class="token operator">-</span>WindowsFeature <span class="token operator">-</span>Name DHCP <span class="token operator">-</span>IncludeManagementTools
</code></pre>
<h4 id="step-gw‑12-–-disable-rogue-detection-non‑domain-fix">Step GW‑12 – Disable rogue detection (non‑domain fix)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set-ItemProperty</span> <span class="token operator">-</span>Path <span class="token string">"HKLM:\SYSTEM\CurrentControlSet\Services\DHCPServer\Parameters"</span> <span class="token operator">-</span>Name <span class="token string">"DisableRogueDetection"</span> <span class="token operator">-</span>Value 1 <span class="token operator">-</span><span class="token function">Type</span> DWord
<span class="token function">Restart-Service</span> DHCPServer
Get<span class="token operator">-</span>DhcpServerSetting   <span class="token comment"># must show IsAuthorized = True</span>
</code></pre>
<h4 id="step-gw‑13-–-create-dhcp-scope-for-gateway–relay-link">Step GW‑13 – Create DHCP scope for Gateway–Relay link</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Add<span class="token operator">-</span>DhcpServerv4Scope <span class="token operator">-</span>Name <span class="token string">"GW-to-Relay"</span> <span class="token operator">-</span>StartRange 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>2 <span class="token operator">-</span>EndRange 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>6 <span class="token operator">-</span>SubnetMask 255<span class="token punctuation">.</span>255<span class="token punctuation">.</span>255<span class="token punctuation">.</span>248 <span class="token operator">-</span>State Active
</code></pre>
<h4 id="step-gw‑14-–-mac-reservation-for-relay’s-internal-adapter">Step GW‑14 – MAC reservation for Relay’s Internal adapter</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Add<span class="token operator">-</span>DhcpServerv4Reservation <span class="token operator">-</span>ScopeId 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>0 <span class="token operator">-</span>IPAddress 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>2 <span class="token operator">-</span>ClientId <span class="token string">"08-00-27-68-02-92"</span> <span class="token operator">-</span>Description <span class="token string">"Relay Internal"</span>
</code></pre>
<p>Replace MAC with actual address from VirtualBox → Relay → Adapter 2 → Advanced.</p>
<h4 id="step-gw‑15-–-scope-options-default-gateway-dns">Step GW‑15 – Scope options (default gateway, DNS)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set</span><span class="token operator">-</span>DhcpServerv4OptionValue <span class="token operator">-</span>ScopeId 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>0 <span class="token operator">-</span>Router 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>1 <span class="token operator">-</span>DnsServer 1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1
</code></pre>
<h4 id="step-gw‑16-–-restart-dhcp-service">Step GW‑16 – Restart DHCP service</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Restart-Service</span> DHCPServer
<span class="token function">Get-Service</span> DHCPServer   <span class="token comment"># Running</span>
</code></pre>
<h4 id="step-gw‑17-–-install-openssh-server">Step GW‑17 – Install OpenSSH server</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Add<span class="token operator">-</span>WindowsCapability <span class="token operator">-</span>Online <span class="token operator">-</span>Name OpenSSH<span class="token punctuation">.</span>Server~~~~0<span class="token punctuation">.</span>0<span class="token punctuation">.</span>1<span class="token punctuation">.</span>0
<span class="token function">Start-Service</span> sshd
<span class="token function">Set-Service</span> <span class="token operator">-</span>Name sshd <span class="token operator">-</span>StartupType Automatic
</code></pre>
<hr>
<h3 id="relay-windows‑relay-configuration">2.4 Relay (windows‑Relay) Configuration</h3>
<h4 id="step-r‑1-–-identify-interfaces-by-mac">Step R‑1 – Identify interfaces by MAC</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Get<span class="token operator">-</span>NetAdapter
</code></pre>
<p>Match MACs with VirtualBox settings:</p>
<ul>
<li>
<p>Adapter 2 (Internal) → <code>"Ethernet 2"</code></p>
</li>
<li>
<p>Adapter 3 (Bridged) → <code>"Ethernet 3"</code></p>
</li>
</ul>
<h4 id="step-r‑2-–-static-ip-on-bridged-interface-facing-node">Step R‑2 – Static IP on Bridged interface (facing Node)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">New<span class="token operator">-</span>NetIPAddress <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet 3"</span> <span class="token operator">-</span>IPAddress 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>81 <span class="token operator">-</span>PrefixLength 28
</code></pre>
<h4 id="step-r‑3-–-enable-ip-forwarding-same-registry-key-as-gw">Step R‑3 – Enable IP forwarding (same registry key as GW)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set-ItemProperty</span> <span class="token operator">-</span>Path <span class="token string">"HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters"</span> <span class="token operator">-</span>Name IPEnableRouter <span class="token operator">-</span>Value 1
</code></pre>
<h4 id="step-r‑4-–-allow-icmp-inbound-for-testing">Step R‑4 – Allow ICMP inbound (for testing)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">New<span class="token operator">-</span>NetFirewallRule <span class="token operator">-</span>Name <span class="token string">"Allow-ICMPv4-In"</span> <span class="token operator">-</span>DisplayName <span class="token string">"Allow ICMPv4 Ping"</span> <span class="token operator">-</span>Protocol ICMPv4 <span class="token operator">-</span>IcmpType 8 <span class="token operator">-</span>Direction Inbound <span class="token operator">-</span>Action Allow
</code></pre>
<h4 id="step-r‑5-–-reboot">Step R‑5 – Reboot</h4>
<pre><code>powershell
Restart-Computer
</code></pre>
<p>After reboot, verify forwarding and test internet:</p>
<pre class=" language-powershell"><code class="prism  language-powershell">Get<span class="token operator">-</span>NetIPInterface <span class="token punctuation">|</span> <span class="token function">Select-Object</span> InterfaceAlias<span class="token punctuation">,</span> AddressFamily<span class="token punctuation">,</span> Forwarding
ping 1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1   <span class="token comment"># must succeed – this confirms Gateway NAT works</span>
</code></pre>
<h4 id="step-r‑6-–-install-dhcp-server-role">Step R‑6 – Install DHCP Server role</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Install<span class="token operator">-</span>WindowsFeature <span class="token operator">-</span>Name DHCP <span class="token operator">-</span>IncludeManagementTools
</code></pre>
<h4 id="step-r‑7-–-disable-rogue-detection-again-non‑domain">Step R‑7 – Disable rogue detection (again, non‑domain)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set-ItemProperty</span> <span class="token operator">-</span>Path <span class="token string">"HKLM:\SYSTEM\CurrentControlSet\Services\DHCPServer\Parameters"</span> <span class="token operator">-</span>Name <span class="token string">"DisableRogueDetection"</span> <span class="token operator">-</span>Value 1 <span class="token operator">-</span><span class="token function">Type</span> DWord
<span class="token function">Restart-Service</span> DHCPServer
Get<span class="token operator">-</span>DhcpServerSetting   <span class="token comment"># True</span>
</code></pre>
<h4 id="step-r‑8-–-create-dhcp-scope-for-relay–node-segment">Step R‑8 – Create DHCP scope for Relay–Node segment</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Add<span class="token operator">-</span>DhcpServerv4Scope <span class="token operator">-</span>Name <span class="token string">"Relay-to-Node"</span> <span class="token operator">-</span>StartRange 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>82 <span class="token operator">-</span>EndRange 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>94 <span class="token operator">-</span>SubnetMask 255<span class="token punctuation">.</span>255<span class="token punctuation">.</span>255<span class="token punctuation">.</span>240 <span class="token operator">-</span>State Active
</code></pre>
<h4 id="step-r‑9-–-mac-reservation-for-node">Step R‑9 – MAC reservation for Node</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Add<span class="token operator">-</span>DhcpServerv4Reservation <span class="token operator">-</span>ScopeId 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>80 <span class="token operator">-</span>IPAddress 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>82 <span class="token operator">-</span>ClientId <span class="token string">"08-00-27-59-D8-F3"</span> <span class="token operator">-</span>Description <span class="token string">"Node"</span>
</code></pre>
<p>Replace MAC with Node’s Adapter 3 MAC.</p>
<h4 id="step-r‑10-–-scope-options-gateway--relay-itself-dns--1.1.1.1">Step R‑10 – Scope options (gateway = Relay itself, DNS = 1.1.1.1)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set</span><span class="token operator">-</span>DhcpServerv4OptionValue <span class="token operator">-</span>ScopeId 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>80 <span class="token operator">-</span>Router 192<span class="token punctuation">.</span>168<span class="token punctuation">.</span>76<span class="token punctuation">.</span>81 <span class="token operator">-</span>DnsServer 1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1<span class="token punctuation">.</span>1
</code></pre>
<h4 id="step-r‑11-–-restart-dhcp-service">Step R‑11 – Restart DHCP service</h4>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Restart-Service</span> DHCPServer
</code></pre>
<h4 id="step-r‑12-–-switch-internal-interface-to-dhcp-client-critical-order">Step R‑12 – Switch Internal interface to DHCP client (critical order)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Remove<span class="token operator">-</span>NetIPAddress <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet 2"</span> <span class="token operator">-</span>Confirm:<span class="token boolean">$false</span>
Remove<span class="token operator">-</span>NetRoute <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet 2"</span> <span class="token operator">-</span>DestinationPrefix 0<span class="token punctuation">.</span>0<span class="token punctuation">.</span>0<span class="token punctuation">.</span>0<span class="token operator">/</span>0 <span class="token operator">-</span>Confirm:<span class="token boolean">$false</span>
<span class="token function">Set</span><span class="token operator">-</span>NetIPInterface <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet 2"</span> <span class="token operator">-</span>Dhcp Enabled
ipconfig <span class="token operator">/</span>release <span class="token string">"Ethernet 2"</span>
ipconfig <span class="token operator">/</span>renew <span class="token string">"Ethernet 2"</span>
Get<span class="token operator">-</span>NetIPConfiguration <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet 2"</span>
</code></pre>
<p>Must show <code>IPv4Address 192.168.76.2</code>, <code>IPv4DefaultGateway 192.168.76.1</code>.</p>
<h4 id="step-r‑13-–-install-sshd-same-as-gw">Step R‑13 – Install sshd (same as GW)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Add<span class="token operator">-</span>WindowsCapability <span class="token operator">-</span>Online <span class="token operator">-</span>Name OpenSSH<span class="token punctuation">.</span>Server~~~~0<span class="token punctuation">.</span>0<span class="token punctuation">.</span>1<span class="token punctuation">.</span>0
<span class="token function">Start-Service</span> sshd
</code></pre>
<h2 id="set-service--name-sshd--startuptype-automatic">Set-Service -Name sshd -StartupType Automatic</h2>
<h3 id="node-windows‑node-configuration">2.5 Node (windows‑Node) Configuration</h3>
<p>Node has Desktop Experience (GUI). Use PowerShell as Administrator.</p>
<h4 id="step-n‑1-–-identify-interface">Step N‑1 – Identify interface</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Get<span class="token operator">-</span>NetAdapter
</code></pre>
<p>Only one enabled (Bridged). Note name, e.g., <code>"Ethernet"</code>.</p>
<h4 id="step-n‑2-–-switch-to-dhcp-client">Step N‑2 – Switch to DHCP client</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">Remove<span class="token operator">-</span>NetIPAddress <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet"</span> <span class="token operator">-</span>Confirm:<span class="token boolean">$false</span>
Remove<span class="token operator">-</span>NetRoute <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet"</span> <span class="token operator">-</span>DestinationPrefix 0<span class="token punctuation">.</span>0<span class="token punctuation">.</span>0<span class="token punctuation">.</span>0<span class="token operator">/</span>0 <span class="token operator">-</span>Confirm:<span class="token boolean">$false</span>
<span class="token function">Set</span><span class="token operator">-</span>NetIPInterface <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet"</span> <span class="token operator">-</span>Dhcp Enabled
ipconfig <span class="token operator">/</span>renew <span class="token string">"Ethernet"</span>
</code></pre>
<h4 id="step-n‑3-–-verify-ip-configuration">Step N‑3 – Verify IP configuration</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">ipconfig
</code></pre>
<p>Expected: <code>192.168.76.82</code>, mask <code>255.255.255.240</code>, gateway <code>192.168.76.81</code>.</p>
<h4 id="step-n‑4-–-allow-icmp-inbound-for-testing">Step N‑4 – Allow ICMP inbound (for testing)</h4>
<pre class=" language-powershell"><code class="prism  language-powershell">New<span class="token operator">-</span>NetFirewallRule <span class="token operator">-</span>Name <span class="token string">"Allow-ICMPv4-In"</span> <span class="token operator">-</span>DisplayName <span class="token string">"Allow ICMPv4 Ping"</span> <span class="token operator">-</span>Protocol ICMPv4 <span class="token operator">-</span>IcmpType 8 <span class="token operator">-</span>Direction Inbound <span class="token operator">-</span>Action Allow
</code></pre>
<hr>
<h3 id="verification-run-from-node">2.6 Verification (run from Node)</h3>
<pre class=" language-cmd"><code class="prism  language-cmd">
ping 192.168.76.81   # Relay
ping 192.168.76.1    # Gateway
ping 1.1.1.1         # Internet
ping google.com      # DNS
tracert 1.1.1.1      # Should show Relay → Gateway → 10.0.2.2 → ...
</code></pre>
<p><strong>Hop‑by‑hop reasoning:</strong></p>
<ul>
<li>
<p>If ping to Relay fails → physical cable or bridged promiscuous mode.</p>
</li>
<li>
<p>If ping to Gateway fails → Relay forwarding not enabled or Gateway return route missing.</p>
</li>
<li>
<p>If ping to 1.1.1.1 fails but Gateway works → NAT missing on Gateway (<code>Get-NetNat</code>).</p>
</li>
</ul>
<hr>
<h3 id="lab-4-firewall-strict-mode-verification">2.7 Lab 4 Firewall Strict Mode Verification</h3>
<p><strong>From school PC (host) – Host‑Only network <code>192.168.56.x</code>:</strong></p>
<ul>
<li>
<p><code>ping 192.168.56.x</code> → <strong>timed out</strong> (ICMP blocked by default).</p>
</li>
<li>
<p><code>ssh Administrator@192.168.56.x</code> → <strong>successful login</strong> (TCP/22 allowed).</p>
</li>
</ul>
<p><strong>From Node (<code>192.168.76.82</code>):</strong></p>
<ul>
<li><code>ping 192.168.76.1</code> → <strong>success</strong> (internal network fully allowed).</li>
</ul>
<p><strong>Toggle commands on Gateway (for demo):</strong></p>
<pre class=" language-powershell"><code class="prism  language-powershell">
<span class="token function">Set</span><span class="token operator">-</span>NetFirewallProfile <span class="token operator">-</span>Profile Domain<span class="token punctuation">,</span>Public<span class="token punctuation">,</span>Private <span class="token operator">-</span>DefaultInboundAction Block   <span class="token comment"># Strict mode</span>
<span class="token function">Set</span><span class="token operator">-</span>NetFirewallProfile <span class="token operator">-</span>Profile Domain<span class="token punctuation">,</span>Public<span class="token punctuation">,</span>Private <span class="token operator">-</span>DefaultInboundAction Allow  <span class="token comment"># Open mode</span>
Get<span class="token operator">-</span>NetFirewallProfile <span class="token punctuation">|</span> <span class="token function">Select-Object</span> Name<span class="token punctuation">,</span> DefaultInboundAction
</code></pre>
<hr>
<h2 id="ubuntu-routing-lab-ubuntu-26.04-–-labs-3--4">3. Ubuntu Routing Lab (Ubuntu 26.04) – Labs 3 &amp; 4</h2>
<p>Parallel topology with identical IP scheme (Y=76). Three VMs: <code>ubuntu-gw</code>, <code>ubuntu-relay</code>, <code>ubuntu-node</code>. All Ubuntu Server 26.04 LTS (no GUI).</p>
<h3 id="network-adapter-mapping-virtualbox">3.1 Network Adapter Mapping (VirtualBox)</h3>
<h4 id="ubuntu‑gw-gateway">ubuntu‑GW (Gateway)</h4>

<table>
<thead>
<tr>
<th>Adapter</th>
<th>Attached to</th>
<th>Name / Promiscuous</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>NAT</td>
<td>-</td>
<td>internet out</td>
</tr>
<tr>
<td>2</td>
<td>Internal Network</td>
<td><code>intnet</code></td>
<td>to Relay</td>
</tr>
<tr>
<td>3</td>
<td>Disabled</td>
<td>-</td>
<td>-</td>
</tr>
<tr>
<td>4</td>
<td>Host‑Only Adapter</td>
<td>VirtualBox Host‑Only</td>
<td>for SSH from K1001 PC</td>
</tr>
</tbody>
</table><h4 id="ubuntu‑relay">ubuntu‑Relay</h4>

<table>
<thead>
<tr>
<th>Adapter</th>
<th>Attached to</th>
<th>Name / Promiscuous</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>Internal Network</td>
<td><code>intnet</code></td>
<td>to Gateway</td>
</tr>
<tr>
<td>2</td>
<td>Bridged</td>
<td>Realtek USB GbE Family Controller, Allow All</td>
<td>to Node (physical cable)</td>
</tr>
</tbody>
</table><h4 id="ubuntu‑node">ubuntu‑Node</h4>

<table>
<thead>
<tr>
<th>Adapter</th>
<th>Attached to</th>
<th>Name / Promiscuous</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td>1</td>
<td>Bridged</td>
<td>Realtek USB GbE Family Controller #2, Allow All</td>
<td>to Relay (physical cable)</td>
</tr>
</tbody>
</table><h3 id="gateway-ubuntu-gw-configuration">3.2 Gateway (ubuntu-gw) Configuration</h3>
<h4 id="set-static-ip-on-internal-interface">Set static IP on internal interface</h4>
<p>Edit <code>/etc/netplan/00-installer-config.yaml</code>:</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">network</span><span class="token punctuation">:</span>
 <span class="token key atrule">version</span><span class="token punctuation">:</span> <span class="token number">2</span>
 <span class="token key atrule">ethernets</span><span class="token punctuation">:</span>
 <span class="token key atrule">enp0s3</span><span class="token punctuation">:</span>   <span class="token comment"># NAT – leave as DHCP</span>
 <span class="token key atrule">dhcp4</span><span class="token punctuation">:</span> <span class="token boolean important">true</span>
 <span class="token key atrule">enp0s8</span><span class="token punctuation">:</span>   <span class="token comment"># Internal</span>
 <span class="token key atrule">dhcp4</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
 <span class="token key atrule">addresses</span><span class="token punctuation">:</span>
 <span class="token punctuation">-</span> 192.168.76.1/29
 <span class="token key atrule">nameservers</span><span class="token punctuation">:</span>
 <span class="token key atrule">addresses</span><span class="token punctuation">:</span> <span class="token punctuation">[</span>1.1.1.1<span class="token punctuation">]</span>
</code></pre>
<p>Apply: <code>sudo netplan apply</code></p>
<h4 id="enable-ip-forwarding-permanent">Enable IP forwarding (permanent)</h4>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token keyword">echo</span> <span class="token string">"net.ipv4.ip_forward=1"</span> <span class="token operator">|</span> <span class="token function">sudo</span> <span class="token function">tee</span> -a /etc/sysctl.conf
<span class="token function">sudo</span> sysctl -p
</code></pre>
<h4 id="add-return-route-to-node-network">Add return route to Node network</h4>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ip route add 192.168.76.80/28 via 192.168.76.2 dev enp0s8
</code></pre>
<p>Make persistent: create <code>/etc/systemd/network/10-return-route.network</code> or add to <code>/etc/rc.local</code>.</p>
<h4 id="install-and-configure-nat-with-iptables">Install and configure NAT with iptables</h4>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> apt update
<span class="token function">sudo</span> apt <span class="token function">install</span> iptables-persistent
<span class="token function">sudo</span> iptables -t nat -A POSTROUTING -s 192.168.76.0/24 -o enp0s3 -j MASQUERADE
<span class="token function">sudo</span> netfilter-persistent save
</code></pre>
<h4 id="install-kea-dhcp-server">Install Kea DHCP server</h4>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> apt update <span class="token operator">&amp;&amp;</span> <span class="token function">sudo</span> apt upgrade -y
<span class="token function">sudo</span> apt <span class="token function">install</span> kea-dhcp4-server -y
</code></pre>
<p>Edit `/etc/kea/kea-dhcp4.conf:</p>
<pre class=" language-text"><code class="prism  language-text">{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp0s8" ]
    },
    "lease-database": {
      "type": "memfile",
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/tmp/kea-dhcp4-ctrl.sock"
    },
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.76.0/29",
        "pools": [
          {
            "pool": "192.168.76.2 - 192.168.76.6"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.76.1"
          },
          {
            "name": "domain-name-servers",
            "data": "1.1.1.1"
          }
        ],
        "reservations": [
          {
            "hw-address": "08:00:27:68:02:92",
            "ip-address": "192.168.76.2"
          }
        ]
      }
    ],
    "loggers": [
      {
        "name": "kea-dhcp4",
        "severity": "INFO",
        "output_options": [
          {
            "output": "stdout"
          }
        ]
      }
    ],
    "ddns-send-updates": false,
    "ddns-override-client-update": false,
    "ddns-override-no-update": false
  }
}
</code></pre>
<h4 id="firewall-ufw-–-lab-4-strict-mode">Firewall (ufw) – Lab 4 strict mode</h4>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ufw default deny incoming
<span class="token function">sudo</span> ufw default allow outgoing
<span class="token function">sudo</span> ufw allow from 192.168.76.0/24 to any
<span class="token function">sudo</span> ufw allow 22/tcp   <span class="token comment"># SSH from K1001 PC</span>
<span class="token function">sudo</span> ufw <span class="token function">enable</span>
</code></pre>
<h4 id="enable-ssh">Enable SSH</h4>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> apt <span class="token function">install</span> openssh-server
<span class="token function">sudo</span> systemctl <span class="token function">enable</span> <span class="token function">ssh</span> --now
</code></pre>
<h3 id="relay-ubuntu-relay-configuration">3.3 Relay (ubuntu-relay) Configuration</h3>
<h4 id="static-ip-on-bridged-interface-enp0s10">Static IP on bridged interface (enp0s10)</h4>
<p>Netplan configuration:</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">network</span><span class="token punctuation">:</span>
 <span class="token key atrule">version</span><span class="token punctuation">:</span> <span class="token number">2</span>
 <span class="token key atrule">ethernets</span><span class="token punctuation">:</span>
 <span class="token key atrule">enp0s8</span><span class="token punctuation">:</span>   <span class="token comment"># Internal – will get DHCP</span>
 <span class="token key atrule">dhcp4</span><span class="token punctuation">:</span> <span class="token boolean important">true</span>
 <span class="token key atrule">enp0s10</span><span class="token punctuation">:</span>  <span class="token comment"># Bridged to Node</span>
 <span class="token key atrule">dhcp4</span><span class="token punctuation">:</span> <span class="token boolean important">false</span>
 <span class="token key atrule">addresses</span><span class="token punctuation">:</span>
 <span class="token punctuation">-</span> 192.168.76.81/28
</code></pre>
<p>Apply.</p>
<h4 id="enable-ip-forwarding">Enable IP forwarding</h4>
<p>Same as Gateway: <code>net.ipv4.ip_forward=1</code> in sysctl.</p>
<h4 id="install-dhcp-server-for-node-segment">Install DHCP server for Node segment</h4>
<p>Edit <code>/etc/kea/kea-dhcp4.conf</code> (second subnet):</p>
<pre class=" language-conf"><code class="prism  language-conf">{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp0s10" ]
    },
    "lease-database": {
      "type": "memfile",
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/tmp/kea-dhcp4-ctrl.sock"
    },
    "subnet4": [
      {
        "id": 2,
        "subnet": "192.168.76.80/28",
        "pools": [
          {
            "pool": "192.168.76.82 - 192.168.76.94"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.76.81"
          },
          {
            "name": "domain-name-servers",
            "data": "1.1.1.1"
          }
        ],
        "reservations": [
          {
            "hw-address": "08:00:27:59:D8:F3",
            "ip-address": "192.168.76.82"
          }
        ]
      }
    ],
    "loggers": [
      {
        "name": "kea-dhcp4",
        "severity": "INFO",
        "output_options": [
          {
            "output": "stdout"
          }
        ]
      }
    ],
    "ddns-send-updates": false,
    "ddns-override-client-update": false,
    "ddns-override-no-update": false
  }
}
 hardware 
</code></pre>
<p>Restart.</p>
<h4 id="firewall-–-allow-icmp-for-testing">Firewall – allow ICMP for testing</h4>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> ufw allow proto icmp from any to any
</code></pre>
<h3 id="node-ubuntu-node-configuration">3.4 Node (ubuntu-node) Configuration</h3>
<p>No static IP – use DHCP on bridged interface. Netplan:</p>
<pre class=" language-yaml"><code class="prism  language-yaml"><span class="token key atrule">network</span><span class="token punctuation">:</span>
 <span class="token key atrule">version</span><span class="token punctuation">:</span> <span class="token number">2</span>
 <span class="token key atrule">ethernets</span><span class="token punctuation">:</span>
 <span class="token key atrule">enp0s3</span><span class="token punctuation">:</span>
 <span class="token key atrule">dhcp4</span><span class="token punctuation">:</span> <span class="token boolean important">true</span>
</code></pre>
<p>Apply, then verify:</p>
<pre class=" language-bash"><code class="prism  language-bash">ip addr show enp0s3   <span class="token comment"># should show 192.168.76.82/28</span>
ip route show default  <span class="token comment"># gateway 192.168.76.81</span>
</code></pre>
<h3 id="ubuntu-verification-same-hop‑by‑hop-tests">3.5 Ubuntu Verification (same hop‑by‑hop tests)</h3>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ping</span> -c 4 192.168.76.81
<span class="token function">ping</span> -c 4 192.168.76.1
<span class="token function">ping</span> -c 4 1.1.1.1
<span class="token function">ping</span> -c 4 google.com
<span class="token function">traceroute</span> -n 1.1.1.1
</code></pre>
<hr>
<h2 id="problems-encountered-and-solutions">4. Problems Encountered and Solutions</h2>
<h3 id="windows-install-remoteaccess-command-not-found">4.1 Windows: <code>Install-RemoteAccess</code> command not found</h3>
<p><strong>Problem:</strong> After <code>Install-WindowsFeature -Name Routing</code>, the cmdlet was not available.<br>
<strong>Cause:</strong> The Routing role installation requires a reboot before the RemoteAccess PowerShell module becomes accessible.<br>
<strong>Solution:</strong> Rebooted the Gateway before running <code>Install-RemoteAccess</code>. After reboot, the command worked.</p>
<h3 id="dhcp-client-on-node-“ip-address-already-in-use-on-the-network”">4.2 DHCP client on Node: “IP address already in use on the network”</h3>
<p><strong>Problem:</strong> Node obtained <code>192.168.76.82</code> but Windows disabled the interface with a duplicate address error, even though no other device used that IP.<br>
<strong>Diagnosis:</strong> Checked Relay DHCP leases: <code>Get-DhcpServerv4Lease -ScopeId 192.168.76.80 -AllLeases</code> showed <code>AddressState = InactiveReservation</code>. The reservation existed but had never been successfully leased.<br>
<strong>Solution:</strong></p>
<ol>
<li>
<p>Removed the reservation: <code>Remove-DhcpServerv4Reservation -ScopeId 192.168.76.80 -IPAddress 192.168.76.82 -Confirm:$false</code></p>
</li>
<li>
<p>Re‑added the reservation with the same MAC.</p>
</li>
<li>
<p>Restarted DHCP service on Relay.</p>
</li>
<li>
<p>On Node: <code>ipconfig /release</code> and <code>ipconfig /renew</code>.<br>
<strong>Still had error.</strong></p>
</li>
<li>
<p>Disabled Duplicate Address Detection on Node:</p>
<pre class=" language-powershell"><code class="prism  language-powershell"><span class="token function">Set</span><span class="token operator">-</span>NetIPInterface <span class="token operator">-</span>InterfaceAlias <span class="token string">"Ethernet"</span> <span class="token operator">-</span>AddressFamily IPv4 <span class="token operator">-</span>DadTransmits 0
</code></pre>
<p>Then rebooted Node. After reboot, <code>ipconfig /renew</code> succeeded and the interface stayed up.</p>
<p><strong>Root cause:</strong> VirtualBox’s bridged mode with promiscuous “Allow All” can cause ARP probes to be reflected, tricking Windows DAD into thinking the IP is taken. Disabling DAD in an isolated lab is safe.</p>
</li>
</ol>
<h3 id="ubuntu-dhcp-server-not-responding-on-relay">4.3 Ubuntu: DHCP server not responding on Relay</h3>
<p><strong>Problem:</strong> Node received APIPA address (169.254.x.x) instead of <code>192.168.76.82</code>.<br>
<strong>Cause:</strong>  <code>kea-dhcp4-server</code> was bound to wrong interface; also needed to disable rogue detection (not a concept on Linux, but service wasn’t listening on bridged interface).<br>
<strong>Solution:</strong></p>
<ul>
<li>
<p>Verified <code>"interfaces": [ "enp0s10" ]</code> in <code>/etc/kea/kea-dhcp4.conf</code>.</p>
</li>
<li>
<p>Checked that <code>enp0s10</code> had the static IP <code>192.168.76.81/28</code>.</p>
</li>
<li>
<p>Restarted service and checked <code>sudo systemctl status kea-dhcp4-server</code>.</p>
</li>
<li>
<p>Also ensured <code>ufw</code> allowed DHCP (UDP port 67) – though by default DHCP server creates its own iptables rule.</p>
</li>
</ul>
<h3 id="windows-relay-could-not-ping-gateway-after-switching-internal-to-dhcp">4.4 Windows Relay could not ping Gateway after switching Internal to DHCP</h3>
<p><strong>Problem:</strong> After <code>Set-NetIPInterface -Dhcp Enabled</code>, Relay got <code>169.254.x.x</code>.<br>
<strong>Cause:</strong> Gateway’s DHCP firewall rule missing or rogue detection still enabled.<br>
<strong>Solution:</strong> On Gateway, confirmed <code>Get-DhcpServerSetting</code> showed <code>IsAuthorized = True</code>. Re‑ran <code>Set-ItemProperty</code> for <code>DisableRogueDetection</code> and restarted DHCP. Also verified <code>Allow-DHCP-In</code> firewall rule existed. Then on Relay, forced renewal: <code>ipconfig /renew "Ethernet 2"</code>. Worked.</p>
<h3 id="ubuntu-default-route-missing-on-relay-after-dhcp">4.5 Ubuntu: Default route missing on Relay after DHCP</h3>
<p><strong>Problem:</strong> Relay got IP <code>192.168.76.2</code> from Gateway’s DHCP but no default route.<br>
<strong>Cause:</strong>  <code>kea-dhcp4-server</code> on Gateway did not include <code>option-data</code> in the subnet declaration.<br>
<strong>Solution:</strong> Added <code>option-data 192.168.76.1;</code> to the subnet block and restarted DHCP. Then on Relay, <code>sudo dhclient -r enp0s8 &amp;&amp; sudo dhclient enp0s8</code>.</p>
<hr>
<h2 id="lab-4-firewall-summary">5. Lab 4 Firewall Summary</h2>
<h3 id="expected-behavior-–-strict-firewall-mode">1. Expected Behavior – Strict Firewall Mode</h3>

<table>
<thead>
<tr>
<th>Source</th>
<th>Destination</th>
<th>Service / Protocol</th>
<th>Windows Result</th>
<th>Linux (UFW) Result</th>
</tr>
</thead>
<tbody>
<tr>
<td>School PC (<code>192.168.56.x</code>)</td>
<td>Gateway</td>
<td>ICMP (ping)</td>
<td>❌ Blocked (timeout)</td>
<td>❌ Blocked (timeout)</td>
</tr>
<tr>
<td>School PC (<code>192.168.56.x</code>)</td>
<td>Gateway</td>
<td>SSH (TCP/22)</td>
<td>✅ Allowed (login)</td>
<td>✅ Allowed (login)</td>
</tr>
<tr>
<td>Node (<code>192.168.76.82</code>)</td>
<td>Gateway</td>
<td>ICMP (ping)</td>
<td>✅ Allowed (reply)</td>
<td>✅ Allowed (reply)</td>
</tr>
<tr>
<td>Node (<code>192.168.76.82</code>)</td>
<td>Internet (any)</td>
<td>Any (HTTP, DNS, …)</td>
<td>✅ Allowed (outbound)</td>
<td>✅ Allowed (outbound)</td>
</tr>
<tr>
<td>Relay (<code>192.168.76.2</code>)</td>
<td>Gateway</td>
<td>DHCP (UDP/67)</td>
<td>✅ Allowed (lease)</td>
<td>✅ Allowed (lease)</td>
</tr>
</tbody>
</table><h2 id="note-all-traffic-from-inside-192.168.76.024-is-fully-permitted-windows-allow-internal-in-rule-linux-ufw-allow-from-192.168.76.024."><strong>Note:</strong> All traffic from inside <code>192.168.76.0/24</code> is fully permitted (Windows <code>Allow-Internal-In</code> rule; Linux <code>ufw allow from 192.168.76.0/24</code>).</h2>
<h2 id="verification-steps">2. Verification Steps</h2>
<h3 id="from-the-k1001-pc-host">From the K1001 PC (Host)</h3>
<p>Run these commands on the <strong>host</strong> machine (or another VM on the same Host‑Only network):</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ping</span> 192.168.56.101          <span class="token comment"># Should time out (replace with Gateway's Host‑Only IP)</span>
<span class="token function">ssh</span> administrator@192.168.56.101   <span class="token comment"># Should succeed (password prompt)</span>
</code></pre>
<p><strong>How to verify on Windows:</strong></p>
<ul>
<li>
<p>Check current default action: <code>Get-NetFirewallProfile | Select Name, DefaultInboundAction</code></p>
</li>
<li>
<p>Toggle: <code>Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultInboundAction Block/Allow</code></p>
</li>
</ul>
<p><strong>On Ubuntu (ufw):</strong></p>
<ul>
<li>
<p><code>sudo ufw status verbose</code> shows default incoming policy.</p>
</li>
<li>
<p>Toggle: <code>sudo ufw default deny incoming</code> / <code>sudo ufw default allow incoming</code>.</p>
</li>
</ul>
<hr>
<h2 id="conclusion">6. Conclusion</h2>
<p>The routing lab was successfully implemented for <strong>Y = 76</strong> on both Windows Server 2022 and Ubuntu 26.04. All three VMs (Gateway, Relay, Node) can communicate through the intended chain, and the Node reaches the internet via NAT on the Gateway. Lab 4’s strict firewall policy blocks unsolicited inbound traffic except SSH, while allowing full internal communication and outbound connections. The encountered problems (missing RRAS cmdlet, DHCP inactive reservation, Ubuntu DHCP interface binding) were documented and resolved. The configuration is reproducible by following the steps above.</p>
<p><strong>Final verification from Node (Windows):</strong></p>
<pre class=" language-cmd"><code class="prism  language-cmd">C:\&gt; ping 192.168.76.81   → replies 
C:\&gt; ping 192.168.76.1    → replies 
C:\&gt; ping 1.1.1.1         → replies 
C:\&gt; tracert 1.1.1.1      → 1: 192.168.76.81, 2: 192.168.76.1, 3: 10.0.2.2
</code></pre>

