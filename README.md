Built-in wireless on the Pi Zero W opens up a huge number of possibilities for the various USB gadget modes. The Pi Zero W can be configured to spoof different USB device types, such as a keyboard, a webcam, or a smart USB flash drive.

At home, many people use a USB flash drive to transfer files to a TV, but it takes time to move the drive to and from the source computer. How about a remotely accessible USB flash drive, permanently located in the TV, where you could transfer files using the wireless connection? Drag and drop, job done!

STEP-01: The Pi Zero W USB flash drive
On Raspbian Jessie, wireless connections can be made via the networking icon on the right-hand end of the desktop menu bar. Left-clicking the icon will bring up a list of available networks. If you see the scanning message, wait for a moment and it should find your network.

Click the network that you want to join. If the network is secured, you will see a dialogue box. Enter the password, click OK, and wait for a few seconds. Finally, write down the IP address the Pi has acquired. Hover the mouse over the signal strength icon and the IP address will appear in a tooltip. If you prefer, you can complete this process on another Pi, and move the SD card over to the Pi Zero W when you reach Step 03.

STEP-02: Disable desktop and enable SSH
From this point we don’t need the desktop, because the Pi Zero is going to have one job: to be a file server for the TV. To save CPU cycles, we should disable the automatic boot to desktop. Go to Menu (Raspberry icon at the top-left corner of the screen) Preferences Raspberry Pi Configuration.

On the System tab, find the Boot option. Select ‘To CLI’ (command line interface), and disable the ‘Login as user pi’ checkbox.
On the Interfaces tab, make sure that SSH is enabled. Click OK and choose Yes to reboot. After the reboot, you should see a text login prompt with a flashing cursor.

STEP-03: Switch to remote access
We recommend using SSH (Secure Shell), because we’ll be doing everything using the command line. SSH allows us to see the Raspberry Pi command prompt in a window on another PC, Mac, smartphone, or tablet.

Go to magpi.cc/2sLqBmM and follow the instructions for the platform you’re using. You’ll need the IP address you wrote down earlier, and the login. The default login for Raspbian is pi, with the password raspberry.

When the SSH link is working, you can disconnect any keyboard and screen.

STEP-04: Free up some disk space
When you’ve logged in over SSH, you can free up disk space by removing some programs that we’re not going to need for this project.
The command df -h shows your disk space usage. Look at the Avail column for /dev/root to see how much free space you have. We can claw back about 1GB if we remove LibreOffice and Wolfram.

Enter the commands below into your SSH client:

sudo apt-get remove --purge libreoffice* -y

You can safely ignore any errors you see here. They happen because the command will try to remove some parts of LibreOffice that aren’t actually installed.

sudo apt-get purge wolfram-engine -y
sudo apt-get clean
sudo apt-get autoremove -y

Now run df -h again and check the Avail column.

STEP-05: Enable the USB driver
Next, we need to enable the USB driver which provides the gadget modes, by editing two configuration files.

sudo nano /boot/config.txt

Scroll to the bottom and append the line below:

dtoverlay=dwc2

Press CTRL+O followed by Enter to save, and then CTRL+X to quit.

sudo nano /etc/modules

Append the line below, just after the i2c-dev line:

dwc2

Press CTRL+O followed by Enter to save, and then CTRL+X to quit.
Now shut down the Pi Zero W with the command:

sudo halt

STEP-06: Switch to TV power
On the Pi Zero W, you’ll see two micro USB ports. One is marked ‘USB’ and the other ‘PWR IN’. You can supply power through either port, but the USB port is for data as well. There are two options.

You could plug the TV into the Pi Zero W USB port, not the PWR IN port, using a standard micro USB cable. The cable will both supply power from the TV and make the USB data connection. The disadvantage is that the TV must be switched on to supply power to the Pi. When someone turns the TV off with the remote, the Pi will also lose power, which can corrupt your SD card.

Alternatively, you can connect a separate, always-on power supply to the PWR IN port, and use a slightly modified micro USB cable to connect the TV to the USB port. The modification is to cut the red wire inside the micro USB cable. This protects the Pi from damage that could be caused by drawing power from two different power sources. The advantage of this method is that the Pi is powered independently from the TV. It will be available on the network even if the TV is off, and there is a reduced risk of sudden power loss and SD card corruption.

You might want to test the system with the first option, and then move onto the second when you want a more permanent setup. Don’t forget to cut the red wire if you use the second option.

Connect the Pi Zero W USB port to the TV using your chosen method, power everything up, and log back in over SSH.

STEP-07: Create a container file
To enable mass storage device mode, we need to create a large file to act as the storage medium on the SD card. This file will emulate the USB flash drive that the TV sees.

The command below will create an empty 2GB binary file (change the count=2048 parameter if you want a different size). Please note that this will be limited by the available free space on your SD card (check the Avail column in df -h), and it may take a few minutes to complete the setup:

sudo dd bs=1M if=/dev/zero of=/piusb.bin count=2048

We now need to format the file as a FAT32 file system so that the TV can understand it. Enter the command below:

sudo mkdosfs /piusb.bin -F 32 -I

STEP-08: Mount the container file
Now let’s mount the container file locally so we can download some test files. First, create a folder on which we can mount the file system:

sudo mkdir /mnt/usb_share

Now let’s add this to fstab, the configuration file that records our available disk partitions:

sudo nano /etc/fstab

Append the line below to the end of the file:

/piusb.bin /mnt/usb_share vfat users,umask=000 0 2

Press CTRL+O followed by Enter to save, and then CTRL+X to quit.
The line we added to fstab allows the USB file system to be error-checked and mounted automatically at boot time. Instead of rebooting, we can manually reload fstab with the command below:

sudo mount -a

STEP-09: Download a test file
Now let’s download a 300MB test file to view on the TV. Big Buck Bunny is a short open-source film, made by the Blender Foundation (www.blender.org), and released under the Creative Commons Attribution License 3.0:

cd /mnt/usbshare
wget http://download.blender.org/peach/bigbuckbunnymovies/bigbuckbunny720psurround.avi

You’ll see a progress bar move from left to right. When the download is complete, run a command to flush any cached data to the disk:

sync

STEP-10: Test mass storage device mode
Now comes the moment of truth. Let’s see whether the TV is going to be friends with the Pi Zero W. The command below will enable USB mass storage device mode, and the TV should pop up a dialogue box. If it doesn’t, you may need to use the Input or Source button on the TV remote to select the USB device.

sudo modprobe g_mass_storage file=/piusb.bin stall=0 ro=1

The TV should provide a file browsing interface. Locate the Big Buck Bunny file and hit Play.

Once you’re satisfied that all is well, try a dismount:

sudo modprobe -r g_mass_storage

The correct behaviour here is for the film or browsing interface to disappear from the screen. You may see a message saying that the USB device was disconnected.

STEP-11: Install and configure Samba
The next step is to provide network access to the /mnt/usb_share folder that we created earlier.

sudo apt-get update
sudo apt-get install samba winbind -y

When the installation is complete, we need to configure a Samba network share. For simplicity, this will not require a user name or password, as it is already protected by your wireless network security. If you want more security, see wiki.samba.org.

sudo nano /etc/samba/smb.conf

Scroll down to the end of the file and append the lines below:

[usb]
browseable = yes
path = /mnt/usb_share
guest ok = yes
read only = no
create mask = 777

Press CTRL+O followed by Enter to save, and then CTRL+X to quit.
Now restart the Samba service for the changes to take effect:

sudo systemctl restart smbd.service

STEP-12: Access the share from another computer
Now we can try to access the share from a Windows PC or a Mac. You’ll need the host name the Raspberry Pi is using. To check this, enter the command below:

cat /etc/hostname

By default this will be raspberrypi.

In Windows, you can bring up Explorer (Windows key + E) and type \raspberrypi into the address bar at the top. The Run dialogue also works (Windows key + R).
On macOS, the Raspberry Pi will show up in the Finder sidebar. Alternatively, from the Finder menu, select Go Connect to server (Apple key + K) and type smb://raspberrypi as the server address.

Pi Zero W USB Flash Drive
Pi Zero W USB Flash Drive

Depending on your network settings, you may still see a login dialogue. Any user name, including a guest or anonymous login, will work. Once you’re in, you’ll see a share named usb. Open this and test that you have write access, either by creating a new folder or by copying over a file.

You can enable mass storage device mode to check that the TV can see your changes, but don’t forget to run the sync command first.

If you want to change the host name, edit the name in /etc/hostname. Make the same change in /etc/hosts, and finally use sudo reboot to apply the changes.

STEP-13: Automate USB device reconnect
In order for the TV to detect any changes we’ve made over the network (for example, file or folder creations and deletions), it needs to be tricked into thinking that the USB device has been removed and reinserted.

We can use a Python library called watchdog (magpi.cc/2sLL1fi), which is designed for monitoring file system events and responding to them. Install this with the command below:

sudo pip3 install watchdog

We then need some code to start a timer whenever something changes in the shared folder. The timer is reset to zero every time a new change occurs, and the USB reconnect is only triggered if we see 30 seconds of inactivity after a change. This avoids spamming the TV while we’re copying over multiple files.

We’ve written a program to do this. To download it, type:

cd /usr/local/share
sudo wget http://rpf.io/usbzw -O usbshare.py // Look in this Repo 
sudo chmod +x usbshare.py

USB Flash Drive
USB Flash Drive

STEP-14: Background service
We need to make this program into a background service, so that it starts automatically at boot time. We can do that by making it into a systemd unit. Enter the commands below:

cd /etc/systemd/system
sudo nano usbshare.service

This will start a new blank file. In the new file, enter:

[Unit]
Description=USB Share Watchdog

[Service]
Type=simple
ExecStart=/usr/local/share/usbshare.py
Restart=always

[Install]
WantedBy=multi-user.target

Press CTRL+O followed by Enter to save, and then CTRL+X to quit.
Now we need to register the service, enable it, and set it running:

sudo systemctl daemon-reload
sudo systemctl enable usbshare.service
sudo systemctl start usbshare.service

Whenever you copy files over to the network share, the USB device should automatically reconnect to the TV after 30 seconds of inactivity.
