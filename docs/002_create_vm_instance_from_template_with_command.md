# テンプレートから VM インスタンスを作成(コマンド)

## 概要

## 前提
* `qemu 9000(almalinux-8-template)` が作成済み  
[AlmaLinux 8 の汎用クラウド(Cloud-init)イメージをテンプレートに登録](docs/001_create_template_almalinux_8_cloud_image.md)

## 詳細
* `local` ストレージのコンテンツタイプに `snippets` を追加  
    ※ cloud-init の cloud-config ファイル置き場に使用
    ```
    pvesm set local --content images,iso,vztmpl,backup,snippets
    ```


* 以下を丸ごと貼り付けて実行
    * `qemu 9000(almalinux-8-template)` をテンプレートに VM を作成
    * VMID:`100`, VM名:`almalinux-8-100`
    * ユーザーID・パスワードは `alma/alma`
    * VM のスペックは、`2core`、メモリ `2GB`、ディスクサイズ `20GB`  
    * 日本語化、ロケール:`ja_JP.utf8`、タイムゾーン:`Asia/Tokyo`
    * `almalinux-8-100.local` で名前解決できるよう `avahi` をインストール
    * 開発用に `nano, git, docker` をインストール
    * IPv6 無効化　※不要な場合はコメントアウトして

```sh
# 設定
template_vm_id=9000
vm_id=100
vm_name=almalinux-8-${vm_id}
vm_user=alma
vm_pass=alma
vm_cores=2
vm_memory=2048
vm_disksize=20G

vm_pass_hash=$(openssl passwd -6 "${vm_pass}")
cloud_config_filename=${vm_name}_cloud-config.yaml

# cloud-init
cat <<EOF > "/var/lib/vz/snippets/${cloud_config_filename}"
#cloud-config
hostname: ${vm_name}
fqdn: ${vm_name}.local

bootcmd:
  - dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

package_update: true
packages:
  - glibc-langpack-ja
  - langpacks-ja
  - avahi
  - nano
  - git
  - docker-ce
  - docker-ce-cli
  - containerd.io

timezone: Asia/Tokyo
locale: ja_JP.utf8
keyboard:
  layout: jp

# Proxmox Console でロケールが適用されない問題への対応
write_files:
  - path: /etc/profile.d/locale_load.sh
    content: |
      if [ -f /etc/locale.conf ]; then
          source /etc/locale.conf
      fi
    permissions: '0644'

ssh_pwauth: true

users:
  - name: ${vm_user}
    passwd: ${vm_pass_hash}
    lock_passwd: false
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: 
      - sudo
      - docker
    shell: /bin/bash

runcmd:
    # IPv6 無効化
    - echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
    - echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
    - sysctl -p

    # avahi-daemon の起動 & 自動起動の有効化
    - systemctl start avahi-daemon
    - systemctl enable avahi-daemon

    # docker の起動 & 自動起動の有効化
    - systemctl start docker
    - systemctl enable docker
EOF

qm clone ${template_vm_id} ${vm_id}
qm set ${vm_id} --name ${vm_name}
qm set ${vm_id} --cores ${vm_cores}
qm set ${vm_id} --memory ${vm_memory}
qm resize ${vm_id} scsi0 ${vm_disksize}
qm set ${vm_id} --cicustom "user=local:snippets/${cloud_config_filename}"
```