# Root Motorola Edge 60 Fusion

Guia para instalar o **KernelSU v3.3.1 em modo LKM** no Motorola Edge 60 Fusion. O procedimento foi executado em um aparelho brasileiro com a configuração abaixo.

| Item | Configuração testada |
| --- | --- |
| Modelo | Motorola Edge 60 Fusion XT2503-1 |
| Codinome e canal | `scout` · `RETBR` |
| Sistema | Android 17 · `A171VCH.41-15` |
| Kernel | `6.1.157-android14-11-g14047afecd8b-ab16012876` |
| Família do módulo KernelSU | ARM64 · `android14-6.1` |
| Partição modificada | `init_boot` do slot ativo |

## 1. Prepare o computador e faça backup

- Faça backup de fotos, documentos, mensagens e credenciais de autenticação. **Desbloquear o bootloader restaura o telefone aos padrões de fábrica.**
- Baixe o [Android SDK Platform-Tools](https://developer.android.com/tools/releases/platform-tools). Abra o terminal na pasta das ferramentas ou adicione-a ao PATH. No PowerShell, use `./adb.exe` e `.astboot.exe` se as ferramentas estiverem na pasta atual.
- No telefone, toque sete vezes em **Configurações → Sobre o telefone → Número da versão**. Em **Opções do desenvolvedor**, ative **Desbloqueio de OEM** e **Depuração USB**.
- Conecte um cabo USB de dados e aceite na tela a autorização de depuração deste computador. Rode `adb devices` e confira se aparece `device`.

## 2. Confirme a variante e a compilação

```sh
adb shell getprop ro.boot.hardware.sku
adb shell getprop ro.carrier
adb shell getprop ro.build.display.id
adb shell uname -r
adb shell getprop ro.boot.flash.locked
```

No teste, os três primeiros comandos retornaram `XT2503-1`, `retbr` e `A171VCH.41-15`. O último retorna `1` com o bootloader bloqueado e `0` com ele desbloqueado. O Android em execução era 17, mas o kernel pertencia à família `android14-6.1`. Se a compilação for diferente, obtenha o firmware correspondente e verifique a compatibilidade do kernel antes de seguir.

## 3. Desbloqueie o bootloader

Se o aparelho já estiver desbloqueado, avance para o passo 4.

```sh
adb reboot bootloader
fastboot devices
fastboot oem get_unlock_data
```

1. Junte os trechos retornados pelo último comando em uma única linha. Remova os prefixos `(bootloader)` e os espaços, mantendo os caracteres `#`.
2. Entre no [portal de desbloqueio da Motorola](https://en-us.support.motorola.com/app/standalone/bootloader/unlock-your-device-a/action/auth), verifique a elegibilidade e solicite sua chave. Mantenha a chave e os dados de desbloqueio privados.
3. Com o backup concluído e o telefone à vista, substitua `SUA_CHAVE_MOTOROLA` pela chave recebida e rode o comando abaixo. Na tela do aparelho, selecione a opção de desbloqueio com os botões de volume e confirme com o botão liga/desliga.

```sh
fastboot oem unlock SUA_CHAVE_MOTOROLA
```

Aguarde o aparelho voltar ao Fastboot. Confirme o desbloqueio com `fastboot getvar securestate`. No aparelho testado, o resultado foi `flashing_unlocked`. Uma resposta `OKAY` após o comando de desbloqueio, por si só, não confirmou a mudança. Rode `fastboot reboot`, conclua a configuração inicial e reative a depuração USB.

## 4. Obtenha o init_boot original

Baixe o firmware para **a mesma variante, canal e compilação** instalada no telefone. Para o aparelho testado, use o pacote `XT2503-1 / RETBR / A171VCH.41-15`. A [página do pacote usado no teste](https://www.stockrom.net/2026/11/xt2503-1-retbr-os17-a171vch-41-15.html) é de um espelho de terceiros. Confira a versão em `flashfile.xml` e compare o checksum de `init_boot.img` com o listado no manifesto.

Extraia `init_boot.img` e guarde uma cópia intocada no computador. No pacote testado, a imagem tinha 8.388.608 bytes e SHA-256 `aeca398f301dd66467bb025424eee1a1c258500f6d67f13c2d602d7d860fe635`. Esse hash identifica a imagem usada no teste. Para outro firmware, use os dados do respectivo pacote.

## 5. Prepare a imagem com KernelSU

1. Baixe apenas o APK oficial da [versão v3.3.1 do KernelSU](https://github.com/tiann/KernelSU/releases/tag/v3.3.1) e instale-o no Android. Também é possível usar `adb install KernelSU_v3.3.1_32680-release.apk`.
2. Abra o gerenciador do KernelSU e confira a compatibilidade. Se indicar que o kernel não é suportado, pare antes de gravar qualquer imagem.
3. Copie a imagem de fábrica para o telefone com `adb push init_boot.img /sdcard/Download/`.
4. No gerenciador, use **Instalar → Selecionar um arquivo** e escolha o `init_boot.img`. Esse método aplica o módulo LKM ao ramdisk da compilação instalada. No teste, o patch foi feito no próprio aparelho com o comando `boot-patch` do utilitário `ksud` da mesma versão.
5. Copie a imagem gerada de volta ao computador e salve-a como `kernelsu-init-boot.img`. Guarde-a separada da imagem original. [Guia oficial de instalação](https://kernelsu.org/guide/installation.html)

## 6. Grave apenas o init_boot do slot ativo

```sh
adb reboot bootloader
fastboot devices
fastboot getvar product
fastboot getvar securestate
fastboot getvar current-slot
```

Confirme `scout`, `flashing_unlocked` e anote o slot ativo. Rode **somente** o comando correspondente ao slot exibido.

```sh
# Se o resultado foi a
fastboot flash init_boot_a kernelsu-init-boot.img

# Se o resultado foi b
fastboot flash init_boot_b kernelsu-init-boot.img
```

No teste, o slot ativo era `b`. A versão mais recente do Fastboot usada no teste ficou parada em `Fetching super` antes da gravação. O Fastboot **35.0.2** do Google concluiu o mesmo comando. Se ocorrer esse travamento, verifique se o aparelho continua em Fastboot, reconfirme o slot e tente com a [versão 35.0.2 para Linux](https://dl.google.com/android/repository/platform-tools_r35.0.2-linux.zip).

Após a gravação retornar `OKAY` nas etapas de envio e escrita, rode `fastboot reboot`.

## 7. Verifique o root

Depois da inicialização, abra o gerenciador do KernelSU e confirme que ele indica que o KernelSU está funcionando. No teste, o Android iniciou no mesmo slot e o comando de diagnóstico do KernelSU retornou `version: 32680` e `runtime_mode: lkm`. Conceda acesso root pelo gerenciador somente ao app que precisa dele. O acesso root pelo shell ADB depende de autorização específica.

## Como desfazer

Se o telefone não iniciar após a gravação, volte ao Fastboot. Com o aparelho desligado, mantenha **Volume para baixo + Liga/Desliga** até entrar no menu. Confira o slot e grave o `init_boot.img` original **na mesma partição que você modificou**. Por exemplo, se gravou no slot `b`:

```sh
fastboot flash init_boot_b init_boot.img
fastboot reboot
```

Após uma atualização do sistema, obtenha a imagem original da nova compilação antes de aplicar outro patch. Uma imagem criada para a compilação anterior pode impedir a inicialização.
