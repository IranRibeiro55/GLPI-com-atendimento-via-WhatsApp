# Implantação do GLPI com Integração de Atendimento via WhatsApp

Documentação da implantação de um ambiente **GLPI + Bot de WhatsApp**, permitindo abertura e consulta de chamados por WhatsApp com integração via **API REST do GLPI**.

---

## Objetivo

Esta solução foi implantada para permitir que usuários:

- abram chamados de TI pelo WhatsApp
- escolham uma categoria pelo menu
- enviem descrição textual
- enviem imagem
- enviem PDF
- enviem texto junto com imagem/PDF
- consultem um chamado pelo número
- recebam o status do chamado
- visualizem o último retorno registrado pela TI
- entrem em contato com a equipe de TI por e-mail

---

## Arquitetura da solução

```text
Usuário -> WhatsApp -> Bot Node.js -> API REST do GLPI -> Ticket
```

### Componentes utilizados

| Componente | Tecnologia / Ferramenta |
|---|---|
| Sistema operacional | Debian 12 |
| Servidor web | Apache2 |
| Banco de dados | MariaDB |
| Plataforma de chamados | GLPI |
| Runtime do bot | Node.js |
| Navegador para sessão do WhatsApp | Chromium |
| Biblioteca de integração WhatsApp | whatsapp-web.js |
| Variáveis de ambiente | dotenv |
| Cliente HTTP | axios |
| Geração de QR Code no terminal | qrcode-terminal |

---

## Pré-requisitos

Antes de iniciar, garantir que o servidor possua:

- acesso à internet
- privilégios administrativos
- IP acessível na rede local
- portas necessárias liberadas
- ambiente Debian 12 atualizado

> **Nota**
>
> Os exemplos abaixo consideram que o GLPI será publicado localmente por IP e que o bot será executado no mesmo servidor ou em servidor com acesso HTTP ao GLPI.

---

## Instalação do GLPI

### 1. Atualização do sistema

Execute:

```bash
sudo apt update && sudo apt full-upgrade -y
```

### 2. Instalação dos pacotes necessários

Execute:

```bash
sudo apt install -y \
  apache2 \
  mariadb-server \
  wget tar bzip2 unzip curl \
  php8.2 \
  libapache2-mod-php8.2 \
  php8.2-cli \
  php8.2-common \
  php8.2-mysql \
  php8.2-xml \
  php8.2-curl \
  php8.2-gd \
  php8.2-intl \
  php8.2-bcmath \
  php8.2-mbstring \
  php8.2-zip \
  php8.2-bz2 \
  php8.2-ldap \
  php8.2-opcache
```

### 3. Habilitar os serviços

Execute:

```bash
sudo systemctl enable --now apache2
sudo systemctl enable --now mariadb
```

### 4. Segurança inicial do banco

Execute:

```bash
sudo mysql_secure_installation
```

Aplique as opções recomendadas:

- definir senha para o usuário root do banco
- remover usuários anônimos
- desabilitar login root remoto
- remover base de testes
- recarregar privilégios

### 5. Criação do banco do GLPI

Entre no MariaDB:

```bash
sudo mariadb
```

Depois execute:

```sql
CREATE DATABASE glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpi'@'localhost' IDENTIFIED BY 'SUA_SENHA_FORTE_AQUI';
GRANT ALL PRIVILEGES ON glpidb.* TO 'glpi'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 6. Download e extração do GLPI

Execute:

```bash
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/11.0.6/glpi-11.0.6.tgz
sudo tar -xzf glpi-11.0.6.tgz -C /var/www/
```

Diretório final:

```text
/var/www/glpi
```

### 7. Configuração do Apache

Crie o arquivo:

```bash
sudo nano /etc/apache2/sites-available/glpi.conf
```

Conteúdo do arquivo:

```apache
<VirtualHost *:80>
    ServerName glpi.local
    DocumentRoot /var/www/glpi/public

    <Directory /var/www/glpi/public>
        Require all granted
        RewriteEngine On

        RewriteCond %{HTTP:Authorization} ^(.+)$
        RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]

        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/glpi_error.log
    CustomLog ${APACHE_LOG_DIR}/glpi_access.log combined
</VirtualHost>
```

Ative a configuração:

```bash
sudo a2enmod rewrite
sudo a2dissite 000-default.conf
sudo a2ensite glpi.conf
sudo systemctl reload apache2
```

### 8. Finalização pelo navegador

Acesse:

```text
http://IP_DO_SERVIDOR
```

No assistente gráfico do GLPI, informe:

- banco: `glpidb`
- usuário: `glpi`
- senha: a definida na criação do banco

---

## Configuração da API do GLPI

### 1. Habilitar a API REST

No GLPI, acesse:

```text
Configurar > Geral > API
```

Habilite:

- API REST
- login com credenciais
- login externo por token

### 2. Criar cliente da API

Ainda em:

```text
Configurar > Geral > API
```

Crie um cliente com o nome:

```text
bot-whatsapp
```

Depois salve e copie o **App-Token**.

Esse valor será usado em:

```env
GLPI_APP_TOKEN
```

### 3. Criar usuário técnico do bot

No GLPI, acesse:

```text
Administração > Usuários
```

Crie o usuário:

| Campo | Valor |
|---|---|
| Usuário | wabot |
| Nome | Bot WhatsApp |
| Ativo | Sim |

### 4. Gerar token do usuário

No cadastro do usuário `wabot`, acesse:

```text
Senhas e chaves de acesso
```

Gere ou regenere o **Token de API**.

Esse valor será usado em:

```env
GLPI_USER_TOKEN
```

> **Atenção**
>
> Não confundir:
>
> - `GLPI_APP_TOKEN` = token do cliente da API
> - `GLPI_USER_TOKEN` = token do usuário técnico

---

## Testes iniciais da API

### Teste de sessão

Execute:

```bash
curl -s -X GET \
  -H "Content-Type: application/json" \
  -H "Authorization: user_token SEU_TOKEN_USUARIO" \
  -H "App-Token: SEU_APP_TOKEN" \
  "http://SEU_IP/apirest.php/initSession"
```

Saída esperada:

```json
{"session_token":"TOKEN_AQUI"}
```

### Teste de criação de ticket

Execute:

```bash
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Session-Token: SEU_SESSION_TOKEN" \
  -H "App-Token: SEU_APP_TOKEN" \
  -d '{"input":{"name":"Teste API GLPI","content":"Chamado criado via API","urgency":3,"impact":3,"priority":3}}' \
  "http://SEU_IP/apirest.php/Ticket/"
```

---

## Instalação do bot do WhatsApp

### 1. Instalar Node.js

Execute:

```bash
sudo apt update
sudo apt install -y curl ca-certificates gnupg
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Valide:

```bash
node -v
npm -v
```

### 2. Instalar Chromium e dependências

Execute:

```bash
sudo apt install -y chromium fonts-liberation libasound2 libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgbm1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 xdg-utils
```

### 3. Criar diretório do projeto

Execute:

```bash
mkdir -p /opt/wa-glpi-bot
cd /opt/wa-glpi-bot
```

### 4. Inicializar projeto Node.js

Execute:

```bash
npm init -y
npm install whatsapp-web.js qrcode-terminal axios dotenv
```

---

## Configuração do arquivo `.env`

Crie o arquivo:

```bash
nano /opt/wa-glpi-bot/.env
```

Conteúdo:

```ini
GLPI_URL=http://192.168.1.81
GLPI_APP_TOKEN=SEU_APP_TOKEN
GLPI_USER_TOKEN=SEU_TOKEN_USUARIO
CHROMIUM_PATH=/usr/bin/chromium
TICKET_PREFIX=WhatsApp
```

### Significado das variáveis

| Variável | Função |
|---|---|
| `GLPI_URL` | Endereço base do GLPI |
| `GLPI_APP_TOKEN` | Token do cliente da API |
| `GLPI_USER_TOKEN` | Token do usuário técnico |
| `CHROMIUM_PATH` | Caminho do executável do Chromium |
| `TICKET_PREFIX` | Prefixo usado no título dos chamados |

---

## Criação do arquivo principal do bot

### 1. Remover arquivo antigo, se existir

Execute:

```bash
rm -f /opt/wa-glpi-bot/index.js
```

### 2. Criar novo arquivo

Execute:

```bash
nano /opt/wa-glpi-bot/index.js
```

### 3. Código-fonte do bot

Cole o conteúdo completo do seu `index.js` dentro do arquivo.

Exemplo de início:

```javascript
require('dotenv').config();

const axios = require('axios');
const qrcode = require('qrcode-terminal');
const { Client, LocalAuth } = require('whatsapp-web.js');

const GLPI_URL = (process.env.GLPI_URL || '').replace(/\/$/, '');
const GLPI_APP_TOKEN = process.env.GLPI_APP_TOKEN;
const GLPI_USER_TOKEN = process.env.GLPI_USER_TOKEN;
const CHROMIUM_PATH = process.env.CHROMIUM_PATH || '/usr/bin/chromium';
const TICKET_PREFIX = process.env.TICKET_PREFIX || 'WhatsApp';

if (!GLPI_URL || !GLPI_APP_TOKEN || !GLPI_USER_TOKEN) {
  console.error('Faltam variáveis no arquivo .env');
  process.exit(1);
}

const TI_EMAIL = 'tecnologia@grupollal.com.br';
const SESSION_TIMEOUT_MS = 15 * 60 * 1000;
const MIN_MESSAGE_INTERVAL_MS = 1200;
const MAX_DESCRIPTION_LENGTH = 1800;
const MAX_ATTACHMENTS = 5;

const ETAPAS = {
  MENU_PRINCIPAL: 'menu_principal',
  AGUARDANDO_CATEGORIA: 'aguardando_categoria',
  AGUARDANDO_DESCRICAO: 'aguardando_descricao',
  AGUARDANDO_CONSULTA: 'aguardando_consulta'
};

const conversas = new Map();
const ultimoProcessamento = new Map();

const MSG_MENU_PRINCIPAL =
`👋 *Olá! Sou o assistente de chamados da TI.*

Escolha uma opção:
*1* - 📝 Abrir chamado
*2* - 🔎 Consultar chamado
*3* - 👨‍💻 Falar com TI

Comandos:
*0* - ↩️ Voltar
*#* - 🏠 Menu inicial`;

const MSG_MENU_CATEGORIA =
`🗂️ *Escolha a categoria do chamado:*

*1* - 🌐 Rede / Internet
*2* - 💻 Computador / Notebook
*3* - 🖨️ Impressora
*4* - 🔐 Acesso / Senha
*5* - 🧩 Sistema / ERP
*6* - 📦 Outro

Comandos:
*0* - ↩️ Voltar
*#* - 🏠 Menu inicial`;

const MSG_PEDIR_DESCRICAO =
`✍️ *Agora descreva o problema com detalhes.*

Você também pode enviar:
📷 imagem
📄 PDF
📝 texto + imagem/PDF na mesma mensagem

🚫 Vídeos não são aceitos.

Comandos:
*0* - ↩️ Voltar
*#* - 🏠 Menu inicial`;

const MSG_PEDIR_NUMERO_CHAMADO =
`🔎 *Informe o número do chamado.*

Exemplo:
*154*

Comandos:
*0* - ↩️ Voltar
*#* - 🏠 Menu inicial`;

function textoInvalido(menuTexto) {
  return `❌ *Opção inválida.*\n\n${menuTexto}`;
}

function normalizarTexto(texto) {
  return (texto || '').trim().replace(/\s+/g, ' ');
}

function isMenuCommand(texto) {
  const t = normalizarTexto(texto).toLowerCase();
  return t === '#' || t === 'menu' || t === 'inicio' || t === 'início' || t === 'oi' || t === 'olá' || t === 'ola';
}

function criarEstadoInicial() {
  return {
    etapa: ETAPAS.MENU_PRINCIPAL,
    historico: [],
    updatedAt: Date.now(),
    categoria: null,
    anexos: []
  };
}

function atualizarEstado(numero, novoEstado) {
  conversas.set(numero, {
    ...novoEstado,
    updatedAt: Date.now()
  });
}

function expirarSeNecessario(numero) {
  const estado = conversas.get(numero);
  if (!estado) return null;

  if (Date.now() - (estado.updatedAt || 0) > SESSION_TIMEOUT_MS) {
    conversas.delete(numero);
    return null;
  }

  return estado;
}

function pushHistorico(estado, etapaAtual) {
  const historico = Array.isArray(estado.historico) ? [...estado.historico] : [];
  historico.push(etapaAtual);
  return historico;
}

function getCategoria(opcao) {
  const mapa = {
    '1': { nome: '🌐 Rede / Internet' },
    '2': { nome: '💻 Computador / Notebook' },
    '3': { nome: '🖨️ Impressora' },
    '4': { nome: '🔐 Acesso / Senha' },
    '5': { nome: '🧩 Sistema / ERP' },
    '6': { nome: '📦 Outro' }
  };

  return mapa[opcao] || null;
}

function voltarEstado(numero) {
  const estado = conversas.get(numero);

  if (!estado) {
    const inicial = criarEstadoInicial();
    atualizarEstado(numero, inicial);
    return { mensagem: MSG_MENU_PRINCIPAL };
  }

  const historico = Array.isArray(estado.historico) ? [...estado.historico] : [];
  const etapaAnterior = historico.pop();

  if (!etapaAnterior) {
    const inicial = criarEstadoInicial();
    atualizarEstado(numero, inicial);
    return { mensagem: `↩️ Você já está no início.\n\n${MSG_MENU_PRINCIPAL}` };
  }

  let novoEstado = { ...estado, historico };

  if (etapaAnterior === ETAPAS.MENU_PRINCIPAL) {
    novoEstado = criarEstadoInicial();
    atualizarEstado(numero, novoEstado);
    return { mensagem: MSG_MENU_PRINCIPAL };
  }

  if (etapaAnterior === ETAPAS.AGUARDANDO_CATEGORIA) {
    novoEstado.etapa = ETAPAS.AGUARDANDO_CATEGORIA;
    novoEstado.categoria = null;
    atualizarEstado(numero, novoEstado);
    return { mensagem: MSG_MENU_CATEGORIA };
  }

  if (etapaAnterior === ETAPAS.AGUARDANDO_DESCRICAO) {
    novoEstado.etapa = ETAPAS.AGUARDANDO_DESCRICAO;
    atualizarEstado(numero, novoEstado);
    return { mensagem: MSG_PEDIR_DESCRICAO };
  }

  if (etapaAnterior === ETAPAS.AGUARDANDO_CONSULTA) {
    novoEstado.etapa = ETAPAS.AGUARDANDO_CONSULTA;
    atualizarEstado(numero, novoEstado);
    return { mensagem: MSG_PEDIR_NUMERO_CHAMADO };
  }

  novoEstado = criarEstadoInicial();
  atualizarEstado(numero, novoEstado);
  return { mensagem: MSG_MENU_PRINCIPAL };
}

function statusTicketLabel(status) {
  const mapa = {
    1: 'Novo',
    2: 'Em atendimento',
    3: 'Planejado',
    4: 'Pendente',
    5: 'Solucionado',
    6: 'Fechado'
  };
  return mapa[status] || `Status ${status}`;
}

function statusUsuarioFinal(status) {
  if (status === 5 || status === 6) return 'Finalizado';
  return 'Em andamento';
}

function formatDateBr(value) {
  if (!value) return 'Não informado';
  const d = new Date(value);
  if (isNaN(d.getTime())) return String(value);
  return d.toLocaleString('pt-BR');
}

function stripHtml(html) {
  if (!html) return '';
  return String(html)
    .replace(/<br\s*\/?>/gi, '\n')
    .replace(/<\/p>/gi, '\n')
    .replace(/<[^>]*>/g, '')
    .replace(/&nbsp;/g, ' ')
    .replace(/&amp;/g, '&')
    .replace(/&lt;/g, '<')
    .replace(/&gt;/g, '>')
    .trim();
}

function extractTicketId(texto) {
  const m = normalizarTexto(texto).match(/\d+/);
  return m ? parseInt(m[0], 10) : null;
}

function sanitizeFilename(name) {
  return (name || 'arquivo')
    .replace(/[^\w.\-]/g, '_')
    .slice(0, 120);
}

function getMediaTypeInfo(mimetype) {
  const mt = (mimetype || '').toLowerCase();

  if (mt === 'application/pdf') {
    return { accepted: true, type: 'pdf', ext: '.pdf' };
  }

  if (mt === 'image/jpeg' || mt === 'image/jpg') {
    return { accepted: true, type: 'image', ext: '.jpg' };
  }

  if (mt === 'image/png') {
    return { accepted: true, type: 'image', ext: '.png' };
  }

  if (mt === 'image/webp') {
    return { accepted: true, type: 'image', ext: '.webp' };
  }

  if (mt.startsWith('video/')) {
    return { accepted: false, type: 'video', ext: '' };
  }

  return { accepted: false, type: 'other', ext: '' };
}

async function initGlpiSession() {
  const response = await axios.get(`${GLPI_URL}/apirest.php/initSession`, {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `user_token ${GLPI_USER_TOKEN}`,
      'App-Token': GLPI_APP_TOKEN
    },
    timeout: 15000
  });

  return response.data.session_token;
}

async function killGlpiSession(sessionToken) {
  try {
    await axios.get(`${GLPI_URL}/apirest.php/killSession`, {
      headers: {
        'Session-Token': sessionToken,
        'App-Token': GLPI_APP_TOKEN
      },
      timeout: 10000
    });
  } catch (_) {}
}

async function criarTicketGLPI({ numero, nome, categoria, descricao, anexos }) {
  const sessionToken = await initGlpiSession();

  try {
    const qtdAnexos = Array.isArray(anexos) ? anexos.length : 0;
    const linhaAnexos = qtdAnexos > 0
      ? `\n\n📎 Anexos enviados pelo WhatsApp: ${qtdAnexos}`
      : '';

    const payload = {
      input: {
        name: `${TICKET_PREFIX} - ${categoria.nome} - ${numero}`,
        content:
`📲 Origem: WhatsApp
👤 Nome: ${nome || 'Não informado'}
📞 Número: ${numero}
🗂️ Categoria: ${categoria.nome}

📝 Descrição:
${descricao}${linhaAnexos}`,
        urgency: 3,
        impact: 3,
        priority: 3
      }
    };

    const response = await axios.post(`${GLPI_URL}/apirest.php/Ticket/`, payload, {
      headers: {
        'Content-Type': 'application/json',
        'Session-Token': sessionToken,
        'App-Token': GLPI_APP_TOKEN
      },
      timeout: 20000
    });

    return response.data.id;
  } finally {
    await killGlpiSession(sessionToken);
  }
}

async function consultarTicketResumo(ticketId) {
  const sessionToken = await initGlpiSession();

  try {
    const ticketRes = await axios.get(`${GLPI_URL}/apirest.php/Ticket/${ticketId}`, {
      headers: {
        'Session-Token': sessionToken,
        'App-Token': GLPI_APP_TOKEN
      },
      timeout: 15000
    });

    const ticket = ticketRes.data;

    let followups = [];
    try {
      const followRes = await axios.get(`${GLPI_URL}/apirest.php/Ticket/${ticketId}/ITILFollowup`, {
        headers: {
          'Session-Token': sessionToken,
          'App-Token': GLPI_APP_TOKEN
        },
        timeout: 15000
      });
      followups = Array.isArray(followRes.data) ? followRes.data : [];
    } catch (_) {
      followups = [];
    }

    let ultimaMensagem = '';

    if (followups.length > 0) {
      const ultima = followups[followups.length - 1];
      ultimaMensagem = stripHtml(ultima.content || ultima.name || '');
    }

    if (!ultimaMensagem && ticket.content) {
      ultimaMensagem = stripHtml(ticket.content);
    }

    return {
      id: ticket.id,
      status: ticket.status,
      statusLabel: statusTicketLabel(ticket.status),
      statusUsuario: statusUsuarioFinal(ticket.status),
      date_mod: ticket.date_mod,
      mensagemUsuario: ultimaMensagem || 'Ainda não há retorno registrado pela TI.'
    };
  } finally {
    await killGlpiSession(sessionToken);
  }
}

async function receberAnexo(msg, estado) {
  if (!msg.hasMedia) return { saved: false };

  if (!estado || estado.etapa !== ETAPAS.AGUARDANDO_DESCRICAO) {
    return {
      saved: false,
      warning: '📎 Recebi um arquivo, mas primeiro escolha *1 - Abrir chamado* no menu.'
    };
  }

  if ((estado.anexos || []).length >= MAX_ATTACHMENTS) {
    return {
      saved: false,
      warning: `⚠️ Limite de anexos atingido. Máximo: ${MAX_ATTACHMENTS} arquivos.`
    };
  }

  const media = await msg.downloadMedia();
  if (!media) {
    return {
      saved: false,
      warning: '⚠️ Não consegui baixar o arquivo. Tente enviar novamente.'
    };
  }

  const mediaInfo = getMediaTypeInfo(media.mimetype);

  if (!mediaInfo.accepted) {
    if (mediaInfo.type === 'video') {
      return {
        saved: false,
        warning: '🚫 *Vídeos não são aceitos.*\n\nEnvie apenas *imagens* ou *PDF*.'
      };
    }

    return {
      saved: false,
      warning: '⚠️ Tipo de arquivo não aceito.\n\nEnvie apenas *imagens* ou *PDF*.'
    };
  }

  const file = {
    filename: sanitizeFilename(media.filename || `anexo_${Date.now()}${mediaInfo.ext}`),
    mimetype: media.mimetype,
    data: media.data
  };

  estado.anexos = estado.anexos || [];
  estado.anexos.push(file);

  return {
    saved: true,
    count: estado.anexos.length,
    file
  };
}

const client = new Client({
  authStrategy: new LocalAuth({ clientId: 'glpi-whatsapp-bot' }),
  puppeteer: {
    headless: true,
    executablePath: CHROMIUM_PATH,
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-gpu'
    ]
  },
  qrMaxRetries: 5
});

client.on('qr', (qr) => {
  console.log('\n=== ESCANEIE O QR CODE NO WHATSAPP ===\n');
  qrcode.generate(qr, { small: true });
});

client.on('authenticated', () => {
  console.log('WhatsApp autenticado com sucesso.');
});

client.on('ready', () => {
  console.log('Bot do WhatsApp pronto.');
});

client.on('auth_failure', (msg) => {
  console.error('Falha na autenticação:', msg);
});

client.on('disconnected', (reason) => {
  console.error('WhatsApp desconectado:', reason);
});

client.on('message', async (msg) => {
  try {
    if (msg.fromMe) return;
    if (msg.from.includes('@g.us')) return;

    const agora = Date.now();
    const ultimo = ultimoProcessamento.get(msg.from) || 0;
    if (agora - ultimo < MIN_MESSAGE_INTERVAL_MS) return;
    ultimoProcessamento.set(msg.from, agora);

    const numero = msg.from.replace('@c.us', '');
    const contato = await msg.getContact();
    const nome = contato.pushname || contato.name || 'Sem nome';
    const texto = normalizarTexto(msg.body || '');

    let estado = expirarSeNecessario(numero);

    if (isMenuCommand(texto)) {
      estado = criarEstadoInicial();
      atualizarEstado(numero, estado);
      await msg.reply(MSG_MENU_PRINCIPAL);
      return;
    }

    if (texto === '0') {
      const retorno = voltarEstado(numero);
      await msg.reply(retorno.mensagem);
      return;
    }

    if (!estado) {
      estado = criarEstadoInicial();
      atualizarEstado(numero, estado);
      await msg.reply(MSG_MENU_PRINCIPAL);
      return;
    }

    if (estado.etapa === ETAPAS.MENU_PRINCIPAL) {
      if (texto === '1') {
        atualizarEstado(numero, {
          ...estado,
          etapa: ETAPAS.AGUARDANDO_CATEGORIA,
          historico: pushHistorico(estado, ETAPAS.MENU_PRINCIPAL),
          anexos: []
        });
        await msg.reply(MSG_MENU_CATEGORIA);
        return;
      }

      if (texto === '2') {
        atualizarEstado(numero, {
          ...estado,
          etapa: ETAPAS.AGUARDANDO_CONSULTA,
          historico: pushHistorico(estado, ETAPAS.MENU_PRINCIPAL)
        });
        await msg.reply(MSG_PEDIR_NUMERO_CHAMADO);
        return;
      }

      if (texto === '3') {
        await msg.reply(
`👨‍💻 *Falar com TI*

Você pode entrar em contato pelo e-mail:
📧 *${TI_EMAIL}*

Se quiser voltar, envie *#*.`
        );
        return;
      }

      await msg.reply(textoInvalido(MSG_MENU_PRINCIPAL));
      return;
    }

    if (estado.etapa === ETAPAS.AGUARDANDO_CATEGORIA) {
      const categoria = getCategoria(texto);
      if (!categoria) {
        await msg.reply(textoInvalido(MSG_MENU_CATEGORIA));
        return;
      }

      atualizarEstado(numero, {
        ...estado,
        etapa: ETAPAS.AGUARDANDO_DESCRICAO,
        historico: pushHistorico(estado, ETAPAS.AGUARDANDO_CATEGORIA),
        categoria,
        anexos: estado.anexos || []
      });

      await msg.reply(`✅ Categoria selecionada: ${categoria.nome}\n\n${MSG_PEDIR_DESCRICAO}`);
      return;
    }

    if (estado.etapa === ETAPAS.AGUARDANDO_DESCRICAO) {
      const mediaResult = await receberAnexo(msg, estado);

      if (mediaResult.warning) {
        await msg.reply(mediaResult.warning);
        return;
      }

      atualizarEstado(numero, estado);

      if (msg.hasMedia && !texto) {
        await msg.reply(`📎 Arquivo recebido com sucesso.\nTotal de anexos: *${estado.anexos.length}*\n\nAgora envie a descrição do chamado.`);
        return;
      }

      if (!texto) {
        await msg.reply(MSG_PEDIR_DESCRICAO);
        return;
      }

      if (texto.length < 5) {
        await msg.reply(`❌ *Descrição muito curta.*\n\n${MSG_PEDIR_DESCRICAO}`);
        return;
      }

      const descricao = texto.slice(0, MAX_DESCRIPTION_LENGTH);

      const ticketId = await criarTicketGLPI({
        numero,
        nome,
        categoria: estado.categoria,
        descricao,
        anexos: estado.anexos
      });

      const anexosOk = Array.isArray(estado.anexos) ? estado.anexos.length : 0;
      const linhaAnexo = anexosOk > 0
        ? `📎 Arquivos recebidos: *${anexosOk}*\n`
        : '';

      await msg.reply(
`✅ *Chamado criado com sucesso!*

🎫 Número: *#${ticketId}*
🗂️ Categoria: ${estado.categoria.nome}
${linhaAnexo}Se quiser consultar depois, envie *#* e escolha *2 - Consultar chamado*.`
      );

      conversas.delete(numero);
      return;
    }

    if (estado.etapa === ETAPAS.AGUARDANDO_CONSULTA) {
      const ticketId = extractTicketId(texto);
      if (!ticketId) {
        await msg.reply(`❌ Número inválido.\n\n${MSG_PEDIR_NUMERO_CHAMADO}`);
        return;
      }

      const resumo = await consultarTicketResumo(ticketId);

      await msg.reply(
`🔎 *Chamado #${resumo.id}*

📌 Situação: *${resumo.statusUsuario}*
📍 Status interno: ${resumo.statusLabel}
🕒 Última atualização: ${formatDateBr(resumo.date_mod)}

💬 Retorno da TI:
${resumo.mensagemUsuario}

Se quiser voltar ao menu, envie *#*.`
      );

      conversas.delete(numero);
      return;
    }

    const inicial = criarEstadoInicial();
    atualizarEstado(numero, inicial);
    await msg.reply(MSG_MENU_PRINCIPAL);

  } catch (error) {
    console.error('Erro ao processar mensagem:', error.response?.data || error.message);

    try {
      await msg.reply(`⚠️ *Não consegui processar sua solicitação agora.*\n\nEnvie *#* para voltar ao menu inicial.`);
    } catch (_) {}
  }
});

client.initialize();

### 4. Validar sintaxe

Execute:

```bash
node --check /opt/wa-glpi-bot/index.js
```

---

## Execução do bot

### Iniciar manualmente

Execute:

```bash
cd /opt/wa-glpi-bot
node index.js
```

Saída esperada:

```text
Bot do WhatsApp pronto.
```

Se a sessão ainda não estiver autenticada, aparecerá o QR Code no terminal.

---

## Autenticação no WhatsApp

### Como ler o QR Code

No celular:

- abrir o WhatsApp
- acessar **Aparelhos conectados**
- clicar em **Conectar um aparelho**
- escanear o QR Code exibido no terminal do servidor

### Como forçar um novo QR Code

Execute:

```bash
pkill -f "node index.js"
pkill -f chromium
pkill -f chrome
rm -rf /opt/wa-glpi-bot/.wwebjs_auth
rm -rf /opt/wa-glpi-bot/.wwebjs_cache
cd /opt/wa-glpi-bot
node index.js
```

---

## Fluxo funcional para o usuário

### Menu principal

| Opção | Ação |
|---|---|
| 1 | Abrir chamado |
| 2 | Consultar chamado |
| 3 | Falar com TI |

### Abertura do chamado

O usuário pode enviar:

- texto
- imagem
- PDF
- texto + anexo na mesma interação

### Exemplo de fluxo

```text
oi
1
2
notebook não liga
```

Ou:

```text
oi
1
2
[envio de imagem com legenda]
```

Resultado esperado:

- o bot recebe a categoria
- recebe a descrição
- registra que houve anexo, quando existir
- cria o chamado no GLPI
- devolve o número do ticket ao usuário

### Consulta de chamado

```text
2
154
```

Retorno esperado:

- situação do chamado
- status interno
- data da última atualização
- último retorno da TI

---

## Comportamento atual dos anexos

### O que acontece nesta versão

**Registrado no GLPI:**

- ticket
- descrição textual
- categoria
- quantidade de anexos recebidos

**Ainda não enviado fisicamente ao ticket:**

- imagem
- PDF

> **Observação técnica**
>
> O bot recebe os anexos, mas o vínculo físico do documento ao ticket no GLPI foi removido nesta versão porque a associação via API não funcionou corretamente no ambiente implantado.

---

## Erros encontrados e correções

### `ERROR_WRONG_APP_TOKEN_PARAMETER`

**Causa provável:**

- App-Token incorreto
- token do usuário usado no lugar do App-Token

**Correção:**

```ini
GLPI_APP_TOKEN=TOKEN_DO_CLIENTE_API
GLPI_USER_TOKEN=TOKEN_DO_USUARIO_WABOT
```

### `Cannot find module '/home/glpi/index.js'`

**Causa provável:**

Execução do comando no diretório errado.

**Correção:**

```bash
cd /opt/wa-glpi-bot
node index.js
```

### `The browser is already running`

**Causa provável:**

Há processo antigo do Chromium ou Node ainda em execução.

**Correção:**

```bash
pkill -f "node index.js"
pkill -f chromium
pkill -f chrome
cd /opt/wa-glpi-bot
node index.js
```

### `QR Code não aparece`

**Correção recomendada:**

```bash
rm -rf /opt/wa-glpi-bot/.wwebjs_auth
rm -rf /opt/wa-glpi-bot/.wwebjs_cache
cd /opt/wa-glpi-bot
node index.js
```

### `Unexpected token` / arquivo JS quebrado

**Causa provável:**

- comandos bash colados dentro do arquivo JavaScript
- código JavaScript executado direto no shell

**Correção:**

- apagar o arquivo
- recriar no editor correto
- colar apenas o código JavaScript
- validar com `node --check`

---

## Comandos operacionais rápidos

### Iniciar o bot

```bash
cd /opt/wa-glpi-bot
node index.js
```

### Validar sintaxe do código

```bash
node --check /opt/wa-glpi-bot/index.js
```

### Resetar sessão do WhatsApp

```bash
pkill -f "node index.js"
pkill -f chromium
pkill -f chrome
rm -rf /opt/wa-glpi-bot/.wwebjs_auth
rm -rf /opt/wa-glpi-bot/.wwebjs_cache
cd /opt/wa-glpi-bot
node index.js
```

---

## Boas práticas operacionais

- manter backup do arquivo `.env`
- nunca expor tokens em prints ou documentação pública
- validar o código com `node --check` antes de executar
- manter o Chromium instalado e funcional
- documentar qualquer alteração de menu, categorias ou comportamento do bot

---

## Conclusão

A implantação documentada permite operar um fluxo básico e funcional de abertura e consulta de chamados via WhatsApp integrado ao GLPI, com autenticação por QR Code e uso da API REST do GLPI para criação e leitura de tickets.

### Evoluções futuras sugeridas

- anexar arquivos fisicamente ao ticket
- identificar usuário por base cadastral
- direcionar categoria automaticamente
- executar como serviço no sistema
- registrar logs estruturados
- integrar com banco de conhecimento
