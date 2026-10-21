# Catálogo de livros

## Subir com Docker

```sh
docker compose up -d --build
```

Depois, acesse <http://localhost:8080>.

Para parar:

```sh
docker compose down
```

## Configurar o domínio na VPS da Locaweb

Os exemplos abaixo consideram uma VPS com Ubuntu ou Debian e o site executando na porta `8080`, conforme definido no `compose.yaml`. Substitua `seudominio.com.br` pelo seu domínio.

### 1. Apontar o domínio para a VPS

Descubra o IP público da VPS no painel da Locaweb. Depois, abra a administração de DNS do domínio e crie ou altere estes registros:

| Tipo | Nome/Host | Valor/Destino |
| --- | --- | --- |
| `A` | `@` | IP público da VPS |
| `A` | `www` | IP público da VPS |

Se o DNS do domínio estiver hospedado em outro provedor, faça a alteração no painel desse provedor, e não no painel da VPS. A propagação pode levar algumas horas. Para conferir:

```sh
dig +short seudominio.com.br
dig +short www.seudominio.com.br
```

Os dois comandos devem retornar o IP público da VPS antes de configurar o HTTPS.

### 2. Subir o site

Na pasta do projeto, execute:

```sh
docker compose up -d --build
```

Confira se o contêiner está funcionando:

```sh
curl http://127.0.0.1:8080
```

### 3. Instalar o Nginx e liberar o firewall

```sh
sudo apt update
sudo apt install -y nginx
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

No painel da VPS da Locaweb, confirme também que as regras de firewall permitem conexões TCP nas portas `80` (HTTP) e `443` (HTTPS).

### 4. Configurar o proxy reverso

Crie o arquivo `/etc/nginx/sites-available/catalogo-livros`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name seudominio.com.br www.seudominio.com.br;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Ative a configuração e recarregue o Nginx:

```sh
sudo ln -s /etc/nginx/sites-available/catalogo-livros /etc/nginx/sites-enabled/catalogo-livros
sudo nginx -t
sudo systemctl reload nginx
```

O site já deve abrir em `http://seudominio.com.br`.

### 5. Ativar HTTPS

Com o DNS já propagado, instale o Certbot e gere o certificado gratuito Let's Encrypt:

```sh
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d seudominio.com.br -d www.seudominio.com.br
```

Quando solicitado, escolha o redirecionamento automático de HTTP para HTTPS. Por fim, teste a renovação do certificado:

```sh
sudo certbot renew --dry-run
```

Depois disso, o site estará disponível em `https://seudominio.com.br`.

## Adicionar um livro

Edite `books.json` e adicione um novo objeto dentro da lista, separado do anterior por vírgula:

```json
[
  {
    "name": "Nome do livro",
    "author": "Nome do autor",
    "description": "Descrição do livro",
    "price": "R$ 100,00",
    "image": "https://exemplo.com/capa.jpg"
  }
]
```

Os campos obrigatórios são `name` e `image`. A imagem também pode ser um link público do Google Drive no formato `https://drive.google.com/file/d/ID/view`.

O arquivo `books.json` está montado no contêiner pelo `compose.yaml`. Assim, basta salvá-lo e recarregar a página; não é necessário reconstruir a imagem Docker.
