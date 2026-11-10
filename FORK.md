# Evolution API · fork com `sendAlbum`

Este fork existe por **uma** razão: a rota `sendAlbum`.

## O problema

O WhatsApp agrupa várias fotos num bloco deslizável, com UMA legenda
embaixo. A Evolution não expõe isso: `sendMedia` publica uma mídia por
chamada, então um produto com sete fotos vira sete mensagens soltas no
grupo.

A capacidade está na biblioteca. O Baileys ganhou envio de álbum na
`7.0.0-rc10`, e a 2.4.0 já usa a `rc13` — só não existe porta para ela.

## O que foi acrescentado

```
POST /message/sendAlbum/{instance}

{
  "number": "5511999999999",
  "caption": "legenda do álbum",
  "media": [
    { "type": "image", "media": "https://exemplo/foto-1.jpg" },
    { "type": "image", "media": "https://exemplo/foto-2.jpg" }
  ]
}
```

De 2 a 100 itens, imagem ou vídeo, por URL ou base64. Funciona para
contato e para grupo.

A legenda vai na PRIMEIRA mídia, e o id devolvido é o dela: quem cita um
álbum precisa citar a foto que carrega o texto, não a mensagem vazia que
abre o bloco.

Aceita `delay`, `quoted`, `mentionsEveryOne` e `mentioned`, como as
outras rotas de envio.

## Os arquivos tocados

| arquivo | o quê |
|---|---|
| `src/api/dto/sendMessage.dto.ts` | `SendAlbumDto` e `AlbumItem` |
| `src/validate/message.schema.ts` | `albumMessageSchema` |
| `src/api/controllers/sendMessage.controller.ts` | `sendAlbum()` |
| `src/api/routes/sendMessage.router.ts` | a rota |
| `src/api/integrations/channel/whatsapp/whatsapp.baileys.service.ts` | `albumMessage()` e um desvio em `sendMessage()` |

## O desvio, que é a única sutileza

Para mídia, a Evolution publica através de `forward`, remontando o
conteúdo. Esse caminho descarta `albumParentKey` — o único campo que faz
o WhatsApp agrupar as fotos. Sem ele, cada foto chega solta e o álbum
desaparece.

Por isso `sendMessage()` ganhou quatro linhas: conteúdo de álbum (o pai
ou um filho) vai direto para o Baileys, sem passar pelo `forward`.

Tudo o mais continua igual: cada foto é persistida, dispara webhook e
honra as opções, como qualquer outra mensagem. A mensagem-pai é a única
que NÃO é persistida, de propósito: ela não tem conteúdo, é um artefato
de protocolo, e guardá-la entregaria uma mensagem vazia para todo
consumidor de webhook interpretar.

## Por que isto dispensa um serviço à parte

Sem esta rota, a saída é publicar o álbum a partir de um segundo serviço
Baileys — o que significa uma SEGUNDA sessão de WhatsApp na mesma conta.
Duas sessões na mesma conta quebram a criptografia de quem recebe: as
mensagens chegam como "Aguardando mensagem" no aparelho do destinatário,
enquanto a API devolve sucesso.

Com a rota aqui, é uma conexão, um pareamento, um número.

## Manutenção

Partiu de `develop` (2.4.0-rc2, baileys rc13). A mudança vale como pull
request para o upstream: se entrar, este fork some.
