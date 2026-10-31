# Plano de implementação: cores de traço por classe CSS no SVG

## Estado e referência

Especificação aprovada pelo usuário: [SPEC-svg-css.md](../SPEC-svg-css.md). Este documento detalha a implementação; o acompanhamento das tarefas fica em [todo.md](todo.md). Não havia plano anterior em `tasks/`.

## Resultado esperado

O importador preserva cores hexadecimais de traço declaradas em classes simples, atributos `stroke` e estilos inline. A precedência e os fallbacks são os da especificação, inclusive a aproximação documentada para declarações fora do subconjunto. FF14 continua aberto para os demais recursos.

## Decisões técnicas

1. Manter `cadcraft_io::read` e o fluxo de OPEN existentes. A mudança pertence ao leitor SVG em L3; não requer comando novo, alteração de UI nem dependência nova.
2. Manter a lógica de CSS privada, preferencialmente em `crates/io/src/svg_read/css.rs`, declarado por `svg_read.rs`. Separar apenas o scanner, as regras de cor e seus orçamentos; evitar uma infraestrutura de CSS geral.
3. Fazer uma passagem preliminar pelo texto XML já descomprimido e validado como UTF-8. Coletar folhas internas em ordem de documento, inclusive em `<defs>`, independentemente de sua posição em relação à geometria. Uma segunda passagem usa o parser geométrico existente. Não construir uma árvore DOM ou duplicar o documento.
4. Na primeira passagem, verificar os orçamentos de texto de estilos e tokens de classe de todos os elementos, inclusive subárvores que o leitor geométrico ignora. Contar e validar atributos `style` uma única vez; decidir uma representação emprestada ou processamento incremental que não armazene estilos inline de todos os elementos. A segunda passagem não deve cobrar novamente o mesmo conteúdo. Verificar o comprimento antes de acumular texto ou normalizar/copiar valores; usar a API existente de XML de modo a preservar seus erros.
5. Usar um scanner que avance sobre o texto de CSS de forma limitada. Reconhecer comentários, strings com escapes e delimitadores para não confundir `;` ou `{` dentro de conteúdo não suportado com estrutura válida. Processar regras aceitas de nível superior; consumir blocos `@` inteiros sem aplicar regras internas, mantendo os contadores da especificação. A profundidade máxima de 64 deve ser controlada explicitamente, sem recursão não limitada. Recuperar erros apenas quando houver um delimitador confiável.
6. Indexar classes por nome em `BTreeMap`, já usado no importador. Reter, por classe, a última cor suportada e sua ordem na folha/documento. Resolver várias classes pelo maior índice de ordem, independentemente da posição dos tokens em `class`. Seletores e propriedades não suportados geram avisos, sem apagar uma declaração aceita anterior, conforme a tabela da especificação. O índice limita a busca a tokens do elemento e evita varrer todas as regras para cada forma.
7. Resolver a cor de cada forma na entrada de `Parser::on_element`: atributo de apresentação, classe e estilo inline. Manter a cor durante toda a geração daquela forma, para que segmentos de um mesmo `path` recebam a mesma cor. `Parser::add` preenche `Common.color` com essa cor ou mantém o padrão existente. Restaurar/atribuir o estado a cada forma para evitar vazamento de cor entre elementos; grupos não propagam `stroke`.
8. Preservar o comportamento existente de `hidden()` (`display="none"` e estilo inline correspondente). A resolução de cor não deve reintroduzir geometria oculta nem se transformar em suporte novo a outras propriedades. Declarar limitações CSS com avisos sem alterar essa regra existente.
9. Agregar avisos em `Drawing.read_notes`, distinguindo estilos parcialmente aplicados, folhas/seletores ignorados, CSS malformado e herança omitida. Detectar declarações de traço em contêineres para emitir aviso, sem herdar cor. Manter os avisos existentes de texto, imagens, `<use>` e cantos arredondados; substituir o aviso genérico de classes não aplicadas apenas onde houver suporte real.
10. Não mudar os orçamentos aprovados. Antes de reter regras/textos ou avançar processamento acima de um limite, retornar `IoError::Format` com o recurso excedido. Testar fronteiras nos limites reais; usar testes locais para estados internos cujo preparo por XML seria desproporcional. Com o índice por classe, o orçamento condicional de comparação de candidatos não se aplica como uma varredura regras × formas: documentar essa escolha e comprovar a busca por token nos testes/revisão.

## Dependências e ordem de execução

```text
T1: cor direta na forma e avisos básicos
  → T2: folhas incondicionais e classes indexadas
    → T3: descarte seguro e avisos de suporte parcial
      → T4: fronteiras e entradas hostis
        → T5: documentação, verificação visual e qualidade final
```

Cada tarefa inclui testes de comportamento que falham antes da mudança e passam depois. Os orçamentos necessários entram junto com cada capacidade; T4 completa e audita as fronteiras, sem deixar o parser provisoriamente ilimitado. As tarefas são sequenciais porque compartilham o importador e seu estado. Uma revisão independente ou a documentação pode ser feita em paralelo somente se solicitada e com os contratos já estabilizados; não é necessária para executar este plano.

## Validação

- Baseline antes da implementação: `cargo test -p cadcraft-io svg`. Registrar falhas preexistentes e corrigir apenas o que pertencer ao escopo.
- Por tarefa: executar o filtro do comportamento novo e `cargo test -p cadcraft-io svg`; compilar com `cargo check -p cadcraft-io` quando a mudança de tipos/estrutura justificar. Não repetir verificações amplas sem alteração ou falha que as justifique.
- Fronteiras: limite exato e limite + 1 para bytes, regras, declarações, profundidade e tokens de classe por elemento/documento; folhas inelegíveis contam apenas bytes de CSS, conforme a especificação. Preparar entradas que isolem um orçamento por vez.
- Segurança: comentários/strings com delimitadores, blocos sem fechamento, regras aninhadas ignoradas, seletores parcialmente parecidos com classes aceitas, UTF-8 e entidades XML, documento com muitas formas e regras repetidas. Nenhum caso deve causar pânico ou omissão silenciosa por estouro.
- Verificação de ponta a ponta com SVG sintético criado em `/tmp/cadcraft-svg-css.svg`: `cargo run -p cadcraft-cli -- convert /tmp/cadcraft-svg-css.svg /tmp/cadcraft-svg-css.png`. Abrir a imagem produzida para comparar cores e geometria, e conferir `Drawing.read_notes` nos testes ou na aplicação. A imagem, sozinha, não comprova a exatidão de canais RGB: os testes inspecionam `Common.color`.
- Antes de commit: `cargo xtask ci`, como exigido pelo projeto. Finalizar revisão do diff para ausência de pânicos/unsafe, dependências de UI, alteração de CI ou recursos não atribuídos.

## Riscos e mitigação

| Risco | Consequência | Mitigação |
|---|---|---|
| Delimitadores em comentários ou strings | Aplicar conteúdo de um bloco que deveria ser ignorado | Scanner com estados mínimos e testes adversariais na T3 |
| Cor temporária do parser persistir entre formas | Recolorir entidades não relacionadas | Testes com formas consecutivas e caminhos que geram várias entidades desde a T1 |
| Duas passagens dobrarem contadores ou alocações | Rejeitar arquivos válidos ou consumir memória excessiva | Definir responsabilidade de cada passagem; verificar bytes antes de acumular; não reter DOM |
| Confundir suporte parcial com CSS completo | Divergência visual sem explicação | Tabela de resultados aprovada e avisos agregados; atualização explícita de FF14 |
| Trabalho proporcional a regras × formas | Arquivo pequeno causar processamento excessivo | Índice por classe e limites de tokens/regras |
| Documento de gaps ou PRs externos mudarem | Duplicar contribuição | Conferir branch upstream e PRs relacionados antes de iniciar o código e novamente antes de publicar |

## Preparação do futuro PR

Preparar branch de feature no fork do usuário ao iniciar o código. Preservar alterações locais alheias ao escopo e incluir somente os arquivos desta contribuição. O fork e o remote do usuário ainda precisam ser identificados antes do primeiro push; nenhuma publicação faz parte da elaboração deste plano. A documentação compartilhada e a ADR local ausentes devem ser verificadas se uma escolha passar a depender delas, especialmente uma nova biblioteca. Como a proposta usa dependências existentes, não há escolha pendente de biblioteca para começar.

Antes de preparar o PR, cumprir a regra do projeto para créditos opcionais do próprio contribuidor e não editar `contributors.json`. Título previsto: `SVG import: support hexadecimal stroke colors in simple CSS classes`. A descrição deve destacar o subconjunto e a aproximação de cascata, com exemplos e evidência de verificação. Nenhum PR pode alegar encerrar todo FF14.

## Revisão do plano

Plano pronto para revisão do usuário. A implementação começa pela T1 após a revisão do plano prevista pelas habilidades de especificação e planejamento.
