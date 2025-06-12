# Documentação do Projeto WordPress AWS

## Sumário

- [1. Principais Objetivos](#1-principais-objetivos)
- [2. Serviços AWS Utilizados](#2-serviços-aws-utilizados)
- [3. Ferramentas](#3-ferramentas)
- [4. Criação da VPC](#4-criação-da-vpc)
- [5. Grupos de Segurança](#5-grupos-de-segurança)
- [6. Configuração do RDS (MySQL)](#6-configuração-do-rds-mysql)
- [7. Configuração do EFS](#7-configuração-do-efs)
- [8. Instâncias EC2 e Deploy Automatizado](#8-instâncias-ec2-e-deploy-automatizado)
- [9. Load Balancer](#9-load-balancer)
- [10. Auto Scaling](#10-auto-scaling)
- [11. Validação do Sistema de Arquivos](#11-validação-do-sistema-de-arquivos)
- [12. Script de Monitoramento](#12-script-de-monitoramento)
- [13. Acesso ao Projeto](#13-acesso-ao-projeto)
- [14. Referências](#14-referências)

---

## 1. Principais Objetivos

- **Instalação e Configuração:**  
  - Instalar e configurar Docker nas instâncias EC2.
  - Realizar instalação automatizada utilizando o arquivo `user_data.sh`.
- **Deploy da Aplicação WordPress:**  
  - Implementar o container da aplicação utilizando Docker Compose.
  - Configurar um banco de dados RDS MySQL para armazenar os dados do WordPress.
- **Configuração de Armazenamento e Balanceamento de Carga:**  
  - Utilizar EFS (Elastic File System) para armazenar arquivos estáticos do WordPress.
  - Configurar um Load Balancer (Classic Load Balancer da AWS) para distribuir o tráfego de entrada.

---

## 2. Serviços AWS Utilizados

- **VPC**
- **RDS (Banco de Dados MySQL)**
- **EFS**
- **Instâncias EC2**
- **Load Balancer**
- **Auto Scaling**

---

## 3. Ferramentas

- **AWS**
- **Shell Script**
- **Linux**
- **Docker**

---

## 4. Criação da VPC

- **Bloco CIDR IPv4:** `10.0.0.0/16`
- **Número de Zonas de Disponibilidade (AZs):** 2
- **Sub-redes:** 2 públicas e 2 privadas
- **Gateway NAT:** 1 por AZ

---

## 5. Grupos de Segurança

- **sgGroup-loadbalancer:**  
  - HTTP / HTTPS liberado para IPv4
- **sgGroup-ec2:**  
  - HTTP / HTTPS liberado para Load Balancer  
  - SSH liberado para qualquer IP
- **sgGroup-rds:**  
  - MySQL/Aurora liberado para sgGroup-ec2
- **sgGroup-efs:**  
  - NFS liberado para sgGroup-ec2

---

## 6. Configuração do RDS (MySQL)

1. **Criar Grupo de Sub-redes Privadas:**
   - Acesse RDS > Grupos de sub-redes > Criar Grupo.
   - Nome do Grupo, Descrição, VPC criada, selecione as zonas de disponibilidade e sub-redes privadas.
2. **Configurações do RDS:**
   - Tipo: MySQL
   - Instância: db.t2.micro
   - Backup e Criptografia: Desativados para testes
   - VPC: Selecionar a criada
   - Grupo de sub-redes: Selecionar o criado acima
   - Acesso público: Não permitir
   - Grupo de Segurança: sgGroup-rds
   - Nome do banco inicial: `wordpress`
   - Salve o endpoint/IP gerado para uso no `user_data.sh`

---

## 7. Configuração do EFS

1. **Criar EFS:**
   - Nome: `meuEFS`
   - VPC: Selecionar a criada
   - Zonas de disponibilidade: Sub-redes privadas 1 e 2
   - Grupo de segurança: sgGroup-efs
2. **Instalar EFS Utils nas EC2:**
   ```bash
   sudo apt-get update
   sudo apt-get -y install git binutils rustc cargo pkg-config libssl-dev
   git clone https://github.com/aws/efs-utils
   cd efs-utils
   ./build-deb.sh
   sudo apt-get -y install ./build/amazon-efs-utils*deb
   ```
3. **Montar o EFS:**
   ```bash
   sudo mkdir -p /mnt/efs
   sudo mount -t efs -o tls fs-XXXXXXXX:/ /mnt/efs
   ```
   > Substitua `fs-XXXXXXXX` pelo ID do EFS.

---

## 8. Instâncias EC2 e Deploy Automatizado

- **Nome e tags:** Seguir padrão da equipe.
- **Sistema operacional:** Ubuntu.
- **Tipo de instância:** Padrão.
- **Par de chaves:** Criar ou reutilizar.
- **Sub-redes:**  
  - Instância 1: Sub-rede privada 1  
  - Instância 2: Sub-rede privada 2
- **Atribuir IP público automaticamente:** Habilitado.
- **Grupo de segurança:** sgGroup-ec2
- **Configurações avançadas:** Adicione o `user_data.sh`.

---

## 9. Load Balancer

- **Tipo:** Classic Load Balancer
- **Nome:** MyLoadBalancer
- **Mapeamento de rede:** Sub-redes públicas
- **Grupo de segurança:** sgGroup-loadbalancer
- **Caminho de ping:** `/wp-admin/install.php` (espera-se retorno com status 200)
- **Instâncias:** Selecionar as duas instâncias privadas criadas

---

## 10. Auto Scaling

- **Modelo de Execução (Template):**
  - Tipo de instância: t2.micro
  - Tags e User Data: Mesmos das instâncias EC2 anteriores
  - Zonas de disponibilidade: Sub-redes privadas
  - Integração: Load Balancer existente
  - Demais configurações: Padrão

---

## 11. Validação do Sistema de Arquivos

- **Bastion Host:** Crie uma instância pública para acesso SSH à VPC.
- **Acesse a instância 1 (privada) via Bastion Host e crie um arquivo no EFS:**
  ```bash
  echo "Hello World" > /mnt/efs/helloworld.txt
  ```
- **Acesse a instância 2 (privada) e verifique o arquivo:**
  ```bash
  cat /mnt/efs/helloworld.txt
  ```
  O arquivo deve estar disponível em ambas as instâncias.

---

## 12. Script de Monitoramento

Crie um script para monitorar o status do container WordPress, uso de CPU, memória e espaço em disco. Exemplo:

````bash
# filepath: /workspaces/CompassUolProjetoWordPress/monitoramento.sh
#!/bin/bash

echo "===== Monitoramento WordPress ====="
echo "Data/Hora: $(date)"
echo

# Verifica se o container WordPress está rodando
echo "Status do container WordPress:"
docker ps --filter "name=wordpress" --format "table {{.Names}}\t{{.Status}}"
echo

# Uso de CPU e Memória
echo "Uso de CPU e Memória:"
top -b -n 1 | head -n 10
echo

# Espaço em disco
echo "Espaço em disco:"
df -h /mnt/efs
echo

# Logs do container WordPress (últimas 10 linhas)
echo "Logs recentes do WordPress:"
docker logs --tail 10 wordpress
`````

---

## 13. Acesso ao Projeto

Após o deploy, acesse o Load Balancer pelo navegador:  
```
http://<DNS_DO_LOAD_BALANCER>
```

Siga o assistente do WordPress para finalizar a configuração.

---

## 14. Referências

- [WordPress](https://wordpress.org/support/)
- [AWS EC2](https://docs.aws.amazon.com/ec2/)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)
