# Spike: Refatoração do Fluxo de Cadastro - Mapeamento de Problemas e Soluções

## 📋 Resumo

Este documento mapeia os problemas identificados no fluxo de cadastro do sistema de contas, incluindo validações de dados, perda de informações entre fluxos e preparação para a evolução do CNPJ. O objetivo é identificar pontos de melhoria e criar um plano de ação para resolver os problemas identificados.

## 🎯 Problemas Identificados

### 1. **Validações de Dados Insuficientes**

#### 1.1 Data de Nascimento
- **Problema**: Data de nascimento sem formatação ou sem preenchimento
- **Localização**: `src/adapters/dtos/user.py` - Campo `birth_date: date`
- **Impacto**: Dados inconsistentes, problemas de validação
- **Status**: ⚠️ **CRÍTICO**

#### 1.2 CNPJ sem Formatação
- **Problema**: CNPJ sem formatação adequada
- **Localização**: Múltiplos DTOs com validação `min_length=14, max_length=14`
- **Arquivos Afetados**:
  - `src/adapters/api/v1/dtos/subscription.py`
  - `src/adapters/api/v1/dtos/platform.py`
  - `src/business/domain/company.py`
- **Impacto**: Dados inconsistentes, problemas de validação
- **Status**: ⚠️ **CRÍTICO**

#### 1.3 Telefone sem Preenchimento
- **Problema**: Telefone sem preenchimento obrigatório
- **Localização**: `src/adapters/dtos/contact.py` - Campo `phone: str`
- **Impacto**: Dados de contato incompletos
- **Status**: ⚠️ **ALTO**

#### 1.4 CPF sem Validação Adequada
- **Problema**: CPF sem validação de dígitos verificadores ou formatação
- **Localização**: 
  - `src/interface_adapters/api/v2/dtos/onboarding.py` - `UserDTO.cpf: str | None`
  - `src/interface_adapters/api/v2/dtos/onboarding.py` - `CompanyDTO.cpf: str | None`
  - `src/interface_adapters/api/common/dtos/professional.py` - `cpf: str | None = Field(min_length=11, max_length=11)`
- **Endpoint Afetado**: `POST /v2/onboarding/broker-group`
- **Validação Atual**: Apenas `min_length=11, max_length=11` (sem validação de dígitos verificadores)
- **Impacto**: CPFs inválidos podem ser aceitos, dados inconsistentes
- **Status**: ⚠️ **CRÍTICO**

#### 1.5 Inconsistência Frontend vs Backend - Formatação de Dados
- **Problema**: Dados podem chegar formatados ou não formatados do frontend, causando inconsistências
- **Descobertas**:
  - **Formatação Acontece APENAS na Saída**: Sistema formata CPF/CNPJ apenas ao enviar para HubSpot
  - **Método `format_document()`**: Formata CPF como `XXX.XXX.XXX-XX` e CNPJ como `XX.XXX.XXX/XXXX-XX`
  - **Dados Chegam "Crus"**: Frontend pode enviar com ou sem formatação
  - **Sem Padronização**: Não há normalização dos dados na entrada
- **Localização**: 
  - `src/application_domain/fetch_broker_data_requests.py` - `format_document()`
  - `src/application_domain/update_broker_lead.py` - `format_document()`
- **Impacto**: Dados inconsistentes entre sistemas, dificuldade de validação
- **Status**: ⚠️ **CRÍTICO**

### 2. **Perda de Dados entre Fluxos de Cadastro**

#### 2.1 Problema de Migração entre Fluxos
- **Problema**: Corretor inicia cadastro no fluxo A (plataforma) e termina no fluxo B (individual)
- **Impacto**: Dados se perdem no HubSpot
- **Localização**: Sistema de `external_id` e sincronização
- **Status**: ⚠️ **CRÍTICO**

#### 2.2 Retrabalho na Atualização de Status
- **Problema**: Retrabalho na atualização de status entre plataformas (backoffice e HubSpot)
- **Localização**: Sistema de sincronização com `external_id`
- **Impacto**: Ineficiência operacional, dados desatualizados
- **Status**: ⚠️ **ALTO**

#### 2.3 Falta de Rastreabilidade de Fluxo
- **Problema**: Dificuldade para rastrear o fluxo completo do usuário entre diferentes etapas
- **Descobertas**:
  - **Sistema de Observabilidade Existe**: Datadog configurado com tags e contextos
  - **External ID Disponível**: Usado para identificação única do usuário
  - **Logs Estruturados**: Já implementados nos use cases
  - **Registration Flow**: Sistema de status de cadastro implementado
- **Oportunidade**: Usar observabilidade + validações + external_id para rastrear fluxo completo
- **Status**: ⚠️ **MÉDIO** (Solução já parcialmente implementada)

### 3. **Evolução do CNPJ - Novo Formato Alfanumérico**

#### 3.1 Preparação para CNPJ com Letras e Números
- **Problema**: Sistema não está preparado para o novo formato de CNPJ
- **Nova Regulamentação**: A partir de julho de 2026, CNPJ terá formato alfanumérico
- **Estrutura do Novo CNPJ**:
  - 14 caracteres total
  - 8 primeiras posições (raiz): letras e números
  - 4 posições seguintes (ordem): alfanuméricas
  - 2 últimas posições (dígitos verificadores): numéricas
- **Status**: ⚠️ **CRÍTICO - URGENTE**

## 🔍 Análise Técnica Detalhada

### Estrutura Atual do Sistema

#### Fluxos de Cadastro Identificados
1. **Fluxo A - Plataforma**: `CreatePlatformUseCase`
2. **Fluxo B - Corretor Individual**: `EnrollProfessionalAccountUseCase`
3. **Fluxo C - Broker Group**: `EnrollBrokerGroupUseCase`

#### Validações Atuais
```python
# CNPJ atual - apenas números
cnpj: str = Field(min_length=14, max_length=14)

# CPF atual - apenas tamanho, sem validação de dígitos verificadores
cpf: str | None = Field(min_length=11, max_length=11)

# Data de nascimento - sem validação de formato
birth_date: date

# Telefone - sem validação de formato
phone: str
```

#### Endpoint POST /v2/onboarding/broker-group
- **Estrutura de Dados**:
  ```python
  class OnboardingBrokerGroupInputDTO:
      is_manual_fill: bool
      company: CompanyDTO  # contém cpf: str | None
      user: UserDTO        # contém cpf: str | None
  ```
- **Validação Atual**: Apenas verifica se CNPJ ou CPF está presente (`if not input_port.broker_group.cnpj and not input_port.broker_group.cpf`)
- **Problema**: Não há validação de dígitos verificadores do CPF

#### Fluxo de Dados - Frontend → Backend → Serviços Externos
```
1. FRONTEND → BACKEND
   - Dados podem chegar formatados ou não: "12345678901" ou "123.456.789-01"
   - Sem normalização na entrada

2. BACKEND → PROCESSAMENTO INTERNO
   - Dados passam sem formatação entre use cases
   - Validação apenas de tamanho (min_length/max_length)

3. BACKEND → SERVIÇOS EXTERNOS (HubSpot)
   - Formatação aplicada APENAS na saída via format_document()
   - CPF: "123.456.789-01"
   - CNPJ: "12.345.678/0001-90"
```

#### Problemas Identificados no Fluxo
- **Inconsistência de Entrada**: Frontend pode enviar formatado ou não
- **Falta de Normalização**: Dados não são padronizados na entrada
- **Formatação Tardia**: Apenas na saída para HubSpot
- **Sem Validação Robusta**: Apenas validação de tamanho

#### Sistema de Sincronização
- **External ID**: Usado para sincronização entre sistemas
- **Status Management**: `RegistrationFlowEnum` para controle de status


## 🚨 Impactos Identificados

### Impactos Técnicos
1. **Validações Inadequadas**: Dados inconsistentes no sistema
2. **Perda de Dados**: Informações perdidas na migração entre fluxos
3. **Incompatibilidade Futura**: Sistema não suportará novo formato de CNPJ

### Impactos Operacionais
1. **Retrabalho**: Atualizações manuais de status
2. **Inconsistência**: Dados desatualizados entre plataformas
3. **Ineficiência**: Processos manuais para correção de dados

## 📊 Mapeamento de Arquivos Afetados

### DTOs que Precisam de Atualização
```
src/adapters/dtos/
├── user.py                    # birth_date validation
├── contact.py                 # phone validation
├── professional.py           # CNPJ validation
└── company.py                # CNPJ validation

src/interface_adapters/api/v2/dtos/
├── onboarding.py             # CPF validation (UserDTO, CompanyDTO)
└── professional.py           # CPF validation
```

### APIs que Precisam de Atualização
```
src/adapters/api/v1/dtos/
├── subscription.py           # CNPJ validation
├── platform.py              # CNPJ validation
└── account.py               # Data validation
```

### Use Cases que Precisam de Atualização
```
src/business/use_cases/
├── create_platform.py        # CNPJ validation
├── enroll_professional_account.py  # Data validation
└── create_registration_account_flow.py  # Status management
```

## 🎯 Plano de Ação Recomendado

### Fase 1: Validações de Dados (Prioridade ALTA)
1. **Implementar validação de data de nascimento**
   - Formato obrigatório
   - Validação de idade mínima/máxima
   - Tratamento de timezone

2. **Implementar formatação de CNPJ**
   - Máscara de formatação
   - Validação de dígitos verificadores
   - Tratamento de caracteres especiais

3. **Implementar validação de telefone**
   - Formato obrigatório
   - Validação de DDD
   - Máscara de formatação

4. **Implementar validação de CPF**
   - Validação de dígitos verificadores
   - Formatação automática
   - Tratamento de caracteres especiais

5. **Implementar normalização de dados na entrada**
   - Padronizar formato de CPF/CNPJ na recepção
   - Remover formatação existente
   - Aplicar validações antes do processamento

### Fase 2: Preparação para CNPJ Alfanumérico (Prioridade CRÍTICA)
1. **Atualizar validações de CNPJ**
   - Suporte a caracteres alfanuméricos
   - Nova lógica de validação
   - Backward compatibility

2. **Atualizar DTOs e Models**
   - Campos de CNPJ para aceitar alfanumérico
   - Validações customizadas
   - Testes de compatibilidade

### Fase 3: Sincronização de Dados (Prioridade ALTA)
1. **Implementar sistema de migração entre fluxos**
   - Preservação de dados
   - Mapeamento de campos
   - Logs de auditoria

2. **Melhorar sincronização com HubSpot**
   - Identificação de integração atual
   - Implementação de sincronização bidirecional
   - Tratamento de conflitos

3. **Implementar rastreamento completo de fluxo**
   - Usar observabilidade + validações + external_id
   - Tags estruturadas para cada etapa do fluxo
   - Correlação entre sistemas via external_id

### Fase 4: Monitoramento e Observabilidade
1. **Implementar logs de auditoria**
   - Rastreamento de mudanças
   - Logs de sincronização
   - Alertas de falhas

2. **Implementar métricas**
   - Taxa de sucesso de cadastros
   - Tempo de sincronização
   - Qualidade dos dados

## 🔧 Implementações Técnicas Necessárias

### 1. Validações de CNPJ para Formato Alfanumérico
```python
# Exemplo de validação para novo formato
@validator("cnpj")
def validate_cnpj_format(cls, v: str) -> str:
    if not v:
        raise ValueError("CNPJ é obrigatório")
    
    # Remover caracteres especiais
    clean_cnpj = re.sub(r'[^A-Za-z0-9]', '', v)
    
    # Validar tamanho
    if len(clean_cnpj) != 14:
        raise ValueError("CNPJ deve ter 14 caracteres")
    
    # Validar formato alfanumérico (a partir de 2026)
    if not re.match(r'^[A-Za-z0-9]{8}[A-Za-z0-9]{4}[0-9]{2}$', clean_cnpj):
        raise ValueError("CNPJ deve seguir o formato: 8 alfanuméricos + 4 alfanuméricos + 2 numéricos")
    
    return clean_cnpj
```

### 2. Validação de Data de Nascimento
```python
@validator("birth_date")
def validate_birth_date(cls, v: date) -> date:
    if not v:
        raise ValueError("Data de nascimento é obrigatória")
    
    # Validar idade mínima (18 anos)
    min_age = 18
    max_age = 100
    
    today = date.today()
    age = today.year - v.year - ((today.month, today.day) < (v.month, v.day))
    
    if age < min_age:
        raise ValueError(f"Idade mínima é {min_age} anos")
    
    if age > max_age:
        raise ValueError(f"Idade máxima é {max_age} anos")
    
    return v
```

### 3. Validação de Telefone
```python
@validator("phone")
def validate_phone(cls, v: str) -> str:
    if not v:
        raise ValueError("Telefone é obrigatório")
    
    # Remover caracteres especiais
    clean_phone = re.sub(r'[^0-9]', '', v)
    
    # Validar DDD e número
    if not re.match(r'^[1-9]{2}9[0-9]{8}$', clean_phone):
        raise ValueError("Telefone deve estar no formato: (XX) 9XXXX-XXXX")
    
    return clean_phone
```

### 4. Validação de CPF
```python
@validator("cpf")
def validate_cpf(cls, v: str | None) -> str | None:
    if not v:
        return None
    
    # Remover caracteres especiais
    clean_cpf = re.sub(r'[^0-9]', '', v)
    
    # Validar tamanho
    if len(clean_cpf) != 11:
        raise ValueError("CPF deve ter 11 dígitos")
    
    # Validar se não são todos os dígitos iguais
    if clean_cpf == clean_cpf[0] * 11:
        raise ValueError("CPF inválido")
    
    # Validar dígitos verificadores
    def calculate_digit(cpf_digits: str, position: int) -> int:
        sum_result = sum(int(digit) * (position - i) for i, digit in enumerate(cpf_digits))
        remainder = sum_result % 11
        return 0 if remainder < 2 else 11 - remainder
    
    # Validar primeiro dígito verificador
    first_digit = calculate_digit(clean_cpf[:9], 10)
    if int(clean_cpf[9]) != first_digit:
        raise ValueError("CPF inválido - primeiro dígito verificador incorreto")
    
    # Validar segundo dígito verificador
    second_digit = calculate_digit(clean_cpf[:10], 11)
    if int(clean_cpf[10]) != second_digit:
        raise ValueError("CPF inválido - segundo dígito verificador incorreto")
    
    return clean_cpf
```

### 5. Normalização de Dados na Entrada
```python
@validator("cpf")
def normalize_cpf(cls, v: str | None) -> str | None:
    if not v:
        return None
    
    # Remover toda formatação existente
    clean_cpf = re.sub(r'[^0-9]', '', v)
    
    # Validar tamanho
    if len(clean_cpf) != 11:
        raise ValueError("CPF deve ter 11 dígitos")
    
    # Validar dígitos verificadores (mesma lógica do exemplo anterior)
    # ... código de validação ...
    
    # Retornar sempre sem formatação para processamento interno
    return clean_cpf

@validator("cnpj")
def normalize_cnpj(cls, v: str | None) -> str | None:
    if not v:
        return None
    
    # Remover toda formatação existente
    clean_cnpj = re.sub(r'[^0-9]', '', v)
    
    # Validar tamanho
    if len(clean_cnpj) != 14:
        raise ValueError("CNPJ deve ter 14 dígitos")
    
    # Validar dígitos verificadores
    # ... código de validação ...
    
    # Retornar sempre sem formatação para processamento interno
    return clean_cnpj
```

### 6. Sistema de Rastreamento de Fluxo com Observabilidade
```python
# Exemplo de rastreamento usando observabilidade + external_id
class FlowTracker:
    def __init__(self, observability_service: ObservabilityService):
        self._obs = observability_service
    
    def track_flow_step(self, external_id: str, step: str, data: dict):
        """Rastrear cada etapa do fluxo com tags estruturadas"""
        self._obs.set_tag("user.external_id", external_id)
        self._obs.set_tag("flow.step", step)
        self._obs.set_tag("flow.timestamp", datetime.now().isoformat())
        self._obs.set_context("flow.data", data)
        
        # Log estruturado para correlação
        logging.info(f"Flow step: {step} for user: {external_id}", extra={
            "external_id": external_id,
            "flow_step": step,
            "flow_data": data
        })
    
    def track_validation(self, external_id: str, field: str, status: str, error: str = None):
        """Rastrear validações com external_id"""
        self._obs.set_tag("user.external_id", external_id)
        self._obs.set_tag("validation.field", field)
        self._obs.set_tag("validation.status", status)
        
        if error:
            self._obs.set_tag("validation.error", error)
            self._obs.set_level("warning")
        
        logging.info(f"Validation: {field} = {status} for user: {external_id}", extra={
            "external_id": external_id,
            "validation_field": field,
            "validation_status": status,
            "validation_error": error
        })
    
    def track_migration(self, external_id: str, from_flow: str, to_flow: str, status: str):
        """Rastrear migração entre fluxos"""
        self._obs.set_tag("user.external_id", external_id)
        self._obs.set_tag("migration.from_flow", from_flow)
        self._obs.set_tag("migration.to_flow", to_flow)
        self._obs.set_tag("migration.status", status)
        
        logging.info(f"Migration: {from_flow} -> {to_flow} = {status} for user: {external_id}", extra={
            "external_id": external_id,
            "migration_from": from_flow,
            "migration_to": to_flow,
            "migration_status": status
        })

# Uso nos Use Cases
class EnrollProfessionalUseCase(UseCase):
    def __init__(self, account_service, core_service, flow_tracker: FlowTracker):
        self._flow_tracker = flow_tracker
    
    async def execute(self, input_port: EnrollProfessionalInputPort):
        # Rastrear início do fluxo
        self._flow_tracker.track_flow_step(
            input_port.user.external_id,
            "enroll_professional_start",
            {"user_type": "professional", "is_sub_user": input_port.is_sub_user}
        )
        
        # Rastrear validações
        if input_port.user.cpf:
            self._flow_tracker.track_validation(
                input_port.user.external_id,
                "cpf",
                "valid" if self._validate_cpf(input_port.user.cpf) else "invalid"
            )
        
        # ... resto da lógica ...
        
        # Rastrear conclusão
        self._flow_tracker.track_flow_step(
            input_port.user.external_id,
            "enroll_professional_complete",
            {"user_id": result.user_id}
        )
```

### 7. Sistema de Migração entre Fluxos
```python
# Exemplo de sistema de migração com backup
async def migrate_user_flow(user_id: str, from_flow: str, to_flow: str):
    # Criar backup antes da migração
    backup_data = await create_user_backup(user_id)
    
    try:
        # Executar migração
        result = await execute_flow_migration(user_id, from_flow, to_flow)
        
        # Log da migração
        await log_migration(user_id, from_flow, to_flow, "SUCCESS")
        
        return result
    except Exception as e:
        # Rollback em caso de falha
        await restore_user_backup(backup_data)
        await log_migration(user_id, from_flow, to_flow, "FAILED", str(e))
        raise
```

### 5. Sincronização com HubSpot
```python
# Exemplo de sistema de sincronização com retry
async def sync_with_hubspot(user_data: dict, retry_count: int = 3):
    for attempt in range(retry_count):
        try:
            response = await hubspot_client.update_contact(user_data)
            return response
        except Exception as e:
            if attempt == retry_count - 1:
                # Última tentativa falhou, enviar para fila de retry
                await queue_retry_sync(user_data, delay_hours=24)
                raise
            else:
                # Aguardar antes da próxima tentativa
                await asyncio.sleep(2 ** attempt)
```

## 📈 Métricas de Sucesso

### Indicadores Técnicos
- [ ] 100% dos CNPJs validados corretamente
- [ ] 100% dos CPFs validados com dígitos verificadores
- [ ] 100% das datas de nascimento formatadas
- [ ] 100% dos telefones validados
- [ ] 100% dos dados normalizados na entrada (sem formatação)
- [ ] 0% de perda de dados entre fluxos

### Indicadores Operacionais
- [ ] Redução de 90% no retrabalho de status
- [ ] Sincronização automática com HubSpot
- [ ] Tempo de cadastro reduzido em 50%
- [ ] Rastreabilidade completa de fluxo via external_id
- [ ] Correlação de logs entre sistemas

## 💰 Estimativa de Esforço

### Fase 1: Validações de Dados
- **Esforço**: 3-5 dias
- **Complexidade**: Média
- **Riscos**: Baixo
- **Dependências**: Nenhuma

### Fase 2: Preparação CNPJ Alfanumérico
- **Esforço**: 8-12 dias
- **Complexidade**: Alta
- **Riscos**: Alto (mudança regulatória)
- **Dependências**: Testes extensivos

### Fase 3: Sincronização de Dados
- **Esforço**: 5-8 dias
- **Complexidade**: Média
- **Riscos**: Médio
- **Dependências**: Integração HubSpot

### Fase 4: Monitoramento
- **Esforço**: 2-3 dias
- **Complexidade**: Baixa
- **Riscos**: Baixo
- **Dependências**: Fases anteriores

**Total Estimado**: 18-28 dias úteis

## 🎯 Soluções Propostas

### Solução 1: Validações Robustas
- **Implementar validadores customizados** para todos os campos críticos
- **Adicionar máscaras de formatação** automática
- **Criar validações de negócio** específicas (idade mínima, DDD válido, etc.)

### Solução 2: Sistema de Migração de Dados
- **Implementar histórico de migrações** entre fluxos
- **Criar backup automático** antes de conversões
- **Implementar rollback** em caso de falha

### Solução 3: Sincronização Inteligente
- **Implementar fila de sincronização** com retry automático
- **Criar sistema de conflitos** para dados divergentes
- **Implementar monitoramento** de falhas de sincronização

### Solução 4: Preparação para CNPJ Alfanumérico
- **Refatorar validações** para suportar caracteres alfanuméricos
- **Implementar validação flexível** (numérico + alfanumérico)
- **Criar migração gradual** com backward compatibility

### Solução 5: Rastreamento Completo de Fluxo
- **Usar observabilidade + validações + external_id** para rastrear fluxo completo
- **Implementar tags estruturadas** para cada etapa do processo
- **Correlacionar logs** entre sistemas via external_id
- **Monitorar migrações** entre fluxos em tempo real
- **Alertas automáticos** para falhas de validação ou sincronização

## ⚠️ Riscos Identificados

### Riscos Técnicos
1. **Quebra de compatibilidade** com sistemas legados
2. **Performance degradada** com novas validações
3. **Falhas de sincronização** durante migração

### Riscos de Negócio
1. **Perda de dados** durante migração entre fluxos
2. **Incompatibilidade** com novo formato de CNPJ
3. **Retrabalho operacional** durante transição

### Riscos Regulatórios
1. **Não conformidade** com nova regulamentação CNPJ
2. **Penalidades** por dados inconsistentes
3. **Bloqueio de funcionalidades** por validações inadequadas

## 🚀 Próximos Passos

1. **Aprovação do Spike**: Validar com stakeholders
2. **Priorização**: Definir ordem de implementação
3. **Estimativa**: Calcular esforço necessário
4. **Implementação**: Executar plano de ação
5. **Monitoramento**: Acompanhar métricas de sucesso

## 📚 Referências

- [Receita Federal - Novo Formato CNPJ](https://www.gov.br/receitafederal/pt-br/assuntos/noticias/2024/outubro/cnpj-tera-letras-e-numeros-a-partir-de-julho-de-2026)
- [Instrução Normativa RFB nº 2.229](https://www.gov.br/receitafederal/pt-br/assuntos/noticias/2024/outubro/cnpj-tera-letras-e-numeros-a-partir-de-julho-de-2026)

---

**Data de Criação**: 20 de dezembro de 2024
**Projeto**: aggregation-account  
**Autor**: Willames de Jesus Campos  
**Versão**: 1.0  
**Status**: Aguardando Aprovação
