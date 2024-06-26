# CRACKY 
## _Digital Forensics_

![alt text](screenshot.PNG)

In the challenge we were given a file named "trafficanti.pcap" and the note said that the router's WIFI password is in this given format "isetcom{special character}{4-digits-number}", and the flag should be "Spark{filename}".

## Approach

We can start by looking into the capture using Wireshark 

![alt text](traffic-capture.PNG)

First thing to notice is `802.11` frames, which are a result of a wireless packets capture. 

Taking a closer look, we found another protocol existing in the capture which is `eapol`, we can verify all it's relating frames by filtering for `eapol` packets :

![alt text](eapol-capture.PNG)

We can verify now that we have a full 4-way handshake that are the main way to accomplish the authentication conversation. We also noticed the security standard used in this wireless connection : `WPA`

![alt text](wpa-capture.PNG)

## Decryption

Refering to the note mentioned in the description we know that the WPA key (WIFI password) is in this format : `example: isetcom*1234` 

We are going to construct a custom wordlist to bruteforce that handshake and find the password, for that case the best choice is to use `crunch` tool :

```sh
crunch 12 12 -t isetcom^%%%% -o mywordlist
```

After extracting our wordlist we may now use another tool to bruteforce the password used for encrypting the traffic, we are going to use the infamous `aircrack-ng`.

Before starting the attack we need the `BSSID` of the victim router. By refering to the workflow of the 4-way handshake in the following link, we would notice that the Access Point is the first device to start the conversation for authentication : 

[WPA 4 way handshake workflow](https://cdn.networklessons.com/wp-content/uploads/2023/12/wpa-4-way-handshake-workflow.png)

so we can look back at the first `eapol` frame to get his `BSSID`:

![alt text](bssid-capture.PNG)

Command construction : 
```sh
aircrack-ng -w mywordlist.txt -b 02:24:78:a6:18:bc trafficanti.pcap
```

results of `aircrack-ng` bruteforce :

![alt text](aircrack-results.PNG)

We found the password! `isetcom-2024`

Now we can use that key to decrypt the traffic and look for the filename :
`Preferences > Protocols > IEEE 802.11 > Edits`

![alt text](edit-key.PNG)

We noticed multiple protocols showed up now, after filtering for `HTTP` we noticed a web server frames and a GET request to `/student/` endpoint :

![alt text](http-traffic.PNG)

Inspecting the frame through `Follow > HTTP stream` we found `74c68f1f36a5814cb30fb0b42f2b5b69.txt` that should be our flag! 

![alt text](student-content.PNG)

`Spark{74c68f1f36a5814cb30fb0b42f2b5b69}`

> Note: `aircrack-ng` might not detect the `eapol` frames to crack it. For that we can use [cap2hashcat](https://hashcat.net/cap2hashcat/) to extract the hash and crack it in `22000` mode.