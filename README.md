# GeoCEP — Padronização e enriquecimento de endereços

## Objetivo

O principal objetivo é padronizar endereços em datalakes. As funções ajudam a independente do que for cadastrado e de como for cadastrado nas origens, que tendo cep e numero o endereço na dimensão será padronizado. Facilitando assim analises futuras de geolocalização e principalmente resolvendo problemas de qualidade de dados.

Projeto de Engenharia de Dados, implementado no **Databricks Free** com **Python, SQL, Spark e Delta Lake**. 

O GeoCEP recebe CEP e número de um sistema de origem, consulta **ViaCEP** e **BrasilAPI CEP v2**, consolida os endereços em uma função SQL e disponibiliza informações padronizadas para consumo analítico. Há uma demonstração de carga de uma dimensão a partir de JSON aninhado.

## Funcionamento

1. **Extração Python:** os notebooks consultam as duas APIs, com timeout, tentativas adicionais em caso de falha, pequena pausa entre requisições e armazenamento do JSON original.
2. **Cache Delta:** cada API possui uma tabela de referência. CEPs com consulta bem-sucedida nos últimos **6 meses** são reutilizados; caso contrário, ocorre nova consulta e atualização por `MERGE`.
3. **Padronização SQL:** `geocep.default.Fcn_Logradouro(cep, numero)` lê os dados de referência, combina os campos, identifica divergências e devolve endereço completo, campos separados e indicadores básicos de qualidade.
4. **Demonstração de origem:** um JSON fictício de um ERP contém dois endereços. `FROM_JSON` e `EXPLODE` desaninham o array; o fluxo consulta os CEPs necessários e insere o resultado na `dim_endereco`.
5. **Data Quality:** notebook separado calcula métricas de nulos, duplicidade, formatos e datas, grava tabela de auditoria e exporta JSON.
6. **Persistência:** o notebook `Fcn_Logradouro` grava histórico de resultados em **Parquet com Snappy**, particionado por ano/mês.

## Estrutura do repositório

```text
Config/
  api_viaCep.dbquery.ipynb         # Criação da tabela Delta ViaCEP
  api_brasilapi.dbquery.ipynb      # Criação da tabela Delta BrasilAPI
  Fcn_Logradouro.dbquery.ipynb    # Criação da função tabular SQL
  dim_endereco.dbquery.ipynb      # Criação da dimensão Delta
src/
  01_api_viaCep.ipynb             # Extração / cache ViaCEP
  02_api_brasilapi.ipynb          # Extração / cache BrasilAPI
Notebooks/
  Fcn_Logradouro.ipynb            # Consulta e histórico Parquet
  Exemplo Uso.ipynb               # Demonstração JSON, EXPLODE e dimensão
DQ/
  DQ_CEP.ipynb                    # Auditoria e exportação JSON
README.md
requirements.txt
```

`Api_Geocep.ipynb` é um notebook inicial de teste de integração Databricks/GitHub.

## Pré-requisitos

- Workspace **Databricks Free**, com notebooks Python e SQL, Spark e acesso ao Unity Catalog.
- Repositório Git conectado a uma pasta Git do workspace (preferencialmente usando a branch `dev_api` para desenvolvimento).
- Catálogo `geocep`, schema `default` e volume gerenciado `geocep` existentes.
- Permissão de criação de tabelas/funções e escrita no caminho `/Volumes/geocep/default/geocep/`.
- Acesso HTTP a `viacep.com.br` e `brasilapi.com.br` a partir do notebook.

As duas APIs utilizadas **não exigem chave** nos endpoints deste projeto. Não há credenciais a colocar no código. Caso sejam acrescentadas fontes autenticadas, utilizar variáveis de ambiente ou mecanismo de segredos, nunca versionar tokens.

## Preparação (uma vez)

1. Abra a pasta Git do projeto no Databricks.
2. Execute os notebooks SQL de `Config/` para criar, nesta ordem, as tabelas de referência `ref_viacep` e `ref_brasilapi`, a função `Fcn_Logradouro` e a tabela `dim_endereco`.
3. Confira os objetos com SQL:

```sql
SHOW TABLES IN geocep.default;
DESCRIBE FUNCTION EXTENDED geocep.default.Fcn_Logradouro;
```

4. Confirme que o volume `/Volumes/geocep/default/geocep/` existe antes de salvar arquivos de saída.

## Como executar

### Consulta de um CEP e histórico de Parquet

Abra `Notebooks/Fcn_Logradouro.ipynb`, informe os widgets `cep` e `numero` e execute **Run All**. O notebook:

- executa `src/01_api_viaCep` e `src/02_api_brasilapi` por `dbutils.notebook.run`;
- consulta a função tabular `Fcn_Logradouro`;
- acrescenta o resultado ao histórico em `/Volumes/geocep/default/geocep/output/historico/`.

Exemplo de chamada direta da função, **após** executar os extratores para carregar o CEP no cache:

```sql
SELECT * FROM geocep.default.Fcn_Logradouro('08717-310', '123');
```

**Atenção:** a função SQL isolada consulta o cache Delta; **não** invoca as APIs. Para CEP novo, execute o notebook principal, que coordena a extração antes da função.

### Demonstração de carga em dimensão

Execute **Run All** em `Notebooks/Exemplo Uso.ipynb`. O notebook monta uma entrada fictícia (`ERP_LEGADO`), desaninha um array de endereços por `FROM_JSON` + `EXPLODE`, consulta os CEPs distintos, aplica a função SQL e grava a tabela `geocep.default.dim_endereco`.

Para visualizar o antes/depois:

```sql
SELECT id_cliente, tipo_endereco, endereco_original,
       endereco_completo, fonte, status_dq
FROM geocep.default.dim_endereco;
```

**Nota:** a demonstração usa `INSERT` simples; repetir a execução insere novamente os exemplos. Os números e endereços de origem são fictícios; o endereço padronizado depende da resposta efetiva das APIs.

### Executar a auditoria

Execute `DQ/DQ_CEP.ipynb` em sequência. Ele gera `geocep.default.dq_cep` e exporta o relatório:

```text
/Volumes/geocep/default/geocep/quality/relatorio_dq.json
```

Consulta de verificação:

```sql
SELECT * FROM geocep.default.dq_cep;
```

## APIs e política de consultas

| Fonte | Endpoint por CEP | Uso |
|---|---|---|
| ViaCEP | `https://viacep.com.br/ws/{cep}/json/` | Campos de endereço preferenciais |
| BrasilAPI CEP v2 | `https://brasilapi.com.br/api/cep/v2/{cep}` | Fallback de campos, comparação e eventual geolocalização |

Os extratores normalizam o CEP para oito dígitos, consultam o cache, fazem requisições com `timeout=10`, até três tentativas, espera de 1 segundo antes de chamadas HTTP e backoff em falhas transitórias. Trata-se de **rate limiting sequencial simples**, não de controle global distribuído.

Não tente enumerar todas as combinações de CEP. Para cargas maiores, use os CEPs distintos efetivamente presentes nas bases de negócio e respeite os limites de uso das APIs.

## Data Quality

- Formato do CEP (oito dígitos) e intervalos de datas.
- Contagens e unicidade de CEPs por fonte.
- Indicadores de valores nulos e respostas HTTP.
- Schema básico (tipos de campos), divergência entre as duas fontes e percentual de completude no endereço consolidado.
- Relatório exportado como JSON para auditoria.

O número do imóvel é informado pelo sistema de origem: **o GeoCEP não verifica a existência física do número**. Concordância das fontes não equivale à confirmação do imóvel.

## Persistência e entregáveis

- **Cache:** `geocep.default.ref_viacep` e `geocep.default.ref_brasilapi`, em Delta.
- **Dimensão:** `geocep.default.dim_endereco`, em Delta.
- **Histórico Parquet:** `/Volumes/geocep/default/geocep/output/historico/`, com compressão Snappy e partição `ano`/`mes`.
- **Auditoria:** `geocep.default.dq_cep` e arquivo `quality/relatorio_dq.json` no Volume.

## Dependências

O ambiente de execução é o Databricks Free. A biblioteca Python externa usada diretamente é `requests`; PySpark, `spark` e `dbutils` são disponibilizados pelo runtime Databricks. 

## Limitações e melhorias futuras

- Para grande volume: materializar e atualizar semanalmente os CEPs conhecidos, consultar APIs apenas para CEPs ausentes ou vencidos, monitorar limites das APIs e evoluir as validações de qualidade.
