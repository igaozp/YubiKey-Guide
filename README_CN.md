这是一份使用 [YubiKey](https://www.yubico.com/products/identifying-your-yubikey/) 作为[智能卡](https://security.stackexchange.com/questions/38924/how-does-storing-gpg-ssh-private-keys-on-smart-cards-compare-to-plain-usb-drives)进行安全加密、签名和身份验证操作的指南。

YubiKey 上的加密密钥是[不可导出的](https://web.archive.org/web/20201125172759/https://support.yubico.com/hc/en-us/articles/360016614880-Can-I-Duplicate-or-Back-Up-a-YubiKey-)，这与基于文件系统的凭证不同，同时在日常使用中保持便利性。YubiKey 可以配置为需要物理触摸才能进行加密操作，从而降低未经授权访问的风险。

- [购买 YubiKey](#购买-yubikey)
- [准备环境](#准备环境)
- [安装软件](#安装软件)
- [准备 GnuPG](#准备-gnupg)
   * [配置](#配置)
   * [身份](#身份)
   * [密钥](#密钥)
   * [过期时间](#过期时间)
   * [密码短语](#密码短语)
- [创建认证密钥](#创建认证密钥)
- [创建子密钥](#创建子密钥)
- [验证密钥](#验证密钥)
- [备份密钥](#备份密钥)
- [导出公钥](#导出公钥)
- [配置 YubiKey](#配置-yubikey)
   * [更改 PIN](#更改-pin)
   * [设置属性](#设置属性)
- [传输子密钥](#传输子密钥)
   * [签名密钥](#签名密钥)
   * [加密密钥](#加密密钥)
   * [身份验证密钥](#身份验证密钥)
- [验证传输](#验证传输)
- [完成设置](#完成设置)
- [使用 YubiKey](#使用-yubikey)
   * [加密](#加密)
   * [签名](#签名)
   * [配置触摸](#配置触摸)
   * [SSH](#ssh)
      + [替换代理](#替换代理)
      + [复制公钥](#复制公钥)
      + [导入 SSH 密钥](#导入-ssh-密钥)
      + [SSH 代理转发](#ssh-代理转发)
         - [使用 ssh-agent](#使用-ssh-agent)
         - [使用 S.gpg-agent.ssh](#使用-sgpg-agentssh)
         - [链式转发](#链式转发)
   * [GitHub](#github)
   * [GnuPG 代理转发](#gnupg-代理转发)
      + [旧版发行版](#旧版发行版)
      + [链式 GnuPG 代理转发](#链式-gnupg-代理转发)
   * [使用多个 YubiKey](#使用多个-yubikey)
   * [电子邮件](#电子邮件)
      + [Thunderbird](#thunderbird)
      + [Mailvelope](#mailvelope)
      + [Mutt](#mutt)
   * [密钥服务器](#密钥服务器)
- [更新密钥](#更新密钥)
   * [续期子密钥](#续期子密钥)
   * [轮换子密钥](#轮换子密钥)
- [重置 YubiKey](#重置-yubikey)
- [可选加固](#可选加固)
   * [改善熵](#改善熵)
   * [启用 KDF](#启用-kdf)
   * [网络注意事项](#网络注意事项)
- [注意事项](#注意事项)
- [故障排除](#故障排除)
- [替代方案](#替代方案)
- [其他资源](#其他资源)

# 购买 YubiKey

[所有 YubiKey](https://www.yubico.com/store/compare/) *除了* 仅支持 FIDO 的 Security Key 系列和 Bio 系列 YubiKey 都与本指南兼容。

通过访问 [yubico.com/genuine](https://www.yubico.com/genuine/) 来[验证 YubiKey](https://support.yubico.com/hc/en-us/articles/360013723419-How-to-Confirm-Your-Yubico-Device-is-Genuine)。选择 *Verify Device* 开始验证过程。在提示时触摸 YubiKey，并在提示时允许网站查看设备的制造商和型号。这种设备认证可能有助于缓解[供应链攻击](https://media.defcon.org/DEF%20CON%2025/DEF%20CON%2025%20presentations/DEF%20CON%2025%20-%20r00killah-and-securelyfitz-Secure-Tokin-and-Doobiekeys.pdf)。

还建议准备几个便携式存储设备（如 microSD 卡）用于存储加密备份。

# 准备环境

建议使用专用、安全的操作环境来生成加密密钥。

以下是从最不适合到最适合生成密钥材料的环境的一般排名：

1. 公共、共享或他人拥有的计算机
1. 具有无限制网络访问的日常使用个人操作系统
1. 具有有限功能的虚拟化操作系统（例如使用 [virt-manager](https://virt-manager.org/)、VirtualBox 或 VMware）
1. 专用且加固的 [Debian](https://www.debian.org/) 或 [OpenBSD](https://www.openbsd.org/) 安装
1. 在未连接主存储的情况下启动的临时 [Debian Live](https://www.debian.org/CD/live/) 或 [Tails](https://tails.boum.org/index.en.html)
1. 加固的硬件和固件（例如，[Coreboot](https://www.coreboot.org/)、[已移除 Intel ME](https://github.com/corna/me_cleaner)）
1. 无网络功能的气隙系统，最好是基于 ARM 的 Raspberry Pi 或其他架构多样化的等效设备

本指南使用 Debian Live 来平衡可用性和安全性，并附有一些针对 OpenBSD 的额外说明。

下载最新的 Debian Live 镜像和签名文件：

```console
export IMAGE_URL="https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/"

curl -fLO "$IMAGE_URL/SHA512SUMS" -O "$IMAGE_URL/SHA512SUMS.sign"

curl -fLO "$IMAGE_URL/$(awk '/xfce.iso$/ {print $2}' SHA512SUMS)"
```

下载 Debian 签名公钥：

```console
gpg --keyserver hkps://keyring.debian.org \
    --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
```

如果无法接收公钥，请使用不同的密钥服务器或 DNS 服务器：

```console
gpg --keyserver hkps://keyserver.ubuntu.com:443 \
    --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
```

Debian Live 签名公钥也可以在 [pubkeys](https://github.com/drduh/YubiKey-Guide/tree/master/pubkeys) 中导入：

```console
gpg --import pubkeys/debian-DA87E80D6294BE9B.asc
```

验证签名：

```console
gpg --verify SHA512SUMS.sign SHA512SUMS
```

输出中必须出现 `gpg: Good signature from "Debian CD signing key <debian-cd@lists.debian.org>"`。

验证镜像文件的加密哈希值与签名文件中的哈希值匹配：

```console
grep $(sha512sum debian-live-*-amd64-xfce.iso) SHA512SUMS
```

有关更多信息，请参阅[验证 Debian CD 的真实性](https://www.debian.org/CD/verify)。

连接便携式存储设备并识别磁盘标签 - 本指南全程使用 `/dev/sdc`，但此值在您的系统上可能不同：

**Linux**

```console
$ sudo dmesg | tail
usb-storage 3-2:1.0: USB Mass Storage device detected
sd 2:0:0:0: [sdc] Attached SCSI removable disk
```

将 Debian 镜像复制到设备：

```console
sudo dd if=debian-live-*-amd64-xfce.iso of=/dev/sdc bs=4M status=progress ; sync
```

**OpenBSD**

```console
$ dmesg | tail -n2
sd2 at scsibus4 targ 1 lun 0: <TS-RDF5, SD Transcend, TS3A> SCSI4 0/direct removable serial.0000000000000
sd2: 15193MB, 512 bytes/sector, 31116288 sectors

$ doas dd if=debian-live-*-amd64-xfce.iso of=/dev/rsd2c bs=4m
465+1 records in
465+1 records out
1951432704 bytes transferred in 139.125 secs (14026448 bytes/sec)
```

关闭电源，移除内部硬盘驱动器和所有不必要的设备，例如无线网卡。

# 安装软件

加载操作系统并配置网络。与网络相关的可选加固步骤可以在[下面](#network-considerations)找到。

> [!TIP]
> 如果 Debian Live 上的屏幕锁定，请使用 `user` / `live` 解锁

打开终端并安装所需的软件包。

**Debian/Ubuntu**

```console
sudo apt update

sudo apt -y upgrade

sudo apt -y install \
    wget gnupg2 gnupg-agent dirmngr \
    cryptsetup scdaemon pcscd \
    yubikey-personalization yubikey-manager
```

**OpenBSD**

```console
doas pkg_add gnupg pcsc-tools
```

**macOS**

下载并安装 [Homebrew](https://brew.sh/) 和以下软件包：

```console
brew install \
    gnupg yubikey-personalization ykman pinentry-mac wget
```

> [!NOTE]
> 可能需要安装额外的 Python 包依赖项才能使用 [`ykman`](https://support.yubico.com/support/solutions/articles/15000012643-yubikey-manager-cli-ykman-user-guide) - `pip install yubikey-manager`

或使用 [MacPorts](https://www.macports.org/install.php)，安装以下软件包：

```console
sudo port install gnupg2 yubikey-manager pinentry wget
```

**NixOS**

构建气隙 NixOS LiveCD 镜像：

```console
ref=$(git ls-remote https://github.com/drduh/Yubikey-Guide refs/heads/master | awk '{print $1}')

nix build --experimental-features "nix-command flakes" \
    github:drduh/YubiKey-Guide/$ref?dir=nix#nixosConfigurations.yubikeyLive.x86_64-linux.config.system.build.isoImage
```

如果您已经检出此仓库：

推荐但可选：更新 `nixpkgs` 和 `drduh/config`：

```console
nix flake update --commit-lock-file
```

构建镜像：

```console
nix build --experimental-features "nix-command flakes" nix#nixosConfigurations.yubikeyLive.x86_64-linux.config.system.build.isoImage
```

复制到 USB 驱动器：

```console
sudo cp -v result/iso/yubikeyLive.iso /dev/sdc ; sync
```

跳过创建临时工作目录和加固配置的步骤，因为它们已经是镜像的一部分。

使用 QEMU 等虚拟化工具测试构建。请记住，虚拟化环境提供的安全性不如临时系统（参见上面的*准备环境*）。

以下是使用上述 `nix build` 命令将 `yubikeyLive` 放置在 `result/iso` 中后的 QEMU 调用示例：

```console
# 使用 4G 内存、2 个 CPU 和启用 KVM 启动
qemu-system-x86_64 \
    -enable-kvm \
    -m 4G \
    -smp 2 \
    -drive readonly=on,media=cdrom,format=raw,file=result/iso/yubikeyLive.iso
```

**Arch**

```console
sudo pacman -Syu --needed gnupg pcsclite ccid yubikey-personalization
```

**RHEL7**

```console
sudo yum install -y gnupg2 pinentry-curses pcsc-lite pcsc-lite-libs gnupg2-smime
```

**Fedora**

```console
sudo dnf install \
    wget gnupg2 \
    cryptsetup pcsc-lite \
    yubikey-personalization-gui yubikey-manager
```

# 准备 GnuPG

创建一个临时目录，该目录将在[重启](https://en.wikipedia.org/wiki/Tmpfs)时被清除，并将其设置为 GnuPG 目录：

```console
export GNUPGHOME=$(mktemp -d -t $(date +%Y.%m.%d)-XXXX)
```

## 配置

创建或导入[加固配置](https://github.com/drduh/YubiKey-Guide/blob/master/config/gpg.conf)：

```console
cd $GNUPGHOME

wget https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/config/gpg.conf
```

这些选项看起来类似于：

```console
$ grep -v "^#" $GNUPGHOME/gpg.conf
personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
no-comments
no-emit-version
no-greeting
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
require-cross-certification
require-secmem
no-symkey-cache
armor
use-agent
throw-keyids
```

> [!IMPORTANT]
> 在剩余的设置过程中，应禁用网络。

## 身份

使用 GnuPG 创建身份时，默认选项会要求提供"真实姓名"、"电子邮件地址"和可选的"注释"。

根据您计划如何使用 GnuPG，分别设置这些值[^1]：

```console
export IDENTITY="YubiKey User <yubikey@example.domain>"
```

或使用任何能够唯一标识密钥的属性（这可能与某些用例不兼容）：

```console
export IDENTITY="My Cool YubiKey - 2025"
```

## 密钥

设置算法和密钥大小 - 推荐使用 RSA/4096：

```console
export KEY_TYPE=rsa4096
```

## 过期时间

确定所需的子密钥有效期。

设置子密钥过期时间会强制执行身份和凭证生命周期管理。然而，在认证密钥上设置过期时间是没有意义的，因为它可以用来延长自己的有效期[^2]。

本指南建议子密钥的过期时间为两年，以平衡安全性和可用性，但也可以使用更长的期限来降低维护频率。

当子密钥过期时，它们仍可用于 GnuPG 解密和 SSH 身份验证，但**不能**用于加密或签署新消息。

子密钥必须使用认证密钥进行续期或轮换 - 请参阅[更新密钥](#updating-keys)。

将子密钥设置为在计划日期过期：

```console
export EXPIRATION=2027-07-01
```

过期日期也可以是相对的，例如设置为从今天起两年：

```console
export EXPIRATION=2y
```

## 密码短语

为认证密钥生成密码短语。此凭证将用于管理身份子密钥。

为了提高可读性，本指南建议使用仅由大写字母和数字组成的密码短语。

以下命令将生成一个强[^3]密码短语，同时避免某些看起来相似的字符：

```console
export CERTIFY_PASS=$(LC_ALL=C tr -dc "A-Z2-9" < /dev/urandom | \
    tr -d "IOUS5" | \
    fold  -w  ${PASS_GROUPSIZE:-4} | \
    paste -sd ${PASS_DELIMITER:--} - | \
    head  -c  ${PASS_LENGTH:-29})
printf "\n$CERTIFY_PASS\n\n"
```

要更改密码短语长度、分隔符或组大小，请在运行密码短语生成命令之前导出相应的变量，例如：

```console
export PASS_GROUPSIZE=6
export PASS_DELIMITER=+
export PASS_LENGTH=48
```

将密码短语写在安全的地方 - 与用于密钥材料的便携式存储设备分开，或记住它。

此仓库包含一个 [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.html) 模板来帮助凭证转录。保存[原始文件](https://github.com/drduh/YubiKey-Guide/raw/refs/heads/master/templates/passphrase.html)，在浏览器中打开以渲染和打印。

在连续的行上标记密码短语中每个字符对应的字符。[`passphrase.txt`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.txt) 也可以在没有浏览器的情况下打印：

```console
lp -d Printer-Name passphrase.txt
```

[Diceware](https://secure.research.vt.edu/diceware) 是创建易记密码短语的另一种流行方法。

# 创建认证密钥

要生成的主密钥是认证密钥，它负责颁发用于加密、签名和身份验证操作的子密钥。

认证密钥应始终保持离线状态，并且只能从专用且安全的环境中访问以颁发或撤销子密钥。

不要在认证密钥上设置过期日期。

生成认证密钥：

```console
echo "$CERTIFY_PASS" | \
    gpg --batch --passphrase-fd 0 \
        --quick-generate-key "$IDENTITY" "$KEY_TYPE" cert never
```

设置并查看认证密钥标识符和指纹以供以后使用：

```console
export KEYID=$(gpg -k --with-colons "$IDENTITY" | \
    awk -F: '/^pub:/ { print $5; exit }')

export KEYFP=$(gpg -k --with-colons "$IDENTITY" | \
    awk -F: '/^fpr:/ { print $10; exit }')

printf "\nKey ID/Fingerprint: %20s\n%s\n\n" "$KEYID" "$KEYFP"
```

<details>
<summary>添加额外的 ID（可选）</summary>

这是需要[额外身份](https://github.com/drduh/YubiKey-Guide/issues/445)的用例的可选步骤，例如：

- 不同语言的不同电子邮件地址
- 专业与个人的不同电子邮件地址，但请参阅下面关于不将这些地址绑定在一起的替代原因
- 不同 git 提供商的匿名电子邮件地址

另一种方法是拥有不同的密钥，但这样您就需要多个 YubiKey，因为每个 YubiKey 只能为每种类型（签名、加密、身份验证）保存一个密钥。尽管如此，拥有多个 YubiKey 还是有很好的理由的：

- 如果您有用于专业和个人用例的不同电子邮件地址，拥有不同的密钥可以让您取消身份关联
- 如果您还将 YubiKey 用作 U2F 或 FIDO2 设备，通常建议拥有多个 YubiKey 作为备份措施

定义一个包含额外用户 ID 的数组。每个数组元素必须用引号括起来，每个元素必须用空格分隔：

```console
declare -a additional_uids
additional_uids=("Super Cool YubiKey 2025" "uid 1 <uid1@example.org>")
```

将额外的用户 ID 添加到认证密钥：

```console
for uid in "${additional_uids[@]}" ; do \
    echo "$CERTIFY_PASS" | \
    gpg --batch --passphrase-fd 0 \
        --pinentry-mode=loopback --quick-add-uid "$KEYFP" "$uid"
done
```

将额外 ID 的信任度调整为*绝对*：

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key "$KEYID" <<EOF
uid *
trust
5
y
save
EOF
```
</details>

# 创建子密钥

使用先前配置的密钥类型、密码短语和过期时间生成签名和加密子密钥：

```console
echo "$CERTIFY_PASS" | \
    gpg --batch --pinentry-mode=loopback --passphrase-fd 0 \
        --quick-add-key "$KEYFP" "$KEY_TYPE" sign "$EXPIRATION"

echo "$CERTIFY_PASS" | \
    gpg --batch --pinentry-mode=loopback --passphrase-fd 0 \
        --quick-add-key "$KEYFP" "$KEY_TYPE" encrypt "$EXPIRATION"
```

然后是身份验证子密钥：

> [!NOTE]
> 某些系统不再接受 RSA 用于 SSH 身份验证；要使用 [Ed25519](https://ed25519.cr.yp.to/)，请在生成身份验证子密钥之前将 `KEY_TYPE` 变量设置为 `ed25519`。

```
echo "$CERTIFY_PASS" | \
    gpg --batch --pinentry-mode=loopback --passphrase-fd 0 \
        --quick-add-key "$KEYFP" "$KEY_TYPE" auth "$EXPIRATION"
```

# 验证密钥

列出可用的私钥：

```console
gpg -K
```

输出将显示**[C]认证、[S]签名、[E]加密和 [A]身份验证**密钥：

```console
sec   rsa4096/0xF0F2CFEB04341FB5 2025-07-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example>
ssb   rsa4096/0xB3CD10E502E19637 2025-07-01 [S] [expires: 2027-07-01]
ssb   rsa4096/0x30CBE8C4B085B9F7 2025-07-01 [E] [expires: 2027-07-01]
ssb   rsa4096/0xAD9E24E1B8CB9600 2025-07-01 [A] [expires: 2027-07-01]
```

# 备份密钥

保存认证密钥、子密钥和公钥的副本：

```console
echo "$CERTIFY_PASS" | \
    gpg --output $GNUPGHOME/$KEYID-Certify.key \
        --batch --pinentry-mode=loopback --passphrase-fd 0 \
        --armor --export-secret-keys $KEYID

echo "$CERTIFY_PASS" | \
    gpg --output $GNUPGHOME/$KEYID-Subkeys.key \
        --batch --pinentry-mode=loopback --passphrase-fd 0 \
        --armor --export-secret-subkeys $KEYID

gpg --output $GNUPGHOME/$KEYID-$(date +%F).asc \
    --armor --export $KEYID
```

在便携式存储设备上创建**加密**备份，以离线保存在安全且耐用的位置。

建议在多个便携式存储设备上多次重复以下过程，因为它们可能会随着时间的推移而失效。作为额外的备份措施，可以使用 [Paperkey](https://www.jabberwocky.com/software/paperkey/) 制作密钥材料的物理副本以提高耐用性。

> [!TIP]
> [ext2](https://en.wikipedia.org/wiki/Ext2) 卷（不加密）可以在 Linux 和 OpenBSD 上挂载。
> 使用 [FAT32](https://en.wikipedia.org/wiki/Fat32) 或 [NTFS](https://en.wikipedia.org/wiki/Ntfs) 卷来实现 macOS 和 Windows 兼容性。

**Linux**

连接便携式存储设备并检查其标签，在本例中为 `/dev/sdc`：

```console
$ sudo dmesg | tail
usb-storage 3-2:1.0: USB Mass Storage device detected
sd 2:0:0:0: [sdc] Attached SCSI removable disk

$ sudo fdisk -l /dev/sdc
Disk /dev/sdc: 14.9 GiB, 15931539456 bytes, 31116288 sectors
```

> [!CAUTION]
> 在发出以下命令之前确认目标（`of`）- 它是破坏性的！本指南全程使用 `/dev/sdc`，但此值在您的系统上可能不同。

将头部清零以准备加密：

```console
sudo dd if=/dev/zero of=/dev/sdc bs=4M count=1
```

移除并重新连接存储设备。

擦除并创建新的分区表：

```console
sudo fdisk /dev/sdc <<EOF
g
w
EOF
```

创建一个小分区（建议至少 20 Mb 以考虑 LUKS 头大小）用于存储密钥材料：

```console
sudo fdisk /dev/sdc <<EOF
n


+20M
w
EOF
```

使用 [LUKS](https://dys2p.com/en/2023-05-luks-security.html) 加密新分区。

生成另一个唯一的[密码短语](#passphrase)（理想情况下与用于认证密钥的密码短语不同）以保护加密卷：

```console
export LUKS_PASS=$(LC_ALL=C tr -dc "A-Z2-9" < /dev/urandom | \
    tr -d "IOUS5" | \
    fold  -w  ${PASS_GROUPSIZE:-4} | \
    paste -sd ${PASS_DELIMITER:--} - | \
    head  -c  ${PASS_LENGTH:-29})
printf "\n$LUKS_PASS\n\n"
```

此密码短语也将不频繁地用于访问认证密钥，应该非常强。

写下密码短语或记住它。

格式化分区：

```console
echo $LUKS_PASS | \
    sudo cryptsetup -q luksFormat /dev/sdc1
```

挂载分区：

```console
echo $LUKS_PASS | \
    sudo cryptsetup -q luksOpen /dev/sdc1 gnupg-secrets
```

创建 ext2 文件系统：

```console
sudo mkfs.ext2 /dev/mapper/gnupg-secrets -L gnupg-$(date +%F)
```

挂载文件系统并复制带有密钥材料的临时 GnuPG 工作目录：

```console
sudo mkdir -p /mnt/encrypted-storage

sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage

sudo cp -av $GNUPGHOME /mnt/encrypted-storage/
```

卸载并关闭加密卷：

```console
sudo umount /mnt/encrypted-storage

sudo cryptsetup luksClose gnupg-secrets
```

对任何其他存储设备重复该过程（建议至少两个）。

**OpenBSD**

连接 USB 磁盘并确定其标签：

```console
$ dmesg | grep sd.\ at
sd2 at scsibus5 targ 1 lun 0: <TS-RDF5, SD Transcend, TS37> SCSI4 0/direct removable serial.00000000000000000000
```

打印现有分区以确保这是正确的设备：

```console
doas disklabel -h sd2
```

通过创建 FS 类型为 `RAID` 且大小为 25 兆字节的 `a` 分区来初始化磁盘：

```console
$ doas fdisk -giy sd2
Writing MBR at offset 0.
Writing GPT.

$ doas disklabel -E sd2
Label editor (enter '?' for help at any prompt)
sd2> a a
offset: [64]
size: [31101776] 25M
FS type: [4.2BSD] RAID
sd2*> w
sd2> q
No label changes
```

使用唯一的[密码短语](#passphrase)通过 bioctl 进行加密：

```console
$ doas bioctl -c C -l sd2a softraid0
New passphrase:
Re-type passphrase:
softraid0: CRYPTO volume attached as sd3
```

在新的加密卷上创建 `i` 分区和文件系统：

```console
$ doas fdisk -giy sd3
Writing MBR at offset 0.
Writing GPT.

$ doas disklabel -E sd3
Label editor (enter '?' for help at any prompt)
sd3> a i
offset: [64]
size: [16001]
FS type: [4.2BSD]
sd3*> w
sd3> q
No label changes.

$ doas newfs sd3i
```

挂载文件系统并将临时目录与密钥环复制到其中：

```console
doas mkdir -p /mnt/encrypted-storage

doas mount /dev/sd3i /mnt/encrypted-storage

doas cp -av $GNUPGHOME /mnt/encrypted-storage
```

卸载并移除加密卷：

```console
doas umount /mnt/encrypted-storage

doas bioctl -d sd3
```

更多信息请参阅 [OpenBSD FAQ#14](https://www.openbsd.org/faq/faq14.html#softraidCrypto)。

# 导出公钥

> [!IMPORTANT]
> 如果没有公钥，将**无法**使用 GnuPG 解密/签名消息。但是，YubiKey 仍然可以用于 SSH 身份验证。

连接另一个便携式存储设备或在现有设备上创建一个新分区。

**Linux**

使用与上一步相同的 `/dev/sdc` 设备，创建一个小分区（建议至少 20 Mb）用于存储材料：

```console
sudo fdisk /dev/sdc <<EOF
n


+20M
w
EOF
```

创建文件系统并导出公钥：

```console
sudo mkfs.ext2 /dev/sdc2

sudo mkdir -p /mnt/public

sudo mount /dev/sdc2 /mnt/public

gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc

sudo chmod 0444 /mnt/public/*.asc
```

卸载并移除存储设备：

```console
sudo umount /mnt/public
```

**OpenBSD**

```console
$ doas disklabel -E sd2
Label editor (enter '?' for help at any prompt)
sd2> a b
offset: [32130]
size: [31069710] 25M
FS type: [swap] 4.2BSD
sd2*> w
sd2> q
No label changes.
```

创建文件系统并将公钥导出到其中：

```console
doas newfs sd2b

doas mkdir -p /mnt/public

doas mount /dev/sd2b /mnt/public

gpg --armor --export $KEYID | doas tee /mnt/public/$KEYID-$(date +%F).asc
```

卸载并移除存储设备：

```console
doas umount /mnt/public
```

# 配置 YubiKey

连接 YubiKey 并确认其状态：

```console
gpg --card-status
```

如果 YubiKey 被锁定，请[重置](#reset-yubikey)它。

## 更改 PIN

YubiKey 的 [PGP](https://developers.yubico.com/PGP/) 接口有其自己的 PIN，与其他模块（如 [PIV](https://developers.yubico.com/PIV/Introduction/YubiKey_and_PIV.html)）的 PIN 是分开的：

名称 | 默认值 | 功能
:---: | :---: | ---
用户 PIN | `123456` | 加密操作（解密、签名、身份验证）
管理员 PIN | `12345678` | 重置 PIN、更改重置代码、添加密钥和所有者信息
重置代码 | 无 | 重置 PIN（[更多信息](https://forum.yubico.com/viewtopicd01c.html?p=9055#p9055)）

确定所需的 PIN 值。由于暴力破解机会有限，它们可以比认证密钥密码短；用户 PIN 应该足够方便，以便日常使用时能够记住。

*用户 PIN* 至少需要 6 个字符，*管理员 PIN* 至少需要 8 个字符。最多允许 127 个 ASCII 字符。更多信息请参阅[管理 PIN](https://www.gnupg.org/howtos/card-howto/en/ch03s02.html)。

设置 PIN 值，例如 6 位数的用户 PIN 和 8 位数的管理员 PIN：

```console
export ADMIN_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | \
    fold -w8 | head -1)

export USER_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | \
    fold -w6 | head -1)

printf "\nAdmin PIN: %12s\nUser PIN: %13s\n\n" \
    "$ADMIN_PIN" "$USER_PIN"
```

更改管理员 PIN：

```console
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
3
12345678
$ADMIN_PIN
$ADMIN_PIN
q
EOF
```

更改用户 PIN：

```console
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
1
123456
$USER_PIN
$USER_PIN
q
EOF
```

移除并重新插入 YubiKey。

> [!CAUTION]
> 三次错误的*用户 PIN* 输入将导致其被锁定，必须使用*管理员 PIN* 或*重置代码*解锁。三次错误的*管理员 PIN* 或*重置代码*输入将销毁 YubiKey 上的数据。

可以更改[重试次数](https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html#ykman-openpgp-access-set-retries-options-pin-retries-reset-code-retries-admin-pin-retries)，例如设置为 5 次尝试：

```console
ykman openpgp access set-retries 5 5 5 -f -a $ADMIN_PIN
```

## 设置属性

使用之前设置的值：

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-card <<EOF
admin
login
$IDENTITY
$ADMIN_PIN
quit
EOF
```

[智能卡属性](https://gnupg.org/howtos/card-howto/en/smartcard-howto-single.html)也可以使用 `gpg --edit-card` 和 `admin` 模式设置。使用 `help` 查看可用选项。`login` 属性是[必需的](https://github.com/drduh/YubiKey-Guide/issues/461)。

运行 `gpg --card-status` 验证结果（*Login data* 字段）。

# 传输子密钥

> [!IMPORTANT]
> 将密钥传输到 YubiKey 会将磁盘上的密钥转换为"存根" - 使其无法再传输到后续的 YubiKey。在继续之前，请确保已备份密钥。

传输密钥需要认证密钥密码和管理员 PIN。

## 签名密钥

传输签名密钥：

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 1
keytocard
1
$CERTIFY_PASS
$ADMIN_PIN
save
EOF
```

## 加密密钥

对加密密钥重复此过程：

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 2
keytocard
2
$CERTIFY_PASS
$ADMIN_PIN
save
EOF
```

## 身份验证密钥

对身份验证密钥重复此过程：

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 3
keytocard
3
$CERTIFY_PASS
$ADMIN_PIN
save
EOF
```

# 验证传输

使用 `gpg -K` 验证子密钥在 YubiKey 上 - 由 `ssb>` 表示：

```console
sec   rsa4096/0xF0F2CFEB04341FB5 2025-07-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example>
ssb>  rsa4096/0xB3CD10E502E19637 2025-07-01 [S] [expires: 2027-07-01]
ssb>  rsa4096/0x30CBE8C4B085B9F7 2025-07-01 [E] [expires: 2027-07-01]
ssb>  rsa4096/0xAD9E24E1B8CB9600 2025-07-01 [A] [expires: 2027-07-01]
```

标签后的 `>` 表示密钥存储在智能卡上。

# 完成设置

验证是否正确执行了以下步骤：

- [ ] 将认证密钥（身份）密码记住或写在安全且耐用的位置
  * 使用 `echo $CERTIFY_PASS` 再次查看；使用 [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.html) 或 [`passphrase.txt`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.txt) 进行转录
- [ ] 将便携式存储上加密卷的密码记住或写下
  * 使用 `echo $LUKS_PASS` 再次查看；使用 [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.html) 或 [`passphrase.txt`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.txt) 进行转录
- [ ] 将认证密钥和子密钥保存到加密的便携式存储中，离线保存
  * 建议至少进行两次备份，存储在不同的位置
- [ ] 导出公钥副本，以便稍后可以轻松访问
  * 使用了单独的设备或非加密分区
- [ ] 将用户 PIN 和管理员 PIN 记住或写下，它们是唯一的且已从默认值更改
  * 使用 `echo $USER_PIN $ADMIN_PIN` 再次查看；使用 [`passphrase.html`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.html) 或 [`passphrase.txt`](https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/templates/passphrase.txt) 进行转录
- [ ] 已将加密、签名和身份验证子密钥移动到 YubiKey
  * `gpg -K` 显示 3 个子密钥中的每一个都为 `ssb>`

重启，清除临时环境，以完成设置。

# 使用 YubiKey

初始化 GnuPG：

```console
gpg -k
```

创建或导入[加固配置](https://github.com/drduh/YubiKey-Guide/blob/master/config/gpg.conf)：

```console
cd ~/.gnupg

wget https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/config/gpg.conf
```

设置以下选项。这可以避免 GnuPG 重复提示插入已插入的 YubiKey 的问题：

```console
touch scdaemon.conf

echo "disable-ccid" >>scdaemon.conf
```

安装所需的软件包：

**Debian/Ubuntu**

```console
sudo apt update

sudo apt install -y gnupg gnupg-agent scdaemon pcscd
```

**Arch**

```console
sudo pacman -S --needed gnupg pcsc-tools

sudo systemctl enable --now pcscd.service
```

**macOS**

```console
brew install gnupg
```

或使用 MacPorts

```console
sudo port install gnupg2 pcsc-tools
```

**OpenBSD**

```console
doas pkg_add gnupg pcsc-tools

doas rcctl enable pcscd

doas reboot
```

挂载包含公钥的非加密卷：

**Debian/Ubuntu**

```console
sudo mkdir -p /mnt/public

sudo mount /dev/sdc2 /mnt/public
```

**OpenBSD**

```console
doas mkdir -p /mnt/public

doas mount /dev/sd3i /mnt/public
```

导入公钥：

```console
gpg --import /mnt/public/*.asc
```

或从密钥服务器下载公钥：

```console
gpg --recv $KEYID
```

或者使用 YubiKey 上的 URL，使用命令 `gpg --edit-card` 检索公钥。

```console
gpg/card> fetch

gpg/card> quit
```

确定密钥 ID：

```console
gpg -k

export KEYID=0xF0F2CFEB04341FB5
```

通过输入 `trust` 并选择选项 `5` 然后 `quit` 来分配终极信任：

```console
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
trust
5
y
save
EOF
```

移除并重新插入 YubiKey。

使用 `gpg --card-status` 验证状态，它将列出可用的子密钥：

```console
Reader ...........: Yubico YubiKey OTP FIDO CCID 00 00
Application ID ...: D2760001240102010006055532110000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 05553211
Name of cardholder: YubiKey User
Language prefs ...: en
Salutation .......:
URL of public key : [not set]
Login data .......: yubikey@example
Signature PIN ....: not forced
Key attributes ...: rsa4096 rsa4096 rsa4096
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
KDF setting ......: on
Signature key ....: CF5A 305B 808B 7A0F 230D  A064 B3CD 10E5 02E1 9637
      created ....: 2025-07-01 12:00:00
Encryption key....: A5FA A005 5BED 4DC9 889D  38BC 30CB E8C4 B085 B9F7
      created ....: 2025-07-01 12:00:00
Authentication key: 570E 1355 6D01 4C04 8B6D  E2A3 AD9E 24E1 B8CB 9600
      created ....: 2025-07-01 12:00:00
General key info..: sub  rsa4096/0xB3CD10E502E19637 2025-07-01 YubiKey User <yubikey@example>
sec#  rsa4096/0xF0F2CFEB04341FB5  created: 2025-07-01  expires: never
ssb>  rsa4096/0xB3CD10E502E19637  created: 2025-07-01  expires: 2027-07-01
                                  card-no: 0006 05553211
ssb>  rsa4096/0x30CBE8C4B085B9F7  created: 2025-07-01  expires: 2027-07-01
                                  card-no: 0006 05553211
ssb>  rsa4096/0xAD9E24E1B8CB9600  created: 2025-07-01  expires: 2027-07-01
                                  card-no: 0006 05553211
```

`sec#` 表示相应的密钥不可用（认证密钥处于离线状态）。

YubiKey 现在可以使用了！

## 加密

向自己加密消息（用于存储凭据或保护备份）：

```console
echo -e "\ntest message string" | \
    gpg --encrypt --armor \
        --recipient $KEYID --output encrypted.txt
```

解密消息 - 将出现用户 PIN 的提示：

```console
gpg --decrypt --armor encrypted.txt
```

要加密到多个收件人/密钥，请将首选密钥 ID 放在最后：

```console
echo "test message string" | \
    gpg --encrypt --armor \
        --recipient $KEYID_2 --recipient $KEYID_1 --recipient $KEYID \
        --output encrypted.txt
```

使用 [shell 函数](https://github.com/drduh/config/blob/main/zshrc)使加密文件更容易：

```console
secret () {
  output="${1}".$(date +%s).enc
  gpg --encrypt --armor --output ${output} \
    -r $KEYID "${1}" && echo "${1} -> ${output}"
}

reveal () {
  output=$(echo "${1}" | rev | cut -c16- | rev)
  gpg --decrypt --output ${output} "${1}" && \
    echo "${1} -> ${output}"
}
```

示例输出：

```console
$ secret document.pdf
document.pdf -> document.pdf.1580000000.enc

$ reveal document.pdf.1580000000.enc
gpg: anonymous recipient; trying secret key 0xF0F2CFEB04341FB5 ...
gpg: okay, we are the anonymous recipient.
gpg: encrypted with RSA key, ID 0x0000000000000000
document.pdf.1580000000.enc -> document.pdf
```

[drduh/Purse](https://github.com/drduh/Purse) 是一个基于 GnuPG 和 YubiKey 的密码管理器，用于安全存储和使用凭据。

## 签名

签名消息：

```console
echo "test message string" | gpg --armor --clearsign > signed.txt
```

验证签名：

```console
gpg --verify signed.txt
```

输出将类似于：

```console
gpg: Signature made Mon 01 Jan 2025 12:00:00 PM UTC
gpg:                using RSA key CF5A305B808B7A0F230DA064B3CD10E502E19637
gpg: Good signature from "YubiKey User <yubikey@example>" [ultimate]
Primary key fingerprint: 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
     Subkey fingerprint: CF5A 305B 808B 7A0F 230D  A064 B3CD 10E5 02E1 9637
```

## 配置触摸

默认情况下，YubiKey 在使用 PIN 解锁密钥一次后，将在不需要用户任何操作的情况下执行加密操作。

要为每个密钥操作要求触摸，请使用 [YubiKey Manager](https://developers.yubico.com/yubikey-manager/) 和管理员 PIN 设置密钥策略。

加密：

```console
ykman openpgp keys set-touch dec on
```

> [!NOTE]
> 5.1.0 之前版本的 YubiKey Manager 使用 `enc` 而不是 `dec` 进行加密：

```console
ykman openpgp keys set-touch enc on
```

更旧版本的 YubiKey Manager 使用 `touch` 而不是 `set-touch`

签名：

```console
ykman openpgp keys set-touch sig on
```

身份验证：

```console
ykman openpgp keys set-touch aut on
```

要查看和调整策略选项：

```console
ykman openpgp keys set-touch -h
```

对于与电子邮件客户端一起使用 YubiKey，`Cached` 或 `Cached-Fixed` 可能是理想的选择。

当 YubiKey 等待触摸时会闪烁。在 Linux 上，可以使用 [maximbaz/yubikey-touch-detector](https://github.com/maximbaz/yubikey-touch-detector) 来指示 YubiKey 正在等待触摸。

## SSH

创建或导入[加固配置](https://github.com/drduh/YubiKey-Guide/blob/master/config/gpg-agent.conf)：

```console
cd ~/.gnupg

wget https://raw.githubusercontent.com/drduh/YubiKey-Guide/master/config/gpg-agent.conf
```

> [!NOTE]
> 当使用 YubiKey 作为智能卡时，`cache-ttl` 选项**不**适用，因为 PIN 由[智能卡本身缓存](https://dev.gnupg.org/T3362)。要从缓存中清除 PIN（相当于 `default-cache-ttl` 和 `max-cache-ttl`），请移除 YubiKey，或在编辑卡时设置 `forcesig` 以在每次提示输入 PIN。

> [!TIP]
> 将 `pinentry-program` 设置为 `/usr/bin/pinentry-gnome3` 以获得基于 GUI 的提示。

**macOS**

使用 `brew install pinentry-mac` 或 `sudo port install pinentry` 安装 pinentry，然后编辑 `gpg-agent.conf` 将 `pinentry-program` 路径设置为：

* Apple Silicon Macs：`/opt/homebrew/bin/pinentry-mac`
* Intel Macs：`/usr/local/bin/pinentry-mac` 或 `/opt/local/bin/pinentry`（MacPorts）
* MacGPG Suite：`/usr/local/MacGPG2/libexec/pinentry-mac.app/Contents/MacOS/pinentry-mac`

然后运行 `gpgconf --kill gpg-agent` 使更改生效。

要在 macOS 上使用图形应用程序，[需要额外的设置](https://jms1.net/yubikey/make-ssh-use-gpg-agent.md)。

创建 `$HOME/Library/LaunchAgents/gnupg.gpg-agent.plist`，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>gnupg.gpg-agent</string>
        <key>RunAtLoad</key>
        <true/>
        <key>KeepAlive</key>
        <false/>
        <key>ProgramArguments</key>
        <array>
            <string>/usr/local/MacGPG2/bin/gpg-connect-agent</string>
            <string>/bye</string>
        </array>
    </dict>
</plist>
```

加载它：

```console
launchctl load $HOME/Library/LaunchAgents/gnupg.gpg-agent.plist
```

创建 `$HOME/Library/LaunchAgents/gnupg.gpg-agent-symlink.plist`，内容如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/ProperyList-1.0/dtd">
<plist version="1.0">
    <dict>
        <key>Label</key>
        <string>gnupg.gpg-agent-symlink</string>
        <key>ProgramArguments</key>
        <array>
            <string>/bin/sh</string>
            <string>-c</string>
            <string>/bin/ln -sf $HOME/.gnupg/S.gpg-agent.ssh $SSH_AUTH_SOCK</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
    </dict>
</plist>
```

加载它：

```console
launchctl load $HOME/Library/LaunchAgents/gnupg.gpg-agent-symlink.plist
```

重启以激活更改。

**Windows**

Windows 可能已经安装了一些虚拟智能卡读卡器，例如为 Windows Hello 提供的读卡器。要验证 YubiKey 是 scdaemon 使用的正确读卡器，请将其添加到其配置中。

使用 PowerShell 查找 YubiKey 标签：

```powershell
PS C:\WINDOWS\system32> Get-PnpDevice -Class SoftwareDevice | Where-Object {$_.FriendlyName -like "*YubiKey*"} | Select-Object -ExpandProperty FriendlyName
Yubico YubiKey OTP+FIDO+CCID 0
```

更多信息请参阅[如何在 Windows 上使用 YubiKey NEO、GPG 和 Keybase 设置签名的 Git 提交（2018）](https://www.hanselman.com/blog/HowToSetupSignedGitCommitsWithAYubiKeyNEOAndGPGAndKeybaseOnWindows.aspx)。

编辑 `%APPDATA%/gnupg/scdaemon.conf` 添加：

```console
reader-port <device name, e.g. Yubico YubiKey OTP+FIDO+CCID 0>
```

编辑 `%APPDATA%/gnupg/gpg-agent.conf` 添加：

```console
enable-ssh-support
enable-putty-support
```

重启代理：

```console
gpg-connect-agent killagent /bye

gpg-connect-agent /bye
```

验证 YubiKey 详细信息：

```console
gpg --card-status
```

导入公钥并设置终极信任：

```console
gpg --import <公钥文件路径>
```

检索公钥 ID:

```console
gpg --list-public-keys
```

导出 SSH 公钥:

```console
gpg --export-ssh-key <公钥 ID>
```

将公钥复制到文件 - 它对应于 YubiKey 上的私钥,可以复制到 SSH 目标主机。

创建一个指向 `gpg-connect-agent /bye` 的快捷方式,并将其放在启动文件夹 `shell:startup` 中,以确保代理在重启后启动。修改快捷方式属性,使其在"最小化"窗口中启动。

PuTTY 现在可以用于公钥 SSH 身份验证。当服务器要求公钥验证时,PuTTY 将把请求转发给 GnuPG,后者将提示输入 PIN 以授权操作。

**WSL**

目标是配置 WSL 内的 SSH 客户端与 Windows 代理(如 gpg-agent.exe)协同工作。

有关概述,请参阅 [WSL 代理架构](media/schema_gpg.png) 图示。

不支持用于加密操作的 GnuPG 转发。有关更多信息,请参阅 [vuori/weasel-pageant](https://github.com/vuori/weasel-pageant)。

一种转发方式是使用 `ssh -A`(仍需要执行 weasel 来设置本地 ssh-agent),并且仅依赖于 OpenSSH。在这种方式下,ssh/sshd 配置中的 `ForwardAgent` 和 `AllowAgentForwarding` 可能会涉及。但是,当使用 ssh 套接字转发时,请不要在 ssh 配置中启用 `ForwardAgent`。有关更多信息,请参阅 [SSH 代理转发](#ssh-agent-forwarding)。这需要 WSL 使用 Ubuntu 16.04 或更新版本以及 Kleopatra。

下载 [vuori/weasel-pageant](https://github.com/vuori/weasel-pageant)。

将 `eval $(/mnt/c/<提取路径>/weasel-pageant -r -a /tmp/S.weasel-pageant)` 添加到 shell rc 文件中。这里使用命名套接字,以便可以在 `~/.ssh/config` 的 `RemoteForward` 指令中使用它。使用 `source ~/.bashrc` 来源它。

使用 `$ ssh-add -l` 显示 SSH 密钥

编辑 `~/.ssh/config` 为每个代理转发主机添加以下内容:

```console
RemoteForward <远程 SSH 套接字路径> /tmp/S.weasel-pageant
```

可以使用 `gpgconf --list-dirs agent-ssh-socket` 找到远程 SSH 套接字路径

将以下内容添加到 shell rc 文件中:

```console
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
```

将以下内容添加到 `/etc/ssh/sshd_config`:

```console
StreamLocalBindUnlink yes
```

重新加载 SSH 守护进程:

```console
sudo service sshd reload
```

移除 YubiKey 并重启。重新登录 Windows,打开 WSL 控制台并输入 `ssh-add -l` - 不应显示任何输出。

插入 YubiKey,输入相同的命令以显示 ssh 密钥。

连接到远程主机并使用 `ssh-add -l` 确认转发正常工作。

代理转发可以通过多个主机链接。按照相同的[协议](#remote-host-configuration)配置每个主机。

另一种方法是使用 [usbipd-win](https://github.com/dorssel/usbipd-win) 库。如果在配置 usbipd-win 后在 WSL 中访问 YubiKey 时遇到问题,您可能需要添加自定义 polkit 规则以确保 pcscd 服务的正确权限。以下是使用 scard 组的示例配置(组逻辑是可选的):

在 /etc/polkit-1/rules.d/99-pcscd.rules 创建新的规则文件:

```bash
polkit.addRule(function(action, subject) {
    if (action.id == "org.debian.pcsc-lite.access_card" &&
        subject.isInGroup("scard")) {
        return polkit.Result.YES;
    }
});

polkit.addRule(function(action, subject) {
    if (action.id == "org.debian.pcsc-lite.access_pcsc" &&
        subject.isInGroup("scard")) {
        return polkit.Result.YES;
    }
});
```

### 替换代理

要启动 `gpg-agent` 供 SSH 使用,请使用 `gpg-connect-agent /bye` 或 `gpgconf --launch gpg-agent` 命令。

将以下内容添加到 shell rc 文件中:

```console
export GPG_TTY=$(tty)
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
gpg-connect-agent updatestartuptty /bye > /dev/null
```

对于 fish,`config.fish` 应该如下所示(考虑将它们放入 `is-interactive` 块中):

```fish
set -x GPG_TTY (tty)
set -x SSH_AUTH_SOCK (gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

使用 `ForwardAgent` 进行 ssh-agent 转发时,`SSH_AUTH_SOCK` 只需在连接了 YubiKey 的*本地*主机上设置。在*远程*主机上,`ssh` 会在连接时将 `SSH_AUTH_SOCK` 设置为类似 `/tmp/ssh-mXzCzYT2Np/agent.7541` 的内容。**不要**在远程主机上设置 `SSH_AUTH_SOCK` - 这样做会破坏 [SSH 代理转发](#ssh-agent-forwarding)。

对于 `S.gpg-agent.ssh`(有关更多信息,请参阅 [SSH 代理转发](#ssh-agent-forwarding)),`SSH_AUTH_SOCK` 也应在*远程*主机上设置。但是,`GPG_TTY` 不应在*远程*主机上设置,该部分中指定了说明。

### 复制公钥

> [!TIP]
> 如果仅使用 SSH,则**不**需要导入 GnuPG 公钥。

将 `ssh-add` 的输出复制并粘贴到服务器的 `authorized_keys` 文件:

```console
$ ssh-add -L
ssh-rsa AAAAB4NzaC1yc2EAAAADAQABAAACAz[...]zreOKM+HwpkHzcy9DQcVG2Nw== cardno:000605553211
```

**可选** 保存公钥以进行身份文件配置。默认情况下,SSH 会尝试使用代理提供的所有身份。通常最好精确管理 SSH 将使用哪些密钥连接到服务器,例如分离不同的角色或[避免被不受信任的 ssh 服务器指纹识别](https://words.filippo.io/ssh-whoami-filippo-io/)。要做到这一点,您需要使用命令行参数 `-i [identity_file]` 或 `.ssh/config` 中的 `IdentityFile` 和 `IdentitiesOnly` 选项。

传统上,提供给 `IdentityFile` 的参数是_私钥_文件的路径(例如 `IdentityFile ~/.ssh/id_rsa`)。对于 YubiKey,`IdentityFile` 必须指向_公钥_文件,`ssh` 将从 ssh-agent 提供的密钥中选择适当的私钥。要防止 `ssh` 尝试代理中的所有密钥,请使用 `IdentitiesOnly yes` 以及一个或多个针对目标主机的 `-i` 或 `IdentityFile` 选项。

重申一下,使用 `IdentitiesOnly yes` 时,`ssh` 不会枚举加载到 `ssh-agent` 或 `gpg-agent` 中的公钥。这意味着除非通过 `ssh -i [identity_file]` 或在 `.ssh/config` 中按主机明确命名,否则公钥身份验证将不会进行。

在 YubiKey 使用的情况下,从 ssh 代理中提取公钥:

```console
ssh-add -L | grep "cardno:000605553211" > ~/.ssh/id_rsa_yubikey.pub
```

然后明确关联此 YubiKey 存储的密钥与主机(例如 `github.com`)一起使用,如下所示:

```console
$ cat << EOF >> ~/.ssh/config
Host github.com
    IdentitiesOnly yes
    IdentityFile ~/.ssh/id_rsa_yubikey.pub
EOF
```

使用公钥身份验证连接:

```console
$ ssh git@github.com -vvv
[...]
debug2: key: cardno:000605553211 (0x1234567890),
debug1: Authentications that can continue: publickey
debug3: start over, passed a different list publickey
debug3: preferred gssapi-keyex,gssapi-with-mic,publickey,keyboard-interactive,password
debug3: authmethod_lookup publickey
debug3: remaining preferred: keyboard-interactive,password
debug3: authmethod_is_enabled publickey
debug1: Next authentication method: publickey
debug1: Offering RSA public key: cardno:000605553211
debug3: send_pubkey_test
debug2: we sent a publickey packet, wait for reply
debug1: Server accepts key: pkalg ssh-rsa blen 535
debug2: input_userauth_pk_ok: fp e5:de:a5:74:b1:3e:96:9b:85:46:e7:28:53:b4:82:c3
debug3: sign_and_send_pubkey: RSA e5:de:a5:74:b1:3e:96:9b:85:46:e7:28:53:b4:82:c3
debug1: Authentication succeeded (publickey).
[...]
```

> [!TIP]
> 要启用多个连接,请使用 [ControlMaster](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) SSH 选项。

### 导入 SSH 密钥

如果有现有的 SSH 密钥需要通过 `gpg-agent` 使用,则需要导入它们。然后,删除原始私钥。导入密钥时,`gpg-agent` 使用密钥文件名作为标签 - 这样可以更容易地跟踪密钥的来源。在此示例中,我们从仅使用 YubiKey 开始,并导入 `~/.ssh/id_rsa`:

```console
$ ssh-add -l
4096 SHA256:... cardno:00060123456 (RSA)

$ ssh-add ~/.ssh/id_rsa && rm ~/.ssh/id_rsa
```

调用 `ssh-add` 时,将出现 SSH 密钥密码提示,然后 `pinentry` 程序将提示并确认一个新密码来加密 GnuPG 密钥存储中的转换密钥。

迁移的密钥将列在 `ssh-add -l` 中:

```console
$ ssh-add -l
4096 SHA256:... cardno:00060123456 (RSA)
2048 SHA256:... /Users/username/.ssh/id_rsa (RSA)
```

要显示带有 MD5 指纹的密钥,如 `gpg-connect-agent` 的 `KEYINFO` 和 `DELETE_KEY` 命令所使用的:

```console
$ ssh-add -E md5 -l
4096 MD5:... cardno:00060123456 (RSA)
2048 MD5:... /Users/username/.ssh/id_rsa (RSA)
```

使用密钥时,将调用 `pinentry` 来请求密钥密码。密码将在使用之间缓存最多 10 分钟的空闲时间,最多 2 小时。

### SSH 代理转发

> [!CAUTION]
> SSH 代理转发可能会[增加额外的风险](https://matrix.org/blog/2019/05/08/post-mortem-and-remediations-for-apr-11-security-incident/#ssh-agent-forwarding-should-be-disabled) - 请谨慎操作!

有两种 ssh-agent 转发的方法,一种由 OpenSSH 提供,另一种由 GnuPG 提供。

后者可能更不安全,因为原始套接字只是被转发(不像 `S.gpg-agent.extra` 只有有限的功能;如果 OpenSSH 实现的 `ForwardAgent` 也只是转发原始套接字,那么它们的不安全程度是相同的)。但对于后者,一个便利之处是可以转发一次并在远程的任何地方使用此代理。所以再次强调,请谨慎操作!

例如,tmux 在连接到远程主机并附加现有会话时没有 `$SSH_AUTH_SOCK` 等环境变量。对于每个 shell,需要找到套接字并执行 `export SSH_AUTH_SOCK=/tmp/ssh-agent-xxx/xxxx.socket`。但是,使用固定位置的 `S.gpg-agent.ssh`,可以在 shell rc 文件中将其用作 ssh-agent。

#### 使用 ssh-agent

您现在应该能够在_本地_主机上使用 `ssh -A remote` 登录_远程_主机,然后应该能够使用 YubiKey,就像它连接到远程主机一样。例如,在该远程主机上使用 `ssh-add -l` 将显示来自 YubiKey 的公钥(`cardno:`)。始终仅对单个主机使用 `ForwardAgent yes`,绝不对所有服务器使用。

#### 使用 S.gpg-agent.ssh

首先,您需要完成 [GnuPG 代理转发](#gnupg-agent-forwarding),了解 gpg-agent 转发的条件,并了解本地和远程上 `S.gpg-agent.ssh` 的位置。

您可以使用以下命令:

```console
$ gpgconf --list-dirs agent-ssh-socket
```

编辑 `.ssh/config` 以添加远程主机:

```console
Host
    Hostname remote-host.tld
    StreamLocalBindUnlink yes
    RemoteForward /run/user/1000/gnupg/S.gpg-agent.ssh /run/user/1000/gnupg/S.gpg-agent.ssh
    #RemoteForward [远程套接字] [本地套接字]
    #注意这里不需要 ForwardAgent!
```

成功 ssh 到远程主机后,确认 `/run/user/1000/gnupg/S.gpg-agent.ssh` 存在。

然后在*远程*主机上,您可以在命令行中键入或在 shell rc 文件中配置:

```console
export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"
```

在源化 shell rc 文件后,`ssh-add -l` 将返回正确的公钥。

在这个过程中,远程主机上的 gpg-agent 不参与,因此远程主机上的 `gpg-agent.conf` 无用。此外,pinentry 在本地调用。

#### 链式转发

如果您使用 OpenSSH 提供的 `ssh-agent` 并想将其转发到*第三*台机器,您可以在*远程*主机上执行 `ssh -A third`。

同时,如果您使用 `S.gpg-agent.ssh`,假设您已经完成了上述步骤并在*远程*主机上有 `S.gpg-agent.ssh`,并且您想将此代理转发到*第三*台机器,首先您可能需要以与*远程*主机相同的方式配置*第三*台机器的 `sshd_config` 和 `SSH_AUTH_SOCK`,然后在*远程*主机的 ssh 配置中,添加以下行

```console
Host third
    Hostname third-host.tld
    StreamLocalBindUnlink yes
    RemoteForward /run/user/1000/gnupg/S.gpg-agent.ssh /run/user/1000/gnupg/S.gpg-agent.ssh
    #RemoteForward [远程套接字] [本地套接字]
    #注意这里不需要 ForwardAgent!
```

路径必须根据*远程*和*第三*主机上的 `gpgconf --list-dirs agent-ssh-socket` 设置。

## GitHub

YubiKey 可用于签名提交和标签,并在 [Settings](https://github.com/settings/keys) 中配置时对 GitHub 进行 SSH 身份验证。

配置签名密钥:

```console
git config --global user.signingkey $KEYID
```

或者,如果您使用上述 `IdentityFile`(SSH 密钥)进行签名:

```console
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_rsa_yubikey.pub
```

配置 `user.name` 和 `user.email` 选项以匹配与 PGP 身份关联的电子邮件地址:

```console
git config --global user.name 'YubiKey User'
git config --global user.email yubikey@example
```

要签名提交或标签,请使用 `-S` 选项,或考虑默认启用提交和标签签名:

```console
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

**Windows**

配置身份验证:

```console
git config --global core.sshcommand "plink -agent"

git config --global gpg.program 'C:\Program Files (x86)\GnuPG\bin\gpg.exe'
```

然后将存储库 URL 更新为 `git@github.com:USERNAME/repository`

## GnuPG 代理转发

YubiKey 可用于通过 GnuPG 代理转发在远程主机上签名 git 提交和解密文件。要通过另一个网络进行 ssh,特别是使用 ssh 推送到/从 GitHub 拉取,请参阅[远程机器(SSH 代理转发)](#ssh-agent-forwarding)。

远程主机上不需要 `gpg-agent.conf`;转发后,远程 GnuPG 直接与 `S.gpg-agent` 通信,而无需在远程主机上启动 `gpg-agent`。

在远程主机上,编辑 `/etc/ssh/sshd_config` 以设置 `StreamLocalBindUnlink yes`

**可选** 如果在远程主机上没有 root 访问权限来编辑 `/etc/ssh/sshd_config`,则需要在转发工作之前删除远程主机上 `gpgconf --list-dir agent-socket` 位置的套接字。有关更多信息,请参阅 [AgentForwarding GNUPG wiki 页面](https://wiki.gnupg.org/AgentForwarding)。

在远程主机上导入公钥。在本地主机上,将公钥环复制到远程主机:

```console
scp ~/.gnupg/pubring.kbx remote:~/.gnupg/
```

在现代发行版(如 Fedora 30)上,无需在 `~/.ssh/config` 中设置 `RemoteForward`

### 旧版发行版

在本地主机上运行:

```console
gpgconf --list-dirs agent-extra-socket
```

这应该返回 agent-extra-socket 的路径 - `/run/user/1000/gnupg/S.gpg-agent.extra` - 尽管在较旧的 Linux 发行版(和 macOS)上它可能是 `/home/<user>/.gnupg/S.gpg-agent.extra`

在**远程**主机上找到代理套接字:

```console
gpgconf --list-dirs agent-socket
```

这应该返回一个路径,例如 `/run/user/1000/gnupg/S.gpg-agent`

最后,通过将以下内容添加到本地主机的 `~/.ssh/config`(代理套接字可能不同)来为给定主机启用代理转发:

```
Host
  Hostname remote-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent /run/user/1000/gnupg/S.gpg-agent.extra
  #RemoteForward [远程套接字] [本地套接字]
```

可能需要编辑*本地*主机上的 `gpg-agent.conf` 以添加以下信息:

```
pinentry-program /usr/bin/pinentry-gtk-2
extra-socket /run/user/1000/gnupg/S.gpg-agent.extra
```

> [!IMPORTANT]
> pinentry 程序在*本地*主机上启动,而不是远程主机。

可以使用除 `pinentry-tty` 或 `pinentry-curses` 之外的任何 pinentry 程序。这是因为本地 `gpg-agent` 可能会无头启动(由 systemd 启动而没有本地设置 `$GPG_TTY` 告诉它在哪个 tty 上),因此无法获取 pin。远程上的错误可能会误导性地说有 *IO Error*。(是的,内部确实有 *IO Error*,因为它在写入/读取 tty 时发生,但找不到要使用的 tty,但对于最终用户来说这并不友好。)

有关更多信息和故障排除,请参阅 [Issue 85](https://github.com/drduh/YubiKey-Guide/issues/85)。

### 链式 GnuPG 代理转发

假设您已完成上述步骤并在*远程*主机上有 `S.gpg-agent`,并且您想将此代理转发到*第三*台机器,首先您可能需要以与*远程*主机相同的方式配置*第三*台机器的 `sshd_config`,然后在*远程*主机的 ssh 配置中,添加以下行:

```console
Host third
  Hostname third-host.tld
  StreamLocalBindUnlink yes
  RemoteForward /run/user/1000/gnupg/S.gpg-agent /run/user/1000/gnupg/S.gpg-agent
  #RemoteForward [远程套接字] [本地套接字]
```

您应该根据*远程*和*第三*台机器上的 `gpgconf --list-dirs agent-socket` 更改路径。

在*本地*主机上,您有 `S.gpg-agent.extra`,而在*远程*和*第三*台机器上,您只有 `S.gpg-agent`

## 使用多个 YubiKey

当使用 `keytocard` 将 GnuPG 密钥添加到 YubiKey 时,密钥将从密钥环中删除,并添加一个**存根**,指向 YubiKey。存根标识 GnuPG 密钥 ID 和 YubiKey 序列号。

当子密钥添加到另一个 YubiKey 时,存根将被覆盖,现在将指向最新的 YubiKey。GnuPG 将按照存根引用的序列号请求特定的 YubiKey,并且不会识别具有不同序列号的另一个 YubiKey。

要扫描另一个 YubiKey 并重新创建正确的存根:

```console
gpg-connect-agent "scd serialno" "learn --force" /bye
```

或者,使用脚本删除 GnuPG 阴影密钥,其中存储了序列号(请参阅 [GnuPG #T2291](https://dev.gnupg.org/T2291)):

```console
mkdir -p ~/scripts && cat >> ~/scripts/remove-keygrips.sh <<EOF
#!/usr/bin/env bash
(( $# )) || { echo "Specify a key." >&2; exit 1; }
KEYGRIPS=$(gpg --with-keygrip --list-secret-keys "$@" | awk '/Keygrip/ { print $3 }')
for keygrip in $KEYGRIPS
do
    rm "$HOME/.gnupg/private-keys-v1.d/$keygrip.key" 2> /dev/null
done

gpg --card-status
EOF

chmod +x ~/scripts/remove-keygrips.sh

~/scripts/remove-keygrips.sh $KEYID
```

有关更多信息和故障排除步骤,请参阅 Issues [#19](https://github.com/drduh/YubiKey-Guide/issues/19) 和 [#112](https://github.com/drduh/YubiKey-Guide/issues/112) 中的讨论。

## 电子邮件

YubiKey 可用于使用 [Thunderbird](https://www.thunderbird.net/)、[Enigmail](https://www.enigmail.net) 和 [Mutt](http://www.mutt.org/) 解密和签名电子邮件及附件。Thunderbird 支持 OAuth 2 身份验证,可以与 Gmail 一起使用。有关更多信息,请参阅 [此 EFF 指南](https://ssd.eff.org/en/module/how-use-pgp-linux)。Mutt 自 2.0 版本起支持 OAuth 2。

### Thunderbird

按照 [mozilla wiki 上的说明](https://wiki.mozilla.org/Thunderbird:OpenPGP:Smartcards#Configure_an_email_account_to_use_an_external_GnuPG_key)使用外部 gpg 提供程序设置您的 YubiKey 与 Thunderbird 客户端。

> [!NOTE]
> 如果在 `gpg.conf` 中启用了 ASCII `armor` 选项,Thunderbird 将[无法](https://github.com/drduh/YubiKey-Guide/issues/448)解密电子邮件。如果您看到错误 `gpg: [don't know]: invalid packet (ctb=2d)` 或 `message cannot be decrypted (there are unknown problems with this encrypted message)`,只需删除此选项即可。

### Mailvelope

[Mailvelope](https://www.mailvelope.com/en) 允许 YubiKey 与 Gmail 和其他服务一起使用。

> [!NOTE]
> Mailvelope [不能](https://github.com/drduh/YubiKey-Guide/issues/178)与 `gpg.conf` 中设置的 `throw-keyids` 选项一起使用

在 macOS 上,使用 Homebrew 安装 gpgme:

```console
brew install gpgme
```

要允许 Chrome 运行 gpgme,请编辑 `~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/gpgmejson.json` 以添加:

```json
{
    "name": "gpgmejson",
    "description": "Integration with GnuPG",
    "path": "/usr/local/bin/gpgme-json",
    "type": "stdio",
    "allowed_origins": [
        "chrome-extension://kajibbejlbohfaggdiogboambcijhkke/"
    ]
}
```

编辑默认路径以允许 Chrome 找到 GnuPG:

```console
sudo launchctl config user path /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
```

最后,从 Chrome 网上应用店安装 [Mailvelope 扩展](https://chromewebstore.google.com/detail/mailvelope/kajibbejlbohfaggdiogboambcijhkke)。

### Mutt

Mutt 同时具有 CLI 和 TUI 界面 - 后者为处理电子邮件提供了强大的功能。此外,可以集成 PGP,以便在不离开 TUI 的情况下完成加密操作。

要启用 GnuPG 支持,请复制 `/usr/share/doc/mutt/samples/gpg.rc`

编辑文件以启用选项 `pgp_default_key`、`pgp_sign_as` 和 `pgp_autosign`

在 `muttrc` 中 `source` 该文件

> [!NOTE]
> 据报告,将 `pinentry-tty` 设置为 pinentry 程序(在 `gpg-agent.conf` 中)会导致 Mutt TUI 出现问题,因为它使用 curses;请改用 `pinentry-curses` 或其他图形 pinentry 程序。

## 密钥服务器

公钥可以上传到公共服务器以供发现:

```console
gpg --send-key $KEYID

gpg --keyserver keys.gnupg.net --send-key $KEYID

gpg --keyserver hkps://keyserver.ubuntu.com:443 --send-key $KEYID
```

或者如果[上传到 keys.openpgp.org](https://keys.openpgp.org/about/usage):

```console
gpg --export $KEYID | curl -T - https://keys.openpgp.org
```

公钥 URL 也可以添加到 YubiKey(基于 [Shaw 2003](https://datatracker.ietf.org/doc/html/draft-shaw-openpgp-hkp-00)):

```console
URL="hkps://keyserver.ubuntu.com:443/pks/lookup?op=get&search=${KEYID}"
```

使用 `gpg --edit-card` 和管理员 PIN 编辑 YubiKey:

```console
gpg/card> admin

gpg/card> url
URL to retrieve public key: hkps://keyserver.ubuntu.com:443/pks/lookup?op=get&search=0xFF00000000000000

gpg/card> quit
```

# 更新密钥

PGP 不提供[前向保密](https://en.wikipedia.org/wiki/Forward_secrecy),这意味着被泄露的密钥可能用于解密所有过去的消息。虽然存储在 YubiKey 上的密钥更难被利用,但这并非不可能:密钥和 PIN 可能会被物理泄露,或者可能会在固件或用于创建密钥的随机数生成器中发现漏洞,例如。因此,建议定期轮换子密钥。

当子密钥过期时,可以更新或替换。这两种操作都需要访问认证密钥。

- 通过更新过期时间来更新子密钥表示继续保管认证密钥,通常更方便。

- 替换子密钥不太方便,但可能更安全:新子密钥将**不能**解密以前的消息,也不能使用 SSH 进行身份验证等。接收者将需要更新的公钥。任何加密的秘密必须解密并重新加密到新子密钥。此过程在功能上等同于丢失 YubiKey 并配置新的。

两种轮换方法都没有优劣之分,这取决于个人对身份管理的理念和个人威胁模型来决定使用哪一种,或者是否要使子密钥过期。理想情况下,子密钥应该是短暂的:仅用于每个唯一的加密、签名和身份验证事件一次,但在实践中,使用 YubiKey 这并不太实用或值得。高级用户可能会专门使用一台气隙机器进行频繁的凭证轮换。

要更新或轮换子密钥,请遵循与生成密钥相同的过程:引导到安全环境,安装所需软件并禁用网络。

连接带有认证密钥的便携式存储设备并识别磁盘标签。

解密并挂载加密卷:

```console
sudo cryptsetup luksOpen /dev/sdc1 gnupg-secrets

sudo mkdir -p /mnt/encrypted-storage

sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage
```

挂载未加密的公共分区:

```console
sudo mkdir -p /mnt/public

sudo mount /dev/sdc2 /mnt/public
```

将原始私钥材料(更新加密存储目录名称后)复制到临时工作目录:

```console
export GNUPGHOME=$(mktemp -d -t $(date +%Y.%m.%d)-XXXX)

cp -avi /mnt/encrypted-storage/2025.12.31-AbCd/* $GNUPGHOME/
```

确认身份可用,设置密钥 ID 和指纹:

```console
gpg -K

export KEYID=$(gpg -k --with-colons "$IDENTITY" | \
    awk -F: '/^pub:/ { print $5; exit }')

export KEYFP=$(gpg -k --with-colons "$IDENTITY" | \
    awk -F: '/^fpr:/ { print $10; exit }')

echo $KEYID $KEYFP
```

回忆认证密钥密码并设置它,例如:

```console
export CERTIFY_PASS=ABCD-0123-IJKL-4567-QRST-UVWX
```

## 续期子密钥

设置更新的过期日期:

```console
export EXPIRATION=2027-09-01
```

更新子密钥:

```console
echo "$CERTIFY_PASS" | \
    gpg --batch --pinentry-mode=loopback \
        --passphrase-fd 0 --quick-set-expire "$KEYFP" "$EXPIRATION" \
    $(gpg -K --with-colons | awk -F: '/^fpr:/ { print $10 }' | tail -n "+2" | tr "\n" " ")
```

导出更新的公钥:

```console
gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc
```

将公钥传输到目标主机并导入:

```console
gpg --import /mnt/public/*.asc
```

或者,发布到公钥服务器并下载:

```console
gpg --send-key $KEYID

gpg --recv $KEYID
```

GnuPG 身份的有效性将被延长,允许再次用于加密和签名操作。

SSH 公钥在远程主机上**不**需要更新。

## 轮换子密钥

遵循原始过程[创建子密钥](#create-subkeys)。

可以从身份中删除以前的子密钥。

通过将新子密钥传输到 YubiKey 来完成。

将**新**临时工作目录复制到加密存储(仍在挂载中):

```console
sudo cp -avi $GNUPGHOME /mnt/encrypted-storage
```

卸载并关闭加密卷:

```console
sudo umount /mnt/encrypted-storage

sudo cryptsetup luksClose gnupg-secrets
```

导出更新的公钥:

```console
sudo mkdir -p /mnt/public

sudo mount /dev/sdc2 /mnt/public

gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc

sudo umount /mnt/public
```

移除存储设备并按照原始步骤将新子密钥(`4`、`5` 和 `6`)传输到 YubiKey,替换现有的。

重启或安全擦除 GnuPG 临时工作目录。

# 重置 YubiKey

如果超过 PIN 尝试次数,YubiKey 将被锁定,必须[重置](https://developers.yubico.com/ykneo-openpgp/ResetApplet.html)并使用加密备份重新设置。

将以下内容复制到文件并运行 `gpg-connect-agent -r $file`,然后重新插入 YubiKey 以完成重置。

```console
/hex
scd serialno
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 81 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 20 00 83 08 40 40 40 40 40 40 40 40
scd apdu 00 e6 00 00
scd apdu 00 44 00 00
/echo Card has been successfully reset.
/bye
```

或使用 `ykman`(有时在 `~/.local/bin/` 中):

```console
$ ykman openpgp reset
WARNING! This will delete all stored OpenPGP keys and data and restore factory settings? [y/N]: y
Resetting OpenPGP data, don't remove your YubiKey...
Success! All data has been cleared and default PINs are set.
PIN:         123456
Reset code:  NOT SET
Admin PIN:   12345678
```

# 可选加固

以下步骤可能会提高 YubiKey 的安全性和隐私性。

## 改善熵


生成加密密钥需要高质量的[随机性](https://www.random.org/randomness/)，以熵来衡量。大多数操作系统使用基于软件的伪随机数生成器或基于 CPU 的硬件随机数生成器（HRNG）。

可选地，可以使用诸如 [OneRNG](https://onerng.info/onerng/) 之类的设备来[提高速度](https://lwn.net/Articles/648550/)，并可能提高可用熵的质量。

在创建密钥之前，配置 [rng-tools](https://wiki.archlinux.org/title/Rng-tools)：

```console
sudo apt -y install at rng-tools python3-gnupg openssl

wget https://github.com/OneRNG/onerng.github.io/raw/master/sw/onerng_3.7-1_all.deb
```

验证包：

```console
sha256sum onerng_3.7-1_all.deb
```

该值必须匹配：

```console
b7cda2fe07dce219a95dfeabeb5ee0f662f64ba1474f6b9dddacc3e8734d8f57
```

安装包：

```console
sudo dpkg -i onerng_3.7-1_all.deb

echo "HRNGDEVICE=/dev/ttyACM0" | sudo tee /etc/default/rng-tools
```

插入设备并重启 rng-tools：

```console
sudo atd

sudo service rng-tools restart
```

## 启用 KDF

> [!IMPORTANT]
> 此功能可能与较旧的 GnuPG 版本不兼容，尤其是移动客户端。这些不兼容的客户端将无法正常工作，因为 PIN 始终会被拒绝。

此步骤必须在更改 PIN 或移动密钥之前完成，否则会出现错误：`gpg: error for setup KDF: Conditions of use not satisfied`

密钥派生功能（KDF）使 YubiKey 能够存储 PIN 的哈希值，防止 PIN 以明文形式传递。

使用默认管理员 PIN `12345678` 启用 KDF：

```console
gpg --command-fd=0 --pinentry-mode=loopback --card-edit <<EOF
admin
kdf-setup
12345678
EOF
```

## 网络注意事项

本节主要针对基于 Debian / Ubuntu 的系统，但同样的概念适用于连接到网络的任何系统。

无论您是使用虚拟机、在专用硬件上安装，还是临时运行 Live OS，都应在*没有*网络连接的情况下启动，并在连接到网络之前禁用在所有接口上监听的任何不必要的服务。

这样做的原因是因为像 cups 或 avahi 这样的服务可能默认处于监听状态。虽然这不是一个直接的问题，但它只是扩大了攻击面。并非每个人都有专用的子网或他们可以控制的可信网络设备，出于本指南的目的，这些步骤将*任何*网络视为不可信/敌对的。

**禁用监听服务**

- 确保仅运行必要的网络服务
- 如果服务不存在，您将收到"Failed to stop"，这没有问题
- 如果您不需要蓝牙，则仅禁用 `Bluetooth`

```bash
sudo systemctl stop bluetooth exim4 cups avahi avahi-daemon sshd
```

**防火墙**

启用*拒绝入站，允许出站*的基本防火墙策略。请注意，Debian 不附带防火墙，只需在上一步中禁用服务即可。以下选项适用于 Ubuntu 和类似系统。

在 Ubuntu 上，`ufw` 是内置的并且易于启用：

```bash
sudo ufw enable
```

在没有 `ufw` 的系统上，`nftables` 正在取代 `iptables`。[nftables wiki 有示例](https://wiki.nftables.org/wiki-nftables/index.php/Simple_ruleset_for_a_workstation)提供了基线*拒绝入站，允许出站*策略。`fw.inet.basic` 策略涵盖 IPv4 和 IPv6。

（在创建可启动介质时，请记住将此 README 和任何其他资源下载到另一个外部驱动器，以便离线使用此信息）

无论您使用哪种策略，将内容写入文件（例如 `nftables.conf`）并使用以下命令应用策略：

```bash
sudo nft -f ./nftables.conf
```

**检查系统状态**

`NetworkManager` 应该是在端口 68/udp 上唯一监听的服务，用于获取 DHCP 租约（如果您有 IPv6，则为 58/icmp6）。

如果您想查看每个进程的命令行参数，可以使用 `ps axjf`。这会打印一个进程树，可能有大量行，但在 Live 映像或全新安装上应该易于阅读。

```bash
sudo ss -anp -A inet    # 转储所有网络状态信息
ps axjf                 # 在进程树中列出所有进程
ps aux                  # BSD 语法，列出所有进程但没有进程树
```

如果您发现任何其他不需要的在网络上监听的进程，请记录并使用以下命令之一禁用它们：

```bash
sudo systemctl stop <process-name>                      # 停止由 systemctl 管理的服务
sudo pkill -f '<process-name-or-command-line-string>'   # 通过匹配其命令行字符串终止进程
pgrep -f '<process-name-or-command-line-string>'        # 获取 PID
sudo kill <pid>                                         # 通过其 PID 终止进程
```

现在连接网络。

# 注意事项

1. YubiKey 有两种配置，可以通过短按或长按调用。默认情况下，短按模式配置为 HID OTP；简短触摸将发出以 `cccccccc` 开头的 OTP 字符串。可以通过 YubiKey 个性化工具将 OTP 模式交换到第二个配置，或使用 [YubiKey Manager](https://developers.yubico.com/yubikey-manager) 完全禁用：`ykman config usb -d OTP`

1. 使用 YubiKey 进行 GnuPG 不会阻止使用[其他功能](https://developers.yubico.com/)，例如 [WebAuthn](https://developers.yubico.com/WebAuthn/) 和 [OTP](https://developers.yubico.com/OTP/)。

1. 在设置期间使用 `adduid` 命令向认证密钥添加其他身份，然后使用 `trust` 和 `5` 最终信任它以配置使用。

1. 要在 YubiKey 之间切换，请移除第一个 YubiKey 并使用 `pkill "gpg-agent|ssh-agent|pinentry" ; eval $(gpg-agent --daemon --enable-ssh-support)` 重启 gpg-agent、ssh-agent 和 pinentry，然后插入另一个 YubiKey 并运行 `gpg-connect-agent updatestartuptty /bye`

1. 要在多台计算机上使用 YubiKey，导入相应的公钥，然后使用 `gpg --card-status` 确认 YubiKey 可见。使用 `trust` 和 `5` 最终信任导入的公钥，然后 `gpg --list-secret-keys` 将显示正确且受信任的密钥。

1. 当您的认证密钥离线时，*买者自负*：如果您希望[参加密钥签名聚会](https://www.gnupg.org/gph/en/manual/x334.html)，您会发现[签名他人导入的公钥](https://gist.github.com/F21/b0e8c62c49dfab267ff1d0c6af39ab84)需要首先设置一个安全的飞地，例如上面描述的临时环境，并将您的认证密钥导入该飞地。[签名子密钥不能用于签名他人导入的公钥](https://security.stackexchange.com/questions/153057/possible-to-sign-an-imported-key-with-a-subkey-using-gpg)。

# 故障排除

- 使用 `man gpg` 了解 GnuPG 选项和命令行标志。

- 要获取有关潜在错误的更多信息，使用 `pkill gpg-agent; gpg-agent --daemon --no-detach -v -v --debug-level advanced --homedir ~/.gnupg` 重启 `gpg-agent` 进程并将调试输出发送到控制台。

- 许多问题可以通过移除并重新插入 YubiKey 或重启 `gpg-agent` 进程来解决。

- 如果您收到错误 `Yubikey core error: no yubikey present` - 确保 YubiKey 正确插入。插入时它应该闪烁一次。

- 如果您仍然收到错误 `Yubikey core error: no yubikey present` - 您可能需要安装较新版本的 yubikey-personalize，如[安装软件](#install-software)中所述。

- 如果您在卡状态输出中看到 `General key info..: [none]` - 导入公钥。

- 如果您收到错误 `gpg: decryption failed: secret key not available` - 您可能需要安装 GnuPG 2.x 版本。另一种可能性是 PIN 存在问题，例如，它太短或被阻止。

- 如果您收到错误 `Yubikey core error: write error` - YubiKey 可能被锁定。安装并运行 yubikey-personalization-gui 以解锁它。

- 如果您收到错误 `Key does not match the card's capability` - 您可能需要使用 2048 位 RSA 密钥大小。

- 如果您收到错误 `sign_and_send_pubkey: signing failed: agent refused operation` - 确保您按照上述说明用 `gpg-agent` 替换了 `ssh-agent`。

- 如果您仍然收到错误 `sign_and_send_pubkey: signing failed: agent refused operation` - [运行命令](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=835394) `gpg-connect-agent updatestartuptty /bye`

- 如果您仍然收到错误 `sign_and_send_pubkey: signing failed: agent refused operation` - 编辑 `~/.gnupg/gpg-agent.conf` 以设置有效的 `pinentry` 程序路径。`gpg: decryption failed: No secret key` 也可能表示无效的 `pinentry` 路径

- 如果您仍然收到错误 `sign_and_send_pubkey: signing failed: agent refused operation` - 这是一个[已知问题](https://bbs.archlinux.org/viewtopic.php?id=274571)，openssh 8.9p1 及更高版本与 YubiKey 存在问题。将 `KexAlgorithms -sntrup761x25519-sha512@openssh.com` 添加到 `/etc/ssh/ssh_config` 通常可以解决该问题。

- 如果您从 `ssh-add -L` 收到错误 `The agent has no identities`，请确保您已安装并启动 `scdaemon`

- 如果您从 `ssh-add -L` 收到错误 `Error connecting to agent: No such file or directory`，代理用于与其他进程通信的 UNIX 文件套接字可能未正确设置。在 Debian 上，尝试 `export SSH_AUTH_SOCK="/run/user/$UID/gnupg/S.gpg-agent.ssh"`。还要确保 `gpgconf --list-dirs agent-ssh-socket` 返回单个路径，指向现有的 `S.gpg-agent.ssh` 套接字。

- 如果您收到错误 `Permission denied (publickey)`，使用 `-v` 标志增加 ssh 详细程度，并验证卡中的公钥是否被提供：`Offering public key: RSA SHA256:abcdefg... cardno:00060123456`。如果是，验证目标系统上的正确用户 - 而不是本地系统上的用户。否则，请确保此主机未[启用](https://github.com/FiloSottile/whosthere#how-do-i-stop-it) `IdentitiesOnly`。

- 如果 SSH 身份验证仍然失败 - 在 `ssh` 命令中最多添加 3 个 `-v` 标志以增加详细程度。

- 如果仍然失败，在服务器上停止后台 `sshd` 守护进程服务（例如使用 `sudo systemctl stop sshd`）并改为使用 `/usr/sbin/sshd -eddd` 以大量调试输出在前台启动它可能会有所帮助。请注意，服务器将不会分叉，只会处理一个连接，因此必须在每次 `ssh` 测试后重新启动。

- 如果您收到错误 `Please insert the card with serial number`，请参见[使用多个密钥](#using-multiple-yubikeys)。

- 如果您收到错误 `There is no assurance this key belongs to the named user` 或 `encryption failed: Unusable public key` 或 `No public key`，使用 `gpg --edit-key` 将 `trust` 设置为 `5 = I trust ultimately`

- 如果在尝试上述命令时收到错误 `Need the secret key to do this` - 通过使用 `trust-key [key ID]` 指令在 `~/.gnupg/gpg.conf` 中为密钥指定信任。

- 如果在新计算机上使用以前配置的 YubiKey 使用 `pass` 时，在 `pass insert` 上看到以下错误，则需要调整与密钥关联的信任。请参见上面的注释。

```
gpg: 0x0000000000000000: There is no assurance this key belongs to the named user
gpg: [stdin]: encryption failed: Unusable public key
```

- 如果您收到错误 `gpg: 0x0000000000000000: skipped: Unusable public key`、`signing failed: Unusable secret key` 或 `encryption failed: Unusable public key`，子密钥可能已过期，无法再用于加密或签名消息。但是，它仍然可以用于解密和身份验证。

- 如果 _pinentry_ 图形对话框未显示并出现此错误：`sign_and_send_pubkey: signing failed: agent refused operation`，安装 `dbus-user-session` 包并重启以完全继承 `dbus` 用户会话。这是因为 `pinentry` 抱怨 `No $DBUS_SESSION_BUS_ADDRESS found`，回退到 `curses` 但找不到预期的 `tty`

- 如果在尝试上述 `--card-status` 命令时收到错误 `gpg: selecting card failed: No such device` 或 `gpg: OpenPGP card not available: No such device`，最新版本的 pcscd 可能需要 polkit 规则才能正常运行。创建以下文件以允许 `wheel` 组中的用户使用卡。完成后务必重启 pcscd 以使新规则生效。

```console
cat << EOF >  /etc/polkit-1/rules.d/99-pcscd.rules
polkit.addRule(function(action, subject) {
        if (action.id == "org.debian.pcsc-lite.access_card" &&
                subject.isInGroup("wheel")) {
                return polkit.Result.YES;
        }
});
polkit.addRule(function(action, subject) {
        if (action.id == "org.debian.pcsc-lite.access_pcsc" &&
                subject.isInGroup("wheel")) {
                return polkit.Result.YES;
        }
});
EOF
```

- 如果丢失了公钥，请遵循[此指南](https://www.nicksherlock.com/2021/08/recovering-lost-gpg-public-keys-from-your-yubikey/)从 YubiKey 恢复它。

- 参考 Yubico 文章[排除 GPG 问题](https://support.yubico.com/hc/en-us/articles/360013714479-Troubleshooting-Issues-with-GPG)获取更多指导。

# 替代方案

* [`vorburger/ed25519-sk.md`](https://github.com/vorburger/vorburger.ch-Notes/blob/develop/security/ed25519-sk.md) - 使用 YubiKey 进行 SSH 而不使用 GnuPG
* [`smlx/piv-agent`](https://github.com/smlx/piv-agent) - SSH 和 GnuPG 代理，可与 PIV 设备一起使用
* [`keytotpm`](https://www.gnupg.org/documentation/manuals/gnupg/OpenPGP-Key-Management.html) - 在 TPM 系统上使用 GnuPG

# 其他资源

* [Yubico - PGP](https://developers.yubico.com/PGP/)
* [Yubico - Yubikey Personalization](https://developers.yubico.com/yubikey-personalization/)
* [A Visual Explanation of GPG Subkeys (2022)](https://rgoulter.com/blog/posts/programming/2022-06-10-a-visual-explanation-of-gpg-subkeys.html)
* [dhess/nixos-yubikey](https://github.com/dhess/nixos-yubikey)
* [lsasolutions/makegpg](https://gitlab.com/lsasolutions/makegpg)
* [Trammell Hudson - Yubikey (2020)](https://trmm.net/Yubikey)
* [Yubikey forwarding SSH keys (2019)](https://blog.onefellow.com/post/180065697833/yubikey-forwarding-ssh-keys)
* [GPG Agent Forwarding (2018)](https://mlohr.com/gpg-agent-forwarding/)
* [Stick with security: YubiKey, SSH, GnuPG, macOS (2018)](https://evilmartians.com/chronicles/stick-with-security-yubikey-ssh-gnupg-macos)
* [PGP and SSH keys on a Yubikey NEO (2015)](https://www.esev.com/blog/post/2015-01-pgp-ssh-key-on-yubikey-neo/)
* [Offline GnuPG Master Key and Subkeys on YubiKey NEO Smartcard (2014)](https://blog.josefsson.org/2014/06/23/offline-gnupg-master-key-and-subkeys-on-yubikey-neo-smartcard/)
* [Creating the perfect GPG keypair (2013)](https://alexcabal.com/creating-the-perfect-gpg-keypair/)

[^1]: 使用单引号包裹双引号字符（`"`）- `export IDENTITY='My Identity (a.k.a. "YubiKey User") <yubikey@example.domain>'`
[^2]: [撤销证书](https://security.stackexchange.com/questions/14718/does-openpgp-key-expiration-add-to-security/79386#79386)应该用于撤销身份。
[^3]: 有关 NIST 指南讨论，请参见 [issue 477](https://github.com/drduh/YubiKey-Guide/issues/477)。
