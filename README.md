# Guia Definitivo: Implantando e Gerenciando o GLPI 12 via Docker e Docker Compose

> **⚠️ AVISO DE SEGURANÇA:** Este guia e os arquivos de configuração associados (como o `.env.example`) foram criados para **ambientes de teste, laboratório e estudo local**. Nunca utilize credenciais padrão ou exponha este ambiente diretamente na internet sem antes configurar senhas fortes e regras rigorosas de segurança.

## 1. Contexto e Escolha Tecnológica

Para subir o GLPI rapidamente sem a sobrecarga de gerenciar manualmente uma Máquina Virtual (VM) tradicional, optamos pelo uso do **Docker** e do **Docker Compose**.

* **Docker / Containers:** Compartilham o kernel do host, consumindo pouquíssima memória. Sobem em segundos e isolam perfeitamente a aplicação web e o banco de dados.
* **VM Clássica:** Exige alocação estática de recursos e configuração manual exaustiva de todo o ecossistema (pilha LAMP).

## 2. Preparação do Ambiente (Pré-requisitos)

Para que tudo funcione no Windows, você precisa de três ferramentas básicas:

1. **WSL (Subsistema Windows para Linux):** Abra o PowerShell como Administrador e rode o comando `wsl --install`. Depois, reinicie o computador.
2. **Docker Desktop:** Baixe e instale a partir do site oficial.
3. **Git para Windows:** Baixe e instale com as configurações padrão.

---

## 3. Passo a Passo: Como Rodar Este Projeto na Sua Máquina

Se você quer apenas baixar este ambiente e colocá-lo para funcionar rapidamente, abra o seu terminal (CMD, PowerShell ou terminal do VS Code) e siga estes 4 passos:

**1. Baixe os arquivos do projeto para o seu computador:**
```bash
git clone https://github.com/SEU_USUARIO/glpi-docker.git
```

**2. Entre na pasta que acabou de ser baixada:**
```bash
cd glpi-docker
```

**3. Crie o arquivo de senhas:**
Copie o modelo padrão para criar o seu arquivo de configuração definitivo.
```bash
cp .env.example .env
```
*(Dica: Você pode abrir o arquivo `.env` no bloco de notas e alterar as senhas do banco de dados se desejar).*

**4. Ligue o sistema:**
Certifique-se de que o Docker Desktop está aberto e rode o comando abaixo:
```bash
docker compose up -d
```
Pronto! O Docker vai baixar tudo automaticamente e subir o sistema.

---

## 4. Arquitetura do Projeto: O Arquivo `docker-compose.yml`

Nosso arquivo orquestra dois serviços, consumindo as senhas diretamente do arquivo `.env` para maior segurança:

```yaml
version: '3.8'

services:
  db:
    image: mariadb:10.11
    container_name: glpi_db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mariadb-admin", "ping", "-h", "localhost", "-u", "root", "--password=${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5

  glpi:
    image: glpi/glpi:latest
    container_name: glpi_app
    restart: unless-stopped
    ports:
      - "${GLPI_PORT}:80"
    volumes:
      - glpi_data:/var/www/html
    depends_on:
      db:
        condition: service_healthy

volumes:
  db_data:
  glpi_data:
```

## 5. Dicionário de Comandos do Docker (Guia Prático)

* `docker compose up -d`: Inicia a construção e execução dos containers em segundo plano.
* `docker compose ps`: Lista os containers ativos e suas portas.
* `docker compose logs -f`: Exibe os logs em tempo real (útil para descobrir erros).
* `docker compose stop`: Desliga os containers temporariamente sem apagar nada.
* `docker compose down`: Destrói os containers (mas os volumes com os dados do banco e arquivos continuam salvos na sua máquina).

## 6. Assistente de Instalação e Conexão ao Banco

Com os containers em execução:

1. Acesse `http://localhost:8080` (ou a porta que definiu no `.env`).
2. Na etapa de conexão com o banco de dados, preencha com:
   * **Servidor MySQL:** `db`
   * **Utilizador / Senha / Base de dados:** As credenciais que você configurou no seu arquivo `.env`.
3. Conclua o assistente e faça login com os dados padrão do GLPI (`glpi` / `glpi`).

---

## 7. Gerenciamento de Plugins no GLPI 12 (Erros, Acertos e Marketplace)

A versão 12 do GLPI introduziu mudanças estruturais. Plugins famosos do passado tornaram-se incompatíveis como extensões externas, pois suas funcionalidades foram absorvidas nativamente pelo núcleo do sistema.

Para instalar plugins modernos no GLPI 12, o uso da CLI (Linha de Comando) em conjunto com o Marketplace integrado é a melhor forma, pois evita problemas de cache e permissões de pastas.

### 7.1. Obtendo a chave de registro do Marketplace (Gratuita)

Para baixar plugins automaticamente, registre a sua instância (mesmo sendo local/teste):

1. Acesse o portal [services.glpi-network.com/register](https://services.glpi-network.com/register) e crie uma conta gratuita.
2. Copie a **Chave de Registro**.
3. No painel web do GLPI, vá até **Configurar > Geral > GLPI Network** e insira sua chave.

### 7.2. O Método Definitivo para Instalar Plugins (via CLI)

Abra o terminal e execute os comandos abaixo diretamente:

```bash
# 1. Pesquisar o plugin no catálogo
docker exec -it glpi_app php bin/console marketplace:search NOME_DO_PLUGIN

# 2. Fazer o download do plugin
docker exec -it glpi_app php bin/console marketplace:download NOME_DO_PLUGIN

# 3. Instalar o plugin no sistema
docker exec -it glpi_app php bin/console plugin:install NOME_DO_PLUGIN

# 4. Ativar o plugin para uso
docker exec -it glpi_app php bin/console plugin:activate NOME_DO_PLUGIN
```

### 7.3. Troubleshooting (Resolução de Problemas Comuns)

* **Erro de Namespace no Terminal:**
  * No GLPI 12, use sempre o prefixo curto `plugin:` (ex: `plugin:list`). Nunca use `glpi:plugin:list`.
* **Plugin copiado manualmente não aparece na listagem web:**
  * Isso acontece se o usuário `root` ficar como dono da pasta ou se a cache travar. Para resolver e dar acesso ao servidor web, rode:
    ```bash
    docker exec -it glpi_app bash -c "chown -R www-data:www-data /var/www/html/plugins/NOME_DO_PLUGIN && chmod -R 755 /var/www/html/plugins/NOME_DO_PLUGIN && rm -rf /var/www/html/files/_cache/*"
    ```