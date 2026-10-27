# Guia de uso — do zero aos alertas

Este guia acompanha o [README](../README.md) (instalação) e explica a rotina
completa: configurar o Telegram, criar a primeira busca, entender o que chega
nos alertas e usar o painel. Referência de todos os comandos: mande `/help`
para o bot.

## 1. Configurar o Telegram (obrigatório para receber alertas)

1. No Telegram, abra o [@BotFather](https://t.me/BotFather) e envie `/newbot`.
2. Escolha um nome (ex.: `Meu Monitor OLX`). Ele devolve um **token** — algo
   como `123456789:AAE...`. **Não compartilhe** esse token: quem tem ele
   envia mensagens como seu bot.
3. Edite o `.env` na pasta do projeto:
   ```bash
   TELEGRAM_TOKEN=123456789:AAE...
   TELEGRAM_CHAT_ID=
   ```
4. Suba o programa (veja abaixo) e **mande qualquer mensagem** para o seu bot
   (ex.: `oi`). Ele responde com o seu **chat id**.
5. Cole o valor em `TELEGRAM_CHAT_ID` no `.env` e reinicie.

> Para alertar um grupo em vez de você: adicione o bot ao grupo, mande
> qualquer mensagem lá e use o chat id negativo que o bot responder
> (mesmo procedimento).

### Subindo o programa

```bash
# Docker (recomendado)
docker compose up -d

# ou venv (Linux/macOS)
./run.sh

# ou Windows sem Docker
venv\Scripts\python main.py
```

O painel web fica em `http://localhost:8080` (usuário `admin` + a `BOT_PASS`
do `.env` — no Docker, gere uma com `openssl rand -base64 18`).

## 2. Criar a primeira busca

No Telegram (ou no painel, que faz o mesmo):

```
/add rtx 5060 max=1900 estado=go vendedor=particular obs= nada de mineração
```

Opções comuns (todas no `/help`):

| Opção | Exemplo | O que faz |
|---|---|---|
| `min=` / `max=` | `min=500 max=2000` | faixa de preço |
| `estado=` | `estado=go` ou `brasil` | UF ou Brasil todo |
| `vendedor=` | `vendedor=particular` | particular ou loja |
| `envio=` | `envio=com_envio` | com envio / sem envio / frete grátis |
| `garantia=` | `garantia=olx` | só com Garantia da OLX |
| `condicao=` | `condicao=defeito` | procuro peça quebrada (defeito conta a favor) |
| `intervalo=` | `intervalo=30` | minutos entre scans (5–1440) |
| `obs=` | `obs= só com nota fiscal` | regras em texto livre (veja abaixo) |

Para carros existe `/addcarro` com filtros extra: `ano=2011-2016`,
`km=120000`, `motor=1.8`. Ex.: `/addcarro linea ano=2011-2016 estado=go`.

### Regras em `obs=` (o mais poderoso)

Tudo depois de `obs=` vira regra de avaliação:

- `sem X` / `nada de X` → **exclui** anúncios que mencionam X
- `só com X` → **exige** X
- `prefiro X` → **pontua** a favor (não bloqueia)

Ex.: `obs= só com nota fiscal, nada de mineração, prefiro caixa`

## 3. O que chega (e o que não chega)

O bot só avisa o que passou na triagem **e** na avaliação — não é um alerta
cru de "anúncio novo":

| Aviso | Significado |
|---|---|
| 🟢 **COMPRAR** | produto certo, sem problema grave, preço bom (ou justo p/ mediana) |
| 🟡 **VALE NEGOCIAR** | compensa, mas tem ressalva — ou o preço está um pouco acima do seu teto e o vendedor aceita proposta |
| 💸 **queda de preço** | baixou ≥3% e ≥R$50, ou era acima do seu teto e agora cabe |
| 📊 **resumo por busca** | no fim do scan: quantos avaliados, alertas que ficaram de fora, categorias encontradas |

Limites e particularidades:

- **No máximo 5 alertas por busca por rodada** — o resto fica no `/top`.
- Alertas só saem depois que a **descrição** do anúncio é baixada (o avaliador
  lê ela); anúncio sem descrição sai como `(sem descrição)`.
- O **1º scan** de uma busca nova avalia tudo de uma vez e pode mandar até 5
  alertas (os melhores). Se preferir silêncio total na estreia, veja o
  [ROADMAP](../ROADMAP.md) (baseline silencioso é uma melhoria prevista).
- Cada alerta tem botões **👍/👎**: usados pelo classificador que aprende com
  o seu gosto (a partir de 6 rótulos, com ≥2 por classe, o classificador
  entra em ação e muda a pontuação dos próximos).

## 4. Comandos do dia a dia

| Comando | Para que serve |
|---|---|
| `/buscas` | lista as buscas com **id** (use o id nos comandos abaixo) |
| `/top [id]` | melhores achados ativos que não viraram alerta |
| `/status` | tamanho da fila e estado das buscas |
| `/categorias <id>` | categorias da OLX encontradas (`/categorias 1 3` filtra pelas 1ª e 3ª) |
| `/scan <id>` | força um scan agora |
| `/pausar <id>` / `/retomar <id>` | pausa e retoma |
| `/del <id> sim` | apaga a busca (confirma com `sim`) |
| `/help` | referência completa de comandos e opções |

## 5. Painel web

`http://localhost:8080` — mesma coisa que o Telegram, com histórico:

- **Formulário de busca**: todos os campos do `/add` em vez de opções em linha
- **Lista de buscas**: editar, pausar, deletar, "scan agora"
- **Detalhe**: mostra a avaliação de cada anúncio (pros/contras, score,
  recomendação) e o histórico de preços
- **Tempo real**: atualiza sozinho via WebSocket enquanto o scan roda

> O painel usa HTTP Basic **sem HTTPS**: use só em casa/VPN, ou deixe
> `HOST=127.0.0.1` no `.env` para aceitar só conexões da própria máquina.

## 6. Rotina sugerida

1. Instala (README) + Telegram (seção 1)
2. Cria 1–2 buscas bem específicas (filtros reduzem ruído e carga na OLX)
3. Deixa rodar; lê os alertas; dá 👍/👎 nos primeiros
4. De vez em quando, olhe `/top` e o resumo do scan — anúncio bom demais para
   alertar pode estar lá
5. Se uma busca enche demais, estreite os filtros ou use `/categorias`

## Problemas comuns

Não recebe nada? Veja [OPERACAO.md — Troubleshooting](OPERACAO.md#troubleshooting).
