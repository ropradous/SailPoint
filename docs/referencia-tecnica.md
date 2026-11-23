# Referência Técnica: IGA Confluent Cloud + SailPoint + Entra ID

**Versão:** 1.0  
**Data:** 23 de Novembro de 2026  
**Classificação:** Arquitetura de Segurança

---

## 📋 Índice

1. [Links de Referência](#links-de-referência)
2. [Especificações Técnicas](#especificações-técnicas)
3. [Checklist de Implementação](#checklist-de-implementação)
4. [Troubleshooting](#troubleshooting)
5. [Contatos e Escalonamento](#contatos-e-escalonamento)

---

## 📚 Links de Referência

### Confluent Cloud API

| Recurso | URL | Descrição |
|---------|-----|-----------|
| IAM Docs | https://docs.confluent.io/cloud/current/access-management/identity-and-access-management.html | Documentação completa de IAM |
| API v3 Reference | https://docs.confluent.io/cloud/current/api.html | Referência de endpoints /iam/v2 |
| RBAC Roles | https://docs.confluent.io/cloud/current/access-management/rbac/rbac-roles.html | Definição de roles e permissões |
| Service Accounts | https://docs.confluent.io/cloud/current/access-management/identity/service-accounts.html | Guia de Service Accounts |
| Group Mapping | https://docs.confluent.io/cloud/current/access-management/identity/group-mapping.html | Configuração de Group Mapping |
| SAML SSO | https://docs.confluent.io/cloud/current/access-management/identity/saml.html | Configuração SAML 2.0 |
| Audit Log | https://docs.confluent.io/cloud/current/audit-logging/ | Configuração de auditoria |
| CRN Format | https://docs.confluent.io/cloud/current/api/crn-format.html | Formato de CRN patterns |

### SailPoint IdentityNow

| Recurso | URL | Descrição |
|---------|-----|-----------|
| API v3 Documentation | https://developer.sailpoint.com/docs/api/v3/ | Referência completa da API |
| Connectors | https://developer.sailpoint.com/docs/connectors/ | Documentação de conectores |
| Web Services Connector | https://developer.sailpoint.com/docs/api/beta/web-services-connector-spec | Especificação do WSC |
| Account Aggregation | https://developer.sailpoint.com/docs/connectors/account-aggregation/ | Padrão de agregação |
| Entitlements | https://developer.sailpoint.com/docs/connectors/entitlements/ | Mapeamento de direitos |
| Provisioning | https://developer.sailpoint.com/docs/connectors/provisioning/ | Operações de provisionamento |
| Microsoft Connector | https://developer.sailpoint.com/docs/connectors/ms-entra-id/ | Conector Entra ID nativo |

### Microsoft Entra ID

| Recurso | URL | Descrição |
|---------|-----|-----------|
| Entra ID Docs | https://learn.microsoft.com/en-us/entra/fundamentals/whatis | Documentação principal |
| SAML Configuration | https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-saml | Protocolo SAML 2.0 |
| Group Management | https://learn.microsoft.com/en-us/entra/fundamentals/how-to-manage-groups | Gerenciamento de grupos |
| Conditional Access | https://learn.microsoft.com/en-us/entra/identity/conditional-access/ | Políticas de acesso condicional |
| Security Best Practices | https://learn.microsoft.com/en-us/entra/identity/secure-identity-essentials | Essenciais de segurança |

---

## 🔧 Especificações Técnicas

### Web Services Connector - Configuração JSON

```json
{
  "name": "Confluent Cloud IAM Connector",
  "baseUrl": "https://api.confluent.cloud",
  "authenticationType": "BASIC",
  "username": "<CLOUD_API_KEY>",
  "password": "<CLOUD_API_SECRET>",
  "headers": [
    {
      "name": "Content-Type",
      "value": "application/json"
    },
    {
      "name": "Accept",
      "value": "application/json"
    }
  ],
  "pageSize": 100,
  "customHeaders": true
}
```

### Mapeamento de Endpoints

#### Account Aggregation

**Endpoint:** `GET /iam/v2/service-accounts?page_size=100`

**Root JSONPath:** `$.data[*]`

**Field Mappings:**
```json
{
  "id": "account_id",
  "display_name": "account_name",
  "description": "description",
  "metadata": {
    "created_at": "created",
    "updated_at": "modified"
  }
}
```

#### Entitlement Aggregation

**Endpoint:** `GET /iam/v2/role-bindings?page_size=100`

**Root JSONPath:** `$.data[*]`

**Field Mappings:**
```json
{
  "id": "entitlement_id",
  "principal": "account_id",  // Requer transformação: remover "User:" prefix
  "role_name": "entitlement_type",
  "crn_pattern": "entitlement_name"
}
```

**Regra de Transformação:**
```
attribute:principal -> replace("User:", "")
```

### Operações de Provisionamento

#### CREATE Account

```json
{
  "endpoint": "POST /iam/v2/service-accounts",
  "requestBody": {
    "display_name": "$plan.display_name$",
    "description": "AppID: $plan.app_id$ | Owner: $plan.owner_email$ | Managed by SailPoint"
  },
  "responseMapping": {
    "id": "$.id"
  }
}
```

#### ADD Entitlement

```json
{
  "endpoint": "POST /iam/v2/role-bindings",
  "requestBody": {
    "principal": "User:$plan.nativeIdentity$",
    "role_name": "$plan.role_name$",
    "crn_pattern": "$plan.crn_pattern$"
  },
  "responseMapping": {
    "entitlementId": "$.id"
  }
}
```

#### REMOVE Entitlement

```json
{
  "endpoint": "DELETE /iam/v2/role-bindings/$plan.entitlementId$",
  "responseMapping": {}
}
```

#### DELETE Account

```json
{
  "endpoint": "DELETE /iam/v2/service-accounts/$plan.nativeIdentity$",
  "responseMapping": {}
}
```

### Microsoft Entra ID - Padrão de Nomenclatura

#### Grupos de Segurança

```
Padrão: SG-CONFLUENT-<AMBIENTE>-<DOMINIO>-<ROLE>

Exemplos:
- SG-CONFLUENT-PROD-PAGAMENTOS-READ
- SG-CONFLUENT-PROD-PAGAMENTOS-WRITE
- SG-CONFLUENT-DEV-DATA-ENGINEERS
- SG-CONFLUENT-UAT-COMPLIANCE-ADMIN
```

#### Object IDs

```
Mapeamento no Group Mapping do Confluent:
Group Name: SG-CONFLUENT-PROD-PAGAMENTOS-READ
Object ID: a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6

Este Object ID é incluído na asserção SAML como claim 'groups'
```

### Service Accounts - Padrão de Nomenclatura

```
Padrão: sa-<sigla-app>-<ambiente>

Exemplos:
- sa-pix-prod
- sa-core-cartoes-dev
- sa-fraud-detect-prod
- sa-analytics-uat

Descrição (Obrigatória):
AppID: APP-<número> | Owner: <squad@empresa.com> | Managed by SailPoint
```

### CRN Patterns - Referência Rápida

```
# Nível de Organização (admin total)
crn://confluent.cloud/organization=<org-id>/*

# Nível de Ambiente
crn://confluent.cloud/organization=<org-id>/environment=<env-id>

# Nível de Cluster
crn://confluent.cloud/organization=<org-id>/environment=<env-id>/cloud-cluster=<cluster-id>

# Nível de Tópico (com prefixo)
crn://confluent.cloud/organization=<org-id>/environment=<env-id>/cloud-cluster=<cluster-id>/kafka=<cluster-id>/topic=<prefix>*

# Nível de Consumer Group
crn://confluent.cloud/organization=<org-id>/environment=<env-id>/cloud-cluster=<cluster-id>/kafka=<cluster-id>/group=<group-pattern>*

# Nível de Conector (Source ou Sink)
crn://confluent.cloud/organization=<org-id>/environment=<env-id>/connectors=<connector-id>
```

### RBAC Roles - Mapeamento Recomendado

| Uso | Role | Nível | Permissões |
|-----|------|-------|-----------|
| Administração | OrganizationAdmin | Organization | Acesso total |
| Gerenciamento IAM | Assigner | Organization | Criar/editar role bindings |
| Leitura de Dados | DeveloperRead | Tópico/CG | Consumir, descrever |
| Escrita de Dados | DeveloperWrite | Tópico | Produzir, descrever |
| Operação | Operator | Cluster/Env | Gerenciar cluster, não dados |
| Auditoria | AuditAdmin | Organization | Acesso a logs apenas |

---

## ✅ Checklist de Implementação

### Pré-Requisitos

- [ ] Acesso administrativo ao Confluent Cloud
- [ ] Acesso administrativo ao SailPoint IdentityNow
- [ ] Acesso administrativo ao Microsoft Entra ID
- [ ] IP da rede SailPoint (para IP Filtering, se aplicável)
- [ ] Lista de equipes/aplicações que necessitam acesso
- [ ] Aprovação de CIO/Compliance para padrão de nomenclatura

### Fase 1: Preparação (Semana 1-2)

- [ ] Criar Service Account `sa-sailpoint-iga` no Confluent Cloud
- [ ] Atribuir role `OrganizationAdmin` (ou `Assigner`)
- [ ] Gerar Cloud API Key e Secret
- [ ] Armazenar credenciais no cofre seguro (CyberArk/Vault)
- [ ] Documentar Object ID da organização Confluent
- [ ] Testar acesso via `curl` ou Postman
  ```bash
  curl -u <api-key>:<api-secret> https://api.confluent.cloud/iam/v2/service-accounts
  ```

### Fase 2: Integração Entra ID (Semana 2-3)

- [ ] Configurar Entra ID Connector no SailPoint
- [ ] Criar grupos de segurança no Entra ID (SG-CONFLUENT-*)
- [ ] Documentar Object IDs dos grupos Entra ID
- [ ] Configurar Group Mapping no Confluent Cloud
- [ ] Testar mapeamento: adicionar usuário a grupo Entra ID → verificar RBAC Confluent
- [ ] Validar SAML assertions com ferramenta SAML debugger

### Fase 3: Web Services Connector (Semana 3-4)

- [ ] Criar novo connector no SailPoint (tipo "Web Services Connector")
- [ ] Configurar Base URL: `https://api.confluent.cloud`
- [ ] Configurar autenticação Basic Auth (api-key:api-secret)
- [ ] Testar Account Aggregation: `GET /iam/v2/service-accounts`
- [ ] Testar Entitlement Aggregation: `GET /iam/v2/role-bindings`
- [ ] Validar JSONPath mappings
- [ ] Testar transformação de `principal` (remover "User:" prefix)

### Fase 4: Operações de Provisionamento (Semana 4-5)

- [ ] Configurar operação CREATE Account (POST /iam/v2/service-accounts)
- [ ] Configurar operação ADD Entitlement (POST /iam/v2/role-bindings)
- [ ] Configurar operação REMOVE Entitlement (DELETE /iam/v2/role-bindings/{id})
- [ ] Configurar operação DELETE Account (DELETE /iam/v2/service-accounts/{id})
- [ ] Testar cada operação em sequência completa
- [ ] Validar metadados na descrição do Service Account

### Fase 5: Catálogo e Aprovações (Semana 5-6)

- [ ] Criar Access Profiles no catálogo SailPoint
- [ ] Configurar fluxos de aprovação com gestores
- [ ] Configurar SoDs (Segregation of Duties) se necessário
- [ ] Treinar proprietários de aplicações
- [ ] Executar teste piloto com 2-3 aplicações

### Fase 6: Governança Contínua (Semana 6-7)

- [ ] Configurar Confluent Audit Log
- [ ] Integrar com SIEM (Splunk, Datadog, etc.)
- [ ] Habilitar Access Certifications (trimestral)
- [ ] Configurar alertas para anomalias
- [ ] Documentar runbooks operacionais

### Validação Final

- [ ] [ ] Testar JML completo: Joiner → Mover → Leaver
- [ ] [ ] Validar auditoria de todas as operações
- [ ] [ ] Teste de carga: criar 100+ Service Accounts via bulk operations
- [ ] [ ] Teste de failover: validar comportamento em caso de indisponibilidade de APIs
- [ ] [ ] Teste de rollback: validar restauração de estado anterior
- [ ] [ ] Teste de conformidade: verificar compliance com políticas corporativas

---

## 🔧 Troubleshooting

### Erro 401 - Autenticação Falhou

**Sintoma:** 
```
HTTP 401: Unauthorized
```

**Causas:**
- Cloud API Key ou Secret incorretos
- Credenciais expiradas
- Permissões insuficientes (não OrganizationAdmin)

**Solução:**
1. Verificar se Cloud API Key ainda é válida
2. Regenerar Cloud API Secret se necessário
3. Confirmar role OrganizationAdmin atribuída
4. Testar com `curl`:
   ```bash
   curl -u <key>:<secret> -v https://api.confluent.cloud/iam/v2/service-accounts
   ```

### Erro 403 - Permissão Negada

**Sintoma:**
```
HTTP 403: Forbidden
```

**Causas:**
- Service Account sem role apropriada
- IP Filtering bloqueando a requisição

**Solução:**
1. Confirmar que `sa-sailpoint-iga` tem role `OrganizationAdmin`
2. Se usar IP Filtering, adicionar IP de saída SailPoint na whitelist
3. Testar conexão sem IP Filtering temporariamente

### Erro 404 - Recurso Não Encontrado

**Sintoma:**
```
HTTP 404: Not Found
```

**Causas:**
- CRN pattern inválido
- Role Binding ID inválido para DELETE
- Service Account ID incorreto

**Solução:**
1. Validar CRN format usando documentação oficial
2. Confirmar que role_binding_id existe via GET /iam/v2/role-bindings
3. Usar UUID correto do Service Account

### Account Aggregation Retorna Vazio

**Sintoma:**
```
{
  "data": []
}
```

**Causas:**
- Nenhum Service Account criado ainda
- JSONPath mappings incorretos

**Solução:**
1. Verificar se existem Service Accounts: `GET /iam/v2/service-accounts`
2. Validar JSONPath root: deve ser `$.data[*]`
3. Testar manuamente em Postman com exatamente a mesma autenticação

### Group Mapping Não Sincroniza

**Sintoma:**
- Usuário adicionado a grupo Entra ID, mas acesso não é concedido no Confluent

**Causas:**
- Object ID incorreto no mapeamento
- Grupo Entra ID não existe
- SAML assertion não inclui claims groups

**Solução:**
1. Verificar Object ID do grupo: `Get-AzureADGroup -SearchString "SG-CONFLUENT-*" | Select ObjectId`
2. Validar SAML assertion usando SAML tracer ou debugger
3. Confirmar que Confluent Cloud está recebendo claim 'groups' corretamente
4. Teste: fazer logout e login novamente para forçar revalidação

### Web Services Connector Falha em Transformação

**Sintoma:**
```
Cannot find attribute: principal
```

**Causas:**
- Regra de transformação não está aplicada
- Formato do campo principal é diferente do esperado

**Solução:**
1. No SailPoint, adicionar regra de transformação explicitamente:
   ```
   sourceAttribute=principal
   targetAttribute=account_id
   transformation: return value.replace("User:", "")
   ```
2. Validar que resposta da API contém exatamente `"principal": "User:sa-xxxxx"`

---

## 📞 Contatos e Escalonamento

### Equipes Internas

| Área | Contato | Telefone | Email |
|------|---------|----------|-------|
| Arquitetura de Segurança | João Silva | +55 11 99999-1111 | joao.silva@empresa.com |
| Time IAM | Maria Santos | +55 11 99999-2222 | maria.santos@empresa.com |
| DevOps / Infraestrutura | Carlos Oliveira | +55 11 99999-3333 | carlos.oliveira@empresa.com |
| Conformidade / Compliance | Ana Costa | +55 11 99999-4444 | ana.costa@empresa.com |

### Suporte Externo

| Plataforma | Suporte | URL |
|-----------|---------|-----|
| Confluent Cloud | Confluence Support | https://support.confluent.io |
| SailPoint | SailPoint Support | https://support.sailpoint.com |
| Microsoft | Microsoft Support | https://support.microsoft.com |

### Escalação de Incidentes

**Nível 1 (< 2 horas):**
- Contato: Time IAM
- Resposta esperada: 30 minutos

**Nível 2 (< 4 horas):**
- Contato: Arquitetura de Segurança + SailPoint TAM
- Resposta esperada: 1 hora

**Nível 3 (< 24 horas):**
- Contato: Diretor de Segurança + Confluence TAM + Microsoft Premier Support
- Resposta esperada: 2 horas

---

## 📝 Notas e Observações

### Performance

- Batch operations: Confluent Cloud suporta até **50 role bindings** em uma única request
- Account aggregation: paginar com `page_size=100` para melhor performance
- Rate limiting: 100 requests/min por API Key (verificar limites atuais na documentação)

### Segurança

- Rotacionar Cloud API Keys a cada **90 dias**
- Nunca commitar credenciais em repositórios Git
- Usar secretos versionados no cofre corporativo
- Auditar todas as operações via Confluent Audit Log

### Manutenção

- Validar role bindings periodicamente via Access Certifications
- Remover Service Accounts órfãos (sem dono) automaticamente
- Monitorar uso de API Keys não rotacionadas
- Testar procedimento de failover mensalmente

---

**Documento Preparado por:** Time de Arquitetura de Segurança  
**Última Revisão:** 23 de Novembro de 2026  
**Próxima Revisão:** 23 de Maio de 2027
