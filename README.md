1.Download TDExpress16.20 and unzip file
```
7z e TDExpress16.20_Sles11_20181108052529.7z
```
2.Convert vdmk to vdi format
```
VBoxManage clonehd TDExpress16.20_Sles11-disk1.vmdk  TDExpress16.20_Sles11-disk1.vdi -format VDI
VBoxManage clonehd TDExpress16.20_Sles11-disk2.vdmk  TDExpress16.20_Sles11-disk2.vdi -format VDI
VBoxManage clonehd TDExpress16.20_Sles11-disk3.vdmk  TDExpress16.20_Sles11-disk3.vdi -format VDI
```
3.Create td_express_16_20 VirtualBox VM
```
export VM_NAME="td_express_16_20"

vboxmanage createvm --name $VM_NAME --ostype OpenSUSE_64 --register
vboxmanage storagectl $VM_NAME --name "SCSI Controller" --add scsi --controller LsiLogic
vboxmanage storageattach $VM_NAME --storagectl "SCSI Controller" --port 0 --device 0 --type hdd --medium ./TDExpress16.20_Sles11-disk1.vdi
vboxmanage storageattach $VM_NAME --storagectl "SCSI Controller" --port 1 --device 0 --type hdd --medium ./TDExpress16.20_Sles11-disk2.vdi
vboxmanage storageattach $VM_NAME --storagectl "SCSI Controller" --port 2 --device 0 --type hdd --medium ./TDExpress16.20_Sles11-disk3.vdi
vboxmanage modifyvm $VM_NAME --ioapic on
vboxmanage modifyvm $VM_NAME --memory 4096 --vram 128
vboxmanage modifyvm $VM_NAME --nic1 nat
vboxmanage modifyvm $VM_NAME --natpf1 "guestssh,tcp,,2244,,22"
vboxmanage modifyvm $VM_NAME --natpf1 "guestteradata,tcp,,1025,,1025"
vboxmanage modifyvm $VM_NAME --natdnshostresolver1 on
```
4.start VM
```
vboxmanage startvm td_express_16_20 --type headless
```
5.Connect to VM and create atscale database
```
$ sudo netstat -antpl|grep 2244
tcp        0      0 0.0.0.0:2244            0.0.0.0:*               LISTEN      9781/VBoxHeadless   

$ ssh -p 2244 root@localhost

user:root
password:root

Start database if not started (rm /var/opt/teradata/tdtemp/PanicLoopDetected if needed : check /var/log/messages)

TDExpress1620_Sles11:~ # /etc/init.d/tpa start (helps for *** Warning: RDBMS CRASHED OR SESSIONS RESET.  RECOVERY IN PROGRESS.)

TDExpress1620_Sles11:~ # netstat -analpt|grep 1025
tcp        0      0 :::1025                 :::*                    LISTEN      25297/gtwgateway 

TDExpress1620_Sles11:~ # pdestate -a
PDE state is RUN/STARTED.
DBS state is 5: Logons are enabled - The system is quiescent

TDExpress1620_Sles11:~ # verify_pdisks
All pdisks on this node verified.

TDExpress1620_Sles11:~ # bteq

 Teradata BTEQ 16.20.00.04 for LINUX. PID: 6384
 Copyright 1984-2018, Teradata Corporation. ALL RIGHTS RESERVED.
 Enter your logon or BTEQ command:
.logon dbc

.logon dbc
Password: 


 *** Logon successfully completed.
 *** Teradata Database Release is 16.20.23.01                   
 *** Teradata Database Version is 16.20.23.01                     
 *** Transaction Semantics are BTET.
 *** Session Character Set Name is 'ASCII'.
 
 *** Total elapsed time was 1 second.
 
 BTEQ -- Enter your SQL request or BTEQ command: 
CREATE DATABASE atscale AS PERM=10000000000;

CREATE DATABASE DAVAR AS PERM=10000000000;

 *** Database has been created. 
 *** Total elapsed time was 1 second.


 BTEQ -- Enter your SQL request or BTEQ command: 

```

6.Vagrant box creation:

6.1.Add vagrant user and setup suduers
```
TDExpress1620_Sles11:~ # useradd -m -s /bin/bash vagrant -u 666 --groups wheel

TDExpress1620_Sles11:~ # grep vagrant /etc/sudoers 
vagrant ALL=(ALL) NOPASSWD: ALL
```
6.2.Add vagrant public key: https://github.com/hashicorp/vagrant/tree/master/keys

As vagrant user 
```
vagrant@TDExpress1620_Sles11:~> mkdir -p ~/.ssh/
vagrant@TDExpress1620_Sles11:~> touch ~/.ssh/authorized_keys
vagrant@TDExpress1620_Sles11:~> vi .ssh/authorized_keys 
vagrant@TDExpress1620_Sles11:~> cat .ssh/authorized_keys 
ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEA6NF8iallvQVp22WDkTkyrtvp9eWW6A8YVr+kz4TjGYe7gHzIw+niNltGEFHzD8+v1I2YJ6oXevct1YeS0o9HZyN1Q9qgCgzUFtdOKLv6IedplqoPkcmF0aYet2PkEDo3MlTBckFXPITAMzF8dJSIFo9D8HfdOV0IAdx4O7PtixWKn5y2hMNG0zQPyUecp4pzC6kivAIhyfHilFR61RGL+GPXQ2MWZWFYbAGjyiYJnAmCP3NOTd0jMZEnDkbUvxhMmBYSdETk1rRgm+R4LOzFUGaHqHDLKLX+FIPKcF96hrucXzcWyLbIbEgE98OHlnVYCzRdK8jlqm8tehUc9c9WhQ== vagrant insecure public key

vagrant@TDExpress1620_Sles11:~> chmod 0700 .ssh
vagrant@TDExpress1620_Sles11:~> chmod 0600 .ssh/authorized_keys 
```
6.3.Test vagrant ssh from host
```
$ chmod 0600 vagrant.private 

$ ssh -i vagrant.private -p 2244 vagrant@localhost
Last login: Sun Feb 24 16:07:31 2019 from 10.0.2.2
vagrant@TDExpress1620_Sles11:~> sudo su -
Your use is subject to the terms and conditions of
        the click through agreement that brought you to this
        screen ("TERADATA EXPRESS") EVALUATION AND DEVELOPMENT
        LICENSE AGREEMENT), including the restriction that this
        evaluation copy is not for production use.
```
6.4.Create vagrant box:
```
$ vagrant package --base td_express_16_20
==> td_express_16_20: Attempting graceful shutdown of VM...
==> td_express_16_20: Clearing any previously set forwarded ports...
==> td_express_16_20: Exporting VM...
==> td_express_16_20: Compressing package to: /root/TERADATA/package.box
```
6.5.Upload image to vagrant cloud: 

Vagrant-Cloud Link:  https://app.vagrantup.com/davarski/boxes/td_express_16_20

```
$ vagrant cloud auth login

$ vagrant cloud publish davarski/td_express_16_20 1.0.0 virtualbox ./package.box -d "Teradata Express 16.20" --release 

```
6.6.Create Vagrantfile and vagrant up:
```
Vagrant.configure("2") do |config|
  config.vm.box = "davarski/td_express_16_20"
  config.vm.box_version = "1.0.0"
  config.vm.network "forwarded_port", guest: 1025, host: 1025
  config.vm.synced_folder ".", "/vagrant", disabled: true
end

$vagrant up 

```
NOTE:
If you don't want to use vagrant cloud box, but use previous created vagrant box localy
```
$ vagrant box add --force td_express_16_20_dev ./package.box 

Create Vagrantfile:

Vagrant.configure("2") do |config|
  config.vm.box = "td_express_16_20"
  config.vm.network "forwarded_port", guest: 1025, host: 1025
  config.vm.synced_folder ".", "/vagrant", disabled: true
end

$ vagrant up

```

### ****NOTE: TDExpress14.10****

```
7z e TDExpress14.10_Sles11_40GB.7z 

VBoxManage internalcommands sethduuid ./sda.vmdk
VBoxManage internalcommands sethduuid ./PDISK0.vmdk
VBoxManage internalcommands sethduuid ./PDISK1.vmdk
VBoxManage clonehd sda.vmdk sda.vdi -format VDI
VBoxManage clonehd PDISK0.vmdk PDISK0.vdi -format VDI
VBoxManage clonehd PDISK1.vmdk PDISK1.vdi -format VDI

export VM_NAME="teradata_express_14_10"

vboxmanage createvm --name $VM_NAME --ostype OpenSUSE_64 --register
vboxmanage storagectl $VM_NAME --name "SCSI Controller" --add scsi --controller LsiLogic
vboxmanage storageattach $VM_NAME --storagectl "SCSI Controller" --port 0 --device 0 --type hdd --medium ./sda.vdi
vboxmanage storageattach $VM_NAME --storagectl "SCSI Controller" --port 1 --device 0 --type hdd --medium ./PDISK0.vdi
vboxmanage storageattach $VM_NAME --storagectl "SCSI Controller" --port 2 --device 0 --type hdd --medium ./PDISK1.vdi
vboxmanage modifyvm $VM_NAME --ioapic on
vboxmanage modifyvm $VM_NAME --memory 4096 --vram 128
vboxmanage modifyvm $VM_NAME --nic1 nat
vboxmanage modifyvm $VM_NAME --natpf1 "guestssh,tcp,,2244,,22"
vboxmanage modifyvm $VM_NAME --natpf1 "guestteradata,tcp,,1025,,1025"
vboxmanage modifyvm $VM_NAME --natdnshostresolver1 on


vboxmanage startvm $VM_NAME --type headless


TDExpress14.10_Sles11:~ # pdestate -a
PDE state is RUN/STARTED.
DBS state is 4: Logons are enabled - Users are logged on

TDExpress14.10_Sles11:/etc # diff hosts hosts.ORIG 
24,25c24
< #192.168.56.137   nxs109.td.teradata.com   nxs109   nxs109cop1   dbccop1
< 127.0.0.1 localhost dbccop1
---
> 192.168.56.137   nxs109.td.teradata.com   nxs109   nxs109cop1   dbccop1

TDExpress14.10_Sles11:~ # bteq

 Teradata BTEQ 14.10.00.00 for LINUX.
 Copyright 1984-2013, Teradata Corporation. ALL RIGHTS RESERVED.
 Enter your logon or BTEQ command:
.logon dbc

.logon dbc
Password: 


 *** Logon successfully completed.
 *** Teradata Database Release is 14.10.00.02                   
 *** Teradata Database Version is 14.10.00.02                     
 *** Transaction Semantics are BTET.
 *** Session Character Set Name is 'ASCII'.
 
 *** Total elapsed time was 10 seconds.
 
 BTEQ -- Enter your SQL request or BTEQ command: 
select * from dbc.dbcinfo;

select * from dbc.dbcinfo;

 *** Query completed. 3 rows found. 2 columns returned. 
 *** Total elapsed time was 1 second.

InfoKey                        InfoData
------------------------------ --------------------------------------------
LANGUAGE SUPPORT MODE          Standard
RELEASE                        14.10.00.02
VERSION                        14.10.00.02



vboxmanage controlvm  teradata_express_14_10 poweroff

vboxmanage startvm teradata_express_14_10 --type headless
```

## Troubleshouting:

* to connect with bteq and user/pass dbc/dbc:

 Teradata BTEQ 13.00.00.03 for WIN32.
Copyright 1984-2009, Teradata Corporation. ALL RIGHTS RESERVED.
Enter your logon or BTEQ command:
.logon dbc

.logon dbc
Password:
*** CLI error: MTDP: EM_NOHOST(224): name not in HOSTS file or names database.

*** Return code from CLI is: 224
*** Error: Logon failed!

I’ve started reading documentation and Internet. And I found solution! 🙂

Add this entry to your hosts file /etc/hosts:

IP-address dbccop1

where IP-address usually is 127.0.0.1
