# EventoTiguan — Guia de Deploy para Produção

Sistema de check-in via QR Code para o **Lançamento Nova Tiguan 2026 — Mavel Volkswagen**.

Stack: **Next.js 14** (App Router, TypeScript) + **MySQL 8** + **Node.js 18+**.

---

## 1. Requisitos do servidor

- **Node.js** ≥ 18.17 (recomendado 20 LTS)
- **npm** ≥ 9 (vem junto com o Node)
- **Acesso de rede ao MySQL** que hospeda o banco `dw_evento` (já provisionado em `18.209.74.87:3310`)
- **Porta HTTPS pública** (443) — **obrigatório**, pois os convidados acessam via celular pela rede móvel; navegadores modernos bloqueiam várias APIs em HTTP
- **Reverse proxy** (Nginx, Apache ou Cloudflare Tunnel) terminando TLS e fazendo proxy para a porta interna do app (sugestão: porta `3008`)
- **Domínio público com certificado válido** (Let's Encrypt funciona perfeitamente). O QR code precisa apontar para esse domínio.

---

## 2. Variáveis de ambiente

Renomeie `web/.env.local.example` para `web/.env.local` e preencha com os valores reais.

```env
# MySQL dw_evento
DB_HOST=18.209.74.87
DB_PORT=3310
DB_USER=<usuario>
DB_PASSWORD=<senha>
DB_DATABASE=dw_evento

# JWT — gere com: openssl rand -hex 32
JWT_SECRET=<string-aleatoria-de-no-minimo-32-caracteres>
```

> **Importante:** o `JWT_SECRET` deve ser único da produção e nunca commitado em repositório.

---

## 3. Instalação e build

A partir do diretório descompactado:

```bash
cd EventoTiguan/web
npm ci          # instala dependências exatas do package-lock.json (mais rápido e reprodutível que npm install)
npm run build   # gera o build otimizado em .next/
```

O build deve terminar sem erros. Se houver erro de tipo, verifique a versão do Node.

---

## 4. Banco de dados

O schema já existe em produção. Caso precise recriar do zero:

```bash
mysql -h 18.209.74.87 -P 3310 -u <root> -p < db/criar_dw_evento.sql
```

Cria 3 tabelas: `convidados`, `checkin_log`, `admin_user`.

### Criar usuário admin

Pelo menos um admin precisa existir antes de o painel funcionar. No servidor (ou de qualquer máquina com Python 3.12 + acesso ao MySQL):

```bash
cd EventoTiguan/importer
py -3.12 -m pip install -r requirements.txt
py -3.12 atualiza_senha_admin.py
# preencher email + nome + senha
```

Se não houver Python disponível, basta inserir manualmente um registro com hash bcrypt em `admin_user`.

### Importar planilha de convidados (opcional — só se for nova base)

```bash
py -3.12 importar_planilha.py
```

Espera arquivo `Cota_convites_enviados.xlsx` na raiz do projeto.

---

## 5. Rodar em produção

### Opção A — `npm start` direto (mais simples)

```bash
cd EventoTiguan/web
PORT=3008 npm start
```

Aplicação fica disponível em `http://localhost:3008`. Configure o reverse proxy para apontar para essa porta.

### Opção B — PM2 (recomendado, mantém ativo após reboot)

```bash
npm install -g pm2
cd EventoTiguan/web
pm2 start npm --name evento-tiguan -- start
pm2 save
pm2 startup    # gera comando para auto-iniciar com o sistema
```

Logs:

```bash
pm2 logs evento-tiguan
```

### Opção C — Docker (se preferirem)

Não há Dockerfile no projeto, mas é trivial adicionar um padrão Next.js:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY web/package*.json ./
RUN npm ci
COPY web/ .
RUN npm run build
EXPOSE 3000
CMD ["npm","start"]
```

---

## 6. Configuração do reverse proxy (Nginx)

Exemplo de config em `/etc/nginx/sites-available/evento-tiguan`:

```nginx
server {
    listen 443 ssl http2;
    server_name evento-tiguan.mavelveiculos.com.br;

    ssl_certificate     /etc/letsencrypt/live/evento-tiguan.mavelveiculos.com.br/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/evento-tiguan.mavelveiculos.com.br/privkey.pem;

    client_max_body_size 10M;

    location / {
        proxy_pass http://127.0.0.1:3008;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}

server {
    listen 80;
    server_name evento-tiguan.mavelveiculos.com.br;
    return 301 https://$host$request_uri;
}
```

---

## 7. Rotas / URLs

| Rota | Acesso | Descrição |
|---|---|---|
| `/` | público | Tela de check-in (QR code aponta aqui) |
| `/admin/login` | público | Login do painel administrativo |
| `/admin` | autenticado | Dashboard com KPIs e gráficos |
| `/admin/convidados` | autenticado | Tabela de convidados + filtros + export Excel |
| `/api/checkin/*` | público | Endpoints do fluxo de check-in |
| `/api/admin/*` | autenticado | Endpoints do painel |
| `/api/auth/*` | público (login) | Login / logout |

---

## 8. QR code

Após hospedar, gere o QR code apontando para a URL pública da raiz:

```
https://evento-tiguan.mavelveiculos.com.br/
```

Qualquer gerador serve (qrcode-monkey.com, ou bibliotecas como `qrcode` no terminal). Imprima e exponha no estande.

---

## 9. Checklist de smoke test pós-deploy

1. `https://<dominio>/` carrega tela de identificação por telefone
2. Tela está mobile-first (testar pelo celular real, não emulador)
3. Login admin em `/admin/login` autentica e redireciona para `/admin`
4. Dashboard exibe KPIs (mesmo que zerados, deve carregar sem erro)
5. `/admin/convidados` carrega lista paginada com filtros
6. Botão "Exportar Excel" na lista funciona
7. Logs do servidor mostram acesso ao MySQL sem erros
8. Após um check-in de teste, registro aparece em `convidados.data_checkin` e em `checkin_log`

---

## 10. Suporte / dúvidas

Desenvolvido por **Klinger Lima** — `klinger@mavelveiculos.com.br`.

Estrutura de pastas:

```
EventoTiguan/
├── DEPLOY.md          ← este arquivo
├── README.md
├── db/
│   └── criar_dw_evento.sql
├── importer/          ← scripts Python (admin + import planilha)
└── web/               ← aplicação Next.js (deploy aqui)
    ├── public/img/    ← logos + imagens da oferta
    ├── src/           ← código-fonte
    └── package.json
```
