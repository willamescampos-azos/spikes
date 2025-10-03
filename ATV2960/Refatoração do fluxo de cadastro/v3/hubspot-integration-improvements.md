# Spike - Refatoração do fluxo de Cadastro

> **Documento de Investigação Técnica**: Análise e proposta de melhorias para a integração HubSpot, incluindo mapeamento de campos, resiliência, observabilidade e validações

---

## 📋 Descrição

Realizar uma investigação técnica detalhada sobre a integração HubSpot, focando na melhoria da resiliência, observabilidade e mapeamento de campos entre frontend e backend. O objetivo é implementar um sistema robusto onde a responsabilidade fique apenas com o back-end de integrar com o hubspot, com suporte de retry, validações de dados sensíveis e melhor mapeamento de campos para garantir a confiabilidade das integrações.

---

## 🎯 Contexto

O hubspot no fluxo de cadastro é acessado pelo front-end e back-end de forma não gerenciada, onde dados podem ser perdidos, ou inseridos em um formato não esperado. Fluxos de cadastro que são interrompidos têm informações perdidas. Front-end precisa atualizar o registro do hubspot com todos os dados que tem acesso.


**Problemas identificados:**
- O fluxo do hubspot não tem validação prévia para considerar os dados.
- A integração HubSpot atualmente não tem suporte para  resiliência (retry) e observabilidade.
- Fluxos de cadastro podem ser interrompidos, gerando perda de dados que seriam adicionados nas etapas posteriores de cadastro. 
- Há inconsistências no mapeamento de campos entre frontend e backend.
- Erro no preenchimento de data de nascimento do corretor,
sendo colocada como um campo de data de nascimento do segurado.

---

## 🎯 Objetivos

- Mapear todos os campos entre frontend e backend
- Implementar sistema de retry e resiliência
- Criar sistema de observabilidade com anonimização
- Implementar validações robustas para documentos
- Propor mudanças estruturais na integração
- Tratar CNPJ alfanumérico

---

## ✅ Critérios de aceitação

- Entregar mapeamento completo de campos frontend/backend
- Apresentar proposta de resiliência simplificada
- Documentar estratégia de observabilidade com anonimização
- Fornecer validações para CPF, CNPJ e data de nascimento
- Propor um novo fluxo de cadastro

---

## 📝 Notas e resultados

### 1. Mapeamento de Campos Frontend x Backend

| Campo Frontend | Campo Backend | Tipo | Formatação | Observações |
|----------------|---------------|------|------------|-------------|
| `full_name` | `firstname` | string | - | Nome completo |
| `email` | `email` | string | - | Email do corretor |
| `phone` | `phone` | string | Sem formatação | Telefone |
| `-` | `cpf` | string | - | cpf |
| `-` | `cnpj` | string | - | cnpj |
| `-` | `cep` | string | - | CEP |
| `-` | `city` | string | - | Cidade |
| `-` | `state` | string | - | Estado |
| `-` | `street` | string | - | Rua |
| `-` | `neighboardhoot` | string | - | Bairro |
| `-` | `number` | string | - | Número da moradia |
| `-` | `susep_number` | string | - | Rua |
| `-` | `broker_name` | string | - | Nome do corretor |
| `broker__rua` | `broker__rua` | string | - | Rua |
| `broker__numero` | `broker__numero` | string | - | Número |
| `broker__bairro` | `broker__bairro` | string | - | Bairro |
| `broker__cidade` | `broker__cidade` | string | - | Cidade |
| `-` | `broker__estado` | string | - | Cidade |
| `broker__complemento` | -| string | - | Complemento |
| `broker__uf` | `broker__uf` | string | - | UF |
| `broker__cep` | `broker__cep` | string | - | CEP |
| `broker__nome_fantasia_da_empresa` | `broker__nome_fantasia_da_empresa` | string | - | Nome fantasia |
| - | `broker__razao_social` | string | - | Nome da empresa |
| `broker__indicacao` | `broker__indicacao` | string | - | Indicação |
| `broker__fonte` | `broker__fonte` | string | - | Fonte |
| `broker__meio` | `broker__meio` | string | - | Meio |
| `utm__content` | `utm__content` | string | - | UTM Content |
| `utm__medium` | `utm__medium` | string | - | UTM Medium |
| `utm__source` | `utm__source` | string | - | UTM Source |
| `broker__susep_pf` | `broker__susep_pf` | string | - | SUSEP Pessoa Física |
| `broker__susep_pj` | `broker__susep_pj` | string | - | SUSEP Pessoa Jurídica |
| `broker__cnpj` | `broker__cnpj` | string | Formatado | CNPJ formatado |
| `broker__seu_cpf` | `broker__seu_cpf` | string | Formatado | CPF formatado (Cadastro PF) |
| `-` | `broker_razao_social` | string | - | Razão social (PJ) |
| `-` | `broker__digito_ag` | string | - | Razão social (PJ) |
| `-` | `broker__agencia` | string | - | Razão social (PJ) |
| `-` | `broker__conta` | string | - | Razão social (PJ) |
| `-` | `broker__nome_do_banco` | string | - | Razão social (PJ) |
| `-` | `broker__codigo_do_banco` | string | - | Razão social (PJ) |
| `-` | `broker__tipo_de_conta` | string | - | Razão social (PJ) |
| `-` | `broker__valor_da_aliquota_iss` | string | - | Razão social (PJ) |
| `-` | `broker__possui_chave_pix_` | string | - | Razão social (PJ) |
| `-` | `broker__chave_pix` | string | - | Razão social (PJ) |
| `-` | `broker__code_emissao_nf` | string | - | Razão social (PJ) |
| `-` | `broker__code_emissao_nf` | string | - | Razão social (PJ) |
| `-` | `broker__code_emissao_nf` | string | - | Razão social (PJ) |
| `-` | `broker__voce_e_optante_pelo_simples_nacional_` | string | - | Como conheceu a Azos |
| `-` | `broker__inscricao_municipal` | boolean | "Sim"/"Não" | Autorização WhatsApp |
| `-` | `melhor_email` | string | - | Melhor e-mail |
| `-` | `status_do_processo_de_ativacao` | string | - | Status de ativação do corretor |
| `broker_como_voce_conheceu_a_azos_` | `-` | string | - | Como conheceu a Azos |
| `broker_intencao_do_corretor` | `-` | string | - | Intenção do corretor |
| `broker_escritorio_de_investimentos` | `-` | string | - | Escritório de Investimentos |
| `broker_voce_ja_fez_parte_de_uma_escola_de_formacao_de_corretores_de_seguro_de_vida_` | `-` | string | - | Já fez parte de Escola de Corretores de Seguro de Vida |
| `broker_qual_escola_de_formacao_` | `-` | string | - | Qual escola de formação |
| `broker_qual_o_tamanho_da_corretora_` | `-` | string | - | Tamanho da corretora |
| `broker_qual_e_o_tipo_de_seguro_que_voce_mais_comercializa_` | `-` | string | - | Tipo de seguro mais comercializado |
| `broker_quais_desses_produtos_voce_vende_regularmente_` | `-` | string | - | Produtos vendidos regularmente |
| `broker_voce_tem_intencao_de_comercializar_produtos_de_seguro_de_vida_individual_` | `-` | string | - | Deseja vender seguro de vida individual |
| `broker_ha_quanto_tempo_voce_vende_seguro_de_vida_individual_` | `-` | string | - | Tempo que vende seguro de vida individual |
| `broker_quantas_apolices_de_seguro_de_vida_individual_voce_vendeu_no_ultimo_mes_` | `-` | string | - | Apólices de seguros vendidas no último mês |


#### 1.1 Envio de Dados Iniciais sem SUSEP - Front-End

endpoint: POST /default/contact

- full_name
- email
- phone (sem formatação)
- broker__como_voce_conheceu_a_azos_ (Sim/ Não)

#### 1.2 Envio de Dados Iniciais com SUSEP - Backend

front-end envia para o aggregation-account (POST /v1/preonboarding):
 - email
 - number_document
 - account (PF/PJ)
 - utm_medium (opcional)
 - utm_source (opcional)
 - broker_fonte (opcional)
 - broker_indicacao (opcional)

back-end monta o payload após consultar o BureauService:

- email
- firstname
- broker__rua
- broker__numero
- broker__bairro
- broker__cidade
- broker__uf
- broker__nome_fantasia_da_empresa
- broker__indicacao
- broker__fonte
- broker__meio
- utm_content
- broker__cep
- utm_medium
- utm_source
- broker__susep_pj ou broker__susep_pf  # Dinâmico baseado no tipo de conta
- broker__cnpj ou broker__seu_cpf # Dinâmico baseado no tipo de conta
- broker__razao_social

Faz o cadastro no hubspot com post_contact_data_hubspot()

#### 1.3 Vizualização de dados iniciais

o front-end recebe o payload response do aggregation-account
 - susep
 - is_valid
 - full_name
 - address
 - fantasy_name

 faz requisição direta ao hubspot com informações adicionais
 (PATCH /default/contact)
 - broker__fonte
 - broker__meio
 - broker__indicacao

 #### 1.4 Caso o endereço não tenha sido preenchido 
 Quando no cadastro, através do back, as informações de endereço não foram obtidas

 o front abre um formulário para preenchimento
 Requisição do front-end diretamente a integração com Hubspot
- broker_rua,
- broker_numero,
- broker_bairro,
- broker_cidade,
- broker_complemento,
- broker_uf,
- broker_cep,
- broker_susep_pf,
- broker_susep_pj

Front faz a atualização do registro no hubspot (PATCH /default/contact)

#### 1.5 Perfil do Cliente PJ
Atualização dos dados do responsável
Requisição do front-end diretamente a integração com Hubspot

- full_name:
- broker_bairro,
- broker_cep,
- broker_cidade,
- broker_complemento,
- broker_numero,
- broker_rua,
- broker_uf,
- broker_nome_fantasia_da_empresa,
- broker_indicacao,
- broker_fonte,
- broker_meio,
- utm_content,
- utm_medium,
- utm_source,
- broker_cnpj, (formatado)
- broker_seu_cpf, (formatado)
- broker_susep_pf,
- broker_susep_pj,

Reenvio de informações de endereço que foi preenchido em etapa anterior

Informações não enviadas que poderiam ser consideradas
- broker_qual_o_seu_genero_
- broker_data_de_nascimento_do_corretor_responsavel
- broker_voce_e_uma_pessoa_politicamente_exposta_
- comunicacao_por_whatsapp

#### 1.6 Cadastro de Senha
Depois do backend fazer as verificações e criar o usuário na base de dados

o frontend faz uma atualização de redundância dos dados
Requisição do front-end diretamente a integração com Hubspot
- email,
- broker_seu_cpf, // Formatado
- broker_cnpj, // Formatado
- broker_susep_pf,
- broker_susep_pj,
- broker_nome_fantasia_da_empresa,
- broker_razao_social,
- broker_nome_do_responsavel_pela_corretora,
- broker_cpf_adicional,
- broker_data_de_nascimento_do_corretor_responsavel,
- broker_qual_o_seu_genero_,
- broker_como_voce_conheceu_a_azos_,
- broker_indicacao,
- broker_fonte,
- broker_meio,
- utm_content,
- utm_medium,
- utm_source,
- broker_bairro,
- broker_cep,
- broker_cidade,
- broker_complemento,
- broker_numero,
- broker_rua,
- broker_uf,
- broker_voce_e_uma_pessoa_politicamente_exposta_,
- comunicacao_por_whatsapp,
- data_de_nascimento, // 1999-10-01
- nome_completo,
- firstname,
- phone, // Sem formatação


### 1.7 Campos validados com o time de Produto
#### **Data de Nascimento:**
- **`broker__data_de_nascimento_do_corretor_responsavel`** - Data de nascimento do corretor responsável
- **`data_de_nascimento`** - Data de nascimento do beneficiário (conta PF)

#### **Número de Telefone:**
- **Campo:** `phone`
- **Requisito:** Obrigatório, apenas números
- **Formato:** `551199999999` (código do país + DDD + número)
- **Exemplo:** `5511987654321`

#### **CPF e CNPJ:**

**Pessoa Jurídica (PJ):**
- **`broker__cpf_cnpj`** - CPF/CNPJ formatado (obrigatório)
- **`broker__cpf_adicional`** - CPF adicional (opcional)

**Pessoa Física (PF):**
- **`broker__seu_cpf`** - CPF do corretor (quando é PF)
- **`broker__cpf_cnpj`** - CPF formatado (quando PF com CPF)

### 2. Resiliência das Requisições ao HubSpot

#### **Problemas Atuais:**
- ❌ Sem retry para falhas temporárias
- ❌ Sem backoff exponencial
- ❌ Falhas em cascata não controladas
- ❌ Logs limitados para debugging

### **Solução Proposta:**

#### **Configuração de Retry:**
- **Máximo de tentativas:** 3
- **Backoff exponencial:** 1s, 2s, 4s
- **Erros retentáveis:** 429, 500, 502, 503, 504
- **Erros não retentáveis:** 400, 401, 403, 404

### 3. Observabilidade na Resiliência

#### **Logs Estruturados:**
 - ação a ser realizada no hubspot
 - número de tentativas
 - dados anonimizados enviados
 - timestamp

#### **Métricas:**
- Taxa de sucesso por operação
- Tempo de resposta médio
- Número de tentativas por requisição
- Erros por tipo (rate limit, timeout, etc.)

#### **Alertas:**
- Falha após 3 tentativas
- Taxa de erro > 5%
- Tempo de resposta > 30s

### 4. Anonimização de Dados Sensíveis

#### **Campos Sensíveis Identificados:**
- `email` → `***@***.***`
- `cpf` → `***.***.***-**`
- `cnpj` → `**.***.***/****-**`
- `phone` → `(**) ****-****`
- `birth_date` → `****-**-**`

### 5. Validações de CPF, CNPJ e Data de Nascimento

#### **Validação de CPF e CNPJ com validate_docbr:**

**Pacote:** `validate-docbr` - Biblioteca Python para validação de documentos brasileiros

**Repositório:** https://github.com/georgeyk/validate-docbr

**Vantagens do validate_docbr:**
- ✅ Validação oficial de documentos brasileiros
- ✅ Suporte a CNPJ alfanumérico
- ✅ Múltiplos formatos (com/sem pontuação)
- ✅ Casos de teste abrangentes
- ✅ Biblioteca mantida e confiável

#### **Validação de Data de Nascimento:**
 - idade mínima 18
 - idade máxima 120 anos


### 6. Proposta de Mudança

#### **Estrutura Atual:**
```
Use Case → HubSpotDispatcher → HTTP Service → HubSpot API
```

#### **Estrutura Proposta:**
```
Use Case → Validation Layer → Retry Layer → HubSpotDispatcher → HTTP Service → HubSpot API
```

#### **Componentes Novos:**

1. **Validation Layer:**
   - Validação de CPF/CNPJ
   - Validação de data de nascimento
   - Validação de formato de email
   - Validação de campos obrigatórios

2. **Retry Layer:**
   - Decorator de retry
   - Logs estruturados

3. **Anonymization Layer:**
   - Anonimização de dados sensíveis para a observabilidade

### Proposta de fluxo de cadastro nas etapas

#### Dados iniciais
- full_name
- email
- number_document
- account

#### Sem SUSEP
- broker__meio
- como_conheceu_a_azos
- comunicacao_por_whatsapp

##### Cadastro PF
 - phone
 - data_de_nascimento
 - broker__fonte
 - broker__meio
 - broker__indicacao

##### Cadastro PJ

 - phone
 - comunicacao_por_whatsapp
 - broker__meio
 - broker__nome_fantasia_da_empresa
 - broker__razao_social
 - broker__fonte
 - broker__meio
 - broker__indicacao

#### Com SUSEP

#### Cadastro PF

 - broker__seu_cpf
 - broker__susep_pf
 - phone
 - data_de_nascimento
 - broker__qual_o_seu_genero_
 - broker__voce_e_uma_pessoa_politicamente_exposta_
 - comunicacao_por_whatsapp

#### Cadastro PJ (com dados do responsável)
 - broker__cpf_cnpj
 - broker__susep_pj
 - broker__cpf_adicional
 - broker__qual_o_seu_genero_
 - broker__data_de_nascimento_do_corretor_responsavel
 - broker__voce_e_uma_pessoa_politicamente_exposta_
 - comunicacao_por_whatsapp
 - phone


#### Endereço

 - "broker__rua
 - broker__bairro
 - broker__cidade
 - broker__numero
 - broker__complemento
 - broker__uf
 - broker__cep

#### Preenchimento de formulário
retirar:
 - full_name
 - email
 - phone 
Já foram informados no início do fluxo.

- broker__como_voce_conheceu_a_azos_
- broker__intencao_do_corretor
- broker__escritorio_de_investimentos
- broker__voce_ja_fez_parte_de_uma_escola_de_formacao_de_corretores_de_seguro_de_vida_
- broker__qual_escola_de_formacao_
- broker__qual_o_tamanho_da_corretora_
- broker__qual_e_o_tipo_de_seguro_que_voce_mais_comercializa_
- broker__quais_desses_produtos_voce_vende_regularmente_
- broker__voce_tem_intencao_de_comercializar_produtos_de_seguro_de_vida_individual_
- broker__ha_quanto_tempo_voce_vende_seguro_de_vida_individual_
- broker__quantas_apolices_de_seguro_de_vida_individual_voce_vendeu_no_ultimo_mes_

#### Alterações no back-end

### 🎯 **Proposta de Novo Endpoint para Fluxo de Cadastro**

#### **Endpoint Principal:**
```
PATCH /v1/hubspot/contact/update
```
email sempre deve vir no payload para poder fazer os patches.
email é o atributo usado no hubspot para identificar o contato.

#### **Funcionalidades:**
2. **Cadastro PF** - Adição de CPF e data de nascimento
3. **Cadastro PJ** - Adição de CNPJ e dados da empresa
4. **Endereço** - Atualização de dados de endereço

Esse endpoint vai ser exposto para o front passar os payloads para fazer a atualização dos dados de cadastro.


## 🔄 **Fluxo de Dados Proposto**

#### **1. Dados Iniciais**
```json
{
  "full_name": "string",
  "email": "string", 
  "number_document": "string", *
  "account": "PF/PJ" *
}
```
## **Sem SUSEP**
```json
{
  "broker__meio": "string",
  "broker__como_voce_conheceu_a_azos_": "string",
  "comunicacao_por_whatsapp": "Sim/Não",
}
```
### 2. Visualização


#### 2.1 PF
```json
{
  "phone": "string",
  "data_de_nascimento": "YYYY-MM-DD",
  "broker__fonte": "string",
  "broker__meio": "string",
  "broker__indicacao": "string"
}
```

#### 2.2 PJ
```json
{
  "phone": "string",
  "data_de_nascimento": "string",
  "broker__nome_fantasia_da_empresa": "string",
  "broker__razao_social": "string",
  "broker__fonte": "string",
  "broker__meio": "string",
  "broker__indicacao": "string"
}
```

## **Com SUSEP**

#### **1. Cadastro PF**
```json
{
  "broker__seu_cpf": "xxx.xxx.xxx.xx",
  "phone": "string",
  "data_de_nascimento": "YYYY-MM-DD",
  "broker__qual_o_seu_genero_": "string",
  "broker__voce_e_uma_pessoa_politicamente_exposta_": "Sim/Não",
  "comunicacao_por_whatsapp": "Sim/Não"
}
```

#### **2. Cadastro PJ**
```json
{
  "broker__cpf_cnpj": "string (formatado)",
  "broker__cpf_adicional": "string (formatado)",
  "broker__qual_o_seu_genero_": "string",
  "broker__data_de_nascimento_do_corretor_responsavel": "YYYY-MM-DD",
  "broker__voce_e_uma_pessoa_politicamente_exposta_": "Sim/Não",
  "comunicacao_por_whatsapp": "Sim/Não",
  "phone": "string",
}
```
#### **5. Endereço**

```json
{
  "broker__rua": "string",
  "broker__bairro": "string",
  "broker__cidade": "string",
  "broker__numero": "string", 
  "broker__complemento": "string",
  "broker__uf": "string",
  "broker__cep": "string"
}
```

#### **6. Formulário**
```json
{
  "broker__como_voce_conheceu_a_azos_": "string",
  "broker__intencao_do_corretor": "string",
  "broker__escritorio_de_investimentos": "string",
  "broker__voce_ja_fez_parte_de_uma_escola_de_formacao_de_corretores_de_seguro_de_vida_": "string",
  "broker__qual_escola_de_formacao_": "string",
  "broker__qual_o_tamanho_da_corretora_": "string",
  "broker__qual_e_o_tipo_de_seguro_que_voce_mais_comercializa_": "string",
  "broker__quais_desses_produtos_voce_vende_regularmente_": "string",
  "broker__voce_tem_intencao_de_comercializar_produtos_de_seguro_de_vida_individual_": "string",
  "broker__ha_quanto_tempo_voce_vende_seguro_de_vida_individual_": "string",
  "broker__quantas_apolices_de_seguro_de_vida_individual_voce_vendeu_no_ultimo_mes_": "string"
}
```

#### **Implementação Sugerida:**

**Fase 1 (1-2 dias):**
- Implementar decorator de retry
- Adicionar validações básicas
- Implementar anonimização

**Fase 2 (2-3 dias):**
- Logs estruturados
- Testes de integração

**Fase 3 (3-4 dias):**
- Padronização dos contratos de dados sugeridos
- Implementação do endpoint de update de dados do contato integrado ao Hubspot

### 📊 **Análise de Impacto**

#### **Benefícios:**
- **Segurança:** Dados sensíveis protegidos
- **Observabilidade:** Debugging facilitado
- **Manutenibilidade:** Código mais robusto

#### **Riscos:**
- **Complexidade:** Adição de camadas
- **Performance:** Overhead mínimo
- **Compatibilidade:** Mudanças em interfaces

### ⏱️ **Estimativa**

**Tempo total:** 6-9 dias
- **Fase 1:** 1-2 dias
- **Fase 2:** 2-3 dias  
- **Fase 3:** 3-4 dias

### 🚧 **Próximos Passos**

1. **Implementar decorator de retry** no HubSpotDispatcher
2. **Adicionar validações** de CPF, CNPJ e data
3. **Implementar anonimização** de dados sensíveis
4. **Configurar logs estruturados** e métricas
5. **Padronizar contratos de dados**
6. **Criar endpoint para atualizar dados do Hubspot**
5. **Criar testes** para validações e retry
6. **Documentar** mudanças e configurações

### ⚠️ **Riscos Identificados**

- **Mudanças em interfaces** podem quebrar integrações existentes
- **Overhead de validação** pode impactar performance
- **Configuração de retry** precisa ser ajustada para cada ambiente
- **Dados sensíveis** em logs podem vazar informações

---

## 🏷️ Tags

`#spike` `#hubspot` `#integration` `#resilience` `#observability` `#data-validation` `#security`

---

*Documento criado para investigação técnica de melhorias na integração HubSpot*
