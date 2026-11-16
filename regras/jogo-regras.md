# GymGames — Regras do jogo e modelo de pontuação

> Documento de referência para quem for modificar o jogo (inclusive outros modelos de IA). Descreve o que o jogo **faz hoje** e onde cada regra está no código (`fantasy_gymnastics_royale.html`).

## 1. Estrutura dos arquivos

| Arquivo | Conteúdo |
|---|---|
| `index.html` | Redireciona para o jogo (com `?v=` para forçar atualização no GitHub Pages) |
| `fantasy_gymnastics_royale.html` | Jogo inteiro: CSS, dados de atletas (CSV embutido), motor de notas, telas, i18n (pt-BR, en, es) |
| `codigo-data.js` | `window.GYM_CODE` — tabelas de elementos FIG usadas na tela **Código de Pontuação**. Gerado de `regras/elementos.json`; se faltar, o jogo funciona sem a tabela |
| `regras/*.md` | Regras e tabelas de elementos em Markdown (este diretório) |
| `codigo-img/*.jpg` | Páginas das tabelas de elementos dos PDFs (100 dpi, tons de cinza). O jogo recorta delas a figura e o símbolo de cada elemento |
| `regras/elementos.json` | Mesmos dados das tabelas em JSON: `{WAG:{VT,UB,BB,FX}, MAG:{FX,PH,SR,VT,PB,HB}, groups}`. Cada linha: `[código, grupo, valor, descrição, nome, dança?, página do PDF, recortes]`. Variantes da mesma caixa são separadas por ` // ` na descrição; `recortes` tem um `[x, y, largura, altura]` (frações da página) por variante, ou `0` quando não há figura |

O estilo usa Tailwind via CDN + CSS próprio no `<style>`. Tudo fica dentro de uma IIFE; ações de botões passam por `data-act="..."` e pelo objeto `ACTIONS`.

## 2. Fluxo de uma competição clássica

1. **Configuração** — nome da equipe, bandeira, modalidade (GAF/GAM), competição (Olímpicos, Pan-Americanos, Europeu, Asiático) e modo difícil.
2. **Draft** — 5 rodadas; em cada uma é sorteado um país + ano e o jogador escolhe 1 atleta. Há um re-sorteio de ano e um de equipe por draft.
3. **Classificatórias** — 5 atletas por equipe competem em todos os aparelhos; **contam as 3 melhores notas por aparelho**. No salto são 2 saltos (média).
4. **Final por equipes** — 8 melhores equipes; em cada aparelho sobem os 3 melhores da classificatória e as 3 notas contam.
5. **Individual geral** — 24 finalistas, máx. 2 por país.
6. **Finais por aparelho** — 8 finalistas por aparelho, máx. 2 por país.
7. **Resumo** — quadro de medalhas, card exportável (PNG) e registro na Carreira (conquistas, `localStorage`).

Modo tradicional: adversários são equipes reais de um país/ano. Modo difícil: adversários all-star por país.

### Battle Royale livre

64 equipes de 6 atletas (GAF e GAM misturados). Três formatos: **royale** (classificatória 8 rotinas → 12 avançam; semifinal 6 rotinas → 4; final 4 rotinas), **um por aparelho** e **show off** (pontos relativos, alvo 100 por rotina). Estratégias: equilibrada, especialistas, agressiva.

## 3. Modelo de nota (função `buildRoutine`)

`Total = D + E − deduções neutras`

- **D** — valor `d` do atleta no aparelho (dados do CSV), ajustado pela tática e pelo bônus de cravada.
- **E** — 10 menos as deduções sorteadas, avaliada por **5 árbitros**; descartam-se a maior e a menor e faz-se a média das 3 centrais (`judgePanelExecution`).
- Deduções detalhadas em passos de 0,1 (0,1–0,3 por falta, máx. 0,5 por tipo), sorteadas de um conjunto por aparelho (`FAULT_POOL`).
- **Queda = 1,00**; erro grave = 0,3–0,8. Probabilidade depende do risco do aparelho, da consistência do atleta e da pressão (finais, última rotação).
- **Deduções neutras:** solo — fora da área (0,1 ou 0,3); solo e trave — tempo (0,1).
- **Salto com 2 tentativas:** média das duas (`competitionScore` com `twoVaults`).
- **Bônus de cravada (GAM): +0,1 em D** quando a série é "cravada", exceto no cavalo com alças (Código GAM, Art. 7.2.9).

### Tática da equipe (`TACTICS`, só competições clássicas, só atletas do jogador)

| Tática | Nota D | Risco de queda/erro grave | Deduções típicas |
|---|---|---|---|
| Segura | −0,3 | × 0,62 | −0,12 |
| Equilibrada | 0 | × 1 | 0 |
| Ousada | +0,3 | × 1,45 | +0,14 |

Pode ser trocada antes de cada fase (tela da competição e telas de resultado).

## 4. Código de Pontuação dentro do jogo (botão de livro no topo)

- **Regras** — resumo das regras FIG 2025–2028 (`CODEX_RULES`).
- **Elementos** — tabela completa por modalidade/aparelho, filtro por valor e busca.
- **Desafio** — quiz configurável: marca-se quais aparelhos (por sexo) entram, os tipos de pergunta e o número de perguntas (5, 10 ou 20). Tipos: **valor** do elemento; **nome pela figura** (mostra desenho + símbolo, escolhe a descrição); **figura e símbolo** (mostra a descrição, escolhe entre 4 figuras); **ginasta que dá nome**; **grupo** do elemento; **regras**. Filtros em `localStorage` (`gym-royale-quiz-cfg`), recorde em `gym-royale-codex-best`. Em Elementos, "ver figura" mostra o recorte da página.
- **Montador** — monta uma série e calcula a Nota D:
  - GAF: 8 maiores valores incluindo a saída (trave/solo: reserva 3 de dança e 3 acrobáticos), CR marcado manualmente (4 × 0,5), CV manual, bônus de saída +0,2 automático para saída D+.
  - GAM: 7 melhores + saída, máx. 4 por grupo; requisitos de grupo calculados automaticamente (EG I 0,5; II/III 0,5 com D+ ou 0,3; saída = valor da saída; solo com 4 grupos); CV manual.

## 5. Narração

- Frases genéricas por situação: `COMMENTARY` / `EXTRA_COMMENTARY`.
- Histórias por atleta: `ATHLETE_STORIES` (pt/en/es) e `ATHLETE_FEATS` (feitos reais, em português): `[atleta, ano do card ou "Todos", competição ("" | "olympics" | "panamerican"), texto]`.
- A cada rotina ao vivo o narrador conta um feito ainda não usado daquele atleta; têm prioridade os feitos da competição em disputa e do ano do card.
- **Regra para novos feitos: só fatos documentados** (resultados oficiais). Paris 2024 e Santiago 2023 foram conferidos na Wikipédia em 16/11/2026.

## 6. Simplificações em relação ao Código

- As séries não são simuladas elemento a elemento: a Nota D vem do banco de atletas.
- CR/CV do Montador (GAF) são informados pelo usuário; o jogo não detecta ligações.
- Bônus de salto GAF (+0,2 por mortais em direções diferentes), salto nulo, exercício curto e regra de linha no salto não entram na simulação.
- Textos das regras no jogo estão em português; inglês e espanhol traduzem só os rótulos principais.

## 7. Como alterar

- **Valores/elementos:** edite `regras/elementos.json` e regenere `codigo-data.js` (`window.GYM_CODE = <conteúdo do JSON>;`). Atualize também o `.md` correspondente e o `?v=` do `<script src="codigo-data.js">`.
- **Regras exibidas:** `CODEX_RULES`, `CODEX_CR` e `CODEX_QUIZ_RULES` no HTML.
- **Equilíbrio do jogo:** `TACTICS`, `APPARATUS[...].risk`, `buildRoutine`, `BATTLE_CONFIG`.
- **Atletas com notas próprias:** blocos `ATHLETE_*_CSV` (colunas: gênero, nome, país, ano, D e E por aparelho, consistência).
- **Atletas por elenco:** `ATHLETE_COVERAGE_PACKS` e `ATHLETE_WORLDS_OLYMPICS_2021_2026_PACKS` — `{tag, tier, gender, country, year, offset?, names}`. Os atributos são **estimados** pelo `tier` (`olympic`, `continental`, outro) e pela posição do nome na lista (`coverageCsvLine`); `offset` desloca essa posição. Os elencos de 2021–2026 estão em [elencos-2021-2026.md](elencos-2021-2026.md). Países novos entram nas listas de `COMPETITIONS` por continente em `extendCompetitionCountries`.
- **Desempenho do draft:** `draftPool`, `availableYearsForCountry` e `availableCountriesForYear` usam índices em cache (`DRAFT_INDEX`, `DRAFT_GENDER_INDEX`); o banco de atletas é tratado como fixo depois de carregado.
- Depois de publicar, mude o `?v=` em `index.html` para forçar atualização.
