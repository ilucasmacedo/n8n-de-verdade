# Especificação Técnica --- Automação Groner CRM → Conta Azul via n8n

Documento da integração Leonela. O formato segue o arquivo
`Especificacao_Tecnica_Groner_ContaAzul_n8n.md`. Tenant Groner: gerasol.
Isto continua um MVP. Não é integração de produção.

------------------------------------------------------------------------

## 1. Objetivo

Construir uma automação no n8n que integre o **Groner CRM** da Leonela
ao **Conta Azul ERP**.

O objetivo de negócio é:

> Quando uma venda for fechada no Groner, a automação deve obter os
> dados completos dessa venda, transformar os dados para o formato
> aceito pelo Conta Azul, localizar ou criar o cliente necessário, criar
> a venda no Conta Azul, alimentar corretamente o financeiro/parcelas
> quando aplicável, registrar a relação entre a venda do Groner e a
> venda do Conta Azul e impedir que a mesma venda seja lançada duas
> vezes.

Fluxo conceitual:

``` text
VENDA FECHADA NO GRONER
        ↓
n8n
        ↓
Buscar dados completos da venda
        ↓
Validar / normalizar
        ↓
Verificar duplicidade
        ↓
Localizar ou criar cliente no Conta Azul
        ↓
Mapear produtos/itens
        ↓
Criar venda no Conta Azul
        ↓
Criar/configurar parcelas/financeiro
        ↓
Registrar vínculo Groner ↔ Conta Azul
        ↓
SUCESSO
```

O workflow atual é deliberadamente um **MVP/esqueleto de
desenvolvimento**. Ele não deve ser considerado uma integração de
produção até que os pontos marcados como TODO sejam implementados e
testados.

Estado nesta pasta, em 03/10/2026: o n8n local responde em
`http://localhost:5678`. O esqueleto e os rascunhos LEO existem. Nenhuma
venda real foi criada na Conta Azul por esta automação. A tabela de
vínculo ainda não existe.

------------------------------------------------------------------------

# 2. Sistemas envolvidos

## 2.1 Groner CRM

O Groner é a origem dos dados da venda. Nesta integração o tenant é
`gerasol`.

A API do Groner utiliza REST/JSON e autenticação via Bearer/JWT.

O workflow deverá utilizar a API de vendas do Groner, especialmente o
recurso relacionado à consulta de vendas por período:

``` text
/api/Venda/DadosVendaPorPeriodo
```

A URL base possui o formato:

``` text
https://{TENANT}.api.groner.app
```

Para a Leonela:

``` text
https://gerasol.api.groner.app
```

A URL completa da consulta é:

``` text
https://gerasol.api.groner.app/api/Venda/DadosVendaPorPeriodo
```

A documentação pública da API Groner, seção 12.6, mostra essa chamada
com os parâmetros:

``` text
dataInicio
dataFim
```

Exemplo publicado no artigo, sem valores de cliente:

``` text
GET /api/Venda/DadosVendaPorPeriodo?dataInicio=2026-01-01T00:00:00Z&dataFim=2026-04-30T23:59:59Z
```

Os nomes `DataInicial`, `DataFinal`, `PageNumber` e `PageSize` do
esqueleto antigo não aparecem nesse curl. Não devem ser usados como se
fossem o contrato real.

A autenticação NÃO deve ser escrita diretamente nos nodes.

Deve ser configurada como credencial segura do n8n. O nome combinado
nesta pasta é `Groner Token` (Header Auth, header `Authorization`).

O arquivo `.env` tem as chaves `GRONER_TENANT` e `GRONER_API_URL`
preenchidas. A chave `GRONER_TOKEN` está vazia. O valor não entra neste
documento nem no JSON do workflow.

------------------------------------------------------------------------

## 2.2 Conta Azul

O Conta Azul é o destino da venda.

A API utiliza OAuth 2.0.

A base da API utilizada pelo MVP é:

``` text
https://api-v2.contaazul.com
```

O endpoint inicial de criação da venda é:

``` text
POST /v1/venda
```

A autenticação deve ser configurada por credencial OAuth2 do n8n. O
nome combinado nesta pasta é `Conta Azul OAuth2`.

No `.env`, estas chaves estão preenchidas e os valores não são
reproduzidos aqui:

``` text
CONTA_AZUL_APP_NAME
CONTA_AZUL_CLIENT_ID
CONTA_AZUL_CLIENT_SECRET
CONTA_AZUL_REDIRECT_URI
CONTA_AZUL_TEST_USER
CONTA_AZUL_TEST_PASSWORD
```

Estas chaves existem e estão vazias:

``` text
CONTA_AZUL_ACCESS_TOKEN
CONTA_AZUL_REFRESH_TOKEN
```

Nunca colocar no JSON do workflow:

-   client_secret
-   access_token
-   refresh_token
-   senha
-   credenciais do cliente

------------------------------------------------------------------------

# 3. Estado atual do JSON

O esqueleto que segue os oito nodes da especificação está em:

``` text
workflows/LEO - MVP - Venda Fechada Groner.json
```

Id no n8n local: `LEOMvpVendaGron`. Nome:
`Groner → Conta Azul | MVP - Venda Fechada`.

Nodes:

1.  MANUAL - Teste
2.  CONFIG - Preencher depois
3.  GRONER - Buscar vendas
4.  NORMALIZAR - Venda Groner
5.  IF - Venda fechada?
6.  MAPEAR - Conta Azul
7.  CONTA AZUL - Criar venda
8.  SUCESSO - Registro

Fluxo atual:

``` text
MANUAL - Teste
      ↓
CONFIG - Preencher depois
      ↓
GRONER - Buscar vendas          (desativado)
      ↓
NORMALIZAR - Venda Groner
      ↓
IF - Venda fechada?
      ↓
MAPEAR - Conta Azul
      ↓
CONTA AZUL - Criar venda        (desativado)
      ↓
SUCESSO - Registro              (desativado)
```

Os nodes que fazem chamadas externas estão desativados. `SUCESSO -
Registro` também está desativado: no n8n, um node desativado é pulado.
Se só o POST ficasse desativado, o registro marcaria sucesso sem criar
a venda.

Há ainda estes arquivos de apoio, todos inativos:

``` text
LEO - 00 - Config
LEO - 01 - Groner - Gerar Token
LEO - 02 - Groner - Testar Token
LEO - 03 - Conta Azul - Testar Conexão
LEO - 10 - Groner Vendas → Conta Azul
LEO - 99 - Error Workflow
LEO-groner-contato.json
```

`LEO - 10` é o rascunho que tinha desviado do contrato. Foi alinhado:
endpoint `DadosVendaPorPeriodo`, interface interna, IF de venda fechada,
POST de venda desativado e conta a receber fora do caminho. As chamadas
externas desse rascunho estão desativadas.

`LEO-groner-contato.json` não é o fluxo de venda. O webhook sem
autenticação e a busca do Lead ficam desativados. O arquivo não foi
apagado.

O canvas `LEO - Integração Leonela` (`k7Qm2nR8sT4vW9xY`) é uma consulta
somente leitura de pessoas na Conta Azul (`GET /v1/pessoas`). Não é o
fluxo de venda fechada. Não foi reescrito aqui porque outro trabalho
ainda o estava editando.

------------------------------------------------------------------------

# 4. Node 1 --- MANUAL - Teste

## Tipo

Manual Trigger.

## Função

É o gatilho de desenvolvimento.

Ele permite executar o workflow manualmente durante a construção e
testes.

Neste estágio ele existe para facilitar:

-   testes;
-   debugging;
-   inspeção dos dados;
-   desenvolvimento do mapeamento;
-   validação das APIs.

Não deve necessariamente permanecer como gatilho definitivo em produção.

No MVP e no `LEO - 10` o gatilho manual está presente. O agendamento do
`LEO - 10` está desativado.

------------------------------------------------------------------------

# 5. Node 2 --- CONFIG - Preencher depois

## Função

Centralizar configurações básicas utilizadas pelo workflow.

O MVP possui atualmente:

``` json
{
  "groner_base_url": "https://gerasol.api.groner.app",
  "groner_endpoint_vendas": "/api/Venda/DadosVendaPorPeriodo",
  "conta_azul_base_url": "https://api-v2.contaazul.com",
  "conta_azul_endpoint_venda": "/v1/venda",
  "status_groner_fechada": "FECHADA",
  "modo_teste": true
}
```

`FECHADA` continua provisório. `modo_teste` verdadeiro significa que
isto não é produção.

O `LEO - 00 - Config` repete esses parâmetros para os outros workflows
LEO e calcula `baseUrl`, `dataInicio` e `dataFim`. O tenant nesse node
é `gerasol`.

Esses valores são parâmetros de desenvolvimento.

A IA que continuar esse projeto deve evitar espalhar URLs e
configurações hardcoded por vários nodes.

Sempre que possível:

``` text
CONFIG
   ↓
nodes posteriores
```

devem receber os parâmetros por expressão ou credencial.

------------------------------------------------------------------------

# 6. Node 3 --- GRONER - Buscar vendas

## Tipo

HTTP Request.

## Objetivo

Consultar vendas no Groner.

Endpoint:

``` text
GET /api/Venda/DadosVendaPorPeriodo
```

Parâmetros que a documentação da Groner publica para essa chamada:

``` text
dataInicio
dataFim
```

O MVP usa uma janela inicial aproximada de ontem até o fim de hoje, em
UTC, no formato com hora. O node está desativado. A credencial prevista
é `Groner Token`. Não há token gravado no JSON.

### Importante

O corpo da resposta ainda não foi capturado.

Antes de produção, confirmar com uma resposta real da API do cliente:

-   formato das datas já visto no curl, e se a API aceita outro;
-   se esse endpoint pagina e como;
-   estrutura da resposta;
-   campo que representa o status;
-   identificador único da venda;
-   estrutura do cliente;
-   estrutura dos itens;
-   estrutura das parcelas;
-   valor total;
-   descontos;
-   acréscimos;
-   vendedor;
-   forma de pagamento.

A listagem `GET /api/Venda?pagina=&tamanhoPagina=` existe na mesma
documentação e é outro recurso. Não substitui
`DadosVendaPorPeriodo`. O rascunho antigo do `LEO - 10` usava essa
listagem e uma paginação `hasNext` que o curl não mostra. Isso foi
retirado.

------------------------------------------------------------------------

# 7. Dados esperados do Groner

O workflow precisa chegar a uma estrutura interna normalizada semelhante
a:

``` json
{
  "gronerSaleId": "12345",
  "status": "FECHADA",
  "saleNumber": "987",
  "saleDate": "2026-10-03",
  "customer": {
    "name": "Cliente Exemplo",
    "document": "00000000000",
    "email": "cliente@email.com",
    "phone": "41999999999"
  },
  "total": 1500,
  "installments": [],
  "items": []
}
```

Essa estrutura é uma INTERFACE INTERNA.

Ela não significa que esses sejam os nomes reais dos campos do Groner.

Os campos reais deverão ser mapeados depois de receber um JSON
verdadeiro. Esse JSON ainda não está nesta pasta.

------------------------------------------------------------------------

# 8. Node 4 --- NORMALIZAR - Venda Groner

## Tipo

Code node.

## Função

Converter o payload bruto do Groner para um modelo interno padronizado.

O MVP tenta localizar campos usando alternativas como:

``` javascript
v.id
v.Id

v.status
v.Status
v.statusVenda

v.numero
v.Numero

v.dataVenda
v.DataVenda

v.cliente
v.Cliente

v.parcelas
v.Parcelas

v.itens
v.Itens
v.items
```

Isso é apenas uma aproximação inicial.

A implementação definitiva deverá ser feita com base em um payload real.

O `LEO - 10` passou a emitir os mesmos campos internos
(`gronerSaleId`, `status`, `customer`, `total`, `installments`,
`items`) e a chave `GRONER:{gronerSaleId}`. Também é aproximação. Não
houve resposta real para conferir.

------------------------------------------------------------------------

# 9. Regras de normalização

A normalização deverá produzir, no mínimo:

## Identificador

``` text
gronerSaleId
```

Esse campo é obrigatório.

Ele será usado para impedir duplicidade.

## Status

``` text
status
```

Usado para determinar se a venda deve continuar no fluxo.

## Número da venda

``` text
saleNumber
```

## Data

``` text
saleDate
```

## Cliente

``` text
customer.name
customer.document
customer.email
customer.phone
```

## Valor

``` text
total
```

## Parcelas

``` text
installments[]
```

## Itens

``` text
items[]
```

------------------------------------------------------------------------

# 10. Node 5 --- IF - Venda fechada?

## Função

Decidir se a venda deve ser enviada ao Conta Azul.

Regra atual:

``` text
status == status_groner_fechada
```

O valor configurado hoje é:

``` text
FECHADA
```

Se SIM:

``` text
continua
```

Se NÃO:

``` text
encerra / ignora
```

### Importante

O status `FECHADA` é provisório.

A IA deverá confirmar o valor exato usado pelo Groner para representar
uma venda efetivamente fechada.

Não assumir que o sistema usa exatamente a string:

``` text
FECHADA
```

O IF do MVP e o IF do `LEO - 10` leem o valor da config. Enquanto o
campo real não for conhecido, uma venda sem esse status não segue.

------------------------------------------------------------------------

# 11. Controle de duplicidade --- obrigatório

Este é um dos requisitos mais importantes da automação.

Uma venda do Groner NÃO pode ser criada duas vezes no Conta Azul.

Exemplo de problema que deve ser evitado:

``` text
Groner venda 123
      ↓
n8n cria Conta Azul venda 456
      ↓
workflow executa novamente
      ↓
Groner venda 123 aparece novamente
      ↓
n8n cria Conta Azul venda 789
```

Isso é proibido.

A implementação definitiva deve manter uma tabela ou armazenamento
persistente com pelo menos:

``` text
groner_sale_id
conta_azul_sale_id
status
created_at
updated_at
error
```

Exemplo:

``` json
{
  "groner_sale_id": "123",
  "conta_azul_sale_id": "456",
  "status": "SUCCESS",
  "created_at": "2026-10-03T20:00:00-03:00"
}
```

Antes de criar uma venda:

``` text
Existe groner_sale_id?
       ↓
      SIM
       ↓
Não criar novamente
```

Se não existir:

``` text
Criar venda
   ↓
Salvar vínculo
```

Estado nesta pasta: essa tabela não existe. Por isso os POST que criam
venda, cliente ou conta a receber permanecem desativados. Calcular a
chave `GRONER:{gronerSaleId}` no item não substitui o armazenamento.

------------------------------------------------------------------------

# 12. Node 6 --- MAPEAR - Conta Azul

## Tipo

Code node.

## Função

Transformar o modelo interno em payload compatível com o endpoint de
venda da Conta Azul.

O MVP possui atualmente um payload conceitual:

``` json
{
  "id_cliente": "TODO-CONTA-AZUL-CLIENTE-ID",
  "numero": 987,
  "situacao": "APROVADO",
  "data_venda": "2026-10-03",
  "observacoes": "Origem: Groner | Venda: 12345",
  "itens": [
    {
      "descricao": "Produto",
      "quantidade": 1,
      "valor": 1500,
      "id": "TODO-CONTA-AZUL-PRODUTO-ID"
    }
  ]
}
```

Esse payload também é um ESQUELETO.

`situacao: APROVADO` não foi confirmada na conta da Leonela. Os ids de
cliente e de produto continuam `TODO`.

Antes da produção, validar todos os campos contra a versão atual da API
da Conta Azul.

O `LEO - 10` usa o mesmo esqueleto no POST desativado. O corpo anterior,
com condição de pagamento e situação `EM_ANDAMENTO`, saiu desse node
porque esses campos não estão no esqueleto e não foram validados.

------------------------------------------------------------------------

# 13. Cliente na Conta Azul

A integração precisa responder:

> O cliente da venda Groner já existe na Conta Azul?

Estratégia:

``` text
Cliente Groner
     ↓
buscar por identificador confiável
     ↓
encontrou?
 ┌───┴───┐
SIM     NÃO
 ↓       ↓
usar    criar
ID      cliente
         ↓
       usar ID
```

O documento do cliente, quando disponível e permitido pelo sistema, deve
ser preferido como identificador de negócio.

Não utilizar somente nome para deduplicação.

Exemplo ruim:

``` text
João da Silva
```

Exemplo melhor:

``` text
CPF/CNPJ
```

Se não houver documento, deverá ser definida uma estratégia alternativa
usando os dados disponíveis.

Estado nesta pasta: o `LEO - 10` esboça `GET /v1/pessoas` com o
parâmetro `documentos` e deixa o POST de criação desativado. Essa busca
também está desativada, à espera do JSON real e da credencial OAuth
conectada. O canvas `k7Qm2nR8sT4vW9xY` só lista pessoas; não cria
cliente e não fecha o de-para da venda.

------------------------------------------------------------------------

# 14. Produtos / itens

Cada item do Groner precisa ser transformado em um item aceito pelo
Conta Azul.

A integração precisa definir como relacionar:

``` text
Produto Groner
      ↓
Produto Conta Azul
```

Preferência:

``` text
código/SKU
```

ou outro identificador estável.

Evitar depender apenas de:

``` text
nome do produto
```

porque nomes podem mudar.

A IA deverá implementar uma etapa:

``` text
Buscar produto Conta Azul
        ↓
Existe?
   ┌────┴────┐
  SIM       NÃO
   ↓         ↓
usar ID   decidir:
          criar ou
          bloquear erro
```

A regra de criação automática de produtos deve ser definida pelo cliente
antes da produção.

Estado nesta pasta: não há busca de produto. O item do esqueleto usa
`TODO-CONTA-AZUL-PRODUTO-ID`. O `LEO - 00` tem o campo
`contaAzulServicoId` vazio.

------------------------------------------------------------------------

# 15. Node 7 --- CONTA AZUL - Criar venda

## Tipo

HTTP Request.

## Método

``` text
POST
```

## Endpoint inicial

``` text
https://api-v2.contaazul.com/v1/venda
```

No MVP a URL é montada pela config:

``` text
conta_azul_base_url + conta_azul_endpoint_venda
```

## Body

JSON gerado pelo node:

``` text
$json.contaAzulPayload
```

## Autenticação

OAuth 2.0, credencial `Conta Azul OAuth2`.

Nunca colocar tokens diretamente no node.

O node está desativado. Não criar venda enquanto a duplicidade não
estiver persistida.

------------------------------------------------------------------------

# 16. Parcelas / financeiro

Este requisito é parte central do projeto.

A venda fechada no Groner pode possuir:

``` text
valor total
forma de pagamento
número de parcelas
vencimento de cada parcela
valor de cada parcela
```

A automação deve transformar essas informações em registros financeiros
aceitos pelo Conta Azul.

Não assumir que simplesmente criar a venda já significa que todo o
financeiro foi criado exatamente como o cliente espera.

Antes da versão final, validar:

1.  endpoint de contas a receber;
2.  estrutura das parcelas;
3.  vencimentos;
4.  valores;
5.  forma de pagamento;
6.  situação;
7.  categoria;
8.  centro de custo, se necessário;
9.  cliente vinculado;
10. vínculo com a venda.

A implementação final deve preservar a lógica financeira original da
venda do Groner.

Estado nesta pasta: o `LEO - 10` ainda contém um node de conta a
receber, desativado e sem ligação no fluxo. Ele não é um caminho
alternativo à venda. Ligá-lo junto com o POST `/v1/venda` poderia lançar
o valor duas vezes. Os ids de conta financeira e categoria na config
estão vazios.

------------------------------------------------------------------------

# 17. Node 8 --- SUCESSO - Registro

## Função

Marcar que a integração terminou com sucesso.

O MVP cria campos:

``` json
{
  "integracao_status": "SUCESSO",
  "origem": "Groner",
  "destino": "Conta Azul"
}
```

Na versão definitiva, esse node deverá registrar também:

``` text
groner_sale_id
conta_azul_sale_id
timestamp
status
```

Idealmente o registro deverá ser persistido em banco/tabela própria.

Estado nesta pasta: o node existe e está desativado, de propósito, até
o POST existir de verdade. Não há persistência.

------------------------------------------------------------------------

# 18. Tratamento de erros

A versão definitiva deve possuir Error Workflow.

Erros possíveis:

### Groner

-   API indisponível;
-   token expirado;
-   token inválido;
-   timeout;
-   resposta inválida;
-   venda não encontrada.

### Conta Azul

-   OAuth expirado;
-   cliente inexistente;
-   produto inexistente;
-   payload inválido;
-   venda duplicada;
-   rate limit;
-   API indisponível;
-   erro 4xx;
-   erro 5xx.

### Regra

Um erro não deve resultar em uma segunda venda automática sem verificar
se a primeira criação realmente ocorreu.

Exemplo perigoso:

``` text
POST Conta Azul
      ↓
Conta Azul cria venda
      ↓
resposta demora
      ↓
n8n entende timeout
      ↓
retry
      ↓
segunda venda
```

Portanto, retries precisam ser idempotentes ou precedidos de
verificação.

Estado nesta pasta: `LEO - 99 - Error Workflow` monta um resumo com
nome do workflow, node, mensagem, horário e link da execução. Não grava
`groner_sale_id`, não avisa uma pessoa e não está publicado como
tratamento de produção. Vários workflows LEO apontam o id
`LEO99ErrorAlerta` na configuração. Isso não cobre retry idempotente.

------------------------------------------------------------------------

# 19. Idempotência

A automação deve ser idempotente.

Conceito:

> Executar a mesma venda duas ou mais vezes não pode gerar múltiplos
> lançamentos financeiros.

Chave de idempotência recomendada:

``` text
GRONER:{gronerSaleId}
```

Exemplo:

``` text
GRONER:12345
```

Essa chave deve ser persistida.

Estado nesta pasta: o `LEO - 10` calcula a chave no item normalizado.
Nada é gravado. Sem essa gravação, o POST permanece desativado.

------------------------------------------------------------------------

# 20. Estratégia de execução

O MVP usa:

``` text
Manual Trigger
```

Para produção, o projeto deve decidir entre:

### Opção A --- Polling

Executar a cada X minutos:

``` text
Schedule Trigger
      ↓
buscar vendas recentes
      ↓
filtrar fechadas
      ↓
processar novas
```

Vantagem:

-   simples;
-   previsível;
-   fácil de monitorar.

### Opção B --- Webhook/evento do Groner

Se o Groner fornecer um evento adequado para venda fechada, utilizar:

``` text
Groner
   ↓
Webhook
   ↓
n8n
   ↓
processamento imediato
```

Preferir webhook quando houver evento confiável para o evento exato de
venda fechada.

Estado nesta pasta: gatilho manual. O agendamento do `LEO - 10` existe
e está desativado. Não há webhook de venda fechada. O webhook de
contato em `LEO-groner-contato.json` não é esse evento e está
desativado.

------------------------------------------------------------------------

# 21. Janela de busca e duplicidade

Se for utilizado polling, nunca confiar apenas em:

``` text
últimas 24 horas
```

como mecanismo de controle.

Exemplo:

``` text
Dia 1:
venda 123 fechada
```

n8n fica indisponível.

``` text
Dia 2:
n8n volta
```

A venda precisa ser encontrada.

Portanto, a janela de consulta pode ter sobreposição:

``` text
últimas 24/48 horas
```

ou outra janela adequada.

A duplicidade será resolvida pelo armazenamento de `groner_sale_id`.

Assim:

``` text
Buscar mais dados
+
deduplicar
=
segurança
```

Estado nesta pasta: a janela do MVP ainda é aproximadamente de ontem até
hoje, e o `LEO - 00` calcula sete dias. As duas são só desenvolvimento.
Sem a tabela de `groner_sale_id`, alargar a janela e ligar o POST seria
inseguro. A consulta Groner segue desativada.

------------------------------------------------------------------------

# 22. Paginação

Se houver mais de 100 vendas no período, o workflow não pode processar
somente a primeira página.

Deve existir paginação:

``` text
Página 1
 ↓
Página 2
 ↓
Página 3
 ↓
...
```

até não existirem mais resultados.

A implementação deve seguir o mecanismo de paginação efetivamente
documentado pelo Groner.

Estado nesta pasta: o curl publicado de `DadosVendaPorPeriodo` não mostra
parâmetros de página. A listagem `GET /api/Venda` mostra `pagina` e
`tamanhoPagina`, mas é outro endpoint. Não foi inventada paginação
`hasNext` na consulta por período. Paginação deste endpoint continua
TODO, depois de uma resposta real.

------------------------------------------------------------------------

# 23. Segurança

Nunca armazenar no workflow:

``` text
password
client_secret
access_token
refresh_token
JWT
```

Utilizar credenciais nativas do n8n.

Separar:

``` text
dados de configuração
```

de:

``` text
segredos
```

Também não registrar tokens nos logs.

Estado nesta pasta: `.gitignore` ignora `.env`. Os JSON revisados não
contêm segredo. `LEO - 01` tem campos de e-mail e senha vazios, para
preencher na hora e apagar em seguida; o histórico de execução desse
fluxo está desligado no arquivo. Credenciais previstas, só pelo nome:
`Groner Token` e `Conta Azul OAuth2`.

------------------------------------------------------------------------

# 24. Observabilidade

A integração deve permitir responder:

> O que aconteceu com a venda X?

Para cada venda deve ser possível descobrir:

``` text
ID Groner
↓
data/hora
↓
status da integração
↓
ID Conta Azul
↓
erro, se houver
```

Exemplo:

``` text
Groner: 12345
Conta Azul: 98765
Status: SUCCESS
Data: 03/10/2026 15:30
```

Ou:

``` text
Groner: 12346
Conta Azul: -
Status: ERROR
Erro: Produto não encontrado
```

Estado nesta pasta: ainda não dá para responder isso. Não há tabela de
vínculo. O `LEO - 99` só resume a falha do workflow.

------------------------------------------------------------------------

# 25. Fluxo definitivo esperado

A arquitetura final deverá ser aproximadamente:

``` text
                 ┌─────────────────────┐
                 │ GRONER CRM          │
                 │ Venda fechada       │
                 └──────────┬──────────┘
                            ↓
                 ┌─────────────────────┐
                 │ TRIGGER             │
                 │ Webhook ou Schedule │
                 └──────────┬──────────┘
                            ↓
                 ┌─────────────────────┐
                 │ BUSCAR VENDA        │
                 │ Dados completos     │
                 └──────────┬──────────┘
                            ↓
                 ┌─────────────────────┐
                 │ NORMALIZAR          │
                 │ Dados internos      │
                 └──────────┬──────────┘
                            ↓
                 ┌─────────────────────┐
                 │ STATUS = FECHADA?   │
                 └──────┬────────┬─────┘
                        NÃO       SIM
                         ↓         ↓
                       IGNORA   DEDUPLICAR
                                   ↓
                         ┌──────────────────┐
                         │ JÁ INTEGRADA?    │
                         └──────┬─────┬─────┘
                               SIM    NÃO
                                ↓      ↓
                              IGNORA  CONTINUA
                                        ↓
                              ┌─────────────────┐
                              │ CLIENTE         │
                              │ localizar/criar │
                              └────────┬────────┘
                                       ↓
                              ┌─────────────────┐
                              │ PRODUTOS        │
                              │ localizar/mapear│
                              └────────┬────────┘
                                       ↓
                              ┌─────────────────┐
                              │ CRIAR VENDA     │
                              │ CONTA AZUL      │
                              └────────┬────────┘
                                       ↓
                              ┌─────────────────┐
                              │ PARCELAS /      │
                              │ FINANCEIRO      │
                              └────────┬────────┘
                                       ↓
                              ┌─────────────────┐
                              │ REGISTRAR       │
                              │ VÍNCULO         │
                              └────────┬────────┘
                                       ↓
                                  SUCESSO
```

O que já existe no esqueleto é o miolo manual: buscar, normalizar, IF,
mapear, criar venda e registrar. Duplicidade, produto, parcelas e o
registro persistido ainda não existem. As chamadas externas estão
desligadas.

------------------------------------------------------------------------

# 26. O que NÃO deve ser feito

A IA não deve:

1.  Inventar campos da API.
2.  Inventar endpoints.
3.  Colocar tokens diretamente no workflow.
4.  Considerar o JSON atual pronto para produção.
5.  Criar vendas sem controle de duplicidade.
6.  Criar parcelas sem validar a API financeira do Conta Azul.
7.  Considerar nome do cliente como identificador único.
8.  Considerar nome do produto como identificador único.
9.  Fazer retry cego de POST de criação.
10. Ativar todos os nodes antes de testar o payload real.
11. Alterar dados financeiros sem preservar o valor original.
12. Ignorar erros 4xx/5xx.
13. Processar apenas a primeira página de resultados.
14. Depender exclusivamente de uma janela pequena de consulta.
15. Registrar tokens nos logs.

------------------------------------------------------------------------

# 27. Ordem recomendada de implementação

A IA deve seguir esta ordem:

## Etapa 1 --- Capturar venda real

Obter um JSON real retornado pelo Groner em
`/api/Venda/DadosVendaPorPeriodo`.

Não continuar o mapeamento definitivo sem isso.

Pendente. `GRONER_TOKEN` está vazio e o node de busca está desativado.

## Etapa 2 --- Mapear Groner

Descobrir exatamente:

``` text
ID venda
status
data
cliente
documento
itens
quantidade
valor
total
forma pagamento
parcelas
vencimentos
```

Pendente. O Code node só tem a aproximação.

## Etapa 3 --- Testar Conta Azul

Configurar OAuth2.

Testar consulta de clientes/produtos.

Parcial. O app está descrito no `.env` (sem copiar segredo aqui).
Access token e refresh token estão vazios. `LEO - 03` e o canvas
`k7Qm2nR8sT4vW9xY` preparam leitura de pessoas. A conexão não foi
marcada como verde nesta revisão.

## Etapa 4 --- Mapear cliente

Implementar:

``` text
buscar cliente
↓
encontrou?
↓
usar ID
```

ou:

``` text
não encontrou
↓
criar
↓
usar ID
```

Esboço no `LEO - 10`, com a criação desativada. Não validado.

## Etapa 5 --- Mapear produtos

Implementar o vínculo entre Groner e Conta Azul.

Pendente.

## Etapa 6 --- Criar venda

Somente depois dos passos anteriores.

O POST existe e está desativado.

## Etapa 7 --- Financeiro

Implementar parcelas/contas a receber.

Pendente. O node de conta a receber não está no caminho.

## Etapa 8 --- Idempotência

Persistir:

``` text
groner_sale_id
conta_azul_sale_id
```

Pendente.

## Etapa 9 --- Erros

Criar Error Workflow.

Esqueleto em `LEO - 99`. Falta persistir o erro da venda e o aviso.

## Etapa 10 --- Teste integrado

Usar uma venda controlada.

Pendente.

## Etapa 11 --- Produção

Somente depois de validar:

-   venda;
-   cliente;
-   produto;
-   valor;
-   parcelas;
-   duplicidade;
-   erros.

------------------------------------------------------------------------

# 28. Critérios de aceite

A integração só pode ser considerada pronta quando:

-   [ ] Uma venda fechada do Groner é encontrada.
-   [ ] O ID da venda é preservado.
-   [ ] O cliente correto é identificado.
-   [ ] O cliente é criado caso a regra permita.
-   [ ] Os produtos são corretamente relacionados.
-   [ ] O valor total é preservado.
-   [ ] A venda é criada no Conta Azul.
-   [ ] As parcelas são corretamente registradas.
-   [ ] A venda não é duplicada.
-   [ ] Existe registro do ID Groner ↔ ID Conta Azul.
-   [ ] Erros são registrados.
-   [ ] Falhas de API são tratadas.
-   [ ] Retries não geram duplicidade.
-   [ ] Paginação funciona.
-   [ ] Tokens não aparecem nos logs.
-   [ ] A automação pode ser executada novamente sem gerar lançamento
    duplicado.

Nenhum item acima foi aceito. O que existe é preparação: n8n no ar,
esqueleto, credenciais só pelo nome, segredos fora do JSON e POST
desativado.

------------------------------------------------------------------------

# 29. Resultado final esperado

O usuário final não deve precisar executar nenhuma etapa manual.

A experiência desejada é:

``` text
VENDA FECHADA NO GRONER
        ↓
     alguns segundos/minutos
        ↓
VENDA EXISTE NO CONTA AZUL
        ↓
FINANCEIRO/PARCELAS CORRETOS
        ↓
CLIENTE NÃO PRECISA DIGITAR NOVAMENTE
```

O usuário somente confere.

O objetivo não é simplesmente "mandar um JSON de uma API para outra".

O objetivo é criar uma integração confiável entre **CRM e ERP**, com:

``` text
automação
+
transformação
+
idempotência
+
controle financeiro
+
segurança
+
observabilidade
+
tratamento de erros
```

------------------------------------------------------------------------

# 30. Estado do projeto neste momento

O arquivo JSON do MVP representa somente o **esqueleto inicial**, agora
com o tenant `gerasol`, o endpoint documentado e as chamadas externas
desativadas.

Ele NÃO deve ser vendido ou apresentado como integração final.

Os principais pontos ainda pendentes são:

``` text
TODO 1 — payload real do Groner
TODO 2 — status exato de venda fechada
TODO 3 — autenticação Groner (credencial existe no desenho; token vazio)
TODO 4 — OAuth2 Conta Azul conectado e testado
TODO 5 — consulta/criação de cliente validada
TODO 6 — mapeamento de produtos
TODO 7 — criação definitiva da venda
TODO 8 — parcelas/financeiro
TODO 9 — banco/armazenamento de idempotência
TODO 10 — Error Workflow com registro da venda
TODO 11 — paginação do endpoint por período
TODO 12 — estratégia de trigger
TODO 13 — testes integrados
TODO 14 — produção
```

A próxima IA que receber este documento deve **continuar a partir deste
estado**, sem assumir que os TODOs já foram resolvidos.

O princípio fundamental é:

> Primeiro descobrir e validar os dados reais das APIs. Depois construir
> o mapeamento. Depois testar uma venda. Somente então ativar a
> automação em produção.
