# Kali Linux Raspberry Pi Zero W
So I was trying to setup my Raspberry Pi Zero W with Kali Linux like [this](https://www.kali.org/docs/arm/raspberry-pi-zero-w-pi-tail/)  
  
And everything worked great, except for the bluetooth tethering which was very important to me.  
No matter what I did it didn't seem to work.  
Eventually I got it working and this is how I did it.  
  
First you need to pair the phone and the Pi Zero W:  
```sh
# Before acutally doing that we fix some bluetooth errors
timeout 10 sudo bluetoothd  # Run it for 10 seconds to create a bluetooth directory that was missing
sudo systemctl restart hciuart  # Restart hciuart service
sudo systemctl restart bluetooth  # Restart bluetooth service

# Then open the bluetoothctl interface
bluetoothctl
# In the bluetoothctl interface:
scan on  # Start scanning for bluetooth devices, turn on bluetooth on your phone and scan until it is shown
scan off  # Stop scanning for bluetooth devices
pair AA:BB:CC:DD:EE:FF  # Pair with your phones bluetooth address
exit
sudo reboot
```
After our devices are paired, we need to have them connect and enable bluetooth tethering
```sh
# Start with the same setup as before
timeout 10 sudo bluetoothd  # Run it for 10 seconds to create a bluetooth directory that was missing
sudo systemctl restart hciuart  # Restart hciuart service
sudo systemctl restart bluetooth  # Restart bluetooth service
busctl call org.bluez /org/bluez/hci0/dev_AA_BB_CC_DD_EE_FF org.bluez.Network1 Connect s nap  # Connect to phone via bluetooth, replace the MAC address here with your own
sudo ifconfig bnep0 192.168.44.204 netmask 255.255.255.0  # Configure static IP for the new bluetooth tethered interface. Make sure the IP address is in the subnet that matches your phone's bluetooth tethering subnet
```
Now everything should work!  
I created a bash script to make this easier:  
```bash
#!/bin/bash

# While trying to connect via bluetooth tethering to my pi zero w tails kali machine
# I ran into a bunch of bluetooth related errors.
# The following commands seem to fix everything.
# This is done after pairing the phone and raspberry pi via bluetoothctl.
# To get to the point where bluetoothctl works do these commands:
# 	sudo bluetoothd # Wait like 10 seconds
#	sudo systemctl restart hciuart
#	sudo systemctl restart bluetooth
# Now you can open bluetoothctl, enter the 'scan on' command, find the device, do 'scan off', and then 'pair AA:BB:CC:DD:EE:FF'

DEVICE_BLUETOOTH_MAC=AA_BB_CC_DD_EE_FF
BLUETOOTH_INTERFACE_IP=192.168.44.204
BLUETOOTH_INTERFACE_NETMASK=255.255.255.0

echo '[+] Running bluetoothd for 10 seconds'
timeout 10 sudo bluetoothd
echo '[+] Restarting hciuart service'
sudo systemctl restart hciuart
echo '[+] Restarting bluetooth service'
sudo systemctl restart bluetooth
echo '[+] Waiting 7 seconds'
sleep 7
echo '[+] Attempting to connect to device via bluetooth 10 times'
for i in {1..10}
do 
	echo "Attempt $i"
        busctl call org.bluez /org/bluez/hci0/dev_$DEVICE_BLUETOOTH_MAC org.bluez.Network1 Connect s nap
	sleep 1
done
echo '[+] Waiting 5 seconds'
sleep 5
echo '[+] Setting up bluetooth interface IP address'
sudo ifconfig bnep0 $BLUETOOTH_INTERFACE_IP netmask $BLUETOOTH_INTERFACE_NETMASK
echo '[+] Done!'
```
Save this file at /opt/my_scripts/setup_and_connect_bluetooth.sh
  
But now we want to have this script run automatically on startup, so let's make a service:  
```service
[Unit]
Description=Attempts to connect to a device via bluetooth (tethering) on startup
After=network.target

[Service]
ExecStart=/opt/my_scripts/setup_and_connect_bluetooth.sh
User=root
Type=oneshot

[Install]
WantedBy=multi-user.target
```
Save this to the file /etc/systemd/system/setup_and_connect_bluetooth.service  
Then to start the service:  
```sh
systemctl daemon-reload  # reload systemd with this new service unit file
systemctl enable setup_and_connect_bluetooth.service  # enable the service to have it run on startup
```
Now after rebooting the Pi Zero W and attempting to connect from your phone like we did before,  
Everything should work and the Pi Zero W should automatically attempt to connect to your phone  

Enjoy!
