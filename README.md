# Guia Definitivo: Implantando e Gerenciando o GLPI via Docker

> ⚠️ **AVISO DE SEGURANÇA E AMBIENTE DE TESTES:**
>
> Todos os dados, senhas (ex: `secret_root_password`, `sua_senha_segura`), portas e configurações de ambiente apresentados neste repositório/guia são **estritamente para fins didáticos e testes locais**. NUNCA utilize estas credenciais ou configurações padrão em um ambiente de produção real. Ao subir para produção, gere senhas fortes, utilize variáveis de ambiente isoladas (.env) e implemente HTTPS/proxies reversos apropriados.

Este documento registra todo o processo de planejamento, resolução de pré-requisitos, arquitetura de containers, conceitos fundamentais, comandos essenciais e boas práticas utilizados para colocar o **GLPI (Gestionnaire Libre de Parc Informatique)** em funcionamento em um ambiente de testes local utilizando **Docker**.

## Índice

1. [Contexto, Escolha Tecnológica e Conceitos-Chave](#1-contexto-escolha-tecnológica-e-conceitos-chave)
2. [Preparação do Ambiente no Windows](#2-preparação-do-ambiente-no-windows-pré-requisitos)
3. [Arquitetura do Projeto: docker-compose.yml](#3-arquitetura-do-projeto-o-arquivo-docker-composeyml)
4. [Dicionário de Comandos do Docker](#4-dicionário-de-comandos-do-docker-guia-prático)
5. [Assistente de Instalação Web](#5-assistente-de-instalação-e-conexão-ao-banco)
6. [Gestão de Plugins no GLPI 12: Erros, Acertos e GLPI Network](#6-histórico-de-erros-acertos-e-instalação-de-plugins-no-glpi-12)
7. [Boas Práticas e Comandos de Diagnóstico](#7-boas-práticas-e-comandos-de-diagnóstico-úteis)

---

## 1. Contexto, Escolha Tecnológica e Conceitos-Chave

Para subir o GLPI rapidamente sem a sobrecarga de gerenciar manualmente uma Máquina Virtual (VM) tradicional, optamos pelo uso do **Docker** e do **Docker Compose**.

### Comparativo: Docker vs. Máquina Virtual (VM)

| Característica | Docker / Containers | Máquina Virtual (VM) |
| :--- | :--- | :--- |
| **Recursos** | Compartilham o kernel do host, uso baixíssimo de RAM/CPU. | Exige alocação estática (vCPUs, RAM dedicada, disco fixo). |
| **Agilidade** | Inicializa em segundos. | Processo de boot completo de um SO. |
| **Ambiente** | Isola perfeitamente a aplicação e o banco de dados. | Demanda configuração manual de todo o ecossistema (LAMP). |

### Conceitos-Chave Utilizados no Projeto

Para compreender toda a arquitetura deste repositório, é importante destacar dois termos fundamentais presentes no projeto:

* **MariaDB:** É o Sistema de Gerenciamento de Banco de Dados Relacional (SGBD) de código aberto, derivado do MySQL. No nosso ambiente, o MariaDB atua como o "cérebro" de armazenamento do GLPI, onde ficam guardados todos os chamados, inventário de ativos, tabelas de plugins, históricos e cadastros de usuários.
* **CLI (Command Line Interface / Linha de Comando):** É a interface baseada em modo texto onde você interage com o sistema digitando comandos no terminal (como PowerShell ou Bash), sem depender de cliques na interface gráfica (GUI). No GLPI 12, o uso da CLI (`php bin/console`) é o método mais seguro e poderoso para realizar tarefas administrativas avançadas, limpar caches e gerenciar plugins diretamente pelo motor da aplicação.

---

## 2. Preparação do Ambiente no Windows (Pré-requisitos)

O Docker Desktop para Windows roda sobre o subsistema Linux (WSL2), exigindo a ativação de recursos de virtualização a nível de kernel.

### O Desafio da Plataforma de Máquina Virtual

Ao tentar instalar o Docker sem os recursos ativados, ocorre o erro informando a falta da **Virtual Machine Platform**. Para resolver:

1. Abra o **PowerShell** com privilégios de **Administrador**.
2. Execute o comando de instalação do subsistema Linux:
   ```bash
   wsl --install
   ```
3. **Reinicialização Obrigatória:** Reinicie o computador para carregar os módulos de virtualização.
4. O Docker Desktop estará pronto quando o ícone da baleia ficar verde/estável na bandeja do sistema.

---

## 3. Arquitetura do Projeto: O Arquivo `docker-compose.yml`

O Docker Compose orquestra múltiplos containers em uma rede isolada. Nosso arquivo `docker-compose.yml` contém dois serviços principais:

```yaml
version: '3.8'

services:
  db:
    image: mariadb:10.11
    container_name: glpi_db
    restart: unless-stopped
    environment:
      # DADOS APENAS PARA TESTE. NÃO USE EM PRODUÇÃO!
      MYSQL_ROOT_PASSWORD: secret_root_password
      MYSQL_DATABASE: glpidb
      MYSQL_USER: glpiuser
      MYSQL_PASSWORD: sua_senha_segura
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mariadb-admin", "ping", "-h", "localhost", "-u", "root", "--password=secret_root_password"]
      interval: 10s
      timeout: 5s
      retries: 5

  glpi:
    image: glpi/glpi:latest
    container_name: glpi_app
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - glpi_data:/var/www/html
    depends_on:
      db:
        condition: service_healthy

volumes:
  db_data:
  glpi_data:
```

### Detalhamento dos Componentes:

* **`db` (MariaDB):** Armazena a base relacional (chamados, ativos, usuários). O `healthcheck` garante que o GLPI só inicie quando o banco de dados estiver 100% pronto para aceitar conexões.
* **`glpi` (Aplicação):** Contém o Apache e o interpretador PHP com o GLPI oficial. A instrução `ports: "8080:80"` mapeia o container para ser acessado via `http://localhost:8080` na máquina host.
* **`volumes` (`db_data` e `glpi_data`):** Mecanismo de persistência. Mesmo se os containers forem destruídos (`down`), os dados salvos nestes volumes continuam intactos.

---

## 4. Dicionário de Comandos do Docker (Guia Prático)

Comandos para gerenciar o ambiente via terminal integrado (ex: VS Code):

* **Iniciar o Ambiente (em segundo plano):**
  ```bash
  docker compose up -d
  ```
* **Verificar Status dos Containers:**
  ```bash
  docker compose ps
  ```
* **Acompanhar Logs em Tempo Real (Depuração):**
  ```bash
  docker compose logs -f
  ```
* **Pausar Serviços Temporariamente (Recomendado no dia a dia):**
  ```bash
  docker compose stop
  ```
* **Desligar e Remover Containers (Preserva os dados nos volumes):**
  ```bash
  docker compose down
  ```

---

## 5. Assistente de Instalação e Conexão ao Banco

Com os containers rodando, o setup final é realizado no navegador:

1. Acesse: `http://localhost:8080`
2. Selecione o idioma e aceite a licença.
3. Conecte ao Banco de Dados MariaDB preenchendo as informações do `docker-compose.yml`:
   * **Servidor SQL:** `db` *(O DNS interno do Docker traduz o nome do serviço para o IP do container do MariaDB)*
   * **Utilizador:** `glpiuser`
   * **Palavra-passe:** `sua_senha_segura`
   * **Base de dados:** `glpidb`
4. Conclua o assistente e acesse o sistema com as credenciais padrão de fábrica: `glpi` / `glpi`.

---

## 6. Histórico de Erros, Acertos e Instalação de Plugins no GLPI 12

Durante a expansão do sistema com plugins no **GLPI 12.0.1**, documentamos os desafios de estrutura de pastas, permissões de sistema e autenticação.

### 6.1. A Abordagem Inicial (Cópia Manual) e Seus Erros

A tentativa inicial consistiu em baixar arquivos do GitHub e copiá-los via `docker cp`. Isso gerou diversos erros invisíveis na interface web:

* **Nomes com Versões/Hífens:** Pastas com nomes como `glpi-formcreator-2.13.11` são ignoradas. O GLPI exige nomes limpos para os diretórios (ex: `formcreator`).
* **Estrutura Aninhada:** Subpastas duplicadas escondem o arquivo `setup.php` do motor do GLPI.
* **Permissões de Apache (`www-data`):** Arquivos copiados assumiam o dono `root`, causando negação de leitura pelo servidor web.

### 6.2. O Diagnóstico via CLI e a Arquitetura do GLPI 12

Ao acessar a consola do container (`php bin/console list`), validamos que o ambiente roda o **GLPI 12.0.1**. Esta versão introduziu um ecossistema CLI nativo robusto (`plugin:list`, `plugin:install`, `marketplace:download`), tornando o gerenciamento por terminal muito mais ágil e padronizado.

### 6.3. A Exigência da Chave de Registro (GLPI Network)

Para utilizar o ecossistema de plugins no GLPI 12 (via Marketplace ou CLI), o sistema exige uma chave de autenticação vinculada ao *GLPI Network*.

**Como Conseguir e Configurar a Chave (Totalmente Gratuita):**

1. Acesse o portal de registro oficial: [services.glpi-network.com/register](https://services.glpi-network.com/register).
2. Crie uma conta gratuita (destinada a testes e uso comunitário) e copie a **chave de registro** gerada no seu painel.
3. No painel web do GLPI, navegue em **Configurar > Geral > aba GLPI Network**, cole a chave e salve.

### 6.4. A Solução Definitiva: Instalação via CLI

Com a chave cadastrada, a instalação de qualquer plugin tornou-se automatizada via CLI (exemplo utilizando o plugin *dashboard*):

```bash
# 1. Pesquisar o nome exato do plugin no marketplace
docker exec -it glpi_app php bin/console marketplace:search dashboard

# 2. Descarregar o plugin automaticamente pelo repositório
docker exec -it glpi_app php bin/console marketplace:download dashboard

# 3. Executar o script de instalação no MariaDB
docker exec -it glpi_app php bin/console plugin:install dashboard

# 4. Ativar o plugin no sistema
docker exec -it glpi_app php bin/console plugin:activate dashboard
```

---

## 7. Boas Práticas e Comandos de Diagnóstico Úteis

* **Alteração de Senha Padrão:** Imediatamente após a instalação, altere a senha do usuário `glpi` ou crie um administrador dedicado e desative o usuário padrão de fábrica.
* **Listar Plugins via CLI:**
  ```bash
  docker exec -it glpi_app php bin/console plugin:list
  ```
* **Limpeza de Cache Interna (Resolve painéis travados ou incoerências visuais):**
  ```bash
  docker exec -it glpi_app rm -rf /var/www/html/files/_cache/*
  ```
* **Ajuste Corretivo de Permissões Recursivas (Caso mova arquivos manualmente):**
  ```bash
  docker exec -it glpi_app bash -c "chown -R www-data:www-data /var/www/html/plugins/NOME_PLUGIN && chmod -R 755 /var/www/html/plugins/NOME_PLUGIN"
  ```