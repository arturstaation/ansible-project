# Trabalho em Duplas - Projeto Ansible

## Enunciado
https://docs.google.com/document/d/1lgIukApy2OdLAInv8059RumkBfh_oqPz/edit?usp=sharing&ouid=115609016009914963164&rtpof=true&sd=true

## Visão geral
Este projeto automatiza a criação de uma stack AWS com Ansible, incluindo:
- RDS MySQL
- EC2 Web com Nginx (single instance)
- ALB + Target Group + Auto Scaling Group (deployment escalável)
- Security Groups e RDS Subnet Group
- Rolling update via Launch Template + ASG

Repositório: https://github.com/arturstaation/ansible-project

Também há um guia passo a passo em DOCX (incluído abaixo neste README) com as instruções de instalação e execução.

## Estrutura do repositório
- inventories/
  - Inventários e variáveis por ambiente (ex.: região AWS, nomes de VPC/SGs).
- playbooks/
  - Playbooks de alto nível para provisionamento e atualização (RDS, EC2 Web, ALB/ASG).
- roles/
  - rds_mysql: cria/garante RDS MySQL (subnet group, SG, instância, coleta endpoint).
  - ec2_web: cria/garante EC2 com Nginx e site estático via user_data.
  - alb_asg_web: cria ALB, TG, Listener, Launch Template, ASG e executa rolling updates.
  - web_update: aplica atualizações em instâncias Web existentes.
- ansible.cfg
  - Configurações do Ansible (coleções, etc).
- requirements.yml
  - Lista de coleções/roles necessárias (amazon.aws, community.aws).

## Pré-requisitos
- Conta AWS com permissões para EC2, ELBv2, AutoScaling, RDS, VPC e tagging.
- Ansible instalado na máquina de controle.
- AWS CLI configurada (perfil padrão ou variáveis de ambiente).
- Python 3, boto3/botocore no host de controle.

Instalação das coleções Ansible:
```
ansible-galaxy install -r requirements.yml
```

## Variáveis importantes
- aws_region (ex.: us-east-1)
- use_default_vpc (true/false)
- tags_common: Project, Env, Owner
- RDS: rds_identifier, rds_instance_class, rds_username, rds_password, rds_db_name, rds_subnet_group_name, rds_vpc_security_group_name
- Web/EC2: ec2_name, ec2_instance_type, ec2_key_name, ec2_ami, security_group_name, tpl_index_src, tpl_nginx_src
- ALB/ASG: alb_name, tg_name, alb_listener_port, health_check_path, lt_name, asg_name, desired_capacity, min_size, max_size

Defina em group_vars/host_vars, inventários ou com -e.

## Playbooks típicos
- Preparar ambiente local (instalar boto3, AWS CLI, coleções):
  - Ex.: playbooks/prepare.yml (o nome pode variar de acordo com sua árvore)
- Provisionar tudo (RDS + ALB + ASG):
  - playbook que chama roles rds_mysql e alb_asg_web
- Provisionar RDS + EC2 Web (single):
  - playbook que chama roles rds_mysql e ec2_web
- Atualizar Web (single ou ASG rolling update):
  - playbooks para “update_site”/“rolling_update”

Confira os arquivos reais em playbooks/.

## Rolling update (ALB/ASG)
- Versiona o Launch Template com novo user_data (conteúdo do site/config).
- Força replace_all_instances no ASG em batches, aguardando health check ELB/ALB.

### Observações sobre a tag Name nas instâncias do ASG
- O módulo community.aws.ec2_launch_template não aceita tag_specifications.
- Use a tag Name no ASG com propagate_at_launch: true e faça Instance Refresh para propagar às novas instâncias.
- Alternativa: usar o módulo amazon.aws.ec2_launch_template (suporta tag_specifications).

### Segurança
- Evite cidr 0.0.0.0/0 em produção:
  - Instances: preferir permitir HTTP somente do SG do ALB.
  - RDS: permitir 3306 somente do SG das instâncias Web/ASG.
- Considere HTTPS no ALB (ACM + listener 443).

#### Guia passo a passo (do seu DOCX “Trabalho em Dupla”)
Abaixo, o conteúdo do seu guia para instalar Ansible e executar os playbooks, já integrado ao README:

1) Provisionar a máquina de Ansible
- Crie 1 instância EC2 para o Ansible (ex.: t2.small).
- Copie sua chave privada para dentro do laboratório, ex.: vi Chave.pem
- Permissões corretas na chave:
```
chmod 400 "Chave.pem"
```
- Conecte na máquina do Ansible e eleve para root:
```
sudo su -
```

2) Instalar Ansible (Ubuntu/Debian)
```
apt-get install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```

3) Testar instalação do Ansible
```
ansible localhost -c local -i ~/hosts -m ping
```

4) Clonar o repositório
```
git clone https://github.com/arturstaation/ansible-project.git
cd ansible-project
```

5) Preparar ambiente com playbook
- Execute o playbook de preparo:
```
ansible-playbook /root/ansible-project/playbooks/prepare_env.yml
```

6) Configurar AWS CLI
```
aws configure
```
- Consulte “AWS Details” no seu ambiente e clique em “SHOW” do AWS CLI para obter as credenciais.
- Quando pedir a região, informe:
```
us-east-1
```

7) Adaptação do nome da Chave

Antes do próximo passo temos que trocar essa variável para key cadastrada no provimento da máquina

ansible-project/inventories/group_vars/
sudo vi all.yml

ec2_key_name: "TrabalhoDupla"

Substitua “TrabalhoDupla” pela key cadastrada.



8) Provisionar Banco + Web
- Execute:
```
cd ansible-project
ansible-playbook /root/ansible-project/playbooks/provision_all.yml
```

9) Acessar o site
- Use a URL exibida como access_url no output (geralmente o DNS do ALB ou o DNS público da EC2, conforme o playbook usado).

10) Atualizar a Web (novo conteúdo)
- Execute:
```
cd ansible-project
ansible-playbook /root/ansible-project/playbooks/update_site.yml
```

Observação: Os caminhos playbook/... informados no guia refletem sua estrutura. Se os arquivos estiverem em playbook/ em vez de playbooks/, ajuste os caminhos conforme a árvore do repositório.

## Guia Google Docs
https://docs.google.com/document/d/19RbUZsWIamlCofgnTeavYzGTJeQJz-mkq1_Z5cWCLjM/edit?usp=sharing
