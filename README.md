# Safira-Booter
Android Boot Menu ROM para uso de multiplas ROMs no mesmo aparelho

# Buildar o projeto
Para buildar o projeto é ideal que voce tenha o kernel do seu aparelho compatível com features específicas.

## Kernel
Primeiro de tudo voce precisa da source do kernel do seu aparelho. Você pode procurar no [XDA Forums](https://xdaforums.com/), no Github ou pedir diretamente o kernel do seu aparelho para sua fabricante. Neste tutorial vou usar o kernel do meu aparelho ([Motorola Moto G22 Hawaiip](https://github.com/ilpianista/android_kernel_motorola_hawaiip/)).
 

Baixe ou clone o kernel e descubra qual versão da toolchain do LLVM é necessária para o build:

```bash
# Rode o seguinte comando dentro do kernel
grep -Ri clang .
#./build.config.mtk.aarch64:CLANG_TRIPLE=aarch64-linux-gnu-
#./build.config.mtk.aarch64:CC=clang
#./build.config.mtk.aarch64:LD_LIBRARY_PATH=prebuilts/clang/host/linux-x86/clang-r383902/lib64:$$LD_LIBRARY_PATH
#./build.config.mtk.aarch64:CLANG_PREBUILT_BIN=prebuilts/clang/host/linux-x86/clang-r383902/bin
```

Neste meu caso, a versão do clang é `clang-r383902`, então apenas procurei por `clang-r383902 AOSP download` e baixei o ultimo commit do repositório do android:

```bash
git clone https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/ --depth 1
```

Após isso coloque a pasta `bin/` da versão correta do clang do repositório baixado no `PATH`.

Baixe também a toolchain 4.9 para `aarch64` [neste repo](https://github.com/KudProject/aarch64-linux-android-4.9/) e adicione a pasta `bin/` ao `PATH` também.

O próximo passo é voltar ao diretório do kernel e configurar o build:

```bash
# Gere o arquivo .config
make ARCH=arm64 k6877v1_64_defconfig
```
> ATENÇÃO: Eu usei a config `k6877v1_64_defconfig` pois ELA é compatível com o MEU chip Mediatek. Pesquise sobre qual é compativel com o seu e mude o nome para configurar.

Configurar o build:
```bash
# Habilitar e desabilitar features
scripts/config \
--enable CONFIG_KEXEC \
--enable CONFIG_KEXEC_FILE \
--enable CONFIG_KEXEC_CORE \
--enable CONFIG_CRASH_DUMP \
--enable CONFIG_RELOCATABLE \
--enable CONFIG_RANDOMIZE_BASE \
--enable CONFIG_ARM64_CRASH_DUMP \
--enable CONFIG_PHYS_ADDR_T_64BIT \
--enable CONFIG_ATAGS \
--disable CONFIG_SECURITY \
--disable CONFIG_SECURITY_SELINUX \
--disable CONFIG_SECURITY_SELINUX_BOOTPARAM \
--disable CONFIG_DM_VERITY \
--disable CONFIG_DM_ANDROID_VERITY \
--disable CONFIG_FS_VERITY \
--disable CONFIG_SECURITYFS \
--enable CONFIG_BLK_DEV_INITRD \
--enable CONFIG_RD_GZIP \
--enable CONFIG_RD_LZ4 \
--enable CONFIG_RD_XZ \
--enable CONFIG_DEVTMPFS \
--enable CONFIG_DEVTMPFS_MOUNT \
--enable CONFIG_EXT4_FS \
--enable CONFIG_F2FS_FS \
--enable CONFIG_VFAT_FS \
--enable CONFIG_TMPFS \
--enable CONFIG_OVERLAY_FS \
--enable CONFIG_PRINTK \
--enable CONFIG_EARLY_PRINTK \
--enable CONFIG_DEBUG_KERNEL \
--enable CONFIG_DEBUG_INFO \
--enable CONFIG_MAGIC_SYSRQ \
--enable CONFIG_SYSVIPC \
--enable CONFIG_POSIX_MQUEUE \
--enable CONFIG_CGROUPS \
--enable CONFIG_NAMESPACES \
--enable CONFIG_MEMCG \
--enable CONFIG_BLK_DEV_LOOP \
--enable CONFIG_DM_MAPPER \
--enable CONFIG_DM_CRYPT \
--enable CONFIG_BINFMT_MISC \
--disable CONFIG_MTK_SEC_VIDEO_PATH_SUPPORT \
--disable CONFIG_MTK_IN_HOUSE_TEE_SUPPORT
```
Sobre o que essas configs fazem:

| Config                            | Estado     | O que faz                                                             |
| --------------------------------- | ---------- | --------------------------------------------------------------------- |
| CONFIG_KEXEC                      | ✅ Enabled  | Permite carregar outro kernel diretamente via syscall `kexec`.        |
| CONFIG_KEXEC_FILE                 | ✅ Enabled  | Variante moderna do kexec usando arquivos kernel assinados.           |
| CONFIG_KEXEC_CORE                 | ✅ Enabled  | Infraestrutura interna necessária para kexec funcionar.               |
| CONFIG_CRASH_DUMP                 | ✅ Enabled  | Suporte a dump de memória após crash; usado também pelo kexec.        |
| CONFIG_RELOCATABLE                | ✅ Enabled  | Permite kernel rodar em endereços diferentes (essencial p/ kexec).    |
| CONFIG_RANDOMIZE_BASE             | ✅ Enabled  | KASLR (randomização do kernel na RAM).                                |
| CONFIG_ARM64_CRASH_DUMP           | ✅ Enabled  | Crash dump específico para arquitetura ARM64.                         |
| CONFIG_PHYS_ADDR_T_64BIT          | ✅ Enabled  | Endereçamento físico 64-bit (necessário ARM64 moderno).               |
| CONFIG_ATAGS                      | ✅ Enabled  | Compatibilidade antiga de boot params (pode ajudar em kexec Android). |
| CONFIG_SECURITY                   | ❌ Disabled | Desativa framework geral de segurança (LSM).                          |
| CONFIG_SECURITY_SELINUX           | ❌ Disabled | Remove SELinux do kernel.                                             |
| CONFIG_SECURITY_SELINUX_BOOTPARAM | ❌ Disabled | Remove controle SELinux via boot params.                              |
| CONFIG_DM_VERITY                  | ❌ Disabled | Desativa dm-verity (integridade de partições Android).                |
| CONFIG_DM_ANDROID_VERITY          | ❌ Disabled | Implementação Android específica do dm-verity.                        |
| CONFIG_FS_VERITY                  | ❌ Disabled | Verificação criptográfica em nível filesystem.                        |
| CONFIG_SECURITYFS                 | ❌ Disabled | Filesystem usado por módulos de segurança.                            |
| CONFIG_BLK_DEV_INITRD             | ✅ Enabled  | Permite uso de initramfs/ramdisk (boot.img Android).                  |
| CONFIG_RD_GZIP                    | ✅ Enabled  | Suporte a ramdisk gzip.                                               |
| CONFIG_RD_LZ4                     | ✅ Enabled  | Suporte a ramdisk LZ4 (muito comum Android).                          |
| CONFIG_RD_XZ                      | ✅ Enabled  | Suporte a ramdisk XZ.                                                 |
| CONFIG_DEVTMPFS                   | ✅ Enabled  | Criação automática de `/dev` pelo kernel.                             |
| CONFIG_DEVTMPFS_MOUNT             | ✅ Enabled  | Monta `/dev` automaticamente no boot.                                 |
| CONFIG_EXT4_FS                    | ✅ Enabled  | Filesystem EXT4 (system/vendor Android).                              |
| CONFIG_F2FS_FS                    | ✅ Enabled  | Filesystem otimizado para flash (comum Android).                      |
| CONFIG_VFAT_FS                    | ✅ Enabled  | FAT/VFAT para partições misc/boot/media.                              |
| CONFIG_TMPFS                      | ✅ Enabled  | Filesystem temporário em RAM (usado pelo init Android).               |
| CONFIG_OVERLAY_FS                 | ✅ Enabled  | Overlay filesystem (mods, root, multiROM etc.).                       |
| CONFIG_PRINTK                     | ✅ Enabled  | Logs kernel (`dmesg`).                                                |
| CONFIG_EARLY_PRINTK               | ✅ Enabled  | Logs early boot (debug bootloop).                                     |
| CONFIG_DEBUG_KERNEL               | ✅ Enabled  | Opções gerais de debug kernel.                                        |
| CONFIG_DEBUG_INFO                 | ✅ Enabled  | Símbolos debug para análise.                                          |
| CONFIG_MAGIC_SYSRQ                | ✅ Enabled  | SysRq (comandos debug emergenciais).                                  |
| CONFIG_SYSVIPC                    | ✅ Enabled  | IPC System V (usado por libs Android).                                |
| CONFIG_POSIX_MQUEUE               | ✅ Enabled  | Filas POSIX IPC (dependência Android userspace).                      |
| CONFIG_CGROUPS                    | ✅ Enabled  | Control groups (Android usa para gerenciamento processos).            |
| CONFIG_NAMESPACES                 | ✅ Enabled  | Namespaces estilo container Linux.                                    |
| CONFIG_MEMCG                      | ✅ Enabled  | Controle de memória por cgroup.                                       |
| CONFIG_BLK_DEV_LOOP               | ✅ Enabled  | Loop devices (montar imagens ROM).                                    |
| CONFIG_DM_MAPPER                  | ✅ Enabled  | Device mapper (base criptografia/storage virtual).                    |
| CONFIG_DM_CRYPT                   | ✅ Enabled  | Criptografia de storage Android.                                      |
| CONFIG_BINFMT_MISC                | ✅ Enabled  | Executar binários via handlers especiais.                             |
| CONFIG_MTK_SEC_VIDEO_PATH_SUPPORT | ❌ Disabled | Remove restrições TrustZone vídeo MediaTek.                           |
| CONFIG_MTK_IN_HOUSE_TEE_SUPPORT   | ❌ Disabled | Desativa hooks TEE proprietários MediaTek.                            |


```bash
# Resolver dependencias automaticamente
make olddefconfig
```

E então builde o projeto para gerar o Image.gz em arch/arm64/boot/
```bash
make ARCH=arm64 SUBARCH=arm64 CC=clang CLANG_TRIPLE=aarch64-linux-gnu- CROSS_COMPILE=aarch64-linux-android- LLVM=1 Image.gz -j$(nproc)
```

> ATENÇÃO: Pode acontecer de você precisar resolver erros de build manualmente dependendo da versão do kernel baixado, pois podem conter falta de drivers ou funções indefinidas ao compilar. Para corrigir, o jeito mais fácil e rápido é usar IA pra te ajudar. Copie e cole os erros e siga os passos dados pela IA até gerar o arquivo `Image.gz`.

## Repack + Magisk

Agora é preciso pegar o arquivo `boot.img` original da sua ROM (procure a ROM da sua versão de build do seu aparelho) e desempacotá-lo, inserir o kernel novo, empacotar, testar para ver se não há erros de bootloop, e então passar o `boot.img` para o magisk e rootar o aparelho.

### Empacotando o kernel novo
Baixe o APK do [Magisk](https://github.com/topjohnwu/Magisk) e dê unzip no APK. Entre no diretório `lib/arm64-v8a` e após isso:

```bash
# Renomeie o magiskboot
mv libmagiskboot.so magiskboot

# Dê permissões de execução
chmod +x magiskboot

# Envie o arquivo para seu aparelho via adb
adb push magiskboot /data/local/tmp
```

> ANTEÇÃO: é de extrema importância enviar para `/data/local/tmp` pois outros diretórios não vão te deixar executar o magiskboot por segurança.

Agora, dê push no arquivo `Image.gz` (este é seu kernel compilado) e no seu `boot.img` da sua stock ROM para `/sdcard/Download` e entre na shell do android.

```bash
$ adb shell
hawaiip:/ $ cd sdcard/Download
hawaiip:/ $ ls
boot.img Image.gz

# Desempacote o boot.img
hawaiip:/ $ /data/local/tmp/magiskboot unpack boot.img

# Remova o kernel original e mude o nome do Image.gz para "kernel"
hawaiip:/ $ rm kernel && mv Image.gz kernel

# Empacote novamente
hawaiip:/ $ /data/local/tmp/magiskboot repack boot.img
```
Ao fim do processo, será gerado o arquivo `new-boot.img`, dê pull nele com o adb e coloque o celular em modo fasboot. Após isso dê boot no kernel novo com `fastboot boot boot.img`. Se o aparelho iniciar normalmente, deu tudo certo!
No meu caso, o boot falha, então eu uso o [MTKClient](https://github.com/bkerler/mtkclient) e faço flash direto do boot.img em boot_a (`python3 mtk.py w boot_a new-boot.img`).

> ATENÇÃO: Antes de flashear o kernel guarde o `boot.img` orginal da ROM para caso o sistema travar em bootloop voce consiga flashear ele novamente via fastboot ou MTKClient.

### Magisk

Se até aqui tudo correu bem (empacotou o kernel novo e iniciou normalmente) você pode continuar. Para isso baixe no seu aparelho o APK do magisk e dentro dele dê patch no kernel novo (`new-boot.img`) para ter permissões de root. Em seguida dê boot ou flash no arquivo corrigido e verifique se as permissões de root foram devidamente concebidas. 

> ATENÇÃO: Novamente, de flashear o kernel com root guarde o `boot.img` orginal da ROM para caso o sistema travar em bootloop voce consiga flashear ele novamente via fastboot ou MTKClient.
