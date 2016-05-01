# Debian & Ubuntu Server
Install "SSH server" and "standard system utilities".


## GRUB
Reduce the GRUB boot timeout.

```sh
vi /etc/default/grub
update-grub
```


## Updates
Install software updates.

```sh
apt-get update
apt-get upgrade
```


## Packages
Install software packages.

```sh
apt-get install zsh vim tmux tree sudo resolvconf
apt-get install git subversion p7zip zip unzip
```


## Configuration
Inspect the current network configuration.

```sh
ip addr
ip route
cat /etc/resolv.conf
```

Set a static IP address.

```sh
vi /etc/network/interfaces
```

```
auto eth0
iface eth0 inet static
  address 10.0.0.11
  netmask 255.255.255.0
  gateway 10.0.0.1
  dns-nameservers 8.8.8.8 8.8.4.4
```

```sh
ifup --no-act eth0
tmux -c "ifdown eth0; sleep 1; ifup eth0"
echo -e "nameserver 8.8.8.8\nnameserver 8.8.4.4" | resolvconf -a eth0.inet  # Only on Debian 7.9 and older.
```

Add an administrator account at the bottom of the *sudoers* file before the `#includedir` directive.

```sh
EDITOR=vim visudo
```

```diff
# Administrator account.
qis    ALL=(ALL:ALL) NOPASSWD:ALL
```

Change the login shell.

```sh
chsh -s /usr/bin/zsh qis
```

Create an SSH user directory.

```sh
install -o qis -g qis -d /home/qis/.ssh
```

Upload the configuration files.

```sh
scp .ssh/id_rsa.pub qis@debian:.ssh/authorized_keys
scp -r .vim .vimrc .zsh .zshrc .tmux.conf qis@debian:
```


## CMake
<https://cmake.org/download/>

Install an up to date *CMake* version.

```sh
wget https://cmake.org/files/v3.5/cmake-3.5.2-Linux-x86_64.sh
yes | sudo sh cmake-3.5.2-Linux-x86_64.sh --prefix=/opt
sudo mv /opt/cmake-3.5.2-Linux-x86_64 /opt/cmake
# NOTE: Add /opt/cmake/bin to the PATH environment variable.
```


## Compiler
Install *LLVM* and *clang*.

### Debian 7.9
There is no modern *clang* for Debian "wheezy".

```sh
sudo apt-get install make
```

### Debian 8.4
Install *clang* on Debian "jessie".

```sh
wget -O - http://llvm.org/apt/llvm-snapshot.gpg.key | sudo apt-key add -
sudo vim /etc/apt/sources.list.d/llvm.list
```

```
deb http://llvm.org/apt/jessie/ llvm-toolchain-jessie-3.8 main
deb-src http://llvm.org/apt/jessie/ llvm-toolchain-jessie-3.8 main
```

```sh
sudo apt-get update
sudo apt-get install make llvm-3.8 clang-3.8
```

### Ubuntu Server 14.04
Install *clang* on Ubuntu Server "Trusty".

```sh
wget -O - http://llvm.org/apt/llvm-snapshot.gpg.key | sudo apt-key add -
sudo vim /etc/apt/sources.list.d/llvm.list
```

```
deb http://llvm.org/apt/trusty/ llvm-toolchain-trusty-3.8 main
deb-src http://llvm.org/apt/trusty/ llvm-toolchain-trusty-3.8 main
```

```sh
sudo apt-get update
sudo apt-get install make llvm-3.8 clang-3.8
```

### Ubuntu Server 16.04
Install *clang* on Ubuntu Server "Xenal".

```sh
wget -O - http://llvm.org/apt/llvm-snapshot.gpg.key | sudo apt-key add -
sudo vim /etc/apt/sources.list.d/llvm.list
```

```
deb http://llvm.org/apt/xenial/ llvm-toolchain-xenial-3.8 main
deb-src http://llvm.org/apt/xenial/ llvm-toolchain-xenial-3.8 main
```

```sh
sudo apt-get update
sudo apt-get install make llvm-3.8 clang-3.8
```


# Standard Library
<http://libcxxabi.llvm.org/>
<http://libcxx.llvm.org/>

Install *libc++-abi* and *libc++* on Debian 8 and Ubuntu.

```sh
sudo apt-get install musl-dev libunwind-dev
wget http://llvm.org/releases/3.8.0/llvm-3.8.0.src.tar.xz
wget http://llvm.org/releases/3.8.0/cfe-3.8.0.src.tar.xz
wget http://llvm.org/releases/3.8.0/libcxx-3.8.0.src.tar.xz
wget http://llvm.org/releases/3.8.0/libcxxabi-3.8.0.src.tar.xz
wget http://llvm.org/releases/3.8.0/libunwind-3.8.0.src.tar.xz

tar xf llvm-3.8.0.src.tar.xz
tar xf cfe-3.8.0.src.tar.xz
tar xf libcxx-3.8.0.src.tar.xz
tar xf libcxxabi-3.8.0.src.tar.xz
tar xf libunwind-3.8.0.src.tar.xz

mv llvm-3.8.0.src llvm
mv cfe-3.8.0.src llvm/projects/clang
mv libcxx-3.8.0.src llvm/projects/libcxx
mv libcxxabi-3.8.0.src llvm/projects/libcxxabi
mv libunwind-3.8.0.src llvm/projects/libunwind

mkdir llvm/build && cd llvm/build
CC=clang-3.8 CXX=clang++-3.8 cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DLIBUNWIND_ENABLE_SHARED=OFF \
  -DLIBCXXABI_USE_LLVM_UNWINDER=ON \
  -DLIBCXXABI_ENABLE_SHARED=OFF \
  -DLIBCXXABI_ENABLE_STATIC=ON \
  -DLIBCXX_HAS_MUSL_LIBC=ON \
  -DLIBCXX_ENABLE_STATIC_ABI_LIBRARY=ON \
  -DLIBCXX_ENABLE_SHARED=OFF \
  -DLIBCXX_CXX_ABI=libcxxabi \
  -DLIBCXX_CXX_ABI_INCLUDE_PATHS="../projects/libcxxabi/include" \
  ..

make
sudo make install
```


## Test
Try to compile a statically linked binary that runs on older distribution versions.

```cpp
// main.cc
#include <exception>
#include <iostream>
#include <stdexcept>
#include <thread>

int main() {
  try {
    auto thread = std::thread([]() {
      try {
        throw std::runtime_error("1");
      }
      catch (const std::exception& e) {
        std::cout << "success: " << e.what() << std::endl;
      }
    });
    thread.join();
    throw std::runtime_error("2");
  }
  catch (const std::exception& e) {
    std::cout << "success: " << e.what() << std::endl;
    return 0;
  }
  return 1;
}
```

Compile the source code.

```sh
clang++ -std=c++14 -stdlib=libc++ -O3 -c -o main.o main.cc
clang++ -std=c++14 -stdlib=libc++ -static-libgcc -static-libstdc++ -static -pthread -o app main.o
ldd app
./app
```

Copy it over to another system and run it.
