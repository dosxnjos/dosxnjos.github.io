# CEU-3D.md — a view `3D` do board (o céu de cards)

Como funciona, como ler, e onde mexer sem quebrar. Nasceu em 17/08/2026 do
roadmap [`2026-08-17-melhoria-secao-3d-ceu-de-cards.md`](../roadmap/2026-08-17-melhoria-secao-3d-ceu-de-cards.md)
(lá estão as decisões e o que foi **descartado**, com motivo).

**Para que serve:** ler de relance, sem texto, o que está em voo, o que espera
por você, o que está com terceiro e o que está apodrecendo. Não é para operar:
**o céu é radar, o kanban é cockpit** — clique num corpo devolve o card ao board.

## Arquivos

| arquivo | papel |
| --- | --- |
| `ceu.html` | a cena inteira (canvas, mapeamento, loop, interação). Roda dentro de um `<iframe>` |
| `vendor/three.module.min.js` + `three.core.min.js` | three.js r185 vendorizado (MIT). Offline, sem CDN, como todo o resto da central |
| `index.html` | o botão `3D` no `#viewSeg`, `montarCeu()`/`desmontarCeu()`, e o receptor do clique vindo do céu |
| ~~`build-app.ps1`~~ | desmontado em 28/08/2026 junto com o `Central.exe` (`README.md` § app no navegador); o servidor serve `ceu.html` e `vendor/` direto do disco, sem build |

## Como ler o céu (o contrato)

Regra de ouro: **cada propriedade visual carrega uma variável de dado, e só
uma.** Se for preciso empilhar dois significados no mesmo canal (dois sentidos
no brilho, por exemplo), a metáfora estourou — melhor deixar de fora.

| o que se vê | o que é (campo real) |
| --- | --- |
| **estrela** colorida | uma área; a cor vem de `areas[].cor`, a mesma do board |
| estrela **fraca** | área sem nenhum card ativo (`corpo`, `saude`, `gatos` hoje) — presença sem trabalho é informação |
| **centro** brilhante | você. O anel e as radiais são só estrutura, não carregam dado |
| órbita **interna**, com cauda | `status: agora` |
| órbita **média** | `status: proximo` |
| órbita **externa**, apagada | `status: depois` |
| corpo **no meio do caminho** entre a estrela e o centro | `status: decisao` — ponto de Lagrange: não cai para lado nenhum até você empurrar |
| corpo **saindo do sistema**, sobre um trilho | `status: esperando` (terceiro). A distância percorrida é a idade da espera (21 dias = fim do trilho) |
| **estrela no fundo**, longe | `status: feito` nos últimos 30 dias; o setor angular diz **quando** fechou |
| **poeira** distante | feito há mais de 30 dias: só massa, sem identidade |
| **mais vermelho e mais longe** | idade (`criado_em`): 45 dias chega ao vermelho cheio |
| **pulso rápido** | `urgente` |
| **corpo maior** | `importante` |
| corpo **difuso, sem núcleo** | `tipo: ideia` |
| **anéis** no corpo | uma rodada de execução gasta por anel (`modelos`) — retrabalho grita sem texto |
| **pilar âmbar** subindo do corpo + halo | a sessão do card está em `pede_permissao`/`pendurada`: **te espera** |
| corpo âmbar **orbitando o centro** | sessão que te espera **sem card vinculado** (as "sessões soltas" do board). Clique abre a aba |
| pulso médio, sem âmbar | sessão `rodando` naquele card |

Teclas: **`f`** tela cheia · **`l`** esconde a legenda e os contadores ·
arrastar gira · roda aproxima (e desliga o auto-enquadramento) · clique abre no
board.

**Hover: toda orbe responde.** O raycast usa o `mapaHover` (`Object3D` → info),
não a lista de corpos — porque só com os corpos ficavam mudas justamente as
maiores: a **estrela de área** (mostra `área · N ativos · M feitos`), os **feitos**
do céu de fundo (`Points`, resolvidos pelo índice do vértice, com
`ray.params.Points.threshold`) e os **halos**, que ficam por cima do corpo e
engoliam o hover dele (agora apontam para o mesmo dono). Sem hover, de
propósito: a poeira (feito com mais de 30 dias **é** massa sem identidade), o
cinturão e as radiais (estrutura, não dado). Clique só age em card e sessão.

## Determinismo: o que faz disto um radar e não um protetor de tela

Toda posição sai de `hash32(card.id)` (FNV-1a) — **nunca** `Math.random()`,
**nunca** simulação física, **nunca** o índice do array (a ordem muda a cada
gravação). O mesmo estado desenha o mesmo céu, e é isso que permite reconhecer
"aquele ponto às 4 horas da estrela ciano" sem ler nada. Em duas semanas, a
**ausência** de um corpo já é informação.

Duas consequências para quem for mexer:

1. **O céu gira em bloco** (uma volta a cada 20 min), como o céu real. Se cada
   órbita girasse na sua velocidade (kepleriano), a constelação se deformaria a
   cada minuto e o reconhecimento morreria. Movimento local permitido: pulso de
   brilho e respiração — nada que mude a posição relativa entre dois corpos.
2. **Estado transitório não move corpo.** O eclipse de uma sessão que te espera
   destaca (halo + pilar), mas não reposiciona: mover por estado passageiro
   destruiria o mapa mental.

Prova medida em 17/08/2026: duas cargas seguidas do mesmo board deram **59 de 63
corpos com posição idêntica**; os 4 restantes são `status: esperando`, cuja
posição depende da idade da espera (dado contínuo) e variaram na 4ª casa
decimal. Reproduzir: `window.__ceu.posicoes()` nas duas cargas e comparar.

## Não roda escondido (requisito explícito do Gabriel)

Duas camadas:

1. **Fora da view, o iframe não existe.** `montarCeu()` cria; `desmontarCeu()`
   faz `innerHTML=""` e leva embora o contexto WebGL, o `requestAnimationFrame`
   e os `setInterval`. É garantia estrutural, não disciplina de programador.
2. **Dentro do céu**, a política de fps — e, no limite, o **sono**:

| situação | como é detectada | o que acontece |
| --- | --- | --- |
| aba oculta / janela minimizada | `document.visibilityState` | **dorme** |
| canvas fora do viewport | `IntersectionObserver` (no `#palco`) | **dorme** |
| janela não maximizada | `innerWidth < 85%` de `screen.availWidth` **ou** `innerHeight < 70%` de `availHeight` (medido em `window.top`, porque o iframe é menor que a janela) | **dorme** + aviso "maximize a janela" |
| visível, maximizada, **sem** foco | `window.top.document.hasFocus()` | **~8 fps** — o suficiente para o eclipse chamar do canto do olho |
| visível, maximizada, com foco | — | **30 fps** (cap fixo, nunca `rAF` livre) |

**Dormir é mais forte que "0 fps"** (pedido do Gabriel em 17/08/2026: *"ao invés
de 0 fps, na verdade pode desligar o uso de GPU, sumindo com o canvas se for
necessário"*). Parar só o loop deixaria o contexto WebGL vivo, segurando memória
de vídeo à toa. `dormir()` faz, nesta ordem: esconde a dica, descarta a cena
(`limparGrupo` + `corpos`/`posEstrelas` zerados), destrói fundo e núcleo,
`dispose()` em todas as texturas, e então `renderer.dispose()` +
`forceContextLoss()` + **remove o canvas do DOM**. `acordar()` recria canvas,
renderer, fundo e reconstrói a cena a partir do `stateAtual` que ficou em
memória — poucos ms com 84 cards.

Por isso o canvas é **descartável** e o `#palco` é quem permanece: os listeners
de mouse e o alvo do `IntersectionObserver` vivem nele, senão sumiriam junto com
o canvas. Enquanto dorme, os polls não constroem nada — só atualizam
`stateAtual`, para acordar já com o dado certo.

Medido (`window.__ceu`): acordado → `canvasNoDOM: true, temContexto: true,
corpos: 62`; dormindo → `canvasNoDOM: false, temContexto: false, corpos: 0`; e
volta inteiro ao acordar.

⚠️ **Limite real da plataforma:** a web **não sabe** dizer que a janela está
*coberta* por outra. Central maximizada atrás do Power BI continua "visible" e
roda nos 8 fps do caso sem foco. Só um host nativo (pywebview) poderia informar
— pertence à rota de janela/wallpaper dedicada, que não existe ainda.

Orçamento de GPU: `InstancedMesh` não é usado (63 sprites não justificam), glow é
**sprite aditivo com textura gerada em runtime** — nunca post-processing de
bloom, que é o caminho óbvio para "luz bonita" e o mais caro em GPU contínua.
Sem sombra, sem antialias, `powerPreference: "low-power"`, pixel ratio ≤ 1.5.

## Como o dado chega

- `GET /api/state` a cada 15 s → cards e áreas.
- `GET /api/sessoes` a cada 10 s → estado das sessões (eclipse e cauda).
- Nenhum poll roda quando `alvoFps()` é 0.

⚠️ **A detecção de mudança NÃO usa `atualizado_em`.** Ver
[`ARMADILHAS.md#56`](../ARMADILHAS.md): existe quem grave no `pendencias.json`
sem atualizar o carimbo, e quem confia nele fica cego. O céu compara uma
**assinatura do que desenha** (id, status, área, tipo, urgente, importante,
nº de modelos, datas) — ~7 KB de string a cada 15 s, custo irrelevante.

Ainda em 17/08/2026 a raiz foi fechada do outro lado: o `ETag` de `/api/state`
virou **hash do conteúdo** (`c-…`), e não mais o carimbo. Isso não muda nada
aqui — o céu não manda `If-None-Match`, faz `cache: "no-store"` e decide pela
assinatura própria. Fica de propósito: são duas defesas independentes, e a
daqui é a que sabe **o que o desenho usa** (mudar um campo que o céu não
desenha não deve custar uma reconstrução da cena).

## Mexendo aqui

- **Escala é calibrada por screenshot, não por intuição.** As constantes no topo
  do `<script>` (`RAIO_AREA`, `FAIXA`, tamanhos de sprite, `ALVO_NDC`) foram
  ajustadas contra o board real olhando o resultado: corpo tem que dar ~12 px
  para ser distinguível, e a órbita interna tem que sair do glow da estrela,
  senão o aglomerado vira um borrão. Mudou uma delas? Abra e olhe.
- **Enquadramento é medido por projeção**, não por fórmula: `enquadrar()` projeta
  cada corpo em NDC e itera o raio da câmera até a cena ocupar `ALVO_NDC` da
  tela. A fórmula analítica (raio/tan(fov)) ignora a perspectiva e cortava o
  sistema que está na frente.
- **Raio de órbita é FAIXA, não valor fixo** (`FAIXA[status] = [min, max]`, com o
  ponto dentro da faixa vindo do hash): com raio fixo, 14 cards `proximo` da
  mesma área formavam um anel apertado que o blending fundia numa mancha.
- **Cor nova, nunca.** A paleta vem de `areas[].cor`.
- ⚠️ **Cor de área se compara por MATIZ, não por hex** (aprendido em 17/08/2026).
  Duas cores com hex diferente podem ser a mesma coisa a olho nu: `saúde`
  `#e6a94e` e `casa` `#d0a24e` estavam a **2,8°** de matiz, e as 4 áreas `sof-*`
  eram gêmeas das do Gabriel (`sô dinheiro` a **1,2°** de `casa`). No kanban isso
  passa, porque o nome da área está escrito ao lado e a cor é redundante. **Aqui
  não há texto**: a cor é o único canal, e gêmeas viram uma constelação só.
  Régua: nenhum par abaixo de ~20° de matiz. As 11 áreas de hoje estão entre
  24,7° e 32,4°. Medir assim:

  ```python
  # menor distância de matiz entre qualquer par de áreas
  import json, colorsys
  def hue(h): return colorsys.rgb_to_hls(*[int(h[i:i+2],16)/255 for i in (1,3,5)])[0]*360
  a = json.load(open("pendencias.json", encoding="utf-8"))["areas"]
  print(min((min(abs(hue(x["cor"])-hue(y["cor"])), 360-abs(hue(x["cor"])-hue(y["cor"]))),
             x["nome"], y["nome"]) for i,x in enumerate(a) for y in a[i+1:]))
  ```

  Ao criar área nova, o board já escolhe a **primeira cor livre** da paleta de 7
  (era `cores[length % 6]`, que repetia a partir da 7ª e colidia sempre que uma
  área fosse apagada). A paleta é o conjunto canônico: se uma cor de área mudar
  no dado, mudar na paleta também, senão a próxima área nova reintroduz a
  colisão.
- **`window.__ceu`** é o gancho de diagnóstico (só leitura): `n`, `fps`,
  `camera`, `alvos`, `dormindo`, `temContexto`, `posicoes()`, `amostraHover()`.
  É por ele que se testa determinismo, política de fps, sono e hover sem mira
  manual.
- ⚠️ **Nunca deixar o `index.html` aberto em automação** (Playwright, aba
  esquecida) enquanto outra coisa mexe no estado: o board **grava**
  (`save()` → `PUT /api/state`) e, com o estado velho em memória, apaga o que
  entrou depois — foi assim que 28 cards morreram em 17/08
  ([`ARMADILHAS.md#56b`](../ARMADILHAS.md)). Para testar o céu, abrir
  **`ceu.html` direto**: ele só faz `GET`.
- **Mudou `ceu.html`, `vendor/` ou `index.html`? Nenhum build.** O `Central.exe` e o
  `build-app.ps1` foram desmontados em 28/08/2026 (`README.md` § app no navegador);
  o servidor serve os três do disco, e relança sozinho quando o código muda
  (`config.servidor.auto_relancar`). `ARMADILHAS.md#17`/`#18` ficam como histórico.
- O botão `3D` só aparece em **serverMode**: por `file://` o board tem de seguir
  single-file, sem recurso externo (regra do [`CLAUDE.md`](../CLAUDE.md)).
