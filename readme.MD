# Install Ansible
sudo-apt install ansible python3-pip
# Install Tox
pip3 tox
# Ansible Prometheus
git clone https://github.com/cloudalchemy/ansible-prometheus
cd ansible-prometheus
mkdir -p roles/cloudalchemy.prometheus
mv defaults/ handlers/ meta/ molecule/ tasks/ templates/ vars/ roles/cloudalchemy.prometheus/
sudo vi readme.md
sudo vi main.yml
---
- hosts: all
  roles:
  - cloudalchemy.prometheus
  vars:
    prometheus_targets:
        node:
        - targets:
            - localhost:9100
            - demo.cloudalchemy.org:9100
            labels:
                env: demosite
...
sudo vi inventory
localhost ansible_connection=local
ansible-playbook -i inventory main.yml

# In Root Node exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.1.2/node_exporter-1.1.2.linux-amd64.tar.gz
tar xzf node_exporter-1.1.2.linux-amd64.tar.gz
cd node_exporter-1.1.2.linux-amd64
mv node_exporter /usr/local/bin
# Configure node_exporter service
sudo vi /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target

sudo useradd node_exporter
sudo systemctl daemon-reload 
sudo systemctl start node_exporter
sudo systemctl status node_exporter

# Prometheus
sudo vi /etc/prometheus/prometheus.yml
- job_name: 'node_exporter_metrics'
    scrape_interval: 5s
    static_configs:
     - targets: ['localhost:9100']

sudo systemctl restart prometheus
sudo systemctl status prometheus

# Grafana
curl -LO https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.1.4_amd64.deb ;
sudo apt-get install -y adduser libfontconfig ;
sudo dpkg -i grafana_5.1.4_amd64.deb ;
sudo systemctl start grafana-server ;
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
ps -ef | grep grafana

# ELK Stack
git clone https://github.com/elastic/ansible-elasticsearch
cd ansible-elasticsearch
sudo vi main.yml
---
- name: Simple Example
  hosts: localhost
  roles:
   - role: elastic.elasticsearch
...
mkdir -p roles/elastic.elasticsearch
mv defaults/ docs/ files/ filter_plugins/ handlers/ meta/ tasks/ templates/ test/ vars roles/elastic.elasticsearch
sudo vi invetory
localhost ansible_connection=local
ansible-playbook -i inventory main.yml

# Kibana
apt install kibana
apt install nginx
sudo vi /etc/nginx/sites-enabled/default
server {
        listen 80 default_server;
        server_name http://18.207.251.117/;
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/htpasswd.users;

        location / {
                proxy_pass http://localhost:5601;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
        }
}
echo "kibanaadmin:`openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.users
sudo vi /etc/kibana/kibana.yml
elasticsearch.username: "elastic"
elasticsearch.password: "1234"
systemctl start elasticsearch