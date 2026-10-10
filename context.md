# Contexto — Prancheta de Fluxos

Editor de fluxogramas e mapas mentais em HTML puro, feito para substituir Whimsical/Miro em uso próprio. Arquivo único, sem build, sem dependências externas. Rodou até aqui como artifact do Claude.ai; a partir do Claude Code o alvo é o arquivo local `index.html` em `C:\Users\glauc\Desktop\Parkia\Fluxograma`, que pode ser aberto direto no navegador.

## Decisões já tomadas

- **Um arquivo só.** HTML, CSS e JS no mesmo `index.html`. Nenhuma lib, nenhum bundler, nenhuma chamada de rede além da fonte do Google Fonts (Archivo, IBM Plex Sans, IBM Plex Mono).
- **JS em IIFE**, modo estrito, sem framework. Manipulação direta de DOM.
- **Render híbrido**: nós são `<div>` posicionados em coordenadas de mundo; conexões são `<path>` dentro de um `<svg>` no mesmo container transformado. Isso permite texto editável nativo (`contenteditable`) dentro dos nós.
- **Canvas infinito** via `transform: translate(x,y) scale(k)` no `#world`, com `transform-origin: 0 0`. O fundo pontilhado do `#stage` acompanha pan e zoom pelo `background-position` / `background-size`.
- **Tema claro e escuro** por tokens CSS em `:root`, redefinidos em `@media (prefers-color-scheme: dark)` e em `:root[data-theme="dark"]`. Nenhuma cor literal fora dos tokens. A paleta dos nós tem 8 famílias (`slate, blue, teal, green, amber, orange, rose, violet`), cada uma com três tokens: preenchimento `--c-<cor>-f`, traço `--c-<cor>-s`, texto `--c-<cor>-i`.
- **Persistência** em `localStorage` sob a chave `prancheta.v1`, sempre dentro de `try/catch`. Tema em `prancheta.theme`.

## Modelo de dados

```js
state = {
  title: 'Funil de captação',
  nodes: [{ id, x, y, w, h, text, shape, color, fs, bold, collapsed }],
  edges: [{ id, from:{node, side}, to:{node, side}, label, style, curve, arrow, width, color }]
}
view = { x, y, k }   // pan e zoom
```

- `shape`: `process | round | terminal | decision | data | circle | note | table`
- `tbl` (só em `shape:'table'`): `{ cells:[[{t, c?}]], colW:[pesos], al:['left'|'center'|'right' por coluna], hRow, hCol, head:'strong'|'soft'|'plain', lines:'grid'|'rows'|'cols'|'outer'|'none', zebra, pad:'compact'|'normal'|'wide' }`. `c` é a cor de fundo da célula (uma das 8 famílias).
- `side`: `auto | t | r | b | l` (auto escolhe o lado pelo vetor entre os centros)
- `style`: `solid | dashed | dotted`
- `curve`: `ortho | curved | straight`
- `arrow`: `none | end | both`
- `collapsed`: ramo recolhido (mapa mental)

Seleção fica fora do `state`, em `sel = { nodes:Set, edges:Set }`. Histórico de undo/redo é um array de `JSON.stringify(state)` com ponteiro `hi`, limitado a 80 entradas, alimentado pela função `snapshot()`.

## Funções principais

| Função | Papel |
|---|---|
| `render()` | Recalcula visibilidade, sincroniza todos os nós, redesenha conexões e o inspetor |
| `syncNode(n)` | Aplica posição, tamanho, forma, cor e estado de um nó no DOM |
| `renderEdges()` | Redesenha paths, setas e rótulos de todas as conexões |
| `edgeGeom(e)` | Devolve o `d` do path conforme o tipo de traçado |
| `shapePath(shape,w,h)` | Path SVG de cada formato de nó |
| `computeVisibility()` | Calcula `hiddenSet`, `outMap` e `hidCount` a partir dos nós recolhidos |
| `toggleCollapse(ids)` | Recolhe ou expande ramos |
| `snapshot()` | Grava estado no histórico e no localStorage |
| `renderInspector()` | Monta o painel lateral conforme o que está selecionado |
| `copyPNG()` | Rasteriza o quadro visível em canvas e copia para a área de transferência |
| `toMermaid()` | Serializa o quadro como `flowchart TD` |

Toda a interação de ponteiro passa por um único handler em `#stage` (`pointerdown / pointermove / pointerup`) com uma variável `drag` que carrega o tipo da operação em curso: `pan | move | resize | link | marquee | chain | col`.

Clique duplo é detectado dentro do `pointerdown` (`isDoubleDown` / `onDouble`), não pelo evento `dblclick`: com `setPointerCapture` no stage o navegador não entrega `click` nem `dblclick`. Pelo mesmo motivo, nada no `pointerdown` deve chamar `render()` quando o alvo pode ser o primeiro clique de um duplo, porque reanexar o nó no DOM cancela o clique.

## Tabela

Nó com `shape:'table'`, criado pela ferramenta da barra ou pela tecla `T`. O HTML é um `<table>` dentro de `.tblwrap`; o contorno arredondado continua sendo o `.nshape`, desenhado por cima. Funções: `newTable`, `normTable`, `buildTable`, `syncTable` (reconstrói só quando a assinatura JSON muda), `fitTable` (a altura nunca fica menor que o conteúdo, chamada em todo `render`), `tblOp` (inserir e excluir linha e coluna), `startCellEdit` / `finishCellEdit` / `moveCell`, `paintCells`, `inspTable`, `drawTable` (PNG, lendo geometria e cores calculadas pelo navegador).

Seleção de célula fica em `cellSel = {node, r, c}`, fora do `state`. O primeiro clique seleciona a tabela e o segundo seleciona a célula. Com uma célula selecionada, as setas andam entre células, `Del` limpa a célula e `Esc` volta para a tabela. Editando uma célula, `Tab` vai para a próxima (na última, cria uma linha) e `Shift+Enter` quebra a linha. As colunas são redimensionadas arrastando a divisa entre elas.

O inspetor da tabela tem quantidade de linhas e colunas, cabeçalho no topo e na lateral, estilo do cabeçalho, linhas de grade, linhas alternadas, alinhamento, espaçamento, cor e texto. Com uma célula selecionada, também pinta a célula, a linha ou a coluna, alinha a coluna, insere e exclui linhas e colunas. Para o Mermaid, a tabela vira um nó de processo com as linhas separadas por `|`.

`#insp` segura a reconstrução (`inspHold`) enquanto um botão dele está sendo clicado. Sem isso, o blur de uma edição em curso redesenharia o painel e o clique se perderia.

## Regra de visibilidade (recolher ramo)

Um nó fica oculto quando **todo** caminho até ele passa por um nó recolhido. A implementação faz:

1. BFS a partir dos nós recolhidos, marcando todos os descendentes.
2. BFS a partir das raízes (nós sem aresta de entrada), sem atravessar nós recolhidos.
3. `hidden = descendentes − alcançáveis por caminho visível`.

Nós com caminho alternativo aberto continuam visíveis. Criar filho via Tab ou via arrasto da porta expande o pai automaticamente, para o nó novo não nascer escondido.

## O que já funciona

Criação por clique duplo no fundo, edição de texto inline, arraste com alinhamento à grade de 8px, redimensionamento, seleção por área e por Shift, conexão arrastando a porta lateral (soltar no vazio cria nó já ligado), ferramenta de conectar por cliques, rótulos de conexão, inspetor contextual de nó e de conexão, undo/redo, duplicar, copiar e colar interno, recolher e expandir ramos com contador de ocultos, zoom por Ctrl+scroll e pinça, enquadrar tudo, organizar em coluna, tema claro/escuro, autosave, exportação em JSON, Mermaid e PNG, importação de JSON, atalhos de teclado e painel de ajuda.

Atalhos: `V N C H` ferramentas, `Espaço` arrastar tela, `Tab` nó ligado, `Enter` editar, `E` recolher/expandir, `Del` excluir, `Ctrl+D` duplicar, `Ctrl+Z` / `Ctrl+Shift+Z`, `Ctrl+A`, `Shift+1` enquadrar, setas movem 8px (1px com Shift).

## Limites herdados do artifact

Dentro do visualizador do Claude.ai, download de arquivo e `window.print()` não funcionam, por isso toda saída é por cópia para a área de transferência. Rodando como arquivo local no navegador essa restrição deixa de existir: dá para trocar as exportações por download real de `.json`, `.svg` e `.png`, e habilitar impressão.

## Ideias em aberto

- Download real dos arquivos e exportação em SVG vetorial, agora que não há restrição de sandbox
- Modo mapa mental com layout radial automático a partir de um nó central
- Templates de funil prontos para reaproveitar em projetos de clientes
- Agrupamento de nós e camadas
- Importação de Mermaid, não só exportação
- Guias de alinhamento durante o arraste
- Múltiplos quadros no mesmo arquivo, com troca por abas

## Preferências de trabalho

Respostas e textos em português do Brasil, diretos, sem floreio. Nos textos de interface: nada de travessão, nada de estrutura "não é X, é Y", nada de listas de três itens como padrão decorativo. Comentários de código em português e só onde a intenção não é óbvia pelo nome da função.
