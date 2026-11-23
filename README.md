# IGA: Confluent Cloud + SailPoint IdentityNow + Microsoft Entra ID

A solução integra o SailPoint IdentityNow, o Microsoft Entra ID e as APIs do Confluent Cloud para gerenciar o ciclo de vida completo de identidades (JML — Joiner, Mover, Leaver), com governança centralizada, rastreabilidade ponta-a-ponta e conformidade regulatória. O objetivo é eliminar credenciais locais humanas e garantir acesso controlado e auditável ao Confluent Cloud para usuários e aplicações. A autenticação de identidades humanas ocorre exclusivamente via SSO com o Microsoft Entra ID e SAML 2.0, e o SailPoint atua como fonte única da verdade (single source of truth) da governança de identidades. O provisionamento e a revogação no Confluent Cloud são automatizados pelas APIs REST de IAM (`/iam/v2`), por meio do Web Services Connector.

## Documentação

- [Referência técnica](docs/referencia-tecnica.md) — especificações do conector, endpoints de IAM, padrões de nomenclatura, checklist de implementação e troubleshooting.
- [Resumo executivo](docs/resumo-executivo.txt) — resumo para o time de Arquitetura de Segurança.
