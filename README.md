# AGMM Launcher — Beta 0.1.0

Launcher Android ARM64 com interface AGMM e carregador BepInEx integrado. O mod mantém a versão pública **Canary 0.1**.

Esta Beta inclui o carregador e os arquivos de suporte, **sem AGMM.dll**. Em uma instalação nova, jogar com o mod depende de uma distribuição posterior que inclua essa DLL. DLLs do runtime e de interoperabilidade são necessárias e continuam incluídas.

## Instalação e atualização

Instale o APK e autorize a instalação desta origem quando o Android solicitar. O botão **Atualizar APK** consulta o canal configurado, baixa o APK completo dentro do aplicativo e abre o instalador do Android após verificar sua autenticidade. A instalação depende da confirmação do usuário.

A cópia temporária é removida no retorno do instalador ou após a substituição do aplicativo. Se necessário, a limpeza é repetida na próxima abertura. Uma atualização futura pode incluir launcher e DLL juntos.

Compatibilidade alvo: **Among Us 2026.9.29 (7492), Android ARM64**. O launcher exige Android 9 ou superior. O fluxo completo de atualização no aparelho ainda precisa ser validado com uma versão posterior publicada no canal.

Consulte [as informações desta distribuição](AGMM-INTEGRATION.md). O site de referência e o projeto original AGMM-source não foram modificados.

## Arquitetura

Baseado em uma adaptação comunitária não oficial do [BepInEx](https://github.com/BepInEx/BepInEx) para jogos Android IL2CPP ARM64. Utiliza o pacote oficial `Microsoft.NETCore.App.Runtime.android-arm64` (.NET 10 LTS) e os mecanismos de interceptação Dobby e Pine.

```text
Processo do jogo (:game)
  └── libmain.so — inicialização JNI
        └── libfusion.so — hospedagem CoreCLR e interceptação Dobby
              └── Runtime .NET e libcoreclr.so
                    └── BepInEx.Android.zip
                          ├── AndroidEntrypoint.Start()
                          ├── Carregador de plugins IL2CPP
                          └── Plugins do mod (.dll), quando incluídos
```

O launcher hospeda a atividade do jogo em um processo separado. `libfusion.so` inicia o runtime oficial .NET 10 para Android; esta biblioteca é própria desta base e não é o framework FusionCore citado nos créditos. `ExportThunkResolver` resolve as exportações antes da instalação das interceptações Dobby. O BepInEx foi adaptado para logcat, caminhos Android e interoperabilidade.

## Dependências fixadas

Os arquivos de `provenance/` registram `UPSTREAM_URL`, `UPSTREAM_REF` e `UPSTREAM_COMMIT`. Alterações nessas referências são manuais.

| Componente | Referência | Observação |
|---|---|---|
| BepInEx | `provenance/bepinex.env` | Adaptações em `patches/` |
| Runtime .NET | `provenance/dotnet-runtime.env` | Pacote Microsoft verificado por SHA-256 |
| MonoMod | `provenance/monomod.env` | Compilação para repositório NuGet local |
| Il2CppInterop | `provenance/il2cppinterop.env` | Compilação para repositório NuGet local |
| Dobby | `provenance/dobby.env` | Alinhamento de 16 KB |
| Pine | `provenance/pine.env` | Alinhamento de 16 KB; xz-embedded também fixado |

## Estrutura

| Diretório | Conteúdo |
|---|---|
| `launcher/` | Aplicativo Kotlin e bibliotecas C++/CMake |
| `launcher/app/src/main/cpp/` | Código de libmain.so e libfusion.so |
| `launcher/app/src/main/jniLibs/` | Bibliotecas nativas preparadas para compilação |
| `launcher/app/src/main/assets/` | Interface e pacotes de suporte |
| `patches/` | Adaptações aplicadas ao BepInEx de origem |
| `references/` | Adaptações das dependências nativas |
| `scripts/` | Compilação, empacotamento e verificação |
| `tools/` | Ferramentas locais, incluindo publicação de atualizações |
| `provenance/` | Referências, hashes e contratos |
| `docs/` | Documentação técnica da base |

A base original produz os binários por scripts e integração contínua. Esta cópia de trabalho também contém os arquivos preparados para compilar o launcher localmente. `provenance/sha256sums.txt` e os scripts de verificação registram e conferem os artefatos correspondentes.

## Compilação a partir das fontes

Para reconstruir toda a base, utilize Linux x86_64 ou ARM64, JDK 17, Android SDK API 35, NDK `30.0.16248370`, CMake 3.22.1 ou superior, .NET SDK `10.0.401` e as ferramentas git, zip, unzip e curl.

Execute na raiz:

```bash
# Bibliotecas nativas com alinhamento de 16 KB
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/30.0.16248370
./scripts/build_native_deps.sh

# Runtime oficial, com verificação SHA-256
./scripts/fetch_dotnet_runtime.sh /tmp/runtime

# Aplicação das adaptações e compilação do framework
./scripts/apply_patches.sh
./scripts/build_packages_parallel.sh
./scripts/restore_and_build.sh
./scripts/package_bepinex_android.sh
./scripts/stage_launcher_inputs.sh out/runtime out/BepInEx.Android.zip

# APK do launcher
cd launcher
echo "sdk.dir=$ANDROID_HOME" > local.properties
./gradlew assembleDebug
```

O processo de restauração da base usa Bash e `BASH_SOURCE`. Para executar comandos dotnet diretamente em outro shell, configure `BEPINEX_SRC` antes.

APK de depuração: `launcher/app/build/outputs/apk/debug/app-debug.apk`. Na cópia com os binários de suporte já preparados, o launcher pode ser compilado separadamente pelo Gradle. Não distribua chaves privadas nem arquivos locais de configuração.

## Diagnóstico e verificações

```bash
adb logcat -s fusion BepInEx Unity
adb logcat | grep -E '\[Error:Unity\]|\[Error:Il2CppInterop\]'
```

| Script | Verificação |
|---|---|
| `launcher/scripts/verify_prebuilts.sh` | Binários preparados, hashes e alinhamento de 16 KB |
| `launcher/scripts/verify_crypto_pal.sh` | Camada criptográfica Android sem dependência OpenSSL/BoringSSL |
| `launcher/scripts/verify_crypto_java.sh` | Assinaturas JNI e contrato Java da camada criptográfica |
| `scripts/verify_dotnet_pack.sh` | Quantidade de assemblies e estrutura de dotnet.zip |
| `scripts/verify_ci_artifacts.sh` | Concordância de hashes entre produtores e consumidores de artefatos |

## Documentação técnica da base

- [Guia de operação](docs/operations.md)
- [Integração e entrega contínuas](docs/ci-cd.md)
- [Planejamento](docs/todo.md)
- [Decisões de arquitetura](docs/progress.md)
- [Contratos para mods](docs/responsibility.md)
- [Estudo de caso Unity 6000](docs/thetower-6000.md)
- [Auditoria de armazenamento](docs/ci-storage-audit.md)

Os documentos vinculados mantêm seu idioma original.

## Créditos

- [BepInEx](https://github.com/BepInEx/BepInEx): framework original, acompanhado por uma sequência de adaptações.
- [FusionCore](https://github.com/All-Of-Us-Mods/FusionCore), equipe Starlight: referência para injeção, interceptações Pine e arquitetura do carregador nativo. Algumas classes são adaptações diretas, identificadas nos respectivos arquivos.
- [NextBep BepInEx.Android.Launcher](https://github.com/NextBep/BepInEx.Android.Launcher): referência de interface e recursos.
- [Pine](https://github.com/canyie/Pine) e [Dobby](https://github.com/canyie/dobby): mecanismos de interceptação.
- [MonoMod](https://github.com/MonoMod/MonoMod) e [Il2CppInterop](https://github.com/BepInEx/Il2CppInterop): redirecionamento de chamadas e geração de interoperabilidade. O suporte Android de MonoMod deriva do trabalho publicado em [XtraCube/MonoMod](https://github.com/XtraCube/MonoMod), reproduzido como adaptação; os mesmos arquivos também aparecem em [All-Of-Us-Mods/MonoMod](https://github.com/All-Of-Us-Mods/MonoMod). A adaptação ARM64 segue o projeto [All-Of-Us-Mods/Il2CppInterop](https://github.com/All-Of-Us-Mods/Il2CppInterop), referência starlight.
- [dotnet/runtime](https://github.com/dotnet/runtime): runtime distribuído no pacote oficial Microsoft Android ARM64.

## Licenças

- **Launcher e injeção nativa (libmain, libfusion):** [GPL-3.0](LICENSE).
- **Framework BepInEx:** [LGPL-2.1](https://github.com/BepInEx/BepInEx/blob/master/LICENSE).
- **Runtime .NET e bibliotecas base:** [MIT](https://github.com/dotnet/runtime/blob/main/LICENSE.TXT).
- **Dobby:** [Apache-2.0](https://github.com/canyie/dobby/blob/master/LICENSE).
- **Pine:** [Anti 996 v1.0](https://github.com/996icu/996.ICU/blob/master/LICENSE). Consulte a observação de compatibilidade em `launcher/THIRDPARTY_NATIVE.md`.
