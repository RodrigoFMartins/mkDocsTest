---
tags:
  - Linux
---
Para realizar a instalação do Ansible numa nova máquina (master), devem realizar-se os seguintes passos:
1. Criar um user "ansible" e gerar chaves SSH.
	1. Criar o user "ansible":
	 ```bash
	  useradd -r -m -c "Ansible User" ansible
	  ```
	2. Gerar chaves:
	```bash
	  ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
	  ```
2. Permitir que apenas membros do grupo "ansible" façam `sudo` para o user "ansible" sem precisar de password.
    1. Adicionar o user e grupo aos sudoers:
        1. Editar o arquivo sudoers:
            ```bash
            visudo
            ```
        2. Adicionar as seguintes regras:
        ```
            ## ansible 
            %ansible ALL=(ansible) NOPASSWD: ALL
            
            ## Ansible User
            ansible ALL=(ALL)  NOPASSWD:ALL
        ```
3. Caso se queira adicionar outros users ao grupo do Ansible:
    ```bash
    usermod -aG ansible "user"
    ```
4. Instalar o Ansible:
    ```bash
    sudo dnf install epel-release -y
    sudo dnf install ansible -y
    ```
5. Verificar a instalação do Ansible:
    ```shell
    ansible --version
    ```
# Criação do user ansible nos slaves
## Create the user
```bash
adduser ansible --comment "Ansible User"
```
```bash
passwd ansible
```
## Create the .ssh directory
```bash
mkdir -p /home/ansible/.ssh
```
## Set ownership and permissions for .ssh
```bash
chown ansible:ansible /home/ansible/.ssh
```
```bash
chmod 700 /home/ansible/.ssh
```
## Create the authorized_keys file
```bash
touch /home/ansible/.ssh/authorized_keys
```
## Set ownership and permissions for authorized_keys
```bash
chown ansible:ansible /home/ansible/.ssh/authorized_keys
```
```bash
chmod 600 /home/ansible/.ssh/authorized_keys
```
## Adicionar user ao sudoers 
```bash
visudo
```
```
##Ansible User
ansible    ALL=(ALL)       NOPASSWD:ALL
```
## Adicionar chave pub ao authorized_keys
```bash
vi /home/ansible/.ssh/authorized_keys 
```

_______________________________________________________
# Caso de uso criação de utilizadores

Relativamente ao processo de criação de users em Linux (RHEL e Ubuntu) de forma automatizada.

A criação de users nos sistemas é um processo repetitivo e demorado, realizado em vários passos. 
## Passos RHEL
6. sudo -i
7. adduser user1 --comment "Utilizador1 Utilizador1"
8. usermod -aG wheel user1
9. passwd
## Passos Ubuntu 
10. sudo -i
11. adduser user1 --comment "Utilizador1 Utilizador1"
12. usermod -aG sudo user1

Tendo ainda em ambos os casos de ser criada a pasta .ssh, o ficheiro authorized_keys e colocada a chave publica do utilizador nesse ficheiro.

A execução manual deste processo requer a aplicação de comandos em dezenas de máquinas, uma a uma, o que se traduz numa abordagem ineficiente e com limitações significativas em termos de escalabilidade.

Para otimizar este processo, foi implementada a sua automatização através do uso do **Ansible**, permitindo a definição clara das etapas a serem executadas. os dados para os novos users a serem criados são colocados num ficheiro de configuração que será depois utilizado para criar os mesmos nas diversas máquinas.

### Playbooks Ansible
#### Iterador de users
```yaml
---
- name: Create multiple users 
  hosts: all
  become: yes
  gather_facts: true

  vars_files:
    - user_config.yml

  tasks:
    - name: Loop through users in the input file
      ansible.builtin.include_tasks: create_user.yaml
      loop: "{{ users }}"
      loop_control:
        loop_var: user
      no_log: true
```
Este playbook itera sobre cada user no ficheiro de configuração e para cada um executa o playbook seguinte:
#### Main playbook (create_user.yaml)
```yaml
---
- name: Check if the user exists using the 'id' command
  ansible.builtin.command:
    cmd: "id -u {{ user.username }}"
  register: user_check
  failed_when: false
  changed_when: false

- name: Determine user groups based on OS
  set_fact:
    user_groups: |
      {% if ansible_distribution == 'Rocky' %}
      {{ user.rhel_groups | split(',') if user.rhel_groups != '' else [] }}
      {% elif ansible_distribution == 'Ubuntu' %}
      {{ user.ubuntu_groups | split(',') if user.ubuntu_groups != '' else [] }}
      {% else %}
      []
      {% endif %}

- name: Skip user creation if the user exists
  ansible.builtin.debug:
    msg: "User {{ user.username }} already exists. Skipping creation."
  when: user_check.rc == 0

- name: Ensure the user is created
  ansible.builtin.user:
    name: "{{ user.username }}"
    comment: "{{ user.fullname }}"
    groups: "{{ user_groups | join(',') if user_groups | length > 0 else [] }}"
    append: yes
    state: present
    create_home: yes
  when: user_check.rc != 0

- name: Set the user's password
  ansible.builtin.command: chpasswd
  args:
    stdin: "{{ user.username }}:{{ user.user_password }}"
  no_log: true
  when: user_check.rc != 0

- name: Ensure .ssh directory exists
  ansible.builtin.file:
    path: "/home/{{ user.username }}/.ssh"
    state: directory
    owner: "{{ user.username }}"
    group: "{{ user.username }}"
    mode: '0700'
  when: user_check.rc != 0

- name: Copy public key to authorized_keys
  ansible.builtin.copy:
    src: "{{ user.public_key_file }}"
    dest: "/home/{{ user.username }}/.ssh/authorized_keys"
    owner: "{{ user.username }}"
    group: "{{ user.username }}"
    mode: '0600'
  when: user_check.rc != 0

- name: Ensure correct permissions on the user's home directory
  ansible.builtin.file:
    path: "/home/{{ user.username }}"
    state: directory
    owner: "{{ user.username }}"
    group: "{{ user.username }}"
    mode: '0700'
  when: user_check.rc != 0
```
Este playbook realiza diversas tarefas relacionadas à gestão de utilizadores no sistema. A seguir, apresenta-se o fluxo de execução detalhado:

13. **Verificação da existência do utilizador**
    - O playbook verifica se o utilizador já existe no sistema, validando a presença de um ID associado ao nome de utilizador (`username`).
    - Caso o utilizador já exista, nenhuma ação de criação é realizada, e é emitida uma mensagem informativa indicando que o utilizador com esse `username` já existe.
14. **Obtenção de grupos**
    - Dependendo do sistema operativo em uso (RHEL ou Ubuntu), são identificados os grupos adequados para associação ao utilizador.
15. **Criação do utilizador**
    - Utilizando a biblioteca padrão para gestão de utilizadores, é criado um novo utilizador com os parâmetros definidos no ficheiro de configuração, incluindo:
        - `username` (nome de utilizador);
        - `comment` (descrição opcional);
        - `groups` (grupos aos quais o utilizador será adicionado).
    - Adicionalmente, garante-se a criação de um diretório `home` associado ao utilizador.
16. **Definição da palavra-passe**
    - A palavra-passe do utilizador recém-criado é definida utilizando o comando `chpasswd`, evitando a necessidade de confirmação manual.
    - Para segurança adicional, o comando é executado de forma a não deixar registos no histórico de comandos.
17. **Configuração do diretório `.ssh`**
    - É criado o diretório `.ssh` no diretório `home` do utilizador, com as seguintes configurações:
        - Proprietário (`owner`) e grupo (`group`) definidos como o próprio utilizador;
        - Permissões ajustadas para `0700`.
18. **Adição da chave pública**
    - A chave pública especificada no ficheiro de configuração é copiada para o ficheiro `authorized_keys` dentro do diretório `.ssh`.
    - São garantidas as seguintes condições:
        - O ficheiro `authorized_keys` possui como `owner` e `group` o utilizador;
        - As permissões do ficheiro são ajustadas para `0600`.
19. **Validação final**
    - Verifica-se que o diretório `home` do utilizador possui as permissões corretas, conforme especificado nos requisitos.
## Execução do playbook Ansible
Para realizar a execução, é necessário apenas que o Ansible esteja instalado na máquina Master.  
A definição das máquinas onde o Ansible irá executar o playbook é feita através de um Ansible Inventory, que armazena todos os endereços IP necessários para estabelecer as ligações SSH.

![picture](/images/firstImage.png)

Além disso, o Ansible requer informações adicionais, para cada utilizador a ser criado nas máquinas alvo e a localização da chave privada necessária para estabelecer a ligação SSH. Os parâmetros para a ligação ssh são configurados no ficheiro `ansible.cfg`.

![picture](/images/secondImage.png)

Já os dados dos users a serem criados devem estar no ficheiro `user_config.yml` .

![picture](/images/thirdImage.png)

Todos os ficheiros mencionados devem estar localizados no mesmo diretório do projeto.

Nas máquinas onde o Ansible será executado, é apenas necessário garantir que o utilizador especificado no ficheiro `ansible.cfg`, "ansible" exista e possua permissões para executar comandos com `sudo` sem a necessidade de introduzir a palavra-passe.


==Exemplo de comando para execução do playbook:==
```bash
	ansible -i hosts all create_multiple_users.yaml
```
Sendo que neste caso estamos a executar o playbook create_multiple_users.yml, com os parâmetros colocados no ficheiro de configuração e para todas as máquinas (all) no inventário "hosts".
