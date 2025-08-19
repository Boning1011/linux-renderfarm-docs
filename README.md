# Linux Render Farm Setup Guide

A practical guide for setting up and managing a Linux-based render farm, primarily documented from personal implementation. While originally configured for use with Volvoxlabs's infrastructure, this documentation can be adapted for different environments. The examples use specific network paths and studio configurations which should be modified according to your setup. This guide assumes basic familiarity with Linux systems and render farm concepts.

> **Note:** Path examples and certain configurations reference personal/studio-specific setups and should be adjusted for your environment.


## 1. Mount NAS Folder

In Linux, accessing the NAS is handled differently compared to Windows. While Windows can connect directly using paths like `\\VVOX-NAS-1\PROJECTS`, Linux requires the NAS folder to be mounted as a local directory, typically under `/mnt`. This allows applications like Houdini and Deadline to interact with the NAS as if it were a local folder, ensuring smooth and consistent access.

**Note:**  
To maintain cross-platform compatibility, use paths that work on both Windows and Linux. Consider environment variables like `$HIP` or custom variables pointing to asset folders. For Deadline and HQueue, set appropriate path mappings.

### Temporally Mount a NAS Folder:
#### 1. Creating a Mount Point:
To set up a directory for mounting, use:
```
sudo mkdir -p /mnt/VVOX-NAS-1/your_folder_name
```
For example:
```
sudo mkdir -p /mnt/VVOX-NAS-1/projects
sudo mkdir -p /mnt/VVOX-NAS-1/deadline-repo
```
_Note:_ Verify the ownership of the folder. If needed, adjust it using:
```
sudo chown yourusername:yourusername /mnt/VVOX-NAS-1/your_folder_name
```
#### 2. Mount a Folder 
Use the following command to mount a folder with your NAS credentials:
```
sudo mount -t cifs -o username=myusername,password=mypassword,uid=1000,gid=1000,dir_mode=0777,file_mode=0777,vers=3.0 //10.0.10.11/PROJECTS /mnt/VVOX-NAS-1/projects
sudo mount -t cifs -o username=myusername,password=mypassword,uid=1000,gid=1000,dir_mode=0777,file_mode=0777,vers=3.0 //10.0.10.11/deadline10-repo /mnt/VVOX-NAS-1/deadline-repo
``` 
**Note:**
`10.0.10.11` is the IP address of VVOX-NAS-1. For unknown reason the `VVOX-NAS-1.local` sometimes having resolving issue on one of the machine, even after `sudo nano /etc/hosts` and add line `10.0.10.11 VVOX-NAS-1.local`
I've constantly met permission issue with NFS mounting so I switched to SMB.

### Automatically Mount at Startup:   
#### 1. Edit the /etc/fstab File
To set up automatic mounting on startup, edit the `/etc/fstab` file
```
sudo nano /etc/fstab
```
#### 2.  Add the Mount Entries
Append the following line:
```
//10.0.10.11/PROJECTS /mnt/VVOX-NAS-1/projects cifs username=myusername,password=mypassword,uid=1000,gid=1000,dir_mode=0755,file_mode=0755 0 0
//10.0.10.11/deadline10-repo /mnt/VVOX-NAS-1/deadline-repo cifs username=myusername,password=mypassword,uid=1000,gid=1000,dir_mode=0755,file_mode=0755 0 0
```
#### 3. Test the Configuration
To verify the settings, run:
```
sudo systemctl daemon-reload
sudo mount -a
```


---

## 2. Packages to Install

```
sudo apt install flatpak timeshift vlc imagemagick
sudo snap install mission-center
sudo snap install nmap
```

---

## 3. Houdini
### Dependencies:
Linux package requirements for Houdini 20.5:

https://www.sidefx.com/Support/system-requirements/linux-package-requirements-for-houdini-205/

```
sudo dnf install alsa-lib compat-openssl11 dbus-libs expat fontconfig glibc libatomic libevent libglvnd-glx libglvnd-opengl libICE libSM libX11 libX11-xcb libxcb libXcomposite libXcursor libXdamage libXext libXfixes libXi libxkbcommon libxkbcommon-x11 libXrandr libXrender libXScrnSaver libXt libXtst libzstd nspr nss nss-util openldap pciutils-libs tbb xcb-util xcb-util-image xcb-util-keysyms xcb-util-renderutil xcb-util-wm zlib 
```
### HQueue
To make sure the HQueue working properly, the two main env variables are: `HOUDINI_HQUEUE_SERVER` and `HOUDINI_HQUEUE_HFS`, one of the most common error is due to the client machine don't have corresponding minor houdini version installed. To solve this, managing the config through centralized packages is highly recommended.

#### Things to Check

1. The client machines are shown available on HQueue dashboard 

2. Make sure all machines env are properly shared and managed

3. `HOUDINI_HQUEUE_HFS` is set correctly.

Then TOP hqscheduler should likely work with all default parameters.

### Environment Variables:
For complete environment variable list: https://www.sidefx.com/docs/houdini/ref/env.html

**Deadline needs these lines to work (Only on Linux):**
```
DEADLINE_PATH="/opt/Thinkbox/Deadline10/bin"
PATH=$PATH:$DEADLINE_PATH
```
**To set a directory as global attribute:**

**Linux:**
```
ASSETS="/mnt/VVOX-NAS-1/projects/_____ASSETS"
```
**Windows**
```
ASSETS="//Vvox-nas-1/projects/_____ASSETS"
```



**For multi-gpu machine failed on render high-res image:**
```
KARMA_XPU_OPTIX_DISABLE_HOST_PINNED = 1
```

*Note: For daily-build after 20.5.395, this environment variable added as an internal debbug to disable the CUDA pin host memory feature. This feature will likely to cause the multi-gpu machine fail on allocating the host memory when rendering high resolution image.* 

**Other useful environment variables:**
```
#Disable CPU for XPU
#KARMA_XPU_DISABLE_EMBREE_DEVICE=1 #traditional method
KARMA_XPU_DEVICE = optix #new method, introduced in 20.5.394


#Disable Specific GPU for XPU 
KARMA_XPU_DISABLE_DEVICE_n=1

#Optimize Multi-GPU Performance
KARMA_XPU_NUM_PER_DEVICE_BLENDING_THREADS = 4

#Use Box1 as HQueue Server
HOUDINI_HQUEUE_SERVER = http://10.0.10.203:5000/
```

### Trouble Shooting Houdini Crash on Startup:

Houdini crashes on startup when launched on headless machine.

#### Solution 1 - Dongle

Using a headless dongle and then restart. Potential issue is the dongle isn't maintaining a stable X11 session when the real monitor disconnects. 

Try congfigure dongle BEFORE disconnecting monitor:

```
xrandr --output [DONGLE_NAME] --mode 1920x1080 --primary
xrandr --output [MONITOR_NAME] --off
```

#### Solution 2 - Launch from terminal:

```
export __GLX_VENDOR_LIBRARY_NAME=nvidia
/opt/hfs20.5.445/bin/houdini
```
*Caution: DO NOT directly adding __GLX_VENDOR_LIBRARY_NAME=nvidia to /etc/environment, which will casue Teamviewer unable to connect*

*Note: When connected to a monitor, launch in terminal in the way above with casue GLX context conflict, resulting black GUI.*

#### Solution 3 - Use virtual display instead of dongle:
```
# Install virtual framebuffer
sudo apt install xvfb

# Create startup script for headless operation
sudo nano /etc/systemd/system/headless-display.service
```
```
[Unit]
Description=Headless Display
After=graphical.target

[Service]
ExecStart=/usr/bin/Xvfb :0 -screen 0 1920x1080x24
Restart=always

[Install]
WantedBy=graphical.target
```

### (Untested)To solve Houdini crash on Fedora(Wayland):

```
sudo dnf install gnome-session-xsession
```
Then restart and choose the Xorg version of Gnome on the login screen.

*Note: Developers said they are planning to add support on Wayland in Houdini 21 with Qt6 build. User from community with 20.5.473 Qt6 runs relatively stable with minor bugs.* 


---

## 4. Troubleshooting Deadline Issues

1. **Deadline Monitor or Worker Fails to Start**
    - **Possible Cause:** `deadline-repo` may not be mounted correctly.
    - **Solution:** Verify that `deadline-repo` is mounted at `/mnt/VVOX-NAS-1/deadline-repo`. Remount if necessary.
2. **Incorrect Path Mapping Between Windows and Linux**
    - **Cause:** Mapped Paths in `Configure Repository Options` are not set up correctly.
    - **Solution:** Ensure the following mapping is configured:
        - From `\\vvox-nas-1\projects\` ➜ To `/mnt/VVOX-NAS-1/projects/`
        - _Note:_ Changes may take a few minutes to take effect.
3. **Worker Displays as "Stalled"**
    - **Solutions:** While restart can solve the issue, there's an even simpler way:
        ```
        sudo systemctl restart systemd-resolved
        ```
        
