# EventoTiguan — Lançamento Nova Tiguan 2026 (Mavel Volkswagen)

Sistema online de check-in digital, controle de presença, captura de leads e dashboard analítico em tempo real.

## Componentes

- **`db/`** — DDL `dw_evento` (MySQL) + script de criação.
- **`importer/`** — Importador da planilha `Cota_convites_enviados.xlsx`.
- **`web/`** — Aplicação Next.js 14 (frontend público QR + admin) com TypeScript, Tailwind, Framer Motion, mysql2.

## Fluxos

1. **Visitante (público)** — escaneia QR → digita telefone → 3 caminhos:
   - **Caso 1** Telefone na base & confirmado → confirma nome + email → check-in → tela de oferta exclusiva.
   - **Caso 2** Telefone na base & não confirmado → mensagem amigável + check-in + oferta.
   - **Caso 3** Telefone novo → cadastro (nome/sobrenome/email/origem) + check-in + oferta.
2. **Tela de oferta** — Tiguan 2026 + R$ 69.900,00 + Taxa zero + Bônus na troca → SIM/NÃO.
3. **Admin** — Login JWT → dashboard KPIs + gráficos + tabela + export Excel.

## Banco de dados

- Servidor: `18.209.74.87:3310` (mesmo do `dw_klinger`).
- Database: `dw_evento`.
- Tabelas: `convidados`, `checkin_log`, `admin_user`.

## Setup (resumo)

```powershell
cd D:\Cliente\GrupoRezende\Etl\EventoTiguan

# 1) Banco
py -3.12 -m pip install -r importer\requirements.txt
py -3.12 db\criar_dw.py

# 2) Importar planilha
py -3.12 importer\importar_planilha.py

# 3) Criar usuario admin
py -3.12 importer\atualiza_senha_admin.py

# 4) Web
cd web
npm install
copy .env.local.example .env.local   # editar JWT_SECRET
npm run dev                           # http://localhost:3008
```

## Deploy

Aguarda credenciais FTP/SSH da hospedagem Mavel. Opções:
- **VPS com Node** → `npm run build` + `npm start` (PM2).
- **Vercel** + DNS Mavel apontando subdomínio para Vercel.
- **Shared FTP** → exigirá refatoração para HTML estático + PHP (não recomendado).
