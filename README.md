使用Lean大代码编译，工作流使用P3TERX大模板改写；
增加安装依赖项，编译过程不报错（eg:antlr3 gperf）

name: OenWRT-Charles

on:
  push:
    branches:
      - master
    paths:
      - 'make.img'
  schedule:
    - cron: 0 0 * * *
env:
  REPO_URL: https://github.com/coolsnowwolf/lede
  REPO_BRANCH: master
  CONFIG_FILE: .config
  DIY_SH: diy.sh
  SSH_ACTIONS: false
  UPLOAD_BIN_DIR: false
  UPLOAD_FIRMWARE: true
  TZ: Asia/Shanghai

jobs:
  build:
    runs-on: ubuntu-latest
    if: github.event.repository.owner.id == github.event.sender.id

    steps:
    - name: Checkout
      uses: actions/checkout@master
      with:
        ref: master
        fetch-depth: 1000000

    - name: Initialization environment
      env:
        DEBIAN_FRONTEND: noninteractive
      run: |
        sudo swapoff /swapfile
        sudo rm -rf /swapfile /etc/apt/sources.list.d/* /usr/share/dotnet /usr/local/lib/android /opt/ghc
        sudo -E apt-get -qq update
        sudo -E apt-get -qq install build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs gcc-multilib g++-multilib p7zip p7zip-full msmtp libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint device-tree-compiler
        sudo -E apt-get -qq autoremove --purge
        sudo -E apt-get -qq clean
        curl -fsSL https://raw.githubusercontent.com/P3TERX/dotfiles/master/.bashrc >> ~/.bashrc
        
    - name:  安装依赖
      run: sudo apt-get install -y build-essential asciidoc binutils bzip2 gawk gettext git libncurses5-dev libz-dev patch unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion flex uglifyjs git-core gcc-multilib g++-multilib p7zip p7zip-full msmtp libssl-dev texinfo libreadline-dev libglib2.0-dev xmlto qemu-utils upx libelf-dev autoconf automake libtool autopoint ccache curl wget vim nano python python3 python-pip python3-pip python-ply python3-ply haveged lrzsz device-tree-compiler scons antlr3 gperf* ecj fastjar
        
    - name: Clone source code
      run: git clone --depth 1 $REPO_URL -b $REPO_BRANCH openwrt

    - name: Update feeds
      run: cd openwrt && ./scripts/feeds update -a

    - name: Install feeds
      run: cd openwrt && ./scripts/feeds install -a
    
    - name: do a config
      run: |
        cd openwrt
        make defconfig
    - name: Load custom configuration
      run: |
        [ -e files ] && mv files openwrt/files
        [ -e $CONFIG_FILE ] && mv $CONFIG_FILE openwrt/.config
        chmod +x $DIY_SH
        cd openwrt
        ../$DIY_SH
    - name: SSH connection to Actions
      uses: P3TERX/debugger-action@master
      if: env.SSH_ACTIONS == 'true'

    - name: Download package
      id: package
      run: |
        cd openwrt
        make defconfig
        make download -j8
        find dl -size -1024c -exec ls -l {} \;
        find dl -size -1024c -exec rm -f {} \;
    - name: Compile the firmware
      id: compile
      run: |
        cd openwrt
        echo -e "$(nproc) thread compile"
        make -j$(nproc) || make -j1 V=s
        echo "::set-output name=status::success"
    - name: Upload bin directory
      uses: actions/upload-artifact@master
      if: steps.compile.outputs.status == 'success' && env.UPLOAD_BIN_DIR == 'true'
      with:
        name: OpenWrt_bin
        path: openwrt/bin

    - name: Organize files
      id: organize
      if: env.UPLOAD_FIRMWARE == 'true' && !cancelled()
      run: |
        mkdir firmware && find openwrt/bin/targets/*/*/* -maxdepth 0 -name "*combined*" | xargs -i mv -f {} ./firmware/
        mv openwrt/.config  ./firmware/config.txt
        cd firmware
        echo "::set-env name=FIRMWARE::$PWD"
        echo "::set-output name=status::success"
        
    - name: Get current date
      id: date
      run: echo "::set-output name=date::$(date +'%m/%d,%Y')"
      
    - name: Upload firmware directory
      uses: actions/upload-artifact@master
      if: steps.organize.outputs.status == 'success' && !cancelled()
      with:
        name: OpenWrt_firmware
        path: ${{ env.FIRMWARE }}
        
    - name: Upload binaries to release
      uses: svenstaro/upload-release-action@v1-release
      if: steps.compile.outputs.status == 'success' && !cancelled()
      with:
        repo_token: ${{ secrets.REPO_TOKEN }}
        file: ${{ env.FIRMWARE }}/*
        tag: ${{steps.date.outputs.date}}
        overwrite: true
        file_glob: true
:
