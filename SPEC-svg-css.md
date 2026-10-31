# Especificação: cores de traço por classe CSS na importação SVG

## Objetivo

Permitir que desenhos SVG simples importados para o CADCraft preservem cores de traço declaradas por classes CSS. Exemplo: `<style>.parede { stroke: #336699 }</style><line class="parede" .../>` produz uma entidade com essa cor. Esta contribuição implementa um subconjunto de FF14; não encerra o gap inteiro.

## Escopo proposto e critérios de aceite

- Aceitar regras locais `.nome { stroke: #RGB; }` e `.nome { stroke: #RRGGBB; }`, conforme a gramática abaixo, inclusive múltiplas classes por elemento, espaços e comentários CSS; cores hexadecimais são insensíveis a maiúsculas e minúsculas.
- Aplicar a cor efetiva a todas as entidades geradas pelas formas e caminhos já suportados, inclusive sob transformações e em camadas Inkscape, preservando a geometria e as camadas existentes.
- Aceitar as mesmas cores no atributo `stroke` e no atributo `style` diretamente na forma. Para declarações normais suportadas na própria forma, usar precedência: atributo de apresentação < regra de classe < estilo inline; entre regras de classe equivalentes, a última declaração na ordem do documento prevalece. Classes na ordem do atributo não alteram essa prioridade.
- Encontrar folhas internas inclusive em `<defs>` e depois da forma no documento, respeitando os critérios de elegibilidade abaixo; nenhum acesso à rede ou ao sistema de arquivos para folhas externas.
- Aplicar somente declarações suportadas diretamente na forma. Descartar declarações fora do subconjunto antes de resolver a precedência das restantes, com aviso de importação parcial. Este resultado é uma aproximação deliberada e não deve ser apresentado como o resultado da cascata CSS completa. Os exemplos abaixo fixam o comportamento esperado.
- Falhas de sintaxe e entradas hostis não devem provocar pânico, processamento desproporcional ou alocações ilimitadas; os orçamentos abaixo limitam dados retidos e trabalho de correspondência.
- Documentar o subconjunto, suas limitações e o estado parcial de FF14. Nenhuma mudança de CI nem ativos ou exemplos proprietários.

## Gramática e folhas elegíveis

- Um seletor aceito contém exatamente `.` seguido de um identificador ASCII que corresponda a `[A-Za-z_][A-Za-z0-9_-]*`. Nomes de classe são sensíveis a maiúsculas e minúsculas. Aceitar espaços CSS antes e depois do seletor e comentários `/* ... */` entre tokens, sem concatenar identificadores separados por comentários.
- O atributo `class` pode conter vários identificadores separados por espaços ASCII. Identificadores fora da gramática são ignorados com aviso; os demais continuam elegíveis.
- Listas de seletores (`.a, .b`), seletores compostos (`.a.b`, `line.a`), descendentes, pseudoclasses (`.a:hover`), escapes e identificadores Unicode ficam fora do subconjunto. Descartar a regra inteira, sem interpretar parcialmente seu seletor como uma classe simples.
- Nomes de propriedade são insensíveis a maiúsculas e minúsculas. Aceitar somente `stroke` com três ou seis dígitos hexadecimais, permitindo espaço em torno de `:` e do valor, ponto e vírgula final opcional e declarações repetidas. Rejeitar o valor inteiro se houver tokens adicionais, incluindo `!important`; nunca aproveitar apenas seu prefixo hexadecimal.
- Uma folha interna é elegível se `type` estiver ausente, vazio ou for `text/css` (comparação ASCII sem distinguir maiúsculas, removendo espaços externos), e `media` estiver ausente, vazio ou for `all`. Folhas com outros valores são ignoradas por inteiro com aviso. Não avaliar condições de mídia nem assumir um ambiente de tela ou impressão.
- Dentro de folhas elegíveis, aceitar apenas regras de classe no nível superior. Ignorar cada regra iniciada por `@`, incluindo seu bloco inteiro ou seu conteúdo até `;`, com aviso. Nunca aplicar regras de dentro de `@media`, `@supports` ou `@layer`.
- Ler texto XML e CDATA de `<style>`. Comentários CSS não são regras. Em CSS malformado, descartar a menor unidade que possa ser delimitada com segurança; se não houver fechamento confiável de comentário ou bloco, descartar o restante da folha e emitir aviso. Erros de XML continuam seguindo o tratamento do leitor XML existente.

## Resultado e avisos para suporte parcial

Resolver primeiro as declarações aceitas na própria forma, usando a precedência descrita acima. Sem nenhuma delas, manter `Common` com a cor padrão da camada. A herança de `stroke` de contêineres não é aplicada; uma declaração de traço em contêiner gera aviso de suporte parcial, mesmo quando hexadecimal.

Nos exemplos, a forma usa `class="a"`, exceto quando indicado. `Cor da camada` significa o comportamento anterior, sem atribuir uma cor explícita à entidade.

| Entrada | Resultado importado | Aviso |
|---|---|---|
| `.a { stroke: #f00; stroke: #0f0; }` | Verde `#00ff00` | Não |
| `.a { stroke: #f00; stroke: blue; }` | Vermelho `#ff0000`; `blue` é descartado | Importação parcial |
| `.a { stroke: #f00; stroke: none; }` | Vermelho `#ff0000`; a entidade continua presente | Importação parcial |
| `.a { stroke: #f00 !important; }` | Cor da camada | Importação parcial |
| `.a { stroke: #f00 !important; }` e `style="stroke: #0f0"` na forma | Verde `#00ff00` | Importação parcial |
| `<g class="a"><line .../></g>` com `.a { stroke: #f00; }` | Cor da camada na linha | Importação parcial por herança omitida |
| `.a:hover { stroke: #f00; }` | Cor da camada | Importação parcial; não aplicar como `.a` |
| `<style media="print">.a { stroke: #f00; }</style>` | Cor da camada | Folha ignorada |
| `@media screen { .a { stroke: #f00; } } .a { stroke: #0f0; }` | Verde `#00ff00` | Bloco condicional ignorado |
| `.a { fill: #f00; stroke: #0f0; }` | Traço verde; preenchimento segue o comportamento anterior | Importação parcial |

Usar `Drawing.read_notes` para informar que estilos foram aplicados parcialmente ou ignorados, com categorias como declaração, seletor, folha condicional e herança. Basta um aviso agregado por categoria no documento; não é necessário listar cada ocorrência, copiar CSS de entrada nem determinar se uma regra não suportada alcançaria alguma forma. Folhas inteiramente suportadas não devem produzir o aviso antigo de que suas classes não foram aplicadas.

## Orçamentos de recursos

Valores iniciais do contrato, cumulativos por documento salvo indicação contrária. O orçamento de bytes inclui todas as folhas, mesmo as inelegíveis. Regras, declarações e profundidade são contadas somente nas folhas elegíveis (e, para declarações, nos estilos inline), incluindo estruturas dentro de blocos `@` ignorados nessas folhas. Não interpretar CSS de uma folha inelegível apenas para contar suas construções; ela continua sujeita ao limite de bytes.

| Recurso | Limite | O que contar |
|---|---|---|
| Texto de estilos | 1 MiB (1.048.576 bytes UTF-8) | Texto/CDATA de todas as folhas e valores de todos os atributos `style` |
| Regras CSS | 4.096 | Regras qualificadas e regras `@` de folhas elegíveis, incluindo regras em blocos ignorados |
| Declarações | 32.768 | Declarações examinadas em folhas elegíveis, incluindo seus blocos ignorados, e estilos inline |
| Profundidade de blocos CSS | 64 | Blocos aninhados de folhas elegíveis, inclusive os ignorados |
| Tokens de classe | 64 por elemento e 1.000.000 por documento | Tokens encontrados em atributos `class`, mesmo fora da gramática |
| Comparações de classes | 2.000.000 | Cada comparação entre token de classe do elemento e seletor candidato, caso a estratégia faça correspondência por comparações |

Estourar qualquer limite deve interromper a importação com `IoError::Format`, identificando o recurso excedido; não retornar silenciosamente um desenho importado só em parte. Verificar orçamentos antes de alocações ou processamento que os excedam, com aritmética verificada. Uma estratégia indexada que evite comparar todas as regras por elemento é preferível; seus acessos continuam sujeitos aos limites de regras e tokens. Manter os limites já existentes de geometria, profundidade de grupos e descompressão SVGZ.

## Contexto técnico

- Rust; importação em `crates/io/src/svg_read.rs`, com `quick_xml`, entidades de `cadcraft-doc` e cores de `cadcraft-color`.
- O parser atual ignora `<style>`, inclusive sob `<defs>`, e cria `Common` com cor padrão em `Parser::add`. Um levantamento separado das folhas antes da construção das entidades é uma opção a avaliar, respeitando limites de recursos; o comportamento final é o contrato, não essa estratégia específica.
- Regras normais de precedência e a posição dos atributos de apresentação devem seguir as especificações [SVG 2](https://www.w3.org/TR/SVG2/styling.html#PresentationAttributes) e [CSS Cascading](https://www.w3.org/TR/css-cascade-5/). O suporte proposto é deliberadamente parcial.
- A maior parte dos testes de importação SVG fica em `crates/io/src/tests.rs`; `crates/io/src/svg_read.rs` também contém teste local de parsing de arcos. Testes de integração relacionados estão em `crates/io/tests/it/`. Acrescentar casos de comportamento junto aos testes de importação existentes; testes de detalhes privados podem ficar junto ao parser. Documentação em `docs/gaps.md` e `docs/file-format-parity.md`.

## Estilo e segurança

Seguir padrões existentes do projeto: `Result` e `?`, sem pânicos em produção, sem `unsafe`, limites para dados de entrada e sem dependência de UI. Por exemplo, o importador já usa `checked_sub` para limitar vértices gerados antes de adicionar uma entidade. Manter compatibilidade com WASM e não adicionar uma dependência sem verificar licença, tamanho e compatibilidade.

## Verificação

- Testes focados com `cargo test -p cadcraft-io svg` e testes adicionais por nome quando necessário.
- Cobrir formas diferentes, declaração anterior/posterior, `<defs>`, CDATA, classes múltiplas, precedência, repetição, `#RGB`/`#RRGGBB`, arquivos sem CSS e transformações. Usar a tabela de resultados como casos de aceitação, incluindo avisos e ausência de avisos quando cabível.
- Cobrir tipos de folha, atributos `media`, regras `@` aninhadas e seletores que compartilham prefixos com classes válidas. Testar comentários e blocos sem fechamento para garantir término e recuperação conforme o contrato.
- Para cada orçamento, testar exatamente o limite e o primeiro valor acima dele, isolando os demais limites. Entradas acima do limite devem retornar erro sem pânico; os casos de fronteira devem comprovar que o limite é inclusivo. Verificar a estratégia de correspondência escolhida para evitar trabalho proporcional a regras × formas sem orçamento.
- Verificar visualmente um SVG sintético importado por meio do CLI ou da aplicação e conferir a cor das entidades e os avisos.
- Antes de um commit, executar `cargo xtask ci` (inclui formatação, clippy, testes, assets, camadas e WASM), conforme as instruções do repositório.

## Limites

- Sempre: preservar dados e avisar sobre recursos ignorados; manter o escopo verificável; atualizar o gap sem afirmar suporte integral a CSS.
- Nunca: copiar código ou recursos proprietários; fazer rede para resolver CSS; modificar fluxos de CI como parte desta feature.
- Fora de escopo inicial: suporte completo a CSS/SVG, preenchimentos, seletores complexos, folhas externas e outras partes de FF14.

## Pendência técnica para o plano

Caso a implementação segura do subconjunto exija uma biblioteca CSS, verificar as decisões de licença, tamanho e WASM antes de escolhê-la; `plan/adr/0001-geometry-libraries.md` não está disponível neste workspace. O plano deve justificar a estratégia de parsing e correspondência escolhida sem ampliar o subconjunto aprovado.
