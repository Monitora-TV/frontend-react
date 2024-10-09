# POC de Autenticação Keycloak para o Monitora TV

Esta POC implanta uma aplicação ReactJS em uma instância AWS EC2 com configuração e setup para lançar a aplicação Monitora TV usando o Nginx e garantindo uma experiência de usuário fluida. 

Instruções para configurar o aplicativo web da POC em uma instância EC2 da AWS, desde o lançamento do servidor até a configuração do Nginx para servir sua aplicação. Neste guia, abordaremos cada passo para implantar uma aplicação ReactJS em um servidor Ubuntu da AWS EC2.


## Pré-requisitos

- Instalação do Keycloak: [ver repositório](https://github.com/Monitora-TV/keycloak-aws)

## Referências

- Deploy React App on AWS EC2 Ubuntu Server With Nginx, SSL And Domain Setup [Assista aqui](https://youtu.be/UK_OVKDRArs?si=G620QaGNjJOA0LGR)
- [react-router-dom + oidc-spa + Vite + Keycloak](https://docs.oidc-spa.dev/documentation/web-api)
- [Vite Insee Starter](https://github.com/InseeFrLab/vite-insee-starter)
- [Todo REST API](https://github.com/InseeFrLab/todo-rest-api)
- [Exemplo Tanstack Router](https://example-tanstack-router.oidc-spa.dev/)
- **Nota**: Se você está começando um novo projeto, considere usar `@tanstack/react-router` em vez de `react-router-dom`. Veja a [configuração de exemplo](https://github.com/keycloakify/oidc-spa/tree/main/examples/tanstack-router).

## 1 - Instalação da Infraestrutura

### Criar Instância EC2

O frontend da aplicação Monitora TV será implantado em uma instância separada da instância do Keycloak. Utilize a mesma VPC e Subnet pública da instância criada para o Keycloak. [ver repositório](https://github.com/Monitora-TV/keycloak-aws)

### Criar Grupo de Segurança

- **Regras de entrada**:
  - TCP 443 (0.0.0.0/0)
  - TCP 22 (0.0.0.0/0)
  - TCP 80 (0.0.0.0/0)

### Criar IP Elástico

- IP: `<IP elástico criado>`
- Alocar o IP elástico à instância criada (`monitoratv-public`).

### Route 53 - Criar Subdomínio

1. Acessar Route 53 > Hosted Zone > `<dominio>.com`
2. Criar registro `monitoratv.<dominio>.com` do tipo "A" e definir o valor da rota como o IP elástico `<IP elástico criado>`.

## 2 - Criar Chave SSH para Clonar o Repositório GitHub

1. O primeiro passo é criar um par de chaves na máquina cliente (geralmente seu computador):
   ```bash
   ssh-keygen
   ```
   
2. Pressione Enter para todas as opções e não adicione uma senha. Em seguida, obtenha a chave com este comando:
   ```bash
   cat ~/.ssh/id_rsa.pub
   ```
   ou
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

3. Adicione as chaves SSH no GitHub: [GitHub Settings](https://github.com/settings/keys)

## 3 - Clonar Repositório 

```bash
git clone https://github.com/Monitora-TV/frontend-react.git
cd frontend-react
yarn
yarn dev
```

## 4 - Atualizar Instância 

- Acesse a instância criada:
  ```bash
  ssh -i "<Utilizar a mesma chave do Keycloak>.pem" ubuntu@exxxx.amazonaws.com
  ```

- Atualizar todos os pacotes da instância:
  ```bash
  sudo apt-get update -y
  ```

- Instalar Node.js:
  ```bash
  sudo apt-get install -y curl
  curl -fsSL https://deb.nodesource.com/setup_22.x -o nodesource_setup.sh
  sudo -E bash nodesource_setup.sh
  sudo apt-get install -y nodejs
  node -v
  ```

- Verificar se o npm está instalado; caso contrário, instale:
  ```bash
  npm --version
  sudo apt install npm -y
  ```

- Instalar Nginx:
  ```bash
  sudo apt update 
  sudo apt install nginx 
  systemctl status nginx
  ```

- Testar se o Nginx está rodando:
  ```
  http://<IP_PÚBLICO_DA_INSTÂNCIA>
  ```

- Clonar o projeto:
  ```bash
  sudo git clone https://github.com/Monitora-TV/frontend-react.git
  cd frontend-react
  npm install
  npm run build
  ```

- Criar diretório para a aplicação:
  ```bash
  sudo mkdir -p /var/www/vhosts/frontend/
  sudo cp -R dist/ /var/www/vhosts/frontend/
  ```

## 5 - Configurar Nginx

- Remover arquivo de configuração padrão do Nginx:
  ```bash
  cd /etc/nginx/sites-enabled/
  sudo rm -rf default
  ```

- Criar um arquivo de configuração para o Nginx:
  ```bash
  sudo vim /etc/nginx/sites-available/react
  ```

- Cole a seguinte configuração dentro do arquivo criado:

  ```nginx
  server {
      listen 80 default_server;
      server_name _;

      location / {
          autoindex on;
          root /var/www/vhosts/frontend/dist;
          try_files $uri /index.html;
      }
  }
  ```

- Ative a configuração:
  ```bash
  sudo ln -s /etc/nginx/sites-available/react /etc/nginx/sites-enabled/
  ```

- Adicione o usuário `www-data` ao grupo `ubuntu`:
  ```bash
  sudo gpasswd -a www-data ubuntu
  ```

- Reinicie o Nginx:
  ```bash
  sudo systemctl restart nginx
  sudo service nginx restart
  ```

- Testar:
  ```
  http://<IP_PÚBLICO_DA_INSTÂNCIA>
  ```

## 6 - Instalar Certbot para SSL 

```bash
sudo apt-get install certbot python3-certbot-nginx
sudo certbot --nginx -d monitoratv.gustavokanashiro.com
sudo systemctl reload nginx
```

- Após criar o certificado com o certbot, verifique o arquivo de configuração Nginx:
  ```bash
  sudo vim /etc/nginx/sites-available/react
  ```

### Configuração Nginx

```nginx
server {
    listen 80 default_server;
    server_name _;

    location / {
        autoindex on;
        root /var/www/vhosts/frontend/dist;
        try_files $uri /index.html;
    }
}

server {
    server_name monitoratv.gustavokanashiro.com; # managed by Certbot

    location / {
        autoindex on;
        root /var/www/vhosts/frontend/dist;
        try_files $uri /index.html;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/monitoratv.gustavokanashiro.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/monitoratv.gustavokanashiro.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = monitoratv.gustavokanashiro.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 81 ;
    server_name monitoratv.gustavokanashiro.com;
    return 404; # managed by Certbot
}
```

## 7 - Testar Acesso

- Acesse:
  - [https://monitoratv.gustavokanashiro.com/](https://monitoratv.gustavokanashiro.com/)
  User: usr_admin
  Password: 102030  

Caso não tenha criado anteriormente um usuário, acesso o keycloak e fala o cadastro de um usuário no realm "Monitora TV"
  - [https://keycloak.gustavokanashiro.com/auth/](https://keycloak.gustavokanashiro.com/auth/)
  User: admin
  Password: admin@102030  

## Exemplo de Configuração

Esta configuração de exemplo está implantada aqui:  
[https://monitoratv.gustavokanashiro.com/](https://monitoratv.gustavokanashiro.com/)

### Vídeo Tutorial Completo no YouTube

[Assista aqui](https://youtu.be/UK_OVKDRArs?si=G620QaGNjJOA0LGR)
