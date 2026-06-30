# GestorFiscal PRO — Pacote de Migração para Nova Conta

Este documento é para o **próximo Claude** (em outra conta) configurar o projeto do zero
em uma **nova conta Supabase**, mantendo tudo funcionando exatamente como está.

## O que é o projeto

App SaaS de precificação tributária para empresas (foco: Acre), em **um único arquivo HTML**
(`GESTORFISCAL.html` = `index.html`, sempre mantidos idênticos). Sem build, sem backend próprio —
roda 100% no navegador, com Supabase como backend (auth + banco) e Stripe para pagamento.

Repositório GitHub: `https://github.com/Delta-1/GESTORFISCALPRO` (pode ser outro repo na nova conta).
Publicado via GitHub Pages.

## Passo a passo da migração

### 1. Criar o novo projeto Supabase
- Crie o projeto na nova conta Supabase.
- Abra o **SQL Editor** e rode o arquivo `migration_supabase_completa.sql` (está nesta mesma pasta) —
  ele cria as 3 tabelas (`empresas`, `admins`, `notas_fiscais`) com RLS e a política de admin.
- **Edite a linha do `INSERT INTO admins`** dentro do SQL antes de rodar, trocando o email pelo
  email que vai administrar o sistema na nova conta.

### 2. Pegar a URL e a anon key do novo projeto
- Em Project Settings → API: copie a **Project URL** e a **anon public key**.
- No `GESTORFISCAL.html`, procure por:
  ```js
  const GF_SB_URL = 'https://qaiqflcxfxygynumapyp.supabase.co';
  const GF_SB_KEY = 'eyJhbGci...'; // anon key antiga
  ```
  e troque pelos valores do novo projeto. **Nunca use a service_role key aqui** — só a anon key
  é segura para ficar no frontend.

### 3. Deploy da Edge Function (webhook do Stripe)
- O arquivo `EDGE_FUNCTION_stripe-webhook.ts` (nesta pasta) é o código completo.
- Deploy via MCP do Supabase (`deploy_edge_function`) ou Supabase CLI, com `verify_jwt = false`.
- Depois do deploy, anote a URL gerada (formato: `https://<project-ref>.supabase.co/functions/v1/stripe-webhook`).

### 4. Configurar o Stripe
- Se for usar a MESMA conta Stripe: só precisa criar um novo webhook endpoint apontando para a
  URL da Edge Function do passo 3 (dashboard.stripe.com/webhooks → Add endpoint → eventos
  `checkout.session.completed`, `customer.subscription.deleted`, `invoice.payment_failed`).
- Copie o **Signing secret** (`whsec_...`) gerado e configure como secret `STRIPE_WEBHOOK_SECRET`
  na Edge Function (Supabase Dashboard → Edge Functions → stripe-webhook → Secrets).
- Os 4 Payment Links já existentes no Stripe continuam funcionando normalmente (não precisam
  ser recriados, a menos que a conta Stripe também mude).
- Os links atuais estão hardcoded em `GESTORFISCAL.html` na constante `STRIPE_LINKS` /
  `STRIPE_LINK_UNIVERSAL` — se a conta Stripe mudar, gere novos Payment Links e atualize essa constante.

### 5. Configurar Auth do Supabase
- Authentication → URL Configuration → **Site URL** = a URL do GitHub Pages onde o app está publicado
  (ex: `https://usuario.github.io/GESTORFISCALPRO/`).
- Adicione a mesma URL em **Redirect URLs**.
- Isso é necessário para o fluxo de "Esqueci minha senha" funcionar (o link do email volta pra essa URL).

### 6. Ativar a conta admin
- Depois de alguém se cadastrar no app com o email que você colocou na tabela `admins`, rode:
  ```sql
  UPDATE empresas SET status = 'active', activated_at = NOW()
  WHERE user_id = (SELECT id FROM auth.users WHERE email = 'SEU_EMAIL_ADMIN');
  ```
  (admins sempre pulam a tela de pagamento automaticamente no código, mas a primeira ativação
  no banco precisa ser manual.)

### 7. Sincronizar os dois arquivos HTML
- `GESTORFISCAL.html` e `index.html` devem ser **sempre idênticos** — todo deploy precisa copiar
  um para o outro antes do commit (é assim que o GitHub Pages serve o `index.html`).

## Arquitetura resumida (para contexto)

- **Sem login**: tela de cadastro/login multi-step.
- **Cadastro novo**: vai direto para tela de pagamento (`status='pending_payment'`, default da tabela).
- **Pagamento via Stripe Payment Link** → webhook ativa a conta (`status='active'`) automaticamente.
- **Admin** (email na tabela `admins`): pula a tela de pagamento, vê o ícone de escudo no header,
  acessa o Painel Admin para ativar/suspender qualquer conta manualmente.
- **Upload de XML de NF-e**: parsing 100% client-side (DOMParser), extrai NCM, CST, CEST, CFOP,
  valores, frete, desconto, ICMS, IPI, dados de Suframa.
- **Automação fiscal do Acre**: tabelas `ICMS_AC_ST_NCM` e `ICMS_AC_CEST_NCM` no próprio JS,
  baseadas na Instrução Normativa DIAT nº 1/2023 — calculam ICMS-ST/Antecipação automaticamente
  por NCM + UF de origem do fornecedor. Vários valores ainda são estimativas (documentado em
  comentário no código); só os NCMs comentados como "oficial" foram confirmados na normativa.
- **Sem integração de IA** — foi removida a pedido do cliente, não reimplementar sem pedir.

## Arquivos deste pacote

| Arquivo | Para que serve |
|---|---|
| `migration_supabase_completa.sql` | Roda no SQL Editor do novo Supabase — cria todo o schema |
| `EDGE_FUNCTION_stripe-webhook.ts` | Código da Edge Function do webhook do Stripe |
| `HANDOFF_GESTORFISCAL.md` | Este documento |

## Dúvidas que o próximo Claude pode ter

- **"Por que `notas_fiscais.empresa_id` não tem FK?"** — coluna legada, não é usada ativamente
  pelo app hoje (o histórico filtra por `user_id`). Pode ignorar.
- **"Onde fica a chave do WhatsApp?"** — constante `WHATSAPP_NUMERO` no JS, não depende de banco.
- **"Tem chave do Stripe no código?"** — não, só os Payment Links públicos (URLs `buy.stripe.com/...`),
  que são seguros de expor. A secret key do Stripe nunca deve ir para o frontend.
