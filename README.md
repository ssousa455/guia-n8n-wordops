## **Guia Definitivo: Instale n8n com PM2 em uma VPS com WordOps – A Alternativa Leve e Poderosa ao Docker**

### **Introdução: A Peça que Faltava no Quebra-Cabeça**

Você é como eu? Usa e confia no **WordOps** para criar sites WordPress de alta performance, otimizando cada recurso da sua VPS Contabo para aguentar muito tráfego. Você domina seu ambiente. Mas quando decide adicionar a automação incrível do **n8n** ao seu arsenal, você esbarra em um muro: quase todos os tutoriais te empurram para o Docker, Portainer, e um universo de contêineres.

Isso significa mais consumo de RAM, mais espaço em disco, e uma camada de complexidade que, muitas vezes, não é necessária para quem já gerencia uma VPS enxuta. A pergunta que fica é: *"Não existe uma forma de instalar o n8n diretamente, tratando-o como um cidadão de primeira classe no meu servidor, lado a lado com meus sites?"*

A resposta é **sim**. E este guia é a prova.

Construído a partir de uma instalação real, enfrentando e resolvendo problemas que outros tutoriais ignoram (como erros de *secure cookie*, caminhos de certificado incorretos e sintaxe de comandos), este é o passo a passo definitivo para instalar o n8n diretamente no seu servidor com Node.js e PM2, de forma otimizada e sem conflitos.

### **✨ Por que Este Método é um divisor de águas?**

Fugir do padrão Docker-para-tudo não é ser contra a tecnologia, mas sim escolher a ferramenta certa para o trabalho. Para um ambiente WordOps, esta abordagem oferece vantagens claras:

*   ⚡ **Performance Nativa:** Sem a camada de virtualização do Docker, o n8n roda diretamente no "metal" do seu servidor. O resultado é um consumo de CPU menor e tempos de resposta mais rápidos.
*   💾 **Uso Mínimo de Recursos:** Espere uma redução de até **70% no consumo de RAM** e **50% no uso de disco** em comparação com um stack Docker + Portainer. Em uma VPS, cada megabyte conta.
*   🚀 **Simplicidade e Controle:** Você gerencia tudo com ferramentas que provavelmente já conhece: `npm` para pacotes, `pm2` para o processo e `nano` para editar arquivos. Sem volumes, redes ou orquestradores complexos.
*   🤝 **Integração Perfeita com WordOps:** O Nginx do WordOps se torna o maestro de todo o tráfego, gerenciando tanto seus sites quanto o acesso ao n8n a partir de uma única porta (443/HTTPS), de forma centralizada e segura.

---

### **📋 O Que Você Vai Precisar**

1.  Uma VPS (este guia foi validado em uma Contabo) com Ubuntu.
2.  WordOps já instalado.
3.  Um domínio ou subdomínio para apontar para o n8n.
4.  Acesso root ao seu servidor via SSH.

---

## 🛠️ PARTE 1: Instalação e Configuração Inicial do N8N

### Passo 1: Instalar Node.js com NVM (Método Recomendado)

```bash
# Conecte-se à sua VPS
ssh root@seu-ip-contabo

# Baixe e execute o script de instalação do NVM (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Carregue o NVM no seu terminal (ou feche e abra o terminal)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

# Instale a versão LTS (Long Term Support) mais recente do Node.js
nvm install --lts
```

### Passo 2: Instalar N8N e PM2 Globalmente

```bash
npm install -g n8n pm2
```

### Passo 3: Criar Estrutura e Configuração Inicial do PM2

```bash
mkdir /home/n8n
cd /home/n8n
nano ecosystem.config.js
```

**Copie e cole o seguinte conteúdo no arquivo:** Esta configuração inicial já inclui a solução para o comum erro de `Secure Cookie`.

```javascript
// /home/n8n/ecosystem.config.js
module.exports = {
  apps: [{
    name: 'n8n',
    script: 'n8n',
    cwd: '/home/n8n',
    env: {
      N8N_HOST: '0.0.0.0',
      N8N_PORT: 5678,
      N8N_PROTOCOL: 'http',
      WEBHOOK_URL: 'http://SEU-IP-DA-VPS:5678',
      // PONTO CRÍTICO: Desativa a exigência de HTTPS temporariamente
      N8N_SECURE_COOKIE: 'false',
      // Segurança básica (troque a senha!)
      N8N_BASIC_AUTH_ACTIVE: 'true',
      N8N_BASIC_AUTH_USER: 'seu_usuario_admin',
      N8N_BASIC_AUTH_PASSWORD: 'sua_senha_muito_forte'
    }
  }]
}
```
**Lembre-se de trocar `SEU-IP-DA-VPS`, `seu_usuario_admin` e `sua_senha_muito_forte`.** Salve e saia (`Ctrl + X`, `Y`, `Enter`).

> **Ponto Crítico:** O próximo passo resolve o erro `"Your n8n server is configured to use a secure cookie, however you are either visiting this via an insecure URL, or using Safari"` ao tentar acessar `http://SEU-IP-DA-VPS:5678`
### Passo 4: Iniciar N8N e Liberar o Firewall

```bash
# Parar o N8N
pm2 delete n8n

# Exportar a variável antes de iniciar
export N8N_SECURE_COOKIE=false

# Iniciar novamente com PM2
cd /home/n8n
pm2 start ecosystem.config.js"

# Inicie o n8n com PM2
pm2 start ecosystem.config.js

# Salve a lista de processos do PM2
pm2 save

# Libere a porta no firewall
ufw allow 5678
```

### Passo 5: Teste de Acesso e Criação da Conta

Abra seu navegador e acesse `http://SEU-IP-DA-VPS:5678`. Você deverá ver a tela de login básico e, em seguida, a tela para criar sua conta de administrador.



Se chegou até aqui, parabéns! Agora vamos profissionalizar o acesso.

---

## 🔒 PARTE 2: Configurando o Acesso com Domínio e SSL

### Passo 1: Apontar seu Subdomínio (DNS)

Vá até o painel do seu provedor de DNS e crie um **Registro A**:
*   **Tipo:** `A`
*   **Nome/Host:** `n8n` (ou o subdomínio que preferir)
*   **Valor/Endereço:** O IP da sua VPS
*   **TTL:** 1 hora ou 3600

### Passo 2: Criar o Site com WordOps

> **Ponto Crítico:** Tutoriais antigos usam `--wp=off`, que não é mais necessário. O comando correto é mais simples.

```bash
# Troque "n8n.seudominio.com" pelo seu subdomínio real
wo site create n8n.seudominio.com --letsencrypt
```

### Passo 3: Configurar o Proxy Reverso no Nginx

> **Ponto Crítico:** Muitos tutoriais erram o caminho do arquivo de configuração. O correto no WordOps é `/etc/nginx/sites-available/`.

```bash
# Abra o arquivo de configuração do site (troque pelo seu domínio)
nano /etc/nginx/sites-available/n8n.seudominio.com
```

**Substitua TODO o conteúdo do arquivo** pelo código de proxy reverso abaixo.

```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name n8n.seudominio.com;

    # PONTO CRÍTICO: Caminhos corretos para os certificados SSL
    ssl_certificate /etc/letsencrypt/live/n8n.seudominio.com/fullchain.pem;
    # ATENÇÃO: O nome da chave pode ser 'key.pem' ou 'privkey.pem'.
    # Verifique o nome real com: ls -l /etc/letsencrypt/live/n8n.seudominio.com/
    ssl_certificate_key /etc/letsencrypt/live/n8n.seudominio.com/key.pem;

    # Redireciona HTTP para HTTPS
    if ($scheme = http) {
        return 301 https://$server_name$request_uri;
    }

    # Bloco do Proxy Reverso
    location / {
        proxy_pass http://127.0.0.1:5678; # Comunicação interna
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Passo 4: Testar e Recarregar o Nginx

```bash
# Testa a sintaxe e, se OK, recarrega o Nginx
nginx -t && systemctl reload nginx
```
Se o teste for bem-sucedido (`test is successful`), o Nginx está pronto!

---

## ⚡ PARTE 3: Otimização Final para Produção

### Passo 1: Aplicar a Configuração PM2 Otimizada

```bash
nano /home/n8n/ecosystem.config.js
```

**Substitua TODO o conteúdo** pela versão otimizada abaixo, adaptando seu domínio e senha.

```javascript
// /home/n8n/ecosystem.config.js - VERSÃO OTIMIZADA
module.exports = {
  apps: [{
    name: 'n8n',
    script: 'n8n',
    cwd: '/home/n8n',
    instances: 1,
    exec_mode: 'fork',
    node_args: ['--max-old-space-size=2048', '--optimize-for-size'],
    max_memory_restart: '1G',
    env: {
      N8N_HOST: '127.0.0.1',
      N8N_PORT: 5678,
      N8N_PROTOCOL: 'https',
      WEBHOOK_URL: 'https://n8n.seudominio.com',
      N8N_EDITOR_BASE_URL: 'https://n8n.seudominio.com',
      N8N_BASIC_AUTH_ACTIVE: 'true',
      N8N_BASIC_AUTH_USER: 'seu_usuario_admin',
      N8N_BASIC_AUTH_PASSWORD: 'sua_senha_muito_forte',
      NODE_ENV: 'production',
      N8N_LOG_LEVEL: 'warn',
      DB_TYPE: 'sqlite',
      DB_SQLITE_ENABLE_WAL: 'true'
    },
    error_file: '/home/n8n/logs/error.log',
    out_file: '/home/n8n/logs/out.log',
    time: true
  }]
}
```

### Passo 2: Criar Diretório de Logs e Aplicar Mudanças

```bash
# Crie a pasta para os logs otimizados
mkdir -p /home/n8n/logs

# Reinicie o n8n para aplicar as novas configurações de performance
pm2 restart n8n
```

### Passo 3: Sucesso!

Acesse seu domínio **`https://n8n.seudominio.com`** e veja sua instalação profissional funcionando!

---

## 📋 PARTE 4: Comandos Essenciais de Manutenção

```bash
# Ver o status de todos os processos
pm2 status

# Ver os logs do n8n em tempo real
pm2 logs n8n

# Reiniciar o n8n
pm2 restart n8n

# Fazer backup do banco de dados (faça isso regularmente!)
cp /home/n8n/.n8n/database.sqlite /home/n8n/backup-$(date +%Y%m%d).sqlite

# Limpar logs com mais de 7 dias
find /home/n8n/logs -name "*.log" -mtime +7 -delete
```

---

### **Conclusão**

Pronto! Você não apenas instalou uma ferramenta poderosa, mas dominou uma nova forma de integrar serviços em seu servidor, fugindo do "padrão Docker" quando a situação pede uma solução mais enxuta, performática e integrada.

⭐ Se este guia te ajudou, considere dar uma estrela no repositório no GitHub!
💬 Gostou do conteúdo? Conecte-se comigo no LinkedIn e vamos trocar mais ideias sobre performance e automação! Compartilhe este post para ajudar outros desenvolvedores a economizarem recursos e dores de cabeça.
