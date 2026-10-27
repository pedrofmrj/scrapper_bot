# Operação — rodar, manter e diagnosticar

Guia de quem administra o Monitor OLX no dia a dia. Para instalar e usar,
veja o [README](../README.md) e o [USANDO.md](USANDO.md).

## Modos de rodar

### Docker (recomendado)

```bash
cp .env.example .env      # 1ª vez: defina TELEGRAM_TOKEN e BOT_PASS
docker compose up -d      # build + start em background
docker compose logs -f    # acompanhar
docker compose down       # parar (o volume "dados" preserva o banco)
```

- Banco em volume nomeado `dados` (`/app/data`) — sobrevive a rebuild/update
- Porta: `${PORT:-8080}` (mude no `.env` se 8080 estiver ocupada)
- `restart: unless-stopped` — volta sozinho após reboot (com Docker ativo)
- Atualizar: `git pull && docker compose up -d --build`

### venv (Linux/macOS)

```bash
./setup.sh              # venv + dependências + .env + testes offline
./run.sh                # foreground (Ctrl+C para parar)
./setup.sh --servico    # instala serviço systemd (bot_web@$USER) + logrotate
```

Serviço instalado:

```bash
systemctl --user status bot_web@$USER    # estado (com --servico)
journalctl --user -u bot_web@$USER -f    # logs
systemctl --user restart bot_web@$USER   # reiniciar após update
```

### Windows sem Docker

```bat
py -m venv venv
venv\Scripts\pip install -r requirements.txt
copy .env.example .env
REM edite .env: BOT_PASS (obrigatória) e TELEGRAM_TOKEN
venv\Scripts\python main.py
```

Para rodar junto com o Windows: agende uma tarefa ao fazer logon apontando
para `venv\Scripts\pythonw.exe main.py` (ou use WSL + o `setup.sh --servico`).

## Arquivos e caminhos

| Caminho | Conteúdo |
|---|---|
| `.env` | segredos e portas (gitignored — nunca versione) |
| `data/bot.db` | banco SQLite (buscas, anúncios, avaliações, preços) |
| `data/backup*` | backup diário automático do banco |
| `logs/` | log do scheduler (fora de Docker; no Docker use `docker logs`) |

## Backup

O banco inteiro é um arquivo: `data/bot.db`. Em execução, o SQLite usa WAL —
para copiar com segurança:

```bash
# parado
cp data/bot.db backup-$(date +%F).db
# ou rodando (modo seguro)
sqlite3 data/bot.db ".backup 'backup-$(date +%F).db'"
```

No Docker, o banco vive no volume `dados`; para exportar:
`docker run --rm -v scrapper_bot_dados:/d -v $PWD:/b alpine cp /d/bot.db /b/`
(ajuste o nome do volume com `docker volume ls`).

## Atualizar

```bash
git pull
# Docker
docker compose up -d --build
# venv
./venv/bin/pip install -r requirements.txt
systemctl --user restart bot_web@$USER   # ou reinicie o ./run.sh
```

Migrações de banco rodam sozinhas no início (`PRAGMA user_version`).

## Monitoramento

- **Alerta automático no Telegram**: se a OLX bloquear ou o layout mudar
  (`OlxIndisponivel`), chega `⚠️ OLX indisponível` — as buscas param até
  resolver; não é spam, é debounced (1x a cada 12h por motivo).
- Falha repetida de uma busca por >6h também gera alerta.
- `/status` no Telegram mostra fila e estado das buscas.
- Log: procure `alerta enviado` (sucesso) e `Erro no scraping` (falha).

## Troubleshooting

### ⚠️ 403 / "OLX indisponível" no Telegram

O Cloudflare bloqueia em duas camadas: **IP de datacenter** (AWS, GCP,
Azure, provedores como OVH) e **fingerprint TLS**. O `curl_cffi` resolve a
segunda; a primeira não tem jeito de software: **o robô precisa de IP
residencial** (casa, Raspberry Pi, VPS residencial). VPNs comerciais
costumam também serem bloqueadas — IP residencial de verdade é o que
funciona. Confirme com:

```bash
./venv/bin/python test_live_olx.py    # 403 = bloqueio de IP/fingerprint
```

### Layout mudou ("página sem flight data")

A OLX mudou o HTML/payload. O projeto falha em vez de raspar às cegas
(comportamento intencional). Verifique se há atualização (`git pull`); se
não, é issue upstream — anote a URL e o trecho da resposta.

### Telegram não entrega (não chega nem `/help`)

- `TELEGRAM_TOKEN` errado → teste: `curl https://api.telegram.org/bot<TOKEN>/getMe`
- `TELEGRAM_CHAT_ID` vazio → mande uma mensagem ao bot e reinicie (ele
  responde com o id); no Docker, reinicie o container após editar o `.env`
- Sem `BOT_PASS` o programa **não sobe** (falha de segurança intencional)

### Painel não abre

- Verifique se o processo/container está de pé e a porta (`PORT` no `.env`)
- HTTP 401 pedindo usuário/senha é **normal** (HTTP Basic: `admin` + `BOT_PASS`)
- Sem senha configurada o programa não inicia

### Busca retorna 0 resultados

- `/go?q=...` não funciona: buscas por estado usam `/estado-XX`; o bot já
  monta a URL certa a partir de `estado=`
- Termo genérico é rejeitado pelo validador (precisa de palavra com 3+ letras
  ou modelo com número)
- Termos amplos podem misturar categorias: use `/categorias <id>` para filtrar

### Banco pesado / limpeza

A faxina diária (`manutencao_diaria`) remove o que expirou. Se quiser zerar:
pare o serviço, apague `data/bot.db*`, reinicie (as buscas recriam no 1º scan).

## Testes

```bash
./venv/bin/python test_offline.py     # 68 regressões (sem rede)
./venv/bin/python test_live_olx.py    # canário: layout, RSC e ordem na OLX real
```

Rode o offline antes de qualquer mudança; o live só em IP residencial.
