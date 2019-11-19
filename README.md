# Final Project

## Installation of Gem5 and Parsec
The following installation commands are adapted from https://github.com/arm-university/arm-gem5-rsk/wiki.

Dependencies check...
```
sudo apt-get install zlib1g-dev
sudo apt-get install m4
sudo apt-get install boost
sudo apt-get install libboost-all-dev
sudo apt-get install gcc-arm-linux-gnueabihf
sudo apt-get update
mkdir comparch
cd comparch
```

Step 1. Install the ARM starter kit
```
wget https://raw.githubusercontent.com/arm-university/arm-gem5-rsk/master/clone.sh
bash clone.sh
```
Step 2. Build gem5
```
cd gem5
scons build/ARM/gem5.opt -j4 
```

Step 3. Get Arm full system disk image and set `$M5_PATH`:
```
mkdir aarch-system
cd aarch-system
wget http://www.gem5.org/dist/current/arm/aarch-system-20170616.tar.xz
tar xvfJ aarch-system-20170616.tar.xz

echo "export M5_PATH=~/Desktop/comparch/gem5/aarch-system" >> ~/.bashrc 
source ~/.bashrc
cd ..
```

Step 4. Download and compile Parsec3.0
```
wget http://parsec.cs.princeton.edu/download/3.0/parsec-3.0.tar.gz
tar -xvzf parsec-3.0.tar.gz
cd parsec-3.0
patch -p1 < ../arm-gem5-rsk/parsec_patches/static-patch.diff
```
replace `config.guess` and `config.sub`:
```
mkdir tmp; cd tmp # make a tmp dir outside the parsec dir
wget -O config.guess 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.guess;hb=HEAD'
wget -O config.sub 'http://git.savannah.gnu.org/gitweb/?p=config.git;a=blob_plain;f=config.sub;hb=HEAD'
cd ~/Desktop/comparch/parsec-3.0 # cd to the parsec dir
find . -name "config.guess" -type f -print -execdir cp {} config.guess_old \;
find . -name "config.guess" -type f -print -execdir cp ~/Desktop/comparch/tmp/config.guess {} \;
find . -name "config.sub" -type f -print -execdir cp {} config.sub_old \;
find . -name "config.sub" -type f -print -execdir cp ~/Desktop/comparch/tmp/config.sub {} \;
```
Step 5. Get the `aarch64-linux-gnu`:
```
wget https://releases.linaro.org/components/toolchain/binaries/latest-5/aarch64-linux-gnu/gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu.tar.xz
tar xvfJ gcc-linaro-5.5.0-2017.10-x86_64_aarch64-linux-gnu.tar.xz 
```
Step 6. Update `xcompile-patch.diff`.Change the `$CC_HOME` and the `$BINUTIL_HOME` in the xcompile-patch.diff to point to the downloaded `<gcc-linaro directory>` and `<gcc-linaro directory>/aarch64-linux-gnu` directories.
```
patch -p1 < ../arm-gem5-rsk/parsec_patches/xcompile-patch.diff
```

Step 7. Compile some package! Replace `<pkgname>` with a Parsec3.0 package.
```
export PARSECPLAT="aarch64-linux" # set the platform 
source ./env.sh
parsecmgmt -a build -c gcc -p <pkgname>
```
Example package: 
```
parsecmgmt -a build -c gcc -p canneal
```

## Running Parsec
Step 1. Expand the disk image to add room for the Parsec library to be copied onto it. 
```
cd ../gem5/aarch-system/disks
cp linaro-minimal-aarch64.img expanded-linaro-minimal-aarch64.img
dd if=/dev/zero bs=1G count=20 >> ./expanded-linaro-minimal-aarch64.img # add 20G zeros
sudo parted expanded-linaro-minimal-aarch64.img resizepart 1 100% # grow partition 1
```
Step 2. Mount the disk. 
```
mkdir disk_mnt
name=$(sudo fdisk -l expanded-linaro-minimal-aarch64.img | tail -1 | awk -F: '{ print $1 }' | awk -F" " '{ print $1 }')
start_sector=$(sudo fdisk -l expanded-linaro-minimal-aarch64.img | grep $name | awk -F" " '{ print $2 }')
units=512
sudo mount -o loop,offset=$(($start_sector*$units)) expanded-linaro-minimal-aarch64.img disk_mnt
```
When running `df`, find the loop that contains the filepath for `disk_mnt`. For example, mine was `/dev/loop18`.
```
df # find /dev/loopX for disk_mnt
sudo resize2fs /dev/loopX # resize filesystem
df # check that the Available space for disk_mnt is increased
sudo cp -r ~/Desktop/comparch/parsec-3.0/ disk_mnt/home/root # copy the compiled parsec-3.0 to the image
ls disk_mnt/home/root # check the parsec-3.0 contents
sudo umount disk_mnt
```
Step 3. Run! 
```
cd ~/Desktop/comparch/arm-gem5-rsk/parsec_rcs
bash gen_rcs.sh -p <pkgname> -i <simsmall/simmedium/simlarge> -n <nth>
cd ~/Desktop/comparch/gem5
./build/ARM/gem5.opt -d fs_results/<benchmark> configs/example/arm/starter_fs.py --cpu="hpi" --num-cores=1 --disk-image=$M5_PATH/disks/expanded-linaro-minimal-aarch64.img --script=../arm-gem5-rsk/parsec_rcs/<benchmark>.rcS
```
Example:
```
cd ~/Desktop/comparch/arm-gem5-rsk/parsec_rcs
bash gen_rcs.sh -i simsmall -p canneal -n 4
cd ~/Desktop/comparch/gem5
./build/ARM/gem5.opt -d fs_results/canneal configs/example/arm/starter_fs.py --cpu="hpi" --num-cores=1 --disk-image=$M5_PATH/disks/expanded-linaro-minimal-aarch64.img --script=../arm-gem5-rsk/parsec_rcs/canneal_simsmall_4.rcS
```
## Running Final Project
```
# Experiment 1
bash gen_rcs.sh -i simsmall -p canneal -n 4 
./build/ARM/gem5.opt -d fs_results/canneal /home/mschiappa/Desktop/comparch/gem5/configs/example/arm/modified_fs_bigLITTLE.py --cpu=exynos --little-cpus=4 --last-cache-level=3 --disk=$M5_PATH/disks/expanded-linaro-minimal-aarch64.img --bootscript=../arm-gem5-rsk/parsec_rcs/canneal_simsmall_4.rcS
```
