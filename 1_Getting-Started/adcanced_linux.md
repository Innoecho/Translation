# Arch及CentOS工具链安装指南

## USB设备配置

Linux用户需要明确地允许JTAG编程适配器可以访问USB总线。

<aside class="note">
对于Archlinux：用```uucp```替换```plugdev```，然后执行下面命令。
</aside>

在sudo模式下执行ls命令以确保之后的命令可以成功执行：

<div class="host-code"></div>

```sh
sudo ls
```

暂时获得sudo权限，执行以下命令：

<div class="host-code"></div>

```sh
cat > $HOME/rule.tmp <<_EOF
# All 3D Robotics (includes PX4) devices
SUBSYSTEM=="usb", ATTR{idVendor}=="26AC", GROUP="plugdev"
# FTDI (and Black Magic Probe) Devices
SUBSYSTEM=="usb", ATTR{idVendor}=="0483", GROUP="plugdev"
# Olimex Devices
SUBSYSTEM=="usb",  ATTR{idVendor}=="15ba", GROUP="plugdev"
_EOF
sudo mv $HOME/rule.tmp /etc/udev/rules.d/10-px4.rules
sudo /etc/init.d/udev restart
```

用户需要被添加到plugdev组：

<div class="host-code"></div>

```sh
sudo usermod -a -G plugdev $USER
```

## 小众Linux系统工具链安装指南

### CentOS

构建需要Python 2.7.5支持，因此应该使用CentOS 7（当前最新版本，较早的CentOS版本可能带有python v2.7.5，但是不推荐使用，因为它可能会破坏yum）。

需要使用EPEL仓库来安装openocd，libftdi-devel和libftdi-python。

<div class="host-code"></div>

```sh
wget https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
sudo yum install epel-release-7-5.noarch.rpm
yum update
yum groupinstall “Development Tools”
yum install python-setuptools
easy_install pyserial
easy_install pexpect
yum install openocd libftdi-devel libftdi-python python-argparse flex bison-devel ncurses-devel ncurses-libs autoconf texinfo libtool zlib-devel cmake
```

注意：你可能还想要安装python-pip和screen。

#### 其它32位库

安装完arm工具链之后，执行以下命令测试：

<div class="host-code"></div>

```sh
arm-none-eabi-gcc --version
```

如果返回下列信息

<div class="host-code"></div>

```sh
bash: gcc-arm-none-eabi-4_7-2014q2/bin/arm-none-eabi-gcc: /lib/ld-linux.so.2: bad ELF interpreter: No such file or directory
```

那么你还需要安装其它的32位库：glibc.i686，ncurses-libs.i686

<div class="host-code"></div>

```sh
sudo yum install glibc.i686 ncurses-libs.i686 
```

<aside class="note">
安装ncurses-libs.i686将会同时安装其它大部分的依赖32位库。CentOS 7将会安装大部分PX4相关设备而不需要添加任何的udev规则。预定义的'dialout'用户组可以访问这些设备，因此可以忽略添加udev规则的资料。所需要确保的仅仅是你的账户是'dialout'组的成员。
</aside>

### Arch Linux

<div class="host-code"></div>

```sh
sudo pacman -S base-devel lib32-glibc git-core python-pyserial zip python-empy
```

安装[Arch User Repository (AUR)](https://wiki.archlinux.org/index.php/Arch_User_Repository)的包管理器[yaourt](https://wiki.archlinux.org/index.php/Yaourt#Installation)。

然后使用它下载，编译以及安装以下内容：

<div class="host-code"></div>

```sh
yaourt -S genromfs
```

#### 权限

用户需要被添加至'uucp'组：

<div class="host-code"></div>

```sh
sudo usermod -a -G uucp $USER
```

执行上述操作后，需要注销账户然后再次登陆。

<aside class="note">
注销账户然后再次登陆的目的是使更改生效，当然移除设备并再次插入也是必要操作。
</aside>

### 工具链安装

执行下面的脚本来安装GCC 4.8.4或者4.9.2:

<div class="host-code"></div>

```sh
pushd .
cd ~
wget https://launchpadlibrarian.net/186124160/gcc-arm-none-eabi-4_8-2014q3-20140805-linux.tar.bz2
tar -jxf gcc-arm-none-eabi-4_8-2014q3-20140805-linux.tar.bz2
exportline="export PATH=$HOME/gcc-arm-none-eabi-4_8-2014q3/bin:\$PATH"
if grep -Fxq "$exportline" ~/.profile; then echo nothing to do ; else echo $exportline >> ~/.profile; fi
. ~/.profile
popd
```

GCC 4.9:

<div class="host-code"></div>

```sh
pushd .
cd ~
wget https://launchpad.net/gcc-arm-embedded/4.9/4.9-2014-q4-major/+download/gcc-arm-none-eabi-4_9-2014q4-20141203-linux.tar.bz2
tar -jxf gcc-arm-none-eabi-4_9-2014q4-20141203-linux.tar.bz2
exportline="export PATH=$HOME/gcc-arm-none-eabi-4_9-2014q4/bin:\$PATH"
if grep -Fxq "$exportline" ~/.profile; then echo nothing to do ; else echo $exportline >> ~/.profile; fi
. ~/.profile
popd
```

<aside class="note">
如果使用Debian Linux，执行下列命令：
</aside>

<div class="host-code"></div>

```sh
sudo dpkg --add-architecture i386
sudo apt-get update
```

安装32位支持库（如果已经是运行在32位，那么可能会失败，则此步骤可以跳过）：

<div class="host-code"></div>

```sh
sudo apt-get install libc6:i386 libgcc1:i386 gcc-4.6-base:i386 libstdc++5:i386 libstdc++6:i386
```

## Ninja构建系统

Ninja比Make更快，并且PX4的CMake生成器可以支持它。不幸的是，Ubuntu目前只支持一个非常过时的版本。下载二进制文件并添加到系统路径来安装最新版本的[Ninja](https://github.com/martine/ninja)：

<div class="host-code"></div>

```sh
mkdir -p $HOME/ninja
cd $HOME/ninja
wget https://github.com/martine/ninja/releases/download/v1.6.0/ninja-linux.zip
unzip ninja-linux.zip
rm ninja-linux.zip
exportline="export PATH=$HOME/ninja:\$PATH"
if grep -Fxq "$exportline" ~/.profile; then echo nothing to do ; else echo $exportline >> ~/.profile; fi
. ~/.profile
```

## 问题解答

### 版本测试

输入：

<div class="host-code"></div>

```sh
arm-none-eabi-gcc --version
```

输出应该类似以下内容：

<div class="host-code"></div>

```sh
arm-none-eabi-gcc (GNU Tools for ARM Embedded Processors) 4.7.4 20140401 (release) [ARM/embedded-4_7-branch revision 209195]
Copyright (C) 2012 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

如果输出是：

<div class="host-code"></div>

```sh
arm-none-eabi-gcc --version
arm-none-eabi-gcc: No such file or directory
```

确认按照安装步骤正确安装32位库。