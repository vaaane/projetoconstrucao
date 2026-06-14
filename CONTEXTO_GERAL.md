# CONTEXTO GERAL — Minha Casa, Meu Projeto

> Projeto "Construindo Sonhos" — sala de aula interativa / simulação econômica de construção de maquetes.
> Documento de contexto para retomar o desenvolvimento em sessões futuras.
> Última atualização: 13/06/2026.

---

## 1. O que é o projeto

Simulação econômica e imobiliária para alunos. Grupos recebem um **orçamento**, compram
**materiais** numa loja para montar uma **maquete (Fase 5)**, e podem **negociar materiais
entre si** (mercado secundário). É um simulador de mercado aplicado à construção de maquetes.

- **Nome:** Minha Casa, Meu Projeto
- **Subtítulo / tema:** Projeto Construindo Sonhos
- **Público:** alunos (8º/9º ano), trabalho em grupos
- **Projeto Firebase:** `minhacasameuprojeto`
- **Pasta local na máquina:** `C:/sistema_construcao/`

## 2. Stack técnica

- HTML + JavaScript puro (sem framework, sem build), tudo num único arquivo grande (~4.700+ linhas).
- **Firebase Realtime Database** via SDK compat 10.7.1.
- App principal: `index.html` na raiz.
- `maquete_fase5.html` é **BACKUP** — não é o arquivo vivo.
- Estado global em memória na variável `ST` (grupos, materiais, transacoes, propostas, feed,
  config, filaAcoes, limiteDiario, matInfo, historicoPrecos, kits, chamada).

## 3. Deploy (NOVO — jun/2026)

- **Hospedagem:** Cloudflare Pages -> `construindosonhos.pages.dev`.
- **Conectado ao GitHub:** `vaaane/projetoconstrucao`, branch `main`.
- **Deploy automático:** cada `git push` na `main` dispara um novo deploy.
- Antes era Firebase Hosting (com descompasso: `public/` servia placeholder em vez do app).
  Migrado para Cloudflare conectado ao Git. Build output directory = raiz (`/`), framework = None,
  sem build command.
- Workflow de trabalho: editar/commitar/push pelo **Claude Code**.

## 4. Estrutura de telas (single-page, 5 screens)

| ID da tela        | Função                                            |
|-------------------|---------------------------------------------------|
| `screen-landing`  | Entrada / seleção de acesso                       |
| `screen-login`    | Login (aluno / professor)                         |
| `screen-aluno`    | Visão do grupo: loja, carrinho, vendas, inventário|
| `screen-prof`     | Painel do professor (abas: Painel, Ações, Entregas, Rankings, Feed, Relatórios, Turmas, Loja) |
| `screen-tv`       | Modo projetor: ranking ao vivo dos grupos         |

## 5. Modelo de dados (Realtime Database)

```
grupos/{key}/        nome, turma, saldo, saldoInicial, bloqueado, pin, membros{0:nome,1:nome...}, compras, vendas
materiais/{key}/     nome, cat, estoque, preco, acao, raro(bool)
chamada/{turma}/     [ "NOME 1", "NOME 2", ... ]   (array de alunos por turma)
professores/{key}/   nome, turma
turmas/{key}/        nome, prof, grupos[]
propostas/{key}/     status
transacoes/          (lista de compras/vendas; sempre tratado como ARRAY)
feed/                eventos
filaAcoes/, limiteDiario/, historicoPrecos/, kits/, config/, matInfo/
config:              saldoInicial:60000, limiteDiarioPadrao:50, limiteTransacao:20, pausado(bool)
```

## 6. Professores e turmas (atual)

- **Prof. Vanessa -> turma 8F** (32 alunos)
- **Prof. Eliete -> turma 9C** (29 alunos)
- **Prof. Monike -> turma 9F** (31 alunos)
- Removidos os de teste: prof_ana, prof_carlos, prof_lucia.

## 7. Funcionalidades-chave

- **Loja + carrinho:** compra com checkout, controle de estoque, limite diário.
- **Kits de materiais:** criar/editar/comprar/ocultar.
- **Mercado secundário entre grupos:** propostas, contrapropostas, aceitar/cancelar venda, notas fiscais.
- **Painel do professor:** ajustar saldo, estoque, preço; pausar simulação; timer de fase;
  ações de mercado; desfazer última ação. Protegido por PIN/senha.
- **Ranking ao vivo** (tela TV) e rankings por saldo/compras/negociações.

## 8. Decisões e implementações da sessão de jun/2026

### 8.1 Reset da loja
- Ferramenta `limpar_loja_reset.html`: apaga `grupos`, `transacoes`, `propostas`, `feed`,
  `filaAcoes`, `limiteDiario`, `historicoPrecos`; restaura estoque dos materiais; preserva
  `professores`, `turmas`, `kits`, `config`, `matInfo`.
- !! TIRAR DO DEPLOY antes de usar com alunos.

### 8.2 Chamada de alunos (NOVO)
- Nó `chamada/{turma}` com a lista de alunos de cada turma.
- **Criação E edição de grupo** agora selecionam membros por **checkbox da chamada**
  (não mais textarea livre). Aluno já alocado em outro grupo aparece marcado+desabilitado.
- A lista de checkboxes reage à troca de turma no seletor; carrega já na abertura do card.
- Painel **"alunos sem grupo"** por turma (compara chamada x membros de todos os grupos).
- Professor pode **adicionar aluno à chamada** da própria turma (permanente, com checagem de
  duplicata, só o dono da turma). `ST.chamada` é populado por listener.

### 8.3 saldoInicial por grupo
- Novo campo `grupos/{key}/saldoInicial`, gravado na criação do grupo.
- Função `migrarSaldoInicial()` (exposta em `window`, rodar no console uma vez) preenche
  grupos antigos: `saldoInicial = saldo atual`.
- Cálculo de gasto/% usa `g.saldoInicial ?? g.saldo ?? SI()` (nunca o fixo 60.000).
- Corrigido em ~10 locais (cálculo + textos de exibição + cor do indicador). Os placeholders
  de formulário de criação seguem usando `SI()` (valor sugerido), intencionalmente.

### 8.4 Item raro (regra: por ESTOQUE, sem tempo)
- Item raro é **liberado manualmente** pelo professor (`liberarItemRaro`), separado de ações de mercado.
- `liberarItemRaro` grava `acao = {tipo:'raro', pct:0, auto:false}` **sem `fimTs`** (sem prazo).
- Fica disponível **até o estoque zerar** (é item único -> "primeiro a comprar leva").
- `verificarExpiracoes` **ignora** ações tipo `'raro'` (não expira por tempo).
- Helper único `isRaroAtivo(m) = m.raro && m.estoque>0 && m.acao?.tipo==='raro'` e
  `getRarosAtivos()` usados em TODOS os pontos (vitrine do aluno, loja, notificação, Painel, Feed)
  para não divergirem.

### 8.5 Ações automáticas de mercado
- O gerador é `verificarAcaoAutomatica` (roda a cada 60s) — separado do item raro.
- **Guardas adicionadas:** só gera/ativa ação automática se simulação **não** pausada
  (`ST.config.pausado !== true`) **E** houver pelo menos um grupo desbloqueado.
- `ativarProximaDaFila(isAuto)`: quando `isAuto=true` (chamado por expiração) aplica as guardas;
  chamada manual do professor não tem guarda.

### 8.6 Login persistente (auto-login)
- Lembra o último acesso (professor OU grupo) e entra direto, **sem pedir PIN**.
- Usa **`sessionStorage`** (isola por aba, sobrevive ao F5, limpa ao fechar navegador) — escolhido
  por causa do cenário multi-aba de teste e sala de aula.
- Flag `LEMBRAR_ACESSO = true` no topo do script (mudar para `false` volta a pedir login sempre).
- Não salva o PIN; valida se a identidade ainda existe; botão "Sair" limpa a memória.
- Para persistir entre dias (após fechar o navegador), trocar `sessionStorage` por `localStorage`.

### 8.7 Robustez do estado vazio
- `ST.transacoes` inicializado como `[]` (era `{}`), e todos os usos protegidos com
  `(ST.transacoes||[])` — evita `TypeError: .filter is not a function` no estado zerado
  (que é o estado em que os alunos começam).

### 8.8 Limpezas de UI
- Removidos da landing os botões admin **"Inicializar BD"** e **"Resetar"**.
- Removido do painel o bloco **"PROFESSORES"** (lista + criar perfil). As funções JS
  (`criarPerfProf`, `abrirEditarProf`, `salvarEdicaoProf`) foram **preservadas**, só tiradas da tela.

### 8.9 Exclusão de grupo e login (correções finais)
- **Exclusão de grupo** (`excluirGrupoFB`) agora remove o grupo de `grupos/{key}` **E** remove o
  nome do array `turmas/{turmaKey}/grupos` (update multi-path). Antes deixava grupo órfão visível
  no login (login lista a partir do array `t.grupos` da turma).
- Função `limparGruposOrfaos()` (console): percorre `turmas/{key}/grupos` e remove nomes que não
  existem mais em `grupos`. Rodada uma vez para limpar resíduos (ex: grupo "B").
- **Login não expõe mais o saldo:** removido o valor em R$ ao lado do nome do grupo na tela de
  seleção de acesso. O dot colorido de status (calculado pelo saldo) permanece — não revela valor.

## 9. Ferramentas de administração (manter FORA do alcance dos alunos)

- `limpar_loja_reset.html` — reset completo da loja.
- `migrarSaldoInicial()` — console; preenche saldoInicial de grupos antigos.
- `limparMercadoEFeed()` — console; zera ações de mercado (!= raro) e apaga o feed.
- `limparGruposOrfaos()` — console; remove nomes de grupo órfãos dos arrays das turmas.
- Funções de professor preservadas mas sem botão na tela (criar/editar professor).
- !! Decidir destino: mover para uma página de admin separada e fora do deploy.

## 10. Pendências / próximos passos

1. **Tirar `limpar_loja_reset.html` do deploy** antes do uso real com alunos.
2. **Reset final** (loja + feed + mercado) imediatamente antes da primeira aula com a turma.
3. Decidir destino das funções de admin preservadas (página separada vs remoção).
4. (Opcional, futuro) Extrair o JavaScript para arquivos `.js` separados se o arquivo único
   incomodar — manter estado global compartilhado, sem módulos ES, sem build.

## 11. Notas operacionais aprendidas

- **Cache do navegador engana:** ao testar mudanças, usar "Empty Cache and Hard Reload"
  (DevTools aberto -> botão direito no reload). Mais de uma vez um "bug" era só cache.
- **`node --check`** confirma sintaxe do arquivo (exit 0 = sem SyntaxError).
- Erros `404` de `logo.png`/`inicial2.png`/`favicon.ico` em localhost são inofensivos.
- `sessionStorage` isola por aba; `localStorage` é compartilhado entre abas do mesmo domínio.
- Dado órfão clássico: nomes em `turmas/{key}/grupos` sobrevivem se só o nó `grupos` é apagado.
  Sempre limpar os dois lugares.
