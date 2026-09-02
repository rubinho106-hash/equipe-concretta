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

## Apelidos e nomes de obra (grupo do WhatsApp escreve errado/informal)

- "Praça" / "Praça Imperial" → **PARQUE IMPERIAL**
- "Praça canto da serra" → **PRAÇA G** ("canto da serra" é bairro, não nome de obra)
- "Cras canto da serra" → **CRAS** (mesma lógica)
- "Creche canto da serra" → **CRECHE**
- "Cleber Lucas" / "Kleber Lucas" → Kleber Lucas Sousa Soares
- "Marlon Molra" → Marlon Moura Silva
- "Dairo Silva" → Dário Silva Dos Santos
- "Wesley Numes" → Wesley Nunes Conceição
- "Leonardo Oliveira" → Leandro Oliveira Silva, Carpinteiro (**não confundir** com "Leonardo paixão" =
  Leonardo Paixão Santos, pessoa diferente, aparecem juntos às vezes)
- "Silvestre de Oliveira" → Silvestre Pereira da Silva
- "Edson Silva" → Edison Silva de Oliveira

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
