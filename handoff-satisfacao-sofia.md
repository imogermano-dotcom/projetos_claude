# Handoff — A5 "Sofia" (satisfação pós-venda) — 21/10/2026

## O que é
Inquérito de satisfação no WhatsApp, feito por uma 5.ª assistente do motor do agentOS (`a5_satisfacao`, "Sofia"):
- **Escritura** (comprador e vendedor), 7 dias depois, 10h de Lisboa: nota 0–10 → porquê → acompanhamento do consultor (1–5) → o que melhorar.
- **Rescisão do CMI** (vendedor), 2 dias depois de detectada: o que pesou → o que podíamos ter feito → voltaria a falar connosco.
- **Quem decide o passo seguinte é o servidor** (`fn_satisf_registar`, no Supabase), não o modelo:
  nota ≤ 6 e rescisões → tarefa `escalar` + email (uma vez); nota ≥ 7 → link do Google; 9–10 → também o testemunho (vídeo por link ou texto na conversa).
- O vendedor recebe a página `/testemunho?t=` com "A sua venda em números" (CMI → proposta aceite → escritura, preço inicial → valor de venda, estudo de mercado).

## Estado
| Parte | Onde | Estado |
|---|---|---|
| BD: `satisfacao_inqueritos`, `satisfacao_config`, `satisfacao_rescisoes_vistas`, RPCs `fn_satisf_*`/`rpc_satisf_*`, trigger `tgr_satisf_completar`, pg_cron `satisfacao-gerar` (06:20 UTC), balde privado `testemunhos`, linha `agente_config` `a5_satisfacao` | Supabase `zphasvfopnbzwnaidsnw` (migrações em `figueira-home-portal/supabase/migrations/20261020_10_*`, `20261021_03_*`, `20261021_04_*`) | ✅ aplicado |
| Edge function `satisfacao-video` (verify_jwt=false; upload por URL assinado + ver só admin) | Supabase | ✅ publicada e testada |
| Portal: aba ⭐ Satisfação, `/testemunho`, `/avaliar` | `figueira-home-portal` `main` (b3e6b71) | ✅ no ar |
| **agentOS: assistente, tools, encaminhamento, envio, cron** | `figueirahome_agentOS`, branch **`claude/recruitment-leads-portal-tab-e6s4na`** (34a09b3, rebased sobre `master` 7b71545) | ⏳ **por fazer merge e deploy** |
| Envio | `satisfacao_config.ligado = false` | 🔒 desligado — nada sai |

Há 5 inquéritos já na fila (escrituras de 01 e 03/10). Com o envio ligado saem na volta seguinte — rever/cancelar na aba antes.

## O que o branch do agentOS muda (plano: `docs/fases/satisfacao-pos-venda-plano.md`)
- `assistants.py`: `A5`, `NOME_A5="Sofia"`, `_PROMPT_A5`, tools `registar_satisfacao` e `registar_testemunho` (**sem** `escalar_para_humano`, que cria cliente/lead).
- `tools.py`: definições + `_registar_satisfacao` (grava por `fn_satisf_registar`, escala com tarefa+`notificar` e `fn_satisf_marcar_escalado`, devolve ao modelo os links que o servidor deu) e `_registar_testemunho`.
- `guards.py`: `satisfacao_aberta()` (`fn_satisf_aberto`); `agente_de_lead` devolve A5 **antes** de qualquer lead (inquérito enviado ≤ 14 dias, não concluído há > 24 h).
- `engine.py`: `_contexto_satisfacao` (tipo, nome, consultor, respostas já dadas; injecta o template no 1.º turno de thread nova; `fn_satisf_respondeu`); A5 fica fora de `promover_se_qualificada`.
- `channels/whatsapp/meta_api.py`: `send_template(to, nome, lingua, parametros)`.
- `agents/broker/satisfacao.py` + `api/satisfacao.py`: `POST /api/satisfacao/enviar` (`X-Automacao-Secret`) — lê `fn_satisf_para_enviar` (vazia com o interruptor desligado ou fora das 9h–20h de Lisboa), só telemóveis PT (9…), template com `{{1}}` = primeiro nome, semeia `agente_conversas` (agente `a5_satisfacao`), marca `fn_satisf_marcar_envio`.
- `crons/schedules.json`: `satisfacao-enviar` (`7 * * * *`).
- `tests/test_satisfacao.py`: 19 testes. Suite: 550 passam; as 6 que falham (`test_dashboard.py` ×5, `test_mapping_tarefa_data.py` ×1) já falhavam no `master` sem estas alterações (precisam da BD real / scraper).
- `CLAUDE.md`: entrada no "Estado actual".

## Para pôr a funcionar (por esta ordem)
1. **Meta / WhatsApp Manager**: criar e aprovar 3 templates, língua `pt_PT`, **sem botões** (o webhook ignora mensagens que não são texto), com o texto **exacto** que está no portal (aba ⭐ Satisfação → Configuração → "Textos dos templates"), `{{1}}` = primeiro nome:
   `satisfacao_comprador`, `satisfacao_vendedor`, `satisfacao_rescisao`.
   Se a Meta obrigar a mudar o texto, actualizar também `satisfacao_config.templates` (o texto semeado tem de ser igual ao enviado).
2. **agentOS**: rever e fazer merge do branch em `master`; `pytest backend/tests/`; `flyctl deploy --app figueirahome-agentos` (de `backend/`). ⚠️ O deploy envia a árvore de trabalho, não o HEAD.
3. **Crons**: registar `satisfacao-enviar` no `figueirahome-crons` (de `crons/`, como os outros de `schedules.json`).
4. **Teste em produção** antes de ligar a todos: na aba, deixar só um inquérito de teste (com o teu telemóvel via "Mudar telefone"), cancelar os outros, ligar, "Enviar já", aguardar o cron (:07) ou chamar `POST /api/satisfacao/enviar` com o `X-Automacao-Secret`, e responder à Sofia com 4 (deve escalar e não mandar links) e noutro com 10 (deve mandar Google + testemunho + resumo).
5. **Portal**: colar o link de avaliação do Google (`https://g.page/r/…/review`) e ligar "Envio automático".

## Armadilhas
- O conector MCP do Supabase pede confirmação a qualquer SQL com `DROP` e pendura até expirar — usar `create or replace` (incl. `create or replace trigger`).
- `fn_tel9` guarda 9 dígitos: números estrangeiros ficam em `erro` ("não é telemóvel português") — contactar à mão.
- Política do Google: o pedido de avaliação vai a todos os de escritura com nota ≥ 7 (não só 9–10), para não ser *review gating*.
- Página do cliente esconde a variação face ao preço inicial quando é negativa; o portal mostra sempre.
- `imovel_precos` só existe desde 03/10/2026 — preço inicial vem de `Angariação.imovel_venda`. Estudos de mercado: ainda nenhum carregado.
- `index.html` do portal está agora em **CRLF** no `main`.
