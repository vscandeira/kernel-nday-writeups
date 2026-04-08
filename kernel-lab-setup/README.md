# Kernel Lab Setup

## English version

#### TLDR

## Versão em português

#### TLDR
Para pular toda a discussão, segue abaixo um quase script bash. A sugestão é rodar linha por linha, pois é mais um registro do passo a passo do que um script pensado para tornar o processo totalmente automático. A etapa de ajustes das flags, por exemplo, é feita via menuconfig. A sequência abaixo cria uma árvore de diretórios para o kernel_lab, clona o git do código fonte do kernel, compila, cria um initrd básico e roda via QEMU (sem debug) com compartilhamento de diretório. Assume-se um host system linux.

```bash
# as 4 linhas abaixo podem ser incluidas no ~/.bashrc para maior facilidade
LINUX_VER="6.1"
KERNEL_LAB="~/kernel_lab"
PATH_KERNEL="$KERNEL_LAB/$LINUX_VER"
MODE_INITRD="$KERNEL_LAB/fs_user/initramfs.cpio.gz"

#instalacao foi pensada para sistemas debian-based, mas pode ser adaptada
sudo apt update && sudo apt install qemu-system-x86 qemu-utils build-essential gcc make gdb gdb-multiarch gcc-multilib libncurses-dev flex bison wget curl git busybox-static clang llvm lld libelf-dev libssl-dev dwarves cpio libncurses-dev libusb-dev

# criar um diretorio especifico para projetos git e trabalhar nele
mkdir -p ~/gits

# criar diretorio sugerido na secao de diretorios
mkdir -p $KERNEL_LAB
mkdir -p $KERNEL_LAB/6.1
mkdir -p $KERNEL_LAB/6.12
mkdir -p $KERNEL_LAB/5.10
mkdir -p $KERNEL_LAB/shared
mkdir -p $KERNEL_LAB/fs_root
mkdir -p $KERNEL_LAB/fs_user

# clonar repo
cd ~/gits
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux

# para versao mais atual
git checkout linux-$LINUX_VER.y
git describe
make kernelversion

# para versao de lancamento
# git checkout v$LINUX_VER
# git describe

# editar flags e compilacao
make clean
make mrproper
make defconfig
make menuconfig
grep -iE "dwarf4|gdp_script|kasan|slub|9p" ./.config
#make -j$(nproc) KCFLAGS="-Wno-error=format"
make -j8 KCFLAGS="-Wno-error=format"
ls -lh vmlinux
cp ./vmlinux $PATH_KERNEL/
ls -lh arch/x86/boot/bzImage
cp arch/x86/boot/bzImage $PATH_KERNEL/
cp .config $PATH_KERNEL/

# Initrd basico root
cd $KERNEL_LAB
mkdir -p fs_root && cd fs_root
mkdir -p initramfs
cd initramfs
mkdir -p bin sbin etc proc sys usr/bin usr/sbin dev
cp /bin/busybox* bin/
cd bin
for cmd in $(./busybox --list); do ln -s busybox $cmd done
rm '[' && rm '[['
$ cd ..
vim init
#incluir o conteúdo abaixo
    #!/bin/sh
    mount -t proc none /proc
    mount -t sysfs none /sys
    # montagem automatica do diretorio shared
    mkdir -p /mnt/host
    mount -t 9p -o trans=virtio hostshare /mnt/host
    echo "Booted into minimal initramfs!"
    echo "Dropping into root shell..."
    exec /bin/sh
chmod +x init
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
ls -lh ../initramfs.cpio.gz

#Initrd basico common user
cd $KERNEL_LAB/
cp -R fs_root/* fs_user/
cd fs_user
cd initramfs/bin
chmod u+s busybox
cd ..
vim etc/passwd
#incluir
    root:x:0:0:root:/root:/bin/sh
    user:x:1000:1000:user:/home/user:/bin/sh
vim etc/shadow
#incluir
#root:$6$lab$YB439mnO/ohBy6IoxOYRp66jviRRHrD0a7E2x.Zi2J4b4fsSAGj.107RMdMhnZnTuifdE/wO755hkEyk8ZKKY1:19500:0:99999:7:::
#user:$6$blabla$etaIfTVD8PXTKnCNj67ojq.tHxh20EvhARAWaLUP6zKpRdyPYzFmqFqChQoxJ.6a.VzBcFSl4HTtkE4gtrpoU/:19500:0:99999:7:::
vim etc/group
#incluir
#root:x:0:
#user:x:1000:
mkdir -p home/user
chown -R 1000:1000 home/user
vim init
#incluir o conteúdo abaixo
    #!/bin/sh
    mount -t proc none /proc
    mount -t sysfs none /sys
    echo "Booted into minimal initramfs!"
    # montagem automatica do diretorio shared
    mkdir -p /mnt/host
    mount -t 9p -o trans=virtio hostshare /mnt/host

    # para testes com senhas de root e outros users
    chmod 600 /etc/shadow
    chown root:root /etc/shadow

    # cria device nodes básicos
    mount -t devtmpfs devtmpfs /dev
    # garante permissões
    chown 1000:1000 /home/user

    echo "Dropping to unprivileged user..."
    exec setsid cttyhack su user

# rodar sem debug
qemu-system-x86_64 -kernel $PATH_KERNEL/bzImage -initrd $MODE_INITRD -nographic -append "console=ttyS0 nokaslr root=/dev/ram" -m 2G -cpu qemu64 -fsdev local,id=fsdev0,path=$KERNEL_LAB/shared,security_model=none -device virtio-9p-pci,fsdev=fsdev0,mount_tag=hostshare
```

Para rodar com debug, basta acrescentar as flags "-s -S", mais detalhes na seção **QEMU**.

#### Hora de fazer escolhas
Para início de conversa, vou assumir que a importância de criar seu próprio laboratório localmente é um ponto pacífico e vou pular essa discussão. Há algumas opções para construção de um laboratório para pesquisa em kernel exploits. As principais opções que consegui pensar na época de construção do lab foram:
1. Virtualizar imagens de distribuições linux em virtualizadores como VirtualBox e VmWare.
2. Compilar manualmente kernel linux, subir um initrd mínimo junto com o executável no QEMU.

Cada uma dessas opções tem suas vantagens e desvantagens e provavelmente há nuances em cada uma que eu, como mero iniciante, nem desconfiava na época. Basicamente a primeira opção tem a vantagem de simular um sistema completo, mais próximo do cenário real de ataque. A segunda opção tem a vantagem de oferecer mais controle e facilidade para transitar entre versões do kernel.

Como toda decisão por tecnologia específica depende dos objetivos e das necessidades em questão, estabeleci para mim que meus objetivos eram reproduzir N-days (como o nome do repositório já denunciou) e estudar o que já foi feito para futuramente poder começar a procurar meus próprios bugs de kernel. Para isso, é mais importante transitar facilmente entre versões de kernel que simular um cenário real com sistema completo. Então, opção 2.

Elenquei 3 versões de kernel em que queria focar: 5.10, 6.1 e 6.12. Todas elas estavam, na data de construção do lab (03/2026), com suporte ativo do [kernel.org](https://www.kernel.org/category/releases.html). Além disso, as 3 versões escolhidas também possuem suporte estendido pelo [Civil Infrastructure Platform (CIP)](https://wiki.linuxfoundation.org/civilinfrastructureplatform/start#kernel_maintainership) e, portanto, estarão vivas entre nós por um bom tempo.
+ **5.10**: versão com suporte do kernel.org até 12/2026 e suporte estendido pelo CIP até 01/2031. A ideia dessa versão é procurar por exploits mais antigos e bem documentados, com mais facilidade de exploração. Foco didático.
+ **6.1**: versão com suporte do kernel.org até 12/2027 e suporte estendido pelo CIP até 08/2033. Meio termo entre ambas as versões.
+ **6.12**: versão com suporte do kernel.org até 12/2028 e suporte estendido pelo CIP até 06/2035. A ideia dessa versão é procurar por exploits mais recentes. Foco em aprender o que mais tem sido feito na pesquisa de kernel research atualmente.

#### Organização de diretórios
Como estou trabalhando com 3 versões de kernel diferentes, criei um diretório separado do fonte, kernel_lab, e dentro dele criei as pastas para cada versão, uma pasta shared para arquivos compartilhados entre host e guest system e duas pastas para initrd que cai no root e outra para um initrd que cai num user comum. A estrutura final ficou mais ou menos assim:
```
kernel_lab
├── 5.10
│   ├── cve-2022-0847_dirty_pipe
│   └── initial_version
├── 6.1
│   └── initial_version
├── 6.12
│   └── initial_version
├── fs_root
│   └── initramfs
│       ├── bin
│       ├── dev
│       ├── etc
│       ├── proc
│       ├── sbin
│       ├── sys
│       └── usr
│           ├── bin
│           └── sbin
├── fs_user
│   └── initramfs
│       ├── bin
│       ├── dev
│       ├── etc
│       ├── home
│       │   └── user
│       ├── proc
│       ├── sbin
│       ├── sys
│       └── usr
│           ├── bin
│           └── sbin
└── shared
```

Você não precisa fazer o mesmo que eu. Mas se optar por seguir essa estrutura, as seções de comandos bash abaixo já criam essa estrutura. Caso opte por não segui-la, os comandos terão que ser adaptados.

#### Código fonte do kernel linux
Um vez que decidimos nosso caminho, nossa organização e nossas versões, mãos à obra :)

Caso você seja um novato como eu, recomendo fortemente um pouco de leitura antes de partir pra ação:
+ [kernel.org - How the development proccess works](https://www.kernel.org/doc/html/latest/process/2.Process.html#how-the-development-process-works)
+ [kernel.org - CVE process](https://docs.kernel.org/process/cve.html)

O git que nos interessa é o [git do próprio Linus](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git). O processo geral consiste em: clonar o git, selecionar a versão, utilizar o utilitário do make para editar as flags importantes para nós, compilar e salvar vmlinux e bzImage em pastas separadas.

+ Flags de interesse:
    + CONFIG_DEBUG_INFO_DWARF4=y
        + **Versão 6.1**: Kernel hacking > Compile-time checks and compiler options > Compile the kernel with debug info > Debug information
        + **Versão 5.10**: Kernel hacking > Compile-time checks and compiler options > Compile the kernel with debug info
        + **Versão 5.10**: Kernel hacking > Compile-time checks and compiler options > Generate dwarf4 debuginfo
        + mais compatível que o dwarf5
    + CONFIG_GDB_SCRIPTS=y
        + Kernel hacking > Compile-time checks and compiler options > Provide GDB scripts for kernel debugging
    + CONFIG_KASAN=y
        + **Versão 6.12**: Kernel hacking > Memory Debugging > KASAN: dynamic memory safety error detector
        + **Versão 6.1**: Kernel hacking > Memory Debugging > KASAN: Kernel Address Sanitizer
        + **Versão 5.10**: Kernel hacking > Memory Debugging > KASAN: runtime memory debugger
    + CONFIG_KASAN_INLINE=y
        + **Versão 6.1**: Kernel hacking > Memory Debugging > KASAN: Kernel Address Sanitizer > Instrumentation type
        + **Versão 5.10**: Kernel hacking > Memory Debugging > KASAN: runtime memory debugger > Instrumentation type
    + CONFIG_SLUB
        + Kernel hacking > Memory Debugging > SLUB debugging on by default
            + no 6.12 só é ativado depois de marcar o KASAN
    + CONFIG_NET_9P e CONFIG_NET_9P_VIRTIO
        + **Todos**: File systems > Network File Systems > Plan 9 Resource Sharing Support (9P2000)
        + **Todos**: File systems > Network File Systems > 9P POSIX Access Control Lists
        + **Todos**: Device Drivers > Virtio drivers > PCI driver for virtio devices
        + **Todos**: Device Drivers > Virtio drivers >  Support for legacy virtio draft 0.9.X and older devices (NEW)
        + **Todos**: Networking support > Plan 9 Resource Sharing Support (9P2000)
        + **Todos**: Networking support > Plan 9 Resource Sharing Support (9P2000) > 9P Virtio Transport
+ Um quase script (comandos bash):
    ```bash
    # as 4 linhas abaixo podem ser incluidas no ~/.bashrc para maior facilidade
    LINUX_VER="6.1"
    KERNEL_LAB="~/kernel_lab"
    PATH_KERNEL="$KERNEL_LAB/$LINUX_VER"
    MODE_INITRD="$KERNEL_LAB/fs_user/initramfs.cpio.gz"

    #instalacao foi pensada para sistemas debian-based, mas pode ser adaptada
    sudo apt update && sudo apt install qemu-system-x86 qemu-utils build-essential gcc make gdb gdb-multiarch gcc-multilib libncurses-dev flex bison wget curl git busybox-static clang llvm lld libelf-dev libssl-dev dwarves cpio libncurses-dev libusb-dev

    # criar um diretorio especifico para projetos git e trabalhar nele
    mkdir -p ~/gits

    # criar diretorio sugerido na secao de diretorios
    mkdir -p $KERNEL_LAB
    mkdir -p $KERNEL_LAB/6.1
    mkdir -p $KERNEL_LAB/6.12
    mkdir -p $KERNEL_LAB/5.10
    mkdir -p $KERNEL_LAB/shared
    mkdir -p $KERNEL_LAB/fs_root
    mkdir -p $KERNEL_LAB/fs_user

    # clonar repo
    cd ~/gits
    git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
    cd linux

    # para versao mais atual
    git checkout linux-$LINUX_VER.y
    git describe
    make kernelversion

    # para versao de lancamento
    # git checkout v$LINUX_VER
    # git describe

    # editar flags e compilacao
    make clean
    make mrproper
    make defconfig
    make menuconfig
    grep -iE "dwarf4|gdp_script|kasan|slub|9p" ./.config
    #make -j$(nproc) KCFLAGS="-Wno-error=format"
    make -j8 KCFLAGS="-Wno-error=format"
    ls -lh vmlinux
    cp ./vmlinux $PATH_KERNEL/
    ls -lh arch/x86/boot/bzImage
    cp arch/x86/boot/bzImage $PATH_KERNEL/
    cp .config $PATH_KERNEL/
    ```

+ É importante ressaltar que para a versão 5.10, compilada com GCC-10, só consegui compilar subindo um docker com debian mais antigo e compilando de lá. `docker run --name kernel-builder -it -v ~/gits/linux:/src debian:bullseye bash`
    ```bash
    # Dentro do container, você instala as dependências:
    apt update && apt install -y build-essential gcc-10 libncurses-dev flex bison libssl-dev libelf-dev bc git
    # Dentro do container, navegue até a pasta montada:
    cd /src
    make clean
    make mrproper CC=gcc-10 HOSTCC=gcc-10
    #se nao funcionar, usar o comando abaixo que faz uma limpeza mais profunda
    #git clean -fdx
    make defconfig CC=gcc-10 HOSTCC=gcc-10
    make menuconfig
    make CC=gcc-10 HOSTCC=gcc-10 olddefconfig
    grep -iE "dwarf4|gdp_script|kasan|slub|9p" ./.config
    #make CC=gcc-10 HOSTCC=gcc-10 -j$(nproc)
    make CC=gcc-10 HOSTCC=gcc-10 -j8
    # De volta ao host system 
    ls -lh vmlinux
    cp ./vmlinux $PATH_KERNEL/
    ls -lh arch/x86/boot/bzImage
    cp arch/x86/boot/bzImage $PATH_KERNEL/
    cp .config $PATH_KERNEL/
    ```

#### Initrd basico
Nao vou discutir todas opções possiveis e imagináveis para se construir um initrd minimalista. Pedi ajuda para meu amigo LLM e ele me indicou utilizar o busybox e funcionou bem para os meus propósitos.
+ Um quase script para criar initrd root basico (comandos bash):
    ```bash
    cd $KERNEL_LAB
    mkdir -p fs_root && cd fs_root
    mkdir -p initramfs
    cd initramfs
    mkdir -p bin sbin etc proc sys usr/bin usr/sbin dev
    cp /bin/busybox* bin/
    cd bin
    for cmd in $(./busybox --list); do ln -s busybox $cmd done
    rm '[' && rm '[['
    $ cd ..
    vim init
    #incluir o conteúdo abaixo
        #!/bin/sh
        mount -t proc none /proc
        mount -t sysfs none /sys
        # montagem automatica do diretorio shared
        mkdir -p /mnt/host
        mount -t 9p -o trans=virtio hostshare /mnt/host
        echo "Booted into minimal initramfs!"
        echo "Dropping into root shell..."
        exec /bin/sh
    chmod +x init
    find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
    ls -lh ../initramfs.cpio.gz
    ```

+ Um quase script para criar um initrd common user basico (comandos bash):
    ```bash
    cd $KERNEL_LAB/
    cp -R fs_root/* fs_user/
    cd fs_user
    cd initramfs/bin
    chmod u+s busybox
    cd ..
    vim etc/passwd
    #incluir
        root:x:0:0:root:/root:/bin/sh
        user:x:1000:1000:user:/home/user:/bin/sh
    vim etc/shadow
    #incluir
    #root:$6$lab$YB439mnO/ohBy6IoxOYRp66jviRRHrD0a7E2x.Zi2J4b4fsSAGj.107RMdMhnZnTuifdE/wO755hkEyk8ZKKY1:19500:0:99999:7:::
    #user:$6$blabla$etaIfTVD8PXTKnCNj67ojq.tHxh20EvhARAWaLUP6zKpRdyPYzFmqFqChQoxJ.6a.VzBcFSl4HTtkE4gtrpoU/:19500:0:99999:7:::
    vim etc/group
    #incluir
    #root:x:0:
    #user:x:1000:
    mkdir -p home/user
    chown -R 1000:1000 home/user
    vim init
    #incluir o conteúdo abaixo
        #!/bin/sh
        mount -t proc none /proc
        mount -t sysfs none /sys
        echo "Booted into minimal initramfs!"
        # montagem automatica do diretorio shared
        mkdir -p /mnt/host
        mount -t 9p -o trans=virtio hostshare /mnt/host

        # para testes com senhas de root e outros users
        chmod 600 /etc/shadow
        chown root:root /etc/shadow

        # cria device nodes básicos
        mount -t devtmpfs devtmpfs /dev
        # garante permissões
        chown 1000:1000 /home/user

        echo "Dropping to unprivileged user..."
        exec setsid cttyhack su user
    ```
+ Caso necessário para seus testes, as senhas incluídas para user e root são:
    + root: `youcantguessme123!`
    + user: `test123`

#### QEMU
Uma vez com os executáveis linux em mãos, o comprimido e o com símbolos, é hora de rodar com os comandos abaixo.
+ Um quase script (comandos bash para rodar):
    ```bash
    # rodar sem debug
    qemu-system-x86_64 -kernel $PATH_KERNEL/bzImage -initrd $MODE_INITRD -nographic -append "console=ttyS0 nokaslr root=/dev/ram" -m 2G -cpu qemu64 -fsdev local,id=fsdev0,path=$KERNEL_LAB/shared,security_model=none -device virtio-9p-pci,fsdev=fsdev0,mount_tag=hostshare

    # rodar com debug
    qemu-system-x86_64 -kernel $PATH_KERNEL/bzImage -initrd $MODE_INITRD -nographic -append "console=ttyS0 nokaslr root=/dev/ram" -m 2G -cpu qemu64 -fsdev local,id=fsdev0,path=$KERNEL_LAB/shared,security_model=none -device virtio-9p-pci,fsdev=fsdev0,mount_tag=hostshare -s -S
    # apos isso, a inicializacao fica trava. em outro terminal, no diretorio do linux
    gdb --nx $PATH_KERNEL/vmlinux
    # o nx tem a finalidade de bypassar o gef caso instalado, a vida e mais facil no kernel debug usando so o gdb, vai por mim
    ```
+ no gdb
    ```
    (gdb) target remote :1234
    #comando abaixo e opcional
    (gdb) b start_kernel
    #comando abaixo equivale a "continue", ele que vai permitir seguir com o boot
    (gdb) c
    ```

Para encerrar um emulador QEMU aberto há algumas opções. A mais limpa para mim é utilizar os atalhos `Ctrl+A` seguido de `x`. **Aviso:** ao fechar um emulador QEMU, muitas vezes meu terminal fica meio ruim de utilizar, por isso eu geralmente fecho-o e abro outro para seguir minhas atividades no host system.

#### Considerações finais
O processo pode ser repetido para compilar diferentes versões e a versão de lançamento de cada minor version. A primeira falha reproduzida após construção desse lab foi a cve-2022-0847. Lá consta, também, algumas dicas de como utilizar o gdb. Caso seja um iniciante como eu, recomendo começar por lá.

+ Referências:
    + [kernel.org/releases](https://www.kernel.org/category/releases.html)
    + [kernel.org - How the development proccess works](https://www.kernel.org/doc/html/latest/process/2.Process.html#how-the-development-process-works)
    + [kernel.org - CVE process](https://docs.kernel.org/process/cve.html)
    + [CIP - Civil Infrastructure Platform - Kernel Maintainership](https://wiki.linuxfoundation.org/civilinfrastructureplatform/start#kernel_maintainership)
    + [Linux Kernel Source Code](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)

