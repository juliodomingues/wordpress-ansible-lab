# WordPress Ansible Lab

This repository contains a complete Ansible exercise for deploying WordPress in the Agile611 `startusingansible` laboratory. It is designed so that a student can pull this repository into the lab workspace and execute the deployment with Ansible.

The lab uses the same four-node topology as the base training environment: an Ansible control node, a database node, a webserver node, and an optional load balancer node. The exercise deploys MySQL, Apache, PHP, WordPress, and optionally Nginx as a reverse proxy.

## 1. Repository contents

| Path | Purpose |
|---|---|
| `hosts` | Inventory for the Agile611 Docker lab network. |
| `ansible.cfg` | Local Ansible configuration for this exercise. |
| `collections/requirements.yml` | Required Ansible collections. |
| `group_vars/all.yml` | Shared WordPress, database, Apache, and Nginx variables. |
| `control.yml` | Installs control-node tools. |
| `database.yml` | Installs Oracle MySQL from the official MySQL APT repository and creates the WordPress database and user. |
| `webserver.yml` | Installs Apache/PHP, downloads WordPress, and configures the site. |
| `loadbalancer.yml` | Installs and configures Nginx as an optional load balancer. |
| `site.yml` | Main playbook that runs the complete deployment in order. |
| `templates/` | Jinja2 templates for WordPress, Apache, and Nginx configuration. |
| `playbooks/stack_status.yml` | Validation playbook for services and HTTP endpoints. |
| `playbooks/clean_wordpress.yml` | Cleanup playbook to reset the exercise. |

## 2. Required lab environment

This exercise is intended to run inside the Docker lab from [`agile611/startusingansible`](https://github.com/agile611/startusingansible). Run the infrastructure commands from your host machine, not from inside the Ansible container.

The database playbook keeps the original exercise goal of using **MySQL**. Because the Agile611 container image is based on Debian Trixie and may not expose `mysql-server` in the default APT repositories, `database.yml` adds Oracle’s official MySQL APT repository on the database node before installing `mysql-server`. In this disposable classroom lab, that repository is added with `trusted=yes` because Oracle’s published APT signing key currently triggers an expired-key validation failure in this environment. Do not use `trusted=yes` in production. The playbook also installs the Debian Bookworm `libaio1` compatibility package because Oracle MySQL 8.4 depends on `libaio1`, while the Debian Trixie lab image does not provide that package name by default.

| Command location | Meaning |
|---|---|
| **Host machine** | Your laptop or workstation where Docker is installed. |
| **Control node** | The `ansible` container, entered with `docker exec -it ansible bash`. |

The expected lab network is shown below.

| Node | IP address | Role |
|---|---:|---|
| `ansible` | `10.11.12.10` | Control node |
| `database` | `10.11.12.20` | MySQL database |
| `loadbalancer` | `10.11.12.30` | Optional Nginx load balancer |
| `webserver` | `10.11.12.40` | Apache, PHP, and WordPress |

## 3. Start the base lab from the host machine

Run these commands on the **host machine**.

```bash
git clone https://github.com/agile611/startusingansible.git
cd startusingansible

mkdir -p ssh
ssh-keygen -t rsa -b 2048 -f ssh/id_rsa -N ""
docker network prune -f
docker compose up -d
sleep 3
chmod +x setup-ssh.sh
./setup-ssh.sh
docker compose ps
```

The base repository is mounted into the Ansible control container at `/home/vagrant/workspace`. The easiest workflow is to clone this WordPress exercise repository inside the base repository directory on the host machine so that it appears automatically inside the control node.

## 4. Pull this exercise repository into the lab

Run this command on the **host machine**, from inside the `startusingansible` folder.

```bash
cd startusingansible
git clone https://github.com/juliodomingues/wordpress-ansible-lab.git
```

Then enter the Ansible control node.

```bash
docker exec -it ansible bash
cd /home/vagrant/workspace/wordpress-ansible-lab
```

If you prefer to clone from inside the control node instead, enter the container first and run the following commands there.

```bash
docker exec -it ansible bash
cd /home/vagrant/workspace
git clone https://github.com/juliodomingues/wordpress-ansible-lab.git
cd wordpress-ansible-lab
```

## 5. Install required Ansible collections

Run these commands inside the **control node**, from `/home/vagrant/workspace/wordpress-ansible-lab`.

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## 6. Verify SSH connectivity

Run this command inside the **control node**, from `/home/vagrant/workspace/wordpress-ansible-lab`.

```bash
ansible -i hosts -u vagrant -m ping all
```

All nodes should return `SUCCESS` and `pong`. If they do not, return to the host machine, go to the `startusingansible` directory, and run `./setup-ssh.sh` again.

## 7. Execute the exercise step by step

Run these commands inside the **control node**, from `/home/vagrant/workspace/wordpress-ansible-lab`.

```bash
# 1. Prepare the Ansible control node.
ansible-playbook -i hosts -u vagrant control.yml

# 2. Configure MySQL and create the WordPress database and user.
ansible-playbook -i hosts -u vagrant database.yml

# 3. Configure Apache, PHP, WordPress files, and wp-config.php.
ansible-playbook -i hosts -u vagrant webserver.yml

# 4. Configure the optional Nginx load balancer.
ansible-playbook -i hosts -u vagrant loadbalancer.yml

# 5. Validate services and HTTP endpoints.
ansible-playbook -i hosts -u vagrant playbooks/stack_status.yml
```

## 8. Execute the complete deployment in one command

After understanding the step-by-step flow, run the complete deployment using the main playbook. Run this command inside the **control node**, from `/home/vagrant/workspace/wordpress-ansible-lab`.

```bash
ansible-playbook -i hosts -u vagrant site.yml
```

The `site.yml` playbook imports the component playbooks in this order: `control.yml`, `database.yml`, `webserver.yml`, `loadbalancer.yml`, and `playbooks/stack_status.yml`.

## 9. Open WordPress in the browser

Open these URLs from the **host machine**.

| URL | Path | Expected result |
|---|---|---|
| `http://localhost/` | Direct webserver access | WordPress installation screen or homepage. |
| `http://localhost:8080/` | Nginx load balancer access | Same WordPress response proxied through Nginx. |

When WordPress asks for database details, use the values in `group_vars/all.yml`.

| Field | Value |
|---|---|
| Database name | `wordpress` |
| Username | `wp_user` |
| Password | `wp_password` |
| Database host | `10.11.12.20` |
| Table prefix | `wp_` |

## 10. Clean and repeat the exercise

Run this command inside the **control node**, from `/home/vagrant/workspace/wordpress-ansible-lab`.

```bash
ansible-playbook -i hosts -u vagrant playbooks/clean_wordpress.yml
```

If you want to fully reset the containers, run these commands on the **host machine**, from inside the `startusingansible` directory.

```bash
docker compose down --volumes --remove-orphans
docker compose up -d
sleep 3
./setup-ssh.sh
```

## 11. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `No package matching mysql-server is available` | The base Debian Trixie lab image does not provide Oracle MySQL in its default repositories | Pull the latest repository version; `database.yml` adds Oracle’s official MySQL APT repository before installing `mysql-server`. |
| `OpenPGP signature verification failed` for `repo.mysql.com` | Oracle’s MySQL APT signing key currently appears expired to the lab image | Pull the latest repository version; `database.yml` removes the old signed source definition and re-adds the MySQL repository in lab-only `trusted=yes` mode. |
| `mysql-community-server-core Depends libaio1 but it is not installable` | Oracle MySQL 8.4 Debian packages require `libaio1`, which is not available by default in the Debian Trixie lab image | Pull the latest repository version; `database.yml` now downloads and installs the Debian Bookworm `libaio1` compatibility package before `mysql-server`. |
| `Plugin mysql_native_password is not loaded` | MySQL 8.4 no longer loads the older `mysql_native_password` plugin by default | Pull the latest repository version; the WordPress user is now created with `caching_sha2_password`, which is supported by MySQL 8. |
| `Failed to find required executable debconf-get-selections` | The `ansible.builtin.debconf` module requires the `debconf-utils` package on the database node | Pull the latest repository version; `database.yml` now installs `debconf-utils` before preseeding MySQL. |
| Ansible returns `UNREACHABLE` | SSH keys are missing or containers are not running | Run `docker compose ps` and `./setup-ssh.sh` from the host inside `startusingansible`. |
| Database modules fail | The MySQL Python connector, official MySQL APT repository, or Ansible collection is missing | Re-run `ansible-galaxy collection install -r collections/requirements.yml` and `database.yml`. |
| WordPress cannot connect to database | Wrong database host, user, password, or MySQL bind address | Check `group_vars/all.yml` and re-run `database.yml`. |
| Apache shows the default page | The WordPress VirtualHost is not enabled | Re-run `webserver.yml` and inspect `/etc/apache2/sites-enabled/`. |
| Nginx returns `502 Bad Gateway` | Webserver is down or upstream is wrong | Re-run `webserver.yml`, then `loadbalancer.yml`. |

## 12. Learning objectives

By completing this lab, students practise inventory management, package installation, service management, MySQL automation, Jinja2 templates, handlers, deployment orchestration with `site.yml`, validation playbooks, and cleanup playbooks.

## References

[Ansible `apt` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)  
[Ansible `template` module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)  
[Ansible MySQL collection](https://docs.ansible.com/projects/ansible/latest/collections/ansible/mysql/index.html)  
[WordPress server requirements](https://developer.wordpress.org/advanced-administration/server/requirements/)  
[Agile611 startusingansible](https://github.com/agile611/startusingansible)
