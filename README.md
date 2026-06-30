### RHCSA Exercises

1. Recovery and Practise Tasks

    * Recover the system and fix repositories:
        ```shell
        # press e at grub menu
        rd.break # add to line starting with "linux16"
        # Replace line containing "BAD" with "x86_64"
        mount -o remount, rw /sysroot
        chroot /sysroot
        passwd
        touch /.autorelabel
        # reboot
        # reboot - will occur automaticaly after relabel (you can now login)
        grub2-mkconfig -o /boot/grub2/grub.cfg # fix grub config
        yum repolist all
        # change files in /etc/yum.repos.d to enable repository
        yum update -y
        # reboot
        ```

    * Add 3 new users alice, bob and charles. Create a marketing group and add these users to the group. Create a directory `/marketing` and change the owner to alice and group to marketing. Set permissions so that members of the marketing group can share documents in the directory but nobody else can see them. Give charles read-only permission. Create an empty file in the directory: 
        ```shell
        useradd alice
        useradd bob
        useradd charles
        groupadd marketing
        mkdir /marketing
        usermod -aG marketing alice
        usermod -aG marketing bob
        usermod -aG marketing charles
        chgrp marketing marketing # may require restart to take effect
        chmod 770 marketing
        setfacl -m u:charles:r marketing
        setfacl -m g:marketing:-wx marketing
        touch file
        ```

    * Set the system time zone and configure the system to use NTP:
        ```shell
        yum install chrony
        systemctl enable chronyd.service
        systemctl start chronyd.service
        timedatectl set-timezone Australia/Sydney
        timedatectl set-ntp true
        ```

    * Install and enable the GNOME desktop:
        ```shell
        yum grouplist
        yum groupinstall "GNOME Desktop" -y
        systemtctl set-default graphical.target
        reboot
        ```

    * Configure the system to be an NFS client:
        ```shell
        yum install nfs-utils
        ```

    * Configure password aging for charles so his password expires in 60 days:
        ```shell
        chage -M 60 charles
        chage -l charles # to confirm result
        ```

    * Lock bobs account:
        ```shell
        passwd -l bob
        passwd --status bob # to confirm result
        ```

    * Find all setuid files on the system and save the list to `/testresults/setuid.list`:
        ```shell
        find / -perm /4000 > setuid.list
        ```

    * Set the system FQDN to *centos.local* and alias to *centos*:
        ```shell
        hostnamectl set-hostname centos --pretty
        hostnamectl set-hostname centos.local
        hostname -s # to confirm result
        hostname # to confirm result
        ```

    * As charles, create a once-off job that creates a file called `/testresults/bob` containing the text "Hello World. This is Charles." in 2 days later:
        ```shell
        vi hello.sh
        # contents of hello.sh
        #####
        #!/bin/bash
        # echo "Hello World. This is Charles." > bob
        #####
        chmod 755 hello.sh
        usermod charles -U -e -- "" # for some reason the account was locked
        at now + 2 days -f /testresults/bob/hello.sh
        cd /var/spool/at # can check directory as root to confirm
        atq # check queued job as charles
        # atrm 1 # can remove the job using this command
        ```

    * As alice, create a periodic job that appends the current date to the file `/testresults/alice` every 5 minutes every Sunday and Wednesday between the hours of 3am and 4am. Remove the ability of bob to create cron jobs:
        ```shell
        echo "bob" >> /etc/at.deny
        sudo -i -u alice
        vi addDate.sh
        # contents of hello.sh
        #####
        ##!/bin/bash
        #date >> alice
        #####
        /testresults/alice/addDate.sh
        crontab -e
        # */5 03,04 * * sun,wed /testresults/alice/addDate.sh
        crontab -l # view crontab
        # crontab -r can remove the job using this command
        ```

    * Set the system SELinux mode to permissive:
        ```shell
        setstatus # confirm current mode is not permissive
        vi /etc/selinux/config # Update to permissive
        reboot
        setstatus # confirm current mode is permissive
        ```

    * Create a firewall rule to drop all traffic from 10.10.10.*:
        ```shell
        firewall-cmd --zone=drop --add-source 10.10.10.0/24
        firewall-cmd --list-all --zone=drop # confirm rule is added
        firewall-cmd --permanent --add-source 10.10.10.0/24
        reboot
        firewall-cmd --list-all --zone=drop # confirm rule remains
        ```

1. Linux Academy - Using SSH, Redirection, and Permissions in Linux

    * Enable SSH to connect without a password from the dev user on server1 to the dev user on server2:
        ```shell
        ssh dev@3.85.167.210
        ssh-keygen # created in /home/dev/.ssh
        ssh-copy-id 34.204.14.34
        ```    

    * Copy all tar files from `/home/dev/` on server1 to `/home/dev/` on server2, and extract them making sure the output is redirected to `/home/dev/tar-output.log`:
        ```shell
        scp *.tar* dev@34.204.14.34:/home/dev
        tar xfz deploy_script.tar.gz > tar-output.log
        tar xfz deploy_content.tar.gz >> tar-output.log
        ```
    
    * Set the umask so that new files are only readable and writeable by the owner:
        ```shell
        umask 0066 # default is 0666, subtract 0066 to get 0600
        ```

    * Verify the `/home/dev/deploy.sh` script is executable and run it:
        ```shell
        chmod 711 deploy.sh
        ./deploy.sh
        ```

1. Linux Academy - Storage Management

    * Create a 2GB GPT Partition:
        ```shell
        lsblk # observe nvme1n1 disk
        sudo gdisk /dev/nvme1n1
        # enter n for new partition
        # accept default partition number
        # accept default starting sector
        # for the ending sector, enter +2G to create a 2GB partition
        # accept default partition type
        # enter w to write the partition information
        # enter y to proceed
        lsblk # observe nvme1n1 now has partition
        partprobe # inform OS of partition change
        ```

    * Create a 2GB MBR Partition:
        ```shell
        lsblk # observe nvme2n1 disk
        sudo fdisk /dev/nvme2n1
        # enter n for new partition
        # accept default partition type
        # accept default partition number
        # accept default first sector
        # for the ending sector, enter +2G to create a 2GB partition
        # enter w to write the partition information
        ```

    * Format the GPT Partition with XFS and mount the device persistently:
        ```shell
        sudo mkfs.xfs /dev/nvme1n1p1
        sudo blkid # observe nvme1n1p1 UUID
        vi /etc/fstab
        # add a line with the new UUID and specify /mnt/gptxfs
        mkdir /mnt/gptxfs
        sudo mount -a
        mount # confirm that it's mounted
        ```

    * Format the MBR Partition with ext4 and mount the device persistently:
        ```shell
        sudo mkfs.ext4 /dev/nvme2n1p1
        mkdir /mnt/mbrext4
        mount /dev/nvme2n1p1 /mnt/mbrext4
        mount # confirm that it's mounted
        ```

1. Linux Academy - Working with LVM Storage

    * Create Physical Devices:
        ```shell
        lsblk # observe disks xvdf and xvdg
        pvcreate /dev/xvdf /dev/xvdg
        ```

    * Create Volume Group:
        ```shell
        vgcreate RHCSA /dev/xvdf /dev/xvdg
        vgdisplay # view details
        ```

    * Create a Logical Volume:
        ```shell
        lvcreate -n pinehead -L 3G RHCSA
        lvdisplay # or lvs, to view details
        ```

    * Format the LV as XFS and mount it persistently at `/mnt/lvol`:
        ```shell
        fdisk -l # get path for lv
        mkfs.xfs /dev/mapper/RHCSA-pinehead
        mkdir /mnt/lvol
        blkid # copy UUID for /dev/mapper/RHCSA-pinehead
        echo "UUID=76747796-dc33-4a99-8f33-58a4db9a2b59" >> /etc/fstab
        # add the path /mnt/vol and copy the other columns
        mount -a
        mount # confirm that it's mounted
        ```

    * Grow the mount point by 200MB:
        ```shell
        lvextend -L +200M /dev/RHCSA/pinehead
        ```

1. Linux Academy - Network File Systems

    * Set up a SAMBA share:
        ```shell
        # on the server
        yum install samba -y
        vi /etc/samba/smb.conf
        # add the below block
        #####
        #[share]
        #    browsable = yes
        #    path = /smb
        #    writeable = yes
        #####
        useradd shareuser
        smbpasswd -a shareuser # enter password
        mkdir /smb
        systemctl start smb
        chmod 777 /smb
        
        # on the client
        mkdir /mnt/smb
        yum install cifs-utils -y
        # on the server hostname -I shows private IP
        mount -t cifs //10.0.1.100/share /mnt/smb -o username=shareuser,password= # private ip used
        ```

    * Set up the NFS share:
        ```shell
        # on the server
        yum install nfs-utils -y
        mkdir /nfs
        echo "/nfs *(rw)" >> /etc/exports
        chmod 777 /nfs
        exportfs -a
        systemctl start {rpcbind,nfs-server,rpc-statd,nfs-idmapd}

        # on the client
        yum install nfs-utils -y
        mkdir /mnt/nfs
        showmount -e 10.0.1.101 # private ip used
        systemctl start rpcbind
        mount -t nfs 10.0.1.101:/nfs /mnt/nfs
        ```

1. Linux Academy - Maintaining Linux Systems

    * Schedule a job to update the server midnight tonight:
        ```shell
        echo "dnf update -y" > update.sh
        chmod +x update.sh
        at midnight -f update.sh
        atq # to verify that job is scheduled
        ```

    * Modify the NTP pools:
        ```shell
        vi /etc/chrony.conf
        # modify the pool directive at the top of the file
        ```

    * Modify GRUB to boot a different kernel:
        ```shell
        grubby --info=ALL # list installed kernels
        grubby --set-default-index=1
        grubby --default-index # verify it worked
        ```

1. Linux Academy - Managing Users in Linux

    * Create the superhero group:
        ```shell
        groupadd superhero
        ```

    * Add user accounts for Tony Stark, Diana Prince, and Carol Danvers and add them to the superhero group:
        ```shell
        useradd tstark -G superhero
        useradd cdanvers -G superhero
        useradd dprince -G superhero
        ```

    * Replace the primary group of Tony Stark with the wheel group:
        ```shell
        usermod tstark -ag wheel
        grep wheel /etc/group # to verify
        ```

    * Lock the account of Diana Prince:
        ```shell
        usermod -L dprince 
        chage dprince -E 0
        ```

1. Linux Academy - SELinux Learning Activity

    * Fix the SELinux permission on `/opt/website`:
        ```shell
        cd /var/www # the default root directory for a web server
        ls -Z # observe permission on html folder
        semanage fcontext -a -t httpd_sys_content_t '/opt/website(/.*)'
        restorecon /opt/website
        ```

    * Deploy the website and test:
        ```shell
        mv /root/index.html /opt/website
        curl localhost/index.html # receive connection refused response
        systemctl start httpd # need to start the service
        setenforce 0 # set to permissive to allow for now
        ```

    * Resolve the error when attempting to access `/opt/website`:
        ```shell
        ll -Z # notice website has admin_home_t
        restorecon /opt/website/index.html
        ```

1. Linux Academy - Setting up VDO

    * Install VDO and ensure the service is running:
        ```shell
        dnf install vdo -y
        systemctl start vdo && systemctl enable vdo
        ```

    * Setup a 100G VM storage volume:
        ```shell
        vdo create --name=ContainerStorage --device=/dev/nvme1n1 --vdoLogicalSize=100G --sparseIndex=disabled
        # spareIndex set to meet requirement of dense index deduplication
        mkfs.xfs -K /dev/mapper/ContainerStorage
        mkdir /mnt/containers
        mount /dev/mapper/ContainerStorage /mnt/containers
        vi /etc/fstab # add line /dev/mapper/ContainerStorage /mnt/containers xfs defaults,_netdev,x-systemd.device-timeout=0,x-systemd.requires=vdo.service 0 0
        ```

    * Setup a 60G website storage volume:
        ```shell
        vdo create --name=WebsiteStorage --device=/dev/nvme2n1 --vdoLogicalSize=60G --deduplication=disabled
        # deduplication set to meet requirement of no deduplication
        mkfs.xfs -K /dev/mapper/WebsiteStorage
        mkdir /mnt/website
        mount /dev/mapper/WebsiteFiles /mnt/website
        vi /etc/fstab # add line for /dev/mapper/WebsiteStorage /mnt/website xfs defaults,_netdev,x-systemd.device-timeout=0,x-systemd.requires=vdo.service 0 0
        ```

1. Linux Academy - Final Practise Exam

    * Start the guest VM:
        ```shell
        # use a VNC viewer connect to IP:5901
        virsh list --all
        virsh start --centos7.0
        # we already have the VM installed, we just needed to start it (so we don't need virt-install)
        dnf install virt-viewer -y
        virt-viewer centos7.0 # virt-manager can also be used
        # now we are connected to the virtual machine
        # send key Ctrl+Alt+Del when prompted for password, as we don't know it
        # press e on GRUB screen
        # add rd.break on the linux16 line
        # now at the emergency console
        mount -o remount, rw /sysroot
        chroot /sysroot
        passwd
        touch /.autorelabel
        reboot -f # needs -f to work for some reason
        # it will restart when it completes relabelling
        ```

    * Create three users (Derek, Tom, and Kenny) that belong to the instructors group. Prevent Tom's user from accessing a shell, and make his account expire 10 day from now:
        ```shell
        groupadd instructors
        useradd derek -G instructors
        useradd tom -s /sbin/nologin -G instructors
        useradd kenny -G instructors
        chage tom -E 2020-10-14
        chage -l tom # to check
        cat /etc/group | grep instructors # to check
        ```

    * Download and configure apache to serve index.html from `/var/web` and access it from the host machine:
        ```shell
        # there is some setup first to establish connectivity/repo
        nmcli device # eth0 shown as disconnected
        nmcli connection up eth0
        vi /etc/yum.repos.d/centos7.repo
        # contents of centos.repo
        #####
        #[centos7]
        #name = centos
        #baseurl = http://mirror.centos.org/centos/7/os/x86_64/
        #enabled = 1
        #gpgcheck = 1
        #gpgkey = file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
        #####
        yum repolist # confirm
        yum install httpd -y
        systemctl start httpd.service
        mkdir /var/web
        vi /etc/httpd/conf/httpd.conf
        # change DocumentRoot to "/var/web"
        # change Directory tag to "/var/web"
        # change Directory tag to "/var/web/html"
        echo "Hello world" > /var/web/index.html
        systemctl start httpd.service
        ip a s # note the first inet address for eth0 # from the guest VM
        curl http://192.168.122.213/ # from the host 
        # note that no route to host returned
        firewall-cmd --list-services # notice no http service
        firewall-cmd --add-service=http --permanent
        firewall-cmd --reload
        firewall-cmd --list-services # confirm http service
        curl http://192.168.122.255/ # from the host 
        # note that 403 error is returned
        # ll -Z comparision between /var/web and /var/www shows that the SELinux type of index.html should be httpd_sys_context_t and not var_t
        yum provides \*/semanage # suggests policycoreutils-python
        yum install policycoreutils-python -y
        semanage fcontext -a -t httpd_sys_content_t "/var/web(/.*)?"
        restorecon -R -v /var/web
        curl http://192.168.122.255/ # from the host - success!
        ```

    * Configure umask to ensure all files created by any user cannot be accessed by the "other" users:
        ```shell
        umask 0026 # also reflect change in /etc/profile and /etc/bashrc
        # default for files is 0666 so will be 0640 after mask
        ```

    * Find all files in `/etc` (not including subdirectories) that are older than 720 days, and output a list to `/root/oldfiles`:
        ```shell
        find /etc -maxdepth 1 -mtime +720 > /root/oldfiles 
        ```

    * Find all log messages in `/var/log/messages` that contain "ACPI", and export them to a file called `/root/logs`. Then archive all of `/var/log` and save it to `/tmp/log_archive.tgz`:
        ```shell
        grep "ACPI" /var/log/messages > /root/logs
        tar -czf /tmp/log_archive.tgz /var/log/ # note f flag must be last!
        ```

    * Modify the GRUB timeout and make it 1 second instead of 5 seconds:
        ```shell
        find / -iname grub.cfgreboot
        # /etc/grub.d, /etc/default/grub and grub2-mkconfig referred to in /boot/grub2/grub.cfg
        vi /etc/default/grub # change GRUB_TIMEOUT to 1
        grub2-mkconfig -o /boot/grub2/grub.cfg
        reboot # confirm timeout now 1 second
        ```

    * Create a daily cron job at 4:27PM for the Derek user that runs `cat /etc/redhat-release` and redirects the output to `/home/derek/release`:
        ```shell
        cd /home/derek
        vi script.sh
        # contents of script.sh
        #####
        ##!/bin/sh
        #cat /etc/redhat-release > /home/derek/release
        #####
        chmod +x script.sh
        crontab -u derek -e
        # contents of crontab
        #####
        #27 16 * * * /home/derek/script.sh
        #####
        crontab -u derek -l # confirm
        ```

    * Configure `time.nist.gov` as the only NTP Server:
        ```shell
        vi /etc/chrony.conf
        # replace lines at the top with server time.nist.gov
        ```

    * Create an 800M swap partition on the `vdb` disk and use the UUID to ensure that it is persistent:
        ```shell
        fdisk -l # note that we have one MBR partitions
        fdisk /dev/vdb
        # select n
        # select p
        # select default
        # select default
        # enter +800M
        # select w
        partprobe
        lsblk # confirm creation
        mkswap /dev/vdb1
        vi /etc/fstab
        # add line containing UUID and swap for the next 2 columns
        swapon -a
        swap # confirm swap is available
        ```

    * Create a new logical volume (LV-A) with a size of 30 extents that belongs to the volume group VG-A (with a PE size of 32M). After creating the volume, configure the server to mount it persistently on `/mnt`:
        ```shell
        # observe through fdisk -l and df -h that /dev/vdc is available with no file system
        yum provides pvcreate # lvm2 identified
        yum install lvm2 -y
        pvcreate /dev/vdc
        vgcreate VG-A /dev/vdc -s 32M
        lvcreate -n LV-A -l 30 VG-A
        mkfs.xfs /dev/VG-A/LV-A
        # note in directory /dev/mapper the name is VG--A-LV--A
        # add an entry to /etc/fstab at /dev/mapper/VG--A-LV--A and /mnt (note that you can mount without the UUID here)
        mount -a
        df -h # verify that LV-A is mounted
        ```

    * On the host, not the guest VM, utilise ldap.linuxacademy.com for SSO, and configure AutoFS to mount user's home directories on login. Make sure to use Kerberos:
        ```shell
        # this objective is no longer required in RHCSA 8
        ```

    * Change the hostname of the guest to "RHCSA":
        ```shell
        hostnamectl set-hostname rhcsa
        ```

1. Asghar Ghori - Exercise 3-1: Create Compressed Archives

    * Create tar files compressed with gzip and bzip2 and extract them:
        ```shell
        # gzip
        tar -czf home.tar.gz /home
        tar -tf /home.tar.gz # list files
        tar -xf home.tar.gz

        # bzip
        tar -cjf home.tar.bz2 /home
        tar -xf home.tar.bz2 -C /tmp
        ```

1. Asghar Ghori - Exercise 3-2: Create and Manage Hard Links

    * Create an empty file *hard1* under */tmp* and display its attributes. Create hard links *hard2* and *hard3*. Edit *hard2* and observe the attributes. Remove *hard1* and *hard3* and list the attributes again:
        ```shell
        touch hard1
        ln hard1 hard2
        ln hard1 hard3
        ll -i
        # observe link count is 3 and same inode number
        echo "hi" > hard2
        # observe file size increased to the same value for all files
        rm hard1
        rm hard3
        # observe link count is 1
        ```

1. Asghar Ghori - Exercise 3-3: Create and Manage Soft Links

    * Create an empty file *soft1* under `/root` pointing to `/tmp/hard2`. Edit *soft1* and list the attributes after editing. Remove *hard2* and then list *soft1*:
        ```shell
        ln -s /tmp/hard2 soft1
        ll -i
        # observe soft1 and hard2 have the same inode number
        echo "hi" >> soft1
        # observe file size increased
        cd /root
        ll -i 
        # observe the soft link is now broken
        ```

1. Asghar Ghori - Exercise 4-1: Modify Permission Bits Using Symbolic Form

    * Create a file *permfile1* with read permissions for owner, group and other. Add an execute bit for the owner and a write bit for group and public. Revoke the write bit from public and assign read, write, and execute bits to the three user categories at the same time. Revoke write from the owning group and write, and execute bits from public:
        ```shell
        touch permfile1
        chmod 444 permfile1
        chmod -v u+x,g+w,o+w permfile1
        chmod -v o-w,a=rwx permfile1
        chmod -v g-w,o-wx permfile1
        ```

1. Asghar Ghori - Exercise 4-2: Modify Permission Bits Using Octal Form

    * Create a file *permfile2* with read permissions for owner, group and other. Add an execute bit for the owner and a write bit for group and public. Revoke the write bit from public and assign read, write, and execute permissions to the three user categories at the same time:
        ```shell
        touch permfile2
        chmod 444 permfile2
        chmod -v 566 permfile2
        chmod -v 564 permfile2
        chmod -v 777 permfile2
        ```

1. Asghar Ghori - Exercise 4-3: Test the Effect of setuid Bit on Executable Files

    * As root, remove the setuid bit from `/usr/bin/su`. Observe the behaviour for another user attempting to switch into root, and then add the setuid bit back:
        ```shell
        chmod -v u-s /usr/bin/su
        # users now receive authentication failure when attempting to switch
        chmod -v u+s /usr/bin/su
        ```

1. Asghar Ghori - Exercise 4-4: Test the Effect of setgid Bit on Executable Files

    * As root, remove the setgid bit from `/usr/bin/write`. Observe the behaviour when another user attempts to run this command, and then add the setgid bit back:
        ```shell
        chmod -v g-s /usr/bin/write
        # Other users can no longer write to root
        chmod -v g+s /usr/bin/write
        ```

1. Asghar Ghori - Exercise 4-5: Set up Shared Directory for Group Collaboration

    * Create users *user100* and *user200*. Create a group *sgrp* with GID 9999 and add *user100* and *user200* to this group. Create a directory `/sdir` with ownership and owning groups belong to *root* and *sgrp*, and set the setgid bit on */sdir* and test:
        ```shell
        groupadd sgrp -g 9999
        useradd user100 -G sgrp 
        useradd user200 -G sgrp 
        mkdir /sdir
        chown root:sgrp sdir
        chmod g+s,g+w sdir
        # as user100
        cd /sdir
        touch file
        # owning group is sgrp and not user100 due to setgid bit
        # as user200
        vi file
        # user200 can also read and write
        ```

1. Asghar Ghori - Exercise 4-6: Test the Effect of Sticky Bit

    * Create a file under `/tmp` as *user100* and try to delete it as *user200*. Unset the sticky bit on `/tmp` and try to erase the file again. Restore the sticky bit on `/tmp`:
        ```shell
        # as user100
        touch /tmp/myfile
        # as user200
        rm /tmp/myfile
        # cannot remove file: Operation not permitted
        # as root
        chmod -v o-t tmp
        # as user200
        rm /tmp/myfile
        # file can now be removed
        # as root
        chmod -v o+t tmp
        ```

1. Asghar Ghori - Exercise 4-7: Identify, Apply, and Erase Access ACLs

    * Create a file *acluser* as *user100* in `/tmp` and check if there are any ACL settings on the file. Apply access ACLs on the file for *user100* for read and write access. Add *user200* to the file for full permissions. Remove all access ACLs from the file:
        ```shell
        # as user100
        touch /tmp/acluser
        cd /tmp
        getfacl acluser
        # no ACLs on the file
        setfacl -m u:user100:rw,u:user200:rwx acluser
        getfacl acluser
        # ACLs have been added
        setfacl -x user100,user200 acluser
        getfacl acluser
        # ACLs have been removed
        ```

1. Asghar Ghori - Exercise 4-8: Apply, Identify, and Erase Default ACLs

    * Create a directory *projects* as *user100* under `/tmp`. Set the default ACLs on the directory for *user100* and *user200* to give them full permissions. Create a subdirectory *prjdir1* and a file *prjfile1* under *projects* and observe the effects of default ACLs on them. Delete the default entries:
        ```shell
        # as user100
        cd /tmp
        mkdir projects
        getfacl projects
        # No default ACLs for user100 and user200
        setfacl -dm u:user100:rwx,u:user200:rwx projects
        getfacl projects
        # Default ACLs added for user100 and user200
        mkdir projects/prjdir1
        getfacl prjdir1
        # Default ACLs inherited
        touch prjdir1/prjfile1
        getfacl prjfile1
        # Default ACLs inherited
        setfacl -k projects
        ```

1. Asghar Ghori - Exercise 5-1: Create a User Account with Default Attributes

    * Create *user300* with the default attributes in the *useradd* and *login.defs* files. Assign this user a password and show the line entries from all 4 authentication files:
        ```shell
        useradd user300
        passwd user300
        grep user300 /etc/passwd /etc/shadow /etc/group /etc/gshadow
        ```


1. Asghar Ghori - Exercise 5-2: Create a User Account with Custom Values

    * Create *user300* with the default attributes in the *useradd* and *login.defs* files. Assign this user a password and show the line entries from all 4 authentication files:
        ```shell
        useradd user300
        passwd user300
        grep user300 /etc/passwd /etc/shadow /etc/group /etc/gshadow
        ```

1. Asghar Ghori - Exercise 5-3: Modify and Delete a User Account

    * For *user200* change the login name to *user200new*, UID to 2000, home directory to `/home/user200new`, and login shell to `/sbin/nologin`. Display the line entry for *user2new* from the *passwd* for validation. Remove this user and confirm the deletion:
        ```shell
        usermod -l user200new -m -d /home/user200new -s /sbin/nologin -u 2000 user200
        grep user200new /etc/passwd # confirm updated values
        userdel -r user200new
        grep user200new /etc/passwd # confirm user200new deleted
        ```

1. Asghar Ghori - Exercise 5-4: Create a User Account with No-Login Access

    * Create an account *user400* with default attributes but with a non-interactive shell. Assign this user the nologin shell to prevent them from signing in. Display the new line entry frmo the *passwd* file and test the account:
        ```shell
        useradd user400 -s /sbin/nologin
        passwd user400 # change password
        grep user400 /etc/passwd
        sudo -i -u user400 # This account is currently not available
        ```

1. Asghar Ghori - Exercise 6-1: Set and Confirm Password Aging with chage

    * Configure password ageing for user100 using the *chage* command. Set the mindays to 7, maxdays to 28, and warndays to 5. Verify the new settings. Rerun the command and set account expiry to January 31, 2020:
        ```shell
        chage -m 7 -M 28 -W 5 user100
        chage -l user100
        chage -E 2021-01-31 user100
        chage -l
        ```

1. Asghar Ghori - Exercise 6-2: Set and Confirm Password Aging with passwd

    * Configure password aging for *user100* using the *passwd* command. Set the mindays to 10, maxdays to 90, and warndays to 14, and verify the new settings. Set the number of inactivity days to 5 and ensure that the user is forced to change their password upon next login:
        ```shell
        passwd -n 10 -x 90 -w 14 user100
        passwd -S user100 # view status
        passwd -i 5 user100
        passwd -e user100
        passwd -S user100
        ```

1. Asghar Ghori - Exercise 6-3: Lock and Unlock a User Account with usermod and passwd

    * Disable the ability of user100 to log in using the *usermod* and *passwd* commands. Verify the change and then reverse it:
        ```shell
        grep user100 /etc/shadow # confirm account not locked by absence of "!" in password
        passwd -l user100 # usermod -L also works
        grep user100 /etc/shadow
        passwd -u user100 # usermod -U also works
        ```

1. Asghar Ghori - Exercise 6-4: Create a Group and Add Members

    * Create a group called *linuxadm* with GID 5000 and another group called *dba* sharing the GID 5000. Add *user100* as a secondary member to group *linxadm*:
        ```shell
        groupadd -g 5000 linuxadm
        groupadd -o -g 5000 dba # note need -o to share GID
        usermod -G linuxadm user100
        grep user100 /etc/group # confirm user added to group
        ```

1. Asghar Ghori - Exercise 6-5: Modify and Delete a Group Account

    * Change the *linuxadm* group name to *sysadm* and the GID to 6000. Modify the primary group for user100 to *sysadm*. Remove the *sysadm* group and confirm:
        ```shell
        groupmod -n sysadm -g 6000 linuxadm
        usermod -g sysadm user100
        groupdel sysadm # can't remove while user100 has as primary group
        ```

1. Asghar Ghori - Exercise 6-6: Modify File Owner and Owning Group

    * Create a file *file10* and a directory *dir10* as *user200* under `/tmp`, and then change the ownership for *file10* to *user100* and the owning group to *dba* in 2 separate transactions. Apply ownership on *file10* to *user200* and owning group to *user100* at the same time. Change the 2 attributes on the directory to *user200:dba* recursively:
        ```shell
        # as user200
        mkdir /tmp/dir10
        touch /tmp/file10
        sudo chown user100 /tmp/file10         
        sudo chgrp dba /tmp/file10
        sudo chown user200:user100 /tmp/file10
        sudo chown -R user200:user100 /tmp/dir10
        ```

1. Asghar Ghori - Exercise 7-1: Modify Primary Command Prompt

    * Customise the primary shell prompt to display the information enclosed within the quotes "\<username on hostname in pwd\>:" using variable and command substitution. Edit the `~/.profile`file for *user100* and define the new value in there for permanence:
        ```shell
        export PS1="< $LOGNAME on $(hostname) in \$PWD>"
        # add to ~/.profile for user100
        ```

1. Asghar Ghori - Exercise 8-1: Submit, View, List, and Remove an at Job

    * Submit a job as *user100* to run the *date* command at 11:30pm on March 31, 2021, and have the output and any error messages generated redirected to `/tmp/date.out`. List the submitted job and then remove it:
        ```shell
        # as user100
        at 11:30pm 03/31/2021
        # enter "date &> /tmp/date.out"
        atq # view job in queue
        at -c 1 # view job details
        atrm 1 # remove job
        ```

1. Asghar Ghori - Exercise 8-2: Add, List, and Remove a Cron Job

    * Assume all users are currently denied access to cron. Submit a cron job as *user100* to echo "Hello, this is a cron test.". Schedule this command to execute at every fifth minute past the hour between 10:00 am and 11:00 am on the fifth and twentieth of every month. Have the output redirected to `/tmp/hello.out`. List the cron entry and then remove it:
        ```shell
        # as root
        echo "user100" > /etc/cron.allow
        # ensure cron.deny is empty
        # as user100
        crontab
        # */5 10,11 5,20 * * echo "Hello, this is a cron test." >> /tmp/hello.out
        crontab -e # list
        crontab -l # remove
        ```

1. Asghar Ghori - Exercise 9-1: Perform Package Management Tasks Using rpm

    * Verify the integrity and authenticity of a package called *dcraw* located in the `/mnt/AppStream/Packages` directory on the installation image and then install it. Display basic information about the package, show files it contains, list documentation files, verify the package attributes and remove the package: 
        ```shell
        ls -l /mnt/AppStream/Packages/dcraw*
        rpmkeys -K /mnt/AppStream/Packages/dcraw-9.27.0-9.e18.x86_64.rpm # check integrity
        sudo rpm -ivh /mnt/AppStream/Packages/dcraw-9.27.0-9.e18.x86_64.rpm # -i is install, -v is verbose and -h is hash
        rpm -qi dcraw # -q is query and -i is install
        rpm -qd dcraw # -q is query and -d is docfiles
        rpm -Vv dcraw # -V is verify and -v is verbose
        sudo rpm -ve # -v is verbose and -e is erase
        ```

1. Asghar Ghori - Exercise 10-1: Configure Access to Pre-Built ISO Repositories

    * Access the repositories that are available on the RHEL 8 image. Create a definition file for the repositories and confirm:
        ```shell
        df -h # after mounting optical drive in VirtualBox
        vi /etc/yum.repos.d/centos.local
        # contents of centos.local
        #####
        #[BaseOS]
        #name=BaseOS
        #baseurl=file:///run/media/$name/BaseOS
        #gpgcheck=0
        #
        #[AppStream]
        #name=AppStream
        #baseurl=file:///run/media/$name/AppStream
        #gpgcheck=0
        #####
        dnf repolist # confirm new repos are added
        ```

1. Asghar Ghori - Exercise 10-2: Manipulate Individual Packages

    * Determine if the *cifs-utils* package is installed and if it is available for installation. Display its information before installing it. Install the package and display its information again. Remove the package along with its dependencies and confirm the removal:
        ```shell
        dnf config-manager --disable AppStream
        dnf config-manager --disable BaseOS
        dnf list installed | greps cifs-utils # confirm not installed
        dnf info cifs-utils # display information
        dnf install cifs-utils -y
        dnf info cifs-utils # Repository now says @System
        dnf remove cifs-utils -y
        ```

1. Asghar Ghori - Exercise 10-3: Manipulate Package Groups

    * Perform management operations on a package group called *system tools*. Determine if this group is already installed and if it is available for installation. List the packages it contains and install it. Remove the group along with its dependencies and confirm the removal:
        ```shell
        dnf group list # shows System Tools as an available group
        dnf group info "System Tools"
        dnf group install "System Tools" -y
        dnf group list "System Tools" # shows installed
        dnf group remove "System Tools" -y
        ```

1. Asghar Ghori - Exercise 10-4: Manipulate Modules

    * Perform management operations on a module called *postgresql*. Determine if this module is already installed and if it is available for installation. Show its information and install the default profile for stream 10. Remove the module profile along with any dependencies and confirm its removal:
        ```shell
        dnf module list "postgresql" # no [i] tag shown so not installed
        dnf module info postgresql:10 # note there are multiple streams
        sudo dnf module install --profile postgresql:10 -y
        dnf module list "postgresql" # [i] tag shown so it's installed
        sudo dnf module remove postgresql:10 -y
        ```

1. Asghar Ghori - Exercise 10-5: Install a Module from an Alternative Stream

    * Downgrade a module to a lower version. Remove the stream *perl* 5.26 and confirm its removal. Manually enable the stream *perl* 5.24 and confirm its new status. Install the new version of the module and display its information:
        ```shell
        dnf module list perl # 5.26 shown as installed
        dnf module remove perl -y
        dnf module reset perl # make no version enabled
        dnf module install perl:5.26/minimal --allowerasing
        dnf module list perl # confirm module installed
        ```

1. Asghar Ghori - Exercise 11-1: Reset the root User Password

    * Terminate the boot process at an early stage to access a debug shell to reset the root password:
        ```shell
        # add rd.break affter "rhgb quiet" to reboot into debug shell
        mount -o remount, rw /sysroot
        chroot /sysroot
        passwd # change password
        touch /.autorelabel
        ```

1. Asghar Ghori - Exercise 11-2: Download and Install a New Kernel

    * Download the latest available kernel packages from the Red Hat Customer Portal and install them:
        ```shell
        uname -r # view kernel version
        rpm -qa | grep "kernel"
        # find versions on access.redhat website, download and move to /tmp
        sudo dnf install /tmp/kernel* -y
        ```

1. Asghar Ghori - Exercise 12-1: Manage Tuning Profiles

    * Install the *tuned* service, start it and enable it for auto-restart upon reboot. Display all available profiles and the current active profile. Switch to one of the available profiles and confirm. Determine the recommended profile for the system and switch to it. Deactive tuning and reactivate it:
        ```shell
        sudo systemctl status tuned-adm # already installed and enabled
        sudo tuned-adm profile # active profile is virtual-guest
        sudo tuned-adm profile desktop # switch to desktop profile
        sudo tuned-adm profile recommend # virtual-guest is recommended
        sudo tuned-adm off # turn off profile
        ```

1. Asghar Ghori - Exercise 13-1: Add Required Storage to server2

    * Add 4x250MB, 1x4GB, and 2x1GB disks:
        ```shell
        # in virtual box add a VDI disk to the SATA controller
        lsblk # added disks shown as sdb, sdc, sdd
        ```

1. Asghar Ghori - Exercise 13-2: Create an MBR Partition

    * Assign partition type "msdos" to `/dev/sdb` for using it as an MBR disk. Create and confirm a 100MB primary partition on the disk:
        ```shell
        parted /dev/sdb print # first line shows unrecognised disk label
        parted /dev/sdb mklabel msdos
        parted /dev/sdb mkpart primary 1m 101m
        parted /dev/sdb print # confirm added partition
        ```

1. Asghar Ghori - Exercise 13-3: Delete an MBR Partition

    * Delete the *sdb1* partition that was created in Exercise 13-2 above:
        ```shell
        parted /dev/sdb rm 1
        parted /dev/sdb print # confirm deletion
        ```

1. Asghar Ghori - Exercise 13-4: Create a GPT Partition

    * Assign partition type "gpt" to `/dev/sdc` for using it as a GPT disk. Create and confirm a 200MB partition on the disk:
        ```shell
        gdisk /dev/sdc
        # enter n for new
        # enter default partition number
        # enter default first sector
        # enter +200MB for last sector
        # enter default file system type
        # enter default hex code
        # enter w to write
        lsblk # can see sdc1 partition with 200M
        ```

1. Asghar Ghori - Exercise 13-5: Delete a GPT Partition

    * Delete the *sdc1* partition that was created in Exercise 13-4 above:
        ```shell
        gdisk /dev/sdc
        # enter d for delete
        # enter w to write
        lsblk # can see no partitions under sdc
        ```

1. Asghar Ghori - Exercise 13-6: Install Software and Activate VDO

    * Install the VDO software packages, start the VDO services, and mark it for autostart on subsequent reboots:
        ```shell
        dnf install vdo kmod-kvdo -y
        systemctl start vdo.service & systemctl enable vdo.service
        ```

1. Asghar Ghori - Exercise 13-7: Create a VDO Volume

    * Create a volume called *vdo-vol1* of logical size 16GB on the `/dev/sdc` disk (the actual size of `/dev/sdc` is 4GB). List the volume and display its status information. Show the activation status of the compression and de-duplication features:
        ```shell
        wipefs -a /dev/sdc # couldn't create without doing this first
        vdo create --name vdo-vol1 --device /dev/sdc --vdoLogicalSize 16G --vdoSlabSize 128
        # VDO instance 0 volume is ready at /dev/mapper/vdo-vol1
        lsblk # confirm vdo-vol1 added below sdc
        vdo list # returns vdo-vol1
        vdo status --name vdo-vol1 # shows status
        vdo status --name vdo-vol1 | grep -i "compression" # enabled
        vdo status --name vdo-vol1 | grep -i "deduplication" # enabled
        ```

1. Asghar Ghori - Exercise 13-8: Delete a VDO Volume

    * Delete the *vdo-vol1* volume that was created in Exercise 13-7 above and confirm the removal:
        ```shell
        vdo remove --name vdo-vol1
        vdo list # confirm removal
        ```

1. Asghar Ghori - Exercise 14-1: Create a Physical Volume and Volume Group

    * Initialise one partition *sdd1* (90MB) and one disk *sdb* (250MB) for use in LVM. Create a volume group called *vgbook* and add both physical volumes to it. Use the PE size of 16MB and list and display the volume group and the physical volumes:
        ```shell
        parted /dev/sdd mklabel msdos
        parted /dev/sdd mkpart primary 1m 91m
        parted /dev/sdd set 1 lvm on
        pvcreate /dev/sdd1 /dev/sdb
        vgcreate -vs 16 vgbook /dev/sdd1 /dev/sdb
        vgs vgbook # list information about vgbook
        vgdisplay -v vbook # list detailed information about vgbook
        pvs # list information about pvs
        ```

1. Asghar Ghori - Exercise 14-2: Create Logical Volumes

    * Create two logical volumes, *lvol0* and *lvbook1*, in the *vgbook* volume group. Use 120MB for *lvol0* and 192MB for *lvbook1*. Display the details of the volume group and the logical volumes:
        ```shell
        lvcreate -vL 120M vgbook
        lvcreate -vL 192M -n lvbook1 vgbook
        lvs # display information
        vgdisplay -v vgbook # display detailed information about volume group
        ```

1. Asghar Ghori - Exercise 14-3: Extend a Volume Group and a Logical Volume

    * Add another partition *sdd2* of size 158MB to *vgbook* to increase the pool of allocatable space. Initialise the new partition prior to adding it to the volume group. Increase the size of *lvbook1* to 336MB. Display the basic information for the physical volumes, volume group, and logical volume:
        ```shell
        parted mkpart /dev/sdd primary 90 250
        parted /dev/sdd set 2 lvm on
        parted /dev/sdd print # confirm new partition added
        vgextend vgbook /dev/sdd2
        pvs # display information
        vgs # display information
        lvextend vgbook/lvbook1 -L +144M
        lvs # display information
        ```

1. Asghar Ghori - Exercise 14-4: Rename, Reduce, Extend, and Remove Logical Volumes

    * Rename *lvol0* to *lvbook2*. Decrease the size of *lvbook2* to 50MB using the *lvreduce* command and then add 32MB with the *lvresize* command. Remove both logical volumes. Display the summary for the volume groups, logical volumes, and physical volumes:
        ```shell
        lvrename vgbook/lvol0 vgbook/lvbook2
        lvreduce vgbook/lvbook2 -L 50M
        lvextend vgbook/lvbook2 -L +32M
        lvremove vgbook/lvbook1
        lvremove vgbook/lvbook2
        pvs # display information
        vgs # display information
        lvs # display information
        ```

1. Asghar Ghori - Exercise 14-5: Reduce and Remove a Volume Group

    * Reduce *vgbook* by removing the *sdd1* and *sdd2* physical volumes from it, then remove the volume group. Confirm the deletion of the volume group and the logical volumes at the end:
        ```shell
        vgreduce vgbook /dev/sdd1 /dev/sdd2
        vgremove vgbook
        vgs # confirm removals
        pvs # can be used to show output of vgreduce
        ```

1. Asghar Ghori - Exercise 14-5: Reduce and Remove a Volume Group

    * Reduce *vgbook* by removing the *sdd1* and *sdd2* physical volumes from it, then remove the volume group. Confirm the deletion of the volume group and the logical volumes at the end:
        ```shell
        vgreduce vgbook /dev/sdd1 /dev/sdd2
        vgremove vgbook
        vgs # confirm removals
        pvs # can be used to show output of vgreduce
        ```

1. Asghar Ghori - Exercise 14-6: Uninitialise Physical Volumes

    * Uninitialise all three physical volumes - *sdd1*, *sdd2*, and *sdb* - by deleting the LVM structural information from them. Use the *pvs* command for confirmation. Remove the partitions from the *sdd* disk and verify that all disks are now in their original raw state:
        ```shell
        pvremove /dev/sdd1 /dev/sdd2 /dev/sdb
        pvs
        parted /dev/sdd
        # enter print to view partitions
        # enter rm 1
        # enter rm 2
        ```

1. Asghar Ghori - Exercise 14-7: Install Software and Activate Stratis

    * Install the Stratis software packages, start the Stratis service, and mark it for autostart on subsequent system reboots:
        ```shell
        dnf install stratis-cli -y
        systemctl start stratisd.service & systemctl enable stratisd.service
        ```

1. Asghar Ghori - Exercise 14-8: Create and Confirm a Pool and File System

    * Create a Stratis pool and a file system in it. Display information about the pool, file system, and device used:
        ```shell
        stratis pool create mypool /dev/sdd
        stratis pool list # confirm stratis pool created
        stratis filesystem create mypool myfs
        stratis filesystem list # confirm filesystem created, get device path
        mkdir /myfs1
        mount /stratis/mypool/myfs /myfs1
        ```

1. Asghar Ghori - Exercise 14-9: Expand and Rename a Pool and File System

    * Expand the Stratis pool *mypool* using the *sdd* disk. Rename the pool and the file system it contains:
        ```shell
        stratis pool add-data mypool /dev/sdd
        stratis pool rename mypool mynewpool
        stratis pool list # confirm changes
        ```

1. Asghar Ghori - Exercise 14-10: Destroy a File System and Pool

    * Destroy the Stratis file system and the pool that was created, expanded, and renamed in the above exercises. Verify the deletion with appropriate commands:
        ```shell
        umount /bookfs1
        stratis filesystem destroy mynewpool myfs
        stratis filesystem list # confirm deletion
        stratis pool destroy mynewpool
        stratis pool list # confirm deletion
        ```

1. Asghar Ghori - Exercise 15-1: Create and Mount Ext4, VFAT, and XFS File Systems in Partitions

    * Create 2x100MB partitions on the `/dev/sdb` disk, initialise them separately with the Ext4 and VFAT file system types, define them for persistence using their UUIDs, create mount points called `/ext4fs` and `/vfatfs1`, attach them to the directory structure, and verify their availability and usage. Use the disk `/dev/sdc` and repeat the above procedure to establish an XFS file system in it and mount it on `/xfsfs1`:
        ```shell
        parted /dev/sdb
        # enter mklabel 
        # enter msdos 
        # enter mkpart 
        # enter primary
        # enter ext4
        # enter start as 0
        # enter end as 100MB
        # enter print to verify
        parted /dev/sdb mkpart primary 101MB 201MB
        # file system entered during partition created is different
        lsblk # verify partitions
        mkfs.ext4 /dev/sdb1
        mkfs.vfat /dev/sdb2
        parted /dev/sdc
        # enter mklabel 
        # enter msdos 
        # enter mkpart
        # enter primary
        # enter xfs
        # enter start as 0
        # enter end as 100MB
        mkfs.xfs /dev/sdc1
        mkdir /ext4fs /vfatfs1 /xfsfs1
        lsblk -f # get UUID for each file system
        vi /etc/fstab
        # add entries using UUIDs with defaults and file system name
        df -hT # view file systems and mount points
        ```

1. Asghar Ghori - Exercise 15-2: Create and Mount XFS File System in VDO Volume

    * Create a VDO volume called *vdo1* of logical size 16GB on the *sdc* disk (actual size 4GB). Initialise the volume with the XFS file system type, define it for persistence using its device files, create a mount point called `/xfsvdo1`, attach it to the directory structure, and verify its availability and usage:
        ```shell
        wipefs -a /dev/sdc
        vdo create --device /dev/sdc --vdoLogicalSize 16G --name vdo1 --vdoSlabSize 128
        vdo list # list the vdo
        lsblk /dev/sdc # show information about disk
        mkdir /xfsvdo1
        vdo status # get vdo path
        mkfs.xfs /dev/mapper/vdo1
        vi /etc/fstab
        # copy example from man vdo create
        mount -a
        df -hT # view file systems and mount points
        ```

1. Asghar Ghori - Exercise 15-3: Create and Mount Ext4 and XFS File Systems in LVM Logical Volumes

    * Create a volume group called *vgfs* comprised of a 160MB physical volume created in a partition on the `/dev/sdd` disk. The PE size for the volume group should be set at 16MB. Create 2 logical volumes called *ext4vol* and *xfsvol* of sizes 80MB each and initialise them with the Ext4 and XFS file system types. Ensure that both file systems are persistently defined using their logical volume device filenames. Create mount points */ext4fs2* and */xfsfs2*, mount the file systems, and verify their availability and usage:
        ```shell
        vgcreate vgfs /dev/sdd --physicalextentsize 16MB
        lvcreate vgfs --name ext4vol -L 80MB
        lvcreate vgfs --name xfsvol -L 80MB
        mkfs.ext4 /dev/vgfs/ext4vol
        mkfs.xfs /dev/vgfs/xfsvol
        blkid # copy UUID for /dev/mapper/vgfs-ext4vol and /dev/mapper/vgfs-xfsvol
        vi /etc/fstab
        # add lines with copied UUID
        mount -a
        df -hT # confirm added
        ```

1. Asghar Ghori - Exercise 15-4: Resize Ext4 and XFS File Systems in LVM Logical Volumes

    * Grow the size of the *vgfs* volume group that was created above by adding the whole *sdc* disk to it. Extend the *ext4vol* logical volume along with the file system it contains by 40MB using 2 separate commands. Extend the *xfsvol* logical volume along with the file system it contains by 40MB using a single command:
        ```shell
        vdo remove --name vdo1 # need to use this disk
        vgextend vgfs /dev/sdc
        lvextend -L +80 /dev/vgfs/ext4vol
        fsadm resize /dev/vgfs/ext4vol
        lvextend -L +80 /dev/vgfs/xfsvol
        fsadm resize /dev/vgfs/xfsvol
        lvresize -r -L +40 /dev/vgfs/xfsvol # -r resizes file system
        lvs # confirm resizing
        ```

1. Asghar Ghori - Exercise 15-5: Create, Mount, and Expand XFS File System in Stratis Volume

    * Create a Stratis pool called *strpool* and a file system *strfs2* by reusing the 1GB *sdc* disk. Display information about the pool, file system, and device used. Expand the pool to include another 1GB disk *sdh* and confirm:
        ```shell
        stratis pool create strpool /dev/sdc
        stratis filesystem create strpool strfs2
        stratis pool list # view created stratis pool
        stratis filesystem list # view created filesystem
        stratis pool add-data strpool /dev/sdd
        stratis blockdev list strpool # list block devices in pool
        mkdir /strfs2
        lsblk /stratis/strpool/strfs2 -o UUID
        vi /etc/fstab
        # add line
        # UUID=2913810d-baed-4544-aced-a6a2c21191fe /strfs2 xfs x-systemd.requires=stratisd.service 0 0
        ```


1. Asghar Ghori - Exercise 15-6: Create and Activate Swap in Partition and Logical Volume

    * Create 1 swap area in a new 40MB partition called *sdc3* using the *mkswap* command. Create another swap area in a 140MB logical volume called *swapvol* in *vgfs*. Add their entries to the `/etc/fstab` file for persistence. Use the UUID and priority 1 for the partition swap and the device file and priority 2 for the logical volume swap. Activate them and use appropriate tools to validate the activation:
        ```shell
        parted /dev/sdc
        # enter mklabel msdos
        # enter mkpart primary 0 40
        parted /dev/sdd
        # enter mklabel msdos
        # enter mkpart primary 0 140
        mkswap -L sdc3 /dev/sdc 40
        vgcreate vgfs /dev/sdd1
        lvcreate vgfs --name swapvol -L 132
        mkswap swapvol /dev/sdd1
        mkswap /dev/vgfs/swapvol
        lsblk -f # get UUID
        vi /etc/fstab
        # add 2 lines, e.g. first line
        # UUID=WzDb5Y-QMtj-fYeo-iW0f-sj8I-ShRu-EWRIcp swap swap pri=2 0 0
        mount -a
        ```

1. Asghar Ghori - Exercise 16-1: Export Share on NFS Server

    * Create a directory called `/common` and export it to *server1* in read/write mode. Ensure that NFS traffic is allowed through the firewall. Confirm the export:
        ```shell
        dnf install nfs-utils -y
        mkdir /common
        firewall-cmd --permanent --add-service=nfs
        firewall-cmd --reload
        systemctl start nfs-server.service & systemctl enable nfs-server.service
        echo "/nfs *(rw)" >> /etc/exports
        exportfs -av
        ```

1. Asghar Ghori - Exercise 16-2: Mount Share on NFS Client

    * Mount the `/common` share exported above. Create a mount point called `/local`, mount the remote share manually, and confirm the mount. Add the remote share to the file system table for persistence. Remount the share and confirm the mount. Create a test file in the mount point and confirm the file creation on the NFS server:
        ```shell
        dnf install cifs-utils -y
        mkdir /local
        chmod 755 local
        mount 10.0.2.15:/common /local
        vi /etc/fstab
        # add line
        # 10.0.2.15:/common /local nfs _netdev 0 0
        mount -a
        touch /local/test # confirm that it appears on server in common
        ```

1. Asghar Ghori - Exercise 16-3: Access NFS Share Using Direct Map

    * Configure a direct map to automount the NFS share `/common` that is available from *server2*. Install the relevant software, create a local mount point `/autodir`, and set up AutoFS maps to support the automatic mounting. Note that `/common` is already mounted on the `/local` mount point on *server1* via *fstab*. Ensure there is no conflict in configuration or functionality between the 2:
        ```shell
        dnf install autofs -y
        mkdir /autodir
        vi /etc/auto.master
        # add line
        #/- /etc/auto.master.d/auto.dir
        vi /etc/auto.master.d/auto.dir
        # add line
        #/autodir 172.25.1.4:/common
        systemctl restart autofs
        ```

1. Asghar Ghori - Exercise 16-4: Access NFS Share Using Indirect Map

    * Configure an indirect map to automount the NFS share `/common` that is available from *server2*. Install the relevant software and set up AutoFS maps to support the automatic mounting. Observe that the specified mount point "autoindir" is created automatically under `/misc`. Note that `/common` is already mounted on the `/local` mount point on *server1* via *fstab*. Ensure there is no conflict in configuration or functionality between the 2:
        ```shell
        dnf install autofs -y
        grep /misc /etc/auto.master # confirm entry is there
        vi /etc/auto.misc
        # add line
        #autoindir 172.25.1.4:/common
        systemctl restart autofs
        ```

1. Asghar Ghori - Exercise 16-5: Automount User Home Directories Using Indirect Map

    * On *server1* (NFS server), create a user account called *user30* with UID 3000. Add the `/home` directory to the list of NFS shares so that it becomes available for remote mount. On *server2* (NFS client), create a user account called *user30* with UID 3000, base directory `/nfshome`, and no user home directory. Create an umbrella mount point called `/nfshome` for mounting the user home directory from the NFS server. Install the relevent software and establish an indirect map to automount the remote home directory of *user30* under `/nfshome`. Observe that the home directory of *user30* is automounted under `/nfshome` when you sign in as *user30*:
        ```shell
        # on server 1 (NFS server)
        useradd -u 3000 user30
        echo password1 | passwd --stdin user30
        vi /etc/exports
        # add line
        #/home *(rw)
        exportfs -avr

        # on server 2 (NFS client)
        dnf install autofs -y        
        useradd user30 -u 3000 -Mb /nfshome
        echo password1 | passwd --stdin user30
        mkdir /nfshome
        vi /etc/auto.master
        # add line
        #/nfshome /etc/auto.master.d/auto.home
        vi /etc/auto.master.d/auto.home
        # add line
        #* -rw &:/home&
        systemctl enable autofs.service & systemctl start autofs.service
        sudo su - user30
        # confirm home directory is mounted
        ```

1. Asghar Ghori - Exercise 17.1: Change System Hostname

    * Change the hostnames of *server1* to *server10.example.com* and *server2* to *server20.example.com* by editing a file and restarting the corresponding service daemon and using a command respectively:
        ```shell
        # on server 1
        vi /etc/hostname
        # change line to server10.example.com
        systemctl restart systemd-hostnamed
        
        # on server 2
        hostnamectl set-hostname server20.example.com
        ```

1. Asghar Ghori - Exercise 17.2: Add Network Devices to server10 and server20

    * Add one network interface to *server10* and one to *server20* using VirtualBox:
        ```shell
        # A NAT Network has already been created and attached to both servers in VirtualBox to allow them to have seperate IP addresses (note that the MAC addressed had to be changed)
        # Add a second Internal Network adapter named intnet to each server
        nmcli conn show # observe enp0s8 added as a connection
        ```

1. Asghar Ghori - Exercise 17.3: Configure New Network Connection Manually

    * Create a connection profile for the new network interface on *server10* using a text editing tool. Assign the IP 172.10.10.110/24 with gateway 172.10.10.1 and set it to autoactivate at system reboots. Deactivate and reactive this interface at the command prompt:
        ```shell
        vi /etc/sysconfig/network-scripts/ifcfg-enp0s8
        # add contents of file
        #TYPE=Ethernet
        #BOOTPROTO=static
        #IPV4_FAILURE_FATAL=no
        #IPV6INIT=no
        #NAME=enp0s8
        #DEVICE=enp0s8
        #ONBOOT=yes
        #IPADDR=172.10.10.110
        #PREFIX=24
        #GATEWAY=172.10.10.1
        ifdown enp0s8
        ifup enp0s8
        ip a # verify activation
        ```

1. Asghar Ghori - Exercise 17.4: Configure New Network Connection Using nmcli

    * Create a connection profile using the *nmcli* command for the new network interface enp0s8 that was added to *server20*. Assign the IP 172.10.10.120/24 with gateway 172.10.10.1, and set it to autoactivate at system reboot. Deactivate and reactivate this interface at the command prompt:
        ```shell
        nmcli dev status # show devices with enp0s8 disconnected
        nmcli con add type Ethernet ifname enp0s8 con-name enp0s8 ip4 172.10.10.120/24 gw4 172.10.10.1
        nmcli conn show # verify connection added
        nmcli con down enp0s8
        nmcli con up enp0s8
        ip a # confirm ip address is as specified
        ```

1. Asghar Ghori - Exercise 17.5: Update Hosts Table and Test Connectivity

    * Update the `/etc/hosts` file on both *server10* and *server20*. Add the IP addresses assigned to both connections and map them to hostnames *server10*, *server10s8*, *server20*, and *server20s8* appropriately. Test connectivity from *server10* to *server20* to and from *server10s8* to *server20s8* using their IP addresses and then their hostnames:
        ```shell
        ## on server20
        vi /etc/hosts
        # add lines
        #172.10.10.120 server20.example.com server20
        #172.10.10.120 server20s8.example.com server20s8
        #192.168.0.110 server10.example.com server10
        #192.168.0.110 server10s8.example.com server10s8

        ## on server10
        vi /etc/hosts
        # add lines
        #172.10.10.120 server20.example.com server20
        #172.10.10.120 server20s8.example.com server20s8
        #192.168.0.110 server10.example.com server10
        #192.168.0.110 server10s8.example.com server10s8
        ping server10 # confirm host name resolves
        ```

1. Asghar Ghori - Exercise 18.1: Configure NTP Client

    * Install the Chrony software package and activate the service without making any changes to the default configuration. Validate the binding and operation:
        ```shell
        dnf install chrony -y
        vi /etc/chrony.conf # view default configuration
        systemctl start chronyd.service & systemctl enable chronyd.service
        chronyc sources # view time sources
        chronyc tracking # view clock performance
        ```

1. Asghar Ghori - Exercise 19.1: Access RHEL System from Another RHEL System

    * Issue the *ssh* command as *user1* on *server10* to log in to *server20*. Run appropriate commands on *server20* for validation. Log off and return to the originating system:
        ```shell
        # on server 10
        ssh user1@server20
        whoami
        pwd
        hostname # check some basic information
        # ctrl + D to logout
        ```

1. Asghar Ghori - Exercise 19.2: Access RHEL System from Windows

    * Use a program called PuTTY to access *server20* using its IP address and as *user1*. Run appropriate commands on *server20* for validation. Log off to terminate the session:
        ```shell
        # as above but using the server20 IP address in PuTTy
        ```

1. Asghar Ghori - Exercise 19.3: Generate, Distribute, and Use SSH Keys

    * Generate a password-less ssh key pair using RSA for *user1* on *server10*. Display the private and public file contents. Distribute the public key to *server20* and attempt to log on to *server20* from *server10*. Show the log file message for the login attempt:
        ```shell
        # on server10
        ssh-keygen
        # press enter to select default file names and no password
        ssh-copy-id server20
        ssh server20 # confirm you can login

        # on server20
        vi /var/log/secure # view login event
        ```

1. Asghar Ghori - Exercise 20.1: Add Services and Ports, and Manage Zones

    * Determine the current active zone. Add and activate a permanent rule to allow HTTP traffic on port 80, and then add a runtime rule for traffic intended for TCP port 443. Add a permanent rule to the *internal* zone for TCP port range 5901 to 5910. Confirm the changes and display the contents of the affected zone files. Switch the default zone to the *internal* zone and activate it:
        ```shell
        # on server10
        firewall-cmd --get-active-zones # returns public with enp0s8 interface
        firewall-cmd --add-service=http --permanent
        firewall-cmd --add-service=https
        firewall-cmd --add-port=80/tcp --permanent
        firewall-cmd --add-port=443/tcp
        firewall-cmd --zone=internal --add-port=5901-5910/tcp --permanent
        firewall-cmd --reload
        firewall-cmd --list-services # confirm result
        firewall-cmd --list-ports # confirm result
        vi /etc/firewalld/zones/public.xml # view configuration
        vi /etc/firewalld/zones/internal.xml # view configuration
        firewall-cmd --set-default-zone=internal
        firewall-cmd --reload
        firewall-cmd --get-active-zones # returns internal with enp0s8 interface
        ```

1. Asghar Ghori - Exercise 20.2: Remove Services and Ports, and Manage Zones

    * Remove the 2 permanent rules added above. Switch back to the *public* zone as the default zone, and confirm the changes:
        ```shell
        firewall-cmd --set-default-zone=public
        firewall-cmd --remove-service=http --permanent
        firewall-cmd --remove-port=80/tcp --permanent
        firewall-cmd --reload
        firewall-cmd --list-services # confirm result
        firewall-cmd --list-ports # confirm result
        ```

1. Asghar Ghori - Exercise 20.3: Test the Effect of Firewall Rule

    * Remove the *sshd* service rule from the runtime configuration on *server10*, and try to access the server from *server20* using the *ssh* command:
        ```shell
        # on server10
        firewall-cmd --remove-service=ssh --permanent
        firewall-cmd --reload
        
        # on server20
        ssh user1@server10
        # no route to host message displayed

        # on server10
        firewall-cmd --add-service=ssh --permanent
        firewall-cmd --reload
        
        # on server20
        ssh user1@server10
        # success
        ```

1. Asghar Ghori - Exercise 21.1: Modify SELinux File Context

    * Create a directory *sedir1* under `/tmp` and a file *sefile1* under *sedir1*. Check the context on the directory and file. Change the SELinux user and type to user_u and public_content_t on both and verify:
        ```shell
        mkdir /tmp/sedir1
        touch /tmp/sedir1/sefile1
        cd /tmp/sedir1
        ll -Z # unconfined_u:object_r:user_tmp_t:s0 shown
        chcon -u user_u -R sedir1
        chcon -t public_content_t -R sedir1
        ```

1. Asghar Ghori - Exercise 21.2: Add and Apply File Context

    * Add the current context on *sedir1* to the SELinux policy database to ensure a relabeling will not reset it to its previous value. Next, you will change the context on the directory to some random values. Restore the default context from the policy database back to the directory recursively:
        ```shell
        semanage fcontext -a -t public_content_t -s user_u '/tmp/sedir1(/.*)?'
        cat /etc/selinux/targeted/contexts/files/file_contexts.local # view recently added policies
        restorecon -Rv sedir1 # any chcon changes are reverted with this
        ```

1. Asghar Ghori - Exercise 21.3: Add and Delete Network Ports

    * Add a non-standard port 8010 to the SELinux policy database for the *httpd* service and confirm the addition. Remove the port from the policy and verify the deletion:
        ```shell
        semanage port -a -t http_port_t -p tcp 8010
        semanage port -l | grep http # list all port settings
        semanage port -d -t http_port_t -p tcp 8010
        semanage port -l | grep http
        ```

1. Asghar Ghori - Exercise 21.4: Copy Files with and without Context

    * Create a file called *sefile2* under `/tmp` and display its context. Copy this file to the `/etc/default` directory, and observe the change in the context. Remove *sefile2* from `/etc/default`, and copy it again to the same destination, ensuring that the target file receives the source file's context:
        ```shell
        cd /tmp
        touch sefile2
        ll -Zrt # sefile2 context is unconfined_u:object_r:user_tmp_t:s0
        cp sefile2 /etc/default
        cd /etc/default
        ll -Zrt # sefile2 context is unconfined_u:object_r:etc_t:s0
        rm /etc/default/sefile2
        cp /tmp/sefile2 /etc/default/sefile2 --preserve=context
        ll -Zrt # sefile2 context is unconfined_u:object_r:user_tmp_t:s0
        ```

1. Asghar Ghori - Exercise 21.5: View and Toggle SELinux Boolean Values

    * Display the current state of the Boolean nfs_export_all_rw. Toggle its value temporarily, and reboot the system. Flip its value persistently after the system has been back up:
        ```shell
        getsebool nfs_export_all_rw # nfs_export_all_rw --> on
        sestatus -b | grep nfs_export_all_rw # also works
        setsebool nfs_export_all_rw_off
        reboot
        setsebool nfs_export_all_rw_off -P
        ```

1. Prince Bajaj - Managing Containers

    * Download the Apache web server container image (httpd 2.4) and inspect the container image. Check the exposed ports in the container image configuration:
        ```shell
        # as root
        usermod user1 -aG wheel
        cat /etc/groups | grep wheel # confirm
        
        # as user1
        podman search httpd # get connection refused
        # this was because your VM was setup as an Internal Network and not a NAT network so it couldn't access the internet
        # see result registry.access.redhat.com/rhscl/httpd-24-rhel7
        skopeo inspect --creds name:password docker://registry.access.redhat.com/rhscl/httpd-24-rhel7
        podman pull registry.access.redhat.com/rhscl/httpd-24-rhel7
        podman inspect registry.access.redhat.com/rhscl/httpd-24-rhel7
        # exposed ports shown as 8080 and 8443
        ```

    * Run the httpd container in the background. Assign the name *myweb* to the container, verify that the container is running, stop the container and verify that it has stopped, and delete the container and the container image:
        ```shell
        podman run --name myweb -d registry.access.redhat.com/rhscl/httpd-24-rhel7
        podman ps # view running containers
        podman stop myweb
        podman ps # view running containers
        podman rm myweb
        podman rmi registry.access.redhat.com/rhscl/httpd-24-rhel7
        ```

    * Pull the Apache web server container image (httpd 2.4) and run the container with the name *webserver*. Configure *webserver* to display content "Welcome to container-based web server". Use port 3333 on the host machine to receive http requests. Start a bash shell in the container to verify the configuration:
        ```shell
        # as root
        dnf install httpd -y
        vi /var/www/html/index.html
        # add row "Welcome to container-based web server"

        # as user1
        podman search httpd
        podman pull registry.access.redhat.com/rhscl/httpd-24-rhel7
        podman inspect registry.access.redhat.com/rhscl/httpd-24-rhel7 # shows 8080 in exposedPorts, and /opt/rh/httpd24/root/var/www is shown as HTTPD_DATA_ORIG_PATH 
        podman run -d=true -p 3333:8080 --name=webserver -v /var/www/html:/opt/rh/httpd24/root/var/www/html registry.access.redhat.com/rhscl/httpd-24-rhel7
        curl http://localhost:3333 # success!
                
        # to go into the container and (for e.g.) check the SELinux context
        podman exec -it webserver /bin/bash
        cd /opt/rh/httpd24/root/var/www/html
        ls -ldZ

        # you can also just go to /var/www/html/index.html in the container and change it there
        ```

    * Configure the system to start the *webserver* container at boot as a systemd service. Start/enable the systemd service to make sure the container will start at book, and reboot the system to verify if the container is running as expected:
        ```shell
        # as root
        podman pull registry.access.redhat.com/rhscl/httpd-24-rhel7
        vi /var/www/html/index
        # add row "Welcome to container-based web server"
        podman run -d=true -p 3333:8080/tcp --name=webserver -v /var/www/html:/opt/rh/httpd24/root/var/www/html registry.access.redhat.com/rhscl/httpd-24-rhel7
        cd /etc/systemd/system
        podman generate systemd webserver >> httpd-container.service
        systemctl daemon-reload
        systemctl enable httpd-container.service --now
        reboot
        systemctl status httpd-container.service
        curl http://localhost:3333 # success

        # this can also be done as a non-root user
        podman pull registry.access.redhat.com/rhscl/httpd-24-rhel7
        sudo vi /var/www/html/index.html
        # add row "Welcome to container-based web server"
        sudo setsebool -P container_manage_cgroup true
        podman run -d=true -p 3333:8080/tcp --name=webserver -v /var/www/html:/opt/rh/httpd24/root/var/www/html registry.access.redhat.com/rhscl/httpd-24-rhel7
        podman generate systemd webserver > /home/jr/.config/systemd/user/httpd-container.service
        cd /home/jr/.config/systemd/user
        sudo semanage fcontext -a -t systemd_unit_file_t httpd-container.service
        sudo restorecon httpd-container.service
        systemctl enable --user httpd-container.service --now
        ```

    * Pull the *mariadb* image to your system and run it publishing the exposed port. Set the root password for the mariadb service as *mysql*. Verify if you can login as root from local host:
        ```shell
        # as user1
        sudo dnf install mysql -y
        podman search mariadb
        podman pull docker.io/library/mariadb
        podman inspect docker.io/library/mariadb # ExposedPorts 3306 
        podman run --name mariadb -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=mysql docker.io/library/mariadb
        podman inspect mariadb # IPAddress is 10.88.0.22
        mysql -h 10.88.0.22 -u root -p
        ```

1. Linux Hint - Bash Script Examples

    * Create a hello world script:
        ```shell
        !#/bin/bash
        echo "Hello World!"
        exit
        ```

    * Create a script that uses a while loop to count to 5:
        ```shell
        !#/bin/bash
        count=0
        while [ $count -le 5 ]
        do
            echo "$count"
            count = $(($count + 1))
        done
        exit
        ```

    * Note the formatting requirements. For example, there can be no space between the equals and the variable names, there must be a space between the "]" and the condition, and there must be 2 sets of round brackets in the variable incrementation.

    * Create a script that uses a for loop to count to 5:
        ```shell
        !#/bin/bash
        count=5
        for ((i=1; i<=$count; i++))
        do
            echo "$i"
        done
        exit
        ```

    * Create a script that uses a for loop to count to 5 printing whether the number is even or odd:
        ```shell
        !#/bin/bash
        count=5
        for ((i=1; i<=$count; i++))
        do
            if [ $(($i%2)) -eq 0 ]
            then
                echo "$i is even"
            else
                echo "$i is odd"
            fi
        done
        exit
        ```

    * Create a script that uses a for loop to count to a user defined number printing whether the number is even or odd:
        ```shell
        !#/bin/bash
        echo "Enter a number: "
        read count
        for ((i=1; i<=$count; i++))
        do
            if [ $(($i%2)) -eq 0 ]
            then
                echo "$i is even"
            else
                echo "$i is odd"
            fi
        done
        exit
        ```

    * Create a script that uses a function to multiply 2 numbers together:
        ```shell
        !#/bin/bash
        Rectangle_Area() {
            area=$(($1 * $2))
            echo "Area is: $area"
        }
        
        Rectangle_Area 10 20
        exit
        ```

    * Create a script that uses the output of another command to make a decision:
        ```shell
        !#/bin/bash
        ping -c 1 $1 > /dev/null 2>&1
        if [ $? -eq 0 ]
        then
            echo "Connectivity to $1 established"
        else
            echo "Connectivity to $1 unavailable"
        fi
        exit
        ```

1. Asghar Ghori - Sample RHCSA Exam 1

    * Setup a virtual machine RHEL 8 Server for GUI. Add a 10GB disk for the OS and use the default storage partitioning. Add 2 300MB disks. Add a network interface, but do not configure the hostname and network connection.

    * Assuming the root user password is lost, reboot the system and reset the root user password to root1234:
        ```shell
        # ctrl + e after reboot
        # add rd.break after Linux line
        # ctrl + d
        mount -o remount, rw /sysroot
        chroot /sysroot
        passwd
        # change password to root12345
        touch /.autorelabel
        exit
        reboot
        ```

    * Using a manual method (i.e. create/modify files by hand), configure a network connection on the primary network device with IP address 192.168.0.241/24, gateway 192.168.0.1, and nameserver 192.168.0.1:
        ```shell
        vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
        systemctl restart NetworkManager.service
        # add line IPADDR=192.168.0.241
        # add line GATEWAY=192.168.0.1
        # add line DNS=192.168.0.1
        # add line PREFIX=24
        # change BOOTPROTO from dhcp to none
        ifup enp0s3
        nmcli con show # validate
        ```

    * Using a manual method (modify file by hand), set the system hostname to rhcsa1.example.com and alias rhcsa1. Make sure the new hostname is reflected in the command prompt:
        ```shell
        vi /etc/hostname
        # replace line with rhcsa1.example.com
        vi /etc/hosts
        # add rhcsa1.example.com and rhcsa1 to first line
        systemctl restart NetworkManager.service
        vi ~/.bashrc
        # add line export PS1 = <($hostname)>
        ```

    * Set the default boot target to multi-user:
        ```shell
        systemctl set-default multi-user.target
        ```

    * Set SELinux to permissive mode:
        ```shell
        setenforce permissive
        sestatus # confirm
        vi /etc/selinux/config
        # change line SELINUX=permissive for permanence
        ```

    * Perform a case-insensitive search for all lines in the `/usr/share/dict/linux.words` file that begin with the pattern "essential". Redirect the output to `/tmp/pattern.txt`. Make sure that empty lines are omitted:
        ```shell
        grep '^essential' /usr/share/dict/linux.words > /tmp/pattern.txt
        ```

    * Change the primary command prompt for the root user to display the hostname, username, and current working directory information in that order. Update the per-user initialisation file for permanence:
        ```shell
        vi /root/.bashrc
        # add line export PS1 = '<$(whoami) on $(hostname) in $(pwd)>'$
        ```

    * Create user accounts called user10, user20, and user30. Set their passwords to Temp1234. Make accounts for user10 and user30 to expire on December 31, 2021:
        ```shell
        useradd user10
        useradd user20
        useradd user30
        passwd user10 # enter password
        passwd user20 # enter password
        passwd user30 # enter password
        chage -E 2021-12-31 user10
        chage -E 2021-12-31 user30
        chage -l user10 # confirm
        ```

    * Create a group called group10 and add users user20 and user30 as secondary members:
        ```shell
        groupadd group10
        usermod -aG group10 user20
        usermod -aG group10 user30
        cat /etc/group | grep "group10" # confirm
        ```

    * Create a user account called user40 with UID 2929. Set the password to user1234:
        ```shell
        useradd -u 2929 user40
        passwd user40 # enter password
        ```

    * Create a directory called dir1 under `/tmp` with ownership and owning groups set to root. Configure default ACLs on the directory and give user user10 read, write, and execute permissions:
        ```shell
        mkdir /tmp/dir1
        cd /tmp
        # tmp already has ownership with root
        setfacl -m u:user10:rwx dir1
        ```

    * Attach the RHEL 8 ISO image to the VM and mount it persistently to `/mnt/cdrom`. Define access to both repositories and confirm:
        ```shell
        # add ISO to the virtualbox optical drive
        mkdir /mnt/cdrom
        mount /dev/sr0 /mnt/cdrom
        vi /etc/yum.repos.d/image.repo
        blkid /dev/sr0 >> /etc/fstab
        vi /etc/fstab
        # format line with UUID /mnt/cdrom iso9660 defaults 0 0
        # contents of image.repo
        #####
        #[BaseOS]
        #name=BaseOS
        #baseurl=file:///mnt/cdrom/BaseOS
        #enabled=1
        #gpgenabled=1
        #gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
        #
        #[AppStream]
        #name=AppStream
        #baseurl=file:///mnt/cdrom/AppStream
        #enabled=1
        #gpgenabled=1
        #gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
        #####
        yum repolist # confirm
        ```

    * Create a logical volume called lvol1 of size 300MB in vgtest volume group. Mount the Ext4 file system persistently to `/mnt/mnt1`:
        ```shell
        mkdir /mnt/mnt1
        # /dev/sdb is already 300MB so don't need to worry about partitioning
        vgcreate vgtest /dev/sdb
        lvcreate --name lvol1 -L 296MB vgtest
        lsblk # confirm
        mkfs.ext4 /dev/mapper/vgtest-lvol1
        vi /etc/fstab
        # add line
        # /dev/mapper/vgtest-lvol1 /mnt/mnt1 ext4 defaults 0 0
        mount -a
        lsblk # confirm
        ```

    * Change group membership on `/mnt/mnt1` to group10. Set read/write/execute permissions on `/mnt/mnt1` for group members, and revoke all permissions for public:
        ```shell
        chgrp group10 /mnt/mnt1
        chmod 770 /mnt/mnt1
        ```

    * Create a logical volume called lvswap of size 300MB in the vgtest volume group. Initialise the logical volume for swap use. Use the UUID and place an entry for persistence:
        ```shell
        # /dev/sdc is already 300MB so don't need to worry about partitioning
        vgcreate vgswap /dev/sdc
        lvcreate --name lvswap -L 296MB vgswap /dev/sdc
        mkswap /dev/mapper-vgswap-lvswap # UUID returned
        blkid /dev/sdc >> /etc/fstab
        # organise new line so that it has UUID= swp swap defaults 0 0
        swapon -a
        lsblk # confirm
        ```

    * Use tar and bzip2 to create a compressed archive of the `/etc/sysconfig` directory. Store the archive under `/tmp` as etc.tar.bz2:
        ```shell
        tar -cvzf /tmp/etc.tar.bz2 /etc/sysconfig
        ```

    * Create a directory hierarchy `/dir1/dir2/dir3/dir4`, and apply SELinux contexts for `/etc` on it recursively:
        ```shell
        mkdir -p /dir1/dir2/dir3/dir4
        ll -Z 
        # etc shown as system_u:object_r:etc_t:s0
        # dir1 shown as unconfined_u:object_r:default_t:s0
        semanage fcontext -a -t etc_t "/dir1(/.*)?"
        restorecon -R -v /dir1
        ll -Z # confirm
        ```

    * Enable access to the atd service for user20 and deny for user30:
        ```shell
        echo "user30" >> /etc/at.deny
        # just don't create at.allow
        ```

    * Add a custom message "This is the RHCSA sample exam on $(date) by $LOGNAME" to the `/var/log/messages` file as the root user. Use regular expression to confirm the message entry to the log file:
        ```shell
        logger "This is the RHCSA sample exam on $(date) by $LOGNAME"
        grep "This is the" /var/log/messages
        ```

    * Allow user20 to use sudo without being prompted for their password:
        ```shell
        usermod -aG wheel user20
        # still prompts for password, could change the wheel group behaviour or add new line to sudoers
        visudo
        # add line at end user20 ALL=(ALL) NOPASSWD: ALL
        ```

1. Asghar Ghori - Sample RHCSA Exam 2

    * Setup a virtual machine RHEL 8 Server for GUI. Add a 10GB disk for the OS and use the default storage partitioning. Add 1 400MB disk. Add a network interface, but do not configure the hostname and network connection.

    * Using the nmcli command, configure a network connection on the primary network device with IP address 192.168.0.242/24, gateway 192.168.0.1, and nameserver 192.168.0.1:
        ```shell
        nmcli con add ifname enp0s3 con-name mycon type ethernet ip4 192.168.0.242/24 gw4 192.168.0.1 ipv4.dns "192.168.0.1"
        # man nmcli-examples can be referred to if you forget format
        nmcli con show mycon | grep ipv4 # confirm
        ```
    
    * Using the hostnamectl command, set the system hostname to rhcsa2.example.com and alias rhcsa2. Make sure that the new hostname is reflected in the command prompt:
        ```shell
        hostnamectl set-hostname rhcsa2.example.com
        hostnamectl set-hostname --static rhcsa2 # not necessary due to format of FQDN
        # the hostname already appears in the command prompt
        ```

    * Create a user account called user70 with UID 7000 and comments "I am user70". Set the maximum allowable inactivity for this user to 30 days:
        ```shell
        useradd -u 7000 -c "I am user70" user70
        chage -I 30 user70
        ```

    * Create a user account called user50 with a non-interactive shell:
        ```shell
        useradd user50 -s /sbin/nologin
        ```

    * Create a file called testfile1 under `/tmp` with ownership and owning group set to root. Configure access ACLs on the file and give user10 read and write access. Test access by logging in as user10 and editing the file:
        ```shell
        useradd user10
        passwd user10 # set password
        touch /tmp/testfile1
        cd /tmp
        setfacl -m u:user10:rw testfile1
        sudo su user10
        vi /tmp/testfile1 # can edit the file
        ```

    * Attach the RHEL 8 ISO image to the VM and mount it persistently to `/mnt/dvdrom`. Define access to both repositories and confirm:
        ```shell
        mkdir /mnt/dvdrom
        lsblk # rom is at /dev/sr0
        mount /dev/sr0 /mnt/dvdrom
        blkid /dev/sr0 >> /etc/fstab
        vi /etc/fstab
        # format line with UUID /mnt/dvdrom iso9660 defaults 0 0
        vi /etc/yum.repos.d/image.repo
        # contents of image.repo
        #####
        #[BaseOS]
        #name=BaseOS
        #baseurl=file:///mnt/dvdrom/BaseOS
        #enabled=1
        #gpgenabled=1
        #gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
        #
        #[AppStream]
        #name=AppStream
        #baseurl=file:///mnt/dvdrom/AppStream
        #enabled=1
        #gpgenabled=1
        #gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
        #####
        yum repolist # confirm
        ```

    * Create a logical volume called lv1 of size equal to 10 LEs in vg1 volume group (create vg1 with PE size 8MB in a partition on the 400MB disk). Initialise the logical volume with XFS file system type and mount it on `/mnt/lvfs1`. Create a file called lv1file1 in the mount point. Set the file system to automatically mount at each system reboot:
        ```shell
        parted /dev/sdb
        mklabel msdos
        mkpart
        # enter primary
        # enter xfs
        # enter 0
        # enter 100MB
        vgcreate vg1 -s 8MB /dev/sdb1
        lvcreate --name lv1 -l 10 vg1 /dev/sdb1
        mkfs.xfs /dev/mapper/vg1-lv1
        mkdir /mnt/lvfs1
        vi /etc/fstab
        # add line for /dev/mapper/vg1-lv1 /mnt/lvfs1 xfs defaults 0 0
        mount -a
        df -h  # confirm
        touch /mnt/lvfs1/hi
        ```

    * Add a group called group20 and change group membership on `/mnt/lvfs1` to group20. Set read/write/execute permissions on `/mnt/lvfs1` for the owner and group members, and no permissions for others:
        ```shell
        groupadd group20
        chgrp group20 -R /mnt/lvfs1
        chmod 770 -R /mnt/lvfs1
        ```

    * Extend the file system in the logical volume lv1 by 64MB without unmounting it and without losing any data:
        ```shell
        lvextend -L +64MB vg1/lv1 /dev/sdb1
        # realised that the partition of 100MB isn't enough
        parted /dev/sdb
        resizepart
        # expand partition 1 to 200MB
        pvresize /dev/sdb1
        lvextend -L +64MB vg1/lv1 /dev/sdb1
        ```

    * Create a swap partition of size 85MB on the 400MB disk. Use its UUID and ensure it is activated after every system reboot:
        ```shell
        parted /dev/sdb
        mkpart
        # enter primary
        # enter linux-swap
        # enter 200MB
        # enter 285MB
        mkswap /dev/sdb2
        vi /etc/fstab
        # add line for UUID swap swap defaults 0 0
        swapon -a
        ```

    * Create a disk partition of size 100MB on the 400MB disk and format it with Ext4 file system structures. Assign label stdlabel to the file system. Mount the file system on `/mnt/stdfs1` persistently using the label. Create file stdfile1 in the mount point:
        ```shell
        parted /dev/sdb
        mkpart
        # enter primary
        # enter ext4
        # enter 290MB
        # enter 390MB
        mkfs.ext4 -L stdlabel /dev/sdb3
        mkdir /mnt/stdfs1
        vi /etc/fstab
        # add line for UUID /mnt/stdfs1 ext4 defaults 0 0
        touch /mnt/stdfs1/hi
        ```

    * Use tar and gzip to create a compressed archive of the `/usr/local` directory. Store the archive under `/tmp` using a filename of your choice:
        ```shell
        tar -czvf /tmp/local.tar.gz /usr/local
        ```

    * Create a directory `/direct01` and apply SELinux contexts for `/root`:
        ```shell
        mkdir /direct01
        ll -Z
        # direct01 has unconfined_u:object_r:default_t:s0
        # root has system_u:object_r:admin_home_t:s0
        semanage fcontext -a -t admin_home_t -s system_u "/direct01(/.*)?" 
        restorecon -R -v /direct01
        ll -Zrt # confirm
        ```

    * Set up a cron job for user70 to search for core files in the `/var` directory and copy them to the directory `/tmp/coredir1`. This job should run every Monday at 1:20 a.m:
        ```shell
        mkdir /tmp/coredir1
        crontab -u user70 -e
        20 1 * * Mon find /var -name core -type f exec cp '{}' /tmp/coredir1 \;
        crontab -u user70 -l # confirm
        ```

    * Search for all files in the entire directory structure that have been modified in the past 30 days and save the file listing in the `/var/tmp/modfiles.txt` file:
        ```shell
        find / -mtime -30 >> /var/tmp/modfiles.txt
        ```

    * Modify the bootloader program and set the default autoboot timer value to 2 seconds:
        ```shell
        vi /etc/default/grub
        # set GRUB_TIMEOUT=2
        grub2-mkconfig -o /boot/grub2/grub.cfg
        ```

    * Determine the recommended tuning profile for the system and apply it:
        ```shell
        tuned-adm recommend
        # virtual-guest is returned
        tuned-adm active
        # virtual-guest is returned
        # no change required
        ```

    * Configure Chrony to synchronise system time with the hardware clock:
        ```shell
        systemctl status chronyd.service
        vi /etc/chrony.conf
        # everything looks alright
        ```

    * Install package group called "Development Tools", and capture its information in `/tmp/systemtools.out` file:
        ```shell
        yum grouplist # view available groups
        yum groupinstall "Development Tools" -y >> /tmp/systemtools.out
        ```

    * Lock user account user70. Use regular expressions to capture the line that shows the lock and store the output in file `/tmp/user70.lock`:
        ```shell
        usermod -L user70
        grep user70 /etc/shadow >> /tmp/user70.lock # observe !
        ```

1. Asghar Ghori - Sample RHCSA Exam 3

    * Build 2 virtual machines with RHEL 8 Server for GUI. Add a 10GB disk for the OS and use the default storage partitioning. Add 1 4GB disk to VM1 and 2 1GB disks to VM2. Assign a network interface, but do not configure the hostname and network connection.

    * The VirtualBox Network CIDR for the NAT network is 192.168.0.0/24.

    * On VM1, set the system hostname to rhcsa3.example.com and alias rhcsa3 using the hostnamectl command. Make sure that the new hostname is reflected in the command prompt:
        ```shell
        hostnamectl set-hostname rhcsa3.example.com
        ```

    * On rhcsa3, configure a network connection on the primary network device with IP address 192.168.0.243/24, gateway 192.168.0.1, and nameserver 192.168.0.1 using the nmcli command:
        ```shell
        nmcli con add type ethernet ifname enp0s3 con-name mycon ip4 192.168.0.243/24 gw4 192.168.0.1 ipv4.dns 192.168.0.1
        ```

    * On VM2, set the system hostname to rhcsa4.example.com and alias rhcsa4 using a manual method (modify file by hand). Make sure that the new hostname is reflected in the command prompt:
        ```shell
        vi /etc/hostname
        # change to rhcsa4.example.com
        ```

    * On rhcsa4, configure a network connection on the primary network device with IP address 192.168.0.244/24, gateway 192.168.0.1, and nameserver 192.168.0.1 using a manual method (create/modify files by hand):
        ```shell
        vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
        #TYPE=Ethernet
        #BOOTPROTO=static
        #DEFROUTE=yes
        #IPV4_FAILURE_FATAL=no
        #IPV4INIT=no
        #NAME=mycon
        #DEVICE=enp0s3
        #ONBOOT=yes
        #IPADDR=192.168.0.243
        #PREFIX=24
        #GATEWAY=192.168.0.1
        #DNS1=192.168.0.1
        ifup enp0s3
        nmcli con edit enp0s3 # play around with print ipv4 etc. to confirm settings
        ```

    * Run "ping -c2 rhcsa4" on rhcsa3. Run "ping -c2 rhcsa3" on rhcsa4. You should see 0% loss in both outputs:
        ```shell
        # on rhcsa3
        vi /etc/hosts
        # add line 192.168.0.244 rhcsa4
        ping rhcsa3 # confirm
        
        # on rhcsa4
        vi /etc/hosts
        # add line 192.168.0.243 rhcsa3
        ping rhcsa4 # confirm
        ```

    * On rhcsa3 and rhcsa4, attach the RHEL 8 ISO image to the VM and mount it persistently to `/mnt/cdrom`. Define access to both repositories and confirm:
        ```shell
        # attach disks in VirtualBox
        # on rhcsa3 and rhcsa4
        mkdir /mnt/cdrom
        mount /dev/sr0 /mnt/cdrom
        blkid # get UUID
        vi /etc/fstab
        # add line with UUID /mnt/cdrom iso9660 defaults 0 0
        mount -a # confirm
        vi /etc/yum.repos.d/image.repo
        #####
        #[BaseOS]
        #name=BaseOS
        #baseurl=file:///mnt/cdrom/BaseOS
        #enabled=1
        #gpgenabled=1
        #gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
        #
        #[AppStream]
        #name=AppStream
        #baseurl=file:///mnt/cdrom/AppStream
        #enabled=1
        #gpgenabled=1
        #gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
        #####
        yum repolist # confirm
        ```

    * On rhcsa3, add HTTP port 8300/tcp to the SELinux policy database:
        ```shell
        semange port -l | grep http # 8300 not in list for http_port_t
        semanage port -a -t http_port_t -p tcp 8300
        ```

    * On rhcsa3, create VDO volume vdo1 on the 4GB disk with logical size 16GB and mounted with Ext4 structures on `/mnt/vdo1`:
        ```shell
        TBC
        ```

    * Configure NFS service on rhcsa3 and share `/rh_share3` with rhcsa4. Configure AutoFS direct map on rhcsa4 to mount `/rh_share3` on `/mnt/rh_share4`. User user80 (create on both systems) should be able to create files under the share on the NFS server and under the mount point on the NFS client:
        ```shell
        # on rhcsa3
        mkdir /rh_share3
        chmod 777 rh_share3
        useradd user80
        passwd user80
        # enter Temp1234
        dnf install cifs-utils -y
        systemctl enable nfs-server.service --now
        firewall-cmd --add-service=nfs --permanent
        firewall-cmd --reload
        vi /etc/exports
        # add line rh_share3 rhcsa4(rw)
        exportfs -av
        
        # on rhcsa4
        useradd user80
        passwd user80
        # enter Temp1234
        mkdir /mnt/rh_share4
        chmod 777 rh_share4
        # mount rhcsa3:/rh_share3 /mnt/nfs
        # mount | grep nfs # get details for /etc/fstab
        # vi /etc/fstab
        # add line rhcsa3:/rh_share3 /mnt/rh_share4 nfs4 _netdev 0 0
        # above not required with AutoFS
        dnf install autofs -y
        vi /etc/auto.master
        # add line /mnt/rh_rhcsa3 /etc/auto.master.d/auto.home
        vi /etc/auto.master.d/auto.home
        # add line * -rw rhcsa3:/rh_share3
        ```

    * Configure NFS service on rhcsa4 and share the home directory for user user60 (create on both systems) with rhcsa3. Configure AutoFS indirect map on rhcsa3 to automatically mount the home directory under `/nfsdir` when user60 logs on to rhcsa3:
        ```shell
        # on rhcsa3
        useradd user60
        passwd user60
        # enter Temp1234
        dnf install autofs -y
        mkdir /nfsdir
        vi /etc/auto.master
        # add line for /nfsdir /etc/auto.master.d/auto.home
        vi /etc/auto.master.d/auto.home
        # add line for * -rw rhcsa4:/home/user60
        systemctl enable autofs.service --now

        # on rhcsa4
        useradd user60
        passwd user60
        # enter Temp1234
        vi /etc/exports
        # add line for /home rhcsa3(rw)
        exportfs -va    
        ```

    * On rhcsa4, create Stratis pool pool1 and volume str1 on a 1GB disk, and mount it to `/mnt/str1`:
        ```shell
        dnf provides stratis
        dnf install stratis-cli -y
        systemctl enable stratisd.service --now
        stratis pool create pool1 /dev/sdc
        stratis filesystem create pool1 vol1
        mkdir /mnt/str1
        mount /stratis/pool1/vol1 /mnt/str1
        blkid # get information for /etc/fstab
        vi /etc/fstab
        # add line for UUID /mnt/str1 xfs defaults 0 0    
        ```

    * On rhcsa4, expand Stratis pool pool1 using the other 1GB disk. Confirm that `/mnt/str1` sees the storage expansion:
        ```shell
        stratis pool add-data pool1 /dev/sdb
        stratis blockdev # extra disk visible
        ```

    * On rhcsa3, create a group called group30 with GID 3000, and add user60 and user80 to this group. Create a directory called `/sdata`, enable setgid bit on it, and add write permission bit for the group. Set ownership and owning group to root and group30. Create a file called file1 under `/sdata` as user user60 and modify the file as user80 successfully:
        ```shell
        TBC
        ```

    * On rhcsa3, create directory `/dir1` with full permissions for everyone. Disallow non-owners to remove files. Test by creating file `/tmp/dir1/stkfile1` as user60 and removing it as user80:
        ```shell
        TBC
        ```

    * On rhcsa3, search for all manual pages for the description containing the keyword "password" and redirect the output to file `/tmp/man.out`:
        ```shell
        man -k password >> /tmp.man.out
        # or potentially man -wK "password" if relying on the description is not enough
        ```

    * On rhcsa3, create file lnfile1 under `/tmp` and create one hard link `/tmp/lnfile2` and one soft link `/boot/file1`. Edit lnfile1 using the links and confirm:
        ```shell
        cd /tmp
        touch lnfile1
        ln lnfile1 lnfile2
        ln -s /boot/file1 lnfile1
        ```

    * On rhcsa3, install module postgresql version 9.6:
        ```shell
        dnf module list postgresql # stream 10 shown as default
        dnf module install postgresql:9.6
        dnf module list # stream 9.6 shown as installed
        ```

    * On rhcsa3, add the http service to the "external" firewalld zone persistently:
        ```shell
        firewall-cmd --zone=external --add-service=http --permanent
        ```

    * On rhcsa3, set SELinux type shadow_t on a new file testfile1 in `/usr` and ensure that the context is not affected by a SELinux relabelling:
        ```shell
        cd /usr
        touch /usr/testfile1
        ll -Zrt # type shown as unconfined_u:object_r:usr_t:s0
        semange fcontext -a -t /usr/testfile1
        restorecon -R -v /usr/testfile1
        ```

    * Configure password-less ssh access for user60 from rhcsa3 to rhcsa4:
        ```shell
        sudo su - user60
        ssh-keygen # do not provide a password
        ssh-copy-id rhcsa4 # enter user60 pasword on rhcsa4
        ```

1. RHCSA 8 Practise Exam

    * Interrupt the boot process and reset the root password:
        ```shell
        # interrupt boot process and add rd.break at end of linux line
        mount -o remount, rw /sysroot
        chroot /sysroot
        passwd 
        # enter new passwd
        touch /.autorelabel
        # you could also add enforcing=0 to the end of the Linux line to avoid having to do this
        # ctrl + D
        reboot
        ```

    * Repos are available from the repo server at http://repo.eight.example.com/BaseOS and http://repo.eight.example.com/AppStream for you to use during the exam. Setup these repos:
        ```shell
        vi /etc/yum.repos.d/localrepo.repo
        #[BaseOS]
        #name=BaseOS
        #baseurl=http://repo.eight.example.com/BaseOS
        #enabled=1
        #
        #[AppStream]
        #name=AppStream
        #baseurl=http://repo.eight.example.com/AppStream
        #enabled=1
        dnf repolist # confirm
        # you could also use dnf config-manager --add-repo
        ```

    * The system time should be set to your (or nearest to you) timezone and ensure NTP sync is configured:
        ```shell
        timedatectl set-timezone Australia/Sydney
        timedatectl set-ntp true
        timedatectl status # confirm status
        ```

    * Add the following secondary IP addresses statically to your current running interface. Do this in a way that doesn’t compromise your existing settings:
        ```shell
        # IPV4 - 10.0.0.5/24
        # IPV6 - fd01::100/64
        nmcli con edit System\ eth0
        goto ipv4.addresses 
        add 10.0.0.5/24
        goto ipv6.addresses 
        add fd01::100/64
        back
        save
        nmcli con edit System\ eth1
        goto ipv4.addresses 
        add 10.0.0.5/24
        goto ipv6.addresses 
        add fd01::100/64
        back
        save
        nmcli con reload
        # enter yes when asked if you want to set to manual
        ```

    * Enable packet forwarding on system1. This should persist after reboot:
        ```shell
        vi /etc/sysctl.conf
        # add line for net.ipv4.port_forward=1
        ```

    * System1 should boot into the multiuser target by default and boot messages should be present (not silenced):
        ```shell
        systemctl set-default multi-user.target
        vi /etc/default/grub
        # remove rhgb quiet from GRUB_CMDLINE_LINUX
        grub2-mkconfig -o /boot/grub2/grub.cfg
        reboot
        ```

    * Create a new 2GB volume group named “vgprac”:
        ```shell
        lsblk
        # /dev/sdb is available with 8GB
        # the file system already has ~36MB in use and is mounted to /extradisk1
        umount /dev/sdb
        parted /dev/sdb
        mklabel
        # enter msdos
        mkpart
        # enter primary
        # enter xfs
        # enter 0
        # enter 2.1GB
        set
        # enter 1
        # enter lvm
        # enter on
        vgcreate vgprac /dev/sdb1
        # enter y to wipe    
        ```

    * Create a 500MB logical volume named “lvprac” inside the “vgprac” volume group:
        ```shell
        lvcreate --name lvprac -L 500MB vgprac
        ```

    * The “lvprac” logical volume should be formatted with the xfs filesystem and mount persistently on the `/mnt/lvprac` directory:
        ```shell
        mkdir /mnt/lvprac
        mkfs.xfs /dev/mapper/vgprac-lvprac
        vi /etc/fstab
        # comment out line for old /dev/sdb
        # add line for /dev/mapper/vgprac-lvprac
        mount -a
        df -h # confirm mounted
        ```

    * Extend the xfs filesystem on “lvprac” by 500MB:
        ```shell
        lvextend -r -L +500MB /dev/vgprac/lvprac
        ```

    * Use the appropriate utility to create a 5TiB thin provisioned volume:
        ```shell
        lsblk
        # /dev/sdc is available with 8GB
        dnf install vdo kmod-vdo -y
        umount /extradisk2
        vdo create --name=myvolume --device=/dev/sdc --vdoLogicalSize=5T --force
        vi /etc/fstab
        # comment out line for old /dev/sdc
        ```

    * Configure a basic web server that displays “Welcome to the web server” once connected to it. Ensure the firewall allows the http/https services:
        ```shell
        vi /var/www/html/index.html
        # add line "Welcome to the web server"
        systemctl restart httpd.service
        curl http://localhost
        # success
        # from server1
        curl http://server2.eight.example.com
        # no route to host shown
        # on server2
        firewall-cmd --add-port=80/tcp --permanent
        firewall-cmd --reload
        # from server1
        curl http://server2.eight.example.com
        # success
        ```

    * Find all files that are larger than 5MB in the /etc directory and copy them to /find/largefiles:
        ```shell
        mkdir -p /find/largefiles
        find /etc/ -size +5M -exec cp {} /find/largefiles \;
        # the {} is substituted by the output of find, and the ; is mandatory for an exec but must be escaped
        ```

    * Write a script named awesome.sh in the root directory on system1. If “me” is given as an argument, then the script should output “Yes, I’m awesome.” If “them” is given as an argument, then the script should output “Okay, they are awesome.” If the argument is empty or anything else is given, the script should output “Usage ./awesome.sh me|them”:
        ```shell
        vi /awesome.sh
        chmod +x /awesome.sh
        # contents of awesome.sh
        ##!/bin/bash
        #if [ $1 = "me" ]; then
        #    echo "Yes, I'm awesome."
        #elif [ $1  = "them"]; then
        #    echo "Okay, they are awesome."
        #else
        #    echo "Usage /.awesome.sh me|them"
        #fi
        #note that = had to be used and not -eq
        ```

    * Create users phil, laura, stewart, and kevin. All new users should have a file named “Welcome” in their home folder after account creation. All user passwords should expire after 60 days and be at least 8 characters in length. Phil and laura should be part of the “accounting” group. If the group doesn’t already exist, create it. Stewart and kevin should be part of the “marketing” group. If the group doesn’t already exist, create it:
        ```shell
        groupadd accounting
        groupadd marketing
        vi /etc/security/pwquality.conf
        # uncomment out the line that already had minlen = 8
        mkdir /etc/skel/Welcome
        useradd phil -G accounting
        useradd laura -G accounting
        useradd stewart -G marketing
        useradd kevin -G marketing
        chage -M 60 phil
        chage -M 60 laura
        chage -M 60 stewart
        chage -M 60 kevin
        chage -l phil # confirm
        # can also change in /etc/login.defs
        ```

    * Only members of the accounting group should have access to the `/accounting` directory. Make laura the owner of this directory. Make the accounting group the group owner of the `/accounting` directory:
        ```shell
        mkdir /accounting
        chmod 770 /accounting
        chown laura:accounting /accounting
        ```

    * Only members of the marketing group should have access to the `/marketing` directory. Make stewart the owner of this directory. Make the marketing group the group owner of the `/marketing` directory:
        ```shell
        mkdir /marketing
        chmod 770 /marketing
        chown stewart:marketing /marketing
        ```

    * New files should be owned by the group owner and only the file creator should have the permissions to delete their own files:
        ```shell
        chmod +ts /marketing
        chmod +ts /accounting
        ```

    * Create a cron job that writes “This practice exam was easy and I’m ready to ace my RHCSA” to `/var/log/messages` at 12pm only on weekdays:
        ```shell
        crontab -e
        #* 12 * * 1-5 echo "This practise exam was easy and I'm ready to ace my RHCSA" >> /var/log/messagees
        # you can look at info crontab if you forget the syntax
        ```

## RHCE 9

- [Be able to perform all tasks expected of a Red Hat Certified System Administrator](#Be-able-to-perform-all-tasks-expected-of-a-Red-Hat-Certified-System-Administrator)
- [Understand core components of Ansible](#Understand-core-components-of-Ansible)
- [Use roles and Ansible Content Collections](#Use-roles-and-Ansible-Content-Collections)
- [Install and configure an Ansible control node](#Install-and-configure-an-Ansible-control-node)
- [Configure Ansible managed nodes](#Configure-Ansible-managed-nodes)
- [Run playbooks with Automation content navigator](#Run-playbooks-with-Automation-content-navigator)
- [Create Ansible plays and playbooks](#Create-Ansible-plays-and-playbooks)
- [Automate standard RHCSA tasks using Ansible modules that work with](#Automate-standard-RHCSA-tasks-using-Ansible-modules-that-work-with)
- [Manage-content](#Manage-content)
- [RHCE Exercises](#RHCE-Exercises)

### Be able to perform all tasks expected of a Red Hat Certified System Administrator

1. Understand and use essential tools

1. Operate running systems

1. Configure local storage

1. Create and configure file systems

1. Deploy, configure, and maintain systems

1. Manage users and groups

1. Manage security

### Understand core components of Ansible

1. Inventories

    * Inventories are what Ansible uses to locate and run against multiple hosts. The default ansible 'hosts' file is `/etc/ansible/hosts`. The default location of the hosts file can be set in `/etc/ansible/ansible.cfg`.

    * The file can contain individual hosts, groups of hosts, groups of groups, and host and group level variables. It can also contain variables that determine how you connect to a host.

    * An example of an INI-based host inventory file is shown below:
        ```shell
        mail.example.com

        [webservers]
        web01.example.com
        web02.example.com

        [dbservers]
        db[01:04].example.com
        ```

    * Note that square brackets can be used instead of writing a separate line for each host.

    * An example of a YAML-based host inventory file is shown below:
        ```shell
        all:
            hosts:
                mail.example.com
            children:
                webservers:
                    hosts:
                        web01.example.com
                        web02.example.com
                dbservers:
                    hosts:
                        db[01:04].example.com
        ```

1. Modules

    * Modules are essentially tools for tasks. They usually take parameters and return JSON. Modules can be run from the command line or within a playbook. Ansible ships with a significant number of modules by default, and custom modules can also be written.

    * Each specific task in Ansible is written through a Module. Multiple Modules are written in sequential order. Multiple Modules for related Tasks are called a Play. All Plays together make a Playbook.
Dragon’s Dogma: Dark Arise
    * To run modules through a YAML file:
        ```shell
        ansible-playbook example.yml
        ```

    * The `--syntax-check` parameter can be used to check the syntax before running a playbook. The `-C` parameter can be used to dry-run a playbook. 

    * To run a module independently:
        ```shell
        ansible myservers -m ping
        ```

1. Variables

    * Variable names should only contain letters, numbers, and underscores. A variable name should also start with a letter. There are three main scopes for variables: Global, Host and Play.

    * Variables are typically used for configuration values and various parameters. They can store the return value of executed commands and may also be dictionaries. Ansible provides a number of predefined variables.

    * An example of INI-based based variables:
        ```yaml
        [webservers]
        host1 http_port=80 maxRequestsPerChild=500
        host2 http_port=305 maxRequestsPerChild=600
        ```

    * An example of YAML-based based variables:
        ```yaml
        webservers
            host1:
                http_port: 80
                maxRequestsPerChild: 500
            host2:
                http_port: 305
                maxRequestsPerChild: 600
        ```

    * An example of variables used within a playbook is shown below:
        ```yaml
        ---
        - name: Create a file with vars
          hosts: localhost
          vars:
            fileName: createdusingvars
          tasks:
            - name: Create a file
              file:
                state: touch
                path: /tmp/{{ fileName }}.txt
        ```

	* When referencing a variable and the `{` character is the first character in the line, the reference must be enclosed in quotes.

1. Facts

    * Facts provide certain information about a given target host. They are automatically discovered by Ansible when it reaches out to a host. Facts can be disabled and can be cached for use in playbook executions.

    * To gather a list of facts for a host:
        ```shell
        ansible localhost -m setup
        ```

    * You can also include this task in a playbook to list the facts:
        ```yaml
        ---
        - name: Print all available facts
          ansible.builtin.debug:
            var: ansible_facts
        ```

    * Nested facts can be referred to using the below syntax:
        ```shell
        {{ ansible_facts['devices']['xvda']['model'] }}
        ```

    * In addition to Facts, Ansible contains many special (or magic) variables that are automatically available.

    * The `hostvars` variable contains host details for any host in the play, at any point in the playbook. To access a fact from another node, you can use the syntax `{{  hostvars['test.example.com']['ansible_facts']['distribution']  }}`.

    * The `groups` variable lists all of the groups and hosts in the inventory. You can use this to enumerate the hosts within a group:
        ```jinja2
        {% for host in groups['app_servers'] %}
        # something that applies to all app servers.
        {% endfor %}
        ```

    * The below example shows using hostvars and groups together to find all of the IP addresses in a group:
        ```jinja2
        {% for host in groups['app_servers'] %}
        {{ hostvars[host]['ansible_facts']['eth0']['ipv4']['address'] }}
        {% endfor %}
        ```

    * The `group_names` variable contains a list of all the groups the current host is in. The below example shows using this variable to create a templated file that varies based on the group membership of the host:
        ```jinja2
        {% if 'webserver' in group_names %}
        # some part of a configuration file that only applies to webservers
        {% endif %}
        ```

    * The `inventory_hostname` variable contains the host as configured as configured in your inventory, as an alternative to `ansible_hostname` when fact-gathering is disabled (`gather_facts: no`). You can also use `inventory_hostname_short` which contains the part up to the first period.

    * The `ansible_play_hosts` is the list of all hosts still active in the current play.

    * The `ansible_play_batch` is a list of hostnames that are in scope for the current ‘batch’ of the play. The batch size is defined by `serial`, when not set it is equivalent to the whole play.

    * The `ansible_playbook_python` is the path to the Python executable used to invoke the Ansible command line tool.

    * The `inventory_dir` is the pathname of the directory holding Ansible’s inventory host file.

    * The `playbook_dir` contains the playbook base directory.

    * The `role_path` contains the current role’s pathname and only works inside a role.

    * The `ansible_check_mode` is a boolean, set to `True` if you run Ansible wtih `--check`.

    * Special variables can be viewed in a playbook such as the below:
        ```yaml
        ---
        - name: Show some special variables
          hosts: localhost
          tasks:
            - name: Show inventory host name
              debug:
                msg: "{{  inventory_hostname  }}"

            - name: Show group
              debug:
                msg: "{{  group_names  }}"

            - name: Show host variables
              debug:
                msg: "{{  hostvars  }}"
        ```

	* Custom facts

1. Loops

    * Loops work hand in hand with conditions as we can loop certain tasks until a condition is met. Ansible provides the `loop` and `with_*` directives for creating loops.

    * An example is shown below:
        ```yaml
        ---
        - name: Create users with loop
          hosts: localhost

          tasks:
            - name: Create users
              user:
                name: "{{ item }}"
              loop:
                - jerry
                - kramer
                - elaine
        ```

    * An alternate example is shown below:
        ```yaml
        ---
        - name: Create users through loop
          hosts: localhost

          vars:
            users: [jerry,kramer,elaine]

          tasks:
            - name: Create users
              user:
                name: "{{ item }}"
              with_items: "{{ users }}"
        ```

    * An example for installing packages using a loop is shown below:
        ```yaml
        ---
        - name: Install packages using a loop
          hosts: localhost

          vars:
            packages: [ftp, telnet, htop]

          tasks:
            - name: Install package
              yum:
                name: "{{ item }}"
                state: present
              with_items: "{{ packages }}"
        ```

    * An alternate example for installing packages which is possible because yum provides the loop implementation natively:
        ```yaml
        ---
        - name: Install packages through loop
          hosts: localhost

          vars:
            packages: [ftp, telnet, htop]

          tasks:
            - name: Install packages
              yum:
                name: "{{ packages }}"
                state: present
        ```

1. Conditional tasks

	* The `when` keyboard is used to make tasks conditional.

1. Plays

    * The goal of a play is to map a group of hosts to some well-defined roles. A play can consist of one or more tasks which make calls to Ansible modules.

1. Handling task failure

	* By default, Ansible stops executing tasks on a host when a task fails on that host. You can use the `ignore_errors` option to continue despite the failure. This directive only works when the task can run and returns a value of 'failed'. For example, the condition `when: response.failed` is used to trigger if the registered variable `response` has returned a failure.
	
	* Ansible lets you define what "failure" means in each task using the `failed_when` conditional. This is commonly used on registered variables:
		```yaml
		- name: Fail task when both files are identical
		  ansible.builtin.raw: diff foo/file1 bar/file2
		  register: diff_cmd
		  failed_when: diff_cmd.rc == 0 or diff_cmd.rc >= 2
        ```

	* Ansible lets you define when a particular task has “changed” a remote node using the `changed_when` conditional. This lets you determine, based on return codes or output, whether a change should be reported in Ansible statistics and whether a handler should be triggered or not. usually a handler is used to handle executing tasks only if there is a change.

1. Playbooks

    * A playbook is a series of plays. An example of a playbook:
        ```yaml
        ---
        - hosts: webservers
          become: yes
          tasks:
            - name: ensure apache is at the latest version
              yum:
                name: httpd
                state: latest
            - name: write our custom apache config file
              template:
                src: /srv/httpd.j2
                dest: /etc/httpd/conf/httpd.conf
            - name: ensure that apache is started
              service:
                name: httpd
                state: started
            - hosts: dbservers
              become: yes
              tasks:
              - name: ensure postgresql is at the latest version
                yum:
                  name: postgresql
                  state: latest
              - name: ensure that postgres is started
                service:
                  name: postgresql
                  state: started
        ```

1. Configuration Files

    * The Ansible configuration files are taken from the below locations in order:
        * `ANSIBLE_CONFIG` (environment variable)
        * `ansible.cfg` (in the current directory)
        * `~/.ansible.cfg` (in the home directory)
        * `/etc/ansible/ansible.cfg`

    * A configuration file will not automatically load if it is in a world-writable directory.

    * The ansible-config command can be used to view configurations:
        * list - Prints all configuration options
        * dump - Dumps configuration
        * view - View the configuration file

    * Commonly used settings:
        * inventory - Specifies the default inventory file
        * roles_path - Sets paths to search in for roles
        * forks - Specifies the amount of hosts configured by Ansible at the same time (Parallelism)
        * ansible_managed - Text inserted into templates which indicate that file is managed by Ansible and changes will be overwritten

1. Roles

    * Roles help to split playbooks into smaller groups. Roles let you automatically load related vars, files, tasks, handlers, and other Ansible artifacts based on a known file structure. After you group your content in roles, you can easily reuse them and share them with other users.

    * Consider the playbook below:
        ```yaml
        ---
        - name: Setup httpd webserver
          hosts: east-webservers

          tasks:
            - name: Install httpd packages
              yum:
                name: httpd
                state: present

            - name: Start httpd
              service:
                name: httpd
                state: started

            - name: Open port http on firewall
              firewalld:
                service: http
                permanent: true
                state: enabled

            - name: Restart firewalld
              service:
                name: firewalld
                state: restarted

          hosts: west-webservers
          tasks:
            - name: Install httpd packages
              yum:
                name: httpd
                state: present

            - name: Start httpd
              service:
                name: httpd
                state: started
        ```

    * This playbook can be simplified with the addition of two roles. The first role is created in `/etc/ansible/roles/fullinstall/tasks/main.yml`:
        ```yaml
        ---
        - name: Install httpd packages
          yum:
            name: httpd
            state: present

        - name: Start httpd
          service:
            name: httpd
            state: started

        - name: Open port http on firewall
          firewalld:
            service: http
            permanent: true
            state: enabled

        - name: Restart firewalld
          service:
            name: firewalld
            state: restarted
        ```

    * The second role is created in `/etc/ansible/roles/basicinstall/tasks/main.yml`:
        ```yaml
        ---
        - name: Install httpd packages
          yum:
            name: httpd
            state: present

        - name: Start httpd
          service:
            name: httpd
            state: started
        ```

    * The simplified playbook is shown below:
        ```yaml
        ---
        - name: Full install
          hosts: all
          roles:
            - fullinstall

        - name: Basic install
          hosts: all
          roles:
            - basicinstall
        ```

    * By default, Ansible downloads roles to the first writable directory in the default list of paths `~/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles`. This installs roles in the home directory of the user running ansible-galaxy. You can override this behaviour by setting `ANSIBLE_ROLES_PATH`, using the `--roles-path` option for the `ansible-galaxy` command, or defining `roles_path` in an `ansible.cfg` file.

1. Use provided documentation to look up specific information about Ansible modules and commands

    * To check the documentation for the `yum` module:
        ```shell
        ansible-doc -t module yum
        ```

### Use roles and Ansible Content Collections

1. Create and work with roles

    * Roles simplify long playbooks by grouping tasks into smaller playbooks.

1. Install roles and use them in playbooks

	* The default location for roles is in `/etc/ansible/roles` which is configurable as `roles_path` in `ansible.cfg`. Ansible also looks for roles in collections, in the `roles` directory relative to the playbook file, and in the directory where the playbook file is located.

	* To install a role run:
        ```shell
        ansible-galaxy install author.role
        ```

	* To install a role from an FTP locations $URL, first run `wget $URL`. Then create a `requirements.yml` file with each collection specified using one of these methods:
        ```yml
		- name: http-role-gz
		  src: https://some.webserver.example.com/files/main.tar.gz
        ```

	* Run the following:
        ```shell
        ansible-galaxy install -r requirements.yml -p .
		ansible-galaxy install -r requirements.yml -p . # allow documentation commands to work
        ```

1. Install Content Collections and use them in playbooks

	* To install a collection run:
        ```shell
        ansible-galaxy collection install authorname.collectionname
        ```

	* Collections can be used in playbooks by using:
        ```shell
        tasks:
		 import_role: 
		    name: awx.awx
        ```

	* Or by using:
        ```shell
        collections:
		 name: awx.awx 
		tasks:
        ```

	* The Fully Qualified Collection Name (FQCN) is used when referring to modules provided by an Ansible Collection.

1. Obtain a set of related roles, supplementary modules, and other content from content collections, and use them in a playbook.

    * To install the `ansible-posix` collection:
        ```shell
        ansible-galaxy collection install ansible-posix
        ```

	* To install the `ansible-posix` collection from an FTP locations $URL, first run `wget $URL`. Then create a `requirements.yml` file with each collection specified using one of these methods:
        ```yml
		collections:
		  # directory containing the collection
		  - source: ./my_namespace/my_collection/
		    type: dir
		
		  # directory containing a namespace, with collections as subdirectories
		  - source: ./my_namespace/
		    type: subdirs
        ```

	* Run the following:
        ```shell
        ansible-galaxy collection install -r requirements.yml -p .
		ansible-galaxy collection install -r requirements.yml # allow documentation commands to work
        ```

	* Note that the collection is installed into `~./ansible/collections/ansible_collections`. This can be overwritten with `-p /path/to/collection`, but you must update `ansible.cfg` accordingly. The documentation of the installed collection can be referred, but the `ansible-doc` command only searches the system directories for documentation.

	* Create and run a playbook enforce-selinux.yml:
        ```yaml
		 ---
		- name: set SELinux to enforcing
		  hosts: localhost
		  become: yes
		  tasks:
		  - name: set SElinux to enforcing
		    ansible.posix.selinux:
		      policy: targeted
		      state: enforcing
        ```

### Install and configure an Ansible control node

1. Install required packages

    * To install Ansible using dnf:
        ```shell
        sudo subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
        sudo dnf install epel-release epel-next-release
        dnf install -y ansible-core
        ```

1. Create a static host inventory file

    * An inventory is a list of hosts that Ansible manages. Inventory files may contain hosts, patterns, groups, and variables. Multiple inventory files may be specified using a directory. Inventory files may be specified in INI or YAML format.

    * The default location is `/etc/ansible/hosts`. The location can be set in `ansible.cfg` or specified in the CLI using:
        ```shell
        ansible -i <filename>
        ```

    * Best practices for inventory variables:
        * Variables should be stored in YAML files located relative to the inventory file.
        * Host and group variables should be stored in the `host_vars` and `group_vars` directories respectively (the directories need to be created).
        * Variable files should be named after the host or group for which they contain variables (files may end in .yml or .yaml).

    * A host file can be static or dynamic. A dynamic host file can reflect updated IP addresses automatically but requires additional plugins.

    * To view the hosts in the file:
        ```shell
        ansible-inventory --list
        ```

1. Create a configuration file

    * An example of creating a custom configuration file, and updating the default configuration file:
        ```shell
        cd ansible
        vi ansible.cfg
        ### contents of file
        [defaults]
        interpreter_python = auto
        inventory = /home/cloud_user/ansible/inventory/inv.ini
        roles_path = /etc/ansible/roles:/home/cloud_user/ansible/roles       
        ```

1. Create and use static inventories to define groups of hosts

	* A sample inventory file is shown below:
        ```yaml
		[webservers]
		foo.example.com
		bar.example.com
		
		[dbservers]
		one.example.com
		two.example.com
		three.example.com
        ```

### Configure Ansible managed nodes

1. Create and distribute SSH keys to managed nodes

    * A control node is any machine with Ansible installed. You can run Ansible commands and playbooks from any control node. A managed node (also sometimes called a "host") is a network device or server you manage with Ansible. Ansible is not installed on managed nodes.

    * The following is an example of generating SSH keys on the control node and distributing them to managed nodes mypearson2c and mypearson3c:
        ```shell
        ssh-keygen
        # enter password
        # now we have id.rsa and id_rsa.pub in /home/cloud_user/.ssh/
        ssh-copy-id cloud_user@mspearson2c.mylabserver.com
        # enter password
        ssh-copy-id cloud_user@mspearson3c.mylabserver.com
        # enter password
        ```

	* A playbook can be used to distribute keys to managed nodes. To install the `openssh_keypair` module:
        ```shell
        ansible-galaxy collection install community.crypto
        ```

	* Create a playbook `/home/ansible/ansible/bootstrap.yml`:
        ```yaml
		---
		- name: Key preparation
		  hosts: localhost
		  tasks:
		    - name: SSH key directory exists
		      file:
		        path: /home/ansible/.ssh
		        state: directory
		        mode: 0700

		    - name: SSH keypair exists
		      community.crypto.openssh_keypair:
		        path: /home/ansible/.ssh/id_rsa
		        state: present

		- name: Bootstrap automation user and install keys
		  hosts: all
		  tasks:
		    - name: Confirm user exists
		      user:
		        name: ansible
		        state: present

		    - name: sudo passwordless access for ansible user
		      copy:
		        content: "ansible ALL=(ALL) NOPASSWD:ALL"
		        dest: /etc/sudoers.d/ansible
		        validate: /usr/sbin/visudo -csf %s

		    - name: Public key is in remote authorized keys
		      ansible.posix.authorized_key:
		        user: ansible
		        state: present
		        key: "{{ lookup('file', '/home/ansible/.ssh/id_rsa.pub')  }}"
        ```

	* Note that until you have setup the keys, in `ansible.cfg` you will need to set `ask_pass=true` and `host_key_checking=false` under `[defaults]`, and `become_ask_pass=true` under `[privilege_escalation]`. This will allow you to provide the authentication and sudo passwords as part of the playbook execution.

1. Configure privilege escalation on managed nodes

    * The following is an example of configuring privilege escalation on managed nodes mypearson2c and mypearson3c:
        ```shell
        # perform these steps on both mypearson2c and mypearson3c
        sudo visudo
        # add line
        cloud_user ALL=(ALL) NOPASSWD: ALL
        ```

	* The example above provides a playbook for setting up privilege escalation on managed nodes.

1. Deploy files to managed nodes

	* The copy module can be used to copy files into managed nodes.

1. Be able to analyze simple shell scripts and convert them to playbooks

	* Common Ansible modules can be used to replicate the functionailty of scripts. The `command` and `shell` modules allow you to execute commands on the managed nodes.

### Run playbooks with Automation content navigator

1. Know how to run playbooks with Automation content navigator

	* Ansible content navigator is a command line, content-creator-focused tool with a text-based user interface. You can use it to launch and watch jobs and playbooks, share playbook and job run artifacts in JSON, browse and introspect automation execution environments, and more.

	* Install Ansible Navigator:
        ```shell
		# install podman
		sudo dnf install container-tools
		# install ansible-navigator
        sudo dnf install python3-pip
		python3 -m pip install ansible-navigator --user
		ansible-navigator
        ```

	* You can also install Ansible Navigator using:
        ```shell
		subscription-manager register
		subscription-manager list --available # find the Pool ID for the subscription containing Ansible Automation Platform
		subscription-manager attach --pool-=$Pool_ID
		subscription-manager repos --list | grep ansible
		subscription-manager repos --enable ansible-automation-platform-2.4-for-rhel-9-aarch64-rpms
		dnf install ansible-navigator -y
        ```

	* After running `ansible-navigator` you are presented with an interactive interface. You can type `:run <playbook> -i <inventory>` to run a playbook. You can also run a playbook using `ansible-navigator run -m stdout $playbook`. This provides backwards compatability with the standard `ansible-playbook` command.

	* Note that there is no `ansible-doc` for `ansible-navigator`, but you can use `ansible-navigator --help`.

1. Use Automation content navigator to find new modules in available Ansible Content Collections and use them

	* Run the following:
        ```shell
		ansible-galaxy collection install community.general -p collections
		ansible-galaxy collection # collection shows as type bind_mount
        ```

1. Use Automation content navigator to create inventories and configure the Ansible environment

	* Run the following:
        ```shell
		ansible-navigator inventory -i assets/inventory.cfg # inspect an inventory
		ansible-navigator config
        ```

### Create Ansible plays and playbooks

1. Know how to work with commonly used Ansible modules

    * Commonly used modules include:
        * Ping
            * Validates a server is running and reachable.
            * No required parameters.
        * Setup
            * Gather Ansible facts.
            * No required parameters.
        * Yum
            * Manage packages with the YUM package manager.
            * Common parameters are name and state.
        * Service
            * Control services on remote hosts.
            * Common parameters are name (required), state and enabled.
        * User
            * Manage user accounts and attributes.
            * Common parameters are name (required), state, group and groups.
        * Copy
            * Copy files to a remote host.
            * Common parameters are src, dest (required), owner, group and mode.
        * File
            * Manage files and directories.
            * Common parameters are path (required), state, owner, group and mode.
        * Git
            * Interact with git repositories.
            * Common parameters are repo (required), dest (required) and clone.

    * A sample playbook to copy a file to a remote client:
        ```yaml
        ---
        - name: Copy a file from local to remote
          hosts: all
          tasks:
          - name: Copying file
            copy:
              src: /apps/ansible/temp
              dest: /tmp
              owner: myname
              group: myname
              mode: 0644
        ```

    * A sample playbook to change file permissions:
        ```yaml
        --
        - name: Change file permissions
          hosts: all
          tasks:
            - name: File Permissions
              file:
                path: /etc/ansible/temp
                mode: '0777'
        ```

    * A sample playbook to check a file or directory status:
        ```yaml
        ---
        - name: File status module
          hosts: localhost
          tasks:
          - name: Check file status and attributes
            stat:
              path: /etc/hosts
            register: fs

          - name: Show results
            debug:
              msg: File attributes {{ fs }}
        ```

    * A sample playbook to create a directory or file and remove
        ```yaml
        ---
        - name: Create and Remove file
          hosts: all
          tasks:
          - name: Create a directory
            file:
              path: /tmp/seinfeld
              owner: myuser
              mode: 770
              state: directory

          - name: Create a file in that directory
            file:
              path: /tmp/seinfeld/jerry
              state: touch

          - name: Stat the new file jerry
            stat:
              path: /tmp/seinfeld/jerry
            register: jf

          - name: Show file status
            debug:
              msg: File status and attributes {{ jf }}

          - name: Remove file
            file:
              path: /tmp/seinfeld/jerry
              state: absent
        ```

    * A sample playbook to create a file and add text:
        ```yaml
        ---
        - name: Create a file and add text
          hosts: localhost
          tasks:
          - name: Create a new file
            file:
              path: /tmp/george
              state: touch

          - name: Add text to the file
            blockinfile:
              path: /tmp/george
              block: Sample text
        ```

    * A sample playbook to setup httpd and open a firewall port:
        ```yaml
        ---
        - name: Setup httpd and open firewall port
          hosts: all
          tasks:
            - name: Install apache packages
              yum:
                name: httpd
                state: present

            - name: Start httpd
              service:
                name: httpd
                state: started

            - name: Open port 80 for http access
              firewalld:
                service: http
                permanent: true
                state: enabled

            - name: Restart firewalld service to load firewall changes
              service:
                name: firewalld
                state: reloaded
        ```

    * Note that managing firewalld requires the following collection:
        ```shell
        ansible-galaxy collection install --ignore-certs ansible.posix
        ```

    * A sample playbook to run a shell script:
        ```yaml
        ---
        - name: Playbook for shell script
          hosts: all
          tasks:
            - name: Run shell script
              shell: "/home/username/cfile.sh"
        ```

    * A sample playbook to setup a cronjob:
        ```yaml
        ---
        - name: Create a cron job
          hosts: all
          tasks:
            - name: Schedule cron
              cron:
                name: This job is scheduled by Ansible
                minute: "0"
                hour: "10"
                day: "*"
                month: "*"
                weekday: "4"
                user: root
                job: "/home/username/cfile.sh"
        ```

    * A sample playbook to download a file:
        ```yaml
        ---
        - name: Download Tomcat from tomcat.apache.org
          hosts: all
          tasks:
            - name: Create a directory /opt/tomcat
              file:
                path: /opt/tomcat
                state: directory
                mode: 0755
                owner: root
                group: root
            - name: Download Tomcat using get_url
              get_url:
                url: https://dlcdn.apache.org/tomcat/tomcat-8/v8.5.95/bin/apache-tomcat-8.5.95.tar.gz
                dest: /opt/tomcat
                mode: 0755
        ```

    * A sample playbook mount a filesystem:
        ```yaml
        ---
        - name: Create and mount new storage
          hosts: all
          tasks:
            - name: Create new partition
              parted:
                name: files
                label: gpt
                device: /dev/sdb
                number: 1
                state: present
                part_start: 1MiB
                part_end: 1GiB

            - name: Create xfs filesystem
              filesystem:
                dev: /dev/sdb1
                fstype: xfs

            - name: Create mount directory
              file:
                path: /data
                state: directory

            - name: Mount the filesystem
              mount:
                src: /dev/sdb1
                path: /data
                fstype: xfs
                state: mounted
        ```

    * A sample playbook to create a user:
        ```yaml
        ---
        - name: Playbook for creating users
          hosts: all
          tasks:
            - name: Create users
              user:
                name: george
                home: /home/george
                shell: /bin/bash
        ```

    * A sample playbook to update the password for a user:
        ```yaml
        ---
        - name: Add or update user password
          hosts: all
          tasks:
            - name: Change "george" password
              user:
                name: george
                update_password: always
                password: "{{ newpassword|password_hash('sha512') }}"
        ```

    * To run the playbook and provide a password:
        ```shell
        ansible-playbook changepass.yml --extra-vars newpassword=abc123
        ```

    * A sample playbook to kill a process:
        ```yaml
        ---
        - name: Find a process and kill it
          hosts: 192.168.1.105
          tasks:
            - name: Get running process from remote host
              ignore_errors: yes
              shell: "ps -few | grep top | awk '{print $2}'"
              register: running_process

            - name: Show process
              debug:
                msg: Processes are {{ running_process }}

            - name: Kill running processes
              ignore_errors: yes
              shell: "kill {{ item }}"
              with_items: "{{ running_process.stdout_lines }}"
        ```

    * A sample playbook to kill a process:
        ```yaml
        ---
        - name: Find a process and kill it
          hosts: 192.168.1.105
          tasks:
            - name: Get running process from remote host
              ignore_errors: yes
              shell: "ps -few | grep top | awk '{print $2}'"
              register: running_process

            - name: Show process
              debug:
                msg: Processes are {{ running_process }}

            - name: Kill running processes
              ignore_errors: yes
              shell: "kill {{ item }}"
              with_items: "{{ running_process.stdout_lines }}"
        ```

    * To run a playbook from a particular task:
        ```shell
        ansible-playbook yamlfile.yml --start-at-task 'Task name'
        ```

    * The `-b` flag will run a module in become mode (privilege escalation).

1. Use variables to retrieve the results of running a command

    * The register keyword is used to store the results of running a command as a variable. Variables can then be referenced by other tasks in the playbook. Registered variables are only valid on the host for the current playbook run. The return values differ from module to module.

    * A sample playbook register.yml is shown below:
        ```yaml
        ---
        - hosts: mspearson2
          tasks:
            - name: create a file
              file:
                path: /tmp/testFile
                state: touch
              register: var
            - name: display debug msg
              debug: msg="Register output is {{ var }}"
            - name: edit file
              lineinfile:
                path: /tmp/testFile
                line: "The uid is {{ var.uid }} and gid is {{ var.gid }}"
        ```

    * This playbook is run using:
        ```shell
        ansible-playbook playbooks/register.yml
        ```

    * The result stored in `/tmp/testFile` shows the variables for uid and gid.

1. Use conditionals to control play execution

    * Handlers are executed at the end of the play once all tasks are finished. They are typically used to start, reload, restart, or stop services.  Sometimes you only want to run a task when a change is made on a machine. For example, restarting a service only if a task updates the configuration of that service.

    * A sample handler is shown below:
        ```yaml
        ---
        - name: Verify Apache installation
          hosts: localhost

          tasks:
            - name: Ensure Apache is at the latest version
              yum:
                name: httpd
                state: latest

            - name: Copy updated Apache config file
              copy:
                src: /tmp/httpd.conf
                dest: /etc/httpd.conf
              notify:
                - Restart Apache

            - name: Ensure Apache is running
              service:
                name: httpd
                state: started

          handlers:
            - name: Restart Apache
              service:
                name: httpd
                state: restarted
        ```

    * The when statement can be used to conditionally execute tasks. An example is shown below:
        ```yaml
        ---
        - name: Install Apache WebServer
          hosts: localhost
          tasks:
            - name: Install Apache on an Ubuntu Server
              apt-get:
                name: apache2
                state: present
              when: ansible_os_family == "Ubuntu"

            - name: Install Apache on CentOS Server
              yum:
                name: httpd
                state: present
              when: ansible_os_family == "Redhat"
        ```

    * Tags are references or aliases to a task. Instead of running an entire playbook, use tags to target a specific task you need to run. Consider the playbook `httpbytags.yml` shown below:
        ```yaml
        ---
        - name: Setup Apache server
          hosts: localhost
          tasks:
            - name: Install httpd
              yum:
                name: httpd
                state: present
              tags: i-httpd
            - name: Start httpd
              service:
                name: httpd
                state: started
              tags: s-httpd
        ```

    * To list tags in a playbook run `ansible-playbook httpbytags.yml --list-tags`.

    * To target tasks using tags run `ansible-playbook httpbytags.yml -t i-httpd`. To skip tasks using tags run `ansible-playbook httpbytags.yml --skip-tags i-httpd`

1. Configure error handling

	* By default, Ansible stops executing tasks on a host when a task fails on that host. You can use `ignore_errors` to continue despite of the failure.

1. Create playbooks to configure systems to a specified state

	* Various example playbooks are provided in the surrounding sections.

### Automate standard RHCSA tasks using Ansible modules that work with:

1. Software packages and repositories

	* The `yum` module in `ansible-core` can be used to manage packages.

1. Services

	* The `service` module in `ansible-core` can be used to manage services.

1. Firewall rules

	* The `ansible.posix.firewalld` module in the `ansible.posix` collection can be used to manage firewall rules.

1. File systems

	* The `community.general.filesystem` module in the `community.general` collection can be used to manage file systems.

1. Storage devices

	* Various modules in the `community.general` collection can be used to manage block devices.

1. File content

	* The `copy` module in `ansible-core` can be used to copy file contents.

1. Archiving

	* The `community.general.archive` module in the `community.general` collection can be used to create or extend an archive.

1. Task scheduling

	* The `cron` module in `ansible-core` can be used to manage task scheduling.

1. Security

	* The `community.general.sefcontext` module in the `community.general` collection can be used to manage SELinux file contexts.

1. Users and groups

	* The `user` and `group` modules in `ansible-core` can be used to manage users and groups.

### Manage content

1. Create and use templates to create customized configuration files

	* The `template` module in `ansible-core` can be used with templates to populate files on managed hosts.

1. Use Ansible Vault in playbooks to protect sensitive data

    * An encrypted playbook can be created using `ansible-playbook $playbook_name --ask-vault-pass`. The YAML file can then be executed by `ansible-playbook $playbook_name --ask-vault-pass`.

    * A non-encrypted playbook can be encrypted using `ansible-vault encrypt $playbook_name`.

    * An encrypted playbook can be viewed using `ansible-vault view $playbook_name` and edited with `ansible-vault edit httpvault.yml`. You will be prompted for passwords where they are not provided in the command.

    * Instead of encrypting the entire playbook, strings within the playbook can be encrypted. This is done using `ansible-vault encrypt_string $string`. The encrypted value can be substituted for the variable value inside of the playbook. The `--ask-vault-pass` option must be provided when running the playbook regardless of whether individual strings are encrypted or the entire playbook.

### RHCE Exercises

#### Practise Exam 1

1. Setup

    * Create a VirtualBox machine for the control node. In VirtualBox create a new machine with the VirtualBox additions and the RHEL 9.2 installer ISOs attached as IDE devices. Set networking mode to Bridged Adapter. Set the Pointing Device to USB Tablet and enable the USB 3.0 Controller. Set shared Clipboard and Drag'n'Drop to Bidirectional. Uncheck Audio output. Start the machine and install RHEL.

    * On the control node run the following setup as root:
        ```shell
        blkid # note the UUID of the RHEL ISO
        mkdir /mnt/rheliso
        vi /etc/fstab
        # add the below line
        # UUID="2023-04-13-16-58-02-00" /mnt/rheliso iso9660 loop 0 0
        mount -a # confirm no error is returned
        ```

    * On the control node setup the dnf repositories as root:
        ```shell
        vi /etc/yum.repos.d/redhat.repo # add the below content
        # [BaseOS]
        # name=BaseOS
        # baseUrl=file:///mnt/rheliso/BaseOS
        # enabled=1
        # gpgcheck=0
        #
        # [AppStream]
        # name=AppStream
        # enabled=1
        # baseurl=file:///mnt/rheliso/AppStream
        # gpgcheck=0
        dnf repolist # confirm repos are returned
        ```

    * On the control node run the following setup as root to install guest additions:
        ```shell
        dnf install kernel-headers kernel-devel
        # run the guest additions installer
        ```

    * On the control node run the following setup:
        ```shell
        useradd ansible
        echo password | passwd --stdin ansible
        echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
        dnf install ansible-core -y
        ssh-keygen # select all defaults
        ```

    * Clone the control node machine five times for managed nodes 1-5. Attach a 2GB SATA drive to nodes 1-3, and a 1GB SATA drive to node 4. Note that cloning means the above steps are also done on each managed node, but if the servers existed already, we would have had to do this manually.

    * On each managed node get the IP address using `ifconfig`. On the control node, switch to the toot user and add hostnames for the managed nodes:
        ```shell
        vi /etc/hosts
        # add the following lines
        # 192.168.1.116 node1.example.com
        # 192.168.1.117 node2.example.com
        # 192.168.1.109 node3.example.com
        # 192.168.1.118 node4.example.com
        # 192.168.1.111 node5.example.com
        ```

    * On the control node run the following to make vim friendlier for playbook creation:
        ```shell
        echo "autocmd FileType yml setlocal ai ts=2 sw=2 et cuc nu" >> ~/.vimrc
        ```

1. Task 1

    * Run the following commands on the control node:
         ```shell
        vi /etc/ansible/ansible/hosts # add the below
        # [dev]
        # node1.example.com
        # 
        # [test]
        # node2.example.com
        # 
        # [proxy]
        # node3.example.com
        # 
        # [prod]
        # node4.example.com
        # node5.example.com
        # 
        # [webservers:children]
        # prod

        vi /etc/ansible/ansible/ansible.cfg
        # [defaults]
        # roles_path=/home/ansible/ansible/roles
        # inventory=/home/ansible/ansible/hosts
        # remote_user=ansible
        # host_key_checking=false

        # [privilege_escalation]
        # become=true
        # become_user=root
        # become_method=sudo
        # become_ask_pass=false
        ```

1. Task 2

    * To list the available modules run `ansible-doc -l`. To get help on a module run `ansible-doc -t module $module`.

    * Create and run the following script on the control node:
        ```shell
        #!/bin/bash
        ansible all -m yum_repository -a "name=EPEL description=RHEL9 baseurl=https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm gpgcheck=no enabled=no"
        ```

1. Task 3

    * Create and run the following playbook on the control node. The playbook can be tested using `ansible-playbook packages.yml -C`.
        ```yaml
        ---
        - name: Install packages
          hosts: dev,prod,webservers
          become: true
          tasks:
            - name: Install common packages
              yum:
                name:
                  - httpd
                  - mod_ssl
                  - mariadb
                state: present

            - name: Install dev specific packages
              yum:
                name:
                  - '@Development tools'
                state: latest
              when: "'dev' in group_names"

            - name: Update all packages on dev to the latest version
              yum:
                name:
                  - '*'
                state: latest
              when: "'dev' in group_names"
        ```

1. Task 4

    * Create the following file `main.yml` at `/home/ansible/ansible/roles/sample-apache/tasks`:
        ```yaml
        - name: Start and enable httpd
          service:
            name: httpd
            state: started
            enabled: yes

        - name: Start and enable firewalld
          service:
            name: firewalld
            state: started
            enabled: yes

        - name: Allow http service
          firewalld:
            service: http
            state: enabled
            permanent: true
            immediate: true

        - name: Create and serve message
          template:
            src: /home/ansible/ansible/roles/sample-apache/templates/index.html.j2
            dest: /var/www/html/index.html
          notify:
            - restart
        ```

    * Create the following file at `/home/ansible/ansible/roles/sample-apache/templates/index.html.j2`:
        ```shell
        Welcome to {{  ansible_fqdn }} on {{  ansible_default_ipv4.address }}
        ```

    * If you forget the ansible facts variables you can run `ansible localhost -m setup`. Note that the `firewalld` module requires you to run `ansible-galaxy collection install ansible.posix`.

    * Create the following file `main.yml` at `/home/ansible/ansible/roles/sample-apache/handlers`:
        ```yaml
        - name: restart
          service:
            name: httpd
            state: restarted
        ```

1. Task 5

    * Create a file `requirements.yml` at `/home/ansible/ansible/roles/requirements.yml`:
        ```yaml
        - name: haproxy-role
          src: geerlingguy.haproxy

        - name: php_role
          src: geerlingguy.php
        ```

    * Install the roles using `ansible-galaxy install -r requirements.yml -p /home/ansible/ansible/roles`. Observe the new roles available in the directory.

1. Task 6

    * Update the `/etc/hosts` file on managed node 3 to define the FQDNs used in the below playbook.

    * Create the playbook `/home/ansible/ansible/role.yml`:
        ```yaml
        ---
        - name: Install haproxy-role
          hosts: proxy
          become: true
          vars:
            haproxy_frontend_port: 81
            haproxy_backend_servers:
              - name: node4.example.com
                address: 192.168.1.118:80
              - name: node5.example.com
                address: 192.168.1.111:80

          tasks:
            - name: Install haproxy-role prereqs
              yum:
                name:
                  - haproxy
                state: present

            - name: Open port 81
              firewalld:
                port: 81/tcp
                permanent: true
                state: enabled
                immediate: true
              notify: Restart httpd service

            - name: Install haproxy-role
              include_role:
                name: haproxy-role

          handlers:
            - name: Restart httpd service
              service:
                name: httpd
                state: restarted

        - name: Install php_role
          hosts: prod
          become: true
          tasks:
            - name: Install php_role prereqs
              yum:
                name:
                  - php
                  - php-xml.x86_64
                state: present

            - name: Install php_role
              include_role:
                name: php_role

          handlers:
            - name: Restart httpd service
              service:
                name:
                  - httpd
                  - firewalld
                state: restarted
        ```

    * Installation currently fails due to a missing php-xmlrpc package. This appears to be an incompatibility with RHEL 9. Attempting to install this package from the EPEL repository gives an error about failing to download metadata.

1. Task 7

    * Create a file `/home/ansible/ansible/secrets.txt`:
        ```shell
        echo reallysafepw > secret.txt
        ```

    * Create a file `lock.yml` at `/home/ansible/ansible/lock.yml` using `ansible-vault encrypt lock.yml --vault-password-file secret.txt`. The contents of the original `lock.yml`:
        ```yaml
        pw_dev: dev
        pw_mgr: mgr
        ```

    * You can also directly create the encrypted playbook using `ansible-vault create lock.yml` and entering the password when prompted.

1. Task 8

    * Create a file `/home/ansible/ansible/users_list.yml`:
        ```yaml
        users:
          - username: bill
            job: developer
          - username: chris
            job: manager
          - username: dave
            job: test
          - username: ethan
            job: developer
        ```

    * Create a file `/home/ansible/ansible/users.yml` and run it with `ansible-playbook users.yml --vault-password-file secret.txt`:
        ```yaml
        ---
        - name: Create users
          hosts: all
          become: true
          vars_files:
            - lock.yml
            - users_list.yml

          tasks:
            - name: Create devops group
              group:
                name: devops
              when: "'dev' in group_names"

            - name: Create managers group
              group:
                name: managers
              when: "('proxy' in group_names)"

            - name: Create developer users
              user:
                name: "{{  item.username }}"
                group: devops
                password: "{{  pw_dev | password_hash('sha512')  }}"
              when: "('dev' in group_names) and ('developer' in item.job)"
              loop: "{{  users  }}"

            - name: Create manager users
              user:
                name: "{{  item.username  }}"
                group: managers
                password: "{{  pw_mgr | password_hash('sha512')  }}"
              when: "('proxy' in group_names) and ('manager' in item.job)"
              loop: "{{  users  }}"
        ```

1. Task 9

    * Create a file `/home/ansible/ansible/report.txt`:
        ```yaml
        HOST=inventory hostname
        MEMORY=total memory in mb
        BIOS=bios version
        SDA_DISK_SIZE=disk size
        SDB_DISK_SIZE=disk size
        ```

    * Create a file `/home/ansible/ansible/report.yml`:
        ```yaml
        ---
        - name: Create a report
          hosts: all
          tasks:
            - name: Copy the report template
              copy:
                src: /home/ansible/ansible/report.txt
                dest: /root/report.txt

            - name: Populate the HOST varible
              lineinfile:
                path: /root/report.txt
                state: present
                regex: "^HOST="
                line: "HOST='{{ ansible_facts.hostname  }}'"

            - name: Populate the MEMORY varible
              lineinfile:
                path: /root/report.txt
                state: present
                regex: "^MEMORY="
                line: "MEMORY='{{ ansible_facts.memtotal_mb  }}'"

            - name: Populate the BIOS varible
              lineinfile:
                path: /root/report.txt
                state: present
                regex: "^BIOS="
                line: "BIOS='{{ ansible_bios_version  }}'"

            - name: Populate the SDA_DISK_SIZE varible
              lineinfile:
                path: /root/report.txt
                state: present
                regex: "^SDA_DISK_SIZE="
                line: "SDA_DISK_SIZE='{{  ansible_devices.sda.size  }}'"
              when: 'ansible_devices.sda.size is defined'

            - name: Populate the SDA_DISK_SIZE varible
              lineinfile:
                path: /root/report.txt
                state: present
                regex: "^SDA_DISK_SIZE=NONE"
                line: "SDA_DISK_SIZE='{{  ansible_devices.sda.size  }}'"
              when: 'ansible_devices.sda.size is not defined'

            - name: Populate the SDB_DISK_SIZE varible
              lineinfile:
                path: /root/report.txt
                state: present
                regex: "^SDB_DISK_SIZE="
                line: "SDB_DISK_SIZE='{{  ansible_devices.sdb.size  }}'"
              when: 'ansible_devices.sdb.size is defined'

            - name: Populate the SDB_DISK_SIZE varible
              lineinfile:
                path: /root/report.txt
                state: present
                regex: "^SDB_DISK_SIZE="
                line: "SDB_DISK_SIZE=NONE"
              when: 'ansible_devices.sdb.size is not defined'
        ```

1. Task 10

    * The `hostvars` variable contains information about hosts in the inventory. You can run it to get an idea of the variables available when creating a j2 template.

    * Create a file `/home/ansible/ansible/hosts.j2`:
        ```jinja2
        {%for host in groups['all']%}
        {{hostvars[host]['ansible_default_ipv4']['address']}} {{hostvars[host]['ansible_fqdn']  {{hostvars[host]['ansible_hostname']}}
        {%endfor%}
        ```

    * Create a playbook `/home/ansible/ansible/hosts.yml` and run it with `ansible-playbook hosts.yml`:
        ```yml
        ---
        - name: Populate j2 template
          hosts: all
          tasks:
            - name: Populate j2 template
              template:
                src: /home/ansible/ansible/hosts.j2
                dest: /root/myhosts
              when: "'dev' in group_names"
        ```

1. Task 11

    * The tasks below are available in the community.general collection. Run `ansible-galaxy collection install community.general -p /home/ansible/ansible/collections` to install the collection.

    * Create a playbook `/home/ansible/ansible/logvol.yml` and run it with `ansible-playbook logvol.yml`:
        ```yml
        ---
        - name: Create logical volumes
          hosts: all
          become: true
          tasks:
            - name: Create partition
              community.general.parted:
                device: /dev/sdb
                state: present
                flags: [ lvm ]
                number: 1
              when: "ansible_devices.sdb is defined"

            - name: Create LVG
              community.general.lvg:
                vg: vg0
                pvs: /dev/sdb1
                state: present
              when: "ansible_devices.sdb.partitions.sdb1 is defined"

            - name: Create 1500MB LVOL
              community.general.lvol:
                vg: vg0
                lv: lv0
                size: 1500m
                state: present
              when: "ansible_lvm.vgs.vg0 is defined and ((ansible_lvm.vgs.vg0.size_g | float) > 1.5) and ansible_lvm.lvs.lv0 is not defined"

            - name: Print error message
              debug:
                msg: "Not enough space for logical volume."
              when: "ansible_lvm.vgs.vg0 is defined and ((ansible_lvm.vgs.vg0.size_g | float) < 1.5)"

            - name: Create 800MB LVOL
              community.general.lvol:
                vg: vg0
                lv: lv0
                size: 800m
                state: present
              when: "ansible_lvm.vgs.vg0 is defined and ((ansible_lvm.vgs.vg0.size_g | float) > 0.8) and ansible_lvm.lvs.lv0 is not defined"

            - name: Create the filesystem
              community.general.filesystem:
                fstype: xfs
                dev: /dev/vg0/lv0
                state: present
              when: "ansible_lvm.vgs.vg0 is defined"
        ```

1. Task 12

    * Create a file at `/home/ansible/ansible/index.html`:
        ```html
        Development
        ```

   * Create a playbook `/home/ansible/ansible/index.html` and run it with `ansible-playbook logvol.yml`:
        ```yaml
        ---
        - name: Serve a file
          hosts: dev
          become: true
          tasks:
            - name: Create the webdev user
              user:
                name: webdev
                state: present

            - name: Create webdev directory
              file:
                path: /webdev
                state: directory
                mode: '2755'
                owner: webdev

            - name: Create the html webdev directory
              file:
                path: /var/www/html/webdev
                state: directory
                mode: '2755'
                owner: root

            - name: Set correct SELinux file context
              sefcontext:
                target: '/var/www/html/webdev(/.*)?'
                setype: httpd_sys_content_t
                state: present

            - name: Set correct SELinux file context
              sefcontext:
                target: '/webdev(/.*)?'
                setype: httpd_sys_content_t
                state: present

            - name: Check firewall rules
              firewalld:
                service: http
                permanent: true
                state: enabled
                immediate: true

            - name: Copy html file into webdev directory
              copy:
                src: /home/ansible/ansible/index.html
                dest: /webdev/index.html
                owner: webdev
              notify: Restart httpd

	        - name: Create the symlink
              file:
                src: /webdev
                dest: /var/www/html/webdev
                state: link
                force: yes

          handlers:
            - name: Restart httpd
              service:
                name: httpd
                state: restarted
                enabled: yes
        ```

    * Note that the for the above to work on the `dev` host I had to set `DocumentRoot "/var/www/html/webdev` in `/etc/httpd/conf/httpd.conf`. There is likely a better solution than that.

1. Task 13

    * The system roles can be installed using:
        ```shell
        dnf install rhel-system-roles -y
        ```

    * Once installed, documentation and sample playbooks are available at `/usr/share/doc/rhel-system-roles/` under each role.

    * Create a playbook `/home/ansible/ansible/timesync.yml` and run it with `ansible-playbook timesync.yml`:
        ```yml
        ---  
        - name: Set the time using timesync
          hosts: all
          vars:
            timesync_ntp_servers:
              - hostname: 0.uk.pool.ntp.org
                boost: true
          roles:
            - /usr/share/ansible/roles/rhel-system-roles.timesync
        ```

1. Task 14

    * Create a playbook `/home/ansible/ansible/regulartasks.yml` and run it with `ansible-playbook regulartasks.yml`:
        ```yml
        ---      
        - name: Append the date
          hosts: all
          become: true
          tasks: 
            - name: Append the date using cron
              cron:
                name: datejob
                hour: "12"
                user: root
                job: "date >> /root/datefile"
        ```

1. Task 15

    * Create a playbook `/home/ansible/ansible/issue.yml` and run it with `ansible-playbook issue.yml`:
        ```yml
        ---    
        - name: Update a file conditionally
          hosts: all
          tasks:
            - name: Update file for dev
              copy:
                content: "Development"
                dest: /etc/issue
              when: "'dev' in group_names"

            - name: Update file for tets
              copy:
                content: "Test"
                dest: /etc/issue
              when: "'test' in group_names"

            - name: Update file for prod
              copy:
                content: "Production"
                dest: /etc/issue
              when: "'prod' in group_names"
        ```

1. Task 16

    * Create an encrypted playbook using `ansible-vault create myvault.yml`:
        ```shell
        ansible-vault create myvault.yml # enter pw as notsafepw
        ansible-vault rekey myvault.yml # enter old and new pw as requested
        ```

1. Task 17

    * Create a playbook `ansible-vault create target.yml`:
        ```yaml
        ---      
        - name: Change default target
          hosts: all
          tasks: 
            - name: Change default target
              file:
                src: /usr/lib/systemd/system/multi-user.target
                dest: /etc/systemd/system/default.target
                state: link
        ```

    * This is not a good solution as it requires implementation level knowledge.

#### Practise Exam 2

1. Setup

    * Create a VirtualBox machine for the control node. In VirtualBox create a new machine with the VirtualBox additions and the RHEL 9.2 installer ISOs attached as IDE devices. Set networking mode to Bridged Adapter. Set the Pointing Device to USB Tablet and enable the USB 3.0 Controller. Set shared Clipboard and Drag'n'Drop to Bidirectional. Uncheck Audio output. Start the machine and install RHEL.

    * On the control node run the following setup as root:
        ```shell
        blkid # note the UUID of the RHEL ISO
        mkdir /mnt/rheliso
        vi /etc/fstab
        echo "UUID="" /mnt/rheliso iso9660 loop 0 0" >> /etc/fstab
        mount -a # confirm no error is returned
        ```

    * On the control node setup the dnf repositories as root:
        ```shell
        vi /etc/yum.repos.d/redhat.repo # add the below content
        # [BaseOS]
        # name=BaseOS
        # baseUrl=file:///mnt/rheliso/BaseOS
        # enabled=1
        # gpgcheck=0
        #
        # [AppStream]
        # name=AppStream
        # enabled=1
        # baseurl=file:///mnt/rheliso/AppStream
        # gpgcheck=0
        dnf repolist # confirm repos are returned
        ```

    * On the control node run the following setup as root to install guest additions:
        ```shell
        dnf install kernel-headers kernel-devel
        # run the guest additions installer
        ```

    * On the control node run the following setup:
        ```shell
        useradd ansible
        echo password | passwd --stdin ansible
        echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
        dnf install ansible-core rhel-system-roles -y
		```

    * On the control node run the following to make vim friendlier for playbook creation:
        ```shell
        echo "autocmd FileType yml setlocal ai ts=2 sw=2 et cuc nu" >> ~/.vimrc
        ```

    * Clone the control node machine five times for managed nodes 1-5. Attach a 2GB SATA drive to nodes 1-3, and a 1GB SATA drive to node 4. Note that cloning means the above steps are also done on each managed node, but if the servers existed already, we would have had to do this manually.

    * On each managed node get the IP address using `ifconfig`. On the control node, switch to the root user and add hostnames for the managed nodes:
        ```shell
        vi /etc/hosts
        # add the following lines
        # 192.168.1.116 node1.example.com
        # 192.168.1.117 node2.example.com
        # 192.168.1.109 node3.example.com
        # 192.168.1.118 node4.example.com
        # 192.168.1.111 node5.example.com
        ```

    * On the control node run the following:
       ```shell
        ssh-keygen # select defaults
        ssh-copy-id ansible@node1.example.com # repeat for the remaining managed nodes
        ansible all -m ping # confirm ssh connectivity
        '''

1. Task 1

    * Run the following commands on the control node:
         ```shell
        vi /etc/ansible/ansible/hosts # add the below
        # [dev]
        # node1.example.com
        # 
        # [test]
        # node2.example.com
        # 
        # [proxy]
        # node3.example.com
        # 
        # [prod]
        # node4.example.com
        # node5.example.com
        # 
        # [webservers:children]
        # prod

        vi /etc/ansible/ansible/ansible.cfg
        # [defaults]
        # roles_path=/home/ansible/ansible/roles
        # inventory=/home/ansible/ansible/hosts
        # remote_user=ansible
        # host_key_checking=false

        # [privilege_escalation]
        # become=true
        # become_user=root
        # become_method=sudo
        # become_ask_pass=false
        ```

1. Task 2

    * Create and run the following `adhoc.sh` script on the control node:
         ```shell
        #!/bin/bash
        # create the 'devops' user with necessary permissions
        ansible all -u ansible -m user -a "name=devops" --ask-pass
        ansible all -u ansible -m shell -a "echo password | passwd --stdin devops"
        ansible all -u ansible -m shell -a "echo 'devops ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/devops"

        # allow SSH without password
        ssh-copy-id -i /home/devops/.ssh/id_rsa.pub node1.example.com
        ssh-copy-id -i /home/devops/.ssh/id_rsa.pub node2.example.com
        ssh-copy-id -i /home/devops/.ssh/id_rsa.pub node3.example.com
        ssh-copy-id -i /home/devops/.ssh/id_rsa.pub node4.example.com
        ssh-copy-id -i /home/devops/.ssh/id_rsa.pub node5.example.com
        ```

    * Note that this requires an existing user `ansible` to authenticate to the managed nodes. The ask pass parameter is used initially as the devops user is not created in the remote systems, and there are no keys setup for the ansible user yet.

1. Task 3

    * Create and run the following playbook `/home/ansible/ansible/motd.yml` script on the control node:
         ```yaml
         ---  
        - name: Write to /etc/motd
          hosts: all
          tasks:
            - name: Write to /etc/motd for dev
              copy:
                content: 'Welcome to Dev Server {{  ansible_fqdn  }}'
                dest: /etc/motd
              when: "'dev' in group_names"

            - name: Write to /etc/motd for webservers
              copy:
                content: 'Welcome to Apache Server {{  ansible_fqdn  }}'
                dest: /etc/motd
              when: "'webservers' in group_names"

            - name: Write to /etc/motd for test
              copy:
                content: 'Welcome to MySQL Server {{  ansible_fqdn  }}'
                dest: /etc/motd
              when: "'test' in group_names"
        ```

1. Task 4

    * Create and run the following playbook `/home/ansible/ansible/sshd.yml` script on the control node:
         ```yaml
        ---
        - name: Configure SSHD daemon
          hosts: all
          become: true
          tasks: 
            - name: Set Banner
              lineinfile:
                path: /etc/ssh/sshd_config.d/50-redhat.conf
                regexp: '^Banner '
                line: Banner /etc/issue
                state: present

            - name: Set PermitRootLogin
              lineinfile:
                path: /etc/ssh/sshd_config.d/50-redhat.conf
                regexp: '^PermitRootLogin '
                line: PermitRootLogin no
                state: present

            - name: Set MaxAuthTries
              lineinfile:
                path: /etc/ssh/sshd_config.d/50-redhat.conf
                regexp: '^MaxAuthTries '
                line: MaxAuthTries 6
                state: present

            - name: Restart SSHD
              service:
                name: sshd
                state: restarted
        ```

1. Task 5

    * Run the following commands on the control node:
         ```shell
        ansible-vault create secret.yml # set password as admin123
        # set file contents as below
        # user_pass: user        
        # database_pass: database
        echo admin123 > secret.txt
        ansible-vault view secret.yml --vault-pass-file secret.txt
        ```

1. Task 6

    * Create `/vars/userlist.yml`:
         ```yaml
        ---      
        users:          
        - username: alice  
          job: developer
        - username: vincent
          job: manager  
        - username: sandy  
          job: tester   
        - username: patrick
          job: developer
        ```

    * Create and run the playbook `/home/ansible/ansible/users.yml`:
         ```yaml
        ---      
        - name: Create users
          hosts: all
          vars_files:
            - userlist.yml
            - secret.yml
          tasks: 
            - name: Create developer users from the userlist
              user:
                name: "{{  item.username }}"
                group: wheel
                password: "{{  user_pass | password_hash('sha512')  }}"
              with_items: "{{  users }}"
              when: "('dev' in group_names) and (item.job == 'developer')"

            - name: Create manager users from the userlist
              user:
                name: "{{  item.username }}"
                group: wheel
                password: "{{  database_pass | password_hash('sha512')  }}"
              with_items: "{{  users }}"
              when: "('test' in group_names) and (item.job == 'manager')"
        ```

1. Task 7

    * Create and run the script `/home/ansible/ansible/repository.sh`:
         ```shell
        #!/bin/bash
        ansible test -m yum_repository -a "name=mysql56-community description='MySQL 5.6 YUM Repo' baseurl=http://repo.example.com/rpms enabled=0"
        ```

1. Task 8

    * Update networking to NAT on the control node to install additional collections. Create and run the script `/home/ansible/ansible/repository.sh`:
         ```shell
        ansible-galaxy collection install ansible.posix
        ansible-galaxy collection install community.general
        ```

    * Revert the networking to Bridged Adapter. Add an additional 1GB and 2GB drive to nodes 4 and 5 respectively. Create and run the playbook `/home/ansible/ansible/logvol.yml`:
	     ```yaml
	    ---    
	    - name: Setup volumes
	      hosts: all
	      become: true
	      tasks:
	        - name: Create partition
	          parted:
	            device: /dev/sdb
	            number: 1
	            flags: [ lvm ]
	            state: present
	          when: "ansible_facts['devices']['sdb'] is defined"
	
	        - name: Create volume group
	          lvg:
	            vg: vg0
	            pvs: /dev/sdb1
	          when: "ansible_facts['devices']['sdb'] is defined"
	
	        - name: Create logical volume
	          lvol:
	            vg: vg0
	            lv: lv0
	            size: 1500m
	            state: present
	          when: "ansible_facts['devices']['sdb'] is defined and ansible_lvm['vgs']['vg0'] is defined and ansible_lvm['lvs']['lv0'] is not defined and (ansible_lvm['vgs']['vg0']['free_g'] | float) > 1.5"
	
	        - name: Print error message
	          debug:
	            msg: "Not enough space in volume group"
	          when: "ansible_facts['devices']['sdb'] is defined and ansible_lvm['vgs']['vg0'] is defined and (ansible_lvm['vgs']['vg0']['free_g']) | float <= 1.5"
	
	        - name: Create logical volume
	          lvol:
	            vg: vg0
	            lv: lv0
	            size: 800m
	            state: present
	          when: "ansible_facts['devices']['sdb'] is defined and ansible_lvm['vgs']['vg0'] is defined and ansible_lvm['lvs']['lv0'] is not defined and (ansible_lvm['vgs']['vg0']['free_g']) | float > 0.8"
	
	        - name: Create the file system
	          filesystem:
	            dev: /dev/vg0/lv0
	            state: present
	            fstype: xfs
	          when: "ansible_lvm['lvs']['lv0'] is defined"
	
	        - name: Mount file system
	          mount:
	            src: /dev/vg0/lv0
	            path: /mnt/data
	            state: mounted
	            fstype: xfs
	          when: "ansible_lvm['lvs']['lv0'] is defined"
	    ```

    * Note that for some reason you can't access the lvm properties using `ansible_facts['ansible_lvm']`.

1. Task 9

    * Create and run the playbook `/home/ansible/ansible/regular_tasks.yml`:
         ```yaml
        ---      
        - name: Create cron job on dev
          hosts: dev
          become: true
          tasks: 
            - name: Create cron job on dev
              cron:
                name: hourly job
                minute: "0"
                job: "date >> /root/time"
        ```

1. Task 10

    * Create the role `/home/ansible/ansible/roles/sample-apache/tasks/main.yml`:
         ```yaml
        - name: Install httpd
          yum:
            name: httpd
            state: present

        - name: Enable http in firewalld
          firewalld:
            service: http
            state: enabled
            permanent: true
            immediate: true

        - name: Start httpd
          service:
            name: httpd
            state: started
            enabled: yes

        - name: Copy template
          template:
            src: /home/ansible/ansible/index.html.j2
            dest: /var/www/html/index.html
          notify:
            - Restart httpd
        ```

    * Create the handler `/home/ansible/ansible/index.html.j2`:
         ```shell
        Welcome to the server {{  ansible_hostname  }}
        ```

    * Create and run the playbook  
         ```yaml
        ---  
        - name: Setup Apache
          hosts: all
          become: true
          roles:
            - sample-apache
        ```

1. Task 11

    * Create the file `/home/ansible/ansible/requirements.yml`:
         ```yaml
        - name: haproxy-role
           src: geerlingguy.haproxy
        ```

    * Install the role using `ansible-galaxy install -r requirements.yml -p roles`.

    * The documentation in `/home/ansible/ansible/roles/haproxy-role` can be referred. Create and run the playbook `/home/ansible/ansible/haproxy.yml`:
         ```yaml
        ---    
        - name: Configure load balancing
          hosts: proxy
          become: true
          vars:
            haproxy_frontend_port: 80
            haproxy_backend_servers:
              - name: node4.example.com
                address: 192.168.1.111:80
              - name: node5.example.com
                address: 192.168.1.112:80
          pre_tasks:
            - name: Install pre-requisites
              yum:
                name:
                  - haproxy
                state: present
          roles:
            - haproxy-role
        ```

    * In this instance the hostnames for node4 and node5 were added to the `/etc/hosts` file in node3. The httpd service was running on node3 which initially prevented playbook completion. This was identified using `sudo ss -6 -tlnp | grep 80` and resolved using `sudo systemctl stop httpd`.

1. Task 12

    * Create and run the playbook `/home/ansible/ansible/timesync.yml`:
         ```yaml
        ---
        - name: Install timesync
          hosts: all
          vars: 
            timesync_ntp_servers:
              - hostname: 0.uk.pool.ntp.org
          tasks:
            - name: Set timezone to UTC
              timezone:
                name: UTC
          roles:
            - /usr/share/ansible/roles/rhel-system-roles.timesync
        ```

1. Task 13

	* Create a file `/home/ansible/ansible/hosts.j2`:
         ```jinja2
		{%for host in groups['all']%}
		{{  hostvars[host]['ansible_default_ipv4']['address']  }} {{  hostvars[host]['ansible_fqdn']  }} {{  hostvars[host]['ansible_hostname']  }}
		{%endfor%}
        ```

    * Create and run the playbook `/home/ansible/ansible/hosts.yml`:
         ```yaml
		---
		- name: Create file based on template
		  hosts: all
		  become: true
		  tasks:
		    - name: Create file based on template
		      template:
		        src: /home/ansible/ansible/hosts.j2
		        dest: /root/myhosts
		      when: "'dev' in group_names"
        ```

1. Task 14

	* Create a file `/home/ansible/ansible/requirements.yml`:
         ```yaml
		---                     
		- name: sample-php_roles
		  src: geerlingguy.php
        ```

	* Run `ansible-galaxy install -r requirements.yml -p roles`.

1. Task 15

	* Create a file `/home/ansible/ansible/specs.empty`:
         ```yaml           
		HOST=
		MEMORY=
		BIOS=
		SDA_DISK_SIZE=
		SDB_DISK_SIZE=
        ```

	* Create and run the playbook `/home/ansible/ansible/specs.yml`:
         ```yaml
		---    
		- name: Populate specs file
		  hosts: all
		  become: true
		  tasks:
		    - name: Copy file to hosts
		      copy:
		        src: /home/ansible/ansible/specs.empty
		        dest: /root/specs.txt
		    - name: Update hosts
		      lineinfile:
		        path: /root/specs.txt
		        regexp: '^HOST='
		        line: 'HOST={{  ansible_hostname  }}'
		    - name: Update memory
		      lineinfile:
		        path: /root/specs.txt
		        regexp: '^MEMORY='
		        line: 'MEMORY={{  ansible_memtotal_mb  }}'
		    - name: Update BIOS
		      lineinfile:
		        path: /root/specs.txt
		        regexp: '^BIOS='
		        line: 'BIOS={{  ansible_bios_version  }}'
		    - name: Update SDA disk size
		      lineinfile:
		        path: /root/specs.txt
		        regexp: '^SDA_DISK_SIZE='
		        line: "SDA_DISK_SIZE={{  ansible_devices['sda']['size']  }}"
		      when: "ansible_devices['sda'] is defined"
		    - name: Update SDB disk size
		      lineinfile:
		        path: /root/specs.txt
		        regexp: '^SDB_DISK_SIZE='
		        line: "SDB_DISK_SIZE={{  ansible_devices['sdb']['size']  }}"
		      when: "ansible_devices['sdb'] is defined"
        ```

1. Task 16

	* Create and run the playbook `/home/ansible/ansible/packages.yml`:
         ```yaml
		---    
		- name: Install packages
		  hosts: all
		  become: true
		  tasks:
		    - name: Install packages using yum for proxy
		      yum:
		        name:
		          - httpd
		          - mod_ssl
		        state: present
		      when: "'proxy' in group_names"
		    - name: Install packages using yum for dev
		      yum:
		        name:
		          - '@Development tools'
		        state: present
		      when: "'dev' in group_names"
        ```

1. Task 17

	* Create and run the playbook `/home/ansible/ansible/mysecret.yml`:
         ```shell
		ansible-vault create mysecret.yml # enter notasafepass
		# add a line dev_pass: devops
		ansible-vault rekey mysecret.yml # enter new password devops123
		ansible-vault edit mysecret.yml
		# add a line dev_pass: devops123
        ```

#### Practise Exam 3

1. Install Essential tools and ensure access to remote hosts

	* Run the following:
         ```shell
		# on the control node
		sudo -su root
		ansible localhost -m user -a "name=ansible"
		echo password | passwd --stdin ansible
		echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
		mdkir ansible
		cd ansible
		ansible-config init --disabled > ansible_reference.cfg
		sudo cp /etc/ansible/hosts hosts.ini
		sudo chown ansible:ansible hosts.ini
		ansible-galaxy collection install community.general community.crypto ansible.posix
		python3 -m pip install ansible-navigator --user
        ```

1. Install Essential tools and ensure access to remote hosts

	* Run the following:
         ```shell
		# on the control node
		sudo -su root
		mkdir /mnt/rheliso
		blkid # note UUID
		echo "UUID='' /mnt/rheliso iso9660 loop 0 0" >> /etc/fstab
		mount -a
		vi /etc/yum.repos.d/redhat.repo
		# add the BaseOS and AppStream repos
		dnf install kernel-headers kernel-devel -y
		# install guest additions
		useradd ansible
		echo password | passwd --stdin ansible
		echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
		sudo -su ansible
		sudo dnf install ansible-core rhel-system-roles
		sudo dnf install pip -y
		python3 -m pip install ansible-navigator --user
		echo "autocmd FileType yml setlocal ai sw=2 ts=2 et cu" >> ~/.vimrc
		mdkir ~/ansible
		cd ~/ansible
		ansible-config init --disabled > ansible_reference.cfg
		sudo cp /etc/ansible/hosts hosts.ini
		sudo chown ansible:ansible hosts.ini
		ansible-galaxy collection install community.general community.crypto ansible.posix
        ```

	* Add the managed nodes in `/etc/fstab`.

	* Update `ansible.cfg` in the working directory:
         ```shell
		[defaults]
		inventory=/home/ansible/ansible/hosts.ini
		host_key_checking=false
		ask_pass=true
		remote_user=ansible

		[privilege_escalation]
		become=false
		become_method=sudo
		become_ask_pass=true
		become_user=root
        ```

	* Login to each managed node and create the ansible user. Create and run the `bootstrap.yml` playbook on the control node:
         ```yml
		---
		- name: Bootstrap automation user
		  hosts: all
		  become: true
		  tasks:
		    - name: Check automation user
		      user:
		        name: ansible
		        password: "{{  'password' | password_hash('sha512')  }}"
		        home: /home/ansible
		        state: present

		    - name: Check key folder
		      file:
		        state: directory
		        mode: 0700
		        name: /home/ansible/.ssh

		    - name: Check sudo access
		      copy:
		        content: "ansible ALL=(ALL) NOPASSWD:ALL"
		        dest: /etc/sudoers.d/ansible
		        validate: /usr/sbin/visudo -csf %s

		    - name: Copy public key into authorized_keys
		      ansible.posix.authorized_key:
		        user: ansible
		        state: present
		        key: "{{  lookup('file', lookup('env', 'HOME') + '/.ssh/id_rsa.pub')  }}"
        ```

1. Create an Ansible Inventory

	* Add the following to `/home/ansible/inventory`:
         ```yml
		[web]
		web01
		web02

		[development]
		dev01 

		[dc1:children]
		web
		development
        ```

1. Configure Ansible Settings

	* Update `ansible.cfg` in the working directory:
         ```yml
		[defaults]
		inventory=/home/ansible/rhce1/inventory
		host_key_checking=False
		ask_pass=False
		remote_user=ansible
		forks=3
		timeout=120

		[privilege_escalation]
		become=True
		become_method=sudo
		become_ask_pass=False
		become_user=root
        ```

1. User Management and Group Assignments

	* Run the following:
         ```shell
		mkdir vars
		cd vars
		ansible-vault create vault.yml # enter 'rocky'
		# enter user_password: password!
		cd ..
		echo rocky > users_vault.txt
        ```

	* Create the playbook `users.yml` and run using `ansible-playbook users.yml --vault-password-file users_vault.txt`:
         ```yml
		---
		- name: Create users in managed nodes
		  hosts: dc1
		  vars_files:
		    - vars/vault.yml
		  tasks:
		    - name: Create the admins groups
		      group:
		        name: admins
		        state: present

		    - name: Create the users group
		      group:
		        name: users
		        state: present

		    - name: Create tony
		      user:
		        name: tony
		        password: "{{  user_password | password_hash('sha512')  }}"
		        groups: admins

		    - name: Create carmela
		      user:
		        name: carmela
		        password: "{{  user_password | password_hash('sha512')  }}"
		        groups: admins

		    - name: Create paulie
		      user:
		        name: paulie
		        password: "{{  user_password | password_hash('sha512')  }}"
		        groups: users

		    - name: Create chris
		      user:
		        name: chris
		        password: "{{  user_password | password_hash('sha512')  }}"
		        groups: users

		    - name: Give tony sudo access
		      copy:
		        content: "tony ALL=(ALL) NOPASSWD:ALL"
		        dest: /etc/sudoers.d/tony
        ```

1. Setup a Cron Job for Logging Date

	*  Create and run the playbook `cron.yml`
         ```yml
		---
		- name: Setup cron jobs on managed nodes
		  hosts: dev01
		  tasks:
		    - name: Setup cron job
		      cron:
		        name: "Write date to file"
		        minute: "*/2"
		        job: "date '+\\%Y-\\%m-\\%d \\%H:\\%M:\\%S' >> /tmp/logme.txt"
		        state: present
        ```

1. Extract Host Information from Inventory using Ansible Navigator

	*  Create and run the script `fetch_host_info.sh`
         ```shell
		#!/bin/bash
		ansible-navigator inventory -i inventory -m stdout --host web01
        ```

1. Configure Time Synchronization Using RHEL System Roles

	*  Run the following:
         ```shell
		sudo dnf install rhel-system-roles -y
        ```

	*  Create and run the playbook `timesync.yml`
         ```yml
		---
		- name: Configure timesync
		  hosts: all
		  vars:
		    timesync_ntp_servers:
		      - hostname: 2.rhel.pool.ntp.org
		        iburst: True
		        pool: True
		  roles:
		    - /usr/share/ansible/collections/ansible_collections/redhat/rhel_system_roles/roles/timesync
        ```

1. Software Installation Using Variables

	*  Create the file `software_vars.yml`
         ```yml
		---
		software_packages:
		  - vim
		  - nmap
        ```

	*  Create and run the playbook `software_install.yml`
         ```yml
		---
		- name: Install software packages
		  hosts: all
		  vars_files:
		    - software_vars.yml
		  vars:
		    software_group: "@Virtualization Host"
		  tasks:
		    - name: Install packages for web
		      yum:
		        name: "{{  item  }}"
		        state: present
		      with_items:
		        - "{{  software_packages  }}"
		      when: "'web' in group_names"

		    - name: Install packages for dev01
		      yum:
		        name: "{{  software_group  }}"
		        state: present
		      when: "'development' in group_names"
        ```

1. Debugging an API Key from an Ansible Vault

	*  Run the following:
         ```shell
		ansible-vault create api_key.yml # trustme!123
        ```

	* Add the following content to `api_key.yml`:
         ```yml
		my_api_key: "f3eb0782983d3a417de12b96eb551a90"
        ```

	*  Run the following:
         ```shell
		ansible-vault create api_key.yml # trustme!123
		echo 'trustme!123' > vault-key.txt
        ```

	* Create `playbook-secret.yml` and run using `ansible-playbook playbook-secret.yml --vault-password-file vault-key.txt:`
         ```yml
		---
		- name: Fetch API keys
		  hosts: localhost
		  vars_files:
		    - api_key.yml
		  tasks:
		    - name: Print API key
		      debug:
		        var: my_api_key
        ```

1. Configure SELinux Settings for dev01

	* Create and run `selinux.yml`:
         ```yml
		---
		- name: Configure SELinux
		  hosts: dev01
		  vars:
		    selinux_policy: targeted
		    selinux_state: enforcing
		    selinux_ports:
		      - {ports: '82', proto: 'tcp', setype: 'http_port_t', state: 'present', local: true}
		  roles:
		    - /usr/share/ansible/collections/ansible_collections/redhat/rhel_system_roles/roles/selinux
		  tasks:
		    - name: Install http package
		      yum:
		        name: httpd
		        state: present

		    - name: Enable http service
		      service:
		        name: httpd
		        state: started
		        enabled: yes

		    - name: Enable firewall port
		      ansible.posix.firewalld:
		        port: 82/tcp
		        state: enabled
		        permanent: yes
		        immediate: yes

		    - name: Check Apache port
		      lineinfile:
		        path: /etc/httpd/conf/httpd.conf
		        regexp: '^Listen '
		        insertafter: '^#Listen '
		        line: Listen 82
        ```

1. Troubleshoot and Fix the Playbook

	* Create and run `fixme.yml`:
         ```yml
		---
		- name: Troublesome Playbook
		  hosts: all
		  become: yes
		  vars:
		    install_package: "vsftpd"
		    service_name: "vsftpd"

		  tasks:
		  - name: Install a package
		    yum:
		      name: "{{ install_package }}"
		      state: "installed"

		  - name: Start and enable a service
		    service:
		      name: "{{ service_name }}"
		      enabled: yes
		      state: started
        ```

1. Ad-Hoc Command Execution via Shell Script

	* Create and run `adhocfile.sh`:
         ```shell
		#!/bin/bash
		ansible all -m file -a "path=/tmp/sample.txt state=touch owner=carmela mode=0644"
		ansible all -m lineinfile -a "path=/tmp/sample.txt line='Hello ansible world'"
        ```

1. Create Swap Partition Based on Memory

	* Create and run `swap.yml`:
         ```yml
		---
		- name: Create Swap Partition Based on Memory
		  hosts: all
		  tasks:
		    - name: Create partition
		      community.general.parted:
		        device: /dev/sdb
		        number: 1
		        state: present
		      when: "ansible_devices['sdb'] is defined and ansible_memtotal_mb < 8000"

		    - name: Display message
		      debug:
		        msg: "Available memory is {{  ansible_memfree_mb  }}"
		      when: "ansible_memtotal_mb < 8000"

		    - name: Create file system
		      community.general.filesystem:
		        fstype: swap
		        dev: /dev/sdb1
		        state: present
		      when: "ansible_devices['sdb']['partitions']['sdb1'] is defined"

			- name: Activate the swap space
			  command: "swapon /dev/sdb1"
			  when: "ansible_devices['sdb']['partitions']['sdb1'] is defined"

		    - name: Mount file system
		      ansible.posix.mount:
		        path: /mnt/swp
		        src: /dev/sdb1
		        state: mounted
		        fstype: swap
		      when: "ansible_devices['sdb']['partitions']['sdb1'] is defined"
        ```

1. Check Webpage Status and Debug on Failure

	* Create and run `check_webpage.yml`:
         ```yml
	    ---
	    - name: Check Webpage Status and Debug on Failure
	      hosts: localhost
	      become: false
	      gather_facts: no
	      tasks:
	        - name: Block to attempt fetching the webpage status
	          block:
	            - name: Attempt to fetch the status of webserver
	              ansible.builtin.uri:
	                url: http://169.254.3.5
	                method: GET
	                status_code: 200
	              register: webpage_result

	          rescue:
	            - name: Display debug message if webpage check fails
	              ansible.builtin.debug:
	                msg: "{{ webpage_result }}"
        ```

1. Configure SSH Security Settings with Ansible Role

	* Create `/home/ansible/rhce1/roles/secure_ssh/tasks/main.yml`:
         ```yml
		---
		- name: Configure X11Forwarding
		  lineinfile:
		    path: /etc/ssh/sshd_config
		    line: X11Forwarding no
		  notify: Restart SSH

		- name: Configure PermitRootLogin
		  lineinfile:
		    path: /etc/ssh/sshd_config
		    line: PermitRootLogin no
		  notify: Restart SSH

		- name: Configure MaxAuthTries
		  lineinfile:
		    path: /etc/ssh/sshd_config
		    line: MaxAuthTries 3
		  notify: Restart SSH

		- name: Configure AllowTcpForwarding
		  lineinfile:
		    path: /etc/ssh/sshd_config
		    line: AllowTcpForwarding no
		  notify: Restart SSH
        ```

	* Create `/home/ansible/rhce1/roles/secure_ssh/handlers/main.yml`:
         ```yml
		---
		- name: Restart SSH
		  service:
		    name: sshd
		    state: restarted
        ```

	* Create and run `secure_ssh_playbookyml`:
         ```yml
		---
		- name: Secure SSH
		  hosts: all
		  roles:
		    - secure_ssh
        ```

1. Task: Configure Web Server with System Information

	* Create `webserver.j2`:
         ```yml
		Servername: {{  ansible_hostname }}
		IP Aaddress: {{ ansible_default_ipv4['address'] }}
		Free Memory: {{ ansible_memfree_mb  }}MB
		OS: {{  ansible_os_family  }}
		Kernel Version: {{  ansible_kernel  }}
        ```

	* Create and run `webserver.yml`:
         ```yml
		---
		- name: Configure Web Server with System Information
		  hosts: web01
		  tasks:
		    - name: Install httpd
		      yum:
		        name: httpd
		        state: present

		    - name: Enable httpd
		      service:
		        name: httpd
		        state: started
		        enabled: yes

		    - name: Enable firewall
		      ansible.posix.firewalld:
		        service: http
		        state: enabled
		        immediate: true
		        permanent: true
			  tags: setfirewall

		    - name: Populate template
		      template:
		        src: webserver.j2
		        dest: /var/www/html/index.html
        ```
