# Tarefas: cores de traço por classe CSS no SVG

Contrato: [especificação aprovada](../SPEC-svg-css.md). Abordagem: [plano técnico](plan.md). Nenhuma tarefa de implementação foi executada.

## T1 — Cor de traço diretamente na forma

**Descrição:** estabelecer o caminho de ponta a ponta entre `stroke`/`style` da forma e `Common.color`, preservando camadas, geometria e ocultação existentes.

**Dependências:** nenhuma. Antes de editar código, conferir o estado do upstream/PRs e preparar branch isolada para a contribuição; executar baseline dos testes SVG.

**Arquivos prováveis:** `crates/io/src/svg_read.rs`, novo `crates/io/src/svg_read/css.rs`, `crates/io/src/tests.rs` (M: 3 arquivos).

**Aceitação:**
- [ ] `#RGB`/`#RRGGBB` em `stroke` e estilo inline tornam-se cor explícita, com estilo inline prevalecendo; declarações repetidas usam a última suportada.
- [ ] Todas as entidades de um caminho recebem a cor da forma; formas seguintes sem cor mantêm o padrão da camada; transformações, camadas e `display:none` continuam corretos.
- [ ] Declarações ignoradas geram aviso parcial; aplicar orçamento de texto inline e declarações com erros explícitos antes de excedê-los.

**Verificação:** demonstrar teste de regressão falhando antes da mudança; executar `cargo test -p cadcraft-io svg_css_direct`, `cargo test -p cadcraft-io svg` e `cargo check -p cadcraft-io`. Os nomes novos deverão conter `svg_css_direct` para que o filtro execute casos reais.

## T2 — Classes em folhas incondicionais

**Descrição:** coletar folhas internas antes da geometria e resolver classes por índice, inclusive quando a folha aparece depois da forma ou em `<defs>`.

**Dependências:** T1.

**Arquivos prováveis:** `crates/io/src/svg_read.rs`, `crates/io/src/svg_read/css.rs`, `crates/io/src/tests.rs` (M: 3 arquivos).

**Aceitação:**
- [ ] Classes da gramática aprovada aplicam cores em formas/caminhos; regras mantêm a ordem de documento, independente da ordem de nomes no atributo `class`.
- [ ] Atributo de apresentação < classe < inline; folhas em texto/CDATA antes/depois das formas produzem o mesmo resultado; tipos/mídias inelegíveis são ignorados com aviso.
- [ ] Índice de classes e coleta respeitam os limites de texto, regras, declarações e tokens; CSS de folhas inelegíveis conta bytes sem ser interpretado. Folhas inteiramente suportadas não emitem aviso de classes não aplicadas.

**Verificação:** testar antes/depois da implementação; executar `cargo test -p cadcraft-io svg_css_class` e `cargo test -p cadcraft-io svg`; inspecionar cores reais de formas distintas e formas consecutivas. Usar o prefixo `svg_css_class` nos novos testes correspondentes.

## Checkpoint — Depois de T1 e T2

- [ ] Casos principais funcionam de ponta a ponta pelos testes de `read`.
- [ ] Testes SVG existentes passam; tipo, quantidade e geometria de entidades permanecem corretos.
- [ ] Revisar com o usuário o recorte entregue e eventuais desvios do contrato antes de seguir.

## T3 — Suporte parcial com descarte seguro

**Descrição:** completar o tratamento determinístico de CSS fora do subconjunto e malformado, incluindo blocos ignorados e herança omitida.

**Dependências:** T2.

**Arquivos prováveis:** `crates/io/src/svg_read.rs`, `crates/io/src/svg_read/css.rs`, `crates/io/src/tests.rs` (M: 3 arquivos).

**Aceitação:**
- [ ] Toda a tabela de resultados da especificação passa, com cores/fallbacks e avisos esperados; traços em contêineres geram aviso sem herança.
- [ ] Seletores compostos, listas, escapes e pseudoclasses nunca são aceitos por prefixo; blocos `@` são descartados inteiros, inclusive com aninhamento e delimitadores em strings/comentários.
- [ ] CSS truncado termina com recuperação/aviso conforme o contrato; avisos são agregados, preservando os de conteúdo SVG já ignorado; orçamento de profundidade e contadores em blocos ignorados são aplicados.

**Verificação:** novos testes com prefixo `svg_css_partial`; executar `cargo test -p cadcraft-io svg_css_partial` e `cargo test -p cadcraft-io svg`. Incluir contraexemplos em que uma regra ignorada precede uma regra válida para detectar vazamento de conteúdo.

## T4 — Fronteiras e arquivos hostis

**Descrição:** completar provas das fronteiras dos orçamentos e verificar que os dois passes não duplicam cobranças nem deixam conteúdo ignorado contornar limites.

**Dependências:** T3; limites já devem acompanhar as capacidades anteriores.

**Arquivos prováveis:** `crates/io/src/svg_read/css.rs`, `crates/io/src/svg_read.rs` somente se necessário, `crates/io/src/tests.rs` (M: até 3 arquivos).

**Aceitação:**
- [ ] Cada limite aplicável aceita o valor exato e retorna `IoError::Format` em limite + 1, com o recurso correto; os testes isolam os demais orçamentos e verificam ausência de pânico.
- [ ] Muitas classes/formas/regras usam busca indexada; confirmar que nenhuma varredura candidatos × formas foi introduzida. Se uma comparação de candidatos for necessária, aplicar e testar também seu orçamento de 2.000.000.
- [ ] Entradas malformadas, comentários/strings adversariais e subárvores ignoradas não causam processamento não limitado; SVGZ mantém o limite de descompressão existente.

**Verificação:** testes com prefixo `svg_css_budget` para fronteiras e `svg_css_hostile` para entradas adversariais; executar ambos os filtros e `cargo test -p cadcraft-io svg`. Preparar testes privados de contador quando isso evita gerar XML desproporcional, mantendo casos reais pela API pública para validar a integração dos contadores.

## Checkpoint — Depois de T3 e T4

- [ ] Todos os casos de aceitação da especificação estão associados a testes concretos.
- [ ] Revisar comportamento parcial e limites com o usuário; nenhuma mudança de escopo foi feita silenciosamente.
- [ ] Revisão de segurança do diff confirma ausência de pânicos em produção, recursão não limitada e alocações acima dos orçamentos.

## T5 — Documentação e validação final

**Descrição:** registrar o subconjunto entregue, conferir o resultado visual e reunir evidências para um PR pequeno e revisável.

**Dependências:** T4.

**Arquivos prováveis:** `docs/gaps.md`, `docs/file-format-parity.md`, `crates/io/src/svg_read.rs` (documentação de módulo), `tasks/todo.md` (M: até 4 arquivos).

**Aceitação:**
- [ ] FF14 e o status de importação SVG descrevem suporte parcial, limitações e avisos, com status/revisão atualizados sem exagerar paridade.
- [ ] Um SVG sintético convertido para PNG mostra as cores esperadas; testes confirmam canais RGB, camadas e geometria; não adicionar ativos binários ao repositório.
- [ ] `cargo xtask ci` passa e a revisão final encontra somente mudanças relacionadas à contribuição; evidências são registradas para a descrição do PR.

**Verificação:** gerar o SVG em `/tmp/cadcraft-svg-css.svg`; executar `cargo run -p cadcraft-cli -- convert /tmp/cadcraft-svg-css.svg /tmp/cadcraft-svg-css.png` e abrir o PNG. Executar `cargo xtask ci` antes de commit e `git diff --check`; revisar a documentação contra o comportamento testado. Se uma verificação falhar por causa preexistente, registrar a falha sem silenciar checks ou declarar a feature concluída.

## Checkpoint — Pronto para preparar o PR

- [ ] Especificação, implementação, testes e documentação concordam.
- [ ] Créditos opcionais do próprio contribuidor tratados conforme `AGENTS.md`; sem editar `contributors.json`.
- [ ] Fork/remote do usuário identificado e branch de feature disponível para futuro push.
- [ ] PR descreverá apenas o subconjunto entregue, com validação e limitações; revisão humana antes do merge.
