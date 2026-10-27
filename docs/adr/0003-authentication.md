# ADR-0003: Autenticação e autorização

- Status: **Proposto**
- Data: 2026-10-26

## Context

O cliente é uma SPA/PWA servida pela mesma origem que a API. Os dados são pessoais e sensíveis. O produto é pessoal (D-08: um ou poucos usuários), mas os dados já são isolados por `UserId`. Tokens guardados em `localStorage` ficam expostos a XSS; a PWA e o Web Push exigem sessões que sobrevivam a reinícios do app. Não há requisito de login social, SSO corporativo ou papéis administrativos.

## Decision

1. **ASP.NET Core Identity** no módulo `Identity`, com **cookie de sessão** (`HttpOnly`, `Secure`, `SameSite=Lax`) na **mesma origem** que a API (sem CORS).
2. Métodos: **passkeys (WebAuthn)** preferidos; senha + **TOTP** como alternativa/segundo fator. Lockout, rate limiting e `SecurityStamp` revalidado periodicamente para revogação.
3. **CSRF**: `SameSite` + antiforgery em header para métodos não seguros.
4. **Autorização por propriedade (ownership)**: sem papéis de negócio; `ICurrentUser` injetado; query filter global; `404` para recurso alheio. IA e Worker operam com identidade explícita (`UserId` + `ActorKind`).
5. **Registro desabilitado por padrão** (flag). Reautenticação para ações sensíveis.
6. Os demais módulos dependem apenas de `ICurrentUser` e `UserId` — nunca de Identity/cookies/claims — para permitir trocar o mecanismo.
7. Chaves de Data Protection persistidas no banco (multi-instância).

## Alternatives considered

1. **JWT em `Authorization` header + refresh token (armazenados no cliente)**: bom para clientes não-web/terceiros; para uma SPA same-origin aumenta superfície de XSS e complexidade de refresh/revogação. Rejeitado para a v1; pode ser adicionado para clientes futuros.
2. **BFF + OIDC com IdP externo (Keycloak/Entra/Auth0/etc.)**: robusto e delega MFA/passkeys, mas adiciona componente/custo/dependência para um app pessoal. Mantido como **evolução**, viabilizada pelo item 6.
3. **Login social apenas (Google/Apple)**: reduz gestão de credenciais, mas cria dependência de terceiro e não foi solicitado.
4. **Auth própria do zero**: risco de segurança elevado; rejeitado.
5. **Papéis/RBAC/ABAC**: sem requisito; YAGNI. Reavaliar se surgir compartilhamento/colaboração.

## Consequences

**Positivas**: superfície de ataque menor no cliente (token fora do alcance de JS); implementação padrão e bem testada; revogação simples; experiência PWA estável; isolamento forte por usuário.

**Negativas / custos**: cookies exigem mesma origem e proteção CSRF; clientes nativos/terceiros futuros demandarão outro esquema (token/OIDC); suporte a passkeys no Identity do .NET 10 precisa ser validado na implementação (fallback: biblioteca FIDO2 atrás de porta); gestão de credenciais fica sob nossa responsabilidade (hashing, lockout, recuperação de conta — fluxo de recuperação ainda a definir, risco de lockout permanente do único usuário → exigir códigos de recuperação).
