# poc-mariadb-blue-green

Laboratório de **Blue/Green Deployment com MariaDB**: atualização de versão (10.6 → 11.8) com replicação GTID, controle de lag, troca por DNS e rollback, com o mínimo de tempo sem escrita.

> A explicação completa de cada passo está no artigo [dev.to](https://dev.to/reinaldocoelho/bluegreen-deployment-no-mariadb-atualizando-o-banco-com-quase-zero-downtime-33dm).

## Componentes

| Serviço | O que é                                                             |
| ------- | ------------------------------------------------------------------- |
| `dns`   | CoreDNS resolvendo `db.lab.internal` para o primário atual (TTL 5s) |
| `blue`  | MariaDB 10.6, o banco em produção                                   |
| `green` | MariaDB 11.8, o banco novo                                          |
| `app`   | Alpine só com o cliente `mariadb`, gravando continuamente pelo DNS  |

Requisitos: Docker com Docker Compose e Bash. Todos os comandos são executados na raiz do repositório.

## 1. Banco atual em uso (blue) e aplicação

```bash
docker compose up -d --wait dns blue
./scripts/seed-blue.sh
docker compose up -d app
docker compose logs -f app        # em outro terminal
```

## 2. Banco novo (green) recebendo replicação

```bash
docker compose up -d --wait green
./scripts/create-green.sh
```

## 2.1 Acompanhando o lag

```bash
./scripts/lag.sh green blue       # terminal 1
./scripts/burst.sh blue           # terminal 2: provoca lag
```

## 3. Conferindo os dados

```bash
./scripts/verify.sh
```

## 4 a 6. Switchover

Escolha o que acontece com o primário antigo após a troca:

```bash
./scripts/switchover.sh blue green reverse    # blue vira réplica do green
./scripts/switchover.sh blue green readonly   # blue fica parado e somente leitura (como no RDS)
./scripts/verify.sh
```

## 7.1 Rollback

```bash
./scripts/switchover.sh green blue reverse    # após um switchover "reverse": sem perda de dados
./scripts/emergency-rollback.sh green blue    # após um switchover "readonly": perde o que foi gravado no green
```

## 7.2 Limpeza

```bash
./scripts/cleanup.sh blue                     # desfaz a replicação e para o servidor antigo
docker compose down -v                        # remove todo o laboratório
./scripts/set-dns.sh blue                     # volta o arquivo dns/hosts ao estado inicial
```

## Licença

[MIT](LICENSE)
