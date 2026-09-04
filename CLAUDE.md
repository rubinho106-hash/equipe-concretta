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
  - `funcionarios/{id}` — cadastro (nome, função, pix, status)
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
