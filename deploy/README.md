# WA-AKG no Render free + Neon free

O gateway guarda a sessão do WhatsApp no banco (`prisma.authState`), então restart/spin-down
não pede QR novo. Só precisa de Postgres e de alguém cutucando o serviço pra ele não dormir.

## 1. Neon (banco)

1. neon.tech → New Project (região `aws-us-east-1`).
2. Copie a connection string (`postgresql://...`), adicione no final:
   `?sslmode=require&schema=waakg`
3. O `schema=waakg` mantém as tabelas isoladas do resto do projeto.

## 2. Repositório

O `prisma/schema.prisma` precisa de `provider = "postgresql"` (use `deploy/schema.postgres.prisma`).
Suba o WA-AKG num repo seu no GitHub (privado serve). `render.yaml` e `Dockerfile` já vão junto.

## 3. Render

1. render.com → New → Blueprint → selecione o repo (usa `render.yaml`).
2. Preencha as variáveis marcadas `sync: false`:
   - `DATABASE_URL` — a string do Neon com `?sslmode=require&schema=waakg`
   - `BASE_URL` e `NEXTAUTH_URL` — `https://wa-akg.onrender.com` (confirme a URL depois do primeiro deploy)
   - `ADMIN_EMAIL` / `ADMIN_PASSWORD` — login do painel
3. Deploy. O `CMD` já roda `prisma db push` + cria o admin no boot.

## 4. Não dormir

Render free hiberna após 15 min sem tráfego. UptimeRobot (ou cron-job.org), monitor HTTP
a cada 5 min em `https://wa-akg.onrender.com/auth/login`. Isso consome ~744h das 750h/mês.

## 5. Ligar no bar

1. Painel → Sessões → escaneia o QR (mesmo número, aparelho conectado novo).
2. API Key: `POST /api/user/api-key`.
3. No banco do bar:
   ```sql
   update v2.configuracoes set valor = to_jsonb('https://wa-akg.onrender.com'::text) where chave = 'whatsapp_gateway';
   update v2.configuracoes set valor = to_jsonb('<nova api key>'::text) where chave = 'whatsapp_gateway_key';
   update v2.configuracoes set valor = to_jsonb('<sessionId>'::text) where chave = 'whatsapp_sessao';
   ```

## Limites do free

- 512 MB RAM (medido: ~304 MB rodando).
- Upload de mídia some no restart (não usamos — só texto).
- Primeira requisição após dormir demora ~30 s (o ping evita).
- Uso pessoal/baixo volume; o protocolo é o mesmo Baileys (risco de ban existe).
