---
title: "MRtrix3系列教程"
date: 2023-01-05T23:05:15+08:00
draft: false
weight: 2.2
pre: "<b>4. </b>"
---


- [简介](#简介)
- [下载安装](#下载安装)
- [校验是否安装成功](#校验是否安装成功)



#### 简介
MRtrix3提供了一套工具来进行各种类型的弥散MRI分析，从各种形式的纤维束成像到下一代的组水平分析。它的设计考虑到了一致性、性能和稳定性，并以开源许可的方式免费提供。它是由该领域的专家团队开发和维护的，培养了一个由不同背景的用户组成的活跃社区。


![MRtrix](/mrtrix/images/mrtrix_home.jpeg)
- **与主要图像格式无缝交互**

![MRtrix](/mrtrix/images/foddec-pansharp.jpeg)
- **FOD-based（Fiber Orientation Distributions，FOD） DEC maps and panchromatic sharpening**


#### 下载安装
访问[MRtrix的下载地址](https://www.mrtrix.org/download/),选择不同平台进行安装即可。


{{< tabs >}}
{{% tab name="macOS" %}}
```shell
sudo bash -c "$(curl -fsSL https://raw.githubusercontent.com/MRtrix3/macos-installer/master/install)"
```
该脚本会自动下载MRtrix3软件的最新的二进制发布版本，解压到`/usr/local/mrtrix3`目录下。
{{% /tab %}}

{{% tab name="Linux" %}}
```shell
$ conda install -c mrtrix3 mrtrix3
```
前提，需要先安装anaconda (or miniconda)。
{{% /tab %}}

{{< /tabs >}}



如果碰到macOS安装时，无法正常下载```install```脚本，可以通过网页访问[raw.githubusercontent.com/MRtrix3/macos-installer/master/install](https://raw.githubusercontent.com/MRtrix3/macos-installer/master/install)，然后自己创建文件进行安装。下面是我安装时的install脚本内容。

```shell
#!/bin/bash -e
tag=$(basename $(/usr/bin/curl -Ls -o /dev/null -w %{url_effective} https://github.com/MRtrix3/mrtrix3/releases/latest))
if [ -z "${tag}" ]; then
  echo "ERROR: could not find tag name for latest release ..."
  exit
fi 

if [ "$1" != "-f" ]; then
  echo "This installer will download MRtrix ${tag} and install it to /usr/local/mrtrix3."
  echo "In addition it will:"
  echo "* create symbolic links in /usr/local/bin to the binaries in /usr/local/mrtrix3/bin"
  echo "* create symbolic links in /Applications to the app bundles in /usr/local/mrtrix3/bin"
fi

if [ $EUID != 0 ]; then
  echo "ERROR: This script requires root privileges, please run as: sudo "$0" "$@""
  exit
fi

if [ -d "/usr/local/mrtrix3" ] || [ -L "/usr/local/mrtrix3" ] ; then
  echo "WARNING: /usr/local/mrtrix3 already exists and will be replaced during installation."
fi

if [ "$1" != "-f" ]; then
  while true; do
    read -p "Are you sure you want to continue? [y/n] " yn
    case $yn in
      [Yy]* ) break;;
      [Nn]* ) exit;;
    esac
  done
fi

if [ ! -d "/usr/local/bin" ]; then
  if [ -e "/usr/local/bin" ]; then
    echo "WARNING: /usr/local/bin is not a directory, cannot create symlinks"
  else
    echo "WARNING: /usr/local/bin does not exist, creating it for you."
    mkdir -p -m 755 /usr/local/bin
  fi
fi

url=https://github.com/MRtrix3/mrtrix3/releases/download/${tag}/mrtrix3-macos-${tag}.tar.gz

if [ -z "${url}" ]; then
  echo "ERROR: Could not find tarball of latest release ..."
  exit
fi

echo "Downloading "${url}" ..."
/usr/bin/curl -sL "${url}" -O

tarball=$(basename "${url}")
if [ ! -f "${tarball}" ]; then
  echo "ERROR: Download not sucessful ..."
  exit
fi

if [ -f /usr/local/mrtrix3/symlinks ]; then
  echo "Removing symbolic links of previous installation ..."
  for l in $(cat /usr/local/mrtrix3/symlinks); do
    if [ -L "${l}" ]; then
      unlink "${l}"
    fi
  done
fi

if [ -d "/usr/local/mrtrix3" ] || [ -L "/usr/local/mrtrix3" ] ; then
  echo "Removing previous installation in /usr/local/mrtrix3 ..."
  rm -rf "/usr/local/mrtrix3"
fi

for l in /usr/local/bin/*; do
  if [ -L "${l}" ]; then
    t="$(readlink "${l}")"
    if [[ "${t}" == *"mrtrix3"* ]]; then
      echo "WARNING: Removing symbolic link "${l}" to "${t}" (conflicting homebrew installation?) ..."
      unlink "${l}"
    fi
  fi
done

echo "Unpacking "${tarball}" to /usr/local/mrtrix3 ..."
tar oxf "${tarball}" -C /usr/local
rm "${tarball}"

echo "Fixing python shebang ..."
cd /usr/local/mrtrix3/bin
for f in $(grep -lr '^#!/usr/bin/env python'); do
  sed -i '' 's|^#!/usr/bin/env python$|#!/usr/bin/python3|' ${f}
done

echo "Applying patch to show correct version numbers in scripts ..."
cd /usr/local/mrtrix3
curl -s https://github.com/MRtrix3/mrtrix3/commit/4a293d30e1c0686037ae637f67932d497eb71ee6.patch | patch -s -p1

echo "Creating symlinks in /usr/local/bin ..."
cd /usr/local/bin
touch /usr/local/mrtrix3/symlinks
for target in $(find ../mrtrix3/bin -maxdepth 1 -type f ! -name "*.*"); do
  ln -sf "${target}"
  echo /usr/local/bin/"$(basename "${target}")" >> /usr/local/mrtrix3/symlinks
done

echo "Creating symlinks in /Applications ..."
cd /Applications
for target in /usr/local/mrtrix3/bin/*.app; do
  ln -sf "${target}"
  echo /Applications/"$(basename "${target}")" >> /usr/local/mrtrix3/symlinks
done

if [[ $SHELL = "/bin/zsh" ]]; then profile=~/.zprofile; else profile=~/.bash_profile; fi
echo "$PATH" | grep -q '/usr/local/bin:' || printf "WARNING: /usr/local/bin is not in PATH. You can add it to the PATH with:\necho 'export PATH=/usr/local/bin:\$PATH' >> ${profile}\n"

echo "Installation complete!"
```

```shell
$ touch install #创建脚本文件，然后粘贴上述脚本内容。
$ chmod +x install
$ sudo ./install # 执行安装,若因为github的原因导致无法下载，建议在terminal内配置github代理，或者手动下载软件包，修改install脚本,让其读取本地已下载的软件包安装即可。
```



#### 校验是否安装成功
新起一个terminal，随意选择MRtrix的命令，输入部分命令，比如`dwi2`,然后按tab键，输入`dwi2mask -h`,如果可以正常显示命令,那么就应该安装成功了。
```shell
$ dwi2mask -h
     dwi2mask: part of the MRtrix3 package

SYNOPSIS

     Generates a whole brain mask from a DWI image

USAGE

     dwi2mask [ options ] input output

        input        the input DWI image containing volumes that are both
                     diffusion weighted and b=0

        output       the output whole-brain mask image


DESCRIPTION

     All diffusion weighted and b=0 volumes are used to obtain a mask that
     includes both brain tissue and CSF.
```


[官网地址](https://www.mrtrix.org/) - 
[GitHub源码地址](https://www.mrtrix.org/download/) - 
[MRtrix官网文档地址](https://mrtrix.readthedocs.io/en/latest/getting_started/beginner_dwi_tutorial.html#dwi-geometric-distortion-correction)