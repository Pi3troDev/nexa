# Segurança

> Status: **Proposto** · Princípio: segurança por padrão, menor privilégio, defesa em profundidade, sem segredos no código.

## 1. Ativos e modelo de ameaça

**Ativos**: agenda, tarefas, rotina de sono e hábitos (dados pessoais sensíveis por natureza); credenciais e passkeys; inscrições push; chaves (VAPID privada, API do LLM, chaves de Data Protection); trilha de auditoria.

**Atores de ameaça**: atacante remoto não autenticado; usuário malicioso (se multiusuário — D-08); XSS/conteúdo injetado; comprometimento de dependência (supply chain); provedor externo (LLM/push) como terceiro com acesso a dados; dispositivo compartilhado/perdido.

| STRIDE | Ameaça | Controle principal |
|---|---|---|
| Spoofing | Roubo/abuso de sessão; credential stuffing | Cookie `HttpOnly; Secure; SameSite`, passkeys/MFA, lockout, rate limit, `SecurityStamp` revalidado |
| Tampering | CSRF; adulteração de payload; mudança indevida por IA | Antiforgery + SameSite; validação estrita; **aprovação explícita** para IA; ETag/If-Match |
| Repudiation | "Eu não fiz isso" | Audit append-only, incl. aprovações da IA com usuário/horário |
| Information disclosure | IDOR; vazamento em logs; envio de dados ao LLM; cache do SW | Filtro global por `user_id` + `404` para alheio; logs sem conteúdo; consentimento e minimização para IA; SW sem cache de API autenticada |
| Denial of service | Abuso de endpoints caros (IA); tempestade de retries | Rate limiting por usuário/IP, cotas de IA, timeouts, circuit breaker, limites de payload |
| Elevation of privilege | Quebra de isolamento entre usuários; IA agindo além do permitido | Query filter + testes de isolamento; IA executa **como o usuário**, catálogo fechado de mudanças |

## 2. Autenticação (ver [ADR-0003](../adr/0003-authentication.md))

- **ASP.NET Core Identity** como módulo `Identity`; autenticação por **cookie de sessão** (aplicação web na mesma origem que a API — sem tokens em `localStorage`).
- **Passkeys (WebAuthn)** como método preferido; senha + **TOTP** como alternativa/segundo fator. Suporte a passkeys no Identity do .NET 10 deve ser validado na implementação; alternativa: biblioteca FIDO2 madura atrás de uma porta (`IPasskeyService`).
- Senhas: hashing do Identity (PBKDF2 com parâmetros atuais; revisitar para Argon2id se disponível), política mínima (tamanho > complexidade), verificação contra senhas vazadas (opcional), lockout progressivo.
- Sessão: expiração deslizante com teto absoluto; `SecurityStamp` revalidado a cada poucos minutos → logout remoto/troca de senha invalidam sessões; reautenticação para ações sensíveis (excluir conta, gerenciar credenciais).
- **Registro**: desabilitado por padrão (`Identity:Registration:Enabled=false`) — o produto é pessoal; abre-se por convite/flag se virar multiusuário (D-08).
- Troca futura para IdP externo (OIDC) é possível porque o restante do sistema só consome `ICurrentUser` (`UserId`) — nenhum módulo conhece cookies, Identity ou claims específicas.

## 3. Autorização

1. **Toda requisição de dados exige usuário autenticado** (política de fallback `RequireAuthenticatedUser`; endpoints públicos são whitelist explícita: health, config pública, login).
2. **Autorização por propriedade (ownership)**: não há papéis de negócio. Cada agregado pertence a `UserId`; repositórios aplicam *global query filter*; handlers recebem `ICurrentUser` — nunca `userId` vindo do corpo/URL. Recurso inexistente **ou** alheio → `404`.
3. IA, Worker e eventos operam com identidade explícita: o Worker processa **por `UserId` do registro**; comandos cross-módulo exigem `ICurrentUser` (HTTP) ou um `ExecutionContext` com `UserId` e `ActorKind` (jobs) — nunca "admin implícito".
4. Papel administrativo: **não existe** na v1 (sem requisito). Operações de manutenção ocorrem por CLI/migrations com credenciais de operação, fora da API.
5. Testes de isolamento obrigatórios por módulo: usuário B não lê/edita/apaga dados do usuário A (ver [`testing-strategy.md`](testing-strategy.md)).

## 4. Proteções da superfície web

| Controle | Detalhe |
|---|---|
| TLS | Obrigatório; HSTS (`max-age` longo, `includeSubDomains`); redirecionamento HTTP→HTTPS no edge |
| CSRF | `SameSite=Lax` + antiforgery (token em header) em métodos não seguros |
| CORS | **Sem CORS**: mesma origem. Qualquer exceção exige ADR |
| CSP | `default-src 'self'`; `script-src 'self'` (sem `unsafe-inline`/`eval`; hashes/nonces se necessário); `worker-src 'self'`; `img-src 'self' data: blob:`; `connect-src 'self'`; `frame-ancestors 'none'`; `object-src 'none'`. Three.js/R3F não exigem `unsafe-eval` — manter assim (shaders customizados são strings GLSL, não `eval`) |
| Headers | `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` mínimo, COOP/CORP conforme necessário |
| Rate limiting | `AddRateLimiter`: login (estrito por IP+conta), mutações (por usuário), IA (por usuário/dia), anônimos (por IP) → `429` + `Retry-After` |
| Limites | Tamanho de corpo, de listas, de strings; paginação máxima; timeouts de requisição |
| Entrada | Validação (FluentValidation) + normalização; saída codificada pelo React (sem `dangerouslySetInnerHTML`; se markdown for adotado, sanitizar) |
| Dados em trânsito/repouso | TLS entre componentes; criptografia em repouso do volume/serviço de banco; campos especialmente sensíveis (chaves de push `p256dh/auth`) com ASP.NET Data Protection |

## 5. PWA / Service Worker

- Escopo do SW restrito à origem; versionamento do precache; update com *prompt*.
- **Nunca** cachear respostas autenticadas de `/api` por padrão (`Cache-Control: no-store`/`private` na API); lista explícita e curta do que pode ser cacheado para leitura offline (D-07).
- Logout: limpar Cache Storage, IndexedDB da app, query cache e **remover a push subscription** do dispositivo (servidor e navegador) — evita notificações do usuário A para o usuário B em dispositivo compartilhado.
- Push exige gesto do usuário para pedir permissão; sem pedir ao carregar.
- Payload de push: conteúdo mínimo (título curto + id + deep link); opção "ocultar conteúdo" (D-14). Trate o `endpoint` da subscription como segredo (permite enviar push a quem o possuir, embora exija VAPID).

## 6. Segurança da IA (resumo; ver [`ai-architecture.md`](ai-architecture.md))

- **Consentimento explícito** por usuário antes de qualquer dado ir ao provedor; escopos de dados granulares; padrão = desligado.
- **Minimização**: enviar só o necessário ao escopo (ex.: títulos e horários, sem notas livres, a menos que habilitado).
- **Prompt injection**: conteúdo do usuário (títulos, notas, descrições) é **dado não confiável**; delimitado e rotulado no prompt; o LLM não tem ferramentas de escrita; saída só vira ação via parser tipado + catálogo fechado + aprovação humana. A defesa final é a aprovação — a UI mostra o **diff exato** do que será alterado.
- Sem segredos, tokens ou dados de outros usuários no contexto (contexto é montado por usuário, nunca compartilhado).
- Cotas e limites de custo; logging sem prompts/respostas por padrão.
- Contrato com o provedor: verificar política de retenção/treinamento de dados antes de ativar (pendência D-03).

## 7. Segredos e configuração

- Nenhum segredo no repositório, imagens ou bundle do frontend. Fonte: secret store da plataforma → variáveis de ambiente (ver `operations.md` §3).
- Segredos: connection string, **VAPID private key**, **chave do LLM**, material de assinatura. Rotação documentada (VAPID: trocar a chave **invalida** subscriptions existentes → exige re-subscribe; tratar como evento planejado).
- `gitleaks` no CI e pre-commit opcional; `.env` ignorado; `appsettings.*` sem valores sensíveis.
- Data Protection keys persistidas no banco (e protegidas), com rotação automática.

## 8. Supply chain e pipeline

- **Dependabot/Renovate** (NuGet, npm, GitHub Actions, Docker). Versões fixas (lockfiles; `packages.lock.json`; CPM).
- CI: `dotnet list package --vulnerable --include-transitive`, `npm audit` (nível alto bloqueia), **CodeQL**, **gitleaks**, **Trivy** (imagem), SBOM (CycloneDX) como artefato.
- Actions fixadas por SHA; permissões mínimas do `GITHUB_TOKEN`; deploy com OIDC federado (sem chaves de longa duração).
- Imagens: base mínima, usuário não-root, sistema de arquivos somente leitura quando possível, sem shell na imagem final.
- Escolha de bibliotecas: preferir mantidas e amplamente usadas; evitar libs de licença/estado incertos para caminhos críticos (push, auth).

## 9. Privacidade e retenção

- Isolamento por usuário e minimização em logs (sem títulos, notas, e-mails em claro; identificadores opacos).
- Exportação e exclusão de dados do usuário projetadas desde já (evento `UserDeleted`; fluxo em `data-flow.md` §7). Obrigações legais específicas (ex.: LGPD) a confirmar — D-15.
- Retenção: outbox processado (dias), idempotency keys (24 h), inbox de eventos (dias), lembretes concluídos (meses), auditoria (período definido — D-15), logs (curto).

## 10. Checklist de revisão de segurança para PRs

- [ ] Endpoint exige autenticação e usa `ICurrentUser` (nenhum `userId` vindo do cliente)?
- [ ] Entidade nova implementa `IUserOwned` e tem teste de isolamento?
- [ ] Entrada validada e limitada? Resposta sem dados internos?
- [ ] Log sem dados pessoais/segredos?
- [ ] Nova dependência: licença, manutenção, scan limpos?
- [ ] Mudança em IA: continua apenas propondo? Catálogo/permissões inalterados ou revisados?
- [ ] Cache do SW: nada autenticado novo foi cacheado?
