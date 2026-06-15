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
<h2 id="network-topology">1. Network Topology</h2>

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
<td>for SSH from school PC</td>
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

