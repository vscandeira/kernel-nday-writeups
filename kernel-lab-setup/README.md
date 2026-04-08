# Kernel Lab Setup

## English version

#### TLDR

## Versão em português

#### TLDR
Para pular toda a discussão, segue abaixo um quase script bash para rodar. A sugestão é rodar linha por linha, pois é mais um registro do passo a passo do que um script pensado para tornar o processo totalmente automático. A etapa de ajustes das flags, por exemplo, é feita via menuconfig. A sequência abaixo cria uma árvore de diretórios para o kernel_lab, clona o git do código fonte do kernel, compila, cria um initrd básico e roda via QEMU (com e sem debug via gdb) com compartilhamento de diretório.

```bash
```

#### Hora de fazer escolhas
Para início de conversa, vou assumir que a importância de criar seu próprio laboratório localmente é um ponto pacífico e vou pular essa discussão. Há algumas opções para construção de um laboratório para pesquisa em kernel exploits. As principais opções que consegui pensar na época de construção do lab são:
1. Virtualizar imagens de distribuições linux em virtualizadores como VirtualBox e VmWare.
2. Compilar manualmente kernel linux, subir um initrd mínimo junto com o executável no QEMU.

Cada uma dessas opções tem suas vantagens e desvantagens e provavelmente há nuances em cada uma que eu, como mero iniciante, nem desconfiava na época. Basicamente a primeira opção tem a vantagem de simular um sistema completo, mais próximo do cenário real de ataque. A segunda opção tem a vantagem de oferecer mais controle e facilidade para transitar entre versões do kernel.

Como toda decisão por tecnologia específica depende dos objetivos e das necessidades em questão, estabeleci para mim que meus objetivos eram reproduzir N-days (como o nome do repositório já denunciou) e estudar o que já foi feito para futuramente poder começar a procurar meus próprios bugs de kernel. Para isso, é mais importante transitar facilmente entre versões de kernel que simular um cenário real com sistema completo. Então, opção 2.

Elenquei 3 versões de kernel em que queria focar: 5.10, 6.1 e 6.12. Todas elas estavam, na data de construção do lab (03/2026), com suporte ativo do [kernel.org](https://www.kernel.org/category/releases.html). Além disso, as 3 versões escolhidas também possuem suporte estendido pelo [Civil Infrastructure Platform (CIP)](https://wiki.linuxfoundation.org/civilinfrastructureplatform/start#kernel_maintainership) e, portanto, estarão vivas entre nós por um bom tempo.
+ **5.10**: versão com suporte do kernel.org até 12/2026 e suporte estendido pelo CIP até 01/2031. A ideia dessa versão é procurar por exploits mais antigos e bem documentados, com mais facilidade de exploração. Foco didático.
+ **6.1**: versão com suporte do kernel.org até 12/2027 e suporte estendido pelo CIP até 08/2033. Meio termo entre ambas as versões.
+ **6.12**: versão com suporte do kernel.org até 12/2028 e suporte estendido pelo CIP até 06/2035. A ideia dessa versão é procurar por exploits mais recentes. Foco em aprender o que mais tem sido feito na pesquisa de kernel research atualmente.

#### Código fonte do kernel linux
Um vez que decidimos nosso caminho e nossas versões, mãos à obra :)

Caso você seja um novato como eu, recomendo fortemente um pouco de leitura antes de partir pra ação:
+ [kernel.org - How the development proccess works](https://www.kernel.org/doc/html/latest/process/2.Process.html#how-the-development-process-works)
+ [kernel.org - CVE process](https://docs.kernel.org/process/cve.html)

O git que nos interessa é o [git do próprio Linus](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git), até hoje mantido diretamente por ele. O processo geral consiste em: clonar o git, selecionar a versão, utilizar o utilitário do make para editar as flags importantes para nós, compilar e por fim (opcional) salvar vmlinux e bzImage em pastas separadas. Caso você opte por seguir a mesma árvore de diretórios que eu, dê uma olhada na seção de organização de diretórios antes de seguir.

+ Flags de interesse:

+ Um quase script (comandos bash para rodar):
    ```bash
    ```

#### QEMU
Uma vez com os executáveis linux em mãos, o comprimido e o com símbolos, é hora de rodar com os comandos abaixo.
+ Um quase script (comandos bash para rodar):
    ```bash
    ```

Para encerrar um emulador QEMU aberto há algumas opções. A mais limpa para mim é utilizar os atalhos `Ctrl+A` seguido de `x`. **Aviso:** ao fechar um emulador QEMU, muitas vezes meu terminal fica meio ruim de utilizar, por isso eu geralmente fecho-o e abro outro para seguir minhas atividades no host system.


#### Organização de diretórios
Como estou trabalhando com 3 versões de kernel diferentes, criei um diretório separado do fonte, kernel_lab e dentro dele criei as pastas para cada versão, uma pasta shared para arquivos compartilhados e duas pastas para initrd que cai no root e outra para um initrd que cai num user comum. A estrutura final ficou assim:
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

#### Considerações finais

+ Referências:
    + [kernel.org/releases](https://www.kernel.org/category/releases.html)
    + [kernel.org - How the development proccess works](https://www.kernel.org/doc/html/latest/process/2.Process.html#how-the-development-process-works)
    + [kernel.org - CVE process](https://docs.kernel.org/process/cve.html)
    + [CIP - Civil Infrastructure Platform - Kernel Maintainership](https://wiki.linuxfoundation.org/civilinfrastructureplatform/start#kernel_maintainership)
    + [Linux Kernel Source Code](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git)

