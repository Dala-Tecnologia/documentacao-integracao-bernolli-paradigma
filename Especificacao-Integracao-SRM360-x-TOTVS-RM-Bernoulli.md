# Especificação Técnica de Integração
## SRM 360 (Paradigma) x TOTVS RM (Bernoulli Educação)

---

## 1. Dados Gerais

| | |
|---|---|
| **Nome do Cliente** | Bernoulli Educação |
| **Projeto** | Revitalização Portal de Compras — Onda Única |
| **Responsável no Cliente** | Vitor Tadeu Rosenburg Pereira |
| **Gerente do Projeto (Paradigma)** | Moises Mauricio Rodrigues Souza |
| **Plataforma destino** | Paradigma SRM 360 |
| **Sistema legado / ERP** | TOTVS RM — Gestão de Estoque, Compras e Faturamento |
| **Fase** | Especificação técnica de integração |
| **Versão deste documento** | 1.0 |
| **Data** | 13/08/2026 |

---

## 2. Objetivo e Contexto

### 2.1. Objetivo

Este documento detalha, em nível técnico, **todas as integrações entre a plataforma Paradigma SRM 360 e o ERP TOTVS RM** da Bernoulli Educação.

Seu propósito é servir como **contrato técnico entre as partes**: define o que cada sistema envia, o que recebe, em que momento, por qual método, e qual o comportamento esperado em cada cenário — inclusive nos de exceção.

A aprovação deste documento pela Bernoulli é pré-requisito para o início do desenvolvimento da camada de integração.

### 2.2. Escopo deste documento

**Contemplado:** todos os fluxos de integração entre a plataforma SRM 360 e o ERP, além dos cadastros mestres.

**Não contemplado** (módulos que operam integralmente dentro da plataforma SRM 360, **sem qualquer integração com o ERP**):

- Avaliação de Fornecedores
- BI / Power BI
- Exportador de Dados
- Comprador Virtual PIPO — gera cotações e re-pedidos internamente; os **pedidos** resultantes seguem o fluxo padrão de §12
- Dashboard de Movimentações
- Assistente Virtual
- ANA — Autorizações, Notificações e Aprovações mobile
- **Catálogos** — o catálogo em si **não é integrado com o ERP**; apenas os **pedidos** gerados a partir dele são

---

## 3. Visão Geral da Arquitetura

### 3.1. Topologia

```
┌─────────────────────────┐                    ┌─────────────────────────┐
│                         │   ERP → SRM        │                         │
│      TOTVS RM           │  ───────────────>  │    Paradigma SRM 360    │
│  Gestão de Estoque,     │   (POST / PATCH)   │                         │
│  Compras e Faturamento  │                    │   API REST /api/v1.0    │
│                         │  <───────────────  │                         │
│                         │   SRM → ERP        │                         │
└─────────────────────────┘   (GET + Habilitar)└─────────────────────────┘
              ▲                                              ▲
              │                                              │
              └────────── Camada de integração ──────────────┘
                     (agendamento, log, reprocessamento)
```

**A comunicação é sempre iniciada pelo lado do ERP / camada de integração.** A plataforma SRM 360 expõe a API; ela não invoca serviços no ambiente da Bernoulli. Isso vale nos dois sentidos:

- **ERP → SRM (envio):** o integrador faz `POST` ou `PATCH` no endpoint correspondente.
- **SRM → ERP (retorno):** o integrador faz `GET` no endpoint de fila; a plataforma retorna os documentos pendentes e os marca como consumidos.

### 3.2. Modelo de fila e consumo

Os endpoints de retorno (`GET`) operam como **filas de consumo destrutivo**: um documento retornado é marcado como integrado e não aparece na chamada seguinte. Para cada fila existe um endpoint `Habilitar*` correspondente, que recoloca o documento na fila — mecanismo oficial de reprocessamento em caso de falha do lado do ERP (ver §16).

### 3.3. Responsabilidades

| Responsabilidade | Sistema |
|---|---|
| Gestão de dados mestres (fonte da verdade) | TOTVS RM |
| Cadastro e homologação de fornecedores | SRM 360 (com retorno ao RM) |
| Solicitação de compra e check orçamentário | TOTVS RM |
| Processos de negociação (cotação, leilão) | SRM 360 |
| Gestão de contratos e catálogos | SRM 360 |
| Geração do pedido de compra | SRM 360 |
| Validação orçamentária do pedido | TOTVS RM |
| Gestão de entrega / recebimento (NF) | TOTVS RM |
| Títulos financeiros | TOTVS RM (publicados no SRM) |
| Execução, agendamento e log da integração | Camada de integração |

> **Pendência P-01:** definir o responsável técnico pela camada de integração do lado RM — customização TOTVS dentro do módulo Gestão de Estoque, Compras e Faturamento (com processos e JOB Server) ou middleware/integrador próprio da Bernoulli. Este documento adota a redação neutra "o integrador", sem prejuízo de nenhuma das duas opções.

### 3.4. Ambientes

| Ambiente | Finalidade | URL base |
|---|---|---|
| Homologação | Testes integrados e aceite | *A definir* |
| Produção | Operação | *A definir* |

> **Pendência P-02:** URLs base dos ambientes, método de autenticação (token, chave de API, OAuth), política de expiração/renovação de credencial, e existência de rate limit ou paginação nos endpoints de fila. Ver §17.

### 3.5. Versão do ERP

| | |
|---|---|
| Sistema base | RM — TOTVS Gestão de Estoque, Compras e Faturamento |
| Base de dados | SQL Server |
| Versão ERP | *A confirmar* |

---

## 4. Premissas e Restrições

### 4.1. Premissas

1. O escopo funcional do projeto está aprovado pelas partes. Divergências identificadas durante o desenvolvimento serão tratadas com registro de nova versão deste documento.
2. Os cadastros de fornecedores existentes no RM estão **saneados e aptos** a interagir nos processos de compra antes da carga inicial.
3. O ERP é a **fonte da verdade** dos dados mestres. Alterações feitas diretamente no portal em entidades de domínio do ERP serão sobrescritas na próxima sincronização.
4. A plataforma SRM 360 **não trabalha com exclusão de cadastros básicos**. Registros integrados são inativados, nunca excluídos.
5. O check orçamentário e a suplementação de budget ocorrem integralmente no RM; a plataforma apenas reflete o resultado.
6. Todo pedido de compra é gerado **na plataforma SRM 360** e recebe número interno próprio; o número do ERP é devolvido posteriormente e ambos coexistem para rastreabilidade.
7. A gestão de entrega/recebimento ocorre **unicamente no ERP**, via lançamento de notas fiscais.

### 4.2. Restrições

1. Não está contemplado o desenvolvimento ou alteração de relatórios.
2. Não estão contempladas funcionalidades não explicitadas neste documento; solicitações adicionais serão tratadas mediante nova proposta.
3. Não está contemplada a migração de documentos históricos (pedidos, contratos, cotações encerrados) do portal legado para o SRM 360, salvo definição em contrário.
4. Os campos de rastreabilidade cuja estrutura é incompatível entre os dois sistemas (`sCdItemWbc`, `sCdOrigemEmpresa`, `sCdItemOrigemEmpresa`, `sCdItemEmpresa`) permanecem sem correspondente direto e serão tratados caso a caso na especificação de campos (§15).
5. Qualquer alteração posterior nos contratos de API do SRM 360 que impacte a camada de integração será tratada como item fora de escopo.

---

## 5. Matriz de Integrações

Consolidação de todos os pontos de integração previstos no projeto.

**Legenda de gatilho:** *Evento* = disparado por inclusão/alteração no sistema de origem · *Agendado* = execução periódica pelo integrador · *Manual* = acionado por usuário.

### 5.1. Dados mestres — ERP → SRM 360

| # | Entidade | Método | Gatilho | Periodicidade sugerida |
|---|---|---|---|---|
| 01 | Empresas compradoras | `POST /api/v1.0/Empresas` | Evento + carga inicial | Imediato |
| 02 | Empresas fornecedoras | `POST /api/v1.0/Empresas` | Evento + carga inicial | Imediato |
| 03 | Usuários cliente | `POST /api/v1.0/Usuarios` | Evento + carga inicial | Imediato |
| 04 | Categorias de produto | `POST /api/v1.0/Categorias` | Evento + carga inicial | Imediato |
| 05 | Empresa x Categoria | `POST /api/v1.0/EmpresasCategorias` | Evento + carga inicial | Imediato |
| 06 | Unidades de medida | `POST /api/v1.0/UnidadesMedidas` | Evento + carga inicial | Imediato |
| 07 | Itens (produto/serviço) | `POST /api/v1.0/Produtos` | Evento + carga inicial | Imediato |
| 08 | Condições de pagamento | `POST /api/v1.0/CondicoesPagamentos` | Evento + carga inicial | Imediato |
| 09 | Moedas | `POST /api/v1.0/Moedas` | Evento + carga inicial | Imediato |
| 10 | Cotação de moeda (câmbio) | `POST /api/v1.0/MoedasCotacoes` | Agendado | Diária |
| 11 | Centros de custo | `POST /api/v1.0/CentrosCustos` | Evento + carga inicial | Imediato |
| 12 | Contas contábeis | `POST /api/v1.0/ContasContabeis` | Evento + carga inicial | Imediato |
| 13 | Projetos | `POST /api/v1.0/Projetos` | Evento + carga inicial | Imediato |

### 5.2. Fornecedores — SRM 360 → ERP

| # | Fluxo | Método | Gatilho | Periodicidade sugerida |
|---|---|---|---|---|
| 14 | Fornecedor ativado/inativado no portal | `GET /api/v1.0/Empresas/AtivadaInativada` | Agendado | 30 min |
| 15 | Reprocessar retorno de fornecedor | `GET /api/v1.0/Empresas/HabilitarRetornarEmpresaAtivadaInativada` | Manual / tratamento de erro | Sob demanda |
| 16 | Fornecedores sem de/para | `GET /api/v1.0/Empresas/SemDePara` | Agendado | Diária |

### 5.3. Solicitações e requisições

| # | Fluxo | Método | Sentido | Gatilho | Periodicidade |
|---|---|---|---|---|---|
| 17 | Envio de solicitações de compra | `POST /api/v1.0/OrdensCompras` | ERP → SRM | Evento (SC aprovada com check orçamentário) | Imediato |
| 18 | Requisições canceladas no portal | `GET /api/v1.0/Requisicoes/Cancelamento` | SRM → ERP | Agendado | 30 min |
| 19 | Reprocessar cancelamento de requisição | `POST /api/v1.0/Requisicoes/HabilitarRetornarCancelamento` | SRM → ERP | Manual / erro | Sob demanda |
| 20 | Requisições em negociação | `GET /api/v1.0/Requisicoes/Negociacao` | SRM → ERP | Agendado | 30 min |

### 5.4. Contratos

| # | Fluxo | Método | Sentido | Gatilho | Periodicidade |
|---|---|---|---|---|---|
| 21 | Consumo de contrato aprovado | `GET /api/v1.0/Contratos/Contrato` | SRM → ERP | Agendado | 1 h |
| 22 | Reprocessar contrato | `POST /api/v1.0/Contratos/HabilitarContrato` | SRM → ERP | Manual / erro | Sob demanda |
| 23 | Devolver contrato como ativo | `PATCH /api/v1.0/Contratos/{nCdContrato}/ContratoAtualizarParcial` | ERP → SRM | Evento (assinatura concluída) | Imediato |
| 24 | Contrato encerrado | `GET /api/v1.0/Contratos/Encerrado` | SRM → ERP | Agendado | Diária |
| 25 | Contrato rescindido | `GET /api/v1.0/Contratos/Rescindido` | SRM → ERP | Agendado | Diária |

### 5.5. Pedidos de compra

| # | Fluxo | Método | Sentido | Gatilho | Periodicidade |
|---|---|---|---|---|---|
| 26 | Pedidos em processo de integração | `GET /api/v1.0/Pedidos/TodosEmProcessoDeIntegracao` | SRM → ERP | Agendado | 15 min |
| 27 | Reprocessar pedido | `PUT /api/v1.0/Pedidos/HabilitarRetornarPedidoEmProcessoDeIntegracao` | SRM → ERP | Manual / erro | Sob demanda |
| 28 | Atualizar situação do pedido | `PATCH /api/v1.0/Pedidos/{nCdPedido}/PedidoAtualizarParcial` | ERP → SRM | Evento (mudança de situação) | Imediato |
| 29 | Pedido cancelado pelo fornecedor | `GET /api/v1.0/Pedidos/Cancelamento` | SRM → ERP | Agendado | 30 min |
| 30 | Reprocessar cancelamento de pedido | `POST /api/v1.0/Pedidos/HabilitarRetornarCancelamento` | SRM → ERP | Manual / erro | Sob demanda |

### 5.6. Financeiro e fiscal

| # | Fluxo | Método | Sentido | Gatilho | Periodicidade |
|---|---|---|---|---|---|
| 31 | Títulos financeiros (pagamentos) | `POST /api/v1.0/Titulos` | ERP → SRM | Evento + agendado | Diária |
| 32 | Espelho de nota fiscal | `POST /api/v1.0/NotasFiscais` | ERP → SRM | Evento (lançamento de NF) | Imediato |

> **Pendência P-03:** as periodicidades acima são **sugestões** baseadas na criticidade de cada fluxo. Devem ser validadas pela Bernoulli contra a volumetria real e a janela operacional disponível. Ver §17.

---

## 6. Req. 01 — Parâmetros de Configuração

Antes da execução de qualquer fluxo, os parâmetros abaixo devem estar configurados na camada de integração.

### 6.1. Conexão

| Parâmetro | Descrição | Obrigatório |
|---|---|---|
| URL base do ambiente | Endpoint raiz da API SRM 360 | Sim |
| Credencial de acesso | Conforme método de autenticação definido (P-02) | Sim |
| Timeout de requisição | Tempo máximo de espera por resposta | Sim |
| Número de tentativas (retry) | Tentativas automáticas antes de marcar erro | Sim |
| Intervalo entre tentativas | Espaçamento entre retentativas | Sim |

### 6.2. Parâmetros de negócio no RM

| Parâmetro | Descrição |
|---|---|
| Coligada padrão | Coligada de referência para os documentos integrados |
| Regra de busca em coligada global | Procurar o registro na coligada informada e, não encontrando, na coligada global (0). Ver §6.3 |
| Tipo de movimento por origem de pedido | Tipo de movimento RM (`TTMV`) a ser criado para cada uma das seis origens de pedido (§12.1) |
| Tipo de movimento de solicitação de compra | Movimento correspondente à SC integrada |
| Tipo de movimento de ordem de compra | Movimento correspondente à OC |
| Transportadora genérica | Cadastro genérico usado quando o portal não informa transportadora |
| De/para de situações | Tabela de correspondência entre situações do portal e status do RM (§13.3) |
| De/para de taxas | Correspondência entre `nCdTaxa` do portal e `CODTRB` do RM |

### 6.3. Regra de resolução em coligada

> Para todo campo que o portal enviar com identificação de coligada (cliente, produto, filial, centro de custo e demais), a identificação no TOTVS ocorre **primeiro na coligada retornada** e, caso não encontrada, verifica-se a existência na **coligada global (0)**.

**Exemplos:**

| Campo retornado | Resolução |
|---|---|
| `Produto = "2\|0000123"` | Verifica o produto na coligada 2; não existindo, verifica `0000123` na coligada 0 (global) |
| `Fornecedor = "FO\|0\|0000039020"` | Verifica o cliente/fornecedor apenas na coligada 0 (global) |

**Consequência em caso de falha:** o documento **não é integrado** e um log é gravado. Não há integração parcial — ver §16.2.

### 6.4. Persistência dos parâmetros

Os parâmetros de negócio são gravados **por coligada**, em tabela própria da camada de integração.

---

## 7. Req. 02 — Integração de Cadastros (Dados Mestres)

### 7.1. Princípio

O ERP é responsável por gerenciar os dados mestres transacionados na plataforma. As informações partem do ERP e são enviadas ao SRM 360. A única exceção é o cadastro de fornecedores, que possui fluxo bidirecional (§8).

### 7.2. Modos de sincronização

| Modo | Descrição | Quando usar |
|---|---|---|
| **Carga inicial** | Envio em lote de todos os registros ativos | Implantação, e após inclusão de nova entidade no escopo |
| **Gatilho de alteração** | Envio automático a cada inclusão ou alteração no ERP | Operação corrente |

A carga inicial deve oferecer três opções ao usuário:

- **Sincronizar todos os registros** — todos os registros ativos são enviados ao portal.
- **Sincronizar somente os não enviados** — apenas os ativos ainda não enviados em carga anterior.
- **Selecionar registros** — o usuário escolhe manualmente quais registros ativos enviar.

### 7.3. Regra de concatenação

As seguintes entidades devem ser enviadas com **"código + descrição" concatenados** no campo identificador:

- Empresas (compradoras e fornecedoras)
- Centros de custo
- Contas contábeis
- Projetos

> **Pendência P-04:** definir o separador e o formato exato da concatenação (ex.: `001 - ADMINISTRATIVO`), e o comportamento quando a descrição é alterada no ERP — se o identificador muda (quebrando o de/para) ou se o código isolado permanece como chave. Ver §17.

### 7.4. Detalhamento por entidade

| Entidade | Método | Chave no ERP | Observações |
|---|---|---|---|
| **Empresas compradoras** | `POST /api/v1.0/Empresas` | Coligada + Filial | Concatenar código + descrição |
| **Empresas fornecedoras** | `POST /api/v1.0/Empresas` | Coligada + CODCFO | Mesmo endpoint das compradoras; ver fluxo bidirecional em §8 |
| **Usuários** | `POST /api/v1.0/Usuarios` | Código do usuário | **Não são integrados todos os usuários** — a Bernoulli indica quais, via carga inicial |
| **Categorias de produto** | `POST /api/v1.0/Categorias` | Código da tabela de classificação | Corresponde à Classificação de Produto do RM; campo **obrigatório** para a integração de produtos |
| **Empresa x Categoria** | `POST /api/v1.0/EmpresasCategorias` | Fornecedor + Categoria | Vínculo obrigatório para que o fornecedor possa ser selecionado em processo de cotação. Não preenchido no ERP, deve ser complementado no portal |
| **Unidades de medida** | `POST /api/v1.0/UnidadesMedidas` | CODUND | — |
| **Itens (produto/serviço)** | `POST /api/v1.0/Produtos` | Coligada + CODIGOPRD | Exige categoria de produto preenchida |
| **Condições de pagamento** | `POST /api/v1.0/CondicoesPagamentos` | CODCPG | **Não são integradas todas** — a Bernoulli indica quais, via carga inicial |
| **Moedas** | `POST /api/v1.0/Moedas` | Código da moeda | — |
| **Cotação de moeda** | `POST /api/v1.0/MoedasCotacoes` | Moeda + data | Envio diário |
| **Centros de custo** | `POST /api/v1.0/CentrosCustos` | Coligada + CODCCUSTO | Concatenar código + descrição |
| **Contas contábeis** | `POST /api/v1.0/ContasContabeis` | Conta | Concatenar código + descrição |
| **Projetos** | `POST /api/v1.0/Projetos` | Código do projeto | Concatenar código + descrição |

### 7.5. Regras gerais

1. **Não há sincronização de exclusão.** O portal não trabalha com exclusão de cadastros básicos. Registros já integrados **não podem ser excluídos** no ERP — devem ser inativados.
2. Após a carga inicial, toda inclusão ou alteração no ERP é integrada automaticamente, mantendo sincronia e espelhamento, **desde que os gatilhos estejam ativos**.
3. A ausência de um dado mestre no portal impede o uso da entidade dependente nos processos de compra (ex.: fornecedor sem vínculo de categoria não pode participar de cotação).

---

## 8. Req. 03 — Homologação de Fornecedores (fluxo bidirecional)

### 8.1. Visão geral

Este é o **único fluxo de cadastro com sentido duplo**. O cadastro de fornecedores ocorre na plataforma SRM 360, mas a base inicial e o registro definitivo vivem no ERP.

```
ERP ──── carga inicial de fornecedores saneados ────> SRM 360
                                                          │
                          novo cadastro / alteração / renovação (fornecedor)
                                                          │
                                                     análise cadastral
                                                          │
ERP <──── GET /Empresas/AtivadaInativada ────────────  homologado (ativado)
```

### 8.2. Origem ERP → SRM 360

Fornecedores existentes, saneados no ERP e aptos a interagir nos processos de compra são enviados via `POST /api/v1.0/Empresas` (§7.4).

### 8.3. Origem SRM 360 → ERP

Três tipos de operação ocorrem no portal:

| Operação | Descrição |
|---|---|
| **Novo cadastro** | Fornecedor acessa a área de pré-cadastro, informa dados gerais, categorias e documentos. Recebe e-mail de protocolo; cadastro fica em *"Aguardando homologação"* |
| **Alteração cadastral** | Fornecedor atualiza dados e documentos; recebe protocolo de acompanhamento |
| **Renovação cadastral** | Fornecedor renova dados e documentos; recebe protocolo de acompanhamento |

Concluída a etapa pelo fornecedor, o processo entra em fila de análise cadastral, com **quatro desfechos possíveis**:

| Desfecho | Integra com o ERP? |
|---|---|
| Retorno de cadastro para o fornecedor (complementação) | Não — permanece em análise |
| Cadastro reprovado | Não |
| Cadastro aprovado | **Sim** — disponibilizado via `GET /api/v1.0/Empresas/AtivadaInativada` |
| Cadastro aprovado com ressalva | **Sim** — aprovado, com ressalvas registradas no cadastro |

### 8.4. Fornecedores da rede Clicbusiness

Compradores podem buscar fornecedores na rede Clicbusiness e adicioná-los ao portal como **convidados**. O fornecedor convidado só passa a integrar com o ERP após homologação cadastral. **Somente após ativado entre os dois sistemas** o fornecedor pode receber pedidos da negociação realizada.

### 8.5. Fornecedores sem de/para

O endpoint `GET /api/v1.0/Empresas/SemDePara` retorna empresas recém-cadastradas no portal que ainda **não possuem correspondência estabelecida com o ERP**. Serve como mecanismo de conciliação: permite ao integrador identificar registros órfãos e criar o vínculo, evitando duplicidade de fornecedor no RM.

### 8.6. Ação no ERP

Ao consumir um fornecedor ativado, o integrador deve:

1. Verificar a existência do fornecedor no RM pelo CNPJ/CPF (aplicando a regra de coligada de §6.3).
2. **Não existindo:** criar o cadastro de Cliente/Fornecedor no RM com os dados retornados.
3. **Existindo:** atualizar os dados alterados e reativar o cadastro se estiver inativo.
4. Registrar o de/para entre o identificador do portal e `CODCOLIGADA|CODCFO`.
5. Sinalizar que o cadastro pode exigir **complementação manual posterior** de informações fiscais e contábeis dentro do ERP.

> **Pendência P-05:** definir a lista de campos que o portal envia no retorno de fornecedor e quais deles são suficientes para criar um cadastro válido no RM sem intervenção manual. Ver §17.

---

## 9. Req. 04 — Solicitação de Compra e Requisições

### 9.1. Fluxo

1. A solicitação de compra é **criada no RM**.
2. Passa pelo processo de aprovação e **check orçamentário** no RM.
3. **Sem orçamento previsto:** o time solicitante/compras busca suplementação de budget no centro de custo, no ERP.
4. **Não havendo previsão de incremento:** a SC é **cancelada e não é enviada** ao portal.
5. **SC aprovada com check orçamentário:** disponibilizada para integração com o portal.
6. **SC não aprovada no RM:** não é enviada ao portal.

> Regra determinante: **somente solicitações aprovadas e com orçamento validado no RM chegam ao SRM 360.** A plataforma não recebe SC pendente de aprovação.

### 9.2. Envio — ERP → SRM 360

**Método:** `POST /api/v1.0/OrdensCompras`
**Gatilho:** evento, ao atingir a situação de aprovada com check orçamentário.

### 9.3. Retornos — SRM 360 → ERP

| Fluxo | Método | Efeito no ERP |
|---|---|---|
| **Requisição em negociação** | `GET /api/v1.0/Requisicoes/Negociacao` | Informa que o item está vinculado a um processo de negociação na plataforma. O ERP registra a situação, bloqueando novo processo de compra para o mesmo item |
| **Requisição cancelada** | `GET /api/v1.0/Requisicoes/Cancelamento` | Cancela a requisição correspondente no RM |
| **Reprocessamento** | `POST /api/v1.0/Requisicoes/HabilitarRetornarCancelamento` | Recoloca o cancelamento na fila |

### 9.4. Solicitação do tipo "Regularização"

Fluxo específico da Bernoulli, com tratamento diferenciado:

1. A SC de regularização segue o mesmo fluxo de aprovação e envio ao portal.
2. No portal, o comprador realiza a regularização **representando o fornecedor na cotação**.
3. O resultado é um **pedido de compra do tipo regularização**.
4. O integrador consome o arquivo de pedido normalmente (§12).
5. Faz o input do pedido no ERP.
6. **Devolve o pedido diretamente com a situação "Recebido total"** — pulando as situações intermediárias, pois a mercadoria/serviço já foi entregue.

> **Pendência P-06:** confirmar se o pedido de regularização traz marcação própria no payload que permita ao integrador identificá-lo automaticamente, ou se a identificação depende do tipo da SC de origem. Ver §17.

---

## 10. Req. 05 — Contratos

### 10.1. Ciclo de vida

```
Em configuração ──> Aprovação (alçada padrão) ──> Disponível para integração
                                                            │
                                             GET /Contratos/Contrato
                                                            ▼
                                                  Camada de integração
                                                            │
                                                            ▼
                                                    Registrado no RM
                                                            │
                                        minuta + coleta de assinaturas (ERP)
                                                            │
                     PATCH /Contratos/{n}/ContratoAtualizarParcial → "Ativo"
                                                            ▼
                                              Contrato ativo (vigência conta)
                                                            │
                                       ┌────────────────────┴──────────────┐
                                       ▼                                   ▼
                          GET /Contratos/Encerrado            GET /Contratos/Rescindido
                                       └─────────────────┬─────────────────┘
                                                         ▼
                                               Camada de integração
                                                         │
                                        baixa / encerramento do contrato
                                                         ▼
                                                     TOTVS RM
```

### 10.2. Regras

1. Contratos gerados no SRM 360 (por negociação ou manualmente) nascem na situação **"Em configuração"**.
2. Permanecem assim até passarem pela **aprovação por alçada padrão da plataforma**.
3. Aprovado, o portal **disponibiliza o contrato para integração** com o ERP.
4. Após integrado, a Bernoulli executa seus processos internos de elaboração de minuta e coleta de assinaturas no seu ecossistema.
5. Concluída essa etapa, **o ERP devolve a situação "Contrato ativo"** ao portal.
6. **A vigência do contrato no portal passa a ser contada somente a partir desse retorno**, de acordo com o período estipulado na configuração do contrato no portal.

> Ponto de atenção: enquanto o ERP não devolver a situação "ativo", o contrato não entra em vigência no portal e **não gera pedidos**. Falha ou atraso nesse retorno paralisa o consumo do contrato.

### 10.3. Métodos

| Fluxo | Método | Sentido |
|---|---|---|
| Consumo do contrato aprovado | `GET /api/v1.0/Contratos/Contrato` | SRM → ERP |
| Reprocessar um contrato específico | `POST /api/v1.0/Contratos/HabilitarContrato` | SRM → ERP |
| Devolver situação "ativo" | `PATCH /api/v1.0/Contratos/{nCdContrato}/ContratoAtualizarParcial` | ERP → SRM |
| Contrato encerrado no portal | `GET /api/v1.0/Contratos/Encerrado` | SRM → ERP |
| Contrato rescindido no portal | `GET /api/v1.0/Contratos/Rescindido` | SRM → ERP |

### 10.4. Tipos de atendimento e impacto na integração

O tipo de consumo definido no item do contrato determina **se o pedido gerado terá ou não item de requisição associado** — o que muda o tratamento no ERP (§12.3):

| Tipo de atendimento | Geração do pedido | Item de requisição? | Validação orçamentária no ERP |
|---|---|---|---|
| **Por demanda** | Atendimento de solicitação de compra: o comprador decide se a SC será consumida do contrato | **Sim** | Não requerida (já validada na SC) |
| **Entrega programada** | Automática, com antecedência configurada no campo "Notificação de pedido (dias)" da capa do contrato, conforme programação do item | **Não** | **Requerida** |
| **Medição** | A partir de medição lançada e **aprovada pelo Gestor** nomeado na capa do contrato ativo | **Não** | **Requerida** |

**Notas sobre medição:**
- O contrato precisa estar **ativo** para permitir lançamento de medições.
- A medição pode ser por **valor** ou por **quantidade**, conforme configurado no item.
- Medição não aprovada fica com status "não aprovada", **não consome saldo do contrato e não gera pedido**.
- A aprovação da medição ocorre **no portal**, antes de o documento se tornar pedido — portanto antes de qualquer integração.

**Nota sobre entrega programada:**
- O contrato precisa estar **em configuração** para definir a entrega programada.
- A periodicidade pode ser em dia fixo do mês ou em dias corridos, com opção de pular finais de semana.

---

## 11. Req. 06 — Catálogo

**O catálogo não é integrado com o ERP.**

O ciclo de configuração, aprovação, vigência e consumo (loja virtual, carrinho, fechamento) ocorre integralmente na plataforma SRM 360. A integração começa apenas quando o fechamento do carrinho **gera um pedido de compra**, que segue o fluxo do §12.

**Característica relevante para o ERP:** pedidos gerados a partir de catálogo **não possuem item de requisição associado** e, portanto, passam obrigatoriamente pela validação orçamentária no RM (§12.3).

---

## 12. Req. 07 — Pedido de Compra

Este é o fluxo central da integração.

### 12.1. Origens do pedido

O pedido de compra é **sempre gerado na plataforma SRM 360**, com número interno próprio, a partir de seis origens:

| # | Origem | Item de requisição associado? |
|---|---|---|
| 1 | Pedido de cotação | **Sim** |
| 2 | Pedido de leilão | **Sim** |
| 3 | Pedido de contrato por demanda de requisição | **Sim** |
| 4 | Pedido de catálogo | **Não** |
| 5 | Pedido de medição de contrato | **Não** |
| 6 | Pedido de entrega programada de contrato | **Não** |

> Regra de origem no portal: cotações e leilões **do tipo "Pedido"** só admitem itens de requisição de compra. Itens sem requisição são permitidos apenas em negociações de contrato, catálogo ou orçamento.

### 12.2. Fluxo de integração

**Passo 1 — Consumo do pedido**
O integrador consome os pedidos pendentes via `GET /api/v1.0/Pedidos/TodosEmProcessoDeIntegracao` e grava os dados no ERP.

Caso o arquivo tenha sido consumido e precise voltar à fila:
`PUT /api/v1.0/Pedidos/HabilitarRetornarPedidoEmProcessoDeIntegracao`

**Passo 2 — Confirmação de recebimento**
Assim que o ERP consumir o pedido, deve informar ao portal, via
`PATCH /api/v1.0/Pedidos/{nCdPedido}/PedidoAtualizarParcial`:

- Situação **"Em análise financeira"** — **na capa e nos itens**
- O **número sequencial do pedido no ERP**

**Passo 3 — Bifurcação orçamentária** (ver §12.3)

**Passo 4 — Gestão de entregas**
Após a situação "Confirmado", o recebimento ocorre unicamente no ERP, via lançamento de notas fiscais. As mudanças de situação (**Recebido parcial** / **Recebido total**) são enviadas ao portal pelo mesmo `PATCH /api/v1.0/Pedidos/{nCdPedido}/PedidoAtualizarParcial`, **na capa e nos itens**.

### 12.3. Regra de bifurcação orçamentária

```
                     Pedido consumido do portal
                                │
                    situação → "Em análise financeira"
                                │
                    ┌───────────┴───────────┐
                    │  Tem requisição       │
                    │  associada?           │
                    └───────────┬───────────┘
              Sim ──────────────┴────────────── Não
               │                                 │
               │                    Validação orçamentária no ERP
               │                                 │
               │                    ┌────────────┴────────────┐
               │              Tem orçamento            Não tem orçamento
               │                    │                         │
               │                    │              suplementação de budget
               │                    │                         │
               │                    │              ┌──────────┴─────────┐
               │                    │         Suplementado      Não suplementado
               ▼                    ▼              ▼                    ▼
         "Confirmado"         "Confirmado"   "Confirmado"         "Cancelado"
```

Em todos os casos, o retorno usa `PATCH /api/v1.0/Pedidos/{nCdPedido}/PedidoAtualizarParcial`, atualizando **capa e itens** e **mantendo o número sequencial do pedido do ERP**.

### 12.4. Situações do pedido

| Situação | Origem | Como é comunicada |
|---|---|---|
| Em processo de integração | SRM 360 | Estado do documento na fila de `TodosEmProcessoDeIntegracao` |
| **Em análise financeira** | ERP | `PATCH .../PedidoAtualizarParcial` |
| **Confirmado** | ERP | `PATCH .../PedidoAtualizarParcial` |
| **Cancelado** | ERP ou fornecedor | `PATCH` (ERP) ou `GET /Pedidos/Cancelamento` (fornecedor) |
| **Recebido parcial** | ERP | `PATCH .../PedidoAtualizarParcial` |
| **Recebido total** | ERP | `PATCH .../PedidoAtualizarParcial` |

### 12.5. Reconfiguração de pedido

Enquanto estiver **"Confirmado"**, o pedido pode ser reconfigurado pelo comprador no portal. **Apenas dois campos são alteráveis:**

- Data de entrega do item
- Quantidade do item

Ocorrendo alteração em qualquer um deles, o pedido **retorna à situação "Em processo de integração"** e reaparece em `GET /api/v1.0/Pedidos/TodosEmProcessoDeIntegracao`, para que o integrador colete o arquivo atualizado e faça o input da atualização no RM.

> Ponto de atenção para a camada de integração: um pedido pode ser consumido **mais de uma vez**. O integrador deve tratar a reentrada como **atualização** do movimento existente no RM (localizado pelo número do pedido no portal), nunca como novo movimento. Ver §16.3.

### 12.6. Cancelamento pelo fornecedor

1. O fornecedor cancela o pedido no portal.
2. O documento retorna ao ERP via `GET /api/v1.0/Pedidos/Cancelamento`.
3. O integrador **cancela o pedido no ERP**.
4. Havendo falha no consumo, o arquivo pode ser redisponibilizado por `POST /api/v1.0/Pedidos/HabilitarRetornarCancelamento`.

**Desdobramento sobre a requisição** — o comprador tem três caminhos:

| Decisão do comprador | Efeito |
|---|---|
| Retomar a negociação / escolher outro fornecedor vencedor | Item segue no processo de compra no portal |
| **Liberar manualmente** o item para novo processo de compra | Item volta a ficar disponível para negociação |
| **Não liberar** o item, justificando a motivação | O sistema **cancela a requisição**, e o cancelamento é consumido pelo ERP via `GET /api/v1.0/Requisicoes/Cancelamento` |

---

## 13. Req. 08 — Títulos Financeiros

**Sentido:** ERP → SRM 360
**Método:** `POST /api/v1.0/Titulos`

A gestão dos pagamentos é feita integralmente no ERP. A plataforma SRM 360 publica os pagamentos agendados/efetuados, permitindo consulta pelo comprador, pelo fornecedor e pela área financeira.

**Regras:**
- À medida que ocorrerem atualizações no título financeiro, as situações são enviadas **no mesmo sentido** (ERP → Portal).
- O módulo prevê a conciliação de **antecipações de pagamento** com o posterior lançamento de notas fiscais e o **vínculo com o pedido** no título financeiro.

> **Pendência P-07:** definir o conjunto de situações de título a serem publicadas e o campo de vínculo entre o título e o pedido do portal. Ver §17.

---

## 14. Req. 09 — Notas Fiscais (Espelho)

**Sentido:** ERP → SRM 360
**Método:** `POST /api/v1.0/NotasFiscais`

O módulo trabalha com o **espelho de nota fiscal**: representação simplificada da NF-e contendo emitente, destinatário, itens, valores e impostos.

**Natureza do documento:** o espelho é utilizado **exclusivamente para fins operacionais e de controle interno**. Não substitui o documento fiscal oficial e **não possui valor legal**.

**Gatilho:** lançamento da nota fiscal no ERP — o mesmo evento que atualiza a situação de recebimento do pedido (§12.2, passo 4). Recomenda-se que ambas as chamadas sejam executadas na mesma transação lógica, para evitar divergência entre a situação do pedido e a existência do espelho de NF no portal.

---

## 15. Mapeamento de Campos

### 15.1. Nota metodológica

O mapeamento abaixo relaciona os elementos do documento no portal às tabelas e colunas correspondentes do TOTVS RM.

> **Regra editorial deste documento:** campos cujo nome no contrato REST do SRM 360 ainda não foi publicado estão marcados como `A DEFINIR`. **Nenhum nome de campo foi inferido ou inventado.** O preenchimento dessas lacunas depende da entrega da documentação de API pela Paradigma (Pendência P-08).

### 15.2. Pedido — Capa

| Elemento | Tipo | Tam. | Descrição | Tabela RM | Coluna RM |
|---|---|---|---|---|---|
| `sCdPedidoWbc` | String | 20 | Cód. da negociação que gerou o pedido | TMOVCOMPL | PEDIDOWBC |
| `nIdTipoOrigem` | Int | — | Tipo/origem da negociação | — | Define o tipo de movimento (§6.2) |
| `sCdPedidoErp` | String | 20 | Número do pedido no ERP | TMOV | CODCOLIGADA \| IDMOV |
| `sCdFornecedor` | String | 14 | Empresa fornecedora | TMOV | `'FO'`\|CODCOLIGADA\|CODCFO |
| `sCdComprador` | String | 14 | Empresa compradora | TMOV | `'FI'`\|CODCOLIGADA\|CODFILIAL |
| `sCdTransportadora` | String | 14 | Transportadora | TMOV | `'TR'`\|CODCOLIGADA\|CODTRA |
| `sCdFrete` | String | 1 | Tipo do frete: C = CIF, F = FOB | TMOV | FRETECIFOUFOB |
| `nCdSituacao` | Int | — | Código da situação | TMOV | Ver de/para em §15.5 |
| `sCdCondicaoPagamento` | String | 60 | Condição de pagamento | TMOV | CODCPG |
| `sDsObservacoes` | String | 4000 | Observações | TMOV | OBSERVACAO |
| `sCdUsuario` | String | 100 | Usuário responsável | TMOV | CODUSUARIO |
| `tDtCadastro` | DateTime | — | Data de inclusão | TMOV | DATACRIACAO |
| `sCdCentroCusto` | String | 20 | Centro de custo | TMOV | CODCOLIGADA \| CODCCUSTO |
| `nCdTipo` | Int | — | Tipo do pedido (1 = Pedido Normal) | TMOV | Valor fixo = 1 |
| `tDtEmissao` | DateTime | — | Data de emissão | TMOV | DATAEMISSAO |
| `tDtFaturamento` | DateTime | — | Data de faturamento | TMOV | *A DEFINIR* |
| `dVlTotal` | Decimal | — | Valor total | TMOV | VALORLIQUIDO |
| `sCdMoeda` | String | 1 | Moeda: 1=Real, 2=Dólar, 3=Euro, 4=Peso mexicano | TMOV | *A DEFINIR* |
| `nNrVersao` | Int | — | Número da versão do pedido | — | Relevante para reconfiguração (§12.5) |
| `sCdUsuarioProgramador` | String | — | Usuário responsável pela programação de entrega | — | *A DEFINIR* |
| — | — | — | **Conta contábil** (novo no escopo) | — | *A DEFINIR* |
| — | — | — | **Projeto** (novo no escopo) | — | *A DEFINIR* |

### 15.3. Pedido — Itens

| Elemento | Tipo | Descrição | Tabela RM | Coluna RM |
|---|---|---|---|---|
| `sCdProduto` | String | Produto | TITMMOV / TPRODUTO | CODCOLIGADA \| CODIGOPRD |
| `sDsItem` | String | Descrição do item do pedido | TPRODUTO | NOMEFANTASIA |
| `sCdClasse` | String | Categoria de produto | — | CODCOLIGADA \| CODTTB*FAT |
| `sCdUnidadeMedida` | String | Unidade de medida | TITMMOV | CODUND |
| `dQtItem` | Decimal | Quantidade | TITMMOV | QUANTIDADE |
| `dVlItem` | Decimal | Valor unitário do item | TITMMOV | PRECOUNITARIO |
| `sDsObservacao` | String | Observação | TITMMOVHISTORICO | HISTORICOCURTO |
| `dPcDesconto` | Decimal | Percentual de desconto | TITMMOV | PERCENTUALDESC ou percentual calculado a partir de VALORDESC |
| `nSqItemMov` | — | Sequencial do item | TITMMOV | NSEQITMMOV |
| `sCdItemWbc` | String | Cód. do item da negociação que gerou o pedido | — | **Não enviado** — estrutura de rastreabilidade incompatível entre os sistemas |
| `sCdOrigemEmpresa` | String | Cód. da requisição que gerou o item | — | **Não enviado** — idem |
| `sCdItemOrigemEmpresa` | String | Cód. do item da requisição que gerou o item | — | **Não enviado** — idem |
| `sCdItemEmpresa` | String | Cód. do item do pedido no ERP | — | **Não enviado** — idem |

> **Ponto de atenção crítico — P-09:** no escopo deste projeto — com recebimento parcial por item, medição de contrato e reconfiguração de item — **a ausência de vínculo item-a-item entre portal e ERP pode inviabilizar a atualização correta de situação por item**, exigida em §12.2. Este ponto precisa ser reavaliado tecnicamente antes do desenvolvimento. Ver §17.

`*` `CODTTB*FAT`: código da tabela de classificação parametrizada como categoria de produto.

### 15.4. Pedido — Taxas e entregas

| Elemento | Descrição | Tabela RM | Coluna RM |
|---|---|---|---|
| Taxa do item — `nCdTaxa` | Código da taxa | TTRBMOV | De/para entre códigos do portal e CODTRB |
| Taxa do item — `bFlIncluso` | Taxa inclusa no valor da proposta (0=Não, 1=Sim) | TTRBMOV | Fixo 1 |
| Taxa do item — `dPcTaxa` | Percentual da taxa | TTRBMOV | ALIQUOTA |
| Entrega do item — `dQtEntrega` | Quantidade de entrega | TITMMOV | QUANTIDADE |
| Entrega do item — `tDtEntrega` | Data de entrega | TMOV | DATAENTREGA |
| Entrega do item — `sCdEmpresaEntregaEndereco` | Empresa do endereço de entrega | TMOV | `'FI'`\|CODCOLIGADA\|CODFILIAL |
| Entrega do item — `sCdEmpresaCobrancaEndereco` | Empresa do endereço de cobrança | TMOV | `'FI'`\|CODCOLIGADA\|CODFILIAL |
| Entrega do item — `sCdEmpresaFaturamentoEndereco` | Empresa do endereço de faturamento | TMOV | `'FI'`\|CODCOLIGADA\|CODFILIAL |

### 15.5. De/para de situações

| Situação no SRM 360 | Momento | Situação/status no RM |
|---|---|---|
| Em processo de integração | Pedido disponível na fila | *A DEFINIR* |
| Em análise financeira | Consumido pelo ERP | *A DEFINIR* |
| Confirmado | Orçamento validado | *A DEFINIR* |
| Cancelado | Sem orçamento, ou cancelado pelo fornecedor | *A DEFINIR* |
| Recebido parcial | NF parcial lançada | *A DEFINIR* |
| Recebido total | NF total lançada | *A DEFINIR* |

> **Pendência P-10:** construir a tabela de de/para de situações em conjunto com a equipe funcional da Bernoulli, considerando os status nativos de movimento do RM e os campos de controle da integração. Ver §17.

### 15.6. Demais entidades

O mapeamento campo a campo das demais entidades — cadastros mestres, solicitação de compra, contrato, título financeiro e nota fiscal — será detalhado **após a entrega da documentação de contratos da API REST pela Paradigma** (P-08), seguindo o mesmo formato tabular acima.

---

## 16. Tratamento de Erros, Log e Reprocessamento

### 16.1. Mecanismo oficial de reprocessamento

A plataforma disponibiliza quatro endpoints `Habilitar*`, que recolocam na fila documentos já consumidos:

| Fila | Endpoint de reprocessamento |
|---|---|
| Pedidos em processo de integração | `PUT /api/v1.0/Pedidos/HabilitarRetornarPedidoEmProcessoDeIntegracao` |
| Cancelamento de pedido | `POST /api/v1.0/Pedidos/HabilitarRetornarCancelamento` |
| Cancelamento de requisição | `POST /api/v1.0/Requisicoes/HabilitarRetornarCancelamento` |
| Contrato | `POST /api/v1.0/Contratos/HabilitarContrato` |
| Retorno de empresa ativada/inativada | `GET /api/v1.0/Empresas/HabilitarRetornarEmpresaAtivadaInativada` |

**Diretriz:** o integrador deve tratar o consumo e a gravação no ERP como uma **transação única**. Falhando a gravação, o documento deve ser reposto na fila pelo endpoint correspondente, e não descartado.

### 16.2. Regra de integralidade

> **Não há integração parcial.** O documento só é incluído no ERP quando **todos** os seus campos e referências forem identificados. Caso algum produto, fornecedor, centro de custo ou demais referências não seja localizado no TOTVS (aplicada a regra de coligada de §6.3), **o documento inteiro não é integrado** e um log é gravado.

### 16.3. Idempotência

Dois cenários exigem tratamento explícito de idempotência:

| Cenário | Tratamento exigido |
|---|---|
| **Reconfiguração de pedido** (§12.5) | O mesmo pedido reaparece na fila. O integrador deve localizar o movimento existente pelo número do pedido no portal e **atualizar**, nunca criar novo movimento |
| **Reprocessamento manual** (§16.1) | Documento já parcialmente gravado antes da falha. A gravação deve ser idempotente pela chave de correlação |

**Chave de correlação por documento:**

| Documento | Chave no portal | Chave no ERP |
|---|---|---|
| Pedido | `nCdPedido` | TMOVCOMPL.PEDIDOWBC ↔ TMOV.CODCOLIGADA \| IDMOV |
| Contrato | `nCdContrato` | *A DEFINIR* |
| Requisição / SC | *A DEFINIR* | TMOV.CODCOLIGADA \| IDMOV |
| Empresa | Identificador do portal | CODCOLIGADA \| CODCFO |

### 16.4. Log

A camada de integração deve registrar, para cada tentativa:

- Data/hora, endpoint e sentido
- Identificador do documento nos dois sistemas
- Payload enviado/recebido (ou referência a ele)
- Resultado: sucesso, erro de negócio (referência não localizada) ou erro técnico (indisponibilidade, timeout)
- Em caso de erro de negócio: **qual referência específica não foi localizada**, para permitir correção do cadastro

O log deve ser consultável pela equipe de compras, não apenas pela TI — a maior parte dos erros esperados é de cadastro, e sua correção é responsabilidade da área de negócio.

### 16.5. Cenários de indisponibilidade

| Cenário | Comportamento esperado |
|---|---|
| API SRM 360 indisponível no envio (ERP → SRM) | Retentativa conforme parâmetros de §6.1; persistindo, enfileirar localmente e alertar |
| API SRM 360 indisponível no consumo (SRM → ERP) | Nenhum documento é perdido — a fila permanece no portal |
| ERP indisponível após consumo | Repor documento na fila via endpoint `Habilitar*` |
| Falha parcial em lote de cadastros | Registrar por registro; não abortar o lote inteiro |

---

## 17. Pendências para Definição

Consolidação de todos os pontos que **dependem de definição** antes ou durante o desenvolvimento. Este é o checklist de resposta esperado da Bernoulli e da Paradigma.

| # | Pendência | Responsável | Bloqueia |
|---|---|---|---|
| **P-01** | Definição do responsável técnico pela camada de integração do lado RM (customização TOTVS ou middleware próprio) | Bernoulli | Início do desenvolvimento |
| **P-02** | URLs base dos ambientes, método de autenticação, política de credencial, rate limit e paginação | Paradigma | Desenvolvimento |
| **P-03** | Validação das periodicidades sugeridas na matriz (§5) contra volumetria real e janela operacional | Bernoulli | Configuração |
| **P-04** | Formato exato da concatenação "código + descrição" e comportamento na alteração da descrição | Paradigma | Cadastros |
| **P-05** | Lista de campos retornados no fluxo de fornecedor homologado e suficiência para criar cadastro válido no RM | Paradigma + Bernoulli | Req. 03 |
| **P-06** | Identificação automática do pedido do tipo "regularização" no payload | Paradigma | Req. 04 |
| **P-07** | Situações de título financeiro a publicar e campo de vínculo título ↔ pedido | Bernoulli + Paradigma | Req. 08 |
| **P-08** | **Entrega da documentação de contratos da API REST** (payloads de request/response de todos os endpoints) | Paradigma | **§15 — mapeamento de campos** |
| **P-09** | **Rastreabilidade item a item** entre portal e ERP, diante do recebimento parcial por item, medição e reconfiguração | Paradigma + Bernoulli | **Req. 07** |
| **P-10** | Tabela de de/para de situações de pedido entre portal e RM | Bernoulli | Req. 07 |
| **P-11** | Confirmação da versão do TOTVS RM em produção | Bernoulli | Desenvolvimento |
| **P-12** | Definição da lista de usuários e de condições de pagamento a integrar (carga inicial seletiva) | Bernoulli | Cadastros |
| **P-13** | Volumetria estimada por entidade (registros/dia) para dimensionamento | Bernoulli | Configuração |

---

## 18. Plano de Testes e Critérios de Aceite

### 18.1. Cadastros

| # | Cenário | Critério de aceite |
|---|---|---|
| T-01 | Carga inicial de cada entidade mestre | Todos os registros ativos presentes no portal, com código + descrição no formato acordado |
| T-02 | Inclusão de produto no RM | Produto aparece no portal automaticamente, com categoria vinculada |
| T-03 | Alteração de centro de custo no RM | Alteração refletida no portal sem intervenção manual |
| T-04 | Produto sem categoria de produto | Integração rejeitada com log claro indicando o campo faltante |
| T-05 | Inativação de fornecedor no RM | Fornecedor inativado no portal, sem exclusão |

### 18.2. Fornecedores

| # | Cenário | Critério de aceite |
|---|---|---|
| T-06 | Pré-cadastro no portal → aprovação | Fornecedor retornado em `Empresas/AtivadaInativada` e criado no RM |
| T-07 | Aprovação com ressalva | Fornecedor criado no RM, com a ressalva registrada |
| T-08 | Cadastro reprovado | Fornecedor **não** retorna ao ERP |
| T-09 | Alteração cadastral homologada | Dados atualizados no RM, sem duplicar cadastro |
| T-10 | Fornecedor sem de/para | Registro aparece em `Empresas/SemDePara`; vínculo criado sem duplicidade |

### 18.3. Solicitação de compra

| # | Cenário | Critério de aceite |
|---|---|---|
| T-11 | SC aprovada com check orçamentário | SC enviada ao portal via `OrdensCompras` |
| T-12 | SC reprovada no RM | SC **não** enviada ao portal |
| T-13 | SC sem orçamento e sem suplementação | SC cancelada no RM, não enviada |
| T-14 | Item vinculado a negociação no portal | ERP recebe a situação de negociação e bloqueia novo processo para o item |
| T-15 | Requisição cancelada no portal | Requisição cancelada no RM |
| T-16 | SC de regularização, ciclo completo | Pedido de regularização gerado, integrado e devolvido como "Recebido total" |

### 18.4. Contratos

| # | Cenário | Critério de aceite |
|---|---|---|
| T-17 | Contrato aprovado no portal | Contrato consumido pelo ERP e registrado |
| T-18 | Devolução de "contrato ativo" | Vigência iniciada no portal a partir do retorno |
| T-19 | Contrato ativo com item por demanda | Pedido gerado com item de requisição associado |
| T-20 | Contrato com entrega programada | Pedido gerado automaticamente na antecedência configurada, sem item de requisição |
| T-21 | Medição aprovada pelo gestor | Pedido gerado; saldo do contrato consumido |
| T-22 | Medição não aprovada | Nenhum pedido gerado; saldo não consumido |
| T-23 | Contrato encerrado / rescindido | Situação refletida no ERP |

### 18.5. Pedidos — cenários críticos

| # | Cenário | Critério de aceite |
|---|---|---|
| T-24 | Pedido de cotação (com requisição) | Consumido, "Em análise financeira", depois "Confirmado" com número do ERP |
| T-25 | Pedido de leilão | Idem T-24 |
| T-26 | Pedido de catálogo (sem requisição) com orçamento | Passa por validação orçamentária e retorna "Confirmado" |
| T-27 | Pedido de catálogo sem orçamento e sem suplementação | Retorna "Cancelado" ao portal |
| T-28 | Pedido de medição de contrato | Validação orçamentária aplicada; situação devolvida corretamente |
| T-29 | Pedido de entrega programada | Idem T-28 |
| T-30 | Pedido de contrato por demanda | Item de requisição associado; sem nova validação orçamentária |
| T-31 | Reconfiguração de data de entrega | Pedido reaparece na fila; ERP **atualiza** o movimento existente, sem duplicar |
| T-32 | Reconfiguração de quantidade | Idem T-31 |
| T-33 | Cancelamento pelo fornecedor + liberação do item | Pedido cancelado no ERP; item liberado para novo processo |
| T-34 | Cancelamento pelo fornecedor + não liberação | Pedido cancelado; requisição cancelada e consumida pelo ERP |
| T-35 | Recebimento parcial (NF parcial) | Situação "Recebido parcial" na capa **e nos itens** do portal |
| T-36 | Recebimento total | Situação "Recebido total" na capa **e nos itens** |
| T-37 | Pedido com produto inexistente no RM | Pedido **não** integrado (nem parcialmente); log identifica o produto |
| T-38 | Reprocessamento via `Habilitar*` | Documento retorna à fila e é integrado sem duplicidade |

### 18.6. Financeiro e fiscal

| # | Cenário | Critério de aceite |
|---|---|---|
| T-39 | Publicação de título financeiro | Título visível no portal para comprador, fornecedor e financeiro |
| T-40 | Atualização de situação do título | Nova situação refletida no portal |
| T-41 | Espelho de NF | NF publicada no portal, coerente com a situação de recebimento do pedido |

### 18.7. Resiliência

| # | Cenário | Critério de aceite |
|---|---|---|
| T-42 | API indisponível durante envio | Retentativa conforme parâmetro; alerta após esgotar tentativas; nenhum dado perdido |
| T-43 | Falha no ERP após consumo da fila | Documento reposto na fila; integrado na execução seguinte |
| T-44 | Falha parcial em lote de cadastros | Registros válidos integrados; inválidos logados individualmente |

---

## 19. Glossário de Equivalência

| SRM 360 (Portal) | TOTVS RM (ERP) |
|---|---|
| Pedido de Compra | Ordem de Compra (movimento 1.1.04 na configuração atual) |
| Solicitação / Ordem de Compra (`OrdensCompras`) | Solicitação de Compra (movimento 1.1.02 na configuração atual) |
| Item de Requisição | Item de Cotação / Item de Solicitação |
| Negociação (cotação / leilão) | Cotação |
| Empresa compradora | Coligada / Filial |
| Empresa fornecedora | Cliente/Fornecedor (CODCFO) |
| Categoria de produto | Classificação de Produto |
| Item / Produto | Produto (CODIGOPRD) |
| Centro de custo | Centro de Custo (CODCCUSTO) |
| Título financeiro | Título a pagar |
| Espelho de nota fiscal | Nota fiscal de entrada |
| Situação do pedido | Status do movimento |

---

## 20. Histórico de Versões

| Data | Autor | Versão | Descrição das alterações | Requer aprovação |
|---|---|---|---|---|
| 13/08/2026 | Paradigma Business Solutions | 1.0 | Documento inicial | Sim |

---

## 21. Aprovação

Estando de pleno acordo com as informações apresentadas neste documento, as partes assinam:

<br>

| | |
|---|---|
| \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ | \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ |
| **Paradigma Business Solutions Ltda.** | **Bernoulli Educação** |
| Responsável técnico | Suprimentos |

<br>

| | |
|---|---|
| \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ | \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_ |
| **Moises Mauricio Rodrigues Souza** | **Vitor Tadeu Rosenburg Pereira** |
| Gerente de Projeto — Paradigma | Responsável no Cliente — Bernoulli |
