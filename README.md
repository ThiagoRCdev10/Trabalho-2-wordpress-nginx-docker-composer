# Trabalho 2 – WordPress com Múltiplas Instâncias usando Nginx e Docker Compose

## 1. Objetivo

Executar múltiplas instâncias do WordPress para aumentar a escalabilidade, usando o **Nginx como balanceador de carga** e o **Docker Compose** para orquestrar os contêineres.

## 2. Arquitetura

```
                      ┌──────────────┐
                 ┌───▶│ wordpress1   │───┐
                 │    └──────────────┘    │
Cliente ──▶ nginx ───▶│ wordpress2   │───┼──▶ mysql
(porta 80)       │    └──────────────┘    │
                 └───▶│ wordpress3   │───┘
                      └──────────────┘
```

São cinco contêineres:

| Serviço      | Imagem                           | Função                                          |
|--------------|----------------------------------|-------------------------------------------------|
| `nginx`      | `nginx:1.19.0`                   | Balanceador de carga (única porta exposta: 80)  |
| `wordpress1` | `wordpress:5.4.2-php7.2-apache`  | Instância 1 do WordPress                        |
| `wordpress2` | `wordpress:5.4.2-php7.2-apache`  | Instância 2 do WordPress                        |
| `wordpress3` | `wordpress:5.4.2-php7.2-apache`  | Instância 3 do WordPress                        |
| `db`         | `mysql:5.7`                      | Banco de dados compartilhado pelos três WordPress |

## 3. Estrutura de arquivos

```
Trabalho 2/
├── docker-compose.yml
├── nginx.conf
├── README.md
└── wordpress/        <- criada automaticamente ao subir os contêineres
```

- `docker-compose.yml`: define os cinco contêineres, a rede e os volumes.
- `nginx.conf`: configura o Nginx para repassar as requisições aos três WordPress (pool `upstream`, balanceamento round-robin).
- `wordpress/`: pasta do host compartilhada entre os WordPress e o Nginx.

## 4. O que foi feito

1. Criado o `nginx.conf` com:
   - `upstream wordpress` contendo `wordpress1`, `wordpress2` e `wordpress3`;
   - servidor escutando na porta 80;
   - `proxy_pass http://wordpress;` repassando todas as requisições ao pool;
   - cabeçalho `X-Upstream: $upstream_addr` para identificar qual contêiner respondeu.
2. Criado o `docker-compose.yml` com os cinco serviços, atendendo aos requisitos:
   - **Imagens exigidas:** `mysql:5.7`, `wordpress:5.4.2-php7.2-apache` e `nginx:1.19.0`.
   - **WordPress inacessível pelo host:** os serviços `wordpress1/2/3` não têm `ports`; só o Nginx os alcança, pela rede interna.
   - **MySQL inacessível pelo host:** o serviço `db` também não tem `ports`; só os WordPress o acessam.
   - **Mesmo banco para todos:** os três WordPress usam `WORDPRESS_DB_HOST: db:3306` com as mesmas credenciais.
   - **Pasta compartilhada:** os três WordPress montam `./wordpress:/var/www/html`.
   - **Nginx enxerga a mesma pasta:** `./wordpress:/usr/share/nginx/html`.
   - **Configuração do Nginx:** `./nginx.conf:/etc/nginx/nginx.conf:ro`.
   - **Persistência do banco:** volume `db_data` em `/var/lib/mysql`.
3. Subidos os contêineres, instalado o WordPress em `http://localhost/` e criada a conta de administrador.
4. Testado o balanceamento com `curl -I` (ver seção 6).

## 5. Como executar

### Pré-requisitos

- Docker e Docker Compose instalados.
- No Windows/Mac, o **Docker Desktop** precisa estar aberto e rodando.
- A porta **80** livre no computador.

### Passo a passo

1. Abra o terminal dentro da pasta `Trabalho 2` (onde estão o `docker-compose.yml` e o `nginx.conf`).

2. Verifique o Docker:
   ```bash
   docker --version
   docker compose version
   ```

3. Suba os cinco contêineres:
   ```bash
   docker compose up -d
   ```
   Na primeira vez as imagens são baixadas, então pode demorar.

4. Confira se todos estão rodando:
   ```bash
   docker compose ps
   ```
   Devem aparecer `db`, `wordpress1`, `wordpress2`, `wordpress3` e `nginx` com status *running/Up*.

5. Acesse `http://localhost/` no navegador e conclua a instalação do WordPress (idioma, título, usuário e senha). Como os três contêineres usam o mesmo banco, a instalação vale para todos.

## 6. Como testar o balanceamento

Execute várias vezes:

```bash
curl -I http://localhost/
```

> No PowerShell do Windows, use `curl.exe -I http://localhost/`.

O que observar:

- **Antes da instalação do WordPress:** `HTTP/1.1 302 Found`, com `Location: .../wp-admin/install.php` (redirecionamento esperado).
- **Depois da instalação:** `HTTP/1.1 200 OK`.
- O cabeçalho **`X-Upstream`** deve alternar entre três IPs diferentes (round-robin), que são os IPs dos contêineres WordPress. Exemplo do resultado obtido:

  ```
  X-Upstream: 172.18.0.3:80
  X-Upstream: 172.18.0.5:80
  X-Upstream: 172.18.0.4:80
  X-Upstream: 172.18.0.3:80
  ...
  ```
Resultados obtidos do terminal:
![Teste do balanceamento](teste%20do%20balanceamento.png)

### O que o comando faz

- `curl`: faz requisições HTTP pelo terminal.
- `-I`: envia uma requisição `HEAD`, retornando apenas os cabeçalhos.
- `http://localhost/`: endereço do Nginx, na porta 80.

### Teste pelo navegador (opcional)

1. Abra `http://localhost/` e pressione **F12** > aba **Rede** (Network).
2. Marque **Desativar cache** e dê refresh.
3. Veja o `X-Upstream` nos *Response Headers* de **diferentes requisições** (CSS, JS, imagens) do mesmo carregamento.

> **Observação:** o round-robin conta cada requisição, não cada refresh. Como uma página do WordPress dispara várias requisições, a requisição principal (HTML) pode cair sempre no mesmo contêiner por coincidência da contagem. Isso não indica erro. Para ver a alternância em um refresh simples, use uma página de requisição única, como `http://localhost/license.txt`.

### Teste de conteúdo compartilhado

1. Acesse `http://localhost/wp-admin` e publique um post.
2. Atualize `http://localhost/` várias vezes: o post aparece sempre, independentemente do contêiner que responde, pois todos usam o mesmo banco e a mesma pasta.

## 7. Comandos úteis

| Comando                                 | O que faz                                                        |
|-----------------------------------------|------------------------------------------------------------------|
| `docker compose up -d`                  | Sobe os contêineres em segundo plano                             |
| `docker compose ps`                     | Lista os contêineres e seus status                               |
| `docker compose logs nginx`             | Mostra os logs de um serviço (troque `nginx` pelo nome desejado) |
| `docker compose down`                   | Para e remove os contêineres (os dados continuam guardados)      |
| `docker compose down -v`                | Para tudo e **apaga o banco de dados** (recomeça do zero)        |

Para recomeçar totalmente do zero: `docker compose down -v` e apague a pasta `wordpress` (no Linux/Mac pode ser necessário `sudo rm -rf wordpress`).

## 8. Problemas comuns

- **Nginx não sobe (`host not found in upstream`):** os WordPress ainda não estavam prontos. Rode `docker compose up -d` novamente.
- **Porta 80 ocupada:** pare o programa que usa a porta ou troque `"80:80"` por `"8080:80"` no compose e acesse `http://localhost:8080/`.
- **Erro nos arquivos do WordPress na primeira subida:** os três contêineres compartilham a mesma pasta; apague a pasta `wordpress`, rode `docker compose down` e suba novamente.
- **Aparece a tela de instalação de novo:** o volume `db_data` foi apagado (`down -v`) ou o projeto está em outra pasta, o que cria outro banco. Cada projeto Compose tem seus próprios volumes.

## 9. Observações

- As senhas do `docker-compose.yml` são apenas de exemplo, para uso acadêmico.
- Este trabalho é independente de outros trabalhos com WordPress/MySQL: cada projeto usa seu próprio banco e seus próprios arquivos, então é preciso instalar o WordPress e criar a conta novamente.