# Efeito Vendas — Quiz de Diagnóstico

Landing page de captação (quiz de 8 passos) que qualifica leads e:

- envia **todos** os leads para o Supabase (tabela `leads`);
- redireciona **qualificados** (quente/morno) para o Calendly;
- redireciona **nurture** (frio/iniciantes) para o grupo VIP do WhatsApp.

É um site estático puro: um único `index.html`, sem build, sem framework.

---

## Estrutura

```
.
├── index.html       # quiz completo (HTML + CSS + JS inline)
├── vercel.json      # config de cache/headers da Vercel
├── .gitignore
└── README.md
```

---

## Deploy na Vercel

O repositório já está conectado ao projeto Vercel.
Cada `git push` na branch `main` dispara um deploy automático.

Como é site estático, **não há build step**: a Vercel apenas serve o `index.html`
e aplica os headers do `vercel.json`.

---

## Domínio (`quiz.consultoriaefeitovendas.com.br`)

A ideia é expor o quiz num **subdomínio dedicado**, sem interferir no app
principal já hospedado em `consultoriaefeitovendas.com.br`.

### 1. Vercel → Adicionar o subdomínio

1. Vá no projeto do quiz na Vercel → **Settings → Domains**.
2. Adicione: `quiz.consultoriaefeitovendas.com.br`.
3. A Vercel vai mostrar **qual registro DNS criar**. Hoje ela exige um
   CNAME apontando para um host único do projeto, no formato
   `<id>.vercel-dns-017.com` (use **exatamente** o valor que aparecer na
   sua aba *DNS Records* — não use o antigo `cname.vercel-dns.com`).

### 2. Hostinger → Criar o registro DNS

Como os nameservers atuais são do parking (`ns1.dns-parking.com` /
`ns2.dns-parking.com`), você tem duas opções:

**Opção A (recomendada):** continuar gerenciando o DNS no Hostinger.

1. Hostinger → **Domain Overview → DNS/Nameservers → Edit**.
2. Adicione um registro:
   - **Tipo:** `CNAME`
   - **Nome:** `quiz`
   - **Valor:** o host que a Vercel mostrar em *DNS Records*
     (ex.: `1525f46deb78a3a8.vercel-dns-017.com`). **Não** use o antigo
     `cname.vercel-dns.com`.
   - **TTL:** padrão (3600).
3. Salve. A propagação leva de minutos a algumas horas.
4. Volte na Vercel e clique em **Refresh**; o status deve ir de
   `Invalid Configuration` para `Valid Configuration`.

**Opção B:** trocar os nameservers para a Vercel
(`ns1.vercel-dns.com` e `ns2.vercel-dns.com`).
**Atenção:** isso transfere o controle de **todo o DNS do domínio raiz**
para a Vercel — se o app principal está hospedado fora da Vercel, prefira
a Opção A.

### 3. Verificar

Depois da propagação:

```bash
dig +short quiz.consultoriaefeitovendas.com.br CNAME
# deve responder algo como: 1525f46deb78a3a8.vercel-dns-017.com.
```

A aba **Domains** do projeto na Vercel passa para `Valid Configuration` e o
HTTPS é provisionado automaticamente.

---

## Integração com Supabase

Credenciais estão **inline** no `index.html` (chave `anon`, segura para uso
em front-end **desde que** as policies RLS estejam corretas):

- URL: `https://gtlqjeuuwhxoxsffvepg.supabase.co`
- Tabela destino: `leads`

A função `saveToSupabase()` faz `POST /rest/v1/leads` ao final do quiz e
grava todos os campos do lead (incluindo `tier`, `source`, `origem`,
histórico e snapshot bruto em `raw`).

> **Importante:** revise as policies RLS no Supabase para permitir
> **apenas `INSERT`** público na tabela `leads` (sem `SELECT`/`UPDATE`/`DELETE`).

---

## Roteamento de leads

Calculado em `calcTemperatura(data)`:

| Temperatura | Critério                                                                    | Destino             |
| ----------- | --------------------------------------------------------------------------- | ------------------- |
| `quente`    | Não-iniciante + orçamento reservado **ou** quer ver caso de sucesso         | Calendly            |
| `morno`     | Não-iniciante + "Talvez, depende do valor"                                  | Calendly            |
| `frio`      | Iniciante **ou** "só quero orientação gratuita"                             | Grupo VIP WhatsApp  |

URLs configuradas no topo do `<script>`:

```js
const URL_QUALIFIED = 'https://calendly.com/efeitovendas02/30min';
const URL_NURTURE   = 'https://chat.whatsapp.com/JekqKChZndmChlCCYytCHw';
```

---

## Utilitários de admin (via querystring)

Abrir a página com:

- `?export=qualificados` → baixa CSV dos qualificados do `localStorage` (fallback).
- `?export=nurture`      → baixa CSV dos nurture do `localStorage` (fallback).
- `?export=todos`        → baixa CSV unificado.
- `?limpar=sim`          → limpa o `localStorage` local.

> O `localStorage` só é populado quando o `POST` no Supabase **falha**
> (fallback). Em operação normal, todos os leads estão no Supabase.

---

## Desenvolvimento local

Como é HTML puro, basta abrir `index.html` no navegador.
Para evitar problemas de CORS/`file://`, prefira servir via HTTP:

```bash
# Python 3
python3 -m http.server 5173
# ou Node
npx serve .
```

Acesse `http://localhost:5173`.

> Atenção: ao finalizar o quiz em ambiente local, **um lead real é gravado**
> no Supabase de produção. Use dados fictícios para testar e remova-os depois
> no painel do Supabase, ou comente temporariamente a chamada `saveToSupabase`.
