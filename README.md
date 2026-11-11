# Conciliação Geral · Lope

Conciliação dos checkouts Pagar.me com os extratos de liquidação (Pagar.me, Stone e PagBank), com a marca (Freixenet, MATCON, Everest) em cada transação e em cada repasse.

| Peça | Onde roda | Para quê |
|---|---|---|
| Site (`index.html`, `admin.html`, `app.enc`) | GitHub Pages | Login, gestão de acessos e o sistema (criptografado) |
| Supabase Auth | Supabase | Usuários, senhas, convites e sessões |
| Banco + Storage | Supabase | Base compartilhada, regras, pedidos de acesso e histórico |
| Edge Functions `acesso-publico` e `acesso-admin` | Supabase | Pedidos de acesso, convites, "esqueci a senha" e ações de administrador |
| Resend | resend.com | Envio dos e-mails |

## O que vai para cada lugar

- **GitHub:** `index.html`, `admin.html`, `app.enc`, `config.js`, `.nojekyll`, `README.md`.
- **Supabase:** `supabase_setup.sql` e as duas funções da pasta `functions`. **Não suba o SQL no GitHub**: ele tem a chave que abre o sistema.

---

## Parte 1 — Resend (e-mails)

1. Crie a conta em **resend.com** com o e-mail que vai receber os pedidos de acesso (ex.: `andre.santos@grupottx.com.br`).
2. Em **API Keys → Create API Key**, crie uma chave com **Sending access** e guarde. Ela começa com `re_`.

Sem domínio verificado, o Resend envia a partir de `onboarding@resend.dev` e **só entrega para o e-mail da própria conta**. Por isso:
- O aviso de **pedido de acesso** chega normalmente para o administrador.
- O **convite** e o link de **senha nova** não chegam para as outras pessoas. Nesses casos, a página Acessos mostra o link na tela, com o botão **Copiar link**, para você mandar por WhatsApp ou pelo seu e-mail.
- Se um dia quiser que tudo saia por e-mail, basta verificar um domínio em **Domains** no Resend e cadastrar o segredo `EMAIL_FROM` (ex.: `Conciliação Lope <conciliacao@lope.com.br>`). Nada mais muda.

## Parte 2 — Supabase

**2.1 Projeto e banco**
1. Em **supabase.com**, crie um projeto (região São Paulo).
2. Abra o **SQL Editor → New query**, cole o conteúdo inteiro do `supabase_setup.sql` e clique em **Run**. Ele cria as tabelas, as regras de acesso (RLS), o armazenamento da base e o histórico. Pode rodar de novo sem perder nada.

**2.2 Autenticação**
1. **Authentication → Sign In / Providers → Email**:
   - Desligue **Allow new users to sign up**. Só entra quem for convidado ou aprovado.
   - Deixe **Confirm email** ligado.
2. **Authentication → URL Configuration**:
   - **Site URL:** o endereço do GitHub Pages, ex.: `https://SEU-USUARIO.github.io/conciliacao-geral/`
   - **Redirect URLs:** adicione o mesmo endereço.
3. Em **Authentication → Sign In / Providers → Email**, aumente o **Email OTP Expiration** para `86400` (24 h). Como os links de convite e de senha nova vão ser enviados à mão, assim eles valem por um dia, e não só por uma hora.

**2.3 Primeiro administrador**
1. **Authentication → Users → Add user → Create new user**: seu e-mail e uma senha, com **Auto Confirm User** marcado.
2. No **SQL Editor**, rode (trocando nome e e-mail):
   ```sql
   insert into public.perfis (id, nome, email, papel)
   select id, 'André Santos', email, 'admin' from auth.users where email = 'andre.santos@grupottx.com.br'
   on conflict (id) do update set papel = 'admin', ativo = true;
   ```

**2.4 Edge Functions**
1. **Edge Functions → Deploy a new function → Via Editor**.
2. Crie a função **`acesso-publico`**, cole o `functions/acesso-publico/index.ts` e clique em **Deploy**.
   - Depois, em **Details/Settings** da função, **desligue "Verify JWT"**. Ela atende quem ainda não fez login (pedido de acesso e esqueci a senha).
3. Crie a função **`acesso-admin`**, cole o `functions/acesso-admin/index.ts` e clique em **Deploy**. Aqui o **Verify JWT** fica **ligado**.
4. **Edge Functions → Secrets → Add new secret**, cadastre:

   | Nome | Valor |
   |---|---|
   | `RESEND_API_KEY` | a chave `re_...` do Resend |
   | `ADMIN_EMAILS` | o e-mail da conta Resend, ex.: `andre.santos@grupottx.com.br` |
   | `SITE_URL` | o endereço do GitHub Pages, com a barra no final |

   `SUPABASE_URL` e `SUPABASE_SERVICE_ROLE_KEY` já existem nas funções. `EMAIL_FROM` só entra se um dia você verificar um domínio.

**2.5 Chaves para o site**
Em **Project Settings → API** (em alguns projetos, **API Keys**), copie:
- **Project URL**, ex.: `https://abcdxyz.supabase.co`
- **anon public** (nos projetos novos, **publishable key**, começa com `sb_publishable_`)

⚠️ Nunca use a `service_role` / `secret key` no site.

## Parte 3 — GitHub

1. Crie o repositório `conciliacao-geral` como **Public** (o GitHub Pages gratuito exige isso; o sistema continua protegido pelo login e pela criptografia).
2. Abra o `config.js` no Bloco de Notas e cole a Project URL e a chave pública:
   ```js
   window.CG_CONFIG = {
     SUPABASE_URL: 'https://abcdxyz.supabase.co',
     SUPABASE_ANON_KEY: 'sb_publishable_...'
   };
   ```
   A chave pública pode ficar no GitHub: quem protege os dados são as regras de acesso do banco.
3. Suba `index.html`, `admin.html`, `app.enc`, `config.js`, `.nojekyll` e `README.md` (**uploading an existing file → Commit changes**). Se o `.nojekyll` sumir ao arrastar, crie em **Add file → Create new file** com esse nome e sem conteúdo.
4. **Settings → Pages → Deploy from a branch → main / (root) → Save**. Em 1 ou 2 minutos aparece o endereço. Confira se é o mesmo que você colocou no `SITE_URL` e na URL Configuration do Supabase.

## Parte 4 — Primeiro uso

1. Abra o site e entre com o e-mail e a senha do administrador.
2. Arraste os arquivos. Cada um é salvo na base compartilhada na hora.
3. Teste o fluxo com outra pessoa: **Solicitar acesso** → o e-mail chega para o administrador → **Aprovar e enviar convite** → a pessoa recebe o link, cria a senha e já entra vendo a mesma base.

---

## Como funciona

**Acessos**
- **Pedido de acesso:** pela tela de login. Os `ADMIN_EMAILS` recebem o e-mail com o botão **Analisar solicitação**, e a pessoa recebe a confirmação de que o pedido chegou.
- **Aprovação:** na página **Acessos**, com o papel (Usuário ou Administrador). Sai o convite com o link para **criar a própria senha**. Se o e-mail não puder ser enviado, o link aparece na tela para você copiar e mandar. Ninguém vê nem envia senha.
- **Esqueci a senha:** na aba Usuários, **Gerar link de senha** cria um link novo para a pessoa, que aparece na tela para copiar. (Com domínio verificado no Resend, a própria pessoa também recebe pela tela de login.)
- **Desativar:** na aba Usuários. A pessoa perde o acesso na hora, até com a sessão aberta, porque o banco recusa tudo de quem está inativo.
- **Convidar direto:** na aba Usuários, sem precisar de pedido.

**Papéis**
- **Usuário:** vê tudo e envia arquivos. Enviar um arquivo com o mesmo nome e tipo substitui o anterior.
- **Administrador:** também remove arquivos, limpa a base, muda as regras de marca e gerencia os acessos.

**Base compartilhada**
- Cada arquivo fica compactado no Storage do Supabase, e a lista mostra quem enviou e quando.
- O navegador guarda uma cópia local. Ao abrir de novo, só baixa o que mudou.
- Quando outra pessoa atualiza a base, aparece um aviso no topo com o botão **Atualizar a base**.

**Histórico:** a aba Histórico da página Acessos registra pedidos, aprovações, envios e remoções de arquivos, mudanças de regras e de usuários, com quem fez e quando.

## Atualizar o sistema

Quando houver versão nova, só o `app.enc` muda: **Add file → Upload files → app.enc → Commit changes**. Em 1 ou 2 minutos todo mundo abre a versão nova (se não, Ctrl+F5).

A chave que abre o `app.enc` fica na tabela `app_segredo` do Supabase. Um `app.enc` novo precisa ser gerado com a mesma chave.

## Regras da conciliação (resumo)

- **Pagar.me:** cada marca tem a sua conta. A conta é reconhecida pelos IDs que casam com o checkout e pode ser fixada na lista de arquivos.
- **Stone:** o Stone ID é o NSU/TID do checkout, e a marca vem da venda da Everest ou da MATCON.
- **PagBank:** o código da transação é o TID do checkout sem o "CHAR_". As vendas que não passaram pelo Pagar.me seguem a regra da aba Regras (hoje Freixenet).
- **Divergências:** lançamentos sem venda, com o motivo, e vendas pagas sem liquidação ou com valor diferente.
