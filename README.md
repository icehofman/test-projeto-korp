# Projeto Korp - Serviço HTTP, Monitoramento e Automação

Este repositório contém a solução completa do desafio **Korp**, contemplando:

- Serviço HTTP em **Golang** expondo métricas para Prometheus
- Proxy reverso com **NGINX** (funcional com resolução de DNS e dependência de inicialização)
- Monitoramento com **Prometheus** e **Grafana** (dashboard provisionado automaticamente, com gauge corrigido)
- Automação total do ambiente com **Ansible**
- Testes do playbook Ansible via Docker
- Script Docker para teste de carga (milhares de requisições)

---

## Estrutura do Projeto

```
projeto-korp/
├── http-server-projeto-korp/     # Código fonte e Dockerfile do serviço Go
│   ├── main.go
│   ├── go.mod
│   └── Dockerfile
├── nginx/                        # Configuração do proxy reverso
│   └── http-server-projeto-korp.conf
├── prometheus/                   # Configuração do Prometheus
│   └── prometheus.yml
├── grafana/                      # Provisionamento automático do Grafana
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── datasource.yml
│   │   └── dashboards/
│   │       ├── dashboard.yml
│   │       └── http-server-projeto-korp-dashboard.json
├── docker-compose.yml            # Orquestração dos containers
├── playbook.yml                  # Automação Ansible
├── Dockerfile.test               # Teste rápido do playbook (dry‑run)
├── Dockerfile.fulltest           # Teste completo com provisionamento real
├── Dockerfile.loadtest           # Container para teste de carga
└── README.md
```

---

## Pré-requisitos

### Para execução manual com Docker Compose

- [Docker](https://docs.docker.com/engine/install/)
- [Docker Compose](https://docs.docker.com/compose/install/)

### Para automação com Ansible

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/index.html) (>= 2.9) instalado na máquina de controle
- Coleção `community.docker`:
  ```bash
  ansible-galaxy collection install community.docker
  ```
- Acesso SSH ao host de destino (Linux, testado em Ubuntu 22.04)

---

## Como executar manualmente (Docker Compose)

1. Clone o repositório:
   ```bash
   git clone <url-do-repositorio>
   cd projeto-korp
   ```
2. Suba os containers:
   ```bash
   docker-compose up -d --build
   ```
3. Aguarde alguns segundos e teste o endpoint:
   ```bash
   curl http://localhost:8080/projeto-korp
   ```
   **Resposta esperada:**
   ```json
   {"nome":"Projeto Korp","horario":"2026-07-30T18:11:11Z"}
   ```
4. Acesse as interfaces de monitoramento:
   - **Prometheus:** http://localhost:9090
   - **Grafana:** http://localhost:3000 (usuário `admin`, senha `admin` – **sem exigência de troca de senha**)
   - O dashboard **"Projeto Korp - Métricas"** já estará disponível automaticamente.

---

## Automação com Ansible

O playbook **`playbook.yml`** provisiona todo o ambiente em um host Linux remoto (ou local via container), incluindo instalação do Docker, build da imagem, subida dos containers e validação.

### Uso

Edite o comando abaixo com os dados do seu host:
```bash
ansible-playbook -i 'IP_DO_HOST,' -u usuario --private-key chave.pem playbook.yml
```
> **Nota:** Se estiver executando localmente, use `-i 'localhost,' -c local` e remova `become: true` ou garanta permissões.

O playbook realiza as seguintes etapas:
1. Instala Docker e Docker Compose no destino
2. Copia todos os arquivos do projeto para `/opt/projeto-korp`
3. Gera as configurações do Prometheus, NGINX e Grafana (evitando conflitos de diretórios)
4. Constrói a imagem do serviço Go
5. Sobe os containers com Docker Compose
6. Valida o serviço internamente (via container temporário) e exibe a resposta JSON
7. Exibe as URLs de acesso

Ao final, o serviço e o monitoramento estarão ativos.

---

## Testando o Playbook Ansible com Docker

Você pode validar o playbook sem precisar de uma máquina remota, utilizando containers Docker que simulam o ambiente de destino.

### Teste rápido de sintaxe e simulação (dry‑run)

Utilize o arquivo `Dockerfile.test` para construir uma imagem que executa o playbook em modo *check*, apenas verificando se a sintaxe e as tarefas são válidas.

**Construir e executar:**
```bash
docker build -t playbook-test -f Dockerfile.test .
docker run --rm playbook-test
```

### Teste completo com provisionamento real

O arquivo `Dockerfile.fulltest` utiliza uma imagem que suporta Docker, permitindo que o playbook provisione o próprio container localmente (via socket do Docker).

**Construir:**
```bash
docker build -t playbook-fulltest -f Dockerfile.fulltest .
```

**Executar (usando o socket do host – requer `--privileged`):**
```bash
docker run --rm --privileged -v /var/run/docker.sock:/var/run/docker.sock playbook-fulltest
```

> **Atenção:** Esse método afeta o Docker do host. Para um teste totalmente isolado, utilize um ambiente Docker‑in‑Docker (ex.: `docker:dind`) e ajuste o playbook conforme necessário.

Após a execução, o playbook exibirá a resposta do serviço (JSON) e as URLs de acesso. O Grafana e Prometheus estarão acessíveis nas portas mapeadas.

---

## Teste de carga (milhares de requisições)

Um container dedicado permite disparar milhares de requisições contra o serviço, ideal para visualizar o gráfico de volume no Grafana.

### Pré‑requisito: ambiente em execução

Certifique‑se de que os containers estão rodando:

```bash
docker-compose up -d --build
```

### Identifique o nome da rede Docker

O nome da rede é gerado a partir do diretório do projeto. Para listá‑lo:

```bash
docker network ls | grep korp
# Exemplo de saída: test-projeto-korp_korp-net
```

Anote o nome exato que aparecer na coluna `NAME`.

### Usando o Dockerfile.loadtest (recomendado)

1. Construa a imagem:
   ```bash
   docker build -t load-test -f Dockerfile.loadtest .
   ```

2. Execute o teste informando a rede correta:
   ```bash
   docker run --rm --network <NOME_DA_REDE> load-test
   ```
   - Por padrão, envia **10.000 requisições**. Para alterar, informe a variável `REQUESTS`:
     ```bash
     docker run --rm --network <NOME_DA_REDE> -e REQUESTS=50000 load-test
     ```

   Exemplo com o nome `test-projeto-korp_korp-net`:
   ```bash
   docker run --rm --network test-projeto-korp_korp-net load-test
   ```

### Alternativa rápida (sem build)

Substitua `<NOME_DA_REDE>` pelo nome identificado anteriormente:

```bash
docker run --rm --network <NOME_DA_REDE> curlimages/curl sh -c 'for i in $(seq 10000); do curl -s http://http-server-projeto-korp:8080/projeto-korp; echo; done'
```

Enquanto o teste executa, acesse o dashboard no Grafana e veja o gráfico de "Volume de Requisições" subir.

---

## Configuração do Grafana (bônus)

O Grafana é provisionado automaticamente via arquivos estáticos:

- **datasource.yml** – conecta ao Prometheus interno (`http://prometheus:9090`)
- **dashboard.yml** – registra o provedor de dashboards
- **http-server-projeto-korp-dashboard.json** – painel com as métricas:
  - **Disponibilidade** (gauge `service_up`, escala 0–1, verde quando ≥ 1)
  - **Volume de requisições** (taxa de `http_requests_total`)

O gauge foi ajustado para exibir o arco completo e verde quando o serviço está disponível, sem sobreposição de cores.

Nenhuma ação manual é necessária.

---

## Métricas expostas

O serviço Go expõe em `/metrics` as seguintes métricas no formato Prometheus:

- `http_requests_total` (counter): número total de requisições ao endpoint `/projeto-korp`
- `service_up` (gauge): valor `1` quando o serviço está operacional

---

## Notas finais

- O endpoint responde sempre com o horário UTC corrente (`time.Now().UTC()`).
- Toda a comunicação entre containers ocorre na rede `korp-net` (bridge).
- A porta **8080** do host é mapeada para o NGINX (porta 80 interna); o serviço Go não é exposto diretamente.
- O playbook Ansible foi testado em Ubuntu 22.04, mas é adaptável a outras distribuições.
- Os Dockerfiles de teste permitem integração contínua e validação rápida do playbook.
- O NGINX foi configurado com `depends_on` e `restart` para garantir que o proxy reverso funcione corretamente, resolvendo o problema de DNS entre containers.

---

**Desenvolvido como parte do desafio técnico Korp.**