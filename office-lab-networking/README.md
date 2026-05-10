# Office Lab Networking

Real network configurations implemented on top of the sysadmin-lab environment.

On this section I'll document the evolution from a basic static IP setup to a more complete network infrastructure, simulating how it would grow in a real small corporate environment.
---

## 01 - DHCP

Previously PC01 had a static IP configured manually during the initial lab setup. Configuring DHCP on DC01 means the server now distributes IPs automatically to any client that connects to the Internal Network.

### Install DHCP role on DC01

```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```

### Configure the scope

The scope defines the range of IPs the server can distribute.

```powershell
Add-DhcpServerv4Scope -Name "office.lab" -StartRange 10.0.0.100 -EndRange 10.0.0.200 -SubnetMask 255.255.255.0
```

> 📝 Note: Range starts at .100 to keep the lower IPs reserved for servers and network devices with static IPs. This gives room for up to 100 clients.

### Configure gateway and DNS for the scope

```powershell
Set-DhcpServerv4OptionValue -ScopeId 10.0.0.0 -Router 10.0.0.1 -DnsServer 10.0.0.1 -DnsDomain "office.lab"
```

Without setting the gateway and DNS on the scope, clients would receive an IP but wouldn't know how to reach other networks or resolve domain names. These options tell every client automatically where to send traffic and where to look up names, so you don't have to configure each machine manually.

### Authorize DHCP server in Active Directory

```powershell
Add-DhcpServerInDC -DnsName "DC01.office.lab" -IPAddress 10.0.0.1
```

Authorizing the DHCP server in Active Directory is a security measure. It prevents rogue DHCP servers from handing out IPs on the domain network. If a server isn't authorized, AD simply ignores it. Sounds simple but it's actually an important step that is easy to forget.

### Verify DHCP is active

```powershell
Get-DhcpServerv4Scope
Get-DhcpServerInDC
```

---

## Remove static IP from PC01

With DHCP now configured, PC01 no longer needs a static IP. Make sure to go back to the PC01 terminal and remove the static IP, the gateway and most importantly enable DHCP and reset the DNS, otherwise the machine won't pick up the new settings.

```powershell
# Remove static IP
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false

# Remove static gateway
Remove-NetRoute -InterfaceAlias "Ethernet" -Confirm:$false

# Enable DHCP
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled

# Reset DNS to automatic
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses
```

### Verify PC01 received an IP from DHCP

```powershell
ipconfig /all
```

Check that DHCP Enabled shows Yes, IPv4 Address is within the scope range (10.0.0.100 to 10.0.0.200) and DNS Server points to 10.0.0.1.

### Confirm connectivity

```powershell
ping 10.0.0.1
nslookup office.lab
```

### Check active leases on DC01

This shows all IPs currently distributed by the DHCP server and which machine received each one. Useful to confirm everything is working and to track which clients are on the network.

```powershell
Get-DhcpServerv4Lease -ScopeId 10.0.0.0
```

---

## 02 - DNS Forwarder

DC01 resolves names inside the office.lab domain perfectly. But without a forwarder, any external domain like google.com would fail because DC01 doesn't know anything outside its own domain.

A DNS Forwarder tells DC01: if you don't know how to resolve a name, forward the query to this external DNS server.

### Configure DNS Forwarder on DC01

```powershell
Set-DnsServerForwarder -IPAddress "8.8.8.8","1.1.1.1"
```

### Verify

```powershell
Get-DnsServerForwarder
```

### Test from PC01

```powershell
nslookup google.com
```

DC01 should appear as the DNS server and google.com should resolve successfully, confirming DC01 is forwarding external queries to Google.

---

## 03 - Ubuntu Server Client (PC02)

Created a third VM on VirtualBox running Ubuntu Server to optimize RAM usage and test DHCP cross-platform distribution.

### Installation

During the Ubuntu Server installation, I chose to enable OpenSSH server. This allows me to access PC02 via SSH from DC01 or PC01, which gives a better terminal experience with proper resolution and copy/paste support.

### Network Configuration

After installation, connect the VM to Internal Network (adlab) in VirtualBox settings. Ubuntu Server comes with DHCP enabled by default via netplan, so no manual configuration is needed for the IP.

```bash
sudo netplan apply
ip addr show enp0s3
```

### Configure DNS server

DHCP assigns the IP automatically but DNS needs to be set explicitly on Linux. I first tried setting it via netplan:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
      nameservers:
        addresses: [10.0.0.1]
```

```bash
sudo netplan apply
```

Even after this, nslookup was still resolving through a different DNS server. The issue is that Ubuntu's systemd-resolved intercepts DNS queries before netplan settings take effect. The fix was to disable it and set the DNS manually:

```bash
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 10.0.0.1" | sudo tee /etc/resolv.conf
```

### Confirm on DC01

```powershell
Get-DhcpServerv4Lease -ScopeId 10.0.0.0
```

![DHCP leases on DC01](screenshots/dhcp-leases.png)

PC02 received 10.0.0.100 and PC01 received 10.0.0.101, confirming the DHCP server distributes IPs correctly across both Windows and Linux clients.

PC01 - Confirming DHCP Distribution
![PC01 IP address](screenshots/pc01-ip.png)

PC02 - Confirming DHCP Distribution
![PC02 IP address](screenshots/pc02-ip.png)
