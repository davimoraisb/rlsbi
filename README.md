# Projeto Power BI - Análise de Repasses e Segurança RLS

Este repositório contém o projeto de Power BI desenvolvido para a análise de dados de repasses financeiros públicos (programa Bolsa), estruturado com tabelas fato e dimensão, medidas em DAX e controles rígidos de acesso através de Segurança em Nível de Linha (RLS - Row-Level Security).

---

## 1. Estrutura das Fontes de Dados e ETL (Power Query)

O processo de extração, transformação e carga (ETL) foi realizado no Power Query para modelar e preparar as informações a partir de bases brutas. O modelo final é constituído por três tabelas:

### amostra_bolsa (Tabela Fato)
Tabela contendo os registros de transações de repasses.
- Origem: Arquivo CSV local ("amostra_bolsa.csv").
- Transformações Aplicadas:
  - Promoção de cabeçalhos e tipagem correta de todas as colunas.
  - Duplicação do campo "MÊS COMPETÊNCIA" para a geração de duas novas dimensões de tempo:
    - ANO: Obtido através da extração dos 4 primeiros caracteres de "MÊS COMPETÊNCIA".
    - MÊS: Obtido através da extração dos 2 últimos caracteres de "MÊS COMPETÊNCIA".
- Estrutura de Colunas:
  - MÊS COMPETÊNCIA (Número Inteiro)
  - UF (Texto)
  - CÓDIGO MUNICÍPIO SIAFI (Número Inteiro)
  - NOME MUNICÍPIO (Texto)
  - CPF FAVORECIDO (Texto)
  - NIS FAVORECIDO (Número Inteiro)
  - NOME FAVORECIDO (Texto)
  - VALOR PARCELA (Decimal - Formato de Moeda)
  - ANO (Texto)
  - MÊS (Texto)

### dim_regioes (Tabela Dimensão)
Tabela auxiliar de mapeamento geográfico que relaciona os estados brasileiros às suas regiões.
- Origem: Tabela digitada manualmente na modelagem.
- Estrutura de Colunas:
  - Regiao (Texto) - Ex: Norte, Nordeste, Centro-Oeste, Sudeste, Sul.
  - UF (Texto) - Unidade Federativa correspondente.

### dim_usuarios_rls (Tabela Dimensão de Segurança)
Tabela utilizada para o gerenciamento de acesso dinâmico de segurança.
- Origem: Tabela digitada manualmente na modelagem.
- Estrutura de Colunas:
  - email_usuario (Texto) - Endereço de e-mail institucional do gestor correspondente.
  - regiao_permitida (Texto) - Região geográfica à qual o gestor possui permissão de leitura.

---

## 2. Modelo de Dados e Relacionamentos

Para garantir a propagação correta dos filtros de visualização e de segurança no modelo estrela (Star Schema) adaptado, foram definidos os seguintes relacionamentos:

- Relacionamento 1 (Geográfico):
  - Tabela Fato: amostra_bolsa (Coluna: UF)
  - Tabela Dimensão: dim_regioes (Coluna: UF)
  - Cardinalidade: Muitos para Um (*:1)
  - Direção do Filtro: dim_regioes filtra amostra_bolsa

- Relacionamento 2 (Segurança):
  - Tabela Fato/Dimensão: dim_regioes (Coluna: Regiao)
  - Tabela Dimensão: dim_usuarios_rls (Coluna: regiao_permitida)
  - Cardinalidade: Muitos para Um (*:1)
  - Direção do Filtro: dim_usuarios_rls filtra dim_regioes

---

## 3. Medidas DAX Desenvolvidas

Foram programadas quatro medidas em linguagem DAX na tabela fato amostra_bolsa para calcular os KPIs do dashboard:

### Total Repasses
Calcula a somatória total das parcelas pagas aos beneficiários.
- Fórmula DAX:
  ```dax
  Total Repasses = SUM(amostra_bolsa[VALOR PARCELA])
  ```
- Formatação: Moeda (R$ com padrão de cultura pt-BR).

### Total Beneficiarios
Calcula o número absoluto de repasses efetuados (equivalente ao total de registros).
- Fórmula DAX:
  ```dax
  Total Beneficiarios = COUNTROWS(amostra_bolsa)
  ```
- Formatação: Número Inteiro.

### Ticket Medio
Calcula o valor médio repassado por beneficiário. Implementado de forma segura utilizando a função DIVIDE para evitar erros de divisão por zero.
- Fórmula DAX:
  ```dax
  Ticket Medio = DIVIDE([Total Repasses], [Total Beneficiarios])
  ```
- Formatação: Moeda (R$ com padrão de cultura pt-BR).

### Total Municipios
Calcula a quantidade de municípios únicos que receberam repasses no período analisado.
- Fórmula DAX:
  ```dax
  Total Municipios = DISTINCTCOUNT(amostra_bolsa[CÓDIGO MUNICÍPIO SIAFI])
  ```
- Formatação: Número Inteiro.

---

## 4. Segurança em Nível de Linha (RLS - Row-Level Security)

A segurança em nível de linha foi implementada para que diferentes perfis acessem somente os dados que lhes são autorizados.

### Perfis Estáticos (Filtro por Região)
Filtros aplicados diretamente sobre a coluna Regiao da tabela dim_regioes:
- Admin: Sem restrições aplicadas (acesso completo à base).
- Gestor_CentroOeste: Filtro aplicado `[Regiao] == "Centro-Oeste"`
- Gestor_Nordeste: Filtro aplicado `[Regiao] == "Nordeste"`
- Gestor_Norte: Filtro aplicado `[Regiao] == "Norte"`
- Gestor_Sudeste: Filtro aplicado `[Regiao] == "Sudeste"`
- Gestor_Sul: Filtro aplicado `[Regiao] == "Sul"`

### Perfil Dinâmico (Filtro por Usuário Logado)
Para uma escala corporativa automatizada, foi criado um perfil dinâmico baseado na identidade do usuário:
- Gestor_Regional_Dinamico: Filtro aplicado na tabela dim_usuarios_rls:
  ```dax
  [email_usuario] = USERPRINCIPALNAME()
  ```
- Mecanismo de Funcionamento: Quando o usuário visualiza o relatório no Power BI Service, a função USERPRINCIPALNAME() captura seu e-mail de login corporativo. A tabela dim_usuarios_rls é automaticamente filtrada para essa linha. Através dos relacionamentos definidos, o filtro flui para a tabela dim_regioes e, consequentemente, restringe as informações visíveis na tabela fato amostra_bolsa.

---

## 5. Estrutura do Relatório (Layout das Páginas)

O dashboard é composto por duas páginas interativas:

### Página 1 (Visão Geral)
Página voltada para a análise executiva dos principais KPIs consolidados:
- Cartões com os KPIs: Ticket Médio (configurado com 2 casas decimais), Total de Beneficiários, Total de Municípios e Total de Repasses.
- Gráfico de barras horizontais detalhando o Total de Repasses agrupado por Região.
- Gráfico de funil analisando o volume de lançamentos agrupado por ANO.
- Segmentador de dados que permite filtrar de forma global as informações pelo campo ANO.

### Página 2 (Visão Detalhada)
Página voltada para a análise tabular e filtros espaciais:
- Tabela detalhada contendo UF, Ticket Médio, Total de Beneficiários, Total de Municípios e Total de Repasses (ordenada de forma decrescente pelo Ticket Médio).
- Segmentador de dados para filtrar e navegar entre as opções de Região.
- Gráfico de barras verticais mostrando o volume de Total de Repasses por NOME MUNICÍPIO.
