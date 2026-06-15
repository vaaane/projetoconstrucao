# CONTEXTO GERAL — Minha Casa, Meu Projeto (Construindo Sonhos)

> Sala de aula interativa / simulação econômica de construção de maquetes.
> Documento de contexto detalhado para retomar o desenvolvimento sem reintroduzir erros.
> Última atualização: 14/06/2026.

---

## 1. O QUE É

Simulação econômica para alunos. Grupos recebem um **orçamento**, compram **materiais** numa
loja para montar uma **maquete (Fase 5)**, e podem **negociar materiais entre si** (mercado
secundário). Professor controla a simulação (pausar/retomar), libera itens raros, ajusta
saldo/estoque, confirma entregas e cancela notas. Há um **Painel TV** para projeção em sala.

- **Projeto Firebase:** `minhacasameuprojeto`
- **Pasta local:** `C:/sistema_construcao/`
- **Deploy:** Cloudflare Pages → `construindosonhos.pages.dev`, conectado ao GitHub
  `vaaane/projetoconstrucao` (branch `main`). Push na main = deploy automático.
- **Build output:** raiz (`/`), framework None, sem build command.

## 2. STACK / ARQUITETURA

- HTML + JavaScript puro, SEM framework, SEM build, SEM módulos ES. Tudo em `index.html` (~4.900 linhas).
- **Firebase Realtime Database** (SDK compat 10.7.1).
- Estado global em memória na variável **`ST`** (objeto). Listeners do Firebase populam `ST.*`.
- App principal: `index.html` na raiz. `maquete_fase5.html` é BACKUP (ignorar).
- **Workflow:** implementação pelo **Claude Code** (tem a versão atual do arquivo); o chat web
  para análise, decisão e geração de prompts/trechos. SEMPRE pedir ao Claude Code o código atual
  de um trecho antes de montar substituição — o arquivo muda e os números de linha desatualizam.

### IMPORTANTE — campos do ST inicializados como ARRAY
- `ST.transacoes` DEVE ser `[]` na inicialização (era `{}` e quebrava `.filter`). Todos os usos
  protegidos com `(ST.transacoes||[])`. Atenção a qualquer campo novo usado como array.

## 3. TELAS (single-page, screens)

- `screen-landing` — seleção de acesso (lista professores e grupos; NÃO mostra saldo).
- `screen-login` — login.
- `screen-aluno` — visão do grupo (abas: Início, Loja, Carrinho, Mercado, Rankings, Feed,
  Notas Fiscais, Relatório, Orçamento, Diário).
- `screen-prof` — painel do professor (abas: Painel, Ações, Entregas, Rankings, Feed, Relatórios, Turmas, Loja).
- `screen-tv` — Painel TV (projetor). REDESENHADO (ver seção 9).

## 4. MODELO DE DADOS (Realtime Database)

```
grupos/{key}/        nome, turma, saldo, saldoInicial, bloqueado, pin,
                     membros{0:nome,1:nome...}, compras/{cKey}, vendas/{vKey}
   compras/{cKey}/   { id:'NF-XXXXX', data, itens{0:{nome,qty,preco}}, total,
                       entregue:bool, cancelada:bool, ts, [via:'mercado'|'revenda', de:nome] }
materiais/{key}/     nome, cat, estoque, preco, acao, raro(bool)
   acao = ações de mercado: { tipo:'oferta'|'alta'|'escassez'|'indisponivel', pct, fimTs, auto }
        = item raro liberado: { tipo:'raro', pct:0, auto:false }  ← SEM fimTs (não expira por tempo)
chamada/{turma}/     [ "NOME 1", "NOME 2", ... ]   (array de alunos)
professores/{key}/   nome, turma
turmas/{key}/        nome, prof, grupos[]   (array de NOMES de grupos)
propostas/{key}/     status:'aberta'|'aceita'|...   {vendedor, qty, item, precoUn}
transacoes/          lista de trocas de mercado {comprador, vendedor, item, qty, total, via, ts}
feed/                eventos { tipo:'compra'|'mercado'|'entrega'|..., txt, data, ts }
historicoPrecos/, kits/, config/, matInfo/
config:              saldoInicial:60000 (padrão sugerido), pausado(bool)
```
- REMOVIDOS do sistema: `limiteDiario` (nó), `config.limiteDiarioPadrao`, `config.limiteTransacao`,
  `config.timerFim` (ver 8.1 e 8.6). NÃO reintroduzir.

## 5. PROFESSORES / TURMAS / CHAMADA (atual)

- **Vanessa → 8F** (32 alunos), **Eliete → 9C** (29), **Monike → 9F** (31).
- Chamada de cada turma em `chamada/{turma}` (array). `ST.chamada` populado por listener.
- Removidos professores de teste: prof_ana, prof_carlos, prof_lucia.

## 6. GESTÃO DE GRUPOS

- **Criação E edição de grupo:** membros selecionados por **checkbox da chamada** (NÃO textarea
  livre). Aluno já alocado em outro grupo da turma aparece marcado+desabilitado (trava 1-grupo-só).
- A lista de checkboxes reage à troca de turma no seletor e carrega na abertura do card.
- Membros gravados como `{0:nome,1:nome...}`, lidos com `Object.values(g.membros||{}).filter(Boolean)`.
- **Painel "alunos sem grupo"** por turma (chamada × união de membros dos grupos).
- Professor pode **adicionar aluno à chamada** da própria turma (permanente, checa duplicata, só dono).
- **Exclusão de grupo (`excluirGrupoFB`):** remove de `grupos/{key}` E remove o nome do array
  `turmas/{turmaKey}/grupos` (multi-path) — senão o grupo vira órfão e reaparece no login.
- `limparGruposOrfaos()` (console): remove nomes órfãos dos arrays das turmas.
- **Login NÃO expõe saldo** (removido valor em R$; mantém só nome + dot de status).

## 7. SALDO / GASTO

- Cada grupo tem `saldoInicial` (gravado na criação). `migrarSaldoInicial()` (console, em `window`)
  preenche grupos antigos: `saldoInicial = saldo atual`.
- Cálculo de gasto/%: SEMPRE `g.saldoInicial ?? g.saldo ?? SI()`. NUNCA o fixo `SI()`(60.000) sozinho.
- Corrigido em ~10 locais (cálculo + textos de exibição + cor de indicador). Placeholders de
  formulário de criação usam `SI()` intencionalmente (valor sugerido).

## 8. SIMULAÇÃO, COMPRAS, MERCADO, ITEM RARO, NOTAS

### 8.1 Controle da simulação
- ÚNICO controle: **Pausar/Retomar** no Painel do professor (`config.pausado`).
- O antigo "Timer da Simulação" (cronômetro de fase) foi **REMOVIDO por completo** (card, funções
  `iniciarTimer`/`confirmarIniciarTimer`/`confirmarEncerrarTimer`, banners de tempo no aluno, timer
  da TV, `config.timerFim`, `dashTimerIv`). NÃO reintroduzir. `verificarExpiracoes` só trata ações
  de mercado (`acao.fimTs`), nunca timer.

### 8.2 Bloqueio quando pausada
- Função guard `simulacaoBloqueada()` (retorna true se `config.pausado===true`, com toast).
- Aplicada no INÍCIO de cada ação que mexe em saldo/estoque (não só no botão):
  `checkoutFromCart`, `confirmarCompraKit`, `finalizarCompra`, `criarVenda`, `aceitarVenda`,
  `publicarOferta`, `aceitarProposta`, `enviarContraproposta`, `aceitarContraproposta`.
- `recusarContraproposta` e `cancelarProposta` NÃO bloqueiam (não transferem nada).
- Ações do PROFESSOR (painel) funcionam mesmo pausado — não bloquear.

### 8.3 Ações automáticas de mercado
- Gerador: `verificarAcaoAutomatica` (a cada 60s). Guard: só gera se `pausado!==true` E houver
  grupo desbloqueado. `ativarProximaDaFila(isAuto)` aplica guard quando `isAuto=true`; chamada
  manual do professor não tem guard.

### 8.4 Item raro (regra: por ESTOQUE, NUNCA por tempo)
- Liberado MANUALMENTE pelo professor (`liberarItemRaro`), grava `acao={tipo:'raro',pct:0,auto:false}`
  SEM `fimTs`. É SEPARADO de ações de mercado.
- Helper único: `isRaroAtivo(m) = m.raro && m.estoque>0 && m.acao?.tipo==='raro'` e `getRarosAtivos()`.
- USAR esse helper em TODOS os pontos (vitrine aluno, loja, notificação, Painel, Feed, TV). NUNCA
  usar só `m.raro && m.estoque>0` (flag permanente → mostra raro sem liberação = bug recorrente).
- `verificarExpiracoes` IGNORA `tipo==='raro'` (não expira por tempo). Some só quando estoque zera.

### 8.5 Notas fiscais — cancelamento (só LOJA)
- Estados: `entregue` (bool) + `cancelada` (bool). Cancelada não conta como pendente nem entregue.
- `cancelarNotaFiscal(gKey, cKey, modo)`: estorno COMPLETO — devolve `total` ao saldo + repõe
  estoque de cada item (`mKey` resolvido por `m.nome===nome`) + marca `cancelada:true`. Multi-path.
  Funciona para nota aguardando OU entregue. Guard `isMeu`. Confirmação antes. NÃO permite recancelar.
- SÓ notas de loja (sem `via`). Notas de mercado (com `via`): botão de cancelar NÃO aparece.
- UX: nota **aguardando** → botão "✕ Recusar compra" (modo='recusa'); nota **entregue** → "✕"
  discreto no canto superior direito do card (modo='cancelamento').
- Selo no card (`renderNF`), prioridade: `cancelada` → "❌ Compra cancelada"; senão `entregue` →
  "✓ Entregue"; senão "⏳ Aguardando entrega".
- Bloco "❌ Canceladas" em Notas Fiscais (aluno) e Entregas (professor).
- Filtro `!c.cancelada` aplicado em TODOS os contadores/estatísticas (calcStats, renderPainelProf
  ×3, renderEntregasProf, renderRelatoriosProf, "Últimas compras" + sort). Nota cancelada não conta.
- ATENÇÃO: ao mexer em contadores de compras, sempre verificar se há `!c.cancelada` ao lado de
  qualquer `!c.entregue` ou `compras.length`.

### 8.6 Limite de compra — REMOVIDO
- NÃO há mais limite diário nem cap por transação. Aluno compra QUALQUER quantidade, limitado
  SÓ pelo estoque (`qty > m.estoque` e `qty <= 0` permanecem). Controle de abuso = professor
  recusar/cancelar a compra.
- Removidos: `ST.limiteDiario`, listener `limiteDiario`, `config.limiteDiarioPadrao`,
  `config.limiteTransacao`, exibição "X/20 hoje", badge "Limite" no card, "Hoje: X/Y (restam)" no
  carrinho, e as gravações/checagens em `finalizarCompra`/`confirmarCompraKit`/`checkoutFromCart`.
- NÃO confundir com `atualizarLimiteVenda` (limite de REVENDA entre grupos, ~20% do inventário
  próprio) — esse é OUTRO conceito e PERMANECE.

## 9. PAINEL TV (redesenhado — telão de competição)

- Filosofia: tela de PROJETOR parada. `height:100vh; overflow:hidden` — ZERO scroll, tudo numa tela.
- Estrutura: cabeçalho (logo + relógio `tv-relogio` + botão Som) / faixa de ALERTAS no topo /
  2 colunas: Feed ao vivo (1.5fr) + "Situação dos grupos" (1fr).
- **Alertas (prioridade visual):** item raro (usar `isRaroAtivo`!), estoque baixo, ação de mercado.
  Cards pulsam (`@keyframes tvPulse`). Some a faixa se não houver alerta.
- **Estoque baixo:** limiar FIXO — `m.estoque>0 && m.estoque < LIMIAR_ESTOQUE_BAIXO` (=10). NÃO usar
  `ESTOQUE_INICIAL[mKey]||100` (o fallback 100 dava falso alarme). (Melhoria futura: `estoqueInicial`
  por material + migração, p/ limiar proporcional.)
- **Situação dos grupos:** NEUTRO — só nome + saldo, ordenado por saldo, SEM troféu/medalha/pódio/
  cor de julgamento. Decisão pedagógica: gastar muito NÃO é vitória; o sistema não julga, o professor
  conduz a leitura. Título "Situação dos grupos" (não "Ranking").
- **Feed:** últimos ~6 itens, barra colorida por tipo (compra=verde, mercado=âmbar, entrega=azul).
- **SOM (já existe, reaproveitado):** `ativarSomTV(btn)` destrava AudioContext no clique (browser
  bloqueia autoplay). `tvSound(tipo)` toca presets: 'raro','stock','acao','mercado'. `playTone(...)`
  via Web Audio API. `checkTvSounds(items)` dedup por Set `_tvSoundedKeys` — toca 1× por alerta.
  Eventos com som no TV: item raro, estoque baixo, ação de mercado (SEM som de compra, por decisão).
- IDs usados: `tv-alertas`, `tv-feed-wrap`, `tv-grupos-wrap`, `tv-relogio`. IDs antigos removidos:
  `tv-acoes-wrap`, `tv-rank-wrap`, `tv-fila-wrap`.

## 10. VISÃO DO ALUNO — AJUSTES VISUAIS

- Banners de status: UM por vez. Se `pausado` → só "⏸ Simulação pausada" (azul calmo #eff6ff/#1e40af).
  (O timer foi removido; não há mais "tempo restante"/"tempo encerrado".)
- **Rankings do aluno (`renderRankings`): TRÊS LISTAS NEUTRAS** (Saldo, Compras, Negoc.), igual ao
  TV. SEM pódio, SEM troféu, SEM medalhas. Cada linha: posição em número simples ("1.", cinza), nome,
  turma, barra cinza neutra (#cbd5e1, mesma cor p/ todos) e valor. Destaque do "meu grupo" mantido
  (fundo azul claro + ◀), sem conotação de vitória. (O pódio 🥇🥈🥉 que existia foi REMOVIDO.)
- **Loja:** removido o link "ver exemplos ↗" dos cards. Cards mostram nome, categoria, preço,
  "X un. em estoque" e botão de adicionar. (Limite diário/badge "Limite" também removidos — ver 8.6.)
- Cabeçalho do aluno: logo compacto. (Pendente: media query pra encolher só no celular — não aplicado.)

## 11. FERRAMENTAS DE ADMIN (manter FORA do alcance dos alunos)

- `limpar_loja_reset.html` — reset completo da loja (apaga grupos, transacoes, propostas, feed,
  filaAcoes, limiteDiario, historicoPrecos; restaura estoque; preserva professores/turmas/kits/
  config/matInfo). Inclui esvaziar `turmas/*/grupos`.
- `migrarSaldoInicial()` — console; saldoInicial dos grupos antigos.
- `limparMercadoEFeed()` — console; zera ações de mercado (≠raro) e apaga feed.
- `limparGruposOrfaos()` — console; remove nomes de grupo órfãos das turmas.

## 12. PENDÊNCIAS / PRÓXIMOS PASSOS

1. **Tirar `limpar_loja_reset.html` do alcance dos alunos** antes do uso real.
2. **Reset final** (loja + feed + mercado) antes da 1ª aula.
3. Decidir destino das funções de admin preservadas (criar/editar professor — sem botão hoje):
   página de admin separada vs remoção.
4. Substituição do logo menor SÓ no celular (media query) — não aplicada.
5. (Futuro) `estoqueInicial` por material (limiar de estoque baixo proporcional) — hoje fixo (<10).
6. (Futuro) Extrair JS para arquivos `.js` separados se o arquivo único incomodar (manter estado
   global, sem módulos ES, sem build).
7. Commit pendente das últimas mudanças (telão, rankings neutros, remoção de limites, ver exemplos).

## 13. NOTAS OPERACIONAIS (lições aprendidas)

- **CACHE ENGANA:** sempre testar com "Empty Cache and Hard Reload" (DevTools aberto → botão direito
  no reload). Várias vezes um "bug" era só cache servindo versão antiga.
- **`node --check`** confirma sintaxe (exit 0 = sem SyntaxError). Rodar após remoções em massa.
- Após remover variável/função, **buscar referências órfãs** no arquivo todo (ex: `timerFim`,
  `dashTimerIv`, `limiteDiario`) — uma referência órfã quebra a função inteira.
- Erros `404` de `logo.png`/`inicial2.png`/`favicon.ico` em localhost são INOFENSIVOS.
- `sessionStorage` isola por aba (sobrevive a F5, limpa ao fechar); `localStorage` é compartilhado.
  Auto-login usa sessionStorage (flag `LEMBRAR_ACESSO`). NÃO salva PIN; "Sair" limpa; valida se a
  identidade ainda existe.
- **Descompasso nome↔chave** é fonte recorrente de bug: nota guarda nome do material ("Palha Seca"),
  estoque é indexado por chave ("Palha_seca"); turma tem nome legível ("8F") e key de nó. Sempre
  resolver com match/normalização, nunca assumir igualdade direta.
- **Padrão de dado órfão:** apagar um nó (grupos) mas deixar referência em outro (turmas/*/grupos).
  Sempre limpar os dois lados.
- **Padrão "bug só no estado vazio":** vários bugs (transacoes.filter, item raro sem liberação,
  contadores) só aparecem com banco zerado — que é como os alunos começam. Testar SEMPRE no estado
  zerado antes do uso real.
- Ao adicionar feature que conta/lista compras: lembrar do filtro `!c.cancelada`.

## 14. ESTILO / PADRÕES DE CÓDIGO

- Cores: navy (`--c-navy`), laranja (`--c-orange`), verde saldo `#16a34a`. TV é tema escuro `#0f1729`.
- Confirmações destrutivas usam `confirm()` nativo (padrão do sistema). Toasts via `showToast`.
- Sem `position:fixed` (quebra layout) — usar `position:absolute` dentro de `position:relative`.
- Formatação de moeda: função `fmt()`.
- Filosofia transversal: o sistema MOSTRA os fatos (saldo, gasto, estoque), NÃO julga. As decisões
  de valor (quem planejou bem, recusar compra abusiva) ficam com o professor. Por isso: nada de
  troféu/competição nos rankings, e nenhum limite automático de compra (professor recusa se preciso).
- Renders principais: `renderTV`, `renderRankings`, `renderNotasAluno`, `renderEntregasProf`,
  `renderPainelProf`, `renderTurmasProf`, `renderLoja`, `renderCarrinho`, `renderDashAluno`, `calcStats`.
