# Handoff — A5 "Sofia" (satisfação pós-venda) — actualizado 28/10/2026

## O que é
Inquérito de satisfação depois da **escritura** (comprador e vendedor) e da **rescisão do CMI** (vendedor).
- **Nada sai sem um clique**: o portal cria a fila sozinho (todos os dias às 06:20 UTC), mas cada inquérito só sai
  depois de alguém carregar em **«Enviar já»** na aba ⭐ Satisfação (coluna `aprovado_em`).
- **Canais, depois do clique** (portal, 21/10):
  1. **Email primeiro** (edge function `satisfacao-email`, pg_cron `satisfacao-email` a cada 10 min, 9h–20h, máx. 10/dia):
     o cliente escolhe a nota no email e responde na página `/inquerito?t=`.
  2. **WhatsApp 2 dias depois** se não respondeu ao email — é aqui que entra a **Sofia** (agentOS). Sem email válido → WhatsApp logo.
  3. Sem WhatsApp nem email → tarefa «LIGAR» e a resposta regista-se na aba («📝 Registar resposta»).
- **Perguntas** — escritura: nota 0–10 → porquê → acompanhamento do consultor (1–5) → o que melhorar.
  Rescisão: o que pesou → o que podíamos ter feito → voltaria a falar connosco.
- **Quem decide o passo seguinte é o servidor** (`fn_satisf_registar`): nota ≤ 6 e rescisões → tarefa `escalar` + email à direção;
  nota ≥ 7 → link do Google; 9–10 → também testemunho (vídeo por link ou texto).
- O vendedor recebe a página `/testemunho?t=`: "A sua venda em números" (CMI → proposta aceite → escritura, preço inicial → valor de venda,
  estudo de mercado), Google e testemunho.

## Estado (28/10)
| Parte | Onde | Estado |
|---|---|---|
| BD (tabelas `satisfacao_*`, RPCs, crons `satisfacao-gerar` e `satisfacao-email`, balde `testemunhos`), edge functions `satisfacao-video` e `satisfacao-email` | Supabase `zphasvfopnbzwnaidsnw` | ✅ no ar |
| Portal: aba ⭐ Satisfação (com «Enviar já», vistas Confirmados / Ligar / Sem email), `/testemunho`, `/avaliar`, `/inquerito` | `figueira-home-portal` `main` | ✅ no ar |
| Configuração | `satisfacao_config` | ✅ envio ligado, link do Google preenchido, escrituras desde 29/01/2024 |
| **Sofia no agentOS** (WhatsApp) | `figueirahome_agentOS`, branch **`claude/recruitment-leads-portal-tab-e6s4na`** (4ea0b21, sobre `master` 389818d) | ⏳ **por fazer merge e deploy** |
| Templates da Meta | WhatsApp Manager | ⏳ por criar/aprovar |

Fila: **140 inquéritos** (130 por confirmar, 8 sem telefone, 2 confirmados de teste a 21/10). Como a fila começa em
**escrituras de 2024**, decidir antes de confirmar em massa se se quer inquirir vendas com mais de um ano.
Até 28/10 saíram 0 emails e 0 WhatsApps.

**Sem a Sofia publicada**, o email já funciona, mas o WhatsApp de seguimento (e o WhatsApp de quem não tem email) **não sai** —
o cron `satisfacao-enviar` só existe no branch.

## O que o branch do agentOS muda (plano: `docs/fases/satisfacao-pos-venda-plano.md`)
- `assistants.py`: `A5 = "a5_satisfacao"`, `NOME_A5 = "Sofia"`, `_PROMPT_A5` (guião), tools `registar_satisfacao` e `registar_testemunho`
  (**sem** `escalar_para_humano`, que cria cliente/lead).
- `tools.py`: as duas tools; `_registar_satisfacao` grava por `fn_satisf_registar`, escala com tarefa + `notificar` + `fn_satisf_marcar_escalado`
  e devolve ao modelo os links que o servidor deu.
- `guards.py`: `satisfacao_aberta()` (`fn_satisf_aberto`); `agente_de_lead` devolve A5 **antes** de qualquer lead.
- `engine.py`: `_contexto_satisfacao` (tipo, nome, consultor, respostas já dadas; template no 1.º turno de thread nova; `fn_satisf_respondeu`);
  A5 fora de `promover_se_qualificada`.
- `channels/whatsapp/meta_api.py`: `send_template()`.
- `agents/broker/satisfacao.py` + `api/satisfacao.py`: `POST /api/satisfacao/enviar` (`X-Automacao-Secret`) — lê `fn_satisf_para_enviar`
  (só confirmados, 9h–20h, interruptor ligado; inclui os seguimentos de email), só telemóveis PT, template com `{{1}}` = primeiro nome,
  semeia `agente_conversas` (agente `a5_satisfacao`), marca `fn_satisf_marcar_envio`.
- `crons/schedules.json`: `satisfacao-enviar` (`7 * * * *`).
- Testes: `tests/test_satisfacao.py` (19). Com o branch: **775 passam**; as mesmas **6 falham também no `master`**
  (`test_dashboard.py` ×5 e `test_mapping_tarefa_data.py` ×1 — precisam da BD real / scraper).

## Passos para pôr o WhatsApp a funcionar
1. **Meta / WhatsApp Manager**: criar e aprovar 3 templates, `pt_PT`, sem botões, com o texto **exacto** da aba
   (Configuração → "Textos dos templates"), `{{1}}` = primeiro nome: `satisfacao_comprador`, `satisfacao_vendedor`, `satisfacao_rescisao`.
   Se a Meta obrigar a mudar o texto, actualizar também `satisfacao_config.templates` (o texto semeado na conversa tem de ser igual ao enviado).
2. **agentOS**: rever e fazer merge do branch em `master`; `pytest backend/tests/`; `flyctl deploy --app figueirahome-agentos` (de `backend/`).
   ⚠️ O deploy envia a árvore de trabalho, não o HEAD.
3. **Crons**: registar `satisfacao-enviar` no `figueirahome-crons` (de `crons/`).
4. **Teste**: na aba, pôr o teu telemóvel num inquérito sem email («Mudar telefone»), «Enviar já», esperar o cron (:07) ou chamar
   `POST /api/satisfacao/enviar` com o `X-Automacao-Secret`. Responder 4 num (deve escalar, sem links) e 10 noutro
   (deve mandar Google + testemunho + resumo da venda, se for vendedor).

## Armadilhas
- O conector MCP do Supabase pede confirmação a qualquer SQL com `DROP` e pendura até expirar — usar `create or replace`.
- **Antes de aplicar uma função `fn_satisf_*`/`rpc_satisf_*`, ler a última migração que a define** (`20261021_13` é a mais recente
  para `fn_satisf_para_enviar` e `rpc_satisf_accao`); a 28/10 uma versão antiga foi aplicada por engano e reposta minutos depois.
- `fn_tel9` guarda 9 dígitos: números estrangeiros ficam em erro ("não é telemóvel português") — contactar à mão.
- Google: o pedido vai a todos os de escritura com nota ≥ 7 (não só 9–10), para não ser *review gating*.
- A página do cliente esconde a variação face ao preço inicial quando é negativa; o portal mostra sempre.
- `imovel_precos` só existe desde 03/10/2026 — o preço inicial vem de `Angariação.imovel_venda`.
- `index.html` do portal está em CRLF no `main`.
