# ESP32_VirtualPrinter
ESP32 emulator for LPT/parallel port printers

![ESP32 wired up to a paralell port breakout](ESP32Printbox.jpg)
![ESP32 wired up to a paralell port breakout](ESP32Printboxinside.jpg)
![Demoprint](HM507print.pdf)

## Project status
Very rough, but works well enough™   
Hardcoded SSID, PSK, target TCP socket settings. This would be very easy to fix, if required. 

## Possible improvements
Currently, one IRQ needs to be executed for each byte and the bits are read manually.  
A `while (true)` reading from the pins and doing the handshaking might be better suited.  

A FIFO-buffer should be implemented to buffer the data between the LPT and the TCP socket.  
The buffers fill-level could/should be used to indicate the BUSY flag to the LPT host.  

Maybe possible: The ESP32 has an I2S peripheral with a parallel input & DMA capability. This mode is meant to be used with 8-bit camera modules, but it might (maybe with a bit of external logic) be compatible with a parallel port. This would allow for very high speed data input into the ESP32. Needs investigation.

## Why?
To capture data from historic/vintage machines without using actual printers.  
The ESP32 will act as a LPT/IEEE1284 printer and send all received data over a raw TCP socket.  
This socket can be terminated in DIY software, in CUPS (to generate .pdf files) or in a real network printer directly. 

## What do I need?
An ESP32 and some way to connect it to the DB25 or Centronics connector.  
There are cheap DB25 breakouts available or you can wire your own connector.  

Note: ESP32 are not really 5V-compatible. It still works when connected directly, but this is out-of-spec.   
In a production setup, putting resistors of about 100 Ohms to 5000 Ohms between all the pins (except GND) is recommended. 1kOhm is probably a good value.  

| Name            	| ESP32 Pin 	| LPT Pin 	|
|-----------------	|-----------	|---------	|
| !Strobe         	| 34 (changed)| 1       	|
| D0              	| 4         	| 2       	|
| D1              	| 14        	| 3       	|
| D2              	| 27        	| 4       	|
| D3              	| 26        	| 5       	|
| D4              	| 25        	| 6       	|
| D5              	| 33        	| 7       	|
| D6              	| 32        	| 8       	|
| D7              	| 18        	| 9       	|
| !Ack            	| 15        	| 10      	|
| Busy            	| 17        	| 11      	|
| Paper Out       	| 16        	| 12      	|
| Select In       	| 35        	| 13      	|
| !Line feed      	| 23        	| 14      	|
| Error           	| 22        	| 15      	|
| Reset           	| 21        	| 16      	|
| !Select printer 	| 19        	| 17      	|
| Ground          	| GND       	| 18      	|

## Configuration

Update the target TCP host and port, the WiFi SSID and PSK in the source code before flashing.  
PlatformIO is recommended, but the Arduino IDE with installed arduino-esp32 should also work (untested!)

The ESP32 will open a new TCP connection for each print job and close old ones after a fixed timeout of no data being sent.  
Setting the TCP host/port directly to Port 9100 of a real network printer will work fine.

## Can I change the pinout?
In theory, yes. None of the pins are fixed function.  
This project requires a lot of GPIOs though and many of the GPIOs on the ESP32 have additional functionality like strapping pins and might lead to strange behaviour.  
The current pinout in main.cpp was tested and works on a ESP32S Dev Kit C V4-clone from AZDelivery.

## Troubleshooting
Open a serial console with 115200 baud to the ESP32.   
Is the WiFI connection established successfully? You should see the ESP32's IP being listed.  
You can uncomment the `Serial.write()` and `Serial.flush()` calls in the receive-loop to get the raw data dumped to your serial terminal.  

On the parallel port, there are 2 lines called "SELECT", one is "SELECT_IN" (LPT pin 13), the other is "SELECT_PRINTER" (LPT pin 17). 
**There are wrong pinout graphics on the internet! Don't get fooled!**

## Virtual PDF printer using CUPS
Install CUPS and `cups-pdf`, visit CUPS configuration page at https://localhost:631, create a new printer of type "CUPS-PDF" and set it as the default. 
I use a plain Debian LXC on Proxmox and install cups, cups-pdf, cups-bsd (for lpr), nginx and samba.
To setup the Printer i link CUPS to Port 88. To do this create:
`/etc/nginx/sites-enabled/drucker` containing (the pass inside is not mine and i think it is not needed - but i also dont want to change this running system):
```
server {
  listen 88;
  location /.well-known/acme-challenge/ {
    root /tmp/certbot/;
    auth_basic off;
  }
  # Alle anderen Anfragen an localhost:631 weiterleiten
  location / {
    proxy_set_header Authorization "Basic ZHJ1Y2tlcjp0ZXN0";
    proxy_pass http://127.0.0.1:631;
  }
}
```

To easy get the PDF over samba add this to smb.conf:
```
[CUPS-PDFs]
comment = Samba on Debian
path = /home/pdf/
read-only = no
browsable = yes
```

And also change /etc/cups/cups-pdf.conf - so search for those 3 Rows inside the file and Change it:
```
Out /home/pdf
```
```
AnonUMask 0000
```
```
UserUMask 0000
```

Create `/etc/systemd/system/printserver@.service` containing
```ini
[Unit]
Description=Virtual IPP printserver Socket Per-Connection Server

[Service]
ExecStart=-/opt/printserver.sh
StandardInput=socket
StandardError=null
DynamicUser=true
```

Create `/etc/systemd/system/printserver.socket` containing
```ini
[Unit]
Description=Virtual IPP printserver Socket for Per-Connection Servers

[Socket]
ListenStream=31337
Accept=yes

[Install]
WantedBy=sockets.target
```

modified for PS Output on Scope:
and a shell script called `/opt/printserver.sh` containing
```bash
#!/bin/bash

lpr -PPDF -T "Scopeprint" -o landscape -l
```

Then run
```bash
chmod +x /opt/printserver.sh
systemctl enable --now printserver.socket
```
and try to `telnet 127.0.0.1 31337` and send some data.
After closing the connection, you should see a new file in `/var/spool/cups-pdf` (or an error message somewhere in the journal or CUPS log).
