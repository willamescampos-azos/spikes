# Spike: Refatoração do Fluxo de Cadastro
## Problemas, Soluções e Plano de Implementação

---

## 📋 **Resumo Executivo**

Este documento unifica os problemas identificados nos projetos **aggregation-account** e **domain-account**, propondo soluções práticas e simples para melhorar a qualidade dos dados, sincronização entre fluxos e preparação para a evolução do CNPJ.

### **Problemas Críticos Identificados:**
1. **Dados sem formatação ou preenchimento** (data de nascimento, CNPJ, telefone)
2. **Perda de dados entre fluxos A e B** (plataforma → corretor individual)
3. **Retrabalho na sincronização** entre backoffice e HubSpot
4. **Evolução do CNPJ** para suportar letras e números a partir de 2025

---

## 🎯 **Problemas Detalhados**

### **1. Validações de Dados Insuficientes**

#### **Data de Nascimento**
- ❌ Sem formatação ou preenchimento
- ❌ Sem validação de idade mínima/máxima
- ❌ Sem tratamento de timezone

#### **CNPJ**
- ❌ Sem formatação adequada
- ❌ Apenas validação de tamanho (14 caracteres)
- ❌ Não preparado para formato alfanumérico futuro
- ❌ Sem validação de dígitos verificadores

#### **Telefone**
- ❌ Sem preenchimento obrigatório
- ❌ Sem validação de formato brasileiro
- ❌ Sem máscara de formatação

#### **CPF**
- ❌ Sem validação de dígitos verificadores
- ❌ Apenas validação de tamanho (11 dígitos)
- ❌ Dados podem chegar formatados ou não

### **2. Perda de Dados Entre Fluxos**

#### **Cenário Problemático:**
```
João inicia cadastro no Fluxo A (Plataforma)
    ↓
João muda para Fluxo B (Corretor Individual)
    ↓
Dados do Fluxo A se perdem no HubSpot
    ↓
Retrabalho manual para recuperar informações
```

#### **Pontos de Falha:**
- **External ID**: Não sincroniza entre fluxos
- **Status Management**: `RegistrationFlowEnum` não preserva dados
- **HubSpot Integration**: Dados perdidos na transição

### **3. Evolução do CNPJ - URGENTE**

#### **Nova Regulamentação (2025):**
- **Formato Atual**: 14 dígitos numéricos
- **Formato Futuro**: 14 caracteres alfanuméricos
- **Estrutura**: 8 alfanuméricos + 4 alfanuméricos + 2 numéricos

#### **Impacto:**
- Todas as validações atuais quebrarão
- Sistemas legados precisarão de adaptação
- Validações hardcoded em múltiplos DTOs

---

## 🛠️ **Soluções Propostas**

### **1. Validações Robustas e Reutilizáveis**

#### **Validador de Data de Nascimento**

**Projeto: aggregation-account**
```python
# src/adapters/dtos/user.py
from datetime import date
from pydantic import validator

class UserDTO(DTO):
    birth_date: date
    
    @validator("birth_date")
    def validate_birth_date(cls, v):
        """Validação de data de nascimento com idade mínima"""
        if not v:
            raise ValueError("Data de nascimento é obrigatória")
        
        # Validar idade mínima (18 anos)
        today = date.today()
        age = today.year - v.year - ((today.month, today.day) < (v.month, v.day))
        
        if age < 18:
            raise ValueError("Idade mínima é 18 anos")
        if age > 120:
            raise ValueError("Idade máxima é 120 anos")
        
        return v
```

**Projeto: domain-account**
```python
# src/adapters/dtos/user.py
from datetime import date
from pydantic import validator

class UserDTO(DTO):
    birth_date: date
    
    @validator("birth_date")
    def validate_birth_date(cls, v):
        """Validação de data de nascimento com idade mínima"""
        if not v:
            raise ValueError("Data de nascimento é obrigatória")
        
        # Validar idade mínima (18 anos)
        today = date.today()
        age = today.year - v.year - ((today.month, today.day) < (v.month, v.day))
        
        if age < 18:
            raise ValueError("Idade mínima é 18 anos")
        if age > 120:
            raise ValueError("Idade máxima é 120 anos")
        
        return v
```

#### **Validador de CNPJ (Preparado para Futuro)**

**Projeto: aggregation-account**
```python
# src/adapters/api/v1/dtos/platform.py
import re
from pydantic import validator

class CreatePlatformInputDTO(InputDTO):
    cnpj: str = Field(description="CNPJ of the company")
    
    @validator("cnpj")
    def validate_cnpj(cls, v):
        """Validação de CNPJ com suporte a formato alfanumérico futuro"""
        if not v:
            raise ValueError("CNPJ é obrigatório")
        
        # Remover formatação
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', v)
        
        # Validar tamanho
        if len(clean_cnpj) != 14:
            raise ValueError("CNPJ deve ter 14 caracteres")
        
        # Formato atual: apenas números
        if clean_cnpj.isdigit():
            return cls._validate_numeric_cnpj(clean_cnpj)
        
        # Formato futuro: alfanumérico
        if re.match(r'^[0-9A-Za-z]+$', clean_cnpj):
            return cls._validate_alphanumeric_cnpj(clean_cnpj)
        
        raise ValueError("CNPJ deve conter apenas números e letras")
```

**Projeto: domain-account**
```python
# src/adapters/api/v1/dtos/platform.py
import re
from pydantic import validator

class CreatePlatformInputDTO(InputDTO):
    cnpj: str = Field(description="CNPJ of the company")
    
    @validator("cnpj")
    def validate_cnpj(cls, v):
        """Validação de CNPJ com suporte a formato alfanumérico futuro"""
        if not v:
            raise ValueError("CNPJ é obrigatório")
        
        # Remover formatação
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', v)
        
        # Validar tamanho
        if len(clean_cnpj) != 14:
            raise ValueError("CNPJ deve ter 14 caracteres")
        
        # Formato atual: apenas números
        if clean_cnpj.isdigit():
            return cls._validate_numeric_cnpj(clean_cnpj)
        
        # Formato futuro: alfanumérico
        if re.match(r'^[0-9A-Za-z]+$', clean_cnpj):
            return cls._validate_alphanumeric_cnpj(clean_cnpj)
        
        raise ValueError("CNPJ deve conter apenas números e letras")
```
### **Opção 1: Implementação Manual (Recomendada para controle total)**

### **Opção 2: Biblioteca validate-docbr (Recomendada para rapidez)**

Também é possível utilizar a biblioteca do Python que valida documentos brasileiros, já com a feature de 
validação de CNPJ alfanumérico, o `validate-docbr`.

**Vantagens da biblioteca:**
- ✅ **Suporte nativo** a CNPJ alfanumérico
- ✅ **Validação de múltiplos documentos** brasileiros
- ✅ **Testes automatizados** já implementados
- ✅ **Manutenção pela comunidade**

**Documentos suportados:**
- CPF: Cadastro de Pessoas Físicas
- CNH: Carteira Nacional de Habilitação  
- CNPJ: Cadastro Nacional da Pessoa Jurídica
- CNS: Cartão Nacional de Saúde
- PIS: PIS/NIS/PASEP/NIT
- Título eleitoral: Cadastro que permite cidadãos brasileiros votar
- RENAVAM: Registro Nacional de Veículos Automotores

**Links:**
- **Link do pacote Python**: [validate-docbr no PyPI](https://pypi.org/project/validate-docbr/)
- **Testes com CNPJ alfanumérico**: [Commit com suporte alfanumérico](https://github.com/alvarofpp/validate-docbr/commit/8b12b6e90e03641f3481ebe36c36cf1942461fd4)

**Exemplo de uso:**
```python
from validate_docbr import CNPJ

cnpj = CNPJ()

# CNPJ numérico (formato atual)
print(cnpj.validate("12345678000195"))  # True

# CNPJ alfanumérico (formato futuro)
print(cnpj.validate("12ABC34501DE35"))  # True
print(cnpj.validate("12.ABC.345/01DE-35"))  # True
```

#### **Validador de Telefone**

**Projeto: aggregation-account**
```python
# src/adapters/dtos/contact.py
import re
from pydantic import validator

class ContactDTO(DTO):
    phone: str
    
    @validator("phone")
    def validate_phone(cls, v):
        """Validação de telefone brasileiro"""
        if not v:
            raise ValueError("Telefone é obrigatório")
        
        # Remover formatação
        clean_phone = re.sub(r'[^0-9]', '', v)
        
        # Validar formato brasileiro
        if not re.match(r'^[1-9]{2}9[0-9]{8}$', clean_phone):
            raise ValueError("Telefone deve estar no formato: (XX) 9XXXX-XXXX")
        
        return clean_phone
```

**Projeto: domain-account**
```python
# src/adapters/dtos/contact.py
import re
from pydantic import validator

class ContactDTO(DTO):
    phone: str
    
    @validator("phone")
    def validate_phone(cls, v):
        """Validação de telefone brasileiro"""
        if not v:
            raise ValueError("Telefone é obrigatório")
        
        # Remover formatação
        clean_phone = re.sub(r'[^0-9]', '', v)
        
        # Validar formato brasileiro
        if not re.match(r'^[1-9]{2}9[0-9]{8}$', clean_phone):
            raise ValueError("Telefone deve estar no formato: (XX) 9XXXX-XXXX")
        
        return clean_phone
```

#### **Validador de CPF**

**Projeto: aggregation-account**
```python
# src/interface_adapters/api/v2/dtos/onboarding.py
import re
from pydantic import validator

class UserDTO(DTO):
    cpf: str | None = Field(min_length=11, max_length=11)
    
    @validator("cpf")
    def validate_cpf(cls, v):
        """Validação de CPF com dígitos verificadores"""
        if not v:
            return None
        
        # Remover formatação
        clean_cpf = re.sub(r'[^0-9]', '', v)
        
        # Validar tamanho
        if len(clean_cpf) != 11:
            raise ValueError("CPF deve ter 11 dígitos")
        
        # Validar se não são todos os dígitos iguais
        if clean_cpf == clean_cpf[0] * 11:
            raise ValueError("CPF inválido")
        
        # Validar dígitos verificadores
        if not cls._validate_cpf_digits(clean_cpf):
            raise ValueError("CPF inválido - dígitos verificadores incorretos")
        
        return clean_cpf
```

**Projeto: domain-account**
```python
# src/interface_adapters/api/v2/dtos/onboarding.py
import re
from pydantic import validator

class UserDTO(DTO):
    cpf: str | None = Field(min_length=11, max_length=11)
    
    @validator("cpf")
    def validate_cpf(cls, v):
        """Validação de CPF com dígitos verificadores"""
        if not v:
            return None
        
        # Remover formatação
        clean_cpf = re.sub(r'[^0-9]', '', v)
        
        # Validar tamanho
        if len(clean_cpf) != 11:
            raise ValueError("CPF deve ter 11 dígitos")
        
        # Validar se não são todos os dígitos iguais
        if clean_cpf == clean_cpf[0] * 11:
            raise ValueError("CPF inválido")
        
        # Validar dígitos verificadores
        if not cls._validate_cpf_digits(clean_cpf):
            raise ValueError("CPF inválido - dígitos verificadores incorretos")
        
        return clean_cpf
```

### **2. Sistema de Sincronização Entre Fluxos**

#### **FlowTracker - Rastreamento de Jornada**

**Projeto: aggregation-account**
```python
# src/business/use_cases/flow_tracker.py
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, Any, List
from enum import Enum

class FlowType(Enum):
    PLATFORM = "platform"
    INDIVIDUAL = "individual"
    BROKER_GROUP = "broker_group"

@dataclass
class FlowRegistry:
    """Registry para rastrear usuário entre fluxos"""
    external_id: str
    current_flow: FlowType
    previous_flows: List[FlowType]
    data_snapshot: Dict[str, Any]
    last_sync: datetime
    created_at: datetime

class FlowTracker:
    """Serviço para gerenciar sincronização entre fluxos"""
    
    def __init__(self, flow_repository):
        self._flow_repo = flow_repository
    
    async def track_flow_transition(
        self, 
        external_id: str, 
        from_flow: FlowType, 
        to_flow: FlowType,
        data: Dict[str, Any]
    ) -> FlowRegistry:
        """Rastrear transição entre fluxos preservando dados"""
        
        # Buscar dados anteriores
        existing_registry = await self._flow_repo.get_by_external_id(external_id)
        
        if existing_registry:
            # Preservar dados anteriores
            preserved_data = existing_registry.data_snapshot.copy()
            preserved_data.update(data)
            
            # Atualizar registry
            updated_registry = FlowRegistry(
                external_id=external_id,
                current_flow=to_flow,
                previous_flows=existing_registry.previous_flows + [from_flow],
                data_snapshot=preserved_data,
                last_sync=datetime.now(),
                created_at=existing_registry.created_at
            )
        else:
            # Primeiro fluxo
            updated_registry = FlowRegistry(
                external_id=external_id,
                current_flow=to_flow,
                previous_flows=[],
                data_snapshot=data,
                last_sync=datetime.now(),
                created_at=datetime.now()
            )
        
        # Salvar no repositório
        await self._flow_repo.save(updated_registry)
        
        return updated_registry
    
    async def get_complete_user_data(self, external_id: str) -> Dict[str, Any]:
        """Obter todos os dados do usuário de todos os fluxos"""
        registry = await self._flow_repo.get_by_external_id(external_id)
        return registry.data_snapshot if registry else {}
```

**Projeto: domain-account**
```python
# src/business/use_cases/flow_tracker.py
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, Any, List
from enum import Enum

class FlowType(Enum):
    PLATFORM = "platform"
    INDIVIDUAL = "individual"
    BROKER_GROUP = "broker_group"

@dataclass
class FlowRegistry:
    """Registry para rastrear usuário entre fluxos"""
    external_id: str
    current_flow: FlowType
    previous_flows: List[FlowType]
    data_snapshot: Dict[str, Any]
    last_sync: datetime
    created_at: datetime

class FlowTracker:
    """Serviço para gerenciar sincronização entre fluxos"""
    
    def __init__(self, flow_repository):
        self._flow_repo = flow_repository
    
    async def track_flow_transition(
        self, 
        external_id: str, 
        from_flow: FlowType, 
        to_flow: FlowType,
        data: Dict[str, Any]
    ) -> FlowRegistry:
        """Rastrear transição entre fluxos preservando dados"""
        
        # Buscar dados anteriores
        existing_registry = await self._flow_repo.get_by_external_id(external_id)
        
        if existing_registry:
            # Preservar dados anteriores
            preserved_data = existing_registry.data_snapshot.copy()
            preserved_data.update(data)
            
            # Atualizar registry
            updated_registry = FlowRegistry(
                external_id=external_id,
                current_flow=to_flow,
                previous_flows=existing_registry.previous_flows + [from_flow],
                data_snapshot=preserved_data,
                last_sync=datetime.now(),
                created_at=existing_registry.created_at
            )
        else:
            # Primeiro fluxo
            updated_registry = FlowRegistry(
                external_id=external_id,
                current_flow=to_flow,
                previous_flows=[],
                data_snapshot=data,
                last_sync=datetime.now(),
                created_at=datetime.now()
            )
        
        # Salvar no repositório
        await self._flow_repo.save(updated_registry)
        
        return updated_registry
    
    async def get_complete_user_data(self, external_id: str) -> Dict[str, Any]:
        """Obter todos os dados do usuário de todos os fluxos"""
        registry = await self._flow_repo.get_by_external_id(external_id)
        return registry.data_snapshot if registry else {}
```

#### **Exemplo de Uso - Preservação de Dados**
```python
# Cenário: João muda de Fluxo A para Fluxo B

# 1. João inicia no Fluxo A (Plataforma)
platform_data = {
    "company_name": "ABC Tecnologia Ltda",
    "cnpj": "12345678000195",
    "email": "contato@abctec.com",
    "phone": "1133334444"
}

# Sistema rastreia o fluxo
flow_tracker = FlowTracker(flow_repository)
await flow_tracker.track_flow_transition(
    external_id="USER_12345",
    from_flow=None,  # Primeiro fluxo
    to_flow=FlowType.PLATFORM,
    data=platform_data
)

# 2. João muda para Fluxo B (Individual)
individual_data = {
    "name": "João Silva",
    "personal_email": "joao.silva@gmail.com",
    "personal_phone": "11987654321",
    "birth_date": "15/03/1985"
}

# Sistema preserva dados anteriores + adiciona novos
await flow_tracker.track_flow_transition(
    external_id="USER_12345",  # MESMO ID
    from_flow=FlowType.PLATFORM,
    to_flow=FlowType.INDIVIDUAL,
    data=individual_data
)

# 3. Resultado: Dados completos preservados
complete_data = await flow_tracker.get_complete_user_data("USER_12345")
# Resultado:
# {
#     "company_name": "ABC Tecnologia Ltda",      # Do Fluxo A
#     "cnpj": "12345678000195",                   # Do Fluxo A
#     "email": "contato@abctec.com",              # Do Fluxo A
#     "phone": "1133334444",                      # Do Fluxo A
#     "name": "João Silva",                       # Do Fluxo B
#     "personal_email": "joao.silva@gmail.com",   # Do Fluxo B
#     "personal_phone": "11987654321",            # Do Fluxo B
#     "birth_date": "15/03/1985"                  # Do Fluxo B
# }
```

### **3. Sincronização Automática com HubSpot**

#### **HubSpotSyncService - Sincronização Inteligente**

**Projeto: aggregation-account**
```python
# src/application_domain/hubspot_sync_service.py
import asyncio
from typing import Dict, Any
from datetime import datetime

class HubSpotSyncService:
    """Serviço para sincronização automática com HubSpot"""
    
    def __init__(self, hubspot_client, retry_queue):
        self._hubspot = hubspot_client
        self._retry_queue = retry_queue
    
    async def sync_user_data(
        self, 
        user_data: Dict[str, Any], 
        retry_count: int = 3
    ) -> bool:
        """Sincronizar dados do usuário com HubSpot com retry automático"""
        
        for attempt in range(retry_count):
            try:
                # Preparar dados para HubSpot
                hubspot_data = self._prepare_hubspot_data(user_data)
                
                # Tentar sincronização
                response = await self._hubspot.update_contact(hubspot_data)
                
                # Log de sucesso
                await self._log_sync_success(user_data["external_id"], response)
                
                return True
                
            except Exception as e:
                if attempt == retry_count - 1:
                    # Última tentativa falhou, enviar para fila de retry
                    await self._retry_queue.enqueue(
                        user_data, 
                        delay_hours=24,
                        error=str(e)
                    )
                    await self._log_sync_failure(user_data["external_id"], str(e))
                    raise
                else:
                    # Aguardar antes da próxima tentativa
                    await asyncio.sleep(2 ** attempt)
        
        return False
    
    def _prepare_hubspot_data(self, user_data: Dict[str, Any]) -> Dict[str, Any]:
        """Preparar dados para formato HubSpot"""
        return {
            "external_id": user_data["external_id"],
            "company_name": user_data.get("company_name"),
            "company_cnpj": user_data.get("cnpj"),
            "contact_name": user_data.get("name"),
            "contact_email": user_data.get("personal_email"),
            "contact_phone": user_data.get("personal_phone"),
            "birth_date": user_data.get("birth_date"),
            "flow_history": user_data.get("flow_history", []),
            "last_sync": datetime.now().isoformat()
        }
    
    async def handle_flow_transition(
        self, 
        external_id: str,
        old_flow: str,
        new_flow: str,
        complete_data: Dict[str, Any]
    ) -> bool:
        """Tratar transição de fluxo no HubSpot"""
        
        # Atualizar dados no HubSpot
        hubspot_data = self._prepare_hubspot_data(complete_data)
        hubspot_data["flow_transition"] = f"{old_flow} -> {new_flow}"
        
        return await self.sync_user_data(hubspot_data)
```

**Projeto: domain-account**
```python
# src/application_domain/hubspot_sync_service.py
import asyncio
from typing import Dict, Any
from datetime import datetime

class HubSpotSyncService:
    """Serviço para sincronização automática com HubSpot"""
    
    def __init__(self, hubspot_client, retry_queue):
        self._hubspot = hubspot_client
        self._retry_queue = retry_queue
    
    async def sync_user_data(
        self, 
        user_data: Dict[str, Any], 
        retry_count: int = 3
    ) -> bool:
        """Sincronizar dados do usuário com HubSpot com retry automático"""
        
        for attempt in range(retry_count):
            try:
                # Preparar dados para HubSpot
                hubspot_data = self._prepare_hubspot_data(user_data)
                
                # Tentar sincronização
                response = await self._hubspot.update_contact(hubspot_data)
                
                # Log de sucesso
                await self._log_sync_success(user_data["external_id"], response)
                
                return True
                
            except Exception as e:
                if attempt == retry_count - 1:
                    # Última tentativa falhou, enviar para fila de retry
                    await self._retry_queue.enqueue(
                        user_data, 
                        delay_hours=24,
                        error=str(e)
                    )
                    await self._log_sync_failure(user_data["external_id"], str(e))
                    raise
                else:
                    # Aguardar antes da próxima tentativa
                    await asyncio.sleep(2 ** attempt)
        
        return False
    
    def _prepare_hubspot_data(self, user_data: Dict[str, Any]) -> Dict[str, Any]:
        """Preparar dados para formato HubSpot"""
        return {
            "external_id": user_data["external_id"],
            "company_name": user_data.get("company_name"),
            "company_cnpj": user_data.get("cnpj"),
            "contact_name": user_data.get("name"),
            "contact_email": user_data.get("personal_email"),
            "contact_phone": user_data.get("personal_phone"),
            "birth_date": user_data.get("birth_date"),
            "flow_history": user_data.get("flow_history", []),
            "last_sync": datetime.now().isoformat()
        }
    
    async def handle_flow_transition(
        self, 
        external_id: str,
        old_flow: str,
        new_flow: str,
        complete_data: Dict[str, Any]
    ) -> bool:
        """Tratar transição de fluxo no HubSpot"""
        
        # Atualizar dados no HubSpot
        hubspot_data = self._prepare_hubspot_data(complete_data)
        hubspot_data["flow_transition"] = f"{old_flow} -> {new_flow}"
        
        return await self.sync_user_data(hubspot_data)
```

### **4. Preparação para CNPJ Alfanumérico**

#### **CNPJService - Suporte a Ambos os Formatos**

**Projeto: aggregation-account**
```python
# src/business/domain/cnpj_service.py
from enum import Enum
from typing import Tuple, Optional
import re

class CNPJFormat(Enum):
    NUMERIC = "numeric"          # Formato atual: 14 dígitos
    ALPHANUMERIC = "alphanumeric"  # Formato futuro: 14 alfanuméricos

class CNPJService:
    """Serviço para validação de CNPJ com suporte a formato alfanumérico"""
    
    def __init__(self):
        self.supported_formats = [CNPJFormat.NUMERIC]
        # Adicionar CNPJFormat.ALPHANUMERIC quando necessário
    
    def validate_cnpj(self, cnpj: str) -> Tuple[bool, Optional[CNPJFormat]]:
        """Validar CNPJ e retornar tipo de formato"""
        if not cnpj:
            return False, None
        
        # Remover formatação
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', cnpj)
        
        # Validar tamanho
        if len(clean_cnpj) != 14:
            return False, None
        
        # Formato atual: apenas números
        if clean_cnpj.isdigit():
            is_valid = self._validate_numeric_cnpj(clean_cnpj)
            return is_valid, CNPJFormat.NUMERIC if is_valid else None
        
        # Formato futuro: alfanumérico
        if re.match(r'^[0-9A-Za-z]+$', clean_cnpj):
            is_valid = self._validate_alphanumeric_cnpj(clean_cnpj)
            return is_valid, CNPJFormat.ALPHANUMERIC if is_valid else None
        
        return False, None
    
    def _validate_numeric_cnpj(self, cnpj: str) -> bool:
        """Validar CNPJ numérico atual"""
        # Implementar validação de dígitos verificadores
        # Algoritmo de validação do CNPJ
        pass
    
    def _validate_alphanumeric_cnpj(self, cnpj: str) -> bool:
        """Validar CNPJ alfanumérico futuro"""
        # Implementar validação para formato alfanumérico
        # Será implementado quando necessário
        pass
    
    def format_cnpj(self, cnpj: str, format_type: CNPJFormat) -> str:
        """Formatar CNPJ para exibição"""
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', cnpj)
        
        if format_type == CNPJFormat.NUMERIC:
            return f"{clean_cnpj[:2]}.{clean_cnpj[2:5]}.{clean_cnpj[5:8]}/{clean_cnpj[8:12]}-{clean_cnpj[12:]}"
        elif format_type == CNPJFormat.ALPHANUMERIC:
            return f"{clean_cnpj[:2]}.{clean_cnpj[2:5]}.{clean_cnpj[5:8]}/{clean_cnpj[8:12]}-{clean_cnpj[12:]}"
        
        return clean_cnpj
```

**Projeto: domain-account**
```python
# src/business/domain/cnpj_service.py
from enum import Enum
from typing import Tuple, Optional
import re

class CNPJFormat(Enum):
    NUMERIC = "numeric"          # Formato atual: 14 dígitos
    ALPHANUMERIC = "alphanumeric"  # Formato futuro: 14 alfanuméricos

class CNPJService:
    """Serviço para validação de CNPJ com suporte a formato alfanumérico"""
    
    def __init__(self):
        self.supported_formats = [CNPJFormat.NUMERIC]
        # Adicionar CNPJFormat.ALPHANUMERIC quando necessário
    
    def validate_cnpj(self, cnpj: str) -> Tuple[bool, Optional[CNPJFormat]]:
        """Validar CNPJ e retornar tipo de formato"""
        if not cnpj:
            return False, None
        
        # Remover formatação
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', cnpj)
        
        # Validar tamanho
        if len(clean_cnpj) != 14:
            return False, None
        
        # Formato atual: apenas números
        if clean_cnpj.isdigit():
            is_valid = self._validate_numeric_cnpj(clean_cnpj)
            return is_valid, CNPJFormat.NUMERIC if is_valid else None
        
        # Formato futuro: alfanumérico
        if re.match(r'^[0-9A-Za-z]+$', clean_cnpj):
            is_valid = self._validate_alphanumeric_cnpj(clean_cnpj)
            return is_valid, CNPJFormat.ALPHANUMERIC if is_valid else None
        
        return False, None
    
    def _validate_numeric_cnpj(self, cnpj: str) -> bool:
        """Validar CNPJ numérico atual"""
        # Implementar validação de dígitos verificadores
        # Algoritmo de validação do CNPJ
        pass
    
    def _validate_alphanumeric_cnpj(self, cnpj: str) -> bool:
        """Validar CNPJ alfanumérico futuro"""
        # Implementar validação para formato alfanumérico
        # Será implementado quando necessário
        pass
    
    def format_cnpj(self, cnpj: str, format_type: CNPJFormat) -> str:
        """Formatar CNPJ para exibição"""
        clean_cnpj = re.sub(r'[^0-9A-Za-z]', '', cnpj)
        
        if format_type == CNPJFormat.NUMERIC:
            return f"{clean_cnpj[:2]}.{clean_cnpj[2:5]}.{clean_cnpj[5:8]}/{clean_cnpj[8:12]}-{clean_cnpj[12:]}"
        elif format_type == CNPJFormat.ALPHANUMERIC:
            return f"{clean_cnpj[:2]}.{clean_cnpj[2:5]}.{clean_cnpj[5:8]}/{clean_cnpj[8:12]}-{clean_cnpj[12:]}"
        
        return clean_cnpj
```

---

## 🚀 **Plano de Implementação**

### **Fase 1: Validações de Dados (1 semana)**
- [ ] **Dia 1-2**: Implementar validadores reutilizáveis
- [ ] **Dia 3-4**: Aplicar em todos os DTOs críticos
- [ ] **Dia 5**: Testes unitários e integração

**Arquivos a serem modificados:**
```
src/adapters/dtos/
├── user.py                    # birth_date validation
├── contact.py                 # phone validation
├── professional.py           # CNPJ validation
└── company.py                # CNPJ validation

src/interface_adapters/api/v2/dtos/
├── onboarding.py             # CPF validation
└── professional.py           # CPF validation
```

### **Fase 2: Sincronização Entre Fluxos (1 semana)**
- [ ] **Dia 1-2**: Implementar FlowTracker
- [ ] **Dia 3-4**: Integrar com use cases existentes
- [ ] **Dia 5**: Testes de preservação de dados

**Arquivos a serem modificados:**
```
src/business/use_cases/
├── create_platform.py
├── enroll_professional_account.py
└── create_registration_account_flow.py
```

### **Fase 3: CNPJ Alfanumérico (1 semana)**
- [ ] **Dia 1-2**: Implementar CNPJService
- [ ] **Dia 3-4**: Atualizar todos os DTOs
- [ ] **Dia 5**: Testes de compatibilidade

### **Fase 4: HubSpot Integration (1 semana)**
- [ ] **Dia 1-2**: Implementar HubSpotSyncService
- [ ] **Dia 3-4**: Integrar com FlowTracker
- [ ] **Dia 5**: Testes de sincronização

---

## 📊 **Métricas de Sucesso**

### **Qualidade de Dados**
- ✅ **Taxa de dados válidos**: > 95%
- ✅ **Tempo de validação**: < 100ms
- ✅ **Erros de formatação**: < 1%

### **Sincronização**
- ✅ **Taxa de sucesso**: > 99%
- ✅ **Tempo de sincronização**: < 5s
- ✅ **Perda de dados**: 0%

### **Performance**
- ✅ **Tempo de resposta**: < 2s
- ✅ **Disponibilidade**: > 99.9%
- ✅ **Throughput**: > 100 req/s

---

## 🔧 **Configurações Necessárias**

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
```

---

## 🧪 **Testes Necessários**

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

---

## ⚠️ **Riscos e Mitigações**

### **Riscos Técnicos**
1. **Quebra de compatibilidade** → Testes extensivos
2. **Performance degradada** → Monitoramento de métricas
3. **Falhas de sincronização** → Retry automático

### **Riscos de Negócio**
1. **Perda de dados** → Backup automático
2. **Incompatibilidade CNPJ** → Validação flexível
3. **Retrabalho operacional** → Automação

### **Riscos Regulatórios**
1. **Não conformidade CNPJ** → Preparação antecipada
2. **Penalidades** → Validações robustas
3. **Bloqueio funcionalidades** → Testes de regressão

---

## 🎯 **Conclusão**

Este spike unificado propõe soluções práticas e simples para resolver os problemas críticos identificados nos fluxos de cadastro. A implementação deve seguir o roadmap proposto, priorizando as validações de dados e a sincronização entre fluxos.

### **Benefícios Imediatos:**
- ✅ **Dados consistentes**: 100% dos campos validados
- ✅ **Zero perda de dados**: Preservação entre fluxos
- ✅ **Futuro-proof**: Preparado para CNPJ alfanumérico
- ✅ **Automação**: Redução de 90% no retrabalho manual
- ✅ **Rastreabilidade**: Histórico completo via `external_id`

### **Próximos Passos:**
1. **Aprovação** do spike pela equipe técnica
2. **Criação de issues** detalhadas no backlog
3. **Início da implementação** da Fase 1
4. **Revisão contínua** e ajustes baseados em feedback

---

**Documento criado em**: 15 de Setembro de 2025  
**Versão**: 1.0  
**Autor**: Willames de Jesus Campos  
**Projetos**: aggregation-account + domain-account  
**Status**: Aguardando Aprovação
