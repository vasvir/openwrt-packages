# OpenWrt apt-cacher-ng feed

## News
* 2023-10-08 Update to apt-cacher-ng 3.7.4 and openwrt SDK 22.0.3
* 2022-11-27 Update to apt-cacher-ng 3.6.4
* 2020-06-08 Update to openwrt 19.07.3- apt-cacher-ng 3.6.3

## Build the package from apt-cacher-ng feed
1. Some variables - your setup will vary
   ```
   BASEDIR=/home/bill/Downloads/hardware/nanopi-r5c
   ACNG_VERSIO=3.7.4
   OPENSSL_VERSION=${OPENSSL_VERSION}
   TARGET=rockchip-armv8
   PLATFORM=aarch64_generic
   SNAPSHOT=yes
   if [ "${SNAPSHOT}" == yes ]; then
     OPENWRT_SDK=openwrt-sdk-${TARGET}_gcc-12.3.0_musl.Linux-x86_64
     DOWNLOAD_URL=https://downloads.openwrt.org/snapshots/targets/rockchip/armv8
   else
     OPENWRT_VERSION=22.0.3
     OPENWRT_SDK=openwrt-sdk-${OPENWRT_VERSION}-${TARGET}_gcc-12.3.0_musl.Linux-x86_64
     DOWNLOAD_URL=https://downloads.openwrt.org/releases/${OPENWRT_VERSION}/targets/rockchip/armv8
   fi
   ```
1. Download SDK for your board. Note this is only an example. You have to select and download the SDK for your particular board.
    ```
    cd ${BASEDIR}
    wget ${DOWNLOAD_URL}/${OPENWRT_SDK}.tar.xz
    ```

1. Extract SDK
    ```
    tar -xJf {OPENWRT_SDK}.tar.xz
    ```

1. Use the provided feed
    * a. Enable local modifications
        - Download apt-cacher-ng feed from github
            ```
            git clone https://github.com/vasvir/openwrt-packages.git
            ```
        - Edit feeds.conf.default in the sdk downloaded and extracted before. Use the expanded version and not the BASEDIR variable. Add
            ```
            src-link local ${BASEDIR}/openwrt-packages
            ```

    * b. Or Use the feed directly from github
        - Adjust feeds.conf.default. Add
            ```
            src-git local https://github.com/vasvir/openwrt-packages.git
            ```

1. Configure local packages
    ```
    cd ${OPENWRT_SDK}
    ./scripts/feeds update -a
    # required library c-ares
    ./scripts/feeds install libcares
    ./scripts/feeds install apt-cacher-ng
    make menuconfig
    ```

    menuconfig should show apt-cacher-ng under Network/Web Servers/Proxies [1]

1. Install local signing keys
    ```
    ./staging_dir/host/bin/usign -G -s ./key-build -p ./key-build.pub -c "Local build key"
    ```

1. Build package [2]
    ```
    make -j5
    ```

1. Copy ipk file for apt-cacher-ng and libopenssl3 over to openwrt router. libopenssl3 is not installable via opkg in 22.0.3
    ```
    rsync -av bin/packages/${PLATFORM}/local/apt-cacher-ng_${ACNG_VERSION}-1_${PLATFORM}.ipk bin/packages/${PLATFORM}/base/libopenssl3_${OPENSSL_VERSION}-1_${PLATFORM}.ipk root@openwrt:
    ```

1. Install the apt-cacher-ng package and its dependencies [3]
    ```
    ssh root@openwrt
    opkg install libopenssl3_${OPENSSL_VERSION}-1_${PLATFORM}.ipk
    opkg install apt-cacher-${ACNG_VERSION}_${PLATFORM}.ipk
    ```
    
1. Configure apt-cacher-ng service
    * edit /etc/apt-cacher-ng/acng.conf to configure the cachedir and logdir
    * create cachedir, logdir and /var/run/apt-cacher-ng/. Chown them to apt-cacher-ng.apt-cacher.ng

1. Restart the service
    ```
    /etc/init.d/apt-cacher-ng restart
    ```

## TODO
* Autoconfigure step 9

## Troubleshooting
This is possible only if you have followed step 3a.

friendly commands
* make V=s
* make V=s package/feeds/local/apt-cacher-ng/compile
* make V=s package/feeds/local/apt-cacher-ng/install

### [1] Buildroot Makefile problem.
Adjust ${BASEDIR}/openwrt-packages/apt-cacher-ng/Makefile and retry
```
./scripts/feeds uninstall apt-cacher-ng
rm -rf  build_dir/target-${PLATFORM}_musl/apt-cacher-ng-${ACNG_VERSION}/
./scripts/feeds install apt-cacher-ng
make menuconfig
```

This may help if you need to update dependencies
```
./scripts/feeds update -a
```

### [2] Package CMakeLists.txt problem.
Adjust /home/bill/Downloads/hardware/linksys1200ac/openwrt-packages/apt-cacher-ng/patches/000-add_install_target.patch and retry:
* setup once:
    * download the apt-cacher-ng source
        ```
        wget http://ftp.us.debian.org/debian/pool/main/a/apt-cacher-ng/apt-cacher-ng_${ACNG_VERSION}.orig.tar.xz
        ```

    * extract it
         ```
         tar -xJf apt-cacher-ng_${ACNG_VERSION}.orig.tar.xz
         ```

    * rename and copy it to have an easy diff target
         ```
         mv apt-cacher-ng_${ACNG_VERSION} a
         cp -a a b
         ```

* edit test cycle
    * modify b - produce patches for CMakeLists.txt and move them over to /home/bill/Downloads/hardware/linksys1200ac/openwrt-packages/apt-cacher-ng/patches/000-add_install_target.patch
        ```
       diff -ur a b > ${BASEDIR}/openwrt-packages/apt-cacher-ng/patches/000-add_install_target.patch
        ```

    * build
        ```
        rm -rf build_dir/target-${PLATFORM}_musl/apt-cacher-ng-${ACNG_VERSION} 
        make V=s
        ```

### [3] Installation problems in postinst.
Adjust Buildroot Makefile and retry:
```
ssh root@openwrt opkg remove apt-cacher-ng
./scripts/feeds uninstall apt-cacher-ng
rm -rf build_dir/target-${PLATFORM}_musl/apt-cacher-ng-${ACNG_VERSION}
./scripts/feeds install apt-cacher-ng
make -j5
rsync -av bin/packages/${PLATFORM}/local/apt-cacher-ng_${ACNG_VERSION}-1_${PLATFORM}.ipk bin/packages/${PLATFORM}/base/libopenssl3_${OPENSSL_VERSION}-1_${PLATFORM}.ipk root@openwrt:
ssh root@openwrt opkg install libopenssl3_${OPENSSL_VERSION}-1_${PLATFORM}.ipk apt-cacher-${ACNG_VERSION}_${PLATFORM}.ipk
```
