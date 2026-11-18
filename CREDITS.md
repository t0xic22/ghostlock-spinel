# Créditos

Este fork do GhostLock foi adaptado para o **Redmi Note 15 4G (spinel)**, kernel
`6.12.69-android16-6-g014ca48a80ee-ab15889839-4k`. Todo o trabalho original
pertence aos autores abaixo — este repo contém apenas adaptações locais.

## Projeto base

- **[YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app)** — aplicativo
  GhostLock original: UI Compose,解析/merge de profiles HOCON, UserService via
  Shizuku, pipeline de ataque. Licença conforme o repo upstream.
- **[YuKongA/CVE-2026-98260](https://github.com/YuKongA/CVE-2026-98260)** — exploit
  timer LPE (POSIX CPU timers UAF) e perfil de kernel por `uname -r`. MIT.

## Exploit

- **[woshigenm/CVE-2026-98260](https://github.com/woshigenm/CVE-2026-98260)** —
  `src/uid0.c` e `src/poc.c` (MIT), base do `timer.c` conforme creditado no README
  original do YuKongA.
- Perfil do spinel (`kbuilds/6.12.69-...h`) derivado de análise do boot.img e do
  BTF extraído do device — ver `root-spinel-notes/fw/btf.bin` e `parse_btf.py`.

## Shizuku

- **[RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku)** — API e serviço
  que permitem executar o binário nativo como uid 2000 (shell) sem root persistent.
- **[thedjchi/Shizuku](https://github.com/thedjchi/Shizuku)** — fork v13.7.0 usado
  no device: corrige o NPE em `LoadedApk.makeApplicationInner` em ROMs MediaTek
  (issue [RikkaApps/Shizuku#1198](https://github.com/RikkaApps/Shizuku/issues/1198)).
- Forks `wess09/shizuku-m` e `hwangsihu/Shizuku` também foram avaliados durante
  a investigação do bug MediaTek.

## Outras referências

- **[Mods-Center](https://github.com/Mods-Center)** — módulos HyperOS avaliados
  para personalização (blur / recentes empilhados / patch de assinatura).
- Comunidade HyperOS (reddit r/HyperOS) — documentação dos comandos
  `service call miui.mqsas.IMQSNative 21` e `settings put task_stack_view_layout_style`.

## Adaptações locais

- Perfil HOCON `6.12.69-android16-6-g014ca48a80ee-ab15889839-4k.conf` e entrada
  no `index.conf` para satisfazer o gate do app.
- `isKernelSupported()` simplificada; `appComponentFactory` declarado no manifest.
- Task Gradle de build nativo/Rust neutralizada para usar o binário pré-compilado.

Consulte também [`CREDITS.md`](https://github.com/t0xic22/cve98260-spinel/blob/main/CREDITS.md)
e [`CREDITS.md`](https://github.com/t0xic22/root-spinel-notes/blob/main/CREDITS.md).
