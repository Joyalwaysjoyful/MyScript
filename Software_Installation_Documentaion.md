

# Software Installation Documentation #

			
This is documentation written and updated by Jiuke Chen at Empa, aiming at recording the installation and troubleshooting steps. 

Updated by jc: 11/5/2021 4:05:47 PM



## CONTENTS ##

1. **Ubuntu installation**
2. **OpenMPI**
3. **FFTW3**
4. **CUDA**
5. **Xrdp**
6. **Mount cifs**
6. **Build LAMMPS with CMAKE**
	- **GPU package**
	- **KOKKOS package**



## 1. UBUNTU INSTALLATION ##

When grub is not installed correctly, grub will throw an error and enter in rescue mode.
error: symbol 'grub_file_filters' not found.

### Troubleshooting ###

1) Enter Ubuntu live mode using a 'rescue' USB

2) Open a terminal and run:

    sudo su
    fdisk -l
    mount /dev/sda1 /mnt
    chroot /mnt
    
    grub-install /dev/sda or grub-install --root-directory=/mnt /dev/sda

If everything's going well, you will see a message: Installation finished. No error reported.

3) Cleanup and reboot

	exit
	unmount /mnt
	reboot

if "error: symbol 'grub_disk_native_sectors' not found
then chek BIOS boot options, put ubuntu booting in the 1st order.



## 2. OpenMPI ##

1) Installation

	tar -zxvf openmpi.tar.gz
	cd openmpi

	./configure --prefix=/opt/openmpi
	make all install


2) Add to bash file

	export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/opt/openmpi/lib"
	export OPAL_PREFIX=/opt/openmpi




## 3. FFTW3 ##

1) Installation

	tar -zxvf Downloads/fftw-3.3.9.tar.gz
	cd fftw-3.3.9/

	./configure --prefix=/usr/local/fftw
	make
	make install



## 4. CUDA ##

1) Installation via deb internet [check website]

	wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/cuda-ubuntu2004.pin
	sudo mv cuda-ubuntu2004.pin /etc/apt/preferences.d/cuda-repository-pin-600
	sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/7fa2af80.pub
	sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2004/x86_64/ /"
	sudo apt-get update
	sudo apt-get -y install cuda

2) NVIDIA driver

	check GPU info: sudo lshw -C display OR ubuntu-drivers devices

	apt-cache search nvidia-driver
	sudo apt update
	sudo apt upgrade
	
	sudo apt install nvidia-driver-460 OR sudo ubuntu-drivers autoinstall
	sudo reboot

3) Verification

	nvidia-smi

4) Add to bash file

	export CUDA_PATH=/usr/local/cuda-11.4/bin

### Troubleshooting ###

	To solve 'Failed to initialize NVML: Driver/library version mismatch', check if it mismatch. Otherwise, reboot makes it work in most cases. 



## 5. Xrdp ##


1) Get the desktop manager: 

	sudo apt update
	sudo apt install ubuntu-desktop(Gnome)/xubuntu-desktop(xfce)

2) Install xrdp

	sudo apt install xrdp
	sudo systemctl status xrdp
	sudo systemctl enable xrdp

3) Add users

	#xrdp uses ssl-cert key file readable only by "ssl-cert" group.
	sudo adduser xrdp ssl-cert

	#create the user.
	sudo adduser --home /home/<username> <username>
	
	#create the groups.
	sudo addgroup tsusers
	sudo addgroup tsadmins

	#add the new user to the xrdp group.
	sudo adduser <username> tsusers

	#restart xrdp for changes to take effect.
	sudo service restart xrdp

### Troubleshooting ###

If come across the error when log in : xrdp failed for display 0

	a. check username/passwd
	b. check if the user has been added to the groups

change to xfce

	a. install desktop
	b. echo "xfce4-session" > .xsession

for Ubuntu 22.04, how to set xrdp can check [here](https://linuxhint.com/enable-remote-desktop-ubuntu-access-windows/).

REMEMBER: after using, you need to log out from the remote session, otherwise, on local Ubuntu this account cannot access the graphical interface .

## 6. Mount Cifs ##

1) Install utils

	sudo apt install cifs-utils key

	**Ali
	**ONLY for the first time:

	sudo apt-get install cifs-utils

	**after EVERY reboot:

	sudo mount -t cifs //empa.emp-eaw.ch/data/Projects/SimPro /home/ali/Documents/sharedData -o username=gal,vers=2.1 

	Then enter the huge password for empa login!


	***in newer versions Ubuntu 20.04LTS, i needed this too:
	sudo apt-get install keyutils
	then:
	sudo mount -t cifs //empa.emp-eaw.ch/data/Projects/SimPro /home/username/Documents/sharedData -o user=shortname,vers=3


## 7. Build LAMMPS with CMake ##

Prerequisite: MPICH

1) Requirements for CMake

	check cmake: which cmake which cmake3
	sudo apt install build-essential cmake gfortran
	apt update
	apt upgrade


2) Generate a build environment in a new folder

	mkdir build ; cd build

	#Impatient way
	cmake ../cmake
	cmake --build

	#Custom
	cmake -C ../cmake/presets/minimal.cmake ../cmake
	make
	make install	

### Troubleshooting ###

	'Parse error in command line argument...
	CMake Error: No cmake script provided.
	Cmake Error: Problem processing arguments. Aborting.'
	No space after '-D' expression. It should be like '-D EXPRESSION=EXPRESSION'

3) Build LAMMPS/GPU with CMake

	cmake -C ../cmake/presets/minimal.cmake -D PKG_GPU=on -D GPU_API=cuda -D GPU_ARCH=sm_61 ../cmake
	make -j20
	sudo make install

4) Build LAMMPS/KOKKOS with CMake

	cmake -C ../cmake/presets/kokkos-cuda.cmake -DBUILD_OMP=yes -D LAMMPS_MACHINE=mpi -D PKG_KOKKOS=on 
	-DKokkos_ARCH_PASCAL61=on -D Kokkos_ENABLE_CUDA=yes -D Kokkos_ENABLE_OPENMP=yes -DCMAKE_CXX_COMPILER=/.../lib/kokkos/bin/nvcc_wrapper ../cmake
	make -j20
	sudo make install

### Run with kokkos package ###

    nohup mpirun -np *number1 lmp_mpi* -k on g *number2 -sf kk -pk kokkos [newton/neigh binsize] < in. > out. &

> number1 refers to how many cores and number2 refers to gpu you will use. 

### Ethernet unmanaged ###

1. open /etc/NetworkManager/NetworkManger.conf, change managed=false to managed=true
2. if nmcli device show *-network* DISABLED, it means that the ethernet cable not being found. To correct it, edit /usr/lib/NetworkManager/conf.d/10-globally-managed-devices.conf (if not exists, create it by 'touch')
	
`unmanaged-devices=*,except:type:ethernet,except:type:wifi` 

3. after step 1&2, the network is online. Otherwise, reboot or sudo ystemctl restart network-manager.service
