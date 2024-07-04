# Installing MySQL on an EC2 Instance Using Ansible

## Overview

This documentation provides a comprehensive guide to installing and configuring MySQL on an Amazon EC2 instance using Ansible. The process involves setting up an EC2 instance, installing Ansible on your local machine, writing an Ansible playbook, executing it to ensure MySQL is installed and secured, and verifying the installation.

![alt text](https://raw.githubusercontent.com/AhnafNabil/Ansible-Labs/main/Ansible-Mysql/images/diagram-mysql-ansible.png)

## Prerequisites

1. **EC2 Instance:**
   - An Ubuntu-based EC2 instance running on AWS.
   - Security group allowing inbound SSH traffic from your IP address.

2. **Local Machine:**
   - Ansible installed on your local machine.
   - SSH access to the EC2 instance using a key pair.
   - Python and necessary libraries installed.

## Steps

### Step 1: Set up Your EC2 Instance

Ensure you have an EC2 instance running and accessible via SSH. Make sure the security group associated with your instance allows inbound SSH traffic from your IP address.

### Step 2: Install Ansible

If Ansible is not already installed on your local machine, you can install it using `pip`:

```bash
sudo apt update
sudo apt install ansible
```

Verify the installation by checking the Ansible version:

```bash
ansible --version
```

### Step 3: Create an Ansible Playbook

Create a directory for your Ansible project:

```bash
mkdir ansible-mysql
cd ansible-mysql
```

Create an inventory file named `hosts.ini` to specify your EC2 instance details:

```ini
[mysql_servers]
ec2-instance ansible_host=<EC2_PUBLIC_IP> ansible_user=ubuntu ansible_ssh_private_key_file=/path/to/your-key.pem
```

Replace `<EC2_PUBLIC_IP>` with the public IP address of your EC2 instance and `/path/to/your-key.pem` with the path to your SSH private key file.

Create a playbook file named `install_mysql.yml`:

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

### Step 4: Run the Ansible Playbook

Run the playbook using the following command:

```bash
ansible-playbook -i hosts.ini install_mysql.yml
```

This command will connect to your EC2 instance and perform the steps defined in the playbook to install and configure MySQL.

![alt text](https://raw.githubusercontent.com/AhnafNabil/Ansible-Labs/main/Ansible-Mysql/images/ansible-mysql-01.png)

### Step 5: Verify the Installation

1. **Connect to the EC2 instance:**

   ```bash
   ssh -i /path/to/your-key.pem ubuntu@<EC2_PUBLIC_IP>
   ```

2. **Verify MySQL is running:**

   Check the status of the MySQL service:

   ```bash
   sudo systemctl status mysql
   ```

   The output should indicate that MySQL is active and running.

   ![alt text](https://raw.githubusercontent.com/AhnafNabil/Ansible-Labs/main/Ansible-Mysql/images/ansible-mysql-02.png)

3. **Log in to the MySQL shell:**

    Try logging in to the MySQL shell using the root user and the password you set in the Ansible playbook:

    ```bash
    mysql -u root -p
    ```

    You will be prompted to enter the root password. If MySQL is installed correctly, you should be able to log in and see the MySQL prompt:

    ```bash
    mysql>
    ```
  
4. **Check the MySQL version:**

   While in the MySQL shell, you can check the installed MySQL version with the following command:

   ```sql
   SELECT VERSION();
   ```

   This will display the version of MySQL that is installed.

   ![alt text](https://raw.githubusercontent.com/AhnafNabil/Ansible-Labs/main/Ansible-Mysql/images/ansible-mysql-03.png)


## Explanation of Playbook

1. **Update apt-get and Install Necessary Packages:**
   - The `apt` module is used to install `mysql-server`, `python3-mysqldb`, and other necessary packages.

2. **Start MySQL Service:**
   - The `service` module ensures the MySQL service is started and enabled to start on boot.

3. **Set MySQL Root Password and Secure Installation:**
   - The `mysql_user` module sets the root password and grants necessary privileges.
   - This step also removes the default `test` database and any anonymous users to improve security.

4. **Disallow Root Login Remotely:**
   - The `mysql_user` module restricts root login to local connections only.

5. **Verify MySQL Installation:**
   - The `shell` module runs `mysql --version` to verify the MySQL installation.
   - The `debug` module prints the MySQL version to confirm the installation.

## Additional Tips

- **Security Group:** Ensure the security group associated with your EC2 instance allows inbound SSH traffic from your IP address.
- **Permissions:** Keep the permissions of your private key secure (`chmod 600 /path/to/your-key.pem`).

By following these steps, you will successfully install and configure MySQL on your EC2 instance using Ansible. Adjust the playbook as needed based on your specific requirements and environment.