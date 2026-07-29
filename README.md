---

# Projeto Korp - Serviço HTTP, Monitoramento e Automação

Este repositório contém a solução completa do desafio **Korp**, contemplando:

- Serviço HTTP em **Golang** expondo métricas para Prometheus
- Proxy reverso com **NGINX**
- Monitoramento com **Prometheus** e **Grafana** (dashboard provisionado automaticamente)
- Automação total do ambiente com **Ansible**

---

## Estrutura do Projeto

```
test-projeto-korp/
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
│   │       └── dashboard.yml
│   └── dashboards/
│       └── http-server-projeto-korp-dashboard.json
├── docker-compose.yml            # Orquestração dos containers
└── playbook.yml                  # Automação Ansible
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
   curl http://localhost:80/projeto-korp
   ```
   **Resposta esperada:**
   ```json
   {"nome":"Projeto Korp","horario":"2025-01-01T12:00:00Z"}
   ```
4. Acesse as interfaces de monitoramento:
   - **Prometheus:** http://localhost:9090
   - **Grafana:** http://localhost:3000 (usuário `admin`, senha `admin`)
   - O dashboard **"Projeto Korp - Métricas"** já estará disponível automaticamente.

---

## Automação com Ansible

O playbook **`playbook.yml`** provisiona todo o ambiente em um host Linux remoto, incluindo instalação do Docker, build da imagem, subida dos containers e validação.

### Uso

Edite o comando abaixo com os dados do seu host:
```bash
ansible-playbook -i 'IP_DO_HOST,' -u usuario --private-key chave.pem playbook.yml
```
> **Nota:** Se estiver executando localmente, use `-i 'localhost,' -c local` e remova a opção `become: true` ou garanta permissões.

O playbook realiza as seguintes etapas:
1. Instala Docker e Docker Compose no destino
2. Copia todos os arquivos do projeto para `/opt/projeto-korp`
3. Constrói a imagem do serviço Go
4. Sobe os containers com Docker Compose
5. Aguarda inicialização e valida a resposta HTTP
6. Exibe a resposta no console e as URLs de acesso

Ao final, o serviço e o monitoramento estarão ativos.

---

## Configuração do Grafana (bônus)

O Grafana é provisionado automaticamente via arquivos estáticos:

- **datasource.yml** – conecta ao Prometheus interno (`http://prometheus:9090`)
- **dashboard.yml** – registra o provedor de dashboards
- **http-server-projeto-korp-dashboard.json** – painel com as métricas:
  - **Disponibilidade** (gauge `service_up`)
  - **Volume de requisições** (taxa de `http_requests_total`)

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
- A porta 80 do host é mapeada para o NGINX; o serviço Go não é exposto diretamente.
- O playbook Ansible foi testado em Ubuntu 22.04, mas é adaptável a outras distribuições.

---

**Desenvolvido como parte do desafio técnico Korp.**
