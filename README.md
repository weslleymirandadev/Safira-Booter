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
make defconfig

# Habilitar e desabilitar features
scripts/config \
--enable CONFIG_KEXEC \                    # Permite carregar outro kernel via kexec syscall
--enable CONFIG_KEXEC_FILE \               # Variante mais segura do kexec (usada no Android moderno)
--enable CONFIG_KEXEC_CORE \               # Infraestrutura base do kexec no kernel
--enable CONFIG_CRASH_DUMP \               # Necessário para memória de dump e alguns fluxos kexec
--enable CONFIG_RELOCATABLE \              # Permite kernel rodar em endereços diferentes (essencial pro kexec)
--enable CONFIG_RANDOMIZE_BASE \           # KASLR — geralmente exigido junto com relocatable
--enable CONFIG_ARM64_CRASH_DUMP \         # Suporte crash dump específico ARM64
--enable CONFIG_PHYS_ADDR_T_64BIT \        # Endereçamento físico 64-bit (Android ARM64 moderno)
--enable CONFIG_ATAGS \                    # Compatibilidade legacy boot params (às vezes ajuda em kexec Android)
--disable CONFIG_SECURITY \                # Remove LSM framework (SELinux/AppArmor etc.)
--disable CONFIG_SECURITY_SELINUX \        # Desativa SELinux completamente
--disable CONFIG_SECURITY_SELINUX_BOOTPARAM \ # Remove override SELinux via boot param
--disable CONFIG_DM_VERITY \               # Remove dm-verity (integridade forçada de system/vendor)
--disable CONFIG_DM_ANDROID_VERITY \       # Versão Android do verity
--disable CONFIG_FS_VERITY \               # Verificação criptográfica por filesystem
--disable CONFIG_SECURITYFS \              # FS usado por módulos de segurança
--enable CONFIG_BLK_DEV_INITRD \           # Permite usar ramdisk/initrd (boot.img Android)
--enable CONFIG_RD_GZIP \                  # Suporte ramdisk gzip
--enable CONFIG_RD_LZ4 \                   # Suporte ramdisk LZ4 (muito comum Android)
--enable CONFIG_RD_XZ \                    # Suporte ramdisk XZ
--enable CONFIG_DEVTMPFS \                 # Criação automática de /dev pelo kernel
--enable CONFIG_DEVTMPFS_MOUNT \           # Monta /dev automaticamente no boot
--enable CONFIG_EXT4_FS \                  # Filesystem EXT4 (system/vendor Android)
--enable CONFIG_F2FS_FS \                  # Filesystem flash-friendly comum em Android
--enable CONFIG_VFAT_FS \                  # FAT/VFAT (partições misc/boot/media)
--enable CONFIG_TMPFS \                    # FS temporário RAM (usado pelo Android init)
--enable CONFIG_OVERLAY_FS \               # OverlayFS (muito usado em mods/root/ROMs)
--enable CONFIG_PRINTK \                   # Logs kernel (dmesg)
--enable CONFIG_EARLY_PRINTK \             # Logs early boot — crucial debug kexec
--enable CONFIG_DEBUG_KERNEL \             # Ativa opções gerais debug kernel
--enable CONFIG_DEBUG_INFO \               # Inclui símbolos debug
--enable CONFIG_MAGIC_SYSRQ \              # SysRq debug (reinício forçado etc.)
--enable CONFIG_SYSVIPC \                  # IPC System V (Android userspace usa)
--enable CONFIG_POSIX_MQUEUE \             # Filas POSIX — libs Android dependem
--enable CONFIG_CGROUPS \                  # Control groups (Android init usa pesado)
--enable CONFIG_NAMESPACES \               # Namespaces estilo container
--enable CONFIG_MEMCG \                    # Memory control groups Android LMKD etc.
--enable CONFIG_BLK_DEV_LOOP \             # Loop devices (montar imagens ROM)
--enable CONFIG_DM_MAPPER \                # Device mapper (base crypt/fs virtual)
--enable CONFIG_DM_CRYPT \                 # Criptografia storage Android
--enable CONFIG_BINFMT_MISC \              # Executar binários não nativos (scripts/init mods)
--disable CONFIG_MTK_SEC_VIDEO_PATH_SUPPORT \ # Remove restrição TrustZone vídeo MTK (se existir)
--disable CONFIG_MTK_IN_HOUSE_TEE_SUPPORT     # Remove hooks TEE MediaTek (às vezes bloqueia mods)

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
