# Automating MySQL Installation on an EC2 Instance Using Ansible and GitHub Actions

This guide provides a detailed step-by-step approach to automating the installation and configuration of MySQL on an Amazon EC2 instance using Ansible and GitHub Actions. The automation process includes setting up an EC2 instance, writing an Ansible playbook, and creating a GitHub Actions workflow to execute the playbook.

## Prerequisites

1. **EC2 Instance:**
   - An Ubuntu-based EC2 instance running on AWS.
   - Security group allowing inbound SSH traffic from your IP address.

2. **Local Machine:**
   - Ansible installed on your local machine.
   - SSH access to the EC2 instance using a key pair.
   - Python and necessary libraries installed.

3. **GitHub Repository:**
   - A GitHub repository to store your Ansible playbook and workflow files.
   - GitHub Secrets configured to store your SSH private key.

## Steps

### Step 1: Set up Your EC2 Instance

Ensure you have an EC2 instance running and accessible via SSH. Make sure the security group associated with your instance allows inbound SSH traffic from your IP address.

### Step 2: Create Ansible Playbook and Inventory File

1. **Create a directory for your Ansible project:**

    ```bash
    mkdir ansible-mysql
    cd ansible-mysql
    ```

2. **Create an inventory file named `hosts.ini`:**

    ```ini
    [mysql_servers]
    ec2-instance ansible_host={{ EC2_PUBLIC_IP }} ansible_user=ubuntu ansible_ssh_private_key_file={{ SSH_PRIVATE_KEY_PATH }}
    ```

    Replace `<EC2_PUBLIC_IP>` with the public IP address of your EC2 instance and `/path/to/your-key.pem` with the path to your SSH private key file.

3. **Create a playbook file named `install_mysql.yml`:**

    ```yaml
    ---
    - name: Install MySQL on EC2 instance
      hosts: mysql_servers
      become: yes
      tasks:
        - name: Update apt-get and install necessary packages
          apt:
            name: "{{ item }}"
            state: present
          loop:
            - mysql-server  # MySQL Server
            - python3-mysqldb  # MySQL Python library for Python 3.x
            - mysql-client   # MySQL Client (optional but useful for management tasks)
            - libmysqlclient-dev  # MySQL development files (optional for certain applications)
            - python3-pip  # Python package installer (optional but useful for installing Python packages)

        - name: Start MySQL service
          service:
            name: mysql
            state: started
            enabled: yes

        - name: Set MySQL root password and secure installation
          mysql_user:
            name: root
            password: "your_root_password"
            host_all: yes
            login_unix_socket: /var/run/mysqld/mysqld.sock
            priv: '*.*:ALL,GRANT'
            state: present

        - name: Remove test database and access to it
          mysql_db:
            name: test
            state: absent
            login_user: root
            login_password: "your_root_password"

        - name: Remove anonymous MySQL users
          mysql_user:
            name: ''
            host_all: yes
            state: absent
            login_user: root
            login_password: "your_root_password"

        - name: Disallow root login remotely
          mysql_user:
            name: root
            host: "{{ item }}"
            state: absent
            login_user: root
            login_password: "your_root_password"
          with_items:
            - "{{ ansible_default_ipv4.address }}"
            - "::1"
            - "127.0.0.1"

        - name: Verify MySQL installation
          shell: mysql --version
          register: mysql_version

        - name: Print MySQL version
          debug:
            var: mysql_version.stdout
    ```

    Replace `"your_root_password"` with a secure password of your choice.

### Step 3: Create a GitHub Actions Workflow File

1. **Create a directory named `.github/workflows` in the root of your repository:**

    ```bash
    mkdir -p .github/workflows
    ```

2. **Inside this directory, create a file named `ansible-mysql.yml`:**

    ```yaml
    name: Deploy MySQL using Ansible

    on:
    push:
        branches:
        - main
    workflow_dispatch:

    jobs:
    deploy:
        runs-on: ubuntu-latest

        steps:
        - name: Checkout repository
        uses: actions/checkout@v2

        - name: Set up Python
        uses: actions/setup-python@v2
        with:
            python-version: '3.x'

        - name: Install Ansible
        run: |
            sudo apt update
            sudo apt install ansible -y

        - name: Set up SSH Key and known_hosts
        env:
            SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
            EC2_PUBLIC_IP: ${{ secrets.EC2_PUBLIC_IP }}
        run: |
            mkdir -p ~/.ssh
            echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
            chmod 600 ~/.ssh/id_rsa
            ssh-keyscan -H $EC2_PUBLIC_IP >> ~/.ssh/known_hosts

        - name: Replace placeholders in inventory file
        run: |
            sed -i "s/{{ EC2_PUBLIC_IP }}/${{ secrets.EC2_PUBLIC_IP }}/g" Ansible-Mysql-Github-Actions/hosts.ini
            sed -i "s|{{ SSH_PRIVATE_KEY_PATH }}|~/.ssh/id_rsa|g" Ansible-Mysql-Github-Actions/hosts.ini

        - name: Run Ansible Playbook
        run: |
            ansible-playbook -i Ansible-Mysql-Github-Actions/hosts.ini Ansible-Mysql-Github-Actions/install_mysql.yml
    ```

### Step 4: Store the SSH Key as a GitHub Secret

1. **Go to your repository on GitHub.**
2. **Navigate to Settings > Secrets and variables > Actions.**
3. **Click on "New repository secret".**
4. **Add the following secrets:**

   - **Name:** `SSH_PRIVATE_KEY`

     **Value:** Paste the content of your SSH private key.
   - **Name:** `EC2_PUBLIC_IP`
   
     **Value:** The public IP address of your EC2 instance.

### Step 5: Run the Workflow

- **Push to Main Branch:** Whenever you push changes to the main branch, the workflow will execute and run the Ansible playbook.
- **Manual Trigger:** You can also manually trigger this workflow from the GitHub Actions tab in your repository.

## Verification

1. **Connect to the EC2 instance:**

    ```bash
    ssh -i /path/to/your-key.pem ubuntu@<EC2_PUBLIC_IP>
    ```

2. **Verify MySQL is running:**

    ```bash
    sudo systemctl status mysql
    ```

    The output should indicate that MySQL is active and running.

3. **Log in to the MySQL shell:**

    ```bash
    mysql -u root -p
    ```

    Enter the root password set in the Ansible playbook.

4. **Check the MySQL version:**

    ```sql
    SELECT VERSION();
    ```

    This will display the version of MySQL that is installed.

By following these steps, you will automate the process of installing and configuring MySQL on your EC2 instance using Ansible and GitHub Actions. Adjust the playbook and workflow file as needed based on your specific requirements and environment.