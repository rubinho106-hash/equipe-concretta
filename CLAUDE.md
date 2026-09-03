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

## Fechamento de Ponto (02/09/2026)

4ª aba do segmentado, ao lado da Conferência de Ponto — mesmo mês selecionado (`mesConferencia`
compartilhado). **Agrupado por obra** (não soma tudo por funcionário) — `agruparFechamentoPorObra()`
monta uma seção por obra com subtotal de dias/valor, e cada funcionário aparece uma vez em cada obra em
que trabalhou naquele mês (dias fatiados por obra, `calcularDiasPorObra()`). Dias trabalhados ×
`valorDiaria` (campo novo no cadastro, R$ por dia completo) = valor a pagar. Total geral no topo, busca
filtra por obra ou por funcionário/função, CSV com coluna Obra. Botão "Fechar mês"/"Reabrir mês" grava
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
"Todas as obras" + uma por obra com dias/valor daquela obra). `mesConferencia` continua compartilhado
entre as duas telas.

**Pill de pagamento (Pendente/Aprovado)** em cada linha do Fechamento — guardada por obra, não por mês
inteiro (`funcionarios/{id}/pontos/{mes}.pagamentos.{obra}`, `statusPagamento()`/`alternarPagamento()`),
já que uma pessoa pode aparecer em mais de uma obra no mesmo mês. **Independente do mês estar fechado** —
fechar só trava o lançamento dos dias; aprovar pagamento é uma etapa separada, geralmente feita depois.
2 cards extras no topo mostram "Aprovado" e "Falta aprovar" (dias+valor, respeitando o filtro de obra) —
"Falta aprovar" é sempre `total - aprovado`.

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

## Página de teste isolada: cartoes.html (02/09/2026, ao vivo mas não integrada)

Rubens pediu duas vezes uma forma do funcionário conferir o próprio cartão antes do fechamento. A
primeira tentativa (`funcionario.html`, link único por pessoa, com botão "Copiar link" no `index.html`)
foi construída, publicada, testada — e **desfeita** (`git revert`) logo depois, sem explicação. A segunda
tentativa, `cartoes.html`, segue um formato diferente que ele pediu explicitamente: **um único link com a
lista inteira de funcionários** (busca por nome, contador de dias do mês, botão "Abrir Cartão" por
pessoa) em vez de link individual — e com a instrução clara **"não implementar no sistema agora, versão
somente para teste"**.

Está **publicada e ao vivo** em `https://rubinho106-hash.github.io/equipe-concretta/cartoes.html`, mas
**totalmente isolada** do `index.html` — nenhum link aponta pra ela a partir do painel principal, e ela
não referencia nada de lá. "Não implementar no sistema" foi interpretado como "não mexer no `index.html`
nem no fluxo do admin", não como "não publicar nada" (o Rubens pediu um link de verdade pra testar).
Mesmo padrão de segurança do `funcionario.html`: zero escrita no Firestore, sem PIX, sem valor, só
mês/obra/dias.

**Se ele pedir pra "integrar de vez"**: a forma natural seria um botão/link discreto em algum canto do
`index.html` apontando pra essa URL — só fazer quando ele pedir explicitamente, e checar primeiro se essa
página ainda existe/não foi desfeita de novo nesse meio tempo.

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
