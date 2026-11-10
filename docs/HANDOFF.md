# Revela a Oito! · Handoff

Documento para continuar o projeto numa conversa nova. Não contém respostas das perguntas.

## 1. O que é
Jogo de festa para 2 a 5 jogadores numa tela só (TV, notebook ou celular passado de mão em mão).
A pergunta aparece escondida em retângulos numerados. Na sua vez, o jogador pede uma palavra pelo número
("quero a 8"), ela é revelada para todos e ele tem o tempo da vez para **escrever** a resposta ou passar.
Quem acerta leva 1 ponto por palavra que ainda estava escondida. Pontuação (vírgula, interrogação) não conta e fica sempre à mostra.

## 2. Estado atual (pronto e funcionando)
- **Arquivo único** `index.html` (HTML + CSS + JS + dados). Dependências: Google Fonts e o three.js da ampulheta 3D (cópia local em `assets/vendor/three.module.min.js`, com CDN de reserva; sem ele, cai para a ampulheta 2D).
- **Jogadores (elenco + mesa):** tela "Quem vai jogar?" com duas partes.
  - *Na mesa:* quem joga a próxima partida (2 a 5), na ordem das vezes; ✕ tira da mesa; linha "Novo jogador" (emoji + nome) cadastra e já senta na mesa. "Limpar mesa" esvazia só a mesa.
  - *Jogadores salvos:* chips de todos os jogadores do aparelho (mais recentes primeiro); tocar põe/tira da mesa. "Editar" mostra ✕ para excluir (aviso com "Desfazer" por 5 s) e tocar no chip abre a folha de editar nome/emoji.
  - Nome único no elenco (sem diferenciar acento/caixa; digitar um nome salvo chama o jogador existente). Emoji fixo por jogador e único no elenco (limite = 30 emojis).
  - Dados: `revela:roster` = `[{id, name, emoji, createdAt, lastPlayedAt}]` (id UUID fixo, futura chave no Supabase) e `revela:table` = ids da mesa. O formato antigo `revela:players` é migrado sozinho na primeira abertura.
- **Temas:** 30 temas, 518 perguntas, cada tema com cerca de 1/4 de perguntas longas (20+ palavras). Card "Aleatório" sorteia um tema por pergunta, sem repetir o mesmo tema seguido. Uma pergunta só conta como jogada quando chega à resposta (`markPlayed` em `endQuestion`; `revela:used`). Jogadas não voltam até esgotar o tema; aí só aquele tema recomeça (`refreshExhausted`). Cada card de tema mostra "N de M jogadas" + barrinha; cheio = "Tudo jogado · vai recomeçar" (`themeCount`, `playedIn`).
- **Configurações:** nível (Normal ou Difícil), perguntas por partida (5/8/10/12), modo (Clássico ou Rápido · fichas), tempo (30 s / 40 s / 1 min / 90 s; padrão 40 s, chave `revela:tempo`), dicas on/off, som e vibração.
- **Nível Difícil:** sem dicas e tema secreto (o header mostra "Tema secreto" e revela o tema quando a pergunta termina).
- **Partida:**
  - Retângulos numerados; revelar por toque ou digitando o número no teclado (+ Enter).
  - Quadro que se ajusta ao telão: calcula o maior tamanho de letra em que a pergunta inteira cabe (função `fitBoard`/`fitEl`, medição por canvas com Outfit 800).
  - Ampulheta animada (areia, gira a cada vez, balança e fica vermelha nos últimos 10 s; tique a partir de 15 s, mais rápido nos últimos 10 s).
  - Resposta digitada com correção tolerante: ignora acento, hífen, caixa, artigos iniciais; aceita respostas alternativas (`ALIASES`) e erros de digitação por distância de Levenshtein proporcional ao tamanho. Faixa "Quase!" dá uma segunda tentativa.
  - Errou: campo vermelho, som de erro (`assets/sons/erro.mp3`, embutido em base64 no `index.html` e tocado pelo Web Audio via `decodeAudioData`; reserva: `<audio>`; o "béé" sintetizado não é mais usado), passa a vez sem mostrar a resposta.
  - Acertou: fanfarra + confete; o resultado aparece na bancada, com o telão inteiro aberto.
  - Respostas que não passaram aparecem no resultado com botão "Aceitar +N" (decisão do grupo).
  - Ficha de dica: 1 por jogador na partida; mostra primeira letra + quantidade de letras para todos, mas a vez continua de quem usou.
  - Reportar problema: ícone de bandeira no topo e link no resultado; motivos + comentário. Hoje grava no banco do artefato do claude.ai (`window.claude.use("db")`) ou, fora dele, no `localStorage` (`revela:reports`).
  - Menu: pular pergunta, encerrar e ver placar.
- **Final:** pódio animado (2º, 1º, 3º) e lista para 4º e 5º; linha do ranking ("Novo líder…" / "chega a N vitórias") e botão "Ver ranking".
- **Animação da vez:** discreta, na bancada e no placar. Na bancada o jogador anterior sai para cima e o próximo entra de baixo; por 1,6 s a instrução dá lugar a "Ana passou" / "Ana errou" / "Tempo de Ana acabou" / "Nova pergunta · começa a vez" (os toasts antigos saíram). No placar (só no celular) um sublinhado rosa (`.turn-ind`) desliza até quem está na vez; no computador o card já tem contorno, então não há linha lateral e o card cresce 3% e volta (ease-in-out, 0,56 s). Som "plim" curto. O telão não tem mais borda rosa (o rosa fica para quem está na vez). Código: `showTurn`, `moveInd`, `EVT`, `S.turnKey`, `S.pendingEv`, `nextTurn(ev)`, `sfx.turn`.
- **Palavra 8 (regra própria do jogo):** a 8ª palavra aparece com uma mini bola 8 no lugar do número. Fica trancada (cadeado, não clicável, nem pelo teclado) até todos jogarem uma vez na pergunta (`S.turnsTaken`); ao liberar, pulsa uma vez. Quem abre a 8 arrisca: acertou na mesma vez = pontos normais +8; errou, passou ou o tempo acabou = ele −8 e cada um dos outros +8 (aviso com a bola 8 no telão). Na pergunta de ouro vira ±16. "Aceitar" no resultado desfaz a punição e dá o bônus. Placar pode ficar negativo. Nos textos fala-se só "a 8" (sem "bola"). Código: `S.eightIdx`, `S.eightBy`, `eightLocked`, `eightPenalty`, `showEightFlash`, `val8`, `BALL8`, `sfx.eight`.
- **Eu sei! (fora da vez, 1 por partida):** antes de quem está na vez responder (fases de escolher número e de pensar), outro jogador toca no próprio card do placar ("Eu sei!"; no celular, um "!" rosa no chip). Abre a folha `#o-oos`, o tempo da vez pausa, 20 s para escrever, "Quase!" dá 2ª tentativa, "Desistir" não gasta a chance. Acertou = metade dos pontos (arredondado para cima; dobra na de ouro) e encerra a pergunta. Errou ou tempo = −8, chance gasta, a vez continua. Código: `canOOS`, `openOOS`, `oosFail`, `closeOOS`, `p.oos`.
- **Ampulheta:** no computador é 3D (three.js), flutua no espaço livre embaixo do placar, gira devagar, vira ao começar a vez; areia no rosa do DS (`--pink`) e, nos 10 s finais, a ampulheta inteira fica no vermelho de erro (`--bad`) e balança (vale também para a 2D). A areia de cima é um sólido que acompanha o raio interno do vidro (não sai do bulbo). Parallax: ampulheta e número flutuam em ritmos diferentes (5,5 s e 7,4 s) e seguem o mouse com intensidades diferentes (ampulheta mais, número menos); o número fica ao lado. No celular é 2D, na bancada, com o número ao lado. Tique a cada segundo nos 30 s finais e duas vezes por segundo (mais agudo) nos 10 finais; vale também na folha do Eu sei!.
- **Bancada:** jogador da vez + ações (resposta, Passo, Responder, Dica). "Como pontuar" e atalhos foram para o menu da partida em **Como jogar** (pausa o tempo).
- **Modo claro/escuro:** botão sol/lua fixo no canto superior direito em todas as telas; na partida fica no cabeçalho.
- **Aviso da 8:** quando alguém erra/passa/perde o tempo com a 8 aberta, a tela escurece e aparece um cartão grande com a bola 8 e os pontos de cada um (−8 / +8) por 4,5 s (Continuar, toque fora ou Esc fecham). A vez só passa depois.
- **Aleatório:** ao ligar, todos os temas aparecem marcados e ficam travados; ao desligar, volta a seleção anterior.
- **Voz (narração + microfone com orbe):** pronta no código, mas **desligada** por `VOZ_ATIVA=false`. Para ligar, trocar para `true` (aparecem o botão Falar e a chave Narração). Microfone funciona no Chrome e Safari; Firefox não.
- **Roleta + estrelas:** antes de cada pergunta, uma esteira horizontal de cards de tema (temas da partida; todos no Aleatório) gira ~5 s desacelerando, para sob o marcador rosa no tema sorteado, e as estrelas de dificuldade (1 a 5) acendem com o rótulo (Moleza, Fácil, Médio, Difícil, Cabeludo); fica ~3 s e libera a pergunta. "Pular"/toque adianta. No Difícil para em "Tema secreto". Com 1 tema só ou movimento reduzido, mostra só o resultado. Durante o sorteio o cabeçalho mostra "Sorteando…" e não dá para abrir palavra nem usar Eu sei!. Estrelas pequenas ficam no topo do telão. Chave "Roleta antes da pergunta" nas configurações. Estrelas **não** mudam pontos.
- **Dificuldade das perguntas:** `DIFF[tema]` = um dígito (1–5) por pergunta, na ordem do tema. Escala **relativa ao jogo** (≈20% 1★, 30% 2★, 28% 3★, 15% 4★, 7% 5★), feita lendo pergunta e resposta nos bastidores. Ao criar perguntas novas, acrescentar o dígito no tema (sem dígito = 3★).
- **Novo tema / Trocar pergunta (topo da partida):** "Novo tema" troca a pergunta por outra de tema diferente e gira a roleta de novo (só antes de abrir a primeira palavra). "Trocar pergunta" traz outra do mesmo tema (dificuldade aleatória), sem mostrar a resposta nem a pergunta; pede "Trocar?" se já abriram palavras; roleta curta com 1 card. A trocada volta para o monte (não conta como jogada). O "Desistir" foi removido. No celular viram ícones; "Reportar" e "Modo claro/escuro" vão para o menu. Código: `drawQuestion(excludeTheme,onlyTheme)`, `syncGroupBtns`, `resetSwap`, `S.quickSpin`.
- **Modo Rápido · fichas (para partidas rápidas, ex.: 2 jogadores):** cada jogador recebe fichas para a partida toda: `min(8, round(perguntas×0,8))` → 5:4, 8:6, 10/12:8. Na sua vez, depois de abrir a palavra, "Palavra extra" gasta 1 ficha e abre mais uma (máx. 1 por vez; "Cancelar" não gasta). A ampulheta NÃO pausa: decidir faz parte do tempo. Sem custo em pontos (a palavra a mais já baixa o valor e fica aberta para todos). A palavra 8 vira comum (`S.eightIdx=-1`). Placar mostra as fichas. Código: `S.fast` (`revela:fast`), `fichasN`, `syncMode`, `S.extra`, `S.fichaAt`, `p.fichas`, `#b-ficha`, `sfx.coin`.
- **Pergunta de ouro:** a última pergunta da partida vale o dobro (`worth()` × 2). Anúncio com moeda no telão (~2 s, toque fecha), telão com borda dourada, retângulos dourados, "vale X" dourado, selo "★ Pergunta de ouro · 2x" no cabeçalho ("★ 2x" no celular) e último segmento da barra contornado em dourado desde o início.
- **Ranking local:** tela `#s-rank` (botão na home). Por jogador salvo: vitórias, partidas e % de vitórias; líder em rosa (sem líder se empate no topo). Empate no topo = vitória para todos; partida sem pontos não conta; jogador excluído some do ranking; "Zerar ranking" com confirmação (`#o-confirm`). Dados em `revela:matches` = `[{id, at, players:[{id,name,emoji,score}], winners:[id]}]`, pronto para o Supabase.
- **Modo claro/escuro:** botão sol/lua na home e item no menu da partida; salvo em `revela:theme` (sem escolha, segue o sistema). O telão tem versão clara (lavanda com retângulos brancos), tudo por tokens (`--screen-*`, `--tile-*`, `--gold-*`). Contraste revisado (WCAG AA): botões rosa usam `--pink-btn` (#D6206F) nos dois modos, texto rosa sobre rosa claro usa `--pink-ink`.
- **Favicon:** bola 8 em SVG inline (o 8 é desenhado, sem depender de fonte).
- **Pergunta no telão alinhada à esquerda** (não centralizada).
- **Emojis:** 78 no total (30 originais + 48 mais engraçados).

- **Testes de melhorias:** ficam em `docs/testes/` (ex.: `melhoria-2-vez.html`, `teste-3-7-10-12.html`, `teste-4-13.html`, `teste-8-9.html`, `teste-avatares.html`). Regra: teste HTML aprovado antes de mexer no `index.html`.

## 3. Design (DS "Phantom", o mesmo do Duvido!)
- **Conceito:** telão de auditório. O quadro de palavras é a peça principal; o resto é "bancada", discreto e alinhado a ele.
- **Cores:** fundo `#120C22`, superfícies `#1D1535`/`#251C44`, telão `#2B2160`, retângulo `#1A1345`, lavanda `#B9A8FF`, rosa `#FF4F9A`, areia `#F5C451`, ok `#4BD99A`, erro `#FF6B73`. Tema claro definido por tokens; o telão é sempre escuro.
- **Tipografia:** Instrument Serif (títulos, nomes, pontuações, resposta revelada) + Outfit (interface e retângulos em 800, caixa alta).
- **Layout da partida:** 100dvh, sem rolagem. Computador: placar à esquerda, telão ao centro, bancada abaixo do telão. Celular: placar em faixa, telão, bancada fixa embaixo.
- **Linguagem comum:** retângulo escondido = número + barrinha branca; revelado = sublinhado rosa. Card de tema selecionado usa o mesmo sublinhado.
- O Bruno é UX designer e espera acabamento de nível alto (referência: Awwwards). Evitar "esticar tudo com flex"; cada tela precisa de composição.

## 4. Arquitetura do código (index.html)
- `THEMES` (temas e perguntas), `LONG`/`EXTRA` (complementos), `ICONS` (ícones SVG para Chaves, Brasil e Pokémon), `ALIASES` (respostas alternativas).
- Ranking: `getMatches`, `rankTable`, `leaderId`, `saveMatch` (chamado em `finish`), `renderRank`. Tema: `applyTheme`/`toggleTheme` (atributos `data-theme` e `data-mode` no `<html>`). Ouro: `S.gold`, `goldIntro`, `sfx.gold`.
- Estado em `S`; persistência em `store` (prefixo `revela:` no `localStorage`).
- Jogadores: elenco/mesa em `R` (`roster`, `table`), funções `renderPlayers`, `renderRoster`, `addPlayer`, `delPlayer`, `openEmoji`, `openEdit`, `tablePlayers`. `S.players` é montado da mesa em `startGame` (cada jogador carrega `id`). Folha nova: `#o-player`. `toast(msg, {label, fn, ms})` aceita botão de ação.
- Telas: `#s-home`, `#s-players`, `#s-themes`, `#s-game`, `#s-final`, `#s-rank`; folhas: `#o-emoji`, `#o-player`, `#o-menu`, `#o-report`.
- Funções principais: `buildDeck`, `loadQuestion`, `renderBoard`, `renderHUD`, `reveal`, `nextTurn`, `endQuestion`, `finish`, `judge`/`norm`/`lev`, `hintMask`, `fitBoard`, `confetti`, `sfx`.
- Fora do claude.ai, `window.claude` não existe: o código já cai para `localStorage` sozinho.

## 5. Próximos passos aprovados pelo Bruno
Ordem sugerida, das mais simples às que dependem do Supabase.

1. ~~**Jogadores salvos + exclusão.**~~ Feito (ver seção 2).
2. ~~**Animação da vez.**~~ Feito (ver seção 2).
3. ~~**Pergunta de ouro.**~~ Feito.
4. ~~**Chance fora da vez (1 por partida).**~~ Feito como "Eu sei!" (ver seção 2). Não é roubo: cada jogador tem uma chance na partida de responder fora da sua vez; se acertar, ganha menos pontos (definir quanto, ex.: metade). Precisa de UI para "pedir a vez" sem atrapalhar quem está jogando.
5. **Temas novos:** Dinossauros; Nostalgia anos 80; Nostalgia anos 90.
6. **Nível Família.** Para jogar com pais e avós (60 a 70+ anos): brincadeiras antigas, cantores clássicos, craques antigos do futebol, faroeste e outros gêneros de filme antigos, novelas e programas antigos, objetos e costumes de antigamente. No nível Família **não se escolhem temas**: o jogo usa um conjunto próprio de perguntas nostálgicas.
7. **Ranking / histórico.** *Versão local feita (vitórias, partidas, %). Falta: levar para o Supabase.* Recordes, quem mais venceu, maior pontuação numa pergunta. Começa local e depois vai para o Supabase.
8. **Narração.** *Pronta e desligada (`VOZ_ATIVA`).* Ler em voz alta a palavra revelada (Web Speech API, `speechSynthesis`, voz pt-BR).
9. **Responder por fala.** *Pronto e desligado (`VOZ_ATIVA`), com orbe de voz e pedido de número por voz.* Reconhecimento de voz (`SpeechRecognition`, pt-BR) preenchendo a caixa de resposta. Verificar suporte por navegador e manter a digitação como alternativa.

10. ~~**Modo claro e escuro com botão.**~~ Feito (telão claro próprio). Hoje o jogo segue o tema do sistema (tokens prontos em `:root`, `@media (prefers-color-scheme)` e `[data-theme]`). Falta:
    - um botão de alternar (sol/lua) visível na tela inicial e no menu da partida, salvo no `localStorage`;
    - uma revisão visual completa do modo claro em todas as telas (home, jogadores, temas, partida, resultado, final, folhas). Decidir se o telão continua escuro no modo claro (como painel de TV) ou ganha uma versão clara própria;
    - conferir contraste (WCAG AA) nos dois modos.
11. ~~**Roleta antes de cada pergunta (tema + estrelas de dificuldade).**~~ Feito (ver seção 2). Registro original abaixo. Criar um momento de suspense antes da pergunta aparecer:
    - uma roleta/esteira gira entre os temas da partida e desacelera até parar no tema da pergunta sorteada (o sorteio real acontece antes; a roleta só encena o resultado);
    - ao parar, aparece a classificação da pergunta de **1 a 5 estrelas** (5 = resposta muito difícil), com as estrelas acendendo uma a uma;
    - referências visuais em `docs/referencias/`: a esteira horizontal com marcador central (ref. 2) e a roleta/cards com brilho e bordas arredondadas (ref. 1). Adaptar ao DS Phantom, sem copiar;
    - som: tique acelerando enquanto gira, desacelerando até parar, e um acorde ao revelar as estrelas; botão para pular a animação e respeitar `prefers-reduced-motion`;
    - **dados:** cada pergunta precisa de um campo de dificuldade (1 a 5). Proposta: classificar as perguntas existentes com critérios claros (quão conhecida é a resposta para o público geral, quantas palavras precisam ser abertas para ficar óbvia) e revisar com o Bruno por amostragem; perguntas longas não são automaticamente difíceis;
    - **nível Difícil:** como o tema é secreto, a roleta não pode parar no tema. Opções: girar só as estrelas, ou parar em "Tema secreto" e mostrar só a dificuldade;
    - **nível Família:** a roleta pode girar entre as categorias nostálgicas do próprio nível;
    - pensar se a dificuldade afeta a pontuação (ex.: bônus em pergunta de 4 ou 5 estrelas). Ainda não decidido.

12. ~~**Favicon de bola 8 de sinuca.**~~ Feito (PNG para iOS/Android fica para o PWA). Ícone da aba (SVG inline, com versão PNG para iOS/Android depois, junto do PWA). Bola preta com o círculo branco e o "8", no estilo do DS Phantom.
13. ~~**A palavra 8 como mecânica própria.**~~ Feito (ver seção 2). Ideias originais registradas abaixo. Usar o nome do jogo como diferencial. Ideias do Bruno, ainda sem decisão:
    - **Bônus da 8:** quem acertar a resposta depois que a palavra 8 foi revelada ganha +8 pontos (cria tensão: abrir a 8 ajuda a responder, mas também entrega a pista para todos).
    - **8 trancada:** a palavra 8 começa bloqueada e só pode ser aberta depois que todos jogaram a primeira rodada da pergunta.
    - A decidir: o que acontece em perguntas com menos de 8 palavras; se a 8 tem visual próprio no telão (ex.: retângulo com a bola 8); se combina com a pergunta de ouro e o nível Difícil.

Descartados por enquanto: times, aposta.
Ideias para depois: **Modo TV / cada um no seu celular** (telão numa tela, cada jogador entra por código de sala no próprio celular, responde e aperta "Eu sei!" por lá; quem apertar primeiro leva). Precisa de tempo real: usar Supabase Realtime quando o Supabase entrar. Perguntas da casa (criadas pelo grupo).

## 6. Dados e Supabase (a decidir na próxima conversa)
- Ainda não há schema. Entidades prováveis: jogadores, partidas, pontuação por jogador e partida, reportes de pergunta.
- Decidir: login ou grupos anônimos (ex.: código de grupo/família); RLS por grupo; se as perguntas continuam no HTML ou vão para o banco/JSON.
- Desenhar o schema com a skill de boas práticas do Supabase/Postgres antes de criar tabelas.
- O reporte hoje usa o banco do claude.ai; ao migrar, mover para uma tabela de reportes no Supabase.

## 7. Virar app (futuro)
- Primeiro PWA: `manifest.webmanifest`, ícones e service worker para funcionar offline e instalar na tela inicial.
- Depois, se quiser lojas: empacotar com Capacitor (iOS/Android).

## 8. Referências visuais
- `docs/referencias/roleta-ref-1-cards-e-roleta.png`: roleta, cards e brilho (inspiração de acabamento).
- `docs/referencias/roleta-ref-2-esteira-horizontal.png`: esteira horizontal com marcador central (inspiração da roleta de tema).

## 9. Links
- Produção (Vercel): https://revela-ebon.vercel.app · repositório GitHub `br1leal/revela` (deploy automático a cada push na `main`).
- Versão publicada no claude.ai: https://claude.ai/artifact/89ExtJgfxEHBw43bnCGzp9
