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

#### **Solução Proposta:**

```python
# Decorator de retry simplificado
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=30),
    retry=retry_if_exception_type((ConnectionError, TimeoutError, HTTPError))
)
async def post_contact_data(self, data: dict) -> HttpResponseAdapter:
    """Post contact data with retry."""
    return await self._http.post(ServiceEnum.HUBSPOT, "/default/contact/", data={"object_properties": data})
```

#### **Configuração de Retry:**
- **Máximo de tentativas:** 3
- **Backoff exponencial:** 1s, 2s, 4s
- **Erros retentáveis:** 429, 500, 502, 503, 504
- **Erros não retentáveis:** 400, 401, 403, 404

### 3. Observabilidade na Resiliência

#### **Logs Estruturados:**
```python
logger.info("HubSpot request attempt", extra={
    "operation": "post_contact_data",
    "attempt": attempt_number,
    "data_keys": list(data.keys()),
    "timestamp": datetime.utcnow().isoformat()
})
```

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

#### **Implementação:**
```python
def anonymize_sensitive_data(data: dict) -> dict:
    """Anonimiza dados sensíveis para logs."""
    sensitive_fields = ['email', 'cpf', 'cnpj', 'phone', 'birth_date']
    anonymized = data.copy()
    
    for field in sensitive_fields:
        if field in anonymized:
            anonymized[field] = "***"
    
    return anonymized
```

### 5. Validações de CPF, CNPJ e Data de Nascimento

#### **Validação de CPF e CNPJ com validate_docbr:**

**Pacote:** `validate-docbr` - Biblioteca Python para validação de documentos brasileiros

**Repositório:** https://github.com/georgeyk/validate-docbr

**Instalação:**
```bash
pip install validate-docbr
```

**Validação de CPF:**
```python
from validate_docbr import CPF

def validate_cpf(cpf: str) -> bool:
    """Valida CPF usando validate_docbr."""
    cpf_validator = CPF()
    return cpf_validator.validate(cpf)
```

**Validação de CNPJ:**
```python
from validate_docbr import CNPJ

def validate_cnpj(cnpj: str) -> bool:
    """Valida CNPJ usando validate_docbr."""
    cnpj_validator = CNPJ()
    return cnpj_validator.validate(cnpj)
```

**Validação de CNPJ Alfanumérico:**
```python
from validate_docbr import CNPJ

def validate_cnpj_alphanumeric(cnpj: str) -> bool:
    """Valida CNPJ alfanumérico usando validate_docbr."""
    cnpj_validator = CNPJ()
    # O validate_docbr já suporta CNPJ alfanumérico
    return cnpj_validator.validate(cnpj)
```

**Casos de Teste:**
```python
# Testes para CPF
def test_cpf_validation():
    assert validate_cpf("11144477735") == True
    assert validate_cpf("111.444.777-35") == True
    assert validate_cpf("00000000000") == False
    assert validate_cpf("12345678901") == False

# Testes para CNPJ
def test_cnpj_validation():
    assert validate_cnpj("11222333000181") == True
    assert validate_cnpj("11.222.333/0001-81") == True
    assert validate_cnpj("00000000000000") == False
    assert validate_cnpj("12345678901234") == False

# Testes para CNPJ alfanumérico
def test_cnpj_alphanumeric_validation():
    assert validate_cnpj_alphanumeric("12.345.678/0001-90") == True
    assert validate_cnpj_alphanumeric("12345678000190") == True
```

**Vantagens do validate_docbr:**
- ✅ Validação oficial de documentos brasileiros
- ✅ Suporte a CNPJ alfanumérico
- ✅ Múltiplos formatos (com/sem pontuação)
- ✅ Casos de teste abrangentes
- ✅ Biblioteca mantida e confiável

#### **Validação de Data de Nascimento:**
```python
def validate_birth_date(birth_date: str) -> bool:
    """Valida data de nascimento."""
    try:
        date_obj = datetime.strptime(birth_date, "%Y-%m-%d")
        age = (datetime.now() - date_obj).days // 365
        return 18 <= age <= 100  # Idade entre 18 e 100 anos
    except ValueError:
        return False
```

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
- phone

#### Cadastro PF
broker_seu_cpf
data_de_nascimento

#### Cadastro PJ
sem susep
 - como_conheceu_a_azos
 - comunicacao_por_whatsapp
com susep
 - broker__cpf_cnpj
 - broker__cpf_adicional

**dados do responsável**
adicionar:
 - broker_qual_o_seu_genero_
 - broker_data_de_nascimento_do_corretor_responsavel
 - broker_voce_e_uma_pessoa_politicamente_exposta_
 - comunicacao_por_whatsapp

#### Endereço
Se foi informado PF, usa os campos de endereço sem prefixo broker__*

Se foi informado PJ, usa os campos de endereço com prefixo broker__*


#### Preenchimento de formulário
retirar:
 - full_name
 - email
 - phone 

Já foram informados no início do fluxo.

#### Alterações no back-end

### 🎯 **Proposta de Novo Endpoint para Fluxo de Cadastro**

#### **Endpoint Principal:**
```
PATCH /v1/hubspot/contact/{email}/update
```

#### **Funcionalidades:**
2. **Cadastro PF** - Adição de CPF e data de nascimento
3. **Cadastro PJ** - Adição de CNPJ e dados da empresa
4. **Endereço** - Atualização de dados de endereço
5. **Dados Bancários** - Informações financeiras (PJ)

Esse endpoint vai ser exposto para o front passar os payloads para fazer a atualização dos dados de cadastro.


### 🔄 **Fluxo de Dados Proposto**

#### **1. Dados Iniciais (Sem SUSEP)**
```json
{
  "full_name": "string",
  "email": "string", 
  "phone": "string",
  "broker__como_voce_conheceu_a_azos_": "string"
}
```

#### **2. Cadastro PF**
```json
{
  "broker__seu_cpf": "string (formatado)",
  "data_de_nascimento": "YYYY-MM-DD",
  "broker__qual_o_seu_genero_": "string",
  "broker__voce_e_uma_pessoa_politicamente_exposta_": "Sim/Não",
  "comunicacao_por_whatsapp": "Sim/Não"
}
```

#### **3. Cadastro PJ (Sem SUSEP)**
```json
{
  "broker__como_voce_conheceu_a_azos_": "string",
  "comunicacao_por_whatsapp": "Sim/Não"
}
```

#### **4. Cadastro PJ (Com SUSEP)**
```json
{
  "broker__cpf_cnpj": "string (formatado)",
  "broker__cpf_adicional": "string (formatado)",
  "broker__qual_o_seu_genero_": "string",
  "broker__data_de_nascimento_do_corretor_responsavel": "YYYY-MM-DD",
  "broker__voce_e_uma_pessoa_politicamente_exposta_": "Sim/Não",
  "comunicacao_por_whatsapp": "Sim/Não"
}
```

#### **5. Endereço**

##### **5.1. Endereço PJ**
```json
{
  "broker__rua": "string",
  "broker__numero": "string", 
  "broker__bairro": "string",
  "broker__cidade": "string",
  "broker__complemento": "string",
  "broker__uf": "string",
  "broker__cep": "string"
}
```

##### **5.2. Endereço PF**
```json
{
  "rua": "string",
  "numero": "string", 
  "bairro": "string",
  "cidade": "string",
  "complemento": "string",
  "uf": "string",
  "cep": "string"
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

**Fase 3 (1-2 dias):**
- Documentação
- Monitoramento avançado
- Otimizações

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

**Tempo total:** 4-7 dias
- **Fase 1:** 1-2 dias
- **Fase 2:** 2-3 dias  
- **Fase 3:** 1-2 dias

### 🚧 **Próximos Passos**

1. **Implementar decorator de retry** no HubSpotDispatcher
2. **Adicionar validações** de CPF, CNPJ e data
3. **Implementar anonimização** de dados sensíveis
4. **Configurar logs estruturados** e métricas
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
