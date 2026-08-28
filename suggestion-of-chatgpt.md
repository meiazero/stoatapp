Sim. Se o objetivo é um fork do Stoat que torne o Nitro desnecessário **para quem migra do Discord para sua instância**, existe um caminho bastante claro.

Hoje o Nitro oferece principalmente uploads de 500 MB, streaming HD, emojis/stickers globais, avatar/banner animados, temas/perfis personalizados, perfis por servidor, soundboard, super reactions, 4.000 caracteres e até 200 servidores. ([Discord Support][1])

O Stoat já tem uma vantagem importante: vários limites são configuráveis no backend. Atualmente o default upstream está em 20 MB por attachment, mensagens de 2.000 caracteres, 100 servidores e vídeo 1280×720, então parte da "paridade Nitro" nem exige necessariamente novas abstrações. ([GitHub][2])

### O que eu implementaria

| Feature             | Nitro    | Fork Stoat                           | Prioridade |
| ------------------- | -------- | ------------------------------------ | ---------- |
| Upload 500 MB+      | ✅        | **1–5 GB configurável**              | P0         |
| Mensagem >4k chars  | ✅        | **configurável, ex. 16k**            | P0         |
| Avatar animado      | ✅        | GIF/WebP/AVIF animado                | P0         |
| Banner animado      | ✅        | GIF/WebP/AVIF animado                | P0         |
| Profile theme       | ✅        | gradiente + accent + background      | P0         |
| Perfis por servidor | ✅        | **identidade completa por servidor** | P0         |
| Emoji global        | ✅        | **biblioteca pessoal de emojis**     | P0         |
| Stickers globais    | ✅        | **packs pessoais**                   | P0         |
| 1080p60             | ✅        | 1080p60                              | P0         |
| 1440p60 / AV1       | limitado | **✅**                                | P1         |
| Soundboard          | ✅        | ✅                                    | P1         |
| Super reactions     | ✅        | **reaction effects**                 | P1         |
| Video backgrounds   | ✅        | ✅                                    | P1         |
| App themes          | ✅        | **theme engine completo**            | P1         |
| App icon            | ✅        | ✅                                    | P2         |
| Name styles         | ✅        | ✅                                    | P2         |

### Onde eu faria o fork superar o Nitro

**1. Personal Emoji Library**

Em vez de:

```text
emoji pertence ao servidor
```

usar:

```text
User
 └── Emoji Library
      ├── Personal
      ├── Packs
      ├── Imported
      └── Server emojis
```

O usuário poderia criar packs:

```text
Pokémon
Pepe
Memes
Work
Custom
```

e utilizá-los em qualquer servidor que permita `USE_EXTERNAL_EMOJI`.

Isso é superior ao modelo do Discord porque os assets realmente pertencem à biblioteca do usuário.

---

**2. Perfil realmente contextual**

O Nitro tem perfil por servidor.

Eu iria além:

```text
GlobalProfile

ServerProfile
 ├── avatar
 ├── banner
 ├── bio
 ├── displayName
 ├── pronouns
 ├── theme
 └── status

RoleProfile (optional)
ChannelProfile (optional)
```

Por exemplo:

```text
Servidor UFC
Emanuel Ávila
avatar profissional

Servidor amigos
Emanuel
avatar Pokémon

Servidor trabalho
Emanuel Pires
banner institucional
```

Um `ServerProfile` completo seria suficiente inicialmente.

---

**3. Theme Engine aberto**

Não faria "temas premium".

Criaria algo semelhante a:

```json
{
  "name": "Catppuccin Mocha",
  "colors": {},
  "typography": {},
  "radius": {},
  "spacing": {},
  "background": {},
  "effects": {}
}
```

Com:

* import/export JSON;
* URL de theme;
* CSS variables;
* custom CSS opcional;
* temas por servidor;
* sincronização entre dispositivos;
* accent automático baseado no avatar;
* light/dark variants.

Isso já seria muito superior ao Nitro.

---

**4. Uploads tratados como storage de verdade**

Em vez de simplesmente aumentar:

```toml
attachments = 500_000_000
```

eu implementaria:

```text
Client
   ↓
Upload Session
   ↓
multipart/chunk upload
   ↓
Object Storage
   ↓
Media Processor
   ├── thumbnail
   ├── preview
   ├── metadata
   └── transcoding
```

Com:

* multipart;
* resumable upload;
* upload progress;
* SHA-256 deduplication;
* S3/MinIO;
* previews;
* streaming HTTP Range;
* thumbnail generation;
* EXIF stripping opcional;
* AV1/WebM transcode;
* storage quota configurável.

Depois poderia colocar:

```text
Default: 500 MB
Power user: 2 GB
Instance max: 10 GB
```

O upstream já expõe os limites via configuração, então existe uma fundação adequada para isso. ([GitHub][2])

---

**5. Streaming melhor que Nitro**

Como o Stoat está trabalhando atualmente em voice/video/screenshare via LiveKit, essa área está evoluindo rapidamente; versões recentes adicionaram seleção de node por latência e UI de screen sharing. ([GitHub][3])

Eu colocaria:

```text
720p30
720p60
1080p30
1080p60
1440p60
Original
```

e codecs:

```text
H.264
VP9
AV1
```

Com detecção:

```text
AV1 hardware encoder available?
 ├── yes → AV1
 └── no
      ├── VP9 available → VP9
      └── H264
```

Mais:

* bitrate manual;
* FPS manual;
* screen audio;
* application audio;
* window capture;
* HDR quando suportado;
* ultrawide;
* per-viewer quality;
* simulcast;
* noise suppression RNNoise;
* echo cancellation configurável.

Isso seria uma das features com maior potencial de diferenciação.

---

**6. Media Vault**

Feature que Discord praticamente não trata como produto.

Dentro do perfil:

```text
Media
├── Images
├── Videos
├── Files
├── GIFs
└── Saved
```

Pesquisa:

```text
from:emanuel
type:image
server:foo
before:2026-08
```

E você ganha uma espécie de "Google Photos dos arquivos enviados no Stoat".

---

**7. Bookmarks**

Em vez do usuário mandar mensagem para si mesmo:

```text
Save Message
```

com:

```text
Collections
├── Read later
├── Projects
├── Memes
└── Important
```

Incluindo notas e tags.

É pequena e extremamente útil.

---

**8. Scheduled messages + reminders**

Nativamente:

```text
Send now
Schedule...
```

e:

```text
Remind me
 ├── 20 minutes
 ├── 1 hour
 ├── Tomorrow
 └── Custom
```

Não é uma "Nitro feature", mas é justamente o tipo de funcionalidade que faria alguém preferir o cliente.

---

**9. Presence API realmente aberta**

Algo como:

```text
Activity
├── type
├── application
├── title
├── subtitle
├── startedAt
├── image
├── buttons[]
└── metadata
```

Permitindo:

```text
VS Code
Steam
Spotify
Jellyfin
Plex
YouTube
GitHub
JetBrains
custom applications
```

Sem depender de uma whitelist centralizada.

---

**10. Plugin system**

Aqui estaria provavelmente o maior diferencial.

```text
Stoat Extension API

permissions:
  messages.read
  messages.write
  ui.sidebar
  ui.contextMenu
  commands.register
  notifications
  presence
```

Extensões isoladas:

```text
extension.json
index.js
```

com sandbox e capabilities explícitas.

Então coisas que normalmente exigiriam BetterDiscord/Vencord virariam parte suportada da plataforma.

Exemplo:

```json
{
  "name": "GitHub",
  "permissions": [
    "ui.sidebar",
    "commands.register"
  ]
}
```

Isso coloca o Stoat em uma categoria que o Discord dificilmente consegue alcançar oficialmente.

---

### Uma feature que eu considero particularmente forte: **Personal Assets**

Unificaria:

```text
Personal Assets

├── Emojis
├── Stickers
├── Sounds
├── Themes
├── Avatars
├── Banners
└── Backgrounds
```

Tudo sincronizado na conta.

Assim o conceito deixa de ser:

> servidor X possui emoji Y

e passa a ser:

> Emanuel possui um pack de emojis e pode usá-lo onde tiver permissão.

Isso encaixa muito melhor em uma plataforma user-first.

### Roadmap que eu seguiria

**Milestone 1 — "Nitro parity"**

```text
Animated avatars
Animated banners
Profile themes
Server profiles
Personal emojis
Personal stickers
500 MB uploads
4k+ messages
1080p60
Soundboard
Video backgrounds
```

**Milestone 2 — "Better than Nitro"**

```text
Theme engine
Emoji packs
Personal Assets
Bookmarks
Scheduled messages
Reminders
Media Vault
Advanced screen share
AV1
Rich Presence API
```

**Milestone 3 — "Stoat Power User"**

```text
Plugin API
Custom CSS
Multi-account
Advanced notification rules
Local message indexing
Client-side message archive
Config sync
Extension marketplace
```

Eu evitaria copiar coisas como **Boosts, badges, Nitro Rewards, Orbs e loja**. São mecanismos de monetização, não melhorias substanciais de comunicação.

Uma arquitetura que eu acharia particularmente limpa seria introduzir um conceito genérico de **user capabilities**, mesmo que no seu fork todo mundo possua todas:

```rust
pub struct UserCapabilities {
    pub animated_avatar: bool,
    pub animated_banner: bool,
    pub server_profiles: bool,
    pub external_emojis: bool,
    pub external_stickers: bool,

    pub max_upload_size: u64,
    pub max_message_length: usize,

    pub max_stream_width: u16,
    pub max_stream_height: u16,
    pub max_stream_fps: u8,
}
```

Isso evita espalhar checks e magic numbers pelo backend e posteriormente permite capabilities por instância, usuário ou role sem transformar o código em uma cópia do sistema Nitro.

E há um detalhe importante para sua distribuição: backend e `for-web` são AGPL-3.0. Se você disponibilizar sua versão modificada por rede, a AGPL exige disponibilizar o corresponding source dessa versão aos usuários. Isso é totalmente compatível com o modelo que você descreveu. ([GitHub][4])

Se fosse **meu backlog**, eu atacaria primeiro **Server Profiles + Personal Assets + Theme Engine + uploads robustos + 1080p60**. Esses cinco mudariam perceptivelmente o produto e cobririam a maior parte do valor prático pelo qual alguém paga Nitro.

[1]: https://support.discord.com/hc/pt-br/articles/115000435108-O-que-s%C3%A3o-Nitro-e-Nitro-Basic?utm_source=chatgpt.com "O que são Nitro e Nitro Basic? – Discord"
[2]: https://github.com/stoatchat/stoatchat/blob/main/crates/core/config/Revolt.toml?utm_source=chatgpt.com "stoatchat/crates/core/config/Revolt.toml at main · stoatchat/stoatchat · GitHub"
[3]: https://github.com/stoatchat/for-web/releases?utm_source=chatgpt.com "Releases · stoatchat/for-web · GitHub"
[4]: https://github.com/stoatchat/stoatchat/blob/main/LICENSE?utm_source=chatgpt.com "stoatchat/LICENSE at main · stoatchat/stoatchat · GitHub"

