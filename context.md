# Contexto — Prancheta de Fluxos

Editor de fluxogramas e mapas mentais em HTML puro, feito para substituir Whimsical/Miro em uso próprio. Arquivo único, sem build, sem dependências externas. Rodou até aqui como artifact do Claude.ai; a partir do Claude Code o alvo é o arquivo local `index.html` em `C:\Users\glauc\Desktop\Parkia\Fluxograma`, que pode ser aberto direto no navegador.

## Decisões já tomadas

- **Um arquivo só.** HTML, CSS e JS no mesmo `index.html`. Nenhuma lib, nenhum bundler, nenhuma chamada de rede além da fonte do Google Fonts (Archivo, IBM Plex Sans, IBM Plex Mono).
- **Fonte padrão: Bahnschrift** (do Windows) na interface e nos nós, com Archivo e IBM Plex Sans de reserva em outros sistemas. O PNG usa a mesma pilha via `bodyFont()`.
- **JS em IIFE**, modo estrito, sem framework. Manipulação direta de DOM.
- **Render híbrido**: nós são `<div>` posicionados em coordenadas de mundo; conexões são `<path>` dentro de um `<svg>` no mesmo container transformado. Isso permite texto editável nativo (`contenteditable`) dentro dos nós.
- **Canvas infinito** via `transform: translate(x,y) scale(k)` no `#world`, com `transform-origin: 0 0`. O fundo pontilhado do `#stage` acompanha pan e zoom pelo `background-position` / `background-size`.
- **Tema claro e escuro** por tokens CSS em `:root`, redefinidos em `@media (prefers-color-scheme: dark)` e em `:root[data-theme="dark"]`. Nenhuma cor literal fora dos tokens. A paleta dos nós tem 17 famílias (`slate, ink, gray, blue, indigo, cyan, teal, green, lime, yellow, amber, orange, brown, red, rose, magenta, violet`), com nomes em português em `COLOR_NAMES`, cada uma com três tokens: preenchimento `--c-<cor>-f`, traço `--c-<cor>-s`, texto `--c-<cor>-i`.
- **Vários quadros** em `localStorage`, sempre dentro de `try/catch`: o índice fica em `prancheta.index` (`[{id,title,updated}]`) e cada quadro em `prancheta.q.<id>` (`{state,view}`). O quadro aberto vem do endereço (`#q=<id>`), então cada aba do navegador edita um quadro. Se o mesmo quadro estiver em duas abas, o evento `storage` faz a outra adotar a versão nova (`applyRemote`), esperando terminar edição ou arraste. A chave antiga `prancheta.v1` é migrada uma vez (`migrateOld`). Tema em `prancheta.theme`, painel em `prancheta.insp`. O armazenamento é por endereço: os quadros do arquivo local e os do site na Vercel são separados; para levar de um para o outro, use Baixar e Abrir arquivo .json na tela Quadros.

## Modelo de dados

```js
state = {
  title: 'Funil de captação',
  nodes: [{ id, x, y, w, h, text, shape, color, fs, bold, collapsed }],
  edges: [{ id, from:{node, side}, to:{node, side}, label, style, curve, arrow, width, color }]
}
view = { x, y, k }   // pan e zoom
```

- `shape`: `process | round | terminal | decision | data | circle | note | sticky | tag | star | text | image | lane | table`
- Estilo do nó: `fill:'solid'` (fundo `-i`, texto `-f`) ou `fill:'mid'` (fundo `-s`, texto `--n-on`: branco, ou `-i` nas famílias claras), `bw` espessura da borda (0 a 4, padrão 1.5), `sc` cor da borda independente do fundo, `al` alinhamento do texto. `SHAPE_DEFAULTS` define tamanho e estilo com que cada formato nasce. O espaçamento interno do texto é em `em`, então acompanha o tamanho da fonte.
- `image`: `img` (dataURL reduzido a 1400 px), `fit:'cover'|'contain'`, e `text` como descrição, que aparece no lugar quando não há arquivo. Nasce pela ferramenta da barra (`I`) ou colando com Ctrl+V.
- `lane` (raia): faixa de fundo com título em pé numa barra lateral (`barW`, padrão 56 px; `bar: strong | mid | none`). Raias sem barra servem de subdivisão dentro de uma raia maior. Fica atrás das conexões, só é clicável pela barra e, ao ser arrastada, leva junto os nós que estão inteiros dentro dela.
- `tbl` (só em `shape:'table'`): `{ cells:[[{t, c?, solid?, v?, rs?, cs?}]], colW:[pesos], rowH:[px|null], al:['left'|'center'|'right' por coluna], hRow, hCol, head:'strong'|'soft'|'plain', lines:'grid'|'rows'|'cols'|'outer'|'none', zebra, pad:'compact'|'normal'|'wide' }`. Na célula, `c` é a cor de fundo (uma das 17 famílias), `solid` usa o tom forte, `v` põe o texto em pé (90°), `rs`/`cs` são as mesclas (rowspan/colspan).
- `side`: `auto | t | r | b | l` (auto escolhe o lado pelo vetor entre os centros); `t` opcional = posição ao longo do lado (0 a 1, padrão 0.5)
- `pts`: rota manual da conexão, lista de pontos de dobra `[[x,y],...]`. `orthoFix` mantém os trechos retos mesmo se um nó se mover; o inspetor tem "Rota automática" para descartar a rota.
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

Seleção de célula fica em `cellSel = {node, r, c, mode}`, fora do `state`, com `mode` `cell | row | col`. As barrinhas acima das colunas e à esquerda das linhas (`.tsel`) selecionam a coluna ou linha inteira; as divisas (`.colh`, `.rowh`) mudam largura e altura. Linha e coluna pegam só as células que começam nelas, então uma mescla que atravessa não muda de cor junto. `coverMap` resolve as mesclas; `spanShift` as ajusta ao inserir ou excluir linhas e colunas. O primeiro clique seleciona a tabela e o segundo seleciona a célula. Com uma célula selecionada, as setas andam entre células, `Del` limpa a célula e `Esc` volta para a tabela. Editando uma célula, `Tab` vai para a próxima (na última, cria uma linha) e `Shift+Enter` quebra a linha. As colunas são redimensionadas arrastando a divisa entre elas.

O inspetor da tabela tem quantidade de linhas e colunas, cabeçalho no topo e na lateral, estilo do cabeçalho, linhas de grade, linhas alternadas, alinhamento, espaçamento, cor e texto. Com uma célula selecionada, também pinta a célula, a linha ou a coluna, alinha a coluna, insere e exclui linhas e colunas. Para o Mermaid, a tabela vira um nó de processo com as linhas separadas por `|`.

Com uma célula, linha ou coluna selecionada, aparece a barra flutuante `#cellbar` logo acima da seleção, com cores suaves, cores fortes e direção do texto. Ela é montada uma vez (`initCellBar`) e só reposicionada em `updateCellBar`, chamada no `render` e no `applyView`. No inspetor da tabela, a parte da seleção sobe para o topo.

O painel lateral (`#insp`) é recolhível pelo botão do canto ou pela tecla `P`, e o estado fica em `prancheta.insp`. O conteúdo é montado em `#inspbody`.

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

## Preferências de trabalho

Respostas e textos em português do Brasil, diretos, sem floreio. Nos textos de interface: nada de travessão, nada de estrutura "não é X, é Y", nada de listas de três itens como padrão decorativo. Comentários de código em português e só onde a intenção não é óbvia pelo nome da função.

## Mapa de Operações (PDF do treinamento)

`quadros/mapa-de-operacoes-mkt-vendas.json` é a reprodução da página 1 do PDF "[Bônus Treinamento CRM] [Modelo] Mapa de Operações", sem o título grande e com a foto trocada por uma imagem com descrição. Foi gerado extraindo a geometria vetorial do PDF com pdf.js (escala 1 pt = 10 px). A pasta `quadros/` fica fora do Git por ser material do treinamento. Abre pela tela Quadros → Abrir arquivo .json.
