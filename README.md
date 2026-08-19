# PMP Usiblend — YPF & Valvoline

Ferramenta web interativa para gerar o Plano Mestre de Produção (PMP) mensal das
linhas Usiblend (YPF + Valvoline), a partir dos 5 arquivos de origem já usados no
processo mensal. Substitui a planilha Excel validada anteriormente, mantendo
exatamente a mesma lógica de cálculo.

## O que a ferramenta faz

1. O analista faz login e envia os 5 arquivos do mês (Plano Inicial Usiblend,
   Estoque + Carteira, Faturamento, Forecast, OTIF) em **Novo PMP**.
2. O servidor calcula, por SKU: Previsão Líquida, Necessidade Total, Meta de
   Cobertura, Posição Projetada e Ajuste Recomendado (aumentar/diminuir/manter),
   com os mesmos parâmetros e fórmulas validados na planilha Excel original.
3. O resultado fica salvo no histórico (`/dashboard`), pode ser filtrado por
   marca/recomendação/divergência na tela, e exportado em Excel (.xlsx).
4. Administradores (João e Everton) gerenciam os usuários que podem acessar a
   ferramenta em **Usuários** (`/admin/users`): criar, trocar papel, desativar,
   redefinir senha.

## Stack técnica

Next.js 16 (App Router) + TypeScript + Tailwind CSS v4, Prisma ORM (SQLite em
desenvolvimento, Postgres em produção), NextAuth v5 (Credentials + JWT), bcrypt
para senha, ExcelJS para exportação.

## Rodando localmente

Pré-requisitos: Node.js 20+.

```bash
npm install
cp .env.example .env
```

Edite `.env`:
- `DATABASE_URL` pode ficar como está (`file:./dev.db`) para uso local.
- `AUTH_SECRET`: gere um valor aleatório com `npx auth secret` (ou qualquer string longa e aleatória).
- `SEED_ADMIN1_PASSWORD` / `SEED_ADMIN2_PASSWORD`: defina temporariamente as senhas
  iniciais dos dois administradores (João e Everton) — **apenas para rodar o seed uma vez**.

Depois:

```bash
npx prisma migrate deploy   # cria as tabelas
npx prisma db seed          # cria os 2 usuários admin com as senhas do .env
npm run dev
```

Acesse `http://localhost:3000`. Os dois admins são forçados a trocar a senha
inicial no primeiro acesso (o fluxo de troca ainda usa a página de conta —
por ora, a senha pode ser redefinida via **Usuários** por qualquer admin,
incluindo a própria conta trocando pela tela de "Redefinir senha").

**Importante:** depois de rodar o seed, apague `SEED_ADMIN1_PASSWORD` e
`SEED_ADMIN2_PASSWORD` do `.env` (ou do ambiente do Vercel) — elas não são mais
necessárias e não devem ficar salvas em lugar nenhum em texto puro.

## Publicando no GitHub

```bash
git init
git add .
git commit -m "PMP Usiblend - versão inicial"
git branch -M main
git remote add origin <URL_DO_SEU_REPOSITORIO_GITHUB>
git push -u origin main
```

O `.gitignore` já exclui `node_modules`, `.env`, `.next` e o banco SQLite local
(`*.db`) — nenhuma senha ou segredo é versionado.

## Deploy no Vercel

A ferramenta foi desenhada para rodar no Vercel. Como o Vercel não mantém disco
persistente entre execuções, é necessário usar um banco Postgres externo em vez do
SQLite local (o schema do Prisma já é portável entre os dois).

1. **Criar um banco Postgres gratuito** — [Neon](https://neon.tech) ou
   [Supabase](https://supabase.com) funcionam bem no free tier. Copie a
   connection string (formato `postgresql://...`).
2. **Trocar o provider do Prisma**: em `prisma/schema.prisma`, mude
   `provider = "sqlite"` para `provider = "postgresql"` no bloco `datasource db`.
3. **No painel do Vercel** (Project Settings → Environment Variables), defina:
   - `DATABASE_URL` = a connection string do Postgres
   - `AUTH_SECRET` = um valor gerado com `npx auth secret`
   - `NEXTAUTH_URL` = a URL final do projeto (ex: `https://pmp-usiblend.vercel.app`)
   - `SEED_ADMIN1_PASSWORD` e `SEED_ADMIN2_PASSWORD` = senhas temporárias dos admins
     (defina só para a primeira execução do seed — depois remova essas duas variáveis)
4. **Importar o repositório no Vercel** (New Project → selecione o repo do GitHub).
   O Vercel detecta Next.js automaticamente.
5. **Rodar as migrations + seed uma vez** contra o banco de produção. A forma mais
   simples é rodar localmente apontando para o Postgres do Vercel:
   ```bash
   DATABASE_URL="<connection string do Postgres>" npx prisma migrate deploy
   DATABASE_URL="<connection string do Postgres>" SEED_ADMIN1_PASSWORD="..." SEED_ADMIN2_PASSWORD="..." npx prisma db seed
   ```
6. Depois do primeiro deploy funcionando, remova `SEED_ADMIN1_PASSWORD` e
   `SEED_ADMIN2_PASSWORD` das variáveis de ambiente do Vercel.

Depois disso, qualquer novo usuário é criado direto pela tela **Usuários**
(`/admin/users`) — o seed só é necessário uma vez, para os dois primeiros admins.

## Lógica de cálculo (idêntica à planilha validada)

Para cada SKU da lista oficial Usiblend (códigos distintos do Plano Inicial):

- **Marca**: YPF se o código tem 5 dígitos iniciados em "50" (ex: 50419.32.3);
  Valvoline nos demais casos (ex: 419.32.3, 501.12.3).
- **Previsão Líquida** = `max(Forecast do mês − Vendas já realizadas no mês, 0)`
- **Necessidade Total** = `Carteira (pedidos em aberto) + Previsão Líquida`
- **OTIF a Produzir** = soma, por OP de Envase do mês, de
  `max(Litros Previsto − Litros Produzido, 0)` — não usa o campo bruto "a produzir"
  da fonte, que fica zerado incorretamente em OPs já encerradas com produção parcial.
- **Meta de Cobertura** = `Forecast do mês × (dias de cobertura-alvo / 30)`
- **Posição Projetada** = `Estoque Atual + OTIF a Produzir − Necessidade Total`
- **Ajuste Recomendado** = `Meta de Cobertura − Posição Projetada`, arredondado em
  lotes do tamanho mínimo de produção
- **Recomendação**: "Manter" se o ajuste está dentro da tolerância configurada;
  senão "Aumentar produção" ou "Diminuir produção"
- Forecast: se o mês de referência não tem forecast oficial ainda, usa o mês
  anterior mais recente disponível como proxy (fica sinalizado na tela e no Excel).

Essas fórmulas foram validadas linha a linha contra a planilha Excel PMP
Usiblend YPF/Valvoline de Agosto/2026 (96 SKUs, 45 YPF / 51 Valvoline, 41
aumentar / 33 diminuir / 22 manter, 33 divergências Plano×OTIF). A única
diferença encontrada foi uma correção: a planilha original zerava
silenciosamente o estoque de um SKU cujo CSV usava vírgula como separador
decimal (ex: "34782,53"); a ferramenta web lê esse valor corretamente.

## Estrutura do projeto

```
src/lib/pmp/        parsers dos 5 arquivos + motor de cálculo (types.ts, parsers.ts, engine.ts)
src/app/api/pmp/     geração de PMP, listagem, detalhe, exportação Excel
src/app/api/admin/   CRUD de usuários (somente admin)
src/app/pmp/         tela de upload (novo PMP) e tela de resultado
src/app/admin/       tela de administração de usuários
src/app/dashboard/   histórico de PMPs gerados
src/auth.ts          configuração de login (NextAuth, Credentials)
src/proxy.ts         proteção de rotas por papel (ADMIN / ANALYST)
prisma/schema.prisma modelo de dados (User, PmpRun)
```

## Papéis de acesso

- **ADMIN** (João e Everton): tudo que o Analista pode, além de criar/editar/
  desativar usuários e redefinir senhas em `/admin/users`.
- **ANALYST**: gera PMPs e visualiza o histórico de todos os PMPs gerados pela
  equipe (não há isolamento por usuário — é uma equipe pequena e todos precisam
  ver o mesmo histórico).

## Arquivos de origem esperados

| Arquivo | Formato | Uso |
|---|---|---|
| Plano Inicial Usiblend | .xlsx | Lista oficial de SKUs Usiblend + plano de produção do início do mês |
| Estoque + Carteira | .csv (`;`, decimal `,`) | Posição de estoque e pedidos de venda em carteira |
| Histórico de Faturamento | .xlsx | Vendas realizadas (filtrado automaticamente para o mês de referência) |
| Histórico de Forecast | .xlsx | Previsão de vendas por mês |
| OTIF | .xlsx | Ordens de produção (considera só classificação "2 - ENVASE" do mês) |
