# Equipe Concretta — memória do projeto

Site estático (`index.html`, sem build) de gestão de equipe + conferência de ponto da Concretta.
Hospedado no GitHub Pages, com Firebase Firestore como backend.

**Link ao vivo**: https://rubinho106-hash.github.io/equipe-concretta/
**Repo**: `github.com/rubinho106-hash/equipe-concretta` (branch `main`, Pages servindo da raiz)
**Pasta local**: `D:\SITE CONCRETTA` — clone real do repo, é aqui que se edita.

## Como isso se encaixa nos outros sistemas da Concretta

Este é o **terceiro e atual** sistema de "quem tá na equipe + quantos dias trabalhou", substituindo o fluxo
antigo baseado em Excel (`Controle_Ponto_Concretta.xlsx` + Apps Script `Código.gs`/`Página.html`).

- O sistema antigo (Apps Script) **não deve ser editado/redeployado sem pedido explícito** do Rubens —
  ele pediu pra parar depois de um episódio de mudança não solicitada. Não mexer lá.
- Uma versão anterior deste site existiu como Claude Artifact antes de migrar pra cá; pode estar
  abandonada, ninguém pediu pra apagar.

## Stack e dados

- Frontend: HTML/CSS/JS puro em `index.html`, Firebase compat SDK carregado via `<script>` do gstatic.com.
- Firestore (projeto `concretta-equipe`, região `southamerica-east1`), coleções:
  - `funcionarios/{id}` — cadastro (nome, função, pix, status, obraPadrao)
  - `funcionarios/{id}/pontos/{AAAA-MM}` — um doc por mês, campo `dias.{dia}.{m|t}` = nome da obra
    (ou `SÁBADO`/`DOMINGO`/`FERIADO` como marcador), mais `statusConferencia`
  - `obras/{id}` — lista viva de obras (nome), editável na aba "Obras"
- Regras do Firestore: abertas (`allow read, write: if true`) — sem Auth, qualquer um com o link edita.
- **Local e produção compartilham o mesmo banco** — não existe ambiente de teste separado. Todo write
  feito testando localmente cai no banco real. Reverter depois, ou evitar escrever à toa.

## Fluxo de edição local

1. Editar `index.html` direto.
2. Testar local: criar `.claude/launch.json` temporário (fora desta pasta, no projeto Claude Code —
   `python -m http.server <porta> --directory "D:/SITE CONCRETTA"`), testar via browser, **depois apagar
   o `.claude/launch.json`**.
3. `cd "/d/SITE CONCRETTA" && git add -A && git commit -m "..." && git push origin main`
4. GitHub Pages redeploy automático (~10-30s). Pra conferir ao vivo, navegar com `?bust=<algo>` novo a
   cada tentativa — o CDN cacheia por path e pode servir versão antiga mesmo com query string diferente
   às vezes; se `typeof algumaFuncaoNova === "function"` continuar `false`, tentar de novo com outro valor.

## Pegadinhas já resolvidas

- **`ref.update()` do Firestore não faz merge profundo de objeto aninhado** — sempre usar sintaxe de
  caminho com ponto: `ref.update({["dias." + dia + ".m"]: valor})`, nunca `ref.update({dias: {...}})`.
- **`abrirFicha()` chamado direto do console falha silenciosamente** se `mesConferencia` ainda for `null`
  (só é setado ao clicar na aba "Conferência de Ponto" pela UI) — é função `async`, então o erro vira
  promise rejeitada, não uma exceção visível. Sempre clicar na aba antes de testar isso via JS.
- Tirar screenshot (`computer` tool) logo depois de simular drag/resize via `MouseEvent` sintético às
  vezes fecha modais sem motivo aparente — é um artefato do harness de teste, não bug do site. Rodar a
  sequência inteira (abrir + interagir + ler estado) numa única chamada de JS evita o problema.
- **Nunca usar `confirm()`/`alert()` nativos do navegador neste site.** Navegadores embutidos (WhatsApp,
  Instagram etc.) e alguns navegadores mobile bloqueiam esses diálogos silenciosamente — sem erro
  visível, o clique no botão simplesmente "não faz nada". Foi assim que o botão "Excluir funcionário"
  parou de funcionar (02/09/2026). Use a função `confirmarAcao(mensagem)` já existente no código (um
  modal HTML próprio que retorna uma Promise<boolean>) no lugar de `confirm()`.
- **Nome de obra é só texto, não tem ID fixo/FK** — está gravado como string solta em `dias.{dia}.m/t` e
  também é o `nome` de um doc separado em `obras/{id}` (só pra alimentar o dropdown). Isso significa que
  renomear ou apagar um doc de `obras` nunca quebra lançamentos já feitos (eles continuam mostrando o
  nome antigo como texto órfão), mas também significa que é fácil criar **obra duplicada** sem perceber
  — já aconteceu duas vezes (Rubens renomeando "UBS" pra "COLÉGIO MILITAR" ao vivo, coincidindo com uma
  obra nova sendo criada com o mesmo nome). Se o Rubens disser algo como "tem obra repetida na lista",
  ler a coleção `obras` inteira, achar nomes duplicados e apagar um dos docs (não afeta lançamentos).
- **Nunca deixar um "seed se vazio" rodar sozinho no carregamento da página** depois do bootstrap
  inicial. `seedSeVazio()`/`seedObrasSeVazio()` existem no código mas **não são mais chamadas
  automaticamente** (só rodar na mão pelo console se um banco genuinamente vazio precisar ser semeado de
  novo). O guard delas (`colRef.limit(1).get(); if(!snap.empty) return;`) não é confiável contra uma
  leitura vazia espúria (rede/cache instável no load) — foi exatamente isso que aconteceu em 02/09/2026 e
  recriou as 32 pessoas do `SEED` por cima do cadastro real, duplicando 30 funcionários e ressuscitando 2
  que tinham sido excluídos de propósito. 66 cadastrados em vez de 34, corrigido no mesmo dia.
- **Status Ativo/Inativo (resolvido 02/09/2026)**: os 3 cards do topo da Lista de Funcionários (Total/
  Ativos/Inativos) são filtro clicável (`filtroStatus`). Trocar o status de alguém (clicar na pill) pede
  confirmação (`confirmarAcao`, vermelho pra Inativo/azul pra reativar). **Inativo não aparece mais na
  Conferência de Ponto** (nem na lista nem nos stats de total/conferidos/pendentes) — `renderConfLista()`
  filtra por `status !== "Inativo"` antes de tudo.

## Obras ativas (02/09/2026)

CRAS, CRECHE, COLÉGIO MILITAR, PARQUE IMPERIAL, PRAÇA G. **"UBS" não existe mais** — foi renomeada pra
COLÉGIO MILITAR (duas vezes: uma revertida em 15/08, a definitiva em 02/09). Se aparecer "UBS" em
qualquer mensagem futura, é COLÉGIO MILITAR.

## Apelidos (grupo do WhatsApp escreve errado/informal)

- "Praça" / "Praça Imperial" → **PARQUE IMPERIAL**
- "Praça canto da serra" → **PRAÇA G** ("canto da serra" é bairro, não nome de obra)
- "Cras canto da serra" → **CRAS** (mesma lógica)
- "Creche canto da serra" → **CRECHE**
- "UBS" → **COLÉGIO MILITAR** (ver acima)
- "Cleber Lucas" / "Kleber Lucas" → Kleber Lucas Sousa Soares
- "Marlon Molra" → Marlon Moura Silva
- "Dairo Silva" → Dário Silva Dos Santos
- "Wesley Numes" → Wesley Nunes Conceição
- "Leonardo Oliveira" → Leandro Oliveira Silva, Carpinteiro (**não confundir** com "Leonardo paixão" =
  Leonardo Paixão Santos, pessoa diferente, aparecem juntos às vezes)
- "Silvestre de Oliveira" → Silvestre Pereira da Silva
- "Edson Silva" → Edison Silva de Oliveira

## "Banco de Dados" (antiga aba "Fechamento de Ponto") — estado atual (03/09/2026)

Rubens pediu pra esvaziar essa 4ª aba ("trocar texto Fechamento de Ponto... e limpar todo
código pra darmos outra funcionalidade") e renomeá-la — via `AskUserQuestion`, confirmou
inicialmente: renomear o botão da aba pra **"Banco de Dados"**, apagar o conteúdo de
aprovação de pagamento por obra, mas **manter** a trava de mês fechado (seletor de mês +
botão Fechar/Reabrir mês + banner). Minutos depois ele pediu pra apagar isso também
("APAGAR O QUE FOI Mantido... DEIXAR O CODIGO DO BOTAO LIMPO") — então a trava saiu
inteira na sequência, no mesmo dia.

**Estado final: a aba fica 100% vazia** (`<div id="view-fechamento" hidden></div>`, só um
comentário HTML explicando o histórico). Tudo o que existia nessa aba — filtro de obras
clicável, os 3 stat cards Dias/Aprovado/Falta aprovar, busca, tabela funcionário+obra com
pill de pagamento, exportar CSV, seletor de mês, botão Fechar/Reabrir mês, banner de
aviso — foi removido do HTML/CSS/JS. Funções que não existem mais no código:
`agruparFechamentoPorObra()`, `calcularDiasPorObra()`, `statusPagamento()`,
`alternarPagamento()`, `renderFechamentoObrasFiltro()`, `renderFechamento()`,
`renderFechamentoMesSelect()`. A variável `mesesFechados` e o `onSnapshot` da coleção
`fechamentos` também foram removidos.

**A Conferência de Ponto não tem mais nenhum conceito de "mês fechado"** — as checagens
`if(mesesFechados[mesConferencia])` que bloqueavam `alternarConferencia()`/`salvarDia()`,
e o `disabled` condicional nos selects de obra/status do cartão (`abrirFicha()`/
`renderFichaConteudo()`), foram todos removidos. Todo mês agora é sempre editável, sem
exceção.

**Os dados antigos no Firestore continuam intactos** — coleção `fechamentos/{AAAA-MM}`
(campo `fechado`) e o campo `pagamentos` nos docs de `pontos/{mes}` — só que nada no app
lê ou escreve mais neles. Se um dia precisar reaproveitar esse conceito (trava de mês, ou
aprovação de pagamento), o dado histórico ainda está lá, intacto.

IDs internos (`seg-fechamento`, `view-fechamento`) continuam com o nome antigo no código —
só o texto visível do botão da aba mudou. Não renomear esses IDs sem
necessidade real, pra não gerar um diff gigante à toa.

**Primeiro passo (SUBSTITUÍDO no mesmo dia) — cards de mês em mini-calendário**: antes da
tela "Apontamentos" (seção abaixo), o primeiro passo tinha sido só uma grade de cards de
mês em estilo mini-calendário, clicáveis pra selecionar o mês, sem nenhum conteúdo dentro
ainda (`renderBancoMesesGrid()`). Rubens mandou uma imagem de referência completa (tela
"Apontamentos" de outro sistema) horas depois pedindo pra apagar os cards e substituir por
aquilo — `renderBancoMesesGrid()`, `.mini-cal-*`, e o compartilhamento de `mesConferencia`
com a Conferência de Ponto foram todos removidos/revertidos nessa substituição. Só ficou
como nota histórica; ver a seção "Apontamentos" abaixo pro estado atual.

## "Apontamentos" — tela definitiva da aba Banco de Dados (commit `3b8ff77`, 03/09/2026)

Rubens mandou uma imagem de referência (mockup completo de outro sistema, com título
"Apontamentos", filtros Funcionário/Obra/Período + botão Consultar, card resumo do
funcionário, grade tipo calendário, tabela de lançamentos e resumo por obra) pedindo pra
"apagar cards banco de dados e substituir por isso". Como o mockup tinha elementos que não
existem no sistema (login com avatar de usuário, "Faltas"/"Transferido" como estados,
"Lançado por"/horário de auditoria, campo de observação por dia), usei `AskUserQuestion`
duas vezes antes de implementar, pra não inventar dado nenhum:

- **Avatar/usuário logado** ("Rubens Gomes / Administrador") → **fora**, o site não tem
  login.
- **"Faltas"** → **adicionado** como novo marcador `FALTA` (junto de SÁBADO/DOMINGO/
  FERIADO em `MARCADORES`, tanto em `index.html` quanto em `cartoes.html`) — aparece
  como opção normal no dropdown de obra de qualquer dia, em qualquer lugar que já usa
  `opcoesObra()`/`ehMarcador()` (cartão do admin incluído).
- **"Transferido"** → **não implementado**, não ficou claro o que seria diferente de só
  ter obras diferentes de manhã e de tarde no mesmo dia (que já é suportado).
- **"Lançado por" / "Horário" / campo de Observação por dia** → **fora**, exigiriam login
  individual (que não existe) e um novo campo no schema — a tabela "Lançamentos
  detalhados" ficou só com Data/Obra/Período/Valor, dados 100% reais.

**O que a tela faz**: filtros Funcionário (só Ativos, sem "Todos" — precisa escolher um
pra consultar), Obra ("Todas" ou uma específica) e Período (mês, reaproveitando
`listaMeses`/`infoMes()`). Botão "Consultar" busca direto no Firestore
(`funcionarios/{id}/pontos/{mes}`, sob demanda, sem cache) e chama
`renderApontResultado(f, mes, info, obraFiltro)`, que:

1. Classifica cada dia do mês em `d` (dia inteiro, 1.0) / `m` (manhã, 0.5) / `t` (tarde,
   0.5) / `f` (falta, 0.0) / `vazio` (sem lançamento) — aplicando o filtro de obra por
   período: um período só "conta" se a obra bater com o filtro (marcadores como FALTA
   continuam valendo mesmo com filtro de obra ativo, já que não são obra). Quando manhã e
   tarde têm obras reais **diferentes** no mesmo dia, mostra as duas juntas separadas por
   "/" em vez de inventar um estado "transferido".
2. Renderiza: card resumo do funcionário (avatar/função/situação + mini-stats Dias
   inteiros/Meio período/Faltas/Total no mês), grade tipo calendário (linhas Dia/Semana/
   Obra/Período, células de período coloridas por tipo — `.cel-periodo.tipo-d/m/t/f`),
   legenda, tabela "Lançamentos detalhados" (só dias com lançamento real), "Resumo por
   obra" (Períodos/Total por obra) e uma caixa de observações com dicas reais do sistema
   (não as fictícias do mockup).

**Independente do `mesConferencia`/`dadosMes` da Conferência de Ponto** — os filtros dessa
tela são próprios, consultados sob demanda no botão "Consultar", sem sincronizar com a
outra aba (diferente do antigo Fechamento e da tentativa anterior com mini-calendário, que
compartilhavam `mesConferencia`). `calcularListaMeses()` foi extraída de
`carregarMesTabs()` só pra popular o `<select>` de Período sem disparar a carga pesada de
dados de todos os funcionários que só a Conferência de Ponto precisa.

**Bug pego no teste, corrigido antes de publicar**: `mostrarSegmento()` decidia se a
Conferência de Ponto precisava inicializar checando `!listaMeses.length` — funcionava
antes porque só a própria Conferência populava `listaMeses`. Com a nova tela populando
`listaMeses` por conta própria (via `calcularListaMeses()`), visitar "Banco de Dados"
antes de "Conferência de Ponto" deixava essa travada em "Carregando período…" pra sempre
(mesConferencia nunca era setado). Corrigido trocando a checagem pra `!mesConferencia`, o
estado que realmente importa. Reproduzido e reverificado localmente e ao vivo (visitando
as abas nessa ordem) antes de considerar resolvido.

Testado com dados reais: Alex Pereira Silva (CRECHE dia 1) e Weslen Da Silva Alves
bateram certo nos números; lançamento de teste `FALTA` no dia 10 do Weslen apareceu
correto na grade/tabela E no cartão do admin (revertido depois); filtro de obra que não
bate zera tudo corretamente; mês sem documento no Firestore (Janeiro/2027) não quebra.

## FALTA vira sempre dia inteiro — Modelo A (commit `eb3be37`, 04/09/2026)

Depois de a tela Apontamentos ir pro ar, o Rubens mandou uma análise externa (de outra
ferramenta) revisando o site inteiro. Um dos pontos: como `FALTA` era só mais um item de
`MARCADORES`, dava pra selecionar `m:"FALTA", t:"CRAS"` no cartão — meio período de falta
e meio período trabalhado no mesmo dia. Tecnicamente funcionava (a tela Apontamentos já
lida bem com isso, contando 0,5 diária), mas a falta parcial desaparecia da leitura: virava
só "Tarde — 0,5", sem indicar que a manhã foi falta.

Decisão, depois de eu sugerir duas alternativas (Modelo A = falta sempre dia inteiro,
Modelo B = falta por período) e o Rubens escolher **Modelo A**: `FALTA` nunca fica só num
período. Implementado em `salvarDia()`:

1. **Marcar Falta em qualquer período (manhã ou tarde) grava os dois juntos** —
   `dias.{dia}.m = dias.{dia}.t = "FALTA"`, não importa qual dropdown foi mexido.
2. **Escolher uma obra real num período enquanto o outro ainda é `FALTA` limpa os dois
   primeiro** (o dia deixa de ser falta integral) e só depois grava a obra no período
   mexido, deixando o outro vazio — nunca sobra uma "FALTA órfã" sozinha.
3. **Proteção nova**: marcar Falta quando o outro período já tem uma obra real lançada
   pede confirmação antes (`confirmarAcao` estilo "danger"), já que isso apagaria esse
   lançamento — "O outro período desse dia já tem "X" lançado. Marcar Falta apaga esse
   lançamento e deixa o dia inteiro como falta. Confirma?". Cancelar reverte a tela
   (`renderFichaConteudo()`) sem gravar nada.

Só o cartão do admin (`index.html`) precisou mudar — a tela Apontamentos e o `cartoes.html`
já liam `FALTA` corretamente independente de vir num período só ou nos dois (não precisou
tocar neles). Testado localmente e ao vivo: falta em período vazio (sem confirmação), obra
real limpando falta do outro período, falta sobrescrevendo obra real (confirmação testada
cancelando — preserva — e confirmando — sobrescreve) — todos os cenários revertidos depois
do teste.

**Passo 2 do roadmap — coluna "Períodos" virou "Dias" (commit `e92c902`, 04/09/2026)**: no
"Resumo por obra" da tela Apontamentos, `.apont-obra-row` passou de 3 colunas
(Obra/Períodos/Total) pra 2 (Obra/Dias) — removido `Math.round(totalObra / 0.5)`, que só
multiplicava o Total por 2 sem trazer informação nova. Mesma razão de sempre: sistema é de
gestão de dias, não de valor nem de unidade intermediária (o mesmo motivo que já tinha
derrubado o R$ antes). Testado ao vivo: Weslen/CRECHE mostra só "CRECHE 1,0" + "Total 1,0".

## `apontador.html` — Passo 3 do roadmap (commit `0781f92`, 04/09/2026)

Primeiro arquivo real do Apontador dentro do repo (`D:\SITE CONCRETTA\apontador.html`) —
antes só existia o protótipo isolado em
`C:\Users\rubin\Downloads\concretta_apontador_protecao_data_futura.html` (localStorage
puro, funcionários hardcoded). Esse passo especificamente troca os dados fixos por dados
reais do Firestore, mantendo tudo o mais do protótipo igual (tema, modelo de status
full/morning/afternoon/absent, regra `complementar()` sem "Transferido", proteção de data
futura):

- **Firebase**: mesmo `firebaseConfig` de `index.html`/`cartoes.html` (mesmo projeto
  `concretta-equipe`).
- **`OBRAS`** vem de `db.collection("obras")` (populando o `<select>` de Obra) e **`todos`**
  vem de `db.collection("funcionarios")` filtrado a `status !== "Inativo"` — mesmo padrão
  de leitura já usado em `cartoes.html` (`.get()` único, não `onSnapshot` — a página é pra
  uso rápido de uma sessão, não fica aberta acompanhando mudanças ao vivo).
- **Funcionário passa a ser identificado por `funcId`** (o id real do doc no Firestore),
  não mais por nome — resolve um risco que eu já tinha levantado na análise do protótipo
  (nomes iguais/parecidos colidindo). Os registros no `localStorage` agora são
  `{funcId, nome, obra, data, status}`.
- **A lista de funcionários mostra TODOS os Ativos, em qualquer obra selecionada** — sem
  filtro de "pertence a essa obra" (isso já é o comportamento final decidido: "o apontador
  continua podendo tocar em qualquer funcionário"). Falta só a ordenação por "equipe
  provável" no topo — isso é o Passo 4, ainda não feito.

**O que esse passo NÃO faz ainda** (de propósito, é o Passo 5): os lançamentos continuam
gravando em `localStorage`, exatamente como o protótipo — ainda não escrevem em
`funcionarios/{id}/pontos/{AAAA-MM}` no Firestore. Por isso **esse arquivo ainda não tem
link a partir do `index.html`** — não está pronto pra uso real (um lançamento feito aqui
hoje não aparece no cartão do admin nem na tela Apontamentos), só acessível direto pela
URL pra teste/acompanhamento do desenvolvimento.

Testado localmente e ao vivo: 26 funcionários Ativos carregados certo (bate com o total
real), 5 obras reais no dropdown (COLÉGIO MILITAR/CRAS/CRECHE/PARQUE IMPERIAL/PRAÇA G),
lançamento de teste gravou com o `funcId` correto (conferido contra o id real do Alex
Pereira Silva), trocar de obra mantém a lista completa de 26 (comportamento esperado),
sem erro no console em nenhum dos dois ambientes.

## Passo 4 do roadmap — "Equipe provável" no Apontador (commit `a222989`, 04/09/2026)

Dois arquivos mudaram juntos:

**`index.html` — `salvarDia()` ganhou um efeito colateral novo**: toda vez que um período
recebe uma obra real (não marcador, não FALTA, não vazio), também grava
`ultimaObraId`/`ultimaObraNome`/`ultimaObraData` no doc do **funcionário** (não no doc de
pontos) — write "fire-and-forget" com `.catch(() => {})`, nunca trava o salvamento do dia
se falhar. Como obra nunca teve ID próprio nesse sistema (é só texto em todo canto —
`dias.dia.m/t`, o array `OBRAS`, etc.), `ultimaObraId` guarda por enquanto o mesmo valor de
`ultimaObraNome` (redundante de propósito, pra não inventar um conceito de "obra com ID"
que não existe em nenhum outro lugar do código). Esses 3 campos nunca são lidos como fonte
de verdade — só servem de sugestão de ordenação.

**`apontador.html` — grupo "Equipe provável"**: ao selecionar uma obra, `render()` separa
`todos` em dois grupos — quem tem `ultimaObraNome === obra.value` (ordenado por
`ultimaObraData` decrescente, mais recente primeiro) vai pro topo sob o cabeçalho "Equipe
provável"; o resto fica em "Outros funcionários ativos" logo abaixo. Se ninguém bater com a
obra selecionada, os cabeçalhos somem e a lista volta a ser simples (nunca fica um grupo
vazio visível). Ganhou também um campo de busca (`#buscaFunc`, filtra por nome, incide nos
dois grupos) — o resumo/estatísticas do rodapé continuam calculados sobre os 26 Ativos
inteiros, a busca é só pra achar alguém mais rápido na lista, não é filtro de relatório.

Testado local e ao vivo (nos dois ambientes): lançamento real no cartão do admin (Alex
Pereira Silva → CRAS local; Carlos André → PARQUE IMPERIAL em produção) gravou o cache
certo, o Apontador mostrou a pessoa em "Equipe provável" ao abrir a obra correspondente,
obra sem ninguém no cache mostrou lista simples sem cabeçalhos, busca filtrando certo.
Dados de teste revertidos nos dois ambientes (cache apagado com `FieldValue.delete()`, dia
de teste limpo) depois de cada verificação.

## Passo 5 do roadmap — Apontador grava de verdade no Firestore (commit `bd765e8`, 04/09/2026)

O `localStorage` (`concretta_apontamentos_v3`) saiu de vez do Apontador — lançamentos
agora leem/escrevem direto em `funcionarios/{funcId}/pontos/{AAAA-MM}`, o **mesmo** doc que
`index.html` e a tela Apontamentos usam. Um lançamento feito no Apontador aparece
imediatamente no cartão do admin, sem nenhuma sincronização extra.

**Mudança de modelo importante**: o `localStorage` guardava "um registro por
funcionário+obra+data" (várias obras podiam coexistir pro mesmo dia via registros
separados). O Firestore real guarda "um dia com `m`/`t`, obra é dado, não é chave" — então
a antiga lógica de detectar "registros complementares" (`complementar()`/`sitPeriodo()`)
**saiu**, porque deixou de ser necessária: com dado real, manhã numa obra + tarde noutra já
é o resultado natural de duas escritas em campos diferentes (`dias.dia.m` e `dias.dia.t`),
sem precisar de lógica especial pra permitir isso.

**`aplicarLancamento(func, status)`** unifica os 4 status num só fluxo, lendo o estado atual
do dia (`dadosMes`, carregado uma vez por mês em `carregarDadosMes()`, recarregado quando a
data muda de mês):
- `absent` (Falta) → Modelo A: grava `m` e `t` como `"FALTA"` juntos. Se já havia trabalho
  real em qualquer período, confirma antes (mesmo texto/lógica do `index.html`).
- `full`/`morning`/`afternoon` → grava só o(s) período(s) visado(s) com a obra selecionada.
  Se o período alvo já tem uma obra REAL **diferente**, confirma antes de substituir. Limpa
  uma Falta órfã do período oposto quando necessário (mesma regra do Passo 1).
- Ao gravar uma obra real com sucesso, atualiza o cache `ultimaObraId/Nome/Data` (mesmo
  mecanismo do Passo 4).

`garantirDocPontos()` foi copiada do `index.html` (mesmo formato, incluindo
`statusConferencia`, pra não quebrar a Conferência de Ponto se o Apontador for o primeiro a
criar o doc de um mês). O status exibido por funcionário continua **relativo à obra
selecionada no momento** (mesmo visual de antes — "Não lançado" se o trabalho real dele foi
em outra obra, mas a checagem de conflito em `aplicarLancamento()` enxerga isso e avisa
antes de sobrescrever). "Marcar todos" e "Limpar lançamentos" também migraram pra escrita
real (sequencial, não em paralelo, pra manter `dadosMes` consistente).

**O que continua em localStorage**: só "Fechar dia"/"Reabrir dia" (`concretta_fechados_v3`)
— isso é o Passo 7 (Fechar Dia por Data+Obra), ainda não migrado.

Testado localmente e ao vivo (nos dois arquivos, index.html incluído): lançamento real via
Apontador apareceu certo no cartão do admin (Fabio Pereira de Souza → CRAS, resumo por obra
bateu); conflito real (obra diferente já lançada) pediu confirmação, cancelar preservou,
confirmar sobrescreveu só o período certo; complementar (manhã numa obra + tarde noutra,
dia vazio) gravou sem pedir confirmação nenhuma; falta sobrescrevendo trabalho real seguiu
a mesma regra de confirmação; "Marcar todos" testado nos 26 funcionários reais (obra PRAÇA
G, dia seguro) e "Limpar lançamentos" também. Todos os dados de teste revertidos ao final
nos dois ambientes (dias limpos, cache `ultimaObra*` apagado, dados reais pré-existentes
conferidos intactos antes de encerrar).

**Nota de teste aprendida nesta sessão**: escrever direto no Firestore por fora da página
(pra reverter um teste) deixa o `dadosMes` em memória da aba aberta desatualizado — se for
continuar testando na MESMA aba depois de uma reversão manual, chamar
`carregarDadosMes(mesCarregado)` de novo antes, ou só recarregar a página.

## Passo 7 do roadmap — "Fechar Dia" por Data+Obra (commit `1d12d2a`, 04/09/2026)

"Fechar dia" deixou de ser `localStorage` (`concretta_fechados_v3`) e passou a gravar numa
coleção nova e compartilhada: **`fechamentosDia/{AAAA-MM-DD_OBRA}`** — por exemplo
`fechamentosDia/2026-09-03_CRAS`. Sem nenhuma relação com o antigo `fechamentos/{AAAA-MM}`
(removido do sistema em 03/09) — esse é por **mês**, o novo é por **dia + obra**, exatamente
como a análise recomendou: cada obra fecha o dia dela quando termina de apontar, sem
depender das outras obras nem do mês inteiro.

Implementação: `chaveFechamento()` monta a chave (`data.value + "_" + obra.value`);
`carregarFechado()` lê o doc e guarda o resultado numa variável em memória
(`fechadoAtual`), recarregada sempre que **obra** ou **data** mudam (e na carga inicial) —
`isFechado()` virou só a leitura síncrona dessa variável, sem mudar nenhum outro ponto do
código que já dependia dela (banner, desabilitar botões/linhas). "Fechar dia"/"Reabrir dia"
gravam `{obra, data, fechado, atualizadoEm}` com `.set({...}, {merge:true})` — preserva
histórico (`fechado:false` em vez de apagar o doc), mesmo padrão do antigo
`fechamentos/{mês}`.

Testado local e ao vivo: Marcar Todos + Fechar Dia nos 26 funcionários reais (obra PRAÇA G,
dia seguro) — banner "Dia fechado" e todos os botões/linhas desabilitados corretamente;
confirmado que **outra obra no mesmo dia continua aberta** (isolamento por obra+data
funcionando — testado trocando pra CRAS e verificando `fechadoAtual: false`); Reabrir Dia
testado, volta ao normal. Revertido tudo depois nos dois ambientes (26 lançamentos
limpos, cache `ultimaObra*` apagado, doc de teste em `fechamentosDia` deletado).

**Roadmap restante**: Passo 8 (auditoria de quem lançou/reabriu/corrigiu); Firebase
Authentication (site continua com Firestore aberto, `allow read, write: if true`) — esse
último ainda é o maior risco técnico pendente, não tem data pra entrar no roadmap.

## Campo "Obra padrão" no cadastro do funcionário (commit `d1b2f18`, 04/09/2026)

Origem: depois do Passo 5/7, o Rubens viu ao vivo que o Apontador mostra os 26 ativos
inteiros em qualquer obra selecionada (não existe "pertence à obra"), achou confuso, e a
gente discutiu 3 alternativas. Ele escolheu "Cadastro de equipe da obra" e perguntou como
apontar quem vai pra uma obra diferente da equipe dele — a resposta combinada foi: esse
cadastro é só um **filtro/sugestão de exibição**, nunca uma restrição de escrita. Esse
commit implementa o campo em si.

Chegamos a desenhar (e validar com dado real da planilha antiga) um índice de afinidade por
obra baseado em recência/frequência dos últimos 15 dias úteis, pra decidir quem aparece em
destaque — o Rubens decidiu **descartar essa ideia** por ser complexa demais pro ganho: o
`apontador.html` já tem busca por nome (`#buscaFunc`) que enxerga a lista inteira de Ativos
sem nenhuma restrição por obra/equipe (confirmado lendo `render()`: `filtrados` vem sempre de
`todos`, só filtrado pelo termo digitado) — isso já resolve "funcionário foi pra outra obra,
como aponto o dia dele": busca o nome e lança normalmente, mesma mecânica de sempre. Sem
fórmula, sem corte de score, sem grupo novo — o que já existe (Equipe provável / Outros
funcionários ativos, do Passo 4) e a busca já bastam.

- Campo novo `obraPadrao` no modal "Editar/Novo funcionário" (`index.html`), opcional
  (`— nenhuma —` por padrão), populado dinamicamente a partir do array vivo `OBRAS` (mesmo
  padrão de `opcoesObra()` já usado no cartão — se o valor gravado não estiver mais na
  lista viva de obras, aparece como opção extra em vez de sumir silenciosamente).
- Gravado junto com os outros campos do formulário (`nome`, `funcao`, `pix`, `status`) no
  mesmo `.set()/.update()` do submit — nada de lógica nova de salvamento.
- Exibido como linha extra ("Obra: X") sob a função, na Lista de Funcionários — só aparece
  se o campo estiver preenchido, sem alterar as outras telas (Apontamentos, Conferência de
  Ponto) que também listam funcionário.
- **Informativo, nunca restritivo**: usado pra filtrar exibição (Lista de Funcionários, ver
  seção abaixo, e o Apontador, ver "Equipe da obra" mais abaixo) — nenhuma tela usa pra
  restringir quem pode ser lançado/editado em nenhuma obra.
- Testado local e ao vivo: setei "CRECHE" em Alex Pereira Silva, confirmei que grava, que
  reabrir o modal mostra o valor certo, e que a lista mostra "Obra: CRECHE" — revertido pra
  vazio nos dois ambientes depois.

## Filtro por obra na Lista de Funcionários (commit `1ce3292`, 04/09/2026)

Depois de descartar o índice de afinidade (ver acima) e revalidar com o Rubens ("consegui ler
funcionários por obra?"), ele preencheu de verdade o `obraPadrao` de vários funcionários reais
em produção e a resposta final (via AskUserQuestion, "sem preferência" → segui a opção
recomendada) foi: cards de filtro clicáveis, mesmo padrão visual dos já existentes
Total/Ativos/Inativos.

- Nova fileira `#obra-filtro-row`, logo abaixo dos 3 stat-cards de status — um chip "Todas as
  obras" + um chip por obra que tenha pelo menos 1 funcionário com esse `obraPadrao` (cor por
  `corAvatar(nome)`, mesmo esquema dos chips de resumo do cartão) + um chip "Sem obra" se
  houver alguém sem o campo preenchido. Cada chip mostra a contagem.
- `renderObraFiltro()` reconstrói a fileira inteira a cada `render()` (que já roda em toda
  mudança de `todos` via `onSnapshot`) e também quando `OBRAS` muda (`carregarObras()`) —
  cobre tanto gente nova/editada quanto obra renomeada/criada.
- Clicar um chip seleciona `filtroObraSel`; clicar de novo no mesmo chip volta pra "Todas as
  obras" (toggle). `aplicarFiltro()` ganhou essa condição a mais, combinando com o filtro de
  status e a busca já existentes (os três são independentes, aplicam juntos).
- **Só filtra a exibição** — mesma natureza informativa do campo `obraPadrao` em si, não
  restringe nada em nenhuma outra tela.
- Testado local com dado real de produção (só leitura, nada revertido porque nada foi
  escrito): 28 cadastrados, chips mostraram COLÉGIO MILITAR 4 / CRAS 2 / CRECHE 12 / PARQUE
  IMPERIAL 4 / PRAÇA G 4 / Sem obra 2 — clicar CRECHE filtrou pra exatamente os 12, clicar de
  novo voltou pra todos, "Sem obra" mostrou os 2 sem o campo preenchido (Carlos Rikary e
  Dagilson Gomes da Conceição, ambos Inativos).

## `apontador.html` usa "Obra padrão" pra agrupar — "Equipe da obra" (commit `4ed3709`, 04/09/2026)

Rubens mandou print do Apontador aberto em COLÉGIO MILITAR reclamando "NAO ESTA FILTRANDO
LISTA DE FUNCIONARIO PRO OBRA" — o grupo "Equipe provável" só olhava `ultimaObraNome` (último
lançamento real), nunca o `obraPadrao` recém-criado. Como ninguém tinha lançamento recente
em COLÉGIO MILITAR ainda, o grupo ficava vazio e todo mundo aparecia junto, sem separação —
exatamente o cenário que gerou a reclamação original que levou ao campo `obraPadrao`, só que
o campo nunca tinha sido ligado ao Apontador de fato (a implementação anterior só criou o
campo em si, ver seção acima).

Corrigido em `render()`: o grupo (renomeado de "Equipe provável" pra **"Equipe da obra"**)
agora entra por dois sinais, nenhum dos dois restringe quem pode ser tocado (a busca sempre
enxerga todo mundo, grupo "Outros funcionários ativos" continua existindo pra isso):

```
provavel = filtrados.filter(f => f.ultimaObraNome===obra.value || f.obraPadrao===obra.value)
```

Ordenação: quem tem lançamento real recente nessa obra vem primeiro (mais recente no topo,
mesma lógica de antes); quem entrou só pelo `obraPadrao` cadastrado (sem lançamento ainda)
vem depois, em ordem alfabética. Testado local contra dado real de produção (só leitura):
COLÉGIO MILITAR mostrou corretamente os 4 cadastrados (Carlos André, Leonardo Paixão Santos,
Ronilson da Silva, Willian de Souza) em "Equipe da obra", resto em "Outros"; trocando pra
CRECHE a lista trocou junto, mostrando os cadastrados de lá. Nada foi gravado durante o
teste (só troca de obra no `<select>`, sem clicar em nenhum funcionário).

## "Outros funcionários ativos" só aparece durante busca (commit `f57bd47`, 04/09/2026)

Rubens viu a "Equipe da obra" funcionando e mandou print da lista de "Outros funcionários
ativos" (os ~22 restantes) pedindo "APAGAR OUTROS FUNCINARIOS. DIXAR BUSCA PROCURAR PELO
NOME" — achou a lista completa sempre visível abaixo poluída, agora que o grupo principal já
resolve o caso comum.

`render()`: a seção "Outros funcionários ativos" (título + lista) só é montada quando
`termoBusca` não é vazio — sem nada digitado na busca, a tela mostra só "Equipe da obra".
**Ninguém fica inacessível**: digitar qualquer nome ainda encontra e deixa apontar
normalmente, esteja ele na equipe cadastrada ou não (mesmo princípio de sempre — cadastro
nunca restringe). Adicionado também um aviso ("Ninguém cadastrado nessa obra ainda. Busque
pelo nome pra lançar alguém.") pro caso de uma obra sem ninguém na equipe e sem busca ainda
digitada, pra tela não ficar em branco sem explicação.

Testado local contra produção: sem busca, COLÉGIO MILITAR mostrou só os 4 da equipe, sem
nenhum traço de "Outros"; digitando "alex" apareceu "Outros funcionários ativos" com "Alex
Pereira Silva" (que não é da equipe de lá). Nada foi gravado (só texto de busca).

## Bug real: "Equipe da obra" não atualizava sozinha após lançar (commit `e125b0e`, 04/09/2026)

Rubens usou "Marcar todos" em CRAS de verdade (26 lançados, confirmado pelos stats "26
Inteiros / 0 Pendentes") e mandou print reclamando "NAO ESTA FUNCIONANDO CORRETAMETE" — só 2
pessoas apareciam em "Equipe da obra" (os 2 que já tinham `obraPadrao=CRAS` cadastrado antes),
os outros 24 recém-lançados não apareciam ali até recarregar a página inteira.

Causa raiz: `atualizarUltimaObra(funcId)` grava `ultimaObraNome`/`ultimaObraId`/
`ultimaObraData` no Firestore, mas **nunca atualizava o objeto correspondente dentro do
array local `todos`** — só `dadosMes[f.id]` (usado pro status do pill "Dia inteiro • 1", por
isso o pill de cada linha aparecia certo) era refrescado. O `render()` chamado logo depois
(tanto no clique individual quanto no loop do "Marcar todos") lia o `ultimaObraNome` antigo
do objeto em memória, então a pessoa só entrava em "Equipe da obra" depois de um F5 (quando
`todos` recarrega do zero via `onSnapshot`/`.get()` com o valor já gravado).

Corrigido: `atualizarUltimaObra()` agora também localiza o objeto (`todos.find(x=>x.id===
funcId)`) e muta `ultimaObraId/ultimaObraNome/ultimaObraData` nele diretamente, antes/junto
da escrita no Firestore (que continua fire-and-forget) — assim o `render()` seguinte já
reflete o grupo certo sem precisar recarregar.

Testado contra dado real de produção, com cuidado pra não sujar nada: peguei "José Garcia
Souza" (real: `ultimaObraNome=CRAS` de hoje, `obraPadrao=PARQUE IMPERIAL`, dia de hoje
realmente CRAS de verdade por causa do "Marcar todos" do Rubens), chamei
`atualizarUltimaObra()` com a obra selecionada em PRAÇA G só pra testar a mutação local +
`render()` — confirmado que ele apareceu instantaneamente em "Equipe da obra" de PRAÇA G sem
reload — e revertido imediatamente escrevendo de volta `ultimaObraNome/ultimaObraId=CRAS`,
`ultimaObraData=2026-09-04` (os valores reais de antes) direto no Firestore. **Não toquei
em `dias.{dia}.m/t` dele em nenhum momento** — só os 3 campos de cache, que já estavam
sendo testados de propósito.

## Fechamento de Ponto (02/09/2026) — HISTÓRICO, removido em 03/09/2026 (ver seção acima)

4ª aba do segmentado, ao lado da Conferência de Ponto — mesmo mês selecionado (`mesConferencia`
compartilhado). **Agrupado por obra** (não soma tudo por funcionário) — `agruparFechamentoPorObra()`
monta uma seção por obra com subtotal de dias, e cada funcionário aparece uma vez em cada obra em que
trabalhou naquele mês (dias fatiados por obra, `calcularDiasPorObra()`). **Trabalha só com dias, sem
nenhum valor em R$** (ver nota abaixo — teve uma fase intermediária com `valorDiaria`/"valor a pagar",
revertida no mesmo dia). Total de dias no topo, busca filtra por obra ou por funcionário/função, CSV com
coluna Obra. Botão "Fechar mês"/"Reabrir mês" grava
`fechamentos/{AAAA-MM} = {fechado: bool}` — mês fechado trava a Conferência de Ponto daquele mês (selects
e status Conferido/Pendente ficam `disabled` no cartão, e `salvarDia()`/`alternarConferencia()` recusam
escrever mesmo chamados direto, não só via UI).

**Cuidado ao criar outra tabela em grid**: cada linha do Fechamento é um `.fechamento-row` com
`display:grid` PRÓPRIO (não uma tabela HTML de verdade) — colunas com `auto` fazem cada linha calcular
sua própria largura, desalinhando quando uma célula (ex: PIX longo) varia de tamanho entre linhas. Usar
largura fixa em px nas colunas, não `auto`, sempre que fizer uma lista desse estilo.

**Filtro de obra em vez de cards de mês**: o Fechamento não usa mais a fileira grande de cards de mês
(essa continua só na Conferência de Ponto, `#mes-tabs`) — no lugar tem um `<select>` compacto
(`renderFechamentoMesSelect()`) e uma fileira de cards clicáveis por obra (`renderFechamentoObrasFiltro()`,
"Todas as obras" + uma por obra com os dias daquela obra). `mesConferencia` continua compartilhado entre
as duas telas.

**Sem R$ em lugar nenhum do sistema (revertido 02/09/2026, mesmo dia que foi criado)**: o Fechamento
chegou a ter um campo "Valor da diária (R$)" no cadastro (`valorDiaria`), card "Total a pagar", coluna
"Valor" na tabela e no CSV — Rubens pediu pra apagar tudo isso ("apagar qualquer referência a valor em
R$, vamos trabalhar somente com dias trabalhados"). Removido da interface inteira (campo do formulário,
card, coluna da tabela, coluna do CSV, `formatarMoeda()`, e todo cálculo de valor em
`agruparFechamentoPorObra()`) **e também apagado do Firestore** — o campo `valorDiaria` que já estava
gravado nos 32 `funcionarios/{id}` foi removido de vez com `FieldValue.delete()` a pedido do Rubens (não
ficou como dado morto). Se `valorDiaria` for pedido de novo no futuro, a tabela usada em 02/09 foi:
Ajudante R$100, Pedreiro/Carpinteiro/Pintor R$150, Encarregado R$200 (só como referência histórica, não
está mais no banco).

**Pill de pagamento (Pendente/Aprovado)** em cada linha do Fechamento — guardada por obra, não por mês
inteiro (`funcionarios/{id}/pontos/{mes}.pagamentos.{obra}`, `statusPagamento()`/`alternarPagamento()`),
já que uma pessoa pode aparecer em mais de uma obra no mesmo mês. **Independente do mês estar fechado** —
fechar só trava o lançamento dos dias; aprovar pagamento é uma etapa separada, geralmente feita depois.
2 cards extras no topo mostram "Aprovado" e "Falta aprovar" — mostram **quantidade de funcionários**
("N de M funcionários", pedido explícito do Rubens), respeitando o filtro de obra. "Falta aprovar" é
sempre `total - aprovado` em contagem de gente (`qtdTotal`/`qtdAprovados` em
`agruparFechamentoPorObra()`).

**Cuidado extra ao testar toggles de status (Conferido/Pendente, Ativo/Inativo, Pagamento) em produção**:
nunca assumir o estado inicial — ler antes de togglear. Já aconteceu de um teste reverter sem querer uma
aprovação real que o Rubens tinha acabado de marcar na tela dele, porque o teste presumiu "deve estar no
padrão" em vez de checar. Se o Rubens acabou de mandar uma print mexendo numa tela, o estado dela É real,
não padrão.

**`carregarObras()` roda no `iniciar()`, não mais sob demanda**: antes só carregava quando a aba "Obras"
era clicada — até lá, `OBRAS` ficava no fallback hardcoded `OBRAS_SEED`, que chegou a ficar desatualizado
(tinha "UBS" muito depois dela virar "COLÉGIO MILITAR"). Se acontecer de novo — o dropdown de obra em
qualquer lugar mostrando um nome que não existe mais na aba Obras — é sinal de `OBRAS` desatualizado;
confirmar que `carregarObras()` está sendo chamado cedo o bastante e que `OBRAS_SEED` está com os nomes
certos como fallback.

## Análise de arquitetura recebida 02/09/2026 — NÃO IMPLEMENTAR sem ordem explícita

Rubens mandou uma proposta grande (modularizar `index.html` em `css/`/`js/`/`components/`, páginas
admin/funcionário separadas, Firebase Authentication, Cloud Functions pra escritas sensíveis, e um
projeto futuro de GPS+biometria pro funcionário bater ponto sozinho) e disse explicitamente **"não vamos
mexer em nada até segunda ordem"**. Nada disso foi implementado. Se uma sessão futura for trabalhar
nisso: reconfirmar com o Rubens que a ordem foi dada de verdade — não presumir que uma análise antiga já
é permissão, nem que pedidos pontuais (como o `cartoes.html` abaixo) equivalem a essa ordem geral.

## cartoes.html — implantada oficialmente (02/09/2026)

Rubens pediu duas vezes uma forma do funcionário conferir o próprio cartão antes do fechamento. A
primeira tentativa (`funcionario.html`, link único por pessoa, com botão "Copiar link" no `index.html`)
foi construída, publicada, testada — e **desfeita** (`git revert`) logo depois, sem explicação. A segunda
tentativa, `cartoes.html`, segue um formato diferente: **um único link com a lista inteira de
funcionários** (busca por nome, botão "Abrir Cartão" por pessoa, mês selecionável dentro do cartão,
resumo por obra dividido em 1ª/2ª quinzena) em vez de link individual. Foi construída primeiro como
página **isolada** ("não implementar no sistema agora, versão somente para teste"), testada, ajustada
(removida a coluna "Dias" da lista, total virou 2 quinzenas) — e depois **implantada de vez**: Rubens
confirmou com "IMPLANTAR" quando perguntei se quer formalizar.

Está **publicada, ao vivo, e agora oficialmente parte do sistema**:
`https://rubinho106-hash.github.io/equipe-concretta/cartoes.html`, com um link discreto (`.hero-link`,
ícone de corrente) no topo do `index.html`, logo abaixo do subtítulo do hero, abrindo `cartoes.html` numa
aba nova. Mesmo padrão de segurança de antes: zero escrita no Firestore, sem PIX, sem valor, só
mês/obra/dias — e a mesma limitação de privacidade (regras do Firestore abertas, então é obscuridade por
URL, não uma trava de verdade — Rubens já foi avisado disso).

## Cadastro (32 pessoas em 02/09/2026)

Os 32 originais do seed, **menos** Daniel Júlio, Valdecir Alves dos Reis, Lucas Gomes Araujo e Silvestre
Pereira da Silva (todos excluídos de propósito pelo Rubens em 02/09 — confirmado explicitamente cada vez
que sumiram, não é bug), **mais** 4 novos cadastrados no mesmo dia pela equipe do "Willian Ponte" (chefe
de equipe novo no WhatsApp, não confundir com o funcionário Willian de Souza): Willian de Souza
(Encarregado), Carlos André (Ajudante), Walison Ricardo (Carpinteiro), Ronilson da Silva (Ajudante) —
nenhum tem PIX cadastrado ainda. **O Firestore não guarda histórico de quem foi excluído** — se precisar
checar isso de novo, comparar a lista `todos` ao vivo contra o array `SEED` hardcoded no código (só
serve de base pros 32 originais, não pega quem foi cadastrado depois). Nota: o alias "Silvestre de
Oliveira" → Silvestre Pereira da Silva (na lista de apelidos acima) ficou órfão desde essa exclusão —
deixado como está, só volta a valer se essa pessoa for recadastrada.

**Valor da diária por função (definido 02/09/2026)**: Ajudante R$100, Pedreiro/Carpinteiro/Pintor R$150,
Encarregado R$200. Aplicado a todos os 32 via `valorDiaria` no cadastro. Usar essa tabela pra qualquer
funcionário novo — se aparecer uma função fora dessas quatro, perguntar ao Rubens antes de inventar um
valor.

## Filtro de quinzena no cartão do admin — estado final (03/09/2026)

Passou por três versões no mesmo dia até chegar no formato atual:

1. Toggle "1ª quinzena / 2ª quinzena" **dentro** do cartão (botões visíveis, trocavam
   quais dias apareciam na tabela).
2. Rubens pediu pra tirar esse toggle do cartão e deixar o filtro só na lista — nessa
   versão intermediária o cartão passou a mostrar o **mês inteiro sempre**, sem filtro
   nenhum, e o botão "Abrir Cartão" de cada coluna da lista só mudava qual quinzena o
   pill de status refletia (não filtrava os dias).
3. **Ajuste final, mesmo dia**: Rubens esclareceu — "separar as quinzenas: abrir cartão
   quinzena 1 mostra do dia 1 ao 15, abrir cartão quinzena 2 mostra do dia 16 ao fim do
   mês". Ou seja, ele queria o toggle **removido de dentro do cartão** (isso ficou), mas
   o **filtro por dias mantido**, só que decidido por qual botão "Abrir Cartão" da lista
   foi clicado, em vez de um toggle visível dentro do cartão.

**Estado final**: sem toggle dentro do cartão (removido do HTML/CSS, não volta). A
tabela de dias É filtrada — `diaInicio`/`diaFim` em `renderFichaConteudo()` voltaram a
depender de `fichaQuinzena` (`1`→dias 1-15, `2`→dia 16 até o fim do mês). `fichaQuinzena`
é setado só por `abrirFicha(f, quinzenaAlvo)`, chamado pelos botões "Abrir Cartão" da
lista (coluna 1ªQ passa `1`, coluna 2ªQ passa `2` — ver seção "Lista de Conferência de
Ponto" abaixo). O pill de status do cabeçalho segue essa mesma quinzena e tem rótulo
("1ª quinzena: Conferido") já que não tem mais toggle visível mostrando o contexto.

`ajustarAlturaFichaParaConteudo()` roda tanto numa abertura nova (dentro de
`posicionarFichaCentro()`) quanto quando o cartão já estava aberto e a quinzena muda (ex:
clicar "Abrir Cartão" de outra coluna/pessoa com o cartão já na tela) — útil de novo
porque voltou a ter só 15-16 linhas por vez (cabe em 92vh na maioria das telas).

## Status de conferência guardado por quinzena (03/09/2026)

Rubens mandou print do pill "Conferido" no topo do cartão pedindo "guardar status separado
por quizena". `statusConferencia` deixou de ser uma string única pro mês inteiro e virou
`{q1, q2}` (`Pendente`/`Conferido` cada). Pontos importantes:

- `garantirDocPontos()` já cria o doc novo nesse formato de objeto.
- `alternarConferencia(f, quinzena)` (ganhou o parâmetro `quinzena`) só alterna a quinzena
  pedida, preservando a outra — grava `statusConferencia` **inteiro** de uma vez
  (`{q1:..., q2:...}`), nunca um dot-path tipo `statusConferencia.q1`, porque um doc antigo
  pode ter esse campo como string solta e um dot-path em cima disso quebra.
- **Compatível com dados antigos sem migração em lote**: `statusQuinzena(info, quinzena)` e
  `statusMesConferido(info)` leem `statusConferencia` como string (formato de antes de
  03/09) e usam esse valor como fallback pras duas quinzenas — só vira objeto de verdade no
  primeiro toggle que alguém fizer depois desse deploy.
- O pill do cabeçalho do cartão (`#ficha-status`) segue a quinzena selecionada no momento
  (`fichaQuinzena`) — populado dentro de `renderFichaConteudo()` (não mais só em
  `abrirFicha()`), então reage a trocar de quinzena.
- O stat "Conferidos" do topo é resumo do mês inteiro: só conta "Conferido" quando as DUAS
  quinzenas estão conferidas (`statusMesConferido()`, lógica E).
- **(Superseded no mesmo dia)** O pill da linha na lista chegou a virar um resumo único
  (clicar abria o cartão em vez de togglear) — isso foi substituído horas depois por duas
  colunas de pill+botão, uma por quinzena, direto na lista. Ver "Lista de Conferência de
  Ponto: coluna DIAS saiu, virou 2 colunas por quinzena" mais abaixo pro estado atual.

## Lista de Conferência de Ponto: coluna DIAS saiu, virou 2 colunas por quinzena (03/09/2026)

Rubens mandou print marcando a coluna "DIAS" (contagem de dias) da lista principal e pediu
pra excluir, e trocar o pill único "Conferido/Pendente" + botão "Abrir Cartão" por **duas
colunas, uma por quinzena**. Cada linha agora tem: pill "1ªQ Conferido/Pendente" + "Abrir
Cartão" (abre já na 1ª quinzena) e o mesmo pra "2ªQ". `contarDias()` ficou sem uso e foi
removida. `abrirFicha(f, quinzenaAlvo)` ganhou um segundo parâmetro opcional — passar `1`
ou `2` abre o cartão já naquela quinzena (os botões da lista usam isso); sem o parâmetro,
mantém o comportamento de antes (1ª quinzena numa abertura nova, ou a quinzena que já
estava selecionada se o cartão já estava aberto). Clicar direto num pill da lista continua
alternando só aquela quinzena (`alternarConferencia(f, q)`), sem precisar abrir o cartão.

## Lançamento diário de ponto por WhatsApp

Desde 02/09/2026 o lançamento diário vai direto aqui (não mais no Excel antigo). Fluxo:

1. Rubens manda print(s) do WhatsApp com equipe + obra do dia.
2. Resolver nomes contra o array `todos` (carregado do Firestore) usando a lista de apelidos acima.
3. **Sempre restatar a leitura (obra + nomes + data + turno assumido) e pedir confirmação antes de
   gravar** — nomes ambíguos, obra sem data explícita e dia inteiro vs. meio período já geraram mais de
   uma rodada de correção. Não pular essa etapa pra economizar tempo.
4. Depois de confirmado, gravar direto via o próprio JS da página (`garantirDocPontos(funcId, mes)` e
   `ref.update(...)` com sintaxe de ponto) — é dado de produção real, não precisa reverter depois.
5. Nome sem match e sem apelido conhecido: parar e perguntar, não chutar.
