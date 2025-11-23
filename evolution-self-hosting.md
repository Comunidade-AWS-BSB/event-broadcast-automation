# Self-hosted Evolution API

We run our WhatsApp connector using the open-source [Evolution API](https://doc.evolution-api.com/v2/pt/get-started/introduction) so we keep full control of the data that flows between Cognito users and WhatsApp. The automation currently talks to an instance that we self-host on AWS.

## Infrastructure snapshot
- **EC2**: single t3.small in our AWS account (2 vCPU / 2 GiB) that keeps both n8n legacy flows and the latest Evolution API container online.
- **Docker & Docker Compose**: the host runs Docker Engine plus Compose so we can keep Evolution, n8n, and support services (e.g., Redis) in separate containers.
- **Nginx reverse proxy**: exposes friendly HTTPS endpoints:
  - `n8n.seudominio.com.br`
  - `evolution.seudominio.com.br`
- **TLS**: handled by Let's Encrypt (Certbot + Nginx plugin).

```nginx
# evolution.seudominio.com.br
server {
  listen 80;
  server_name evolution.seudominio.com.br;

  location / {
    include proxy_params;
    proxy_pass http://localhost:8081;
  }
}
```

### SSL bootstrap (Let's Encrypt)
```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx \
  -d n8n.seudominio.com.br \
  -d evolution.seudominio.com.br
sudo certbot renew --dry-run
```

### Docker Compose deployment
The source of truth for our container stack lives in the historical repository: [Comunidade-AWS-BSB/Solu-o-Auotma-o-para-Eventos](https://github.com/Comunidade-AWS-BSB/Solu-o-Auotma-o-para-Eventos).

```bash
git clone https://github.com/Comunidade-AWS-BSB/Solu-o-Auotma-o-para-Eventos.git
cd Solu-o-Auotma-o-para-Eventos
sudo docker compose up -d
```

Once the stack is up, open `https://evolution.seudominio.com.br` to pair the WhatsApp phone. Take the generated API key and inject it in Amplify (secret `EVOLUTION_API_KEY`) so the `start-broadcast` Lambda can call `/message/sendText/{instance}` securely.

Because the EC2 host is dedicated to this workload we retain predictable throughput while avoiding SaaS rate limits. The new Amplify-based platform simply consumes the same self-hosted endpoint instead of orchestrating messages via n8n.
