# base_os


> Date: 17 Nov. 25
>
> Author: Adam Svahn


This project is an ansible role for managing the underlying operating system of cloud VMs. It is intended to be imported into downstream projects.

If this role needs to be updated, please update it at the upstream source.

Note, at present this role assumes debian trixie when updating sources. Generalisation to older debian and debian-based systems (i.e. ubuntu) to follow.

## Notes on main tasks

- The update OS task (tag: ```update```) has three steps:
	- Debian default sources will be replaced with the contents of ```files/custom.sources``` if this file exists
	- Apt update/upgrade will be run and a reboot initiated if required
	- Apt cache cleanup 

- The hostnames task (tag: ```hostnames```) will add an entry for all hosts in the ```hosts.yml``` inventory file to /etc/hosts on the target. It will also replace the local hostname with the hostname as listed in hosts.yml. 

- The hide PIDs task (tag: ```proc_hide_pid```) will update the ```/etc/fstab``` entry for ```/proc``` to hide PIDs not belonging to the user ID. 

## Notes on optional tasks

- The stop apt.daily services task (tag ```stop-apt-daily```) will disable apt.daily and unattended updgrade services and timers.

- The zram_swappiness task (tags:```create-update-zram``` and ```zram-swappiness```) will create a zram (compressed memory) device of 80% of total ram to be used as high-priority swap. Swappiness is then updated to reflect in memory swap, see the [Arch wiki page on zram](https://wiki.archlinux.org/title/Zram) and the [kernal docs on swappiness](https://docs.kernel.org/admin-guide/sysctl/vm.html#swappiness). The values for swappiness are in a ```defaults``` file, and can be commented to revert to default values. 