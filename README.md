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
├── wordpress/        <- criada automaticamente ao subir os contêineres
└── escalavel/        <- versão extra com número de WordPress variável (seção 10)
    ├── docker-compose.yml
    ├── nginx.conf
    └── escalar.sh
```

- `docker-compose.yml`: define os cinco contêineres, a rede e os volumes.
- `nginx.conf`: configura o Nginx para repassar as requisições aos três WordPress (pool `upstream`, balanceamento round-robin).
- `wordpress/`: pasta do host compartilhada entre os WordPress e o Nginx.
- `escalavel/docker-compose.yml`: versão com um único serviço `wordpress`, que pode ter quantas cópias você quiser.
- `escalavel/nginx.conf`: igual ao principal, mas o pool aponta para `wordpress:80`, que corresponde a todas as cópias.
- `escalavel/escalar.sh`: script que muda a quantidade de cópias e avisa o Nginx.

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

### Escolha o fluxo

O projeto pode ser executado de duas formas:

| | Fluxo A: versão principal | Fluxo B: versão escalável |
|---|---|---|
| Pasta | raiz do projeto | `escalavel/` |
| Quantidade de WordPress | Sempre 3 (`wordpress1`, `wordpress2`, `wordpress3`) | Você escolhe: 2, 3, 5, 10... |
| Comando para subir | `docker compose up -d` | `./escalar.sh 3` |
| Quando usar | Entrega do trabalho (segue o enunciado à risca) | Para mostrar o número de contêineres mudando |

> **Rode um fluxo por vez.** Os dois usam a porta 80. Para trocar de fluxo, veja [Trocando de um fluxo para o outro](#trocando-de-um-fluxo-para-o-outro).

### Fluxo A: versão principal (3 WordPress fixos)

1. Abra o terminal na **raiz do projeto**, onde estão o `docker-compose.yml` e o `nginx.conf`.

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

6. Teste o balanceamento (detalhes na seção 6):
   ```bash
   curl -I http://localhost/
   ```

7. Para desligar:
   ```bash
   docker compose down
   ```

### Fluxo B: versão escalável (quantidade variável)

O script `escalar.sh` é para **Linux, Mac ou WSL**. No PowerShell do Windows, use os comandos de [Sem o script](#sem-o-script-windows-powershell).

1. **Pare a versão principal, se ela estiver rodando.** Na raiz do projeto:
   ```bash
   docker compose down
   ```
   Não use `-v`, para não apagar o banco da versão principal.

2. **Entre na pasta da versão escalável:**
   ```bash
   cd escalavel
   ```

3. **Suba tudo escolhendo a quantidade de WordPress**, por exemplo 3:
   ```bash
   ./escalar.sh 3
   ```
   - Se aparecer `Permission denied`, rode `chmod +x escalar.sh` e tente de novo.
   - Se o Docker do seu computador exigir `sudo`, use `sudo ./escalar.sh 3`.
   - Na primeira vez, o script mostra `Primeira execução: preparando os arquivos do WordPress...`. Ele sobe 1 cópia, espera os arquivos serem copiados e depois sobe as outras. Isso leva alguns segundos.

   No final, o script mostra a lista de cópias rodando:
   ```
   NAME                    ...   SERVICE     STATUS
   escalavel-wordpress-1   ...   wordpress   Up
   escalavel-wordpress-2   ...   wordpress   Up
   escalavel-wordpress-3   ...   wordpress   Up
   ```

4. **Confira os contêineres:**
   ```bash
   docker compose ps
   ```
   Devem aparecer `escalavel-db-1`, `escalavel-nginx-1` e as cópias `escalavel-wordpress-1`, `-2`, `-3`.

5. **Instale o WordPress** em `http://localhost/`. Essa versão tem banco próprio, então a instalação é feita de novo, mesmo que você já tenha instalado na versão principal.
   > Se aparecer `502 Bad Gateway` logo na primeira subida, é o MySQL ainda se inicializando. Espere uns 10 segundos e recarregue.

6. **Teste o balanceamento.** Faça 20 requisições e conte quantas foram para cada cópia:
   ```bash
   for i in $(seq 20); do curl -sI http://localhost/ | grep X-Upstream; done | sort | uniq -c
   ```
   Com 3 cópias, aparecem 3 IPs diferentes, cada um com uma parte parecida das requisições.

7. **Mude a quantidade com tudo rodando:**
   ```bash
   ./escalar.sh 5      # aumenta para 5 cópias
   ./escalar.sh 2      # diminui para 2 cópias
   ```
   Depois de cada mudança, repita o comando do passo 6. O número de IPs diferentes acompanha a quantidade escolhida.

8. **Para desligar:**
   ```bash
   docker compose down
   ```
   Na próxima vez, basta entrar em `escalavel/` e rodar `./escalar.sh <quantidade>`. O banco e a instalação continuam guardados.

#### Sem o script (Windows PowerShell)

O script só junta comandos do Docker. Eles podem ser rodados à mão, dentro da pasta `escalavel`:

```powershell
# 1) Somente na primeira vez: suba 1 cópia e espere aparecer o arquivo
#    escalavel/wordpress/wp-config.php (alguns segundos)
docker compose up -d --scale wordpress=1

# 2) Escolha a quantidade de cópias
docker compose up -d --scale wordpress=5

# 3) Avise o Nginx que a lista de cópias mudou
docker compose exec nginx nginx -s reload
```

Para testar no PowerShell, rode várias vezes `curl.exe -I http://localhost/` e observe o `X-Upstream`.

### Trocando de um fluxo para o outro

Sempre desligue um fluxo antes de subir o outro. Use `down` **sem** `-v`, assim cada versão mantém seu banco e sua instalação.

**Da principal para a escalável** (partindo da raiz do projeto):
```bash
docker compose down
cd escalavel
./escalar.sh 3
```

**Da escalável para a principal** (partindo da pasta `escalavel`):
```bash
docker compose down
cd ..
docker compose up -d
```

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
| `./escalar.sh 5` (dentro de `escalavel/`) | Muda a versão escalável para 5 cópias do WordPress             |

Os comandos `docker compose` valem para os dois fluxos. Eles agem sobre a versão da pasta em que você está: a raiz ou `escalavel/`.

Para recomeçar totalmente do zero: `docker compose down -v` e apague a pasta `wordpress` (no Linux/Mac pode ser necessário `sudo rm -rf wordpress`). Na versão escalável, faça isso dentro de `escalavel/`.

## 8. Problemas comuns

- **Nginx não sobe (`host not found in upstream`):** os WordPress ainda não estavam prontos. Rode `docker compose up -d` novamente.
- **Porta 80 ocupada:** pare o programa que usa a porta ou troque `"80:80"` por `"8080:80"` no compose e acesse `http://localhost:8080/`.
- **Erro nos arquivos do WordPress na primeira subida:** os três contêineres compartilham a mesma pasta; apague a pasta `wordpress`, rode `docker compose down` e suba novamente.
- **Aparece a tela de instalação de novo:** o volume `db_data` foi apagado (`down -v`) ou o projeto está em outra pasta, o que cria outro banco. Cada projeto Compose tem seus próprios volumes. Por isso a versão escalável pede uma instalação própria.
- **`502 Bad Gateway` logo depois de subir:** o MySQL ou os WordPress ainda estão iniciando. Espere uns 10 segundos e tente de novo.
- **`Bind for 0.0.0.0:80 failed: port is already allocated`:** o outro fluxo ainda está rodando. Rode `docker compose down` na pasta dele.
- **Versão escalável não distribui para as cópias novas:** a quantidade foi mudada com `docker compose up --scale`, sem recarregar o Nginx. Use `./escalar.sh N` ou rode `docker compose exec nginx nginx -s reload`.

## 9. Observações

- As senhas do `docker-compose.yml` são apenas de exemplo, para uso acadêmico.
- Este trabalho é independente de outros trabalhos com WordPress/MySQL: cada projeto usa seu próprio banco e seus próprios arquivos, então é preciso instalar o WordPress e criar a conta novamente.

## 10. Versão escalável (extra)

A versão principal (raiz do projeto) segue o enunciado à risca: três serviços fixos, `wordpress1`, `wordpress2` e `wordpress3`. Para mudar a quantidade, seria preciso copiar e colar serviços no compose e editar o `nginx.conf`.

A pasta `escalavel/` traz uma versão em que a **quantidade de WordPress é escolhida na hora de subir**, sem editar nenhum arquivo.

### O que muda

| | Versão principal (raiz) | Versão escalável (`escalavel/`) |
|---|---|---|
| Serviços WordPress | 3 serviços fixos (`wordpress1/2/3`) | 1 serviço `wordpress` com N cópias |
| Quantidade | Sempre 3 | Qualquer uma: `./escalar.sh 5` |
| Nomes dos contêineres | `...-wordpress1-1`, `...-wordpress2-1`... | `escalavel-wordpress-1`, `-2`, `-3`... |
| Pool do Nginx | `server wordpress1; server wordpress2; server wordpress3;` | `server wordpress:80;` |

Ela fica numa pasta separada porque o enunciado exige os nomes `wordpress1/2/3` e exatamente cinco contêineres. A versão principal continua sendo a entrega do trabalho.

### Como funciona

- No `docker-compose.yml`, há **um único serviço `wordpress`**, sem `ports` e sem `container_name`, o que permite ao Docker criar várias cópias. Por padrão sobem 3 (`deploy.replicas: 3`).
- No `nginx.conf`, o pool tem só `server wordpress:80;`. O DNS interno do Docker responde a esse nome com o **IP de todas as cópias**, e o Nginx faz round-robin entre elas.
- O Nginx 1.19 **só consulta essa lista ao iniciar ou ao recarregar** a configuração. Por isso, depois de mudar a quantidade, é preciso rodar `nginx -s reload`. O script `escalar.sh` faz isso automaticamente.
- Todas as cópias usam a mesma pasta `./wordpress`. Na **primeira execução** essa pasta está vazia, e várias cópias copiando os arquivos ao mesmo tempo poderiam corromper a instalação. O script sobe só 1 cópia, espera o `wp-config.php` aparecer (sinal de que a cópia terminou) e só então sobe as demais.

### Como executar

O passo a passo completo está na seção 5, em [Fluxo B: versão escalável](#fluxo-b-versão-escalável-quantidade-variável).

### Resultado do teste

Resultado obtido ao testar 3 → 5 → 2 cópias, contando as requisições por IP com:

```bash
for i in $(seq 20); do curl -sI http://localhost/ | grep X-Upstream; done | sort | uniq -c
```


```
$ ./escalar.sh 3                         # 12 requisições
      4 X-Upstream: 172.20.0.3:80
      4 X-Upstream: 172.20.0.5:80
      4 X-Upstream: 172.20.0.6:80

$ ./escalar.sh 5                         # 20 requisições
      4 X-Upstream: 172.20.0.3:80
      4 X-Upstream: 172.20.0.5:80
      4 X-Upstream: 172.20.0.6:80
      4 X-Upstream: 172.20.0.7:80
      4 X-Upstream: 172.20.0.8:80

$ ./escalar.sh 2                         # 10 requisições
      5 X-Upstream: 172.20.0.3:80
      5 X-Upstream: 172.20.0.6:80
```

As requisições são divididas igualmente entre todas as cópias, e as cópias removidas deixam de receber tráfego.

> Logo após diminuir, uma requisição pode mostrar dois endereços, como `X-Upstream: 172.20.0.8:80, 172.20.0.6:80`. Isso acontece porque o Nginx ainda tentou uma cópia que acabou de ser removida, não conseguiu e repassou para outra. O usuário não vê erro nenhum.
