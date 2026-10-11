---
title: 在 NixOS 上进行内核开发
date: 2026-10-11 22:02:16
tags:
---

[快速指南](#我不想听讲解-赶紧告诉我怎么做)

## 写在前面

这篇文章里的方式有些(不仅仅是有些) Hack 了, 并不是很符合 "Nix way"
但是能跑就行

## 经典的问题

在常规条件下, Nix 的模型不允许我们在构建的时候使用缓存
而构建内核是十分耗时的, 现有的ccache 方案并不是非常好配置 ([nixpkgs#153343](https://github.com/NixOS/nixpkgs/issues/153343))
我几乎无法让它跑起来

有没有一种方法可以让内核构建吃上缓存呢

## Nix 只关心结果

在 NixOS wiki 中, 从自定义源构建内核使用 `buildLinux` 函数
但是 Nix 构建环境里是没有缓存的, 所以就会出现只改了一个文件我也需要构建整个内核的情况

而在外部的内核源码里手动 `make` 是可以吃到缓存的

所以解决方案就是在外部把 Nix构建产物构建出来并塞进store里
通过命令

```shell
nix-store --add-fixed sha256 "$OUTPUT_PATH" > /dev/null
```

然后我们就可以在 nix modules 里这样引用它

```nix
pkgs.fetchurl {
  # url 可以随意填写, 如果在store中命中缓存会跳过下载流程
  url = "file://${TARBALL_PATH}";
  hash = "${HASH}";
};
```

## 清除死链接

Nix 构建产物中存在 invalid symlink 会报错

```nix
  installPhase = ''
    runHook preInstall
    mkdir -p \$out
    cp bzImage \$out/bzImage
    [ -f System.map ] && cp System.map \$out/System.map
    cp config \$out/config
    cp -r lib \$out/lib
    mkdir -p \$dev/lib/modules/${KERNEL_RELEASE}
    if [ -d "\$out/lib/modules/${KERNEL_RELEASE}/build" ]; then
      mv "\$out/lib/modules/${KERNEL_RELEASE}/build" "\$dev/lib/modules/${KERNEL_RELEASE}/build"
    fi
    rm -f "\$out/lib/modules/${KERNEL_RELEASE}/build"
    rm -f "\$out/lib/modules/${KERNEL_RELEASE}/source"
    ln -sfn build "\$dev/lib/modules/${KERNEL_RELEASE}/source"
    find \$out -xtype l -delete 2>/dev/null || true
    find \$dev -xtype l -delete 2>/dev/null || true
    for bin in \
      "\$dev/lib/modules/${KERNEL_RELEASE}/build/tools/objtool/objtool" \
      "\$dev/lib/modules/${KERNEL_RELEASE}/build/tools/bpf/resolve_btfids/resolve_btfids"; do
      if [ -f "\$bin" ]; then
        patchelf --add-rpath "\${lib.makeLibraryPath [ pkgs.elfutils pkgs.zlib pkgs.zstd ]}" "\$bin" 2>/dev/null || true
      fi
    done
    runHook postInstall
  '';
```

## 解决 NVIDIA 驱动问题

Nix 在 NVIDIA driver 的构建流程中有一些额外的限制
例如构建产物不能引用**任何**外部的依赖

我们要解除这个限制, 不然构建会失败

```nix
boot = {
  kernelPackages = (pkgs.linuxPackagesFor customKernel).extend (kself: ksuper: {
      nvidiaPackages = lib.mapAttrs (name: pkg:
      if lib.isAttrs pkg then
      pkg
      // (lib.optionalAttrs (pkg ? open && lib.isDerivation pkg.open) {
      open = pkg.open.overrideAttrs (_: {
          allowedReferences = null;
          });
      })
      // (lib.optionalAttrs (pkg ? bin && lib.isDerivation pkg.bin) {
      bin = pkg.bin.overrideAttrs (_: {
          allowedReferences = null;
          });
      })
      else if lib.isDerivation pkg then
      pkg.overrideAttrs (_: {
          allowedReferences = null;
          })
      else
      pkg
      ) ksuper.nvidiaPackages;
  });
}
```

## 生成最简Kconfig

在 [`pkgs/os-specific/linux/kernel/common-config.nix`](https://github.com/NixOS/nixpkgs/blob/51b57e721e43f4dcf263534a8174e7708418e8a3/pkgs/os-specific/linux/kernel/common-config.nix#L1) 中对启用的kconfig选项有硬性要求
我们需要提取出这些必要的选项

这可以通过一个简单的脚本来实现

[GitHub Gist](https://gist.github.com/cubewhy/7c3923ff8952e60252056e1706bbdaf2)

```nix

let
  pkgs = import <nixpkgs> { };
  inherit (pkgs) lib;

  version = pkgs.linux.version;
  mandatoryOnly = true;

  arch = pkgs.stdenv.hostPlatform.linuxArch;

  callConfigFile =
    file:
    let
      fn = import file;
      expectedArgs = builtins.functionArgs fn;
      context = {
        inherit lib version;
        inherit (pkgs) stdenv;
        rustAvailable = false;
      };
    in
    fn (builtins.intersectAttrs expectedArgs context);

  commonConfigFile = pkgs.path + "/pkgs/os-specific/linux/kernel/common-config.nix";
  commonConfig = callConfigFile commonConfigFile;

  archConfigFile = pkgs.path + "/pkgs/os-specific/linux/kernel/${arch}-config.nix";
  archConfig = if builtins.pathExists archConfigFile then callConfigFile archConfigFile else { };

  merged = commonConfig // archConfig;

  isMandatory = val: val != null && !(val.optional or false);

  filtered = lib.filterAttrs (
    name: val: if mandatoryOnly then isMandatory val else val != null
  ) merged;

  formatEntry =
    name: val:
    if val ? tristate then
      if val.tristate == "y" then
        "CONFIG_${name}=y"
      else if val.tristate == "m" then
        "CONFIG_${name}=m"
      else if val.tristate == "n" then
        "# CONFIG_${name} is not set"
      else
        null
    else if val ? freeform then
      "CONFIG_${name}=${val.freeform}"
    else if val ? string then
      "CONFIG_${name}=\"${val.string}\""
    else
      null;

  lines = lib.filter (x: x != null) (lib.mapAttrsToList formatEntry filtered);

in
(lib.concatStringsSep "\n" (lib.sort (a: b: a < b) lines)) + "\n"
```

然后执行脚本, 我们可以得到必要的配置选项

```shell
nix eval --raw -f eval-required-kconfig.nix > nixos-required.config
```

接下来运行 `make localmodconfig` 生成基本配置

然后合并两者:

```shell
# 假设你处于linux源码树根目录
./scripts/kconfig/merge_config.sh .config nixos-required.config
make olddefconfig
```

### 处理 initrd

我们还需要处理 initrd, 否则我们将无法开机

现代机器通常需要启用如下选项
可以直接复制粘贴到 `.initrd.config` 中

```kconfig
CONFIG_PCI=y

CONFIG_MD=y
CONFIG_BLK_DEV_DM=y
CONFIG_DM_CRYPT=y
CONFIG_CRYPTO=y
CONFIG_CRYPTO_SKCIPHER=y
CONFIG_CRYPTO_USER_API=y
CONFIG_CRYPTO_USER_API_SKCIPHER=y
CONFIG_CRYPTO_CRYPTD=y
CONFIG_CRYPTO_CBC=y
CONFIG_CRYPTO_ECB=y
CONFIG_CRYPTO_LRW=y
CONFIG_CRYPTO_XTS=y
CONFIG_CRYPTO_AES=y
CONFIG_CRYPTO_AES_NI_INTEL=y
CONFIG_CRYPTO_BLOWFISH=y
CONFIG_CRYPTO_SERPENT=y
CONFIG_CRYPTO_TWOFISH=y
CONFIG_CRYPTO_TWOFISH_COMMON=y
CONFIG_CRYPTO_SHA1=y
CONFIG_CRYPTO_SHA256=y
CONFIG_CRYPTO_SHA512=y

CONFIG_BLK_DEV_NVME=y
CONFIG_NVME_CORE=y
CONFIG_ATA=y
CONFIG_SATA_AHCI=y
CONFIG_ATA_PIIX=y
CONFIG_PATA_MARVELL=y
CONFIG_SATA_NV=y
CONFIG_SATA_SIS=y
CONFIG_SATA_ULI=y
CONFIG_SATA_VIA=y
CONFIG_SCSI=y
CONFIG_BLK_DEV_SD=y
CONFIG_BLK_DEV_SR=y
CONFIG_MMC=y
CONFIG_MMC_BLOCK=y

CONFIG_BTRFS_FS=y
CONFIG_BTRFS_FS_POSIX_ACL=y
CONFIG_EFIVAR_FS=y
CONFIG_AUTOFS_FS=y

CONFIG_USB_SUPPORT=y
CONFIG_USB=y
CONFIG_USB_XHCI_HCD=y
CONFIG_USB_XHCI_PCI=y
CONFIG_USB_EHCI_HCD=y
CONFIG_USB_EHCI_PCI=y
CONFIG_USB_OHCI_HCD=y
CONFIG_USB_OHCI_HCD_PCI=y
CONFIG_USB_UHCI_HCD=y
CONFIG_USB_STORAGE=y

CONFIG_SERIO=y
CONFIG_SERIO_I8042=y
CONFIG_SERIO_PCIPS2=y
CONFIG_KEYBOARD_ATKBD=y
CONFIG_INPUT_LEDS=y
CONFIG_HID=y
CONFIG_HID_GENERIC=y
CONFIG_USB_HID=y
CONFIG_HID_APPLE=y
CONFIG_HID_CHERRY=y
CONFIG_HID_CORSAIR=y
CONFIG_HID_LENOVO=y
CONFIG_HID_LOGITECH=y
CONFIG_HID_LOGITECH_DJ=y
CONFIG_HID_LOGITECH_HIDPP=y
CONFIG_HID_MICROSOFT=y
CONFIG_HID_ROCCAT=y

CONFIG_TCG_TPM=y
CONFIG_TCG_CRB=y
CONFIG_TCG_TIS=y
```

然后合并

```shell
# 假设你处于linux源码树根目录
./scripts/kconfig/merge_config.sh .config .initrd.config
make olddefconfig
```

现在我们有一个可以在NixOS 上通过构建的内核配置了

太棒了, 现在我们来构建内核吧

## 构建内核

我写了一个很简单的脚本来自动化这个流程

[GitHub Gist](https://gist.github.com/cubewhy/7c3923ff8952e60252056e1706bbdaf2)

执行如下命令来构建你的内核

```shell
# 请替换第一个参数为真正的文件路径
.build-for-nix.sh /path/to/your/dotfiles/dev-kernel.nix
```

脚本会将构建产物塞进 nix store 里,
然后会在你指定的目录生成一个 `.nix` 文件和 `kernel-bin-<ver>.tar.gz` 文件 (其实这个 `.tar.gz` 是非必要的, 因为 Nix 从 store 里读取了)

接下来的工作就很简单了, 在你的 dotfiles 里 import 这个文件 (如果使用 flake 还需要 `git add`)

```nix
imports = [
  # ...
  # 请修改为真正的路径
  ./dev-kernel.nix
];
```

然后 rebuild

相信你是知道如何 rebuild 你的系统的, 故此处不列出命令

现在新内核已经安装到你的机器上了, 重启电脑就可以使用了

## 常见问题

### 我不想听讲解, 赶紧告诉我怎么做

- 下载两个脚本 并放到内核源码树根目录
  - [.build-for-nix.sh](https://gist.github.com/cubewhy/7c3923ff8952e60252056e1706bbdaf2)
  - [eval-required-kconfig.nix](https://gist.github.com/cubewhy/0be6f279a30a485f763cf6f5e25ef72d)
- 创建 `.initrd.config`, 内容见 [处理 initrd](#处理-initrd) 部分
- 获取基本Kconfig
  ```shell
  make localmodconfig
  ```
- 执行 `eval-required-kconfig.nix` 以及合并配置
  ```shell
  nix eval --raw -f eval-required-kconfig.nix > nixos-required.config
  ./scripts/kconfig/merge_config.sh .config nixos-required.config .initrd.config
  make olddefconfig
  ```
- 构建内核, 第一个参数指向 nix module 输出路径
  ```shell
  # 请替换第一个参数为真正的文件路径
  .build-for-nix.sh /path/to/your/dotfiles/dev-kernel.nix
  ```
- rebuild 你的操作系统, 重启进入新的 derivation

### NVIDIA 无法构建

如果你的内核比较新, [NVIDIA open-gpu-kernel-modules](https://github.com/NVIDIA/open-gpu-kernel-modules) 可能还没适配
你可能需要暂时禁用掉NVIDIA 相关的选项
