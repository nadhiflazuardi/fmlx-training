# Making a Bootable USB Drive
1. **Install Rufus** 
    - Rufus is a utility app that can be used to write an ISO image to USB.
2. **Download Ubuntu Image**
    - Go to https://ubuntu.com/download/desktop to download an ISO image
3. Copy ISO image to drive using Rufus
	- Open rufus
	- Plug in USB drive
	- After plugging in, some settings should be filled automatically
	- Select ISO image
	- Make sure partition scheme is correct
		- Got to Computer Management -> Disk Management
		- Find the disk of USB drive -> Properties -> Volume -> look for Partition style
	- Press start, and wait until the process is finished

# Booting and Installing Using USB Drive
1. Plug the USB Drive into the computer
2. Restart the computer
3. Immediately press the esc button repeatedly until the Startup Menu opens
4. Enter the menu to choose boot options (F9 for HP 14s) and choose to boot using the drive
5. Wait for the installer program to show up, configure as needed, and wait for the installation to finish