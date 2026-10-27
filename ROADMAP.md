# ROADMAP — fork pedrofmrj do Monitor OLX

Plano de desenvolvimento deste fork. O que já está feito, o que vem a
próximo e como contribuir de volta ao projeto original
([gusTavPeres/scrapper_bot](https://github.com/gusTavPeres/scrapper_bot)).

## Status

- ☑ Adotar o projeto (fork + `setup.sh` + 68 testes offline passando)
- ☑ Documentação: `docs/USANDO.md`, `docs/OPERACAO.md`, `docs/ARQUITETURA.md`
- ☐ Teste de uso completo em máquina local (Telegram + Docker)
- ☐ Validação do canário ao vivo (`test_live_olx.py`) em IP residencial

## Melhorias previstas

Prioridade alta (valide antes de codar — abrir issue upstream primeiro):

- [ ] **Baseline silencioso no 1º scan** — opção para marcar os anúncios já
      existentes como vistos e só notificar os novos a partir do 2º scan
      (hoje o 1º scan pode enviar até 5 alertas dos melhores).
- [ ] **Modo "avisar todo anúncio novo"** — variante para quem não quer a
      curadoria do avaliador (alerta cru de anúncio novo que passa nos filtros).

Prioridade média (conveniência):

- [ ] **Lançador de clique duplo** — `iniciar.bat` (Windows) / `iniciar.sh`
      para quem não quer decorar comandos.
- [ ] Verificar divergência de ordenação: `sf=1` (usado aqui, verificado no
      JSON da OLX) vs. relatos de `sf=1` zerando resultados em buscas regionais.

Baixa:

- [ ] Exemplos prontos de buscas no `docs/USANDO.md` (eletrônicos, imóveis,
      veículos) baseados em uso real.

## Manutenção

- Puxar correções do autor: `gh repo sync pedrofmrj/scrapper_bot --source gusTavPeres/scrapper_bot`
- Antes de cada commit: `./venv/bin/python test_offline.py` (tem que passar)
- O projeto não tem CI — os testes são nossa única rede de segurança

## Contribuindo ao projeto original

Fluxo recomendado (canal: issues — o repo não tem CONTRIBUTING):

1. **Issue prévia** descrevendo o que se quer contribuir e perguntando se é
   bem-vinda (evita trabalho desperdiçado — autor é único e o repo é novo).
2. **Branch pequena e focada** criada a partir do `upstream/master` (não do
   master deste fork, para o diff conter só a contribuição).
3. Rodar `test_offline.py` e seguir o estilo do repo: docstrings que explicam
   o porquê, comentários com o fato verificado, nomes em português.
4. Após o merge: `gh repo sync` e remover o patch local.

Boas candidatas a PR upstream: os itens de prioridade alta do roadmap acima e
qualquer melhoria de documentação genérica (não específica deste fork).

## Convenções deste fork

- Branches: `docs/...`, `feat/...`, `fix/...` — uma ideia por branch
- `PLAN.md`/notas de uso pessoal ficam **fora** do repositório (são internas)
- Segredos (`TELEGRAM_TOKEN`, `BOT_PASS`) só no `.env` (gitignored)
