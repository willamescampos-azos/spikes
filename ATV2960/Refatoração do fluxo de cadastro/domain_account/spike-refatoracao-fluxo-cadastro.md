# Spike: Refatoração do Fluxo de Cadastro - Mapeamento de Problemas e Soluções

## 📋 Resumo Executivo

Este documento mapeia os problemas identificados no fluxo de cadastro do sistema domain-account e propõe soluções para os seguintes cenários críticos:

1. **Dados sem formatação ou preenchimento** (data de nascimento, CNPJ, telefone)
2. **Perda de dados entre fluxos A e B** (plataforma → corretor individual)
3. **Retrabalho na sincronização** entre backoffice e HubSpot
4. **Evolução do CNPJ** para suportar letras e números a partir de 2025

## 🔍 Análise da Arquitetura Atual

### Estrutura do Projeto
```
src/
├── adapters/
│   ├── api/           # APIs REST (v1, v2, v3)
│   ├── data/          # Repositórios e modelos
│   └── dtos/          # Data Transfer Objects
├── business/
│   ├── domain/        # Entidades de domínio
│   ├── use_cases/     # Casos de uso
│   └── enums/         # Enumerações
└── frameworks/        # Infraestrutura
```

### Fluxos de Cadastro Identificados

#### 1. **Fluxo A - Cadastro via Plataforma**
- **Endpoint**: `/v1/company/enroll-platform-company`
- **Use Case**: `EnrollPlatformCompanyUseCase`
- **Perfil**: `PLATFORM`
- **Responsabilidade**: `PLATFORM_MANAGER`

#### 2. **Fluxo B - Cadastro Individual**
- **Endpoint**: `/v1/account/enroll-professional-account`
- **Use Case**: `EnrollProfessionalAccountUseCase`
- **Perfil**: `INDIVIDUAL_BROKER`
- **Responsabilidade**: `INDIVIDUAL_BROKER`

#### 3. **Fluxo de Registro de Conta**
- **Endpoint**: `/v1/registration-account-flow`
- **Use Case**: `CreateRegistrationAccountFlowUseCase`
- **Status**: `DATA_COMPLETED`, `PENDING`, `REFUSED`

## 🚨 Problemas Identificados

### 1. **Validações de Dados Insuficientes**

#### **Data de Nascimento**
```python
# src/adapters/dtos/user.py
class UserDTO(DTO):
    birth_date: date  # ❌ Sem validação de formato
```

**Problemas encontrados:**
- Não há validação de formato de entrada
- Não há validação de idade mínima/máxima
- Não há formatação consistente

#### **CNPJ**
```python
# src/adapters/api/v1/dtos/platform.py
cnpj: str = Field(min_length=14, max_length=14)  # ❌ Apenas validação de tamanho
```

**Problemas encontrados:**
- Validação apenas de tamanho (14 caracteres)
- Não há validação de dígitos verificadores
- Não há preparação para CNPJ com letras e números
- Formatação inconsistente

#### **Telefone**
```python
# src/adapters/dtos/contact.py
class ContactDTO(DTO):
    phone: str  # ❌ Sem validação de formato
```

**Problemas encontrados:**
- Sem validação de formato brasileiro
- Sem máscara de formatação
- Sem validação de DDD

### 2. **Perda de Dados Entre Fluxos**

#### **Problema de Sincronização**
- **External ID**: Usado para rastreamento entre sistemas
- **Status Management**: `RegistrationFlowEnum` não sincroniza entre fluxos
- **HubSpot Integration**: Não encontrada integração direta no código

#### **Pontos de Falha Identificados:**
```python
# src/business/use_cases/create_registration_account_flow.py
if registration_exists.current_step != RegistrationFlowEnum.DATA_COMPLETED.value:
    raise BusinessError("You can't update the registration flow...")
```

### 3. **Preparação para Evolução do CNPJ**

#### **Situação Atual:**
- Todas as validações assumem CNPJ com 14 dígitos numéricos
- Não há preparação para alfanumérico
- Validações hardcoded em múltiplos DTOs

#### **Impacto:**
- **2025**: CNPJ poderá conter letras e números
- **Sistemas legados**: Precisarão de adaptação
- **Validações**: Todas precisarão ser atualizadas

## 🛠️ Soluções Propostas

### 1. **Validações de Dados Robustas**

#### **Data de Nascimento**
```python
from datetime import date, datetime
from pydantic import validator

class UserDTO(DTO):
    birth_date: date
    
    @validator("birth_date", pre=True)
    def parse_birth_date(cls, v):
        """Parse birth date from various formats"""
        if isinstance(v, str):
            formats = [
                "%d/%m/%Y", "%d-%m-%Y", "%Y-%m-%d",
                "%d/%m/%y", "%d-%m-%y"
            ]
            for fmt in formats:
                try:
                    return datetime.strptime(v, fmt).date()
                except ValueError:
                    continue
            raise ValueError("Formato de data inválido")
        return v
    
    @validator("birth_date")
    def validate_age(cls, v):
        """Validate age constraints"""
        today = date.today()
        age = today.year - v.year - ((today.month, today.day) < (v.month, v.day))
        
        if age < 18:
            raise ValueError("Idade mínima de 18 anos")
        if age > 120:
            raise ValueError("Idade máxima de 120 anos")
        return v
```

#### **CNPJ com Preparação para Evolução**
```python
import re
from typing import Union

class CNPJValidator:
    """CNPJ validator with future alphanumeric support"""
    
    @staticmethod
    def validate_cnpj(cnpj: str) -> bool:
        """Validate CNPJ format (current and future)"""
        # Remove formatting
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', cnpj)
        
        # Current format: 14 digits
        if len(clean_cnpj) == 14 and clean_cnpj.isdigit():
            return CNPJValidator._validate_digits(clean_cnpj)
        
        # Future format: alphanumeric (to be implemented)
        # if len(clean_cnpj) == 14 and re.match(r'^[0-9A-Za-z]+$', clean_cnpj):
        #     return CNPJValidator._validate_alphanumeric(clean_cnpj)
        
        return False
    
    @staticmethod
    def _validate_digits(cnpj: str) -> bool:
        """Validate current numeric CNPJ"""
        # Implementation of CNPJ validation algorithm
        pass

class CreatePlatformInputDTO(InputDTO):
    cnpj: str = Field(description="CNPJ of the company")
    
    @validator("cnpj")
    def validate_cnpj_format(cls, v):
        if not CNPJValidator.validate_cnpj(v):
            raise ValueError("CNPJ inválido")
        return v
```

#### **Telefone Brasileiro**
```python
import re

class ContactDTO(DTO):
    phone: str
    
    @validator("phone")
    def validate_phone_format(cls, v):
        """Validate Brazilian phone format"""
        # Remove all non-digits
        digits = re.sub(r'\D', '', v)
        
        # Brazilian phone patterns
        patterns = [
            r'^1[1-9]\d{8}$',  # Mobile: 11xxxxxxxxx
            r'^[1-9]\d{7,8}$',  # Landline: xxxxxxxx or xxxxxxxxx
        ]
        
        if not any(re.match(pattern, digits) for pattern in patterns):
            raise ValueError("Formato de telefone inválido")
        
        return v
    
    @validator("phone")
    def format_phone(cls, v):
        """Format phone number"""
        digits = re.sub(r'\D', '', v)
        if len(digits) == 11:
            return f"({digits[:2]}) {digits[2:7]}-{digits[7:]}"
        elif len(digits) == 10:
            return f"({digits[:2]}) {digits[2:6]}-{digits[6:]}"
        return v
```

### 2. **Sistema de Sincronização Entre Fluxos**

#### **Registry Pattern para Fluxos**
```python
from enum import Enum
from typing import Dict, Any
from dataclasses import dataclass

class FlowType(Enum):
    PLATFORM = "platform"
    INDIVIDUAL = "individual"
    BROKER_GROUP = "broker_group"

@dataclass
class FlowRegistry:
    """Registry to track user across different flows"""
    external_id: str
    current_flow: FlowType
    previous_flows: list[FlowType]
    data_snapshot: Dict[str, Any]
    last_sync: datetime
    
class FlowSynchronizationService:
    """Service to manage data synchronization between flows"""
    
    async def register_flow_transition(
        self, 
        external_id: str, 
        from_flow: FlowType, 
        to_flow: FlowType,
        data: Dict[str, Any]
    ) -> FlowRegistry:
        """Register transition between flows"""
        pass
    
    async def sync_data_between_flows(
        self, 
        external_id: str, 
        source_flow: FlowType,
        target_flow: FlowType
    ) -> bool:
        """Synchronize data between flows"""
        pass
```

#### **HubSpot Integration Service**
```python
class HubSpotSyncService:
    """Service to sync data with HubSpot"""
    
    async def sync_user_data(self, user_data: Dict[str, Any]) -> bool:
        """Sync user data to HubSpot"""
        pass
    
    async def update_user_status(
        self, 
        external_id: str, 
        status: str
    ) -> bool:
        """Update user status in HubSpot"""
        pass
    
    async def handle_flow_transition(
        self, 
        external_id: str,
        old_flow: str,
        new_flow: str
    ) -> bool:
        """Handle flow transition in HubSpot"""
        pass
```

#### **Exemplo Real de Sincronização**

**Cenário:** João quer se cadastrar no sistema

**1. João inicia no Fluxo A (Plataforma)**
```python
# Dados coletados no Fluxo A
platform_data = {
    "company_name": "ABC Tecnologia Ltda",
    "cnpj": "12345678000195",
    "email": "contato@abctec.com",
    "phone": "1133334444"
}

# Sistema cria FlowRegistry
flow_registry = FlowRegistry(
    external_id="USER_12345",
    current_flow=FlowType.PLATFORM,
    previous_flows=[],
    data_snapshot={
        "platform_data": platform_data
    },
    last_sync=datetime.now()
)
```

**2. João muda para Fluxo B (Individual)**
```python
# Dados coletados no Fluxo B
individual_data = {
    "name": "João Silva",
    "personal_email": "joao.silva@gmail.com",
    "personal_phone": "11987654321",
    "birth_date": "15/03/1985"
}

# Sistema preserva dados anteriores e adiciona novos
updated_registry = FlowRegistry(
    external_id="USER_12345",  # MESMO ID
    current_flow=FlowType.INDIVIDUAL,
    previous_flows=[FlowType.PLATFORM],  # Histórico preservado
    data_snapshot={
        # Dados do Fluxo A preservados
        "platform_data": {
            "company_name": "ABC Tecnologia Ltda",
            "cnpj": "12345678000195",
            "email": "contato@abctec.com",
            "phone": "1133334444"
        },
        # Dados do Fluxo B adicionados
        "individual_data": {
            "name": "João Silva",
            "personal_email": "joao.silva@gmail.com",
            "personal_phone": "11987654321",
            "birth_date": "15/03/1985"
        }
    },
    last_sync=datetime.now()
)
```

**3. Sincronização com HubSpot**
```python
# Sistema combina todos os dados para HubSpot
complete_user_data = {
    # Dados da empresa (do Fluxo A)
    "company_name": "ABC Tecnologia Ltda",
    "company_cnpj": "12345678000195",
    "company_email": "contato@abctec.com",
    "company_phone": "1133334444",
    
    # Dados pessoais (do Fluxo B)
    "contact_name": "João Silva",
    "contact_email": "joao.silva@gmail.com",
    "contact_phone": "11987654321",
    "birth_date": "15/03/1985",
    
    # Metadados do sistema
    "external_id": "USER_12345",
    "current_flow": "individual",
    "flow_history": ["platform", "individual"],
    "last_sync": "2025-01-15T10:30:00Z"
}

# HubSpot recebe dados completos
await hubspot_service.sync_user_data(complete_user_data)
```

**4. Benefícios Alcançados**
- ✅ **Dados preservados**: Informações da empresa não foram perdidas
- ✅ **Dados completos**: HubSpot tem visão 360° do usuário
- ✅ **Flexibilidade**: João pode voltar ao Fluxo A se necessário
- ✅ **Rastreabilidade**: Histórico completo de interações
- ✅ **Sincronização**: Todos os sistemas têm dados atualizados

**5. Fluxo de Dados**
```
João inicia Fluxo A (Plataforma)
         ↓
Sistema salva: {"company": "ABC", "cnpj": "12345678000195"}
         ↓
João muda para Fluxo B (Individual)
         ↓
Sistema preserva dados anteriores + adiciona novos
         ↓
Resultado: {"company": "ABC", "cnpj": "12345678000195", "name": "João", "phone": "11987654321"}
         ↓
HubSpot recebe dados completos
         ↓
Sistema mantém histórico para futuras consultas
```

### 3. **Preparação para CNPJ Alfanumérico**

#### **CNPJ Service com Suporte Futuro**
```python
from typing import Union
from enum import Enum

class CNPJFormat(Enum):
    NUMERIC = "numeric"      # Current: 14 digits
    ALPHANUMERIC = "alphanumeric"  # Future: 14 alphanumeric

class CNPJService:
    """Service to handle CNPJ validation and formatting"""
    
    def __init__(self):
        self.supported_formats = [CNPJFormat.NUMERIC]
        # Add CNPJFormat.ALPHANUMERIC when needed
    
    def validate_cnpj(self, cnpj: str) -> tuple[bool, CNPJFormat]:
        """Validate CNPJ and return format type"""
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', cnpj)
        
        if len(clean_cnpj) == 14:
            if clean_cnpj.isdigit():
                return self._validate_numeric(clean_cnpj), CNPJFormat.NUMERIC
            elif re.match(r'^[0-9A-Za-z]+$', clean_cnpj):
                return self._validate_alphanumeric(clean_cnpj), CNPJFormat.ALPHANUMERIC
        
        return False, None
    
    def _validate_numeric(self, cnpj: str) -> bool:
        """Validate current numeric CNPJ"""
        # Current validation logic
        pass
    
    def _validate_alphanumeric(self, cnpj: str) -> bool:
        """Validate future alphanumeric CNPJ"""
        # Future validation logic (to be implemented)
        pass
```

## 📊 Impacto e Priorização

### **Alta Prioridade (Crítico)**
1. **Validações de dados** - Impacta todos os cadastros
2. **Sincronização entre fluxos** - Perda de dados crítica
3. **Preparação CNPJ alfanumérico** - Mudança obrigatória 2025

### **Média Prioridade**
1. **Formatação consistente** - Melhora UX
2. **Logs de sincronização** - Facilita debugging
3. **Testes automatizados** - Garante qualidade

### **Baixa Prioridade**
1. **Documentação** - Facilita manutenção
2. **Métricas de performance** - Monitoramento

## 🧪 Plano de Testes

### **Testes Unitários**
```python
def test_cnpj_validation():
    """Test CNPJ validation for current and future formats"""
    validator = CNPJValidator()
    
    # Current format
    assert validator.validate_cnpj("12345678000195") == True
    assert validator.validate_cnpj("12345678000190") == False
    
    # Future format (when implemented)
    # assert validator.validate_cnpj("ABC12345678901") == True

def test_phone_validation():
    """Test phone validation and formatting"""
    contact = ContactDTO(phone="11987654321")
    assert contact.phone == "(11) 98765-4321"

def test_birth_date_validation():
    """Test birth date validation"""
    # Valid dates
    user = UserDTO(birth_date="01/01/1990")
    assert user.birth_date == date(1990, 1, 1)
    
    # Invalid dates
    with pytest.raises(ValueError):
        UserDTO(birth_date="01/01/2010")  # Under 18
```

### **Testes de Integração**
```python
def test_flow_transition():
    """Test data preservation during flow transitions"""
    # Start with platform flow
    platform_data = create_platform_user()
    
    # Transition to individual flow
    individual_data = transition_to_individual(platform_data.external_id)
    
    # Verify data preservation
    assert individual_data.external_id == platform_data.external_id
    assert individual_data.contact_info == platform_data.contact_info
```

## 🚀 Roadmap de Implementação

### **Fase 1 - Validações (2 semanas)**
- [ ] Implementar validações de data de nascimento
- [ ] Implementar validações de telefone
- [ ] Atualizar validações de CNPJ
- [ ] Criar testes unitários

### **Fase 2 - Sincronização (3 semanas)**
- [ ] Implementar FlowRegistry
- [ ] Criar FlowSynchronizationService
- [ ] Integrar com HubSpot
- [ ] Criar testes de integração

### **Fase 3 - CNPJ Alfanumérico (2 semanas)**
- [ ] Implementar CNPJService
- [ ] Atualizar todos os DTOs
- [ ] Criar migração de dados
- [ ] Testes de compatibilidade

### **Fase 4 - Monitoramento (1 semana)**
- [ ] Implementar logs de sincronização
- [ ] Criar métricas de performance
- [ ] Documentação final

## 📈 Métricas de Sucesso

### **Qualidade de Dados**
- **Taxa de dados válidos**: > 95%
- **Tempo de validação**: < 100ms
- **Erros de formatação**: < 1%

### **Sincronização**
- **Taxa de sucesso**: > 99%
- **Tempo de sincronização**: < 5s
- **Perda de dados**: 0%

### **Performance**
- **Tempo de resposta**: < 2s
- **Disponibilidade**: > 99.9%
- **Throughput**: > 100 req/s

## 🔧 Configurações Necessárias

### **Variáveis de Ambiente**
```yaml
# CNPJ Configuration
CNPJ_SUPPORT_ALPHANUMERIC: false  # Enable when needed
CNPJ_VALIDATION_STRICT: true

# HubSpot Integration
HUBSPOT_API_KEY: ${HUBSPOT_API_KEY}
HUBSPOT_SYNC_ENABLED: true
HUBSPOT_SYNC_RETRY_ATTEMPTS: 3

# Flow Synchronization
FLOW_SYNC_ENABLED: true
FLOW_SYNC_TIMEOUT: 30s
```

### **MongoDB Collections**
```javascript
// Flow tracking collection
db.createCollection("flow_registry", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["external_id", "current_flow", "created_at"],
      properties: {
        external_id: { bsonType: "string" },
        current_flow: { bsonType: "string" },
        previous_flows: { bsonType: "array" },
        data_snapshot: { bsonType: "object" },
        last_sync: { bsonType: "date" },
        created_at: { bsonType: "date" }
      }
    }
  }
});

// Add indexes for performance
db.flow_registry.createIndex({ "external_id": 1 });
db.flow_registry.createIndex({ "current_flow": 1 });
db.flow_registry.createIndex({ "last_sync": 1 });

// Update companies collection to support CNPJ format
db.companies.updateMany(
  {},
  { $set: { "cnpj_format": "numeric" } }
);
```

## 🎯 Conclusão

Este spike mapeia todos os problemas identificados no fluxo de cadastro e propõe soluções técnicas robustas. A implementação deve seguir o roadmap proposto, priorizando as validações de dados e a sincronização entre fluxos.

**Próximos passos:**
1. Aprovação do spike pela equipe técnica
2. Criação de issues detalhadas no backlog
3. Início da implementação da Fase 1
4. Revisão contínua e ajustes baseados em feedback

---

**Documento criado em**: 15 de Janeiro de 2025  
**Versão**: 1.0  
**Autor**: Willames de Jesus Campos  
**Projeto**: domain-account  
**Status**: Aguardando Aprovação

