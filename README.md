# 🚀 Fixing VMware Kernel Module Compilation Error (VMware Workstation 17 Pro - 17.5.2)

If you're facing the **`dev_base_lock` undeclared error** while compiling VMware Workstation modules on **Linux Kernel 6.11.x**, this guide will help you patch the source code and rebuild the modules.

## ✅ My Environment  
- **VMware Workstation 17 Pro** - **17.5.2 build-23775571**  
- **Kernel:** `6.11.0-109017-tuxedo`  

---

## 🔥 **Error in `vmware_errors.log`**
If you see in the log file (`/tmp/vmware-$USER/vmware-*.log` - e.g., `vmware-155342.log`):

```bash
make: *** [Makefile:117: vmnet.ko] Error 2
Unable to install all modules.  See log for details.
```

1️⃣ Find an alternative lock (net_rwsem or netdev_chain_mutex).
2️⃣ Edit vmnetInt.h, replacing dev_base_lock with the correct lock.
3️⃣ Rebuild the VMware kernel modules (vmnet.tar) and move it to /usr/lib/vmware/modules/source/.
4️⃣ Recompile and install the modules.

1️⃣ Identify Available Locks in Your Kernel

Run the following command to check available rw_semaphore (rwsem) locks in your kernel headers:

grep -r "rwsem" /usr/src/linux-headers-$(uname -r)/include/linux/


!!! If devnet_rename_rwsem does not appear, it means this lock is not present in your kernel version. Instead, try searching for alternative locks:

grep -r "lock" /usr/src/linux-headers-$(uname -r)/include/linux/
Look for network-related locks such as net_rwsem or netdev_chain_mutex.

In my case it was netdev_chain_mutex.

2️⃣  Edit vmnetInt.h to Replace dev_base_lock
1) Find your file vmnetInt.h:
sudo find / -type f -name vmnetInt.h 2>/dev/null

in my case
/home/USER/Downloads/vmware-host-modules-workstation-17.5.1/vmnet-only/vmnetInt.h

"ALWAYS MAKE A SAFETY COPY!" p.e. vmnetInt.h.old

2) Open the affected file for editing:
sudo nano /home/USER/Downloads/vmware-host-modules-workstation-17.5.1/vmnet-only/vmnetInt.h

3) Locate the following lines:

#define dev_lock_list()    read_lock(&dev_base_lock)
#define dev_unlock_list()  read_unlock(&dev_base_lock)

4) Replace them with: 
#define dev_lock_list()    mutex_lock(&netdev_chain_mutex)
#define dev_unlock_list()  mutex_unlock(&netdev_chain_mutex)

Save the file

3️⃣ Rebuild VMware Kernel Modules

1) Navigate to the VMware modules directory:

(in my case)
cd /home/USER/Downloads/vmware-host-modules-workstation-17.5.1

2) Create a new compressed archive with the modified files:

tar -cf vmnet.tar vmnet-only

3) Move it to the VMware source directory:

sudo mv vmnet.tar /usr/lib/vmware/modules/source/

4) Extract the updated archive:

cd /usr/lib/vmware/modules/source/
sudo tar -xf vmnet.tar
cd vmnet-only


5) Compile the patched module:

sudo make

4️⃣ Install and Load VMware Modules

1) Run the VMware module installer:
sudo vmware-modconfig --console --install-all

2) Check if the modules were installed successfully:
lsmod | grep vm

"✅ If vmmonandvmnet appear in the list, you successfully installed the modules!"
