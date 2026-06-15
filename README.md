# PM Hub — README

**Shopee BR Product · PM Hub Challenge**

PM Hub é uma ferramenta de gestão de portfólio de produto em **um único arquivo HTML**, sem
dependências obrigatórias, com dados persistidos no `localStorage` do navegador. Este
documento cobre as três entregas do desafio:

- **Missão A** — Bug Hunt: bugs corrigidos e os prompts usados.
- **Missão B** — Solução multi-usuário (autenticação, identidade e visões por papel).
- **Missão C** — Passo a passo de publicação (GitHub Pages + Cloudflare + Google).

> **Como rodar agora:** abra o `index.html` no navegador. Tudo funciona offline, em modo
> demonstração (login simulado, picker de arquivos demo e insights determinísticos). As
> integrações reais (Google e Claude) são opcionais e descritas na Missão C.

---

# Missão A — Bug Hunt

Bugs encontrados no `Shopee PM Hub.html` original, o prompt usado para corrigir cada um e o
resultado. Todos foram verificados no arquivo entregue. Cada prompt nomeia as funções a
alterar e as que **não** devem ser tocadas, seguindo o Mapa Frontend ↔ Código.

## A. Integridade de dados

### A1 — Formatos de ETA fora do padrão quebram ordenação, overdue e a coluna de data
**Onde:** seed + `fmtD()`. Vários projetos guardavam ETAs como texto livre — `"23/04/26"`,
`"28/01/2026"`, `"3/Fev/26"`, `"End of May/26"`, `"Q2 2026"`, `"Lived on Nov/25"`. O `fmtD()`
só entende `YYYY-MM-DD`; para o resto, devolve a string crua. A coluna de ETA mistura datas
formatadas com texto livre, e a ordenação por data e a lógica de overdue quebram.

> Prompt: Os dados do PM Hub têm ETAs em formatos inconsistentes: "23/04/26", "28/01/2026",
> "3/Fev/26", "End of May/26", "Q2 2026", "Lived on Nov/25". O fmtD() só trata YYYY-MM-DD,
> então o resto passa como texto cru e quebra a ordenação por data e a detecção de overdue.
> Normalize todos os campos de data do seed para ISO YYYY-MM-DD. Quando o projeto só existia
> como quarter ("Q2 2026"), preserve o rótulo original num campo separado para não perder a
> informação (ver B5). Não mexa no fmtD() ainda — primeiro os dados, depois o parsing.

**Resultado:** todos os `liveETA`/`localETA` são ISO válido; nenhuma data em texto livre
permanece. Coluna de ETA, ordenação e overdue ficaram consistentes.

### A2 — Projeto 58 com ETA copiada do campo errado
**Onde:** seed, id 58 — `"localETA": "AMS Affiliate Ranking Optimization"` (o `initiative`
colado no campo de data), renderizando como string longa na coluna Local ETA.

> Prompt: No seed do PM Hub, o projeto 58 tem o localETA igual a "AMS Affiliate Ranking
> Optimization" — é o nome do initiative colado no campo de data por engano. Troque pela data
> ISO correta (ou null se desconhecida) e audite os outros projetos procurando o mesmo padrão
> de campo trocado. Mexa só nos dados, não nas funções de render.

**Resultado:** id 58 com data válida; nenhum projeto mostra texto de initiative numa coluna
de data.

### A3 — Typo "TDB" no lugar de "TBD" no projeto 42
**Onde:** seed, projeto 42 — `"liveETA": "TDB"`. Pequeno, mas criava valor-fantasma em
filtros e relatórios.

> Prompt: O projeto 42 no seed do PM Hub tem liveETA "TDB" — typo de "TBD". Corrija e procure
> no dataset typos parecidos em valores de status/ETA/priority, para os filtros e relatórios
> não exibirem buckets-fantasma. Só dados.

**Resultado:** nenhum `TDB` permanece; sem bucket-fantasma.

## B. Datas e lógica

### B1 — Overdue e quarters do roadmap errados em UTC-3 (fuso horário)
**Onde:** `getQ()`, `getMK()` e as contagens de overdue/`up30`. Eles parseiam datas com
`new Date(p.liveETA)` (sem hora), que o JS lê como **meia-noite UTC**. No Brasil (UTC-3) isso
cai no dia anterior: um projeto com ETA **hoje** vira atrasado, e um ETA no 1º de um quarter
(1/Jan, 1/Abr, 1/Jul, 1/Out) cai no **quarter anterior**. Pior: `buildExpand()` e
`renderDashboard()` parseiam do jeito correto (`+'T00:00:00'`), então o mesmo projeto aparece
"atrasado" na tabela e "no prazo" no próprio painel.

> Prompt: O PM Hub tem um bug de fuso. Datas em YYYY-MM-DD são parseadas com new Date(str) em
> alguns lugares (getQ, getMK, contagens de overdue e up30) — o JS trata como UTC, então em
> UTC-3 um projeto com ETA hoje aparece atrasado e o roadmap joga datas de início de quarter
> no quarter anterior. Outros lugares (buildExpand, renderDashboard) já parseiam local com
> new Date(str+'T00:00:00'). Padronize TODO o parsing de YYYY-MM-DD para hora local em getQ(),
> getMK(), renderRoadmap() e nas contagens de overdue/up30. Não altere os dados e não toque em
> renderOrgChart().

**Resultado:** parsing local em todos os pontos. Projeto com ETA hoje não é mais falsamente
atrasado e os quarters do roadmap batem com o ETA.

### B2 — Check de overdue retorna `NaN` para ETAs não-ISO (projetos nunca marcados)
**Onde:** `new Date(p.liveETA+'T00:00:00')` gera Date inválida para `"TBD"`, `"Q2"`, `"N/A"`,
`"Late Jan"`. Um valor como `"23/04/26"` cria data inválida e o projeto **nunca** é marcado
como atrasado mesmo estando.

> Prompt: A lógica de overdue do PM Hub gera Date inválida para ETAs não-ISO ("TBD", "Q2",
> "Late Jan", "23/04/26"), então esses projetos nunca são marcados como atrasados. Deixe a
> detecção robusta: parseie só datas ISO válidas, trate o resto como "sem data" (não atrasado,
> mas visualmente distinto) e nunca deixe uma data inválida passar pelo check < today.

**Resultado:** com o seed normalizado (A1) e o parsing endurecido, overdue só usa datas
válidas; ETAs sem data aparecem como "sem data", nunca como falso negativo.

### B3 — Projetos Frozen mostram barra de progresso a 0%
**Onde:** `progPct()` usa `PIPE.findIndex()`, que retorna `-1` para status do `FROZEN` →
`Math.round((-1+1)/8*100) = 0%`. Um projeto que chegou em "Live Testing" e foi congelado
mostra 0%, apagando o histórico.

> Prompt: No PM Hub, o progPct() retorna 0% para projetos frozen porque PIPE.findIndex()
> devolve -1 para status do FROZEN. Trate projetos frozen como estado visual distinto em vez
> de uma barra enganosa de 0% — mostre uma tag de frozen e mantenha o último progresso
> conhecido. Mexa só no progPct() e na renderização da barra; não altere PIPE nem FROZEN.

**Resultado:** projetos frozen com tag ❄ e sem barra enganosa de 0%.

### B4 — Filtro do card do Dashboard desativa a busca e os chips de filtro
**Onde:** clicar num card de ETA seta `customFilter`; o `getFiltered()` começa com
`if(customFilter) return projects.filter(customFilter)`, retornando **antes** de aplicar a
busca e os chips. Depois de clicar num card, busca e chips parecem ativos mas não fazem nada.

> Prompt: No PM Hub, depois que clico num card de ETA do Dashboard a busca e os chips param de
> funcionar até eu limpar tudo. A causa é o getFiltered() retornar cedo quando customFilter
> está setado. Faça o customFilter ser só mais um critério que se combina com a busca e os
> chips, em vez de substituí-los. Mantenha os cards e o Clear Filters como antes. Só o
> getFiltered().

**Resultado:** filtros de card se combinam com busca e chips; controles continuam ativos.

### B5 — Editar projeto com ETA por quarter perde a informação
**Onde:** o campo de ETA é `<input type="date">`, mas alguns projetos foram criados com alvo
de quarter (`"Q2 2026"`). Abrir um desses para editar limpa o quarter ao salvar.

> Prompt: Alguns projetos do PM Hub miram um quarter, não um dia (ex.: "Q2 2026"), mas o campo
> de ETA do modal é <input type="date">, então editar derruba o quarter ao salvar. Adicione um
> modo de ETA quarter/texto: mantenha o rótulo de quarter num campo próprio, deixe escolher
> "dia" ou "quarter" no modal e faça o round-trip dos dois sem perda. Só openModal()/
> saveModal() e o markup do modal.

**Resultado:** a versão mantém um campo `quarter` + rótulo cru (`localETARaw`); ETAs por
quarter sobrevivem à edição.

### B6 — `editId` desatualizado pode sobrescrever um projeto existente
**Onde:** abrir um projeto para editar (seta `editId`), fechar com Esc sem salvar e abrir o
modal de novo projeto pelo atalho `N` — o `editId` continua setado e o Save pode sobrescrever
o projeto editado.

> Prompt: No PM Hub, se eu abro um projeto para editar (seta editId), fecho com Esc sem salvar
> e abro o modal de novo projeto, o editId continua setado e o Save sobrescreve o existente em
> vez de criar um novo. Faça o "novo projeto" sempre resetar editId para null ao abrir, e o
> Esc/fechar também limpar editId. Só openModal()/openNewModal()/closeModal().

**Resultado:** criar e editar ficaram separados; refazer os passos não sobrescreve nada.

## C. UI / UX

### C1 — Descrição truncada no meio da frase, sem ellipsis CSS
**Onde:** descrições cortadas com substring de JS, deixando o `…` no meio da palavra.

> Prompt: O PM Hub trunca as descrições com substring de JS, então o "..." cai no meio da
> palavra. Troque por um line-clamp CSS (2 linhas), pra o ellipsis ficar limpo e o texto
> completo continuar no DOM para tooltip/expandir. Só estilo + a renderização do snippet.

**Resultado:** descrições com `line-clamp` CSS; ellipsis limpo e texto completo preservado.

### C2 — Atalho de teclado "N" dispara enquanto se digita
**Onde:** o handler global abre o modal no `N` sem checar o foco — apertar N na busca ou num
campo de tarefa abre o modal.

> Prompt: O atalho "N" do PM Hub abre o modal de novo projeto mesmo enquanto digito na busca
> ou num campo de tarefa. Proteja todos os atalhos: ignore-os quando o target do evento for um
> INPUT, TEXTAREA ou contentEditable. Só o handler de teclado.

**Resultado:** atalhos ignorados durante a digitação
(`/INPUT|TEXTAREA/.test(target.tagName) || target.isContentEditable`).

### C3 — Lembrete abre o Google Calendar em nova aba sem perguntar
**Onde:** o `addReminder()` abria uma aba do Calendar automaticamente ao salvar.

> Prompt: O addReminder() do PM Hub abre uma aba do Google Calendar automaticamente quando
> adiciono um lembrete. Torne o hand-off explícito: salve o lembrete e ofereça um botão
> "Adicionar e abrir no Calendar" que o usuário escolhe clicar. Não mude como os lembretes são
> armazenados.

**Resultado:** hand-off pro calendário virou botão explícito ("Add & Open Calendar"), já
linkado com data, horário e o nome do projeto como título da reunião.

### C4 — Heatmap conta o mesmo projeto em vários meses
**Onde:** o `renderHeatmap()` percorria `localETA` e `liveETA`, contando um projeto em dois
buckets de mês.

> Prompt: O heatmap do PM Hub conta em dobro: soma o projeto no mês do localETA e no mês do
> liveETA. Escolha uma única métrica por célula (ex.: entregas por PM por mês, pela data em que
> o projeto foi Live) e conte cada projeto uma vez. Só o renderHeatmap().

**Resultado:** heatmap refeito como "entregas por PM por mês", contando cada entrega uma vez.

### C5 — "Reports To" mostra o nível do gestor, não o do membro
**Onde:** o modal de Team exibia "L2 — Matheus" (nível do gestor) onde deveria refletir o
membro gerenciado.

> Prompt: No modal de Team do PM Hub, a linha "Reports To" mostra o nível do gestor (ex.: "L2 -
> Matheus") em vez de estar escopada ao membro editado. Corrija o rótulo. Só openTeamModal()/
> tmLevelChange() — não toque no renderOrgChart().

**Resultado:** o nível na linha Reports-To do modal de Team está correto.

### C6 — Organograma deixa uma faixa de L3 vazia na home
**Onde:** o `renderTeam()` renderiza a faixa de um nível mesmo quando o L3 está vazio.

> Prompt: O organograma do PM Hub renderiza a faixa de um nível mesmo vazia (L3 fica em branco
> na home). Esconda qualquer faixa de nível sem membros. Só o layout do renderTeam() — o
> renderOrgChart() continua chamado via renderTeam().

**Resultado:** organograma sem faixa de nível vazia.

### C7 — Release notes cortam updates no meio da palavra em 120 chars
**Onde:** o `generateReleaseNotes()` truncava cada update em exatamente 120 caracteres.

> Prompt: O generateReleaseNotes() do PM Hub corta cada update em exatamente 120 caracteres,
> partindo palavras. Trunque numa fronteira de palavra (ou frase completa) com ellipsis limpo.
> Só o generateReleaseNotes().

**Resultado:** trechos de release note terminam numa fronteira de palavra.

### C8 — Command palette não acha projetos por descrição
**Onde:** a paleta buscava só nome e PM.

> Prompt: A command palette do PM Hub só casa nome e PM, então não acho um projeto por uma
> palavra da descrição. Estenda a busca para casar também descrição e product line. Só o
> renderPalette()/o filtro da paleta.

**Resultado:** uma palavra-chave só da descrição faz o projeto aparecer na paleta.

### C9 — Botão de star abre o modal do projeto no mobile (event bubbling)
**Onde:** tocar na star às vezes disparava o handler de abrir-modal da linha no mobile.

> Prompt: No mobile, tocar na star às vezes abre o modal de edição porque o clique propaga pra
> linha. Pare a propagação no controle de star. Só o toggleStar()/a ligação do botão.

**Resultado:** em viewport estreito a star alterna sem abrir o modal.

### Resumo (18 bugs corrigidos e verificados)

| # | Bug | Tipo |
|---|-----|------|
| A1 | Formatos de ETA fora do padrão | Dados |
| A2 | Projeto 58 com ETA de campo errado | Dados |
| A3 | Typo "TDB" | Dados |
| B1 | Overdue/quarter errado por fuso | Lógica |
| B2 | Overdue NaN para ETAs não-ISO | Lógica |
| B3 | Projetos Frozen a 0% | Lógica |
| B4 | Filtro de card mata busca/chips | Lógica |
| B5 | ETA por quarter perdido ao editar | Lógica |
| B6 | editId desatualizado sobrescreve | Lógica |
| C1 | Ellipsis da descrição no meio da palavra | UI |
| C2 | Atalho "N" dispara ao digitar | UI |
| C3 | Lembrete abre Calendar sozinho | UX |
| C4 | Heatmap conta em dobro | Lógica |
| C5 | "Reports To" com nível errado | UI |
| C6 | Faixa de L3 vazia no organograma | UI |
| C7 | Release notes cortam no meio da palavra | UI |
| C8 | Paleta não busca por descrição | UX |
| C9 | Star abre modal no mobile | UI |

---

# Missão B — Multi-usuário: Autenticação, Identidade e Visões por Papel

**Em uma linha:** você entra com sua conta Google da Shopee — o Team Directory decide quem
você é e o que você vê. Sem senhas, sem papéis configurados à mão.

## 1. Login (autenticação)

O PM Hub abre numa tela **"Continue with Google"**, restrita ao domínio **@shopee.com** —
domínios externos são rejeitados. A autenticação só estabelece *quem você é*; nunca define
permissões.

- **Hospedado com credenciais:** com o `AUTH_CONFIG.clientId` preenchido, usa o Google
  Identity Services real (ver Missão C).
- **Zero-config (padrão):** sem credenciais, roda um **login simulado** fiel (digite o e-mail
  @shopee.com), então funciona no GitHub Pages sem setup.

## 2. Identidade → papel (o Team Directory é a fonte única de verdade)

Depois do login, o e-mail é casado com o **Team Directory**, e só esse registro define o
nível e a linha de reporte. **Nada é configurado manualmente.**

| Nível | Quem | Derivado de |
|-------|------|-------------|
| **L1 — GPM / Diretor** | Matheus Mendes | a pessoa sem gestor |
| **L2 — Líder de time** | os 4 squad leads — Leandra Mieko (Seller FBS), Lucas Menegaldo (Taxes FBS), Matheus Torresi (New Services), Danilo Leite (Monetization) | os mentores de squad |
| **L3 — PM (IC)** | os demais PMs | reportam ao lead do seu squad |
| **Convidado** | um @shopee.com **fora** do diretório | read-only |

> Exemplo: `tatiane.porto@shopee.com` → **Tatiane Porto · L3** · reporta a **Matheus Torresi**
> · líder **Matheus Mendes**. Papel atribuído automaticamente.

Se o diretório muda (alguém entra, troca de squad, vira lead), níveis e linhas de reporte
mudam junto — não há tabela de papéis separada para manter.

## 3. Persistência (chaves originais do desafio)

Os dados continuam nas chaves originais exigidas pelo desafio, e na primeira carga qualquer
dado antigo em chaves `pmh_*` é **migrado automaticamente**:

| Chave | Conteúdo |
|-------|----------|
| `sph_v3` | projetos |
| `sph_team3` | time |
| `sph_tasks3` | tarefas |

A sessão de login fica guardada à parte no `localStorage`, então um reload mantém você logado.

## 4. O que cada nível vê — escopo padrão adaptativo

O design **adapta a experiência inicial** de cada pessoa em vez de esconder informação como
uma trava de permissão. Cada nível abre no seu escopo natural:

| Nível | Today (briefing/KPIs) | Tasks | Visões de liderança | Edição |
|-------|------------------------|-------|---------------------|--------|
| **L1** | Todos os times | De todos os membros | ✅ Visíveis | Tudo |
| **L2** | Seu time (squad) | Do seu time | ✅ Visíveis | Projetos do seu time |
| **L3** | **Seu time (squad)** | Só as suas | ❌ Ocultas | Só os próprios |
| **Convidado** | Sem filtro (tudo) | Todas (sem filtro) | ❌ Ocultas | Nenhuma (read-only) |

**Destaque do L3 (IC):** no **Today**, um L3 enxerga o squad inteiro — exatamente como o seu
lead L2 — com briefing, KPIs, "o que mudou", "precisa de atenção", "lançando em breve" e o
card de **balanceamento de carga do time**. Ou seja, o IC ganha contexto do time na tela
inicial, em vez de ver só os próprios projetos. O que **continua pessoal por intenção**: as
**Tasks** (as próprias tarefas) e a **edição** (vê o time, mas só edita os projetos dos quais
é dono).

## 5. Tasks por nível

- **L1** → vê as tasks de todos os membros.
- **L2** → vê as tasks do seu time — tanto as que têm um liderado como owner quanto as ligadas
  a projetos do squad.
- **L3** → vê só as suas tasks.

## 6. Visões de liderança (só L1 + L2)

Aparecem para **L1 e L2** e ficam ocultas para **L3 e convidados**:

- Comparação entre times e comparação de health score (Insights)
- Aba de **Workload** no módulo Team (distribuição de carga entre times)
- Portfolio health por líder e métricas de entrega da organização

## 7. Convidado (@shopee.com fora do time)

Um e-mail @shopee.com válido fora do diretório entra como **convidado read-only**:

- **Visão sem filtro por time** — vê todos os projetos e tasks, sem recorte de squad.
- **Sem visões de liderança** — Workload do Team e Team Comparison não aparecem.
- **Totalmente read-only** — não edita nenhum projeto.

## 8. Preview a level (demo e revisão)

A partir do **chip de identidade** (canto superior direito), qualquer usuário logado pode
**pré-visualizar a experiência como L1, L2 ou L3**. É marcado como preview e não muda as
permissões reais. A tela de login também oferece logins de demo de um clique (um L1, um L2 e
um L3 reais) e um convidado read-only, para avaliar o modelo de ponta a ponta sem configurar
contas.

---

# Missão C — Publicação

O PM Hub é **um único arquivo HTML estático**, então publica em qualquer hospedagem estática.
Há **duas formas de publicar**, e ambas começam pelo GitHub Pages:

- **Opção A — Determinístico (zero setup):** publica como está. Login simulado, file picker
  demo e insights gerados localmente dos dados reais. Sem chaves, sem backend, sem custo.
- **Opção B — Integração completa:** liga o **Google Sign-In + Drive Picker** (login real e
  anexo de arquivos do Drive) e os **insights via Claude** (proxy no Cloudflare). Cada parte
  tem **fallback automático**: se a credencial não estiver preenchida, cai no modo demo — o
  app nunca quebra.

Os três pontos de configuração ficam no topo do `index.html`:

```html
<script>
  // Google Sign-In (login real)
  window.AUTH_CONFIG  = { clientId: '' };
  // Google Drive Picker (anexar arquivos do Drive)
  window.DRIVE_CONFIG = { clientId: '', apiKey: '', appId: '' };
  // Insights via Claude (proxy serverless)
  window.AI_CONFIG    = { endpoint: '' };
</script>
```

Com os três vazios → modo demo (Opção A). Preenchidos → integração real (Opção B).

## Parte 1 — Publicar no GitHub Pages (necessário nas duas opções)

1. Crie um repositório no GitHub (ex.: `pm-hub`).
2. Renomeie o arquivo único para **`index.html`** e suba-o na raiz, junto deste README.
3. No repositório: **Settings → Pages**.
4. Em **Source**, escolha **Deploy from a branch**, branch **main** e pasta **/(root)**.
   Salve.
5. Aguarde ~1 min. A URL fica **`https://SEU_USUARIO.github.io/pm-hub/`**. Anote-a — ela é a
   sua **origem autorizada** para a Opção B.

> Pronto: na **Opção A**, a publicação termina aqui.

## Parte 2 — (Opção B) Google Cloud: Sign-In + Drive Picker

O Google Sign-In e o Picker rodam 100% no navegador via OAuth, mas **só funcionam numa origem
HTTPS autorizada** (a URL do GitHub Pages) — nunca abrindo o arquivo via `file://`.

1. Acesse **console.cloud.google.com** e crie um projeto (ex.: `PM Hub`).
2. **APIs & Services → Library** → ative **Google Picker API** e **Google Drive API**.
3. **APIs & Services → OAuth consent screen** → tipo **Internal** (recomendado, restringe ao
   workspace @shopee.com) → preencha nome do app e e-mail de suporte → adicione o escopo de
   Drive (`.../auth/drive.file`) → salve.
4. **APIs & Services → Credentials → Create Credentials → OAuth client ID**:
   - Application type: **Web application**.
   - Em **Authorized JavaScript origins**, adicione a URL do Pages **sem barra final**
     (ex.: `https://SEU_USUARIO.github.io`).
   - Deixe **Authorized redirect URIs** em branco (o Picker não usa).
   - Crie e **copie o Client ID** (`...apps.googleusercontent.com`).
5. **Create Credentials → API key** → copie a chave. Em seguida, **restrinja** a chave:
   *API restrictions* → Google Picker API + Google Drive API; *Application restrictions* →
   HTTP referrers → adicione a URL do Pages.
6. Pegue o **App ID** = o **Project number** numérico em **IAM & Admin → Settings**.
7. No `index.html`, preencha:

```js
window.AUTH_CONFIG  = { clientId: 'SEU_CLIENT_ID.apps.googleusercontent.com' };
window.DRIVE_CONFIG = {
  clientId: 'SEU_CLIENT_ID.apps.googleusercontent.com',  // o mesmo do AUTH_CONFIG
  apiKey:   'SUA_API_KEY',
  appId:    'SEU_PROJECT_NUMBER'
};
```

8. Suba o `index.html` atualizado para o GitHub. Abrindo pela URL do Pages, o **Continue with
   Google** passa a autenticar de verdade (e a validar @shopee.com), e o **Browse Drive** abre
   o Picker real lendo o Drive da pessoa. Sem as credenciais, ambos caem no modo demo.

## Parte 3 — (Opção B) Cloudflare Worker: insights via Claude

Por que um proxy: o GitHub Pages é **estático e não esconde segredos**. A API key da Anthropic
**não pode** ir no HTML (qualquer um veria no código-fonte e gastaria seus créditos). Um
Worker do Cloudflare guarda a key como *secret* e o site só faz `fetch()` nesse endpoint.

```
GitHub Pages (estático) ──fetch──▶ Cloudflare Worker (guarda a key) ──▶ API Anthropic (Claude)
```

Pré-requisitos: **Node.js 18+** e uma conta no Cloudflare (free tier basta) e na Anthropic.

1. Em **console.anthropic.com → API Keys**, gere uma key e copie-a.
2. No terminal, crie o Worker (o C3 instala o Wrangler junto):
   ```bash
   npm create cloudflare@latest pm-hub-ai
   # escolha: "Hello World" → "Worker only" → JavaScript
   cd pm-hub-ai
   ```
3. Substitua o `src/index.js` pelo `worker.js` deste repositório (pasta `cloudflare-worker/`),
   ou pelo código do Apêndice abaixo. No `wrangler.toml`, ajuste as variáveis:
   ```toml
   name = "pm-hub-ai"
   main = "src/index.js"
   compatibility_date = "2025-01-01"

   [vars]
   ALLOWED_ORIGIN = "https://SEU_USUARIO.github.io"   # trava o CORS no seu site
   MODEL = "claude-haiku-4-5"                          # modelo rápido e barato
   ```
4. Faça login no Cloudflare:
   ```bash
   npx wrangler login
   ```
5. Guarde a API key como **secret** (nunca no arquivo):
   ```bash
   npx wrangler secret put ANTHROPIC_API_KEY
   # cole a key da Anthropic quando solicitado
   ```
6. Faça o deploy:
   ```bash
   npx wrangler deploy
   ```
   Copie a URL retornada (algo como `https://pm-hub-ai.SEU_SUBDOMINIO.workers.dev`).
7. No `index.html`, preencha o endpoint:
   ```js
   window.AI_CONFIG = { endpoint: 'https://pm-hub-ai.SEU_SUBDOMINIO.workers.dev' };
   ```
8. Suba o `index.html` ao GitHub. No Today, clique **"Refresh insight"** — o badge muda de
   "Product pulse · generated" para "· Claude", indicando que o texto agora vem do Claude ao
   vivo. Se a chamada falhar (sem key, rate limit, rede), cai no texto determinístico — nunca
   quebra.

## Custo e segurança

- **Cloudflare Workers:** free tier cobre ~100 mil requisições/dia — sobra para a demo.
- **Tokens da Anthropic:** com `claude-haiku-4-5` e resumos curtos, cada insight custa
  centavos. Você controla o gasto na Console.
- **Segurança:** a `ANTHROPIC_API_KEY` vive **só** no Worker (como secret). O `ALLOWED_ORIGIN`
  trava o CORS no seu domínio. A API key do Google é restrita por referrer e por API.
- **Nota honesta sobre "single file":** o arquivo entregue continua sendo um HTML único e
  autossuficiente no modo demo. Na Opção B, ele carrega os scripts oficiais do Google
  (Identity Services / Picker) e chama o seu Worker — dependências externas inerentes a
  qualquer integração real, sempre com fallback local.

## Apêndice — `worker.js` de referência

Proxy mínimo: recebe o resumo do portfólio, chama o Claude e devolve o texto. Trata CORS e
guarda a key como secret.

```js
export default {
  async fetch(request, env) {
    const ORIGIN = env.ALLOWED_ORIGIN || '*';
    const cors = {
      'Access-Control-Allow-Origin': ORIGIN,
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };
    if (request.method === 'OPTIONS') return new Response(null, { headers: cors });
    if (request.method !== 'POST')
      return new Response('Method Not Allowed', { status: 405, headers: cors });

    try {
      const { summary } = await request.json();
      const r = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'content-type': 'application/json',
          'x-api-key': env.ANTHROPIC_API_KEY,        // secret
          'anthropic-version': '2023-06-01',
        },
        body: JSON.stringify({
          model: env.MODEL || 'claude-haiku-4-5',
          max_tokens: 400,
          messages: [{
            role: 'user',
            content: 'Você é um chefe de produto. Resuma o pulse do portfólio em 3-4 frases, '
                   + 'factual e direto, a partir destes dados:\n\n' + summary,
          }],
        }),
      });
      const data = await r.json();
      const text = (data.content || [])
        .filter(b => b.type === 'text').map(b => b.text).join('\n');
      return new Response(JSON.stringify({ text }), {
        headers: { ...cors, 'content-type': 'application/json' },
      });
    } catch (e) {
      return new Response(JSON.stringify({ error: String(e) }), {
        status: 500, headers: { ...cors, 'content-type': 'application/json' },
      });
    }
  }
};
```

---

# Extras — Mudanças estruturais e novidades

Além das três missões, o `index.html` reorganiza módulos do PM Hub original e adiciona
recursos novos. Esta seção resume o que mudou em relação ao arquivo original.

## Reorganização de módulos

O original concentrava os agrupamentos e relatórios num módulo de Reports/Dashboard. No novo
arquivo eles foram movidos para onde fazem mais sentido:

| No PM Hub original | Onde está agora |
|--------------------|-----------------|
| Agrupar **por status**, **por product line**, **por prioridade** | Módulo **Insights** |
| Agrupar **por PM** e o **Workload Heatmap** | Módulo **Team** (view *Workload*, exclusiva de liderança) |
| Geração de **Reports** | A partir da view **"by priority"** no módulo **Insights** |

Resultado: o Insights vira o lugar de análise do portfólio (saúde, risco, distribuição de
prioridades, relatórios), e o Team concentra tudo que é sobre pessoas (organograma, carga,
heatmap por PM).

## Novidades em relação ao original

- **Notificações** — uma central de "o que precisa da minha atenção agora", derivada dos
  mesmos sinais padronizados (overdue, quiet, needs attention). Não é um módulo separado e sim
  uma superfície de apoio acessível a qualquer momento.
- **Preferências** — configurações mínimas do workspace, também como superfície de apoio.
- **Módulo Team mais completo** — cada pessoa do diretório agora traz **responsabilidades**
  (áreas de atuação), **aniversário** (com link para criar o evento no calendário) e a **data
  de entrada na Shopee** (*joined*), além do nível, squad e linha de reporte.
- **Visão integrada com Claude (IA)** — superfícies como o **Product pulse** (Today) e o
  **Weekly executive summary** são escritas pelo Claude a partir de um resumo factual do
  portfólio, com **fallback determinístico** quando a IA não está configurada (ver Missão C,
  Opção B). O badge indica a origem do texto: "· generated" (local) ou "· Claude" (ao vivo).

---

# Glossário — Definições de métricas (padronizadas em todos os módulos)

Todos os módulos — **Today, Changelog, Projects, Insights** — derivam seus sinais dos
**mesmos helpers compartilhados** (`helpers.js`), então um número significa a mesma coisa em
todo lugar. As janelas de tempo são relativas ao **dia real em que a ferramenta é aberta**
(`daysQuiet` = dias desde o último *movimento* do projeto).

| Termo | Definição | Helper |
|-------|-----------|--------|
| **Movimento** (*movement*) | Uma **mudança de status** OU um **weekly update** registrado. Edições simples do corpo (objetivo, PICs, ETAs, arquivos) **não** contam. | (marca o `lastActivity`) |
| **Moveu esta semana** (*moved this week*) | Teve movimento nos últimos **7 dias** corridos (`daysQuiet ≤ 7`). | `movedThisWeek(p)` |
| **Parado / Gone quiet** (*quiet*) | Em fluxo ativo **e** sem movimento por **mais de 10 dias** (`daysQuiet > 10`). | `isStale(p)` |
| **Atrasado** (*overdue / past live date*) | Passou da Live ETA, ainda não está Live e não está frozen. | `p.overdue` |
| **Precisa de atenção** (*needs attention* — guarda-chuva) | **Atrasado** OU **parado** (quiet). Frozen/paused é um **estado deliberado, não** um risco. É o **sinal único de risco**. | `needsAttention(p)` |
| **Lançando em breve** (*launching soon* — destaque de ETA) | Atrasado OU Live ETA nos próximos **15 dias**. | `etaImminent(p)` |

> **Importante (decisão de produto):** **Frozen**, **Frozen by Local** e **Frozen by
> Regional** são estados deliberados e **não** entram em "precisa de atenção". Uma menção a
> "frozen" no texto de um update também **não** marca o projeto como risco automaticamente.

## Como o Changelog usa esses termos

O Changelog agrupa o último movimento de cada projeto em quatro baldes editoriais:

- **Launches** — o status chegou a Live / A/B / Live Testing, ou o texto do update anuncia um
  lançamento.
- **Progress** — avançando (padrão para movimento ativo).
- **Decisions** — o texto do update registra uma aprovação, sign-off, mudança de escopo ou
  kickoff.
- **Risks & delays** — o projeto **precisa de atenção** (`needsAttention(p)` = atrasado ou
  parado) **ou** o PM sinalizou um bloqueio no próprio texto do update (palavras como
  *blocked, waiting, on hold, slip*). Projetos frozen/paused **não** são auto-marcados como
  risco.

O digest do Changelog reporta `N precisam de atenção` e `N parados (>10 dias)` usando
exatamente esses helpers — os mesmos limiares do briefing do Today e dos quick-filters do
Projects ("Gone quiet", "Past live date").

## Como o Insights usa esses termos

O Insights é *read-first*: cada seção abre com uma frase em linguagem simples e depois a
evidência.

- **Portfolio health score** = média de duas metades: **frescor** (iniciativas ativas que não
  ficaram paradas, `daysQuiet ≤ 10`) e **entrega no prazo** (iniciativas ativas não atrasadas).
  Os fatores "Fresh X/Y" e "On time X/Y" são clicáveis e abrem o Projects filtrado.
- **O trabalho está se movendo?** — "Esta semana" = `movedThisWeek` (`daysQuiet ≤ 7`); "Semana
  anterior" = moveu 8–14 dias atrás. Ambos clicáveis para o Projects filtrado.
- **O que está em risco?** — duas colunas: **Atrasados** (overdue) e **Parados · 10d+**
  (`isStale`). Atrasados são ordenados por prioridade (P0 primeiro) e depois pelo mais antigo;
  parados, pelo mais tempo sem movimento. Os **top 5** de cada um são destacados.
- **Como as prioridades estão distribuídas?** — iniciativas ativas em P0/P1/P2/P3/A definir;
  cada barra abre o Projects filtrado por aquela prioridade.
- **Quais times estão mais saudáveis?** (Missão D) — exclusivo de L1/L2: health por time, % no
  prazo, carga por PM e contagem de parados.

> Há uma **única fonte de verdade** (`helpers.js`) para *moved / quiet / overdue / needs
> attention*, usada de forma idêntica em todos os módulos — então os números do Today, do
> Changelog, do Projects e do Insights sempre batem entre si.
