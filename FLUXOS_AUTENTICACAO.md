# Fluxos de Autenticação - Owl App

## Arquitetura de Isolamento

O sistema utiliza **dois clientes Supabase isolados**:

1. **`supabase`** (creator)
   - Storage Key: `sb-creator-auth`
   - Uso: Dashboard, painel admin, criação de apps
   - Sessão: Criador/produtor de conteúdo

2. **`publicSupabase`** (público)
   - Storage Key: `sb-public-owlapp-{slug}` (dinâmico por app)
   - Uso: Páginas públicas (`/app/:slug`)
   - Sessão: Usuário final/lead/visitante

**IMPORTANTE:** Nunca misturar os clientes! Cada contexto usa seu próprio cliente para garantir que o login público não deslogue o criador.

---

## Modo 1: PRODUTO ISCA (Lead Magnet)

**Quando ativado:** `quiz_mode = true` no app

### Características:
- ❌ **SEM autenticação** Supabase Auth
- ✅ **Acesso direto** ao conteúdo (quiz)
- ✅ **Captura de leads** via tabela `leads`
- ❌ **Não cria usuários** na tabela `auth.users`
- ❌ **Não verifica aprovação**

### Fluxo:
1. Usuário acessa `/app/:slug`
2. Sistema detecta `quiz_mode = true`
3. **Pula completamente** `useAppAccess` e `useUserAuth`
4. Renderiza `QuizViewer` diretamente
5. Ao capturar e-mail, salva em `leads` (não em `auth.users`)

### Implementação:
```typescript
// Em PublicApp.tsx
const isQuizMode = app?.quiz_mode === true;

// Bypass auth no modo produto isca
const { status, loading, hasAccess } = isQuizMode 
  ? { status: 'approved', loading: false, hasAccess: true }
  : useAppAccess(appId, autoApproval, creatorId, slug);

// Renderização sem header/footer
if (isQuizMode && hasValidQuiz) {
  return <QuizViewer quizData={quizData} />;
}
```

---

## Modo 2: NORMAL (App Completo)

**Quando ativado:** `quiz_mode = false` no app

### Características:
- ✅ **Requer autenticação** via Supabase Auth
- ✅ **Cria usuários** em `auth.users`
- ✅ **Verifica aprovação** em `app_users`
- ✅ **Controle de acesso** por módulo

### Fluxo:
1. Usuário acessa `/app/:slug`
2. Sistema verifica se está logado
3. **Se NÃO logado:** Redireciona para `/app/:slug/auth`
4. **Se logado:** Verifica status em `app_users`
5. **Status possíveis:**
   - `approved` → Acesso liberado
   - `pending` → Aguardando aprovação
   - `blocked` → Acesso negado

### Sub-modos de Login:

#### A) Login Completo (`login_type = 'full'`)
- Pede **email + senha**
- Usa `supabase.auth.signInWithPassword()`
- Cria conta com confirmação de email (opcional)

#### B) Login Simplificado (`login_type = 'email_only'`)
- Pede **apenas email**
- Usa edge function `free-access-login`
- Cria usuário automaticamente sem senha
- Envia magic link por email (opcional)

#### C) Acesso Gratuito (`auto_approval = true`)
- Similar ao email_only
- Aprova acesso **imediatamente** (`status = 'approved'`)
- Não requer aprovação manual do criador

### Implementação:
```typescript
// Em useAppAccess.tsx
if (!user || !appId) {
  setLoading(false);
  return;
}

// Verifica se é o criador
if (creatorId && user.id === creatorId) {
  setStatus('approved');
  return;
}

// Busca ou cria registro em app_users
const { data } = await supabasePublic
  .from('app_users')
  .select('status')
  .eq('app_id', appId)
  .eq('user_id', user.id)
  .maybeSingle();

if (data) {
  setStatus(data.status); // approved | pending | blocked
} else {
  // Cria novo registro
  const newStatus = autoApproval ? 'approved' : 'pending';
  await supabasePublic
    .from('app_users')
    .insert({
      app_id: appId,
      user_id: user.id,
      status: newStatus,
      approved_at: autoApproval ? new Date() : null
    });
  setStatus(newStatus);
}

// Se aprovado, concede acesso aos módulos
if (status === 'approved') {
  await supabasePublic.functions.invoke('grant-modules-for-app', {
    body: { app_id: appId }
  });
}
```

---

## Prevenção de Loops e Reloads

### Problema Original:
- `useUserAuth` inicializava `onAuthStateChange` em todos os contextos
- Mudanças de sessão pública disparavam reload no criador
- Modo produto isca entrava em loop infinito

### Solução:
```typescript
// Em useUserAuth.tsx
const init = async () => {
  if (currentSlug) {
    const { data } = await supabasePublic.rpc('get_public_app_by_slug', { 
      p_slug: currentSlug 
    });
    const isQuiz = Array.isArray(data) && data[0]?.quiz_mode;
    
    // MODO PRODUTO ISCA: Não inicializa listeners
    if (isQuiz) {
      setLoading(false);
      return;
    }
  }
  
  // MODO NORMAL: Inicializa listeners normalmente
  cleanup = setupAuthListener();
};
```

---

## Edge Functions

### `grant-modules-for-app`
**Propósito:** Conceder acesso a todos os módulos ativos de um app após aprovação

**Uso:**
```typescript
await supabasePublic.functions.invoke('grant-modules-for-app', {
  body: { app_id: appId }
});
```

**Comportamento:**
1. Verifica autenticação do usuário
2. Verifica acesso ao app (criador, auto_approval ou approved)
3. Lista todos os módulos ativos
4. Insere/atualiza registros em `module_access`

**Importante:**
- Usa `SUPABASE_SERVICE_ROLE_KEY` para bypass de RLS
- Só concede acesso se o usuário tiver permissão

### `free-access-login`
**Propósito:** Login simplificado com apenas e-mail (modo email_only)

**Uso:**
```typescript
await supabasePublic.functions.invoke('free-access-login', {
  body: { email, app_slug: slug }
});
```

**Comportamento:**
1. Verifica se o email é do criador (bloqueia se for)
2. Cria/atualiza usuário em `auth.users` com `auth_type = 'app_alias'`
3. Retorna `access_token` e `refresh_token`
4. Cliente define sessão com `supabasePublic.auth.setSession()`

---

## Resumo

| Característica | Modo Produto Isca | Modo Normal |
|----------------|-------------------|-------------|
| Autenticação | ❌ Não | ✅ Sim |
| Cria usuário | ❌ Não | ✅ Sim |
| Verifica aprovação | ❌ Não | ✅ Sim |
| Captura lead | ✅ Sim (`leads`) | ⚠️ Opcional |
| Requer senha | ❌ Não | ⚠️ Depende |
| Mostra header/nav | ❌ Não | ✅ Sim |
| Acesso imediato | ✅ Sim | ⚠️ Se `auto_approval` |

---

## Troubleshooting

### Problema: Criador é deslogado ao abrir app público
**Causa:** Usando `supabase` ao invés de `publicSupabase` em contexto público  
**Solução:** Sempre usar `publicSupabase` em `/app/:slug`

### Problema: Página recarrega sozinha no modo produto isca
**Causa:** `useUserAuth` inicializando listeners em quiz_mode  
**Solução:** Já corrigido - detecta quiz_mode e pula inicialização

### Problema: Usuário aprovado não tem acesso a módulos
**Causa:** Edge function `grant-modules-for-app` não foi chamado  
**Solução:** Chamar após aprovação em `useAppAccess`

### Problema: Login com email_only falha
**Causa:** Edge function `free-access-login` retornando erro  
**Solução:** Verificar logs da função e retry automático
