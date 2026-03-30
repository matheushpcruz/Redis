# Garantindo Alta Disponibilidade no Redis com Redis Sentinel

> Post de minha autoria publicado em [blog.4linux.com.br](https://blog.4linux.com.br/garantindo-alta-disponibilidade-no-redis-com-redis-sentinel/) — 3 de abril de 2025

O Redis, além de ser amplamente utilizado como banco de dados em memória para caching, também oferece suporte a replicação e failover automático, funcionalidades essenciais para cenários críticos onde a disponibilidade é indispensável. Neste post, vamos abordar como configurar o Redis Sentinel para garantir redundância e failover automático em ambientes Redis.

O **Redis Sentinel** é um utilitário que monitora e gerencia instâncias do Redis, determinando os papéis de master e slave em casos de failover, atuando, de certa forma, como um árbitro. Múltiplos Sentinels formam um quórum para decidir quando uma ação de failover deve ocorrer.

---

## Ambiente de Teste

A demonstração foi realizada em duas máquinas virtuais com **Red Hat Linux 9.4 (Plow)**:

| Papel  | Hostname          | Endereço IP   |
|--------|-------------------|---------------|
| Master | redhat-redis-01   | 172.27.11.10  |
| Slave  | redhat-redis-02   | 172.27.11.20  |

**Portas necessárias:**
- `6379` — Redis
- `26379` — Redis Sentinel

---

## 1. Instalação do Redis (Ambos os Servidores)

Instale o Redis via YUM e habilite o serviço:

```bash
yum install redis
systemctl enable --now redis
```

O parâmetro `enable` faz com que o serviço inicie automaticamente no boot, enquanto `--now` já o inicia imediatamente.

Em seguida, abra as portas necessárias no firewall:

```bash
firewall-cmd --permanent --add-port=6379/tcp
firewall-cmd --permanent --add-port=26379/tcp
firewall-cmd --reload
```

---

## 2. Configuração do Servidor Master

No servidor master (`redhat-redis-01`), edite o arquivo `/etc/redis/redis.conf` e adicione:

```
bind 0.0.0.0
port 6379
requirepass redis123
masterauth redis123
```

---

## 3. Configuração do Servidor Slave

No servidor slave (`redhat-redis-02`), edite o arquivo `/etc/redis/redis.conf` com os mesmos parâmetros do master, adicionando a diretiva `replicaof`:

```
bind 0.0.0.0
port 6379
requirepass redis123
masterauth redis123
replicaof 172.27.11.10 6379
```

**Descrição dos parâmetros:**
- `bind` — Define os endereços de escuta do Redis (0.0.0.0 aceita conexões de qualquer endereço).
- `port` — Porta padrão do Redis.
- `requirepass` — Define a senha de autenticação do Redis.
- `masterauth` — Senha utilizada pelo slave para autenticar com o master durante a replicação.
- `replicaof` — Especifica o IP e a porta do servidor master para replicação.

Após editar, reinicie o Redis em ambos os servidores:

```bash
systemctl restart redis
```

---

## 4. Configuração do Redis Sentinel (Ambos os Servidores)

Antes de configurar, ajuste as permissões dos arquivos de configuração e log do Sentinel:

```bash
chown redis:redis /etc/redis/sentinel.conf
chown redis:redis /var/log/redis/sentinel.log
```

**No servidor master**, edite o arquivo `/etc/redis/sentinel.conf`:

```
bind 172.27.11.10 127.0.0.1
sentinel monitor mymaster 172.27.11.10 6379 2
sentinel auth-pass mymaster redis123
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
```

**No servidor slave**, edite o arquivo `/etc/redis/sentinel.conf` com o IP correspondente:

```
bind 172.27.11.20 127.0.0.1
sentinel monitor mymaster 172.27.11.10 6379 2
sentinel auth-pass mymaster redis123
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
```

**Descrição dos parâmetros:**
- `bind` — Endereços em que o Sentinel irá escutar (IP local e localhost).
- `sentinel monitor` — Define o nome do master, seu IP, porta e o quórum necessário (2 Sentinels devem concordar para o failover ocorrer).
- `sentinel auth-pass` — Senha de autenticação correspondente ao `requirepass` do Redis.
- `sentinel down-after-milliseconds` — Tempo em milissegundos sem resposta para considerar uma instância como inativa (5000ms = 5 segundos).
- `sentinel failover-timeout` — Tempo de espera em milissegundos antes de tentar um novo failover (60000ms = 60 segundos).

Habilite e inicie o Sentinel em ambos os servidores:

```bash
systemctl enable --now redis-sentinel
```

---

## 5. Demonstração do Failover

Com tudo configurado, vamos simular a queda do servidor master parando o Redis:

```bash
systemctl stop redis
```

O Sentinel detecta a inatividade do master e, após atingir o quórum, promove automaticamente o slave a master. O processo completo pode ser acompanhado no log do Sentinel (`/var/log/redis/sentinel.log`):

```
7457:X 02 Feb 2025 16:59:53.101 # +sdown master mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:53.157 # +odown master mymaster 172.27.11.10 6379 #quorum 2/2
7457:X 02 Feb 2025 16:59:53.157 # +new-epoch 1
7457:X 02 Feb 2025 16:59:53.157 # +try-failover master mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:53.170 # +vote-for-leader bb26e0e5c9b7f5c7735a608c6352c804eef425e3 1
7457:X 02 Feb 2025 16:59:53.194 # 79c624a256d5471cbcab7523ecbd02b97af3689d voted for bb26e0e5c9b7f5c7735a608c6352c804eef425e3 1
7457:X 02 Feb 2025 16:59:53.222 # +elected-leader master mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:53.222 # +failover-state-select-slave master mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:53.281 # +selected-slave slave 172.27.11.20:6379 172.27.11.20 6379 @ mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:53.281 * +failover-state-send-slaveof-noone slave 172.27.11.20:6379 172.27.11.20 6379 @ mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:53.353 * +failover-state-wait-promotion slave 172.27.11.20:6379 172.27.11.20 6379 @ mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:54.217 # +promoted-slave slave 172.27.11.20:6379 172.27.11.20 6379 @ mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:54.217 # +failover-state-reconf-slaves master mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:54.293 # +failover-end master mymaster 172.27.11.10 6379
7457:X 02 Feb 2025 16:59:54.293 # +switch-master mymaster 172.27.11.10 6379 172.27.11.20 6379
7457:X 02 Feb 2025 16:59:54.294 * +slave slave 172.27.11.10:6379 172.27.11.10 6379 @ mymaster 172.27.11.20 6379
7457:X 02 Feb 2025 16:59:59.307 # +sdown slave 172.27.11.10:6379 172.27.11.10 6379 @ mymaster 172.27.11.20 6379
```

Os logs evidenciam o processo completo: detecção da queda (`+sdown`, `+odown`), votação (`+vote-for-leader`), eleição do líder (`+elected-leader`), promoção do slave (`+promoted-slave`) e a confirmação da troca de master (`+switch-master`).

Ao reiniciar o Redis no servidor original, ele retorna ao cluster como **réplica secundária** (não como master).

Para restaurar o master original manualmente, execute a partir de um dos Sentinels:

```bash
redis-cli -p 26379 sentinel failover mymaster
OK
```

Os logs confirmam a reassinalação dos papéis e a reconversão do antigo master para slave:

```
7245:X 02 Feb 2025 17:02:40.253 # +elected-leader master mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:40.253 # +failover-state-select-slave master mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:40.309 # +selected-slave slave 172.27.11.10:6379 172.27.11.10 6379 @ mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:40.309 * +failover-state-send-slaveof-noone slave 172.27.11.10:6379 172.27.11.10 6379 @ mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:40.411 * +failover-state-wait-promotion slave 172.27.11.10:6379 172.27.11.10 6379 @ mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:41.319 # +promoted-slave slave 172.27.11.10:6379 172.27.11.10 6379 @ mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:41.319 # +failover-state-reconf-slaves master mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:41.390 # +failover-end master mymaster 172.27.11.20 6379
7245:X 02 Feb 2025 17:02:41.390 # +switch-master mymaster 172.27.11.20 6379 172.27.11.10 6379
7245:X 02 Feb 2025 17:02:41.391 * +slave slave 172.27.11.20:6379 172.27.11.20 6379 @ mymaster 172.27.11.10 6379
```

---

## Conclusão

O Redis Sentinel oferece uma ferramenta nativa de failover, com poucos passos de configuração, essencial para ambientes escaláveis que exigem alta disponibilidade. Com dois ou mais nós Redis em replicação e o Sentinel monitorando o ambiente, é possível garantir a continuidade e a disponibilidade das aplicações mesmo em caso de falha do servidor master.
