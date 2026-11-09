# Configuração do login Supabase

O cadastro usa Supabase Auth (e-mail e senha) e confirmação por e-mail. A senha nunca é salva pelo HTML. O login será liberado depois de instalar a função que limita tentativas a 5 por e-mail e por IP em 20 minutos.

O projeto Supabase já está conectado em `supabase-config.js` com a URL e a chave **publishable** fornecidas. A loja está em modo de desenvolvimento local:

- `siteUrl`: `http://localhost:5500/`
- `siteOrigin`: `http://localhost:5500`
- `loginRateLimitReady`: `false` até a implantação da função e da migration.

## 1. Configurar autenticação e links de confirmação

No Supabase, abra **Authentication → URL Configuration**:

1. Defina **Site URL** como `http://localhost:5500/`.
2. Em **Redirect URLs**, adicione `http://localhost:5500/**`.
3. Abra **Authentication → Providers → Email**; habilite cadastro por e-mail e senha, mantenha **Confirm email** ativado e defina senha mínima de 8 caracteres.
4. Configure SMTP para entrega confiável dos e-mails. O remetente padrão do Supabase pode ter restrições de teste.

Como o site é aberto por `file://`, confirmação de e-mail e chamadas do navegador podem falhar. No VS Code, abra a pasta do projeto, instale/ative a extensão **Live Server** e use **Open with Live Server**. Confirme que o endereço começa exatamente com `http://localhost:5500/`.

## 2. Implantar o limitador de tentativas

Instale a Supabase CLI, abra um terminal na pasta da loja e associe-a ao projeto:

```powershell
supabase login
supabase link --project-ref arwfryehtykaxjrxsugx
supabase db push
```

Gere um segredo aleatório longo localmente; mantenha-o privado. Os valores do exemplo devem ser inseridos no terminal, nunca neste arquivo nem no chat:

```powershell
supabase secrets set SITE_ORIGIN=http://localhost:5500 RATE_LIMIT_HASH_SECRET=COLOQUE_AQUI_UM_SEGREDO_ALEATORIO_LONGO
supabase functions deploy login-rate-limit
```

O Supabase fornece `SUPABASE_URL`, `SUPABASE_ANON_KEY` e `SUPABASE_SERVICE_ROLE_KEY` automaticamente no ambiente da Edge Function. **Não copie `service_role` para o navegador**: ela nunca deve entrar em `index.html` ou `supabase-config.js`.

Depois que **as duas migrations** e a função forem implantadas com sucesso, altere `loginRateLimitReady` para `true` em `supabase-config.js`. Até então, o cadastro pode ser testado, mas o login permanece bloqueado para não contornar o limite de tentativas.

## 3. Preparar o administrador e o estoque

`supabase db push` também aplica a migration `20261108183700_store_inventory.sql`. Ela cadastra os 16 produtos atuais com estoque inicial **zero**, permite leitura pública do catálogo e restringe alterações a usuários com `role: admin` em `app_metadata`. Não há senha administrativa no HTML nem permissões administrativas em `user_metadata`.

Depois que a conta do proprietário já existir no Supabase Auth, execute no **SQL Editor** do painel, substituindo o exemplo pelo e-mail dessa conta:

```sql
update auth.users
set raw_app_meta_data = coalesce(raw_app_meta_data, '{}'::jsonb) || '{"role":"admin"}'::jsonb
where email = 'SEU_EMAIL_DE_ADMIN';
```

Confirme que a consulta alterou uma linha, saia da conta e entre novamente para atualizar o token. Não atribua a função de administrador a contas de clientes. O painel fica em `http://localhost:5500/admin.html`; use a mesma conta autorizada. A interface sozinha não concede acesso: as políticas RLS do banco também verificam a função no token.

Informe no painel a quantidade que você realmente tem disponível para cada produto; atualize a página da loja para recarregar os valores. O catálogo deixa comprar quando o estoque não pode ser verificado ou está zerado. O checkout consulta novamente o estoque e o preço antes de gerar o Pix. Como a conferência do Pix continua manual, o checkout **não reserva nem baixa** unidades automaticamente: confira o pagamento e ajuste o estoque no painel antes de disponibilizar as unidades novamente. Este fluxo não impede duas compras simultâneas da última unidade; para isso será necessário implementar reserva transacional e integração de confirmação do pagamento.

## 4. Publicar a loja

`localhost` serve apenas para desenvolvimento. Antes de publicar:

1. Hospede o site em HTTPS.
2. Troque `siteUrl` e `siteOrigin` em `supabase-config.js` para o endereço final.
3. Atualize **Site URL** e **Redirect URLs** no Supabase.
4. Atualize o segredo da função com a origem exata, sem barra final:

```powershell
supabase secrets set SITE_ORIGIN=https://SEU_DOMINIO RATE_LIMIT_HASH_SECRET=O_MESMO_SEGREDO_ALEATORIO
```

Não publique a loja antes de implantar o limitador e configurar o SMTP.

## 5. Testar

1. Abra `http://localhost:5500/`, escolha **Criar conta** e cadastre um e-mail real com senha de pelo menos 8 caracteres.
2. Abra o link recebido para confirmar o endereço.
3. Quando o limitador estiver implantado e `loginRateLimitReady` estiver `true`, entre na conta. O UUID do usuário aparece em **Minha conta**.
4. Após cinco solicitações de login para o mesmo e-mail ou IP em 20 minutos, a Edge Function deve responder HTTP 429.

Como os contadores ficam no banco, remova janelas antigas periodicamente no SQL Editor:

```sql
delete from public.login_rate_limits
where updated_at < now() - interval '1 day';
```

O QR Pix descrito na tela de pagamento continua manual e não está conectado à autenticação.
