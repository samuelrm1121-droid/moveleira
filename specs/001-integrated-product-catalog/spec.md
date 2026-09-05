# Feature Specification: Catálogo Integrado de Produtos

**Feature Branch**: Não criada (nenhum hook de branch configurado)

**Created**: 2026-09-05

**Status**: Draft

**Input**: User description: "Permitir que fornecedores cadastrem produtos e que lojistas os
adicionem aos próprios catálogos por vínculo, sem recadastramento, para exibição pública controlada."

## Clarifications

### Session 2026-09-05

- Q: No MVP, cada lojista terá exatamente um catálogo público ou poderá manter vários catálogos
  independentes? → A: Cada lojista possui exatamente um catálogo público.
- Q: Como as categorias dos produtos devem ser definidas no MVP? → A: A plataforma mantém uma
  lista central, e os fornecedores selecionam categorias existentes.
- Q: Como o fornecedor deve adicionar imagens aos produtos no MVP? → A: O fornecedor envia os
  arquivos, e a plataforma mantém as imagens.
- Q: Qual deve ser o estado inicial de um produto recém-cadastrado e da entrada criada quando um
  lojista o adiciona? → A: O produto do fornecedor inicia ativo, e a entrada do lojista inicia oculta.
- Q: Quando dois usuários do mesmo fornecedor editarem o mesmo produto ao mesmo tempo, como uma
  gravação baseada em dados desatualizados deve ser tratada? → A: A gravação desatualizada é
  recusada, o conflito é informado e o usuário deve recarregar e reaplicar suas alterações.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Gerenciar produtos do fornecedor (Priority: P1)

Como fornecedor, quero cadastrar e manter meus produtos para que seus dados sejam a fonte oficial
usada pelos lojistas.

**Why this priority**: Sem um produto mantido pelo fornecedor, nenhum dos demais participantes
consegue percorrer o fluxo principal do MVP.

**Independent Test**: Pode ser testada com uma conta de fornecedor que cria um produto, consulta
seus dados, edita seus campos e alterna seu estado entre ativo e inativo.

**Acceptance Scenarios**:

1. **Given** um fornecedor identificado e sem o produto cadastrado, **When** ele informa nome,
   seleciona uma categoria central válida e, opcionalmente, informa descrição, referência, marca,
   modelo, material, dimensões e arquivos de imagem válidos, **Then** o produto é cadastrado sob sua
   propriedade e fica disponível na sua relação de produtos.
2. **Given** um fornecedor com produtos cadastrados, **When** ele acessa sua relação ou abre um
   item, **Then** visualiza somente os produtos que lhe pertencem e os dados atuais de cada um.
3. **Given** um produto pertencente ao fornecedor, **When** o fornecedor altera seus dados válidos,
   **Then** as alterações passam a representar a versão oficial do produto.
4. **Given** um produto ativo, **When** o fornecedor o desativa, **Then** ele deixa de estar
   disponível para novas inclusões e para exibição pública, sem apagar seus vínculos existentes.
5. **Given** um produto inativo, **When** o fornecedor o reativa, **Then** ele volta a estar
   disponível para lojistas e seus vínculos existentes voltam a considerar o estado de visibilidade
   escolhido por cada lojista.
6. **Given** dois usuários do mesmo fornecedor editando a mesma versão de um produto, **When** um
   deles salva primeiro e o outro tenta salvar depois sem carregar a versão atual, **Then** a segunda
   gravação é recusada e o usuário é orientado a recarregar e reaplicar suas alterações.

---

### User Story 2 - Adicionar produto vinculado ao catálogo (Priority: P2)

Como lojista, quero navegar pelos fornecedores e adicionar seus produtos ao meu catálogo sem
redigitar informações, para montar minha oferta com rapidez e manter os dados de origem atualizados.

**Why this priority**: O vínculo sem recadastramento é a proposta de valor central da plataforma e
conecta a gestão do fornecedor ao catálogo do lojista.

**Independent Test**: Pode ser testada com dados de fornecedor previamente disponíveis; um lojista
localiza um produto ativo, adiciona-o ao catálogo e confirma que a entrada referencia o produto de
origem e não exige a repetição dos dados do fornecedor.

**Acceptance Scenarios**:

1. **Given** fornecedores disponíveis com produtos ativos, **When** o lojista acessa a relação de
   fornecedores e escolhe um deles, **Then** visualiza os produtos ativos daquele fornecedor.
2. **Given** um produto ativo ainda ausente do catálogo do lojista, **When** o lojista o adiciona,
   **Then** uma entrada vinculada ao produto original é criada sem solicitar recadastramento dos
   dados mantidos pelo fornecedor.
3. **Given** um produto já vinculado ao catálogo do lojista, **When** o lojista tenta adicioná-lo
   novamente, **Then** nenhuma entrada duplicada é criada e o lojista recebe uma orientação clara.
4. **Given** um produto inativo, **When** o lojista tenta adicioná-lo, **Then** a inclusão é recusada
   sem criar uma entrada parcial no catálogo.

---

### User Story 3 - Controlar a visibilidade no catálogo (Priority: P3)

Como lojista, quero decidir quais produtos vinculados ficam visíveis para que eu controle a seleção
apresentada na minha loja sem alterar os dados oficiais do fornecedor.

**Why this priority**: O lojista precisa administrar sua própria seleção antes que o catálogo possa
ser publicado com segurança para consumidores.

**Independent Test**: Pode ser testada com um produto ativo já vinculado; o lojista alterna a
visibilidade e o resultado é verificado no catálogo público sem editar o produto de origem.

**Acceptance Scenarios**:

1. **Given** um produto recém-adicionado ao catálogo do lojista, **When** a inclusão é concluída,
   **Then** ele começa oculto do público até que o lojista decida publicá-lo.
2. **Given** um produto vinculado, ativo e oculto, **When** o lojista o torna visível, **Then** ele
   passa a integrar o catálogo público daquela loja.
3. **Given** um produto vinculado e visível, **When** o lojista o oculta, **Then** ele deixa de ser
   exibido ao público, mas o vínculo permanece no catálogo administrativo.
4. **Given** uma entrada vinculada, **When** o lojista tenta alterar um dado pertencente ao
   fornecedor, **Then** a alteração é impedida e o dado oficial permanece intacto.

---

### User Story 4 - Consultar catálogo público (Priority: P4)

Como consumidor, quero acessar o catálogo público de uma loja e visualizar seus produtos para
conhecer a seleção oferecida pelo lojista.

**Why this priority**: Esta história completa o fluxo de valor do MVP ao tornar a seleção integrada
útil para o público final.

**Independent Test**: Pode ser testada sem autenticação, usando uma loja com produtos vinculados em
diferentes estados; somente os simultaneamente ativos e visíveis devem aparecer com seus dados e
imagens atuais.

**Acceptance Scenarios**:

1. **Given** uma loja identificável com produtos ativos e marcados como visíveis, **When** um
   consumidor acessa seu catálogo público, **Then** visualiza somente esses produtos.
2. **Given** um produto exibido no catálogo público, **When** o consumidor abre seus detalhes,
   **Then** visualiza nome, descrição, categoria, fornecedor e as imagens disponíveis.
3. **Given** uma loja sem produtos elegíveis para publicação, **When** um consumidor acessa seu
   catálogo, **Then** recebe um estado vazio compreensível e não vê produtos ocultos ou inativos.

---

### User Story 5 - Refletir atualizações do fornecedor (Priority: P5)

Como fornecedor e lojista, quero que alterações nos dados oficiais cheguem aos catálogos vinculados
sem eliminar decisões próprias do lojista, para evitar divergências e retrabalho.

**Why this priority**: A atualização compartilhada comprova que o catálogo é integrado e não uma
coleção de cópias independentes.

**Independent Test**: Pode ser testada com um produto associado a dois lojistas com visibilidades
diferentes; após uma edição do fornecedor, ambos recebem os novos dados e mantêm suas respectivas
configurações de visibilidade.

**Acceptance Scenarios**:

1. **Given** um produto vinculado a um ou mais catálogos, **When** o fornecedor altera nome,
   descrição, categoria, referência, marca, modelo, material, dimensões ou imagens, **Then** todos os
   vínculos passam a apresentar os dados oficiais atualizados.
2. **Given** lojistas com escolhas de visibilidade diferentes para o mesmo produto, **When** o
   fornecedor edita esse produto, **Then** cada escolha de visibilidade permanece inalterada.
3. **Given** um produto vinculado e publicamente visível, **When** o fornecedor o desativa, **Then**
   ele deixa de aparecer no catálogo público sem que o vínculo ou a escolha do lojista sejam
   apagados.
4. **Given** um produto anteriormente desativado com vínculo preservado, **When** o fornecedor o
   reativa, **Then** os dados atuais voltam a ser elegíveis para publicação conforme a escolha de
   visibilidade já mantida pelo lojista.

### Edge Cases

- O fornecedor tenta cadastrar ou salvar um produto sem nome ou categoria.
- O fornecedor envia uma imagem maior que o limite, em formato não aceito ou excede a quantidade
  permitida por produto.
- Um produto válido não possui imagens; seus demais dados continuam disponíveis com uma indicação
  visual apropriada no lugar da imagem.
- O lojista tenta adicionar o mesmo produto mais de uma vez ao próprio catálogo.
- O produto é desativado enquanto um lojista tenta adicioná-lo ou enquanto um consumidor consulta
  o catálogo público.
- Um produto de origem fica indisponível inesperadamente; o vínculo é preservado para diagnóstico,
  mas o item não é exposto ao consumidor.
- Um fornecedor tenta consultar ou modificar um produto pertencente a outro fornecedor.
- Dois usuários do mesmo fornecedor tentam salvar alterações concorrentes no mesmo produto.
- Um lojista tenta alterar a visibilidade de uma entrada pertencente a outro lojista.
- Um consumidor acessa uma loja inexistente, indisponível ou sem produtos publicáveis.
- Uma edição do fornecedor troca ou remove todas as imagens de um produto já publicado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A plataforma MUST distinguir fornecedor, lojista e consumidor e MUST restringir as
  operações administrativas aos dados pertencentes à organização identificada.
- **FR-002**: Um fornecedor MUST poder cadastrar um produto com nome e uma categoria selecionada da
  lista central e, opcionalmente, descrição, código de referência do fornecedor, marca, modelo,
  material, largura, altura, profundidade e fazer upload de até dez arquivos de imagem.
- **FR-003**: Nome MUST conter valor não vazio, e a categoria selecionada MUST existir e estar ativa
  na lista central. Dimensões informadas MUST ser positivas. Cada imagem MUST usar formato JPEG, PNG
  ou WebP e ter no máximo 10 MB; qualquer violação MUST ser informada antes da conclusão da operação.
- **FR-004**: Um produto cadastrado com dados válidos MUST iniciar ativo por padrão para participar
  imediatamente do fluxo principal do MVP.
- **FR-005**: Um fornecedor MUST poder listar, consultar e abrir os produtos que lhe pertencem,
  incluindo seus dados atuais e estado de atividade.
- **FR-006**: Um fornecedor MUST poder editar os dados e as imagens dos produtos que lhe pertencem.
- **FR-007**: Um fornecedor MUST poder ativar ou desativar os produtos que lhe pertencem.
- **FR-008**: A desativação MUST preservar o produto e os vínculos já criados, impedir novas
  inclusões em catálogos e retirar o produto da exibição pública enquanto estiver inativo.
- **FR-009**: Um lojista MUST poder visualizar os fornecedores disponíveis para integração.
- **FR-010**: Um lojista MUST poder selecionar um fornecedor e visualizar seus produtos ativos.
- **FR-011**: Na visualização de um produto, o lojista MUST receber os dados e imagens atuais
  mantidos pelo fornecedor, além da indicação da origem do produto.
- **FR-012**: Um lojista MUST poder adicionar um produto ativo de fornecedor ao próprio catálogo
  sem redigitar os campos mantidos pelo fornecedor.
- **FR-013**: Cada inclusão MUST criar e preservar uma relação identificável entre a entrada do
  catálogo, o lojista, o produto original e seu fornecedor.
- **FR-014**: A inclusão MUST NOT criar uma cópia independente do produto original como
  comportamento padrão.
- **FR-015**: O mesmo produto de origem MUST ter no máximo uma entrada no catálogo de um mesmo
  lojista; uma nova tentativa MUST retornar um resultado compreensível sem duplicação.
- **FR-016**: Uma nova entrada de catálogo MUST iniciar oculta por padrão até uma decisão
  explícita do lojista.
- **FR-017**: O lojista MUST poder tornar visível ou ocultar qualquer entrada que pertença ao seu
  próprio catálogo.
- **FR-018**: A visibilidade efetiva de um produto no catálogo público MUST exigir simultaneamente
  que o produto de origem esteja ativo e que o lojista o tenha marcado como visível.
- **FR-019**: Um consumidor MUST poder acessar o catálogo público de um lojista sem autenticação
  administrativa.
- **FR-020**: O catálogo público MUST exibir apenas entradas efetivamente visíveis da loja
  acessada e MUST NOT expor entradas pertencentes a outra loja.
- **FR-021**: O consumidor MUST poder visualizar nome, categoria, fornecedor e todos os campos
  opcionais e imagens disponíveis de cada produto publicado.
- **FR-022**: Alterações válidas feitas pelo fornecedor nos dados ou imagens do produto MUST ser
  refletidas em todas as entradas vinculadas.
- **FR-023**: A propagação de alterações do fornecedor MUST preservar todos os valores pertencentes
  ao lojista; no MVP, isso inclui obrigatoriamente a escolha de visibilidade.
- **FR-024**: A reativação de um produto MUST restabelecer sua disponibilidade para lojistas e MUST
  reutilizar os vínculos e escolhas de visibilidade preservados.
- **FR-025**: O lojista MUST NOT modificar pela entrada de catálogo os campos cuja propriedade é do
  fornecedor.
- **FR-026**: Operações inválidas ou não autorizadas MUST ser recusadas com mensagem acionável e
  MUST NOT deixar produtos, vínculos ou configurações em estado parcial.
- **FR-027**: A visualização do produto para lojistas e consumidores MUST identificar o fornecedor,
  e a entrada administrativa do catálogo MUST também identificar o produto de origem associado.
- **FR-028**: Cada lojista MUST possuir exatamente um catálogo público, e todas as suas entradas de
  produto e escolhas de visibilidade MUST pertencer a esse catálogo.
- **FR-029**: A plataforma MUST disponibilizar uma lista central e compartilhada de categorias para
  seleção pelos fornecedores; fornecedores MUST NOT criar categorias livres no cadastro de produto.
- **FR-030**: A plataforma MUST manter os arquivos de imagem enviados pelo fornecedor e MUST NOT
  depender de URLs externas informadas por ele para exibir as imagens do produto.
- **FR-031**: Uma tentativa de salvar um produto com base em uma versão anterior à versão atual
  MUST ser recusada sem perda da alteração já confirmada. O usuário MUST ser informado do conflito
  e orientado a recarregar os dados e reaplicar sua alteração.

### Scope Boundaries

**In scope**:

- Gestão de cadastro, consulta, edição, ativação e desativação de produtos pelo fornecedor.
- Descoberta de fornecedores e de seus produtos ativos pelo lojista.
- Inclusão vinculada e sem recadastramento de produto no catálogo do lojista.
- Controle de visibilidade por lojista e consulta pública dos produtos elegíveis.
- Reflexo de alterações do fornecedor com preservação dos valores pertencentes ao lojista.

**Out of scope**:

- Carrinho, pagamentos, pedidos, transações de marketplace ou intermediação comercial.
- Inteligência artificial e analytics avançado.
- Personalizações de produto pelo lojista além da visibilidade.
- Cadastro de produtos independentes pelo lojista, exclusão definitiva de produtos e remoção
  definitiva de vínculos.
- Preços, estoque, logística, avaliações, favoritos, busca avançada e recomendação de produtos.
- Cadastro de organizações, convite de usuários e recuperação de acesso.
- Interface administrativa para criar, editar, ordenar ou excluir categorias da lista central.
- Cadastro de imagens por URL externa.

### Key Entities *(include if feature involves data)*

- **Supplier**: Organização responsável pela origem e manutenção dos dados oficiais de seus
  produtos.
- **SupplierProduct**: Produto pertencente a um fornecedor; contém identidade, nome, categoria,
  descrição, referência, marca, modelo, material, dimensões, estado de atividade e sua coleção de
  imagens.
- **ProductImage**: Arquivo de imagem enviado à plataforma e pertencente ao produto de origem, com
  identidade e ordem de apresentação.
- **Category**: Classificação central compartilhada por todos os fornecedores; possui identidade,
  nome e estado de atividade e pode ser selecionada por vários produtos.
- **Retailer**: Organização que seleciona produtos de fornecedores para compor seu catálogo público.
- **RetailerCatalog**: Seleção administrativa e pública única pertencente a um lojista; cada
  lojista possui exatamente um catálogo.
- **CatalogEntry**: Relação única entre um catálogo de lojista e um produto de fornecedor; mantém a
  escolha de visibilidade e preserva a identidade da origem sem possuir uma cópia independente dos
  dados oficiais.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Pelo menos 90% dos fornecedores participantes do piloto conseguem cadastrar um
  produto válido em até 3 minutos e na primeira tentativa, sem assistência.
- **SC-002**: Pelo menos 90% dos lojistas participantes conseguem localizar e adicionar um produto
  existente ao próprio catálogo em até 2 minutos, sem redigitar dados do fornecedor.
- **SC-003**: Em 100% das inclusões concluídas, a entrada do catálogo mantém uma origem de produto
  identificável e não gera uma segunda entrada para a mesma combinação de lojista e produto.
- **SC-004**: Pelo menos 95% das alterações válidas de fornecedor ficam visíveis aos lojistas e,
  quando elegíveis, aos consumidores em até 60 segundos, preservando 100% dos valores do lojista.
- **SC-005**: Em 100% dos testes de estado, produtos ocultos pelo lojista ou inativos no fornecedor
  não aparecem no catálogo público, enquanto seus vínculos administrativos permanecem preservados.
- **SC-006**: Em pelo menos 95% das consultas sob a carga esperada do MVP, consumidores visualizam
  a relação ou os detalhes de produtos publicados em até 2 segundos.
- **SC-007**: O fluxo mantém os resultados definidos nesta especificação com pelo menos 50
  fornecedores, 5.000 produtos de origem, 200 produtos por catálogo de lojista e 100 consumidores
  consultando catálogos ao mesmo tempo.
- **SC-008**: Pelo menos 85% dos fornecedores e lojistas do piloto avaliam que o fluxo elimina o
  recadastramento manual de informações de produto.
- **SC-009**: Um teste completo demonstra, sem intervenção em dados internos, a sequência fornecedor
  cadastra produto, lojista encontra e adiciona, lojista publica e consumidor visualiza mantendo o
  vínculo com a origem.

## Assumptions

- Fornecedores e lojistas já possuem identidade válida, vínculo com uma organização e o papel
  necessário; criação de contas e recuperação de acesso não fazem parte desta feature.
- Para o MVP, nome e a seleção de uma categoria central ativa são obrigatórios. Descrição,
  referência do fornecedor, marca, modelo, material, dimensões e imagens são opcionais. Os dados do
  produto não incluem preço, estoque, logística ou condições comerciais.
- A lista inicial de categorias é fornecida como dado de referência da plataforma; sua manutenção
  por uma interface administrativa não faz parte desta feature.
- Visibilidade é o único valor personalizável pelo lojista no MVP. Valores próprios futuros
  pertencerão ao lojista e seguirão a mesma regra de preservação durante atualizações do fornecedor.
- A desativação do produto suspende sua exposição, mas não expressa exclusão nem encerra os vínculos.
- Cada loja possui exatamente um catálogo e uma identificação pública estável pela qual o
  consumidor o acessa.
- A plataforma mantém e apresenta os arquivos de imagem enviados pelo fornecedor; URLs externas não
  são aceitas como origem de imagem no MVP.
- A carga descrita em SC-007 representa o volume-alvo inicial para validação do MVP e pode ser
  revista após dados reais do piloto.
