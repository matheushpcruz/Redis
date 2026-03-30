# Gerenciamento de Acessos no Redis com Controle de Usuários e Permissões por Chaves

> Post de minha autoria publicado em [blog.4linux.com.br](https://blog.4linux.com.br/gerenciamento-de-acessos-no-redis-com-controle-de-usuarios-e-permissoes-por-chaves/) — 21 de janeiro de 2025

O Redis é amplamente utilizado para caching, filas e armazenamento de dados em memória de alta performance. Em muitos cenários, é comum que múltiplas aplicações ou usuários compartilhem o mesmo cluster Redis, mas nem todos devem ter o mesmo nível de acesso aos dados. Para isso, o Redis permite a criação de **usuários** e políticas de controle de acesso (ACLs), possibilitando que apenas determinados usuários tenham permissão para acessar certos caches, sendo possível definir a politica de acesso com base em nos **nomes de chave**.

Neste post, vamos abordar como configurar usuários no Redis e como restringir o acesso a caches específicos utilizando esses usuários.

---

### 1. Configuração Básica de Usuários no Redis

A partir do Redis 6, foi adicionado o recurso de ACL (Access Control List), que permite a criação de múltiplos usuários com permissões e acessos específicos. Antes dessa versão, não era possível criar vários usuários, pois havia apenas um usuário definido de forma global, com um único conjunto de permissões ajustado no arquivo `redis.conf`, e a senha era configurada no campo `requirepass`.

Para configurar um novo usuário, pode ser editado o arquivo de configuração do Redis (`redis.conf`) ou utilizar os comandos ACL diretamente no CLI do Redis.

Exemplo de criação de um usuário:

```
ACL SETUSER usuario ON >password123 ~* +@read +@write
```

Basicamente, esse comando cria o usuário:

- `>password123` define a senha.
- `~*` especifica que o usuário pode acessar todas as chaves.
- `+@read` concede permissão de leitura.
- `+@write` concede permissão de escrita.

---

### 2. Restringindo o acesso baseado no nome das chaves

O redis nos permite o uso de alguns padrões (prefixos) para restringir o acesso a certas chaves ou até mesmo limitando o usuário a criar chaves padronizadas. Isso pode ser feito através do caractere `~` seguido do padrão desejado.

Por exemplo, suponha que temos 3 usuários: 1 administrador e 2 específicos para sistemas, como `usuario-rh` e `usuario-prod`, divididos da seguinte maneira:

- O usuário `admin` pode acessar todos os caches, pois é o administrador do ambiente.
- O `usuario-rh` só poderá criar e usar caches que comecem com "sistema-rh".
- O `usuario-prod` só poderá criar e usar caches que comecem com "sistema-prod".

Podemos configurar usuários específicos da seguinte forma:

**1. Usuário Admin:**

```
ACL SETUSER admin ON >admin123 ~* +@all
```

- O usuário `admin` tem acesso a todas as operações, com o comando `+@all`, em todos os caches do ambiente.

**2. Usuário `usuario-rh`:**

```
ACL SETUSER usuario-rh ON >password123 ~sistema-rh* +@read +@write
```

- O usuário `usuario-rh` tem permissões de leitura e escrita em todas as chaves que começam com "sistema-rh".

**3. Usuário `usuario-prod`:**

```
ACL SETUSER usuario-prod ON >password123 ~sistema-prod* +@read +@write
```

- O usuário `usuario-prod` tem permissões de leitura e escrita em todas as chaves que começam com "sistema-prod".

Com essa configuração, garantimos que cada usuário tenha acesso somente aos caches que lhe dizem respeito, aumentando a segurança e a organização dos dados dentro do Redis.

---

### 3. Testando os Acessos

Após configurar os usuários, vamos testar as restrições de acesso para garantir que cada usuário possa acessar e criar apenas as chaves permitidas.

**1. Vamos autenticar com o `usuario-rh`:**

```
127.0.0.1:6379> AUTH usuario-rh password123
OK
127.0.0.1:6379>
```

**2. Tentar criar uma chave permitida:**

```
127.0.0.1:6379> SET sistema-rh-0001 "acesso tela de login"
OK
127.0.0.1:6379>
```

**3. Tentar criar uma chave não permitida:**

```
127.0.0.1:6379> SET chave_nao_permitida "teste"
(error) NOPERM this user has no permissions to access one of the keys used as arguments
127.0.0.1:6379>
```

**4. Agora vamos autenticar com o `usuario-prod`:**

```
127.0.0.1:6379> AUTH usuario-prod password123
OK
127.0.0.1:6379>
```

**5. Tentar acessar a chave anteriormente criada pelo `usuario-rh`:**

```
127.0.0.1:6379> GET sistema-rh-0001
(error) NOPERM this user has no permissions to access one of the keys used as arguments
127.0.0.1:6379>
```

Com isso, podemos validar que os usuários foram limitados a criar as chaves definidas na sua criação, garantindo que só eles tenham acesso às suas chaves.

---

### Conclusão

O uso de ACLs no Redis proporciona uma camada extra de segurança, especialmente útil em ambientes com múltiplos usuários ou aplicações que compartilham o mesmo cluster, permitindo dividir a segurança e segregar os acessos de acordo com a aplicação que os utiliza.

Configurando corretamente as permissões e acessos com base em nomes de chaves, é possível garantir que cada usuário tenha acesso apenas aos dados necessários, reduzindo riscos e melhorando a organização do ambiente.

Lembre-se de que podemos seguir as boas práticas abaixo para gerenciar o Redis:

- **Utilizar Prefixos Consistentes**: Padronizar os nomes das chaves com prefixos significativos para facilitar a configuração de ACLs.
- **Revisar Regularmente as Permissões**: Periodicamente, revisar as permissões dos usuários para garantir que eles não tenham acesso desnecessário.
- **Centralizar as Configurações de ACLs**: Centralizar as configurações em um arquivo de configuração pode facilitar a manutenção e aplicação das políticas, seja no `redis.conf` ou em outro arquivo que possa ser incluído no `redis.conf`.
