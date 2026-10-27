# Arquitetura — mapa dos módulos

Para quem vai contribuir ou entender o fluxo. Visão de usuário:
[USANDO.md](USANDO.md). Operação: [OPERACAO.md](OPERACAO.md).

## Visão geral

```
┌──────────────┐   ┌──────────────┐
│ telegram_bot │   │    main.py   │  FastAPI: painel + API REST + /ws
│  (comandos)  │   │  (interface) │
└──────┬───────┘   └──────┬───────┘
       │                  │
       └────────┬─────────┘
                ▼
           models.py            schemas Pydantic (fonte única: API E bot)
                ▼
           database.py          SQLite (WAL): buscas, anuncios, analises,
                │                historico_precos, fila, feedback
                ▼
          scheduler.py          loop → SCAN/MONITOR → curl_cffi → flight data
                ▼
          avaliador.py          identidade + regras + preço + classificador
                ▼
           notifier.py          Telegram (HTML escapado, fotos, botões, debounce)
```

## Módulo a módulo

| Arquivo | Papel | Detalhes que importam |
|---|---|---|
| `main.py` | App FastAPI: painel estático, API REST (`/api/buscas`), WebSocket (`/ws`), auth HTTP Basic, watchdog do scheduler | Sem `BOT_PASS` não sobe; `HOST`/`PORT` do `.env` |
| `telegram_bot.py` | Parse dos comandos (`/add`, `/buscas`, ...) em `BuscaCreate`/`BuscaUpdate` | Ajuda embutida (`/help`); semântica `obs=` (exige/exclui/pontua) |
| `models.py` | Schemas Pydantic + validadores (UF, termo genérico, faixa de preço, filtros de carro) | Mensagens de erro traduzidas para PT-BR |
| `database.py` | Camada SQLite: conexão com context manager (WAL, FK), migração por `user_version`, CRUD, fila, limpeza diária | **Todo timestamp comparado em SQL é gerado em SQL** (`datetime('now')`) — regra documentada no topo do arquivo |
| `scheduler.py` | Orquestração: monta URL da OLX, baixa RSC/HTML, extrai anúncios, pagina, aplica filtros determinísticos, baixa descrições | `curl_cffi` com `impersonate="chrome"`; `403` → `OlxIndisponivel` (falha explícita, nunca raspa às cegas) |
| `avaliador.py` | O "cérebro": triagem de identidade (~97% do ruído morre aqui), regras do `obs=`, problemas por gravidade, golpe, preço vs. teto/mediana, classificador de 👍/👎, processamento da fila e decisão de alerta | ~1 ms por anúncio; `MAX_ALERTAS = 5` por busca/rodada |
| `notifier.py` | Entrega no Telegram: escape HTML (título com `<` quebrava msg), fotos, teclado 👍/👎, resumo por busca, `alertar()` debounced p/ erros | Token nunca vaza em log (`sem_token()`) |
| `frontend/` | Painel vanilla JS (sem build): formulário de busca, lista, detalhe com histórico de preços, atualização via WebSocket | Espelha 1:1 a API REST |
| `test_offline.py` | 68 testes P1–P6: extração flight, migração, corrida de fila, fuzz, escape/segurança, API | Rodar **antes** de qualquer mudança |
| `test_live_olx.py` | Canário contra a OLX real: layout, payload RSC, ordem por data | Só funciona em IP residencial |

## Ciclo de vida de uma busca

```
criada (status=ativa, fase=scan)
   │
   ▼
SCAN (1º scan e re-scan diário): até 25 páginas, TUDO que a OLX devolve
   │  grava anúncios + entra na fila (prioridade 1)
   ▼
MONITOR (a cada intervalo_min): páginas mais novas (sf=1 = ordem por data),
   │  para na primeira página sem anúncio novo já conhecido (até 10 págs)
   ▼
avaliador.processar_fila():
   triagem de identidade → avaliar → salvar análise →
   se alerta (!= "") e descrição baixada → notifier.enviar_resultado()
```

- `status` (`ativa`/`pausada`/`erro`) é **independente** de `fase`
  (`scan`/`monitor`) — interrupção no meio nunca mata a busca (bug corrigido,
  documentado em `database.py`).
- Reavaliações silenciosas usam `prioridade = -1` (salva sem notificar).
- Queda de preço: compara com o valor anterior no banco; notifica se
  ≥3% e ≥R$50 (em anúncio já aprovado) ou se cruzou o teto do usuário.

## Por que estas escolhas técnicas

| Decisão | Motivo |
|---|---|
| `curl_cffi` (`impersonate="chrome"`) | Cloudflare da OLX marca `requests`/curl puros pelo fingerprint TLS, mesmo de IP residencial |
| Payload **RSC** (`RSC: 1`) com fallback no HTML flight | ~3,4× menor que o HTML; mesmos dados. Sem flight data = layout mudou → falha explícita (fallback por CSS raspa errado — lição documentada no `scheduler.py`) |
| Descrição via **JSON-LD** da página do anúncio | No RSC da página de detalhe o texto não vem; JSON-LD (schema.org) vem e é estável |
| **SQLite** puro, sem ORM | 1 usuário, escala de dezenas de milhares de anúncios; WAL + `user_version` dão conta |
| Filtros determinísticos **antes** do avaliador | Preço/envio/vendedor são baratos em SQL/Python; o avaliador (que visita descrição) é caro — só vê o que passou |
| `sf=1` (ordem por data) | O monitor só faz sentido lendo as novidades primeiro; verificado no campo `sort` do JSON da OLX (2026-10) |
| Máx. 5 alertas/rodada + `/top` | 1º scan de busca ampla não vira enxurrada |
| Falha explícita (`OlxIndisponivel`) | Raspar "meia verdade" por meses é pior que parar e avisar |

## Convenções de código

- **Docstring no topo de cada arquivo** explicando o porquê e bugs históricos —
  mantenha ao editar.
- Comentários de fato verificado datados ("verificado 2026-10-22") — se
  você verificar algo novo, comente com a data.
- Defensivo em todo payload externo: checar tipo antes de usar; 1 anúncio
  fora do formato não derruba a página.
- Texto vindo de anúncio passa por `esc()` antes de qualquer HTML do Telegram.
- Nomes e mensagens em português.
- Timestamps em SQL (`datetime('now')`), nunca hora local do Python.

## Banco de dados

| Tabela | Conteúdo |
|---|---|
| `buscas` | configuração de cada busca (termo, filtros, `obs`, intervalo, status, fase) |
| `anuncios` | um por busca+hash; título, preço, descrição, categoria, `notificado`, `primeira_vez`/`ultima_vez` |
| `analises` | veredito do avaliador (score, recomendação, `alerta`), 1 por anúncio |
| `historico_precos` | série de preços (detecção de queda) |
| `fila` | o que precisa ser avaliado; `prioridade` (-1 = silenciosa) |
| `feedback` | 👍/👎 com **cópia** do texto do anúncio (aprende mesmo se o anúncio sumir) |

## Como contribuir

1. Fork + branch pequena a partir do `upstream/master`.
2. `./venv/bin/python test_offline.py` verde antes do commit.
3. Siga as convenções acima; issue prévia para mudanças de comportamento.
4. Roadmap e itens abertos: [ROADMAP.md](../ROADMAP.md).
