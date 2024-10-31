# linux-renderfarm-docs

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
sudo mount -t cifs -o username=boning,password=follow-HARD-pair-short-2,uid=1000,gid=1000,dir_mode=0777,file_mode=0777,vers=3.0 //10.0.10.11/PROJECTS /mnt/VVOX-NAS-1/projects
sudo mount -t cifs -o username=boning,password=follow-HARD-pair-short-2,uid=1000,gid=1000,dir_mode=0777,file_mode=0777,vers=3.0 //10.0.10.11/deadline10-repo /mnt/VVOX-NAS-1/deadline-repo
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
//10.0.10.11/PROJECTS /mnt/VVOX-NAS-1/projects cifs username=boning,password=follow-HARD-pair-short-2,uid=1000,gid=1000,dir_mode=0755,file_mode=0755 0 0
//10.0.10.11/deadline10-repo /mnt/VVOX-NAS-1/deadline-repo cifs username=boning,password=follow-HARD-pair-short-2,uid=1000,gid=1000,dir_mode=0755,file_mode=0755 0 0
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
#### Dependencies:
Install these dependencies before launching Houdini:
```
sudo apt install libgl1 libglu1-mesa libglx0 libice6 libopengl0 libsm6 libx11-6 libx11-xcb1 libxcb-icccm4 libxcb-image0 libxcb-keysyms1 libxcb-randr0 libxcb-render-util0 libxcb-render0 libxcb-shape0 libxcb-shm0 libxcb-sync1 libxcb-util1 libxcb-xfixes0 libxcb-xinerama0 libxcb-xkb1 libxcb1 libxext6 libxkbcommon-x11-0 libxkbcommon0 libxt6
```
#### Installing HQueue Client
Current HQueue server and port is : http://10.0.10.203:5000 (Box-1) 
#### Environment Variables:
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
ASSETS="\\Vvox-nas-1\projects\_____ASSETS"
```
**Other useful environment variables:**
```
#Disable CPU for XPU
KARMA_XPU_DISABLE_EMBREE_DEVICE=1

#Disable Specific GPU for XPU 
KARMA_XPU_DISABLE_DEVICE_n=1

#Optimize Multi-GPU Performance
KARMA_XPU_NUM_PER_DEVICE_BLENDING_THREADS = 4

#Use Box1 as HQueue Server
HOUDINI_HQUEUE_SERVER = http://10.0.10.203:5000/ 
```


---

## 4. Troubleshooting Deadline Issues

1. **Problem:** Deadline Monitor or Worker Fails to Start
    - **Possible Cause:** `deadline-repo` may not be mounted correctly.
    - **Solution:** Verify that `deadline-repo` is mounted at `/mnt/VVOX-NAS-1/deadline-repo`. Remount if necessary.
2. **Problem:** Incorrect Path Mapping Between Windows and Linux
    - **Cause:** Mapped Paths in `Configure Repository Options` are not set up correctly.
    - **Solution:** Ensure the following mapping is configured:
        - From `\\vvox-nas-1\projects\` ➜ To `/mnt/VVOX-NAS-1/projects/`
        - _Note:_ Changes may take a few minutes to take effect.
3. **Problem:** Worker Displays as "Stalled"
    - **Possible Causes:**
        - Worker process is not running.
        - System issues causing the worker to stop unexpectedly.
    - **Solutions:**
        - Log in to the machine to check if the worker is active. Manually start it if needed.
        - If the worker fails to launch, try restarting the system and check again.
