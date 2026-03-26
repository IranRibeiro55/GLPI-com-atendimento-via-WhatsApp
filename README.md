Implantação do GLPI com Integração de Atendimento via WhatsApp

Documentação da implantação de um ambiente GLPI + Bot de WhatsApp, permitindo abertura e consulta de chamados por WhatsApp com integração via API REST do GLPI.

Objetivo

Esta solução foi implantada para permitir que usuários:

abram chamados de TI pelo WhatsApp
escolham uma categoria pelo menu
enviem descrição textual
enviem imagem
enviem PDF
enviem texto junto com imagem/PDF
consultem um chamado pelo número
recebam o status do chamado
visualizem o último retorno registrado pela TI
entrem em contato com a equipe de TI por e-mail
Arquitetura da solução
Usuário -> WhatsApp -> Bot Node.js -> API REST do GLPI -> Ticket
Componentes utilizados
Componente	Tecnologia / Ferramenta
Sistema operacional	Debian 12
Servidor web	Apache2
Banco de dados	MariaDB
Plataforma de chamados	GLPI
Runtime do bot	Node.js
Navegador para sessão do WhatsApp	Chromium
Biblioteca de integração WhatsApp	whatsapp-web.js
Variáveis de ambiente	dotenv
Cliente HTTP	axios
Geração de QR Code no terminal	qrcode-terminal
Pré-requisitos

Antes de iniciar, garantir que o servidor possua:

acesso à internet
privilégios administrativos
IP acessível na rede local
portas necessárias liberadas
ambiente Debian 12 atualizado

Nota

Os exemplos abaixo consideram que o GLPI será publicado localmente por IP e que o bot será executado no mesmo servidor ou em servidor com acesso HTTP ao GLPI.

Instalação do GLPI
1. Atualização do sistema
sudo apt update && sudo apt full-upgrade -y
2. Instalação dos pacotes necessários
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
3. Habilitar os serviços
sudo systemctl enable --now apache2
sudo systemctl enable --now mariadb
4. Segurança inicial do banco
sudo mysql_secure_installation

Aplicar as opções recomendadas:

definir senha para o usuário root do banco
remover usuários anônimos
desabilitar login root remoto
remover base de testes
recarregar privilégios
5. Criação do banco do GLPI

Entrar no MariaDB:

sudo mariadb

Executar:

CREATE DATABASE glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpi'@'localhost' IDENTIFIED BY 'SUA_SENHA_FORTE_AQUI';
GRANT ALL PRIVILEGES ON glpidb.* TO 'glpi'@'localhost';
FLUSH PRIVILEGES;
EXIT;
6. Download e extração do GLPI
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/11.0.6/glpi-11.0.6.tgz
sudo tar -xzf glpi-11.0.6.tgz -C /var/www/

Diretório final:

/var/www/glpi
7. Configuração do Apache

Criar o arquivo de virtual host:

sudo nano /etc/apache2/sites-available/glpi.conf

Conteúdo:

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

Ativar a configuração:

sudo a2enmod rewrite
sudo a2dissite 000-default.conf
sudo a2ensite glpi.conf
sudo systemctl reload apache2
8. Finalização pelo navegador

Acessar:

http://IP_DO_SERVIDOR

Seguir o assistente gráfico do GLPI e informar:

banco: glpidb
usuário: glpi
senha: a definida na criação do banco
Configuração da API do GLPI
1. Habilitar a API REST

No GLPI, acessar:

Configurar > Geral > API

Habilitar:

API REST
login com credenciais
login externo por token
2. Criar cliente da API

Ainda em:

Configurar > Geral > API

Criar um cliente com o nome:

bot-whatsapp

Depois salvar e copiar o App-Token.

Esse valor será usado na variável:

GLPI_APP_TOKEN
3. Criar usuário técnico do bot

No GLPI, acessar:

Administração > Usuários

Criar o usuário:

Campo	Valor
Usuário	wabot
Nome	Bot WhatsApp
Ativo	Sim
4. Gerar token do usuário

No cadastro do usuário wabot, acessar:

Senhas e chaves de acesso

Gerar ou regenerar o Token de API.

Esse valor será usado na variável:

GLPI_USER_TOKEN

Atenção

Não confundir:

GLPI_APP_TOKEN = token do cliente da API
GLPI_USER_TOKEN = token do usuário técnico
Testes iniciais da API
Teste de sessão
curl -s -X GET \
  -H "Content-Type: application/json" \
  -H "Authorization: user_token SEU_TOKEN_USUARIO" \
  -H "App-Token: SEU_APP_TOKEN" \
  "http://SEU_IP/apirest.php/initSession"

Saída esperada:

{"session_token":"TOKEN_AQUI"}
Teste de criação de ticket
curl -s -X POST \
  -H "Content-Type: application/json" \
  -H "Session-Token: SEU_SESSION_TOKEN" \
  -H "App-Token: SEU_APP_TOKEN" \
  -d '{"input":{"name":"Teste API GLPI","content":"Chamado criado via API","urgency":3,"impact":3,"priority":3}}' \
  "http://SEU_IP/apirest.php/Ticket/"
Instalação do bot do WhatsApp
1. Instalar Node.js
sudo apt update
sudo apt install -y curl ca-certificates gnupg
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

Validar:

node -v
npm -v
2. Instalar Chromium e dependências
sudo apt install -y chromium fonts-liberation libasound2 libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 libgbm1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 xdg-utils
3. Criar diretório do projeto
mkdir -p /opt/wa-glpi-bot
cd /opt/wa-glpi-bot
4. Inicializar projeto Node.js
npm init -y
npm install whatsapp-web.js qrcode-terminal axios dotenv
Configuração do arquivo .env

Criar o arquivo:

nano /opt/wa-glpi-bot/.env

Conteúdo:

GLPI_URL=http://192.168.1.81
GLPI_APP_TOKEN=SEU_APP_TOKEN
GLPI_USER_TOKEN=SEU_TOKEN_USUARIO
CHROMIUM_PATH=/usr/bin/chromium
TICKET_PREFIX=WhatsApp
Significado das variáveis
Variável	Função
GLPI_URL	Endereço base do GLPI
GLPI_APP_TOKEN	Token do cliente da API
GLPI_USER_TOKEN	Token do usuário técnico
CHROMIUM_PATH	Caminho do executável do Chromium
TICKET_PREFIX	Prefixo usado no título dos chamados
Criação do arquivo principal do bot
1. Remover arquivo antigo, se existir
rm -f /opt/wa-glpi-bot/index.js
2. Criar novo arquivo
nano /opt/wa-glpi-bot/index.js
3. Código-fonte completo

Coloque aqui o conteúdo completo do seu index.js.

require('dotenv').config();

// restante do código do bot aqui
4. Salvar e validar

Salvar no nano:

Ctrl + O
Enter
Ctrl + X

Testar sintaxe:

node --check /opt/wa-glpi-bot/index.js
Execução do bot
Iniciar manualmente
cd /opt/wa-glpi-bot
node index.js

Saída esperada:

Bot do WhatsApp pronto.

Se a sessão ainda não estiver autenticada, aparecerá o QR Code no terminal.

Autenticação no WhatsApp
Como ler o QR Code

No celular:

abrir o WhatsApp
acessar Aparelhos conectados
clicar em Conectar um aparelho
escanear o QR Code exibido no terminal do servidor
Como forçar um novo QR Code

Se o aparelho foi desconectado ou a sessão foi perdida:

pkill -f "node index.js"
pkill -f chromium
pkill -f chrome
rm -rf /opt/wa-glpi-bot/.wwebjs_auth
rm -rf /opt/wa-glpi-bot/.wwebjs_cache
cd /opt/wa-glpi-bot
node index.js
Fluxo funcional para o usuário
Menu principal
Opção	Ação
1	Abrir chamado
2	Consultar chamado
3	Falar com TI
Abertura do chamado

O usuário pode enviar:

texto
imagem
PDF
texto + anexo na mesma interação
Exemplo de fluxo
oi
1
2
notebook não liga

Ou:

oi
1
2
[envio de imagem com legenda]

Resultado esperado:

o bot recebe a categoria
recebe a descrição
registra que houve anexo, quando existir
cria o chamado no GLPI
devolve o número do ticket ao usuário
Consulta de chamado
2
154

Retorno esperado:

situação do chamado
status interno
data da última atualização
último retorno da TI
Comportamento atual dos anexos
O que acontece nesta versão

Registrado no GLPI:

ticket
descrição textual
categoria
quantidade de anexos recebidos

Ainda não enviado fisicamente ao ticket:

imagem
PDF

Observação técnica

O bot recebe os anexos, mas o vínculo físico do documento ao ticket no GLPI foi removido nesta versão porque a associação via API não funcionou corretamente no ambiente implantado.

Erros encontrados e correções
ERROR_WRONG_APP_TOKEN_PARAMETER

Causa provável:

App-Token incorreto
token do usuário usado no lugar do App-Token

Correção:

GLPI_APP_TOKEN=TOKEN_DO_CLIENTE_API
GLPI_USER_TOKEN=TOKEN_DO_USUARIO_WABOT
Cannot find module '/home/glpi/index.js'

Causa provável:

Execução do comando no diretório errado.

Correção:

cd /opt/wa-glpi-bot
node index.js
The browser is already running

Causa provável:

Há processo antigo do Chromium ou Node ainda em execução.

Correção:

pkill -f "node index.js"
pkill -f chromium
pkill -f chrome
cd /opt/wa-glpi-bot
node index.js
QR Code não aparece

Correção recomendada:

rm -rf /opt/wa-glpi-bot/.wwebjs_auth
rm -rf /opt/wa-glpi-bot/.wwebjs_cache
cd /opt/wa-glpi-bot
node index.js
Unexpected token / arquivo JS quebrado

Causa provável:

comandos bash colados dentro do arquivo JavaScript
código JavaScript executado direto no shell

Correção:

apagar o arquivo
recriar no editor correto
colar apenas o código JavaScript
validar com node --check
Comandos operacionais rápidos
Iniciar o bot
cd /opt/wa-glpi-bot
node index.js
Validar sintaxe do código
node --check /opt/wa-glpi-bot/index.js
Resetar sessão do WhatsApp
pkill -f "node index.js"
pkill -f chromium
pkill -f chrome
rm -rf /opt/wa-glpi-bot/.wwebjs_auth
rm -rf /opt/wa-glpi-bot/.wwebjs_cache
cd /opt/wa-glpi-bot
node index.js
Boas práticas operacionais
manter backup do arquivo .env
nunca expor tokens em prints ou documentação pública
validar o código com node --check antes de executar
manter o Chromium instalado e funcional
documentar qualquer alteração de menu, categorias ou comportamento do bot
Conclusão

A implantação documentada permite operar um fluxo básico e funcional de abertura e consulta de chamados via WhatsApp integrado ao GLPI, com autenticação por QR Code e uso da API REST do GLPI para criação e leitura de tickets.

Evoluções futuras sugeridas
anexar arquivos fisicamente ao ticket
identificar usuário por base cadastral
direcionar categoria automaticamente
executar como serviço no sistema
registrar logs estruturados
integrar com banco de conhecimento

O erro foi esse: você provavelmente colou o README inteiro dentro de um bloco de código.
No GitHub, só os trechos de comando/código devem ficar entre crases triplas.
