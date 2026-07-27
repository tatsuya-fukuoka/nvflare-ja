.. _base_image_build:

#############################################
CVM ベースイメージとバイナリのビルド
#############################################

このドキュメントでは、 ``nvflare provision`` を実行する前に CVM ビルダーが必要とする、前提となるベースイメージとバイナリのビルド方法を説明します。

イメージビルダーは NVFlare のソースツリーの ``nvflare/lighter/cc/image_builder/`` 配下に含まれており、
``cvm_build.sh`` スクリプトのほか、Ansible のプレイブックとヘルパースクリプトが含まれています。通常は
``~/cc/image_builder`` にインストールされ、これは ``project.yml`` の ``build_image_cmd`` で参照されるパスです。

以下の成果物をビルドし、イメージビルダーのディレクトリに配置する必要があります。

- ``base_images/ubuntu_base.qcow2`` — Ubuntu のベースディスクイメージ
- ``base_images/OVMF.amdsev.fd`` — ``kernel-hashes=on`` をサポートするファームウェア
- ``binaries/snpguest`` — TEE とやり取りするためのツール
- ``binaries/kbs-client`` — Trustee KBS と通信するためのツール

.. note::

   以下の例において、 ``<builder_root>`` は ``cvm_build.sh`` スクリプトを含むディレクトリを指します。

Ubuntu ベースイメージのビルド
========================================

Ubuntu のベースイメージは、 **Ubuntu 25.04 のホスト** 上で **Ubuntu 24.04 のゲスト** を用いてビルドする必要があります。

.. note::

   ホストとゲストで OS のバージョンが異なります。これがテスト済みの唯一の組み合わせです。

以下の手順は、NVIDIA の **Deployment Guide for SecureAI** をもとにしています:
https://docs.nvidia.com/cc-deployment-guide-snp.pdf

GPU Admin Tools のダウンロード
----------------------------------------

.. code-block:: bash

   cd /shared/
   git clone https://github.com/NVIDIA/gpu-admin-tools

VFIO の自動ロード
-------------------------

``/etc/modules-load.d/vfio.conf`` を以下の内容で作成します。

.. code-block:: text

   vfio
   vfio_pci

VFIO モジュールをロードするためにホストを再起動します。

.. code-block:: bash

   sudo reboot

Ubuntu インストールイメージのダウンロード
--------------------------------------------------

Ubuntu 24.04.2 の ISO ファイルをダウンロードします。

.. code-block:: bash

   cd /shared
   wget https://releases.ubuntu.com/24.04.2/ubuntu-24.04.2-live-server-amd64.iso

ドライブイメージの作成
--------------------------------

OS を格納できる十分な大きさのドライブイメージを作成します。Ubuntu と GPU ドライバーをインストールするには最低 30GB が必要です。ビルダーが必要に応じて拡張します。

.. code-block:: bash

   qemu-img create -f qcow2 /shared/ubuntu_base.qcow2 30G

Ubuntu ゲストのインストール
------------------------------------

``/shared/launch_vm.sh`` というファイルを以下の内容で作成します。

.. code-block:: bash

   #!/bin/bash

   CORES=16
   MEM=32
   VDD_IMAGE=/shared/ubuntu_base.qcow2
   FWDPORT=9899
   CDROM=/shared/ubuntu-24.04.2-live-server-amd64.iso

   doecho=false
   docc=true
   sev=""

   while getopts "exp:c:" flag
   do
     case ${flag} in
       e) doecho=true;;
       x) docc=false;;
       p) FWDPORT=${OPTARG};;
       c) sev=${OPTARG};;
     esac
   done

   NVIDIA_GPU=$(lspci -d 10de: | awk '/NVIDIA/{print $1}')
   NVIDIA_PASSTHROUGH=$(lspci -n -s $NVIDIA_GPU | awk -F: '{print $4}' | awk '{print $1}')

   if [ "$doecho" = true ]; then
     echo 10de $NVIDIA_PASSTHROUGH > /sys/bus/pci/drivers/vfio-pci/new_id
   fi

   get_cbitpos() {
       modprobe cpuid
       EBX=$(dd if=/dev/cpu/0/cpuid ibs=16 count=32 skip=134217728 | tail -c 16 | od -An -t u4 -j 4 -N 4 | sed -re 's|^ *||')
       CBITPOS=$((EBX & 0x3f))
   }

   if [ "$docc" = true ]; then
     if [ -n "$sev" ]; then
          case "$sev" in
            sev|sev-es|sev-snp)
              SEV_MODE="$sev"
              USE_CC=true
              get_cbitpos
              ;;
            *)
              echo "Error: unsupported SEV mode '$sev'."
              echo "Use '-c' with valid options: sev, sev-es, sev-snp."
              echo "Or use '-x' to boot without CC modes"
              exit 1
              ;;
          esac
        fi
   fi

   qemu-system-x86_64 \
     -bios /usr/share/ovmf/OVMF.fd \
     -nographic \
     ${USE_CC:+  -machine confidential-guest-support=sev0,vmport=off} \
     ${USE_CC:+$( [ "$SEV_MODE" = sev ] && \
      echo "  -object sev-guest,id=sev0,cbitpos=${CBITPOS},reduced-phys-bits=1,policy=0x1" )} \
     ${USE_CC:+$( [ "$SEV_MODE" = sev-es ] && \
      echo "  -object sev-guest,id=sev0,cbitpos=${CBITPOS},reduced-phys-bits=1,policy=0x5" )} \
     ${USE_CC:+$( [ "$SEV_MODE" = sev-snp ] && \
      echo "  -object sev-snp-guest,id=sev0,cbitpos=${CBITPOS},reduced-phys-bits=1,policy=0x30000" )} \
     -vga none \
     -enable-kvm -no-reboot \
     -cpu EPYC-v4 \
     -machine q35 -smp $CORES -m ${MEM}G,slots=2,maxmem=512G \
     -drive file=$VDD_IMAGE,if=none,id=disk0,format=qcow2 \
     -device virtio-scsi-pci,id=scsi0,disable-legacy=on,iommu_platform=true,romfile= \
     -device scsi-hd,drive=disk0 \
     -netdev user,id=vmnic,hostfwd=tcp::$FWDPORT-:22 \
     -cdrom $CDROM \
     -device virtio-net-pci,disable-legacy=on,iommu_platform=true,netdev=vmnic,romfile= \
     -object iommufd,id=iommufd0 \
     -device pcie-root-port,id=pci.1,bus=pcie.0 \
     -device vfio-pci,host=${NVIDIA_GPU},bus=pci.1,iommufd=iommufd0,romfile=

VM を起動して Ubuntu のインストールを開始します。

.. code-block:: bash

   chmod +x /shared/launch_vm.sh
   sudo /shared/launch_vm.sh -ex

最小構成の Ubuntu 24.04 をインストールしてください。必要なソフトウェアはすべて後でビルダーがインストールします。ゲスト OS のインストールが完了すると、Ubuntu が再起動を求めてきます。その際 VM は終了し、ホストに戻ります。

ベースイメージの保存
------------------------------

VM のイメージを ``base_images`` フォルダーにコピーします。

.. code-block:: bash

   cp /shared/ubuntu_base.qcow2 <builder_root>/base_images

ファームウェアの取得
==============================

``OVMF.amdsev.fd`` が既に ``/usr/share/ovmf`` にあるかどうかを確認します。ない場合は、Ubuntu の proposed リポジトリからインストールします。

.. code-block:: bash

   echo 'deb http://archive.ubuntu.com/ubuntu plucky-proposed main restricted universe multiverse' | \
     sudo tee /etc/apt/sources.list.d/plucky-proposed.list

   sudo tee /etc/apt/preferences.d/99-plucky-proposed <<'EOF'
   Package: *
   Pin: release a=plucky-proposed
   Pin-Priority: 100
   EOF

   sudo apt update
   sudo apt install -t plucky-proposed ovmf

ファームウェアを ``base_images`` フォルダーにコピーします。

.. code-block:: bash

   cp /usr/share/ovmf/OVMF.amdsev.fd <builder_root>/base_images

snpguest のビルド
=========================

``snpguest`` ツールは、TEE (Trusted Execution Environment) とやり取りするために必要です。

.. code-block:: bash

   # Install Rust
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   source "$HOME/.cargo/env"

   sudo apt install -y build-essential

   # Checkout source code
   git clone https://github.com/virtee/snpguest.git
   cd snpguest
   git checkout v0.9.2
   cargo build -r

   cp target/release/snpguest <builder_root>/binaries

kbs-client のビルド
===========================

``kbs-client`` ツールは Trustee とやり取りするために使用します。そのバージョンは Trustee サーバーのバージョンと一致していなければなりません。テスト済みのコミットは ``a2570329cc33daf9ca16370a1948b5379bb17fbe`` です。

.. code-block:: bash

   # Install dependencies
   sudo apt install -y pkg-config libtss2-dev

   # Checkout source code
   git clone https://github.com/confidential-containers/trustee.git
   cd trustee/tools/kbs-client
   git checkout a2570329cc33daf9ca16370a1948b5379bb17fbe

   # Build
   make -C ../../kbs cli

   cp ../../target/release/kbs-client <builder_root>/binaries
