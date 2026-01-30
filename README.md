# Deploy em Red Hat (RHEL) na AWS

Instruções focadas em AMIs baseadas em Red Hat (RHEL) — `ec2-user` é o usuário SSH padrão em muitas AMIs.

1) Acessar a instância via SSH:

```bash
ssh -i /path/minha-chave.pem ec2-user@<EC2_PUBLIC_IP>
```

2) Instalar Node.js (Node 18 via NodeSource):

```bash
curl -fsSL https://rpm.nodesource.com/setup_18.x | sudo bash -
sudo yum install -y nodejs
```

3) Clonar repositório e instalar dependências:

```bash
cd /home/ec2-user
git clone <REPO_URL> app
cd app
npm install
```

4) Instalar e iniciar Nginx:

```bash
sudo yum install -y nginx
sudo systemctl enable --now nginx
```

5) Abrir portas HTTP/HTTPS no firewall (firewalld):

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

6) SELinux — permitir que Nginx conecte ao backend:

```bash
sudo setsebool -P httpd_can_network_connect 1
```

7) Certbot (recomendado via `snap`) e emitir certificado:

```bash
sudo yum install -y snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
sudo snap install core && sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx -d example.com
```

8) Serviço systemd (exemplo já incluído em `deploy/simple-app-rhel.service`):

```bash
sudo cp deploy/simple-app-rhel.service /etc/systemd/system/simple-app.service
sudo systemctl daemon-reload
sudo systemctl enable --now simple-app
sudo systemctl status simple-app
```

9) AWS Security Group: confirme que as portas `22`, `80` e `443` estão liberadas para os IPs necessários.

Notas rápidas:
- Substitua `example.com` pelo seu domínio real antes de rodar o Certbot.
- Se não tiver domínio ainda, gere um certificado self-signed para testes e aponte `ssl_certificate`/`ssl_certificate_key` no arquivo `nginx/simple_site.conf`.
- `deploy/simple-app-rhel.service` assume que a aplicação ficará em `/home/ec2-user/app`.

## Executar frontend e backend

Backend (Node/Express)
```bash
# na raiz do projeto
npm install

# iniciar (se houver script "start" no package.json)
npm start

# ou alternativamente
node server.js
```

Frontend (arquivos estáticos em `public/`)
```bash
# opção 1: servido pelo backend (após iniciar o backend, abrir http://localhost:3000)
# opção 2: servir apenas o front em dev
npx http-server public -p 8080
# abrir http://localhost:8080
```

Produção (RHEL / AWS)
```bash
# iniciar via systemd (exemplo já incluído em deploy/simple-app-rhel.service)
sudo systemctl enable --now simple-app
sudo systemctl status simple-app

# ou usar PM2
sudo npm install -g pm2
pm2 start server.js --name simple-app
pm2 save
sudo pm2 startup systemd
```


