dpdk版本24.11.2

meson版本：1.7.2



下载dpdk 24.11.2的压缩包并解压

```
wget https://fast.dpdk.org/rel/dpdk-24.11.tar.xz
tar xf dpdk-24.11.tar.xz
```



开始构建dpdk库程序

```
$ cd dpdk-24.11
$ mkdir dpdklib                 # 用户自定义的安装目录
$ mkdir dpdkbuild               # 用户自定义的构建目录
$ meson setup dpdkbuild -Denable_kmods=true -Dprefix=$PWD/dpdklib
$ ninja -C dpdkbuild
$ cd dpdkbuild; ninja install
$ export PKG_CONFIG_PATH=$(pwd)/../dpdklib/lib64/pkgconfig/
```



linux下的驱动这块，之前是uio，现在改为什么了呢？

现在的dpdk有几种驱动呢？



