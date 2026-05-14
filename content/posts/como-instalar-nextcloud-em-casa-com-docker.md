---
title: "Como instalar o Nextcloud em casa com Docker — seu próprio Google Drive gratuito"
date: 2026-05-13
draft: false
tags: ["linux", "nextcloud", "docker", "self-hosted", "ubuntu"]
---

Não sei você, mas a ideia de guardar os meus arquivos em plataformas de armazenamento na nuvem nunca me pareceu uma boa — por dois motivos. Primeiro: não quero pagar mensalidade por um serviço que posso ter de graça. Segundo: não me sinto confortável mantendo os meus arquivos em servidores espalhados pelo mundo, nas mãos de empresas que eu não conheço.

Então decidi buscar uma solução que me desse o mesmo serviço, sem pagar nada e com os arquivos guardados aqui em casa, no meu próprio HD. Foi aí que cheguei no **Nextcloud** — uma plataforma de código aberto e completamente gratuita.

É exatamente isso que vamos instalar hoje, passo a passo. No final você vai ter o seu próprio servidor de arquivos funcionando, simples de usar, acessível pelo celular e pelo computador — e o melhor de tudo, sem gastar um centavo.

> **Sobre a distro:** Vamos usar o Ubuntu como base por ser o mais comum. Se você usa outra distribuição, os comandos mudam um pouco na parte do Docker — vou deixar o link da documentação oficial na descrição. Depois que o Docker estiver instalado, **todo o resto do tutorial é exatamente igual** em qualquer distro.

---

## Parte 1 — Instalando o Docker

### Atualizando o sistema

Antes de qualquer coisa, vamos garantir que o sistema está atualizado:

```bash
sudo apt update && sudo apt upgrade -y
```

O `apt update` atualiza a lista de pacotes disponíveis e o `apt upgrade` aplica as atualizações. O `&&` garante que o segundo comando só roda se o primeiro terminar com sucesso.

> **Outros gerenciadores de pacotes:**
> - Fedora: `sudo dnf update -y`
> - Arch Linux: `sudo pacman -Syu`
> - openSUSE: `sudo zypper update -y`

---

### Instalando as dependências

```bash
sudo apt install ca-certificates curl gnupg -y
```

São três ferramentas essenciais:
- `ca-certificates` — garante que o sistema reconhece sites seguros
- `curl` — usado para baixar a chave oficial do Docker
- `gnupg` — verifica se o que baixamos é genuíno e não foi adulterado

---

### Configurando a chave de segurança do Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Esses três comandos juntos garantem que o Docker que vamos instalar é genuíno. O primeiro cria a pasta segura, o segundo baixa a assinatura digital do site oficial e o terceiro aplica as permissões corretas.

---

### Adicionando o repositório oficial do Docker

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Pode parecer complexo, mas não precisa se preocupar com os detalhes — ele detecta automaticamente a arquitetura do seu sistema e a versão do Ubuntu, e registra o repositório oficial do Docker. Copie e cole exatamente como está.

---

### Instalando o Docker

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Esse comando instala tudo de uma vez — o Docker Engine, a interface de linha de comando, o containerd e o Docker Compose, que é o que vamos usar para subir o Nextcloud.

---

### Permissão para rodar sem sudo

```bash
sudo usermod -aG docker $USER
su - $USER
```

O primeiro comando adiciona o seu usuário ao grupo do Docker. O segundo aplica essa permissão imediatamente sem precisar reiniciar. A partir daqui não precisamos mais do `sudo` para comandos Docker.

---

### Verificando a instalação

```bash
docker --version
docker compose version
```

Se os dois retornarem a versão instalada, o Docker está pronto.

---

## Parte 2 — Preparando o HD

### Identificando o disco

```bash
lsblk
```

Aqui precisamos de atenção. O disco onde o Linux está instalado vai aparecer com `/` ou `/boot` na coluna `MOUNTPOINTS` — esses não vamos mexer. O HD que vamos usar vai aparecer com a coluna `MOUNTPOINTS` **vazia**. Confirme também o tamanho na coluna `SIZE` para ter certeza que é o disco certo. Anote o nome dele.

---

### Formatando o disco

```bash
sudo mkfs.ext4 /dev/sdX
```

Substitua `sdX` pelo nome do disco que você identificou. Por exemplo: `sdb` ou `sdc`.

> ⚠️ **Atenção** — esse comando apaga tudo que existe no disco sem pedir confirmação. Certifique-se que é o disco certo antes de apertar Enter.

---

### Criando o ponto de montagem

```bash
sudo mkdir /mnt/nextcloud-hd
sudo mount /dev/sdX /mnt/nextcloud-hd
```

No Linux os discos não aparecem automaticamente — você precisa conectá-los a uma pasta para acessá-los. A partir desse momento tudo que for salvo em `/mnt/nextcloud-hd` será gravado diretamente no seu HD.

---

### Configurando a montagem automática

Sempre que reiniciar a máquina o disco vai perder o ponto de montagem. Para resolver isso, precisamos registrar o HD no arquivo de configuração do sistema. Primeiro descobrimos o UUID do disco:

```bash
sudo blkid /dev/sdX
```

Anote o UUID que aparecer. Agora abrimos o arquivo `fstab`:

```bash
sudo nano /etc/fstab
```

E adicionamos essa linha no final, substituindo pelo seu UUID:

```
UUID=SEU-UUID-AQUI /mnt/nextcloud-hd ext4 defaults 0 2
```

Salve com `Ctrl+O`, `Enter`, `Ctrl+X`.

---

### Testando a montagem

```bash
sudo mount -a
sudo systemctl daemon-reload
lsblk
```

Na coluna `MOUNTPOINTS` do `lsblk` deve aparecer `/mnt/nextcloud-hd`. Para confirmar que vai funcionar sempre, reinicie e rode o `lsblk` novamente — se o HD aparecer montado sem você ter feito nada, está tudo configurado corretamente.

---

## Parte 3 — Configurando o Docker Compose

### Criando a estrutura do projeto

```bash
mkdir -p ~/nextcloud
cd ~/nextcloud
```

Essa pasta é só para guardar o arquivo de configuração — seus arquivos e fotos vão todos para o HD configurado anteriormente.

---

### Criando a pasta de dados

```bash
sudo mkdir -p /mnt/nextcloud-hd/nextcloud-data
sudo chown -R www-data:www-data /mnt/nextcloud-hd/nextcloud-data
```

O segundo comando ajusta as permissões para que o Nextcloud consiga gravar os arquivos corretamente.

---

### Criando o docker-compose.yml

```bash
nano ~/nextcloud/docker-compose.yml
```

Cole o conteúdo abaixo — mas antes, **troque as senhas de exemplo por senhas fortes**:

```yaml
services:
  db:
    image: mariadb:10.11
    container_name: nextcloud-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: sua-senha-root-aqui
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: sua-senha-nextcloud-aqui
    volumes:
      - db_data:/var/lib/mysql

  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud-app
    restart: always
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: sua-senha-nextcloud-aqui
    volumes:
      - /mnt/nextcloud-hd/nextcloud-data:/var/www/html/data

volumes:
  db_data:
```

Salve com `Ctrl+O`, `Enter`, `Ctrl+X`.

---

### Subindo o Nextcloud

```bash
cd ~/nextcloud
docker compose up -d
```

Esse comando vai baixar tudo necessário e iniciar os containers — pode demorar alguns minutos dependendo da sua internet. Para confirmar que estão rodando:

```bash
docker ps
```

Se os dois containers aparecerem com status `Up`, está tudo funcionando.

---

### Primeiro acesso

Descubra o IP da sua máquina:

```bash
hostname -I
```

Abra o navegador em qualquer dispositivo na mesma rede e acesse:

```
http://SEU-IP:8080
```

Na tela de configuração inicial:

1. Crie o usuário administrador com nome e senha forte
2. Em banco de dados, troque de **SQLite** para **MySQL/MariaDB**
3. Preencha:
   - Usuário do banco: `nextcloud`
   - Senha do banco: a senha que você definiu no docker-compose
   - Nome do banco: `nextcloud`
   - Host: `db`
4. Clique em **Instalar** e aguarde

Quando terminar, você já estará no painel. **Seu Nextcloud está no ar.**

---

## Parte 4 — IP Fixo

Por padrão o roteador pode mudar o IP da sua máquina a cada reinicialização. Vamos configurar o IP como fixo para que o endereço seja sempre o mesmo.

### Coletando as informações de rede

```bash
ip a
ip route | grep default
nmcli con show
```

O primeiro mostra o IP atual, o segundo mostra o gateway (IP do roteador) e o terceiro mostra o nome da conexão.

---

### Aplicando o IP fixo

Substitua os valores pelos que você coletou:

```bash
sudo nmcli con mod "NOME-DA-CONEXAO" ipv4.addresses SEU-IP/24
sudo nmcli con mod "NOME-DA-CONEXAO" ipv4.gateway SEU-GATEWAY
sudo nmcli con mod "NOME-DA-CONEXAO" ipv4.dns "8.8.8.8 8.8.4.4"
sudo nmcli con mod "NOME-DA-CONEXAO" ipv4.method manual
sudo nmcli con up "NOME-DA-CONEXAO"
```

Para confirmar:

```bash
ip a show NOME-DA-INTERFACE
```

Na linha do IPv4 deve aparecer `valid_lft forever` — isso confirma que o IP é fixo. Reinicie e acesse novamente com o mesmo IP para confirmar.

---

## Parte 5 — Conectando o celular

Baixe o app do Nextcloud:
- **Android** — Play Store, busca por "Nextcloud"
- **iPhone** — App Store, busca por "Nextcloud"

Abra o app, toque em **Entrar**, digite o endereço do servidor:

```
http://SEU-IP:8080
```

Entre com o usuário e senha do admin — e pronto. Seu celular está conectado ao seu próprio servidor.

Nas configurações do app você pode ativar o **backup automático de fotos** — assim toda foto que você tirar já vai direto para o seu HD sem precisar fazer nada manualmente.

---

Seus dados estão no seu HD, na sua casa, sob o seu controle. Sem big techs, sem mensalidade, sem ninguém tendo acesso às suas fotos e arquivos.

Se tiver alguma dúvida, deixa nos comentários.
