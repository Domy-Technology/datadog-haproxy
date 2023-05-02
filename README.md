# datadog-haproxy

## Vagrant

**Install Vagrant**

https://developer.hashicorp.com/vagrant/downloads

**Setup VM**

```
vagrant init
```

**Start VM**

```
vagrant up
```

**Connect VM**

```
vagrant ssh
```

## Setup HAProxy

```shell
sudo dnf install haproxy

# Backup Default Config
cd /etc/haproxy/
cp haproxy.cfg haproxy.cfg.orig

# Open Firewall Ports
sudo firewall-cmd --permanent --add-port=3833/tcp
sudo firewall-cmd --permanent --add-port=3834/tcp
sudo firewall-cmd --permanent --add-port=3835/tcp
sudo firewall-cmd --permanent --add-port=3836/tcp
sudo firewall-cmd --permanent --add-port=3837/tcp
sudo firewall-cmd --permanent --add-port=3838/tcp
sudo firewall-cmd --permanent --add-port=10514/tcp
sudo firewall-cmd --permanent --add-port=3839/tcp
sudo firewall-cmd --permanent --add-port=3840/tcp
sudo firewall-cmd --permanent --add-port=3841/tcp
sudo firewall-cmd --permanent --add-port=3842/tcp
sudo firewall-cmd --permanent --add-port=3843/tcp
sudo firewall-cmd --permanent --add-port=443/tcp

# install datadog certificates
yum install ca-certificates
# The path to the certificate is /etc/ssl/certs/ca-bundle.crt

sudo setsebool -P haproxy_connect_any=1
```

## Setup HAproxy Config File

```shell
# Setup Folders
sudo chmod 777 -R /var/lib/haproxy
sudo mkdir /var/lib/haproxy/dev
sudo chmod 777 -R /var/lib/haproxy/dev
sudo chmod 777 /var/log/haproxy.log
```

```shell
# Copiar o conteudo do arquivo http.conf deste diretorio para o arquivo /etc/haproxy/haproxy.cfg da VM
vi /etc/haproxy/haproxy.cfg
```

## Setup Logging

```shell
sudo vi /etc/rsyslog.d/99-haproxy.conf
```

```shell
# Paste the content Below
$AddUnixListenSocket /var/lib/haproxy/dev/log

# Send HAProxy messages to a dedicated logfile
:programname, startswith, "haproxy" {
  /var/log/haproxy.log
  stop
}
```

**Verify SELinux’s current policy
```shell
getenforce
# If the getenforce command returned either Permissive or Disabled, then you can restart Rsyslog with the following command:
# Else....
vi rsyslog-haproxy.te
```

**Paste this content**
```shell
module rsyslog-haproxy 1.0;

require {
    type syslogd_t;
    type haproxy_var_lib_t;
    class dir { add_name remove_name search write };
    class sock_file { create setattr unlink };
}

#============= syslogd_t ==============
allow syslogd_t haproxy_var_lib_t:dir { add_name remove_name search write };
allow syslogd_t haproxy_var_lib_t:sock_file { create setattr unlink };
```

```shell
sudo dnf install checkpolicy
checkmodule -M -m rsyslog-haproxy.te -o rsyslog-haproxy.mod
semodule_package -o rsyslog-haproxy.pp -m rsyslog-haproxy.mod
sudo semodule -i rsyslog-haproxy.pp
sudo semodule -l |grep rsyslog-haproxy
```

**Restart Syslog**
```shell
sudo systemctl restart rsyslog
```

## Habilitar o HAProxy

```shell
sudo systemctl enable --now haproxy
# Após alterar o arquivo
sudo systemctl reload haproxy
```

## Enable HTTPS:

```shell
# Install CertBot for Certificate Generation

# Install Snap
sudo dnf install epel-release
sudo dnf upgrade
sudo yum install snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap

# Install Certbot
sudo dnf remove certbot
sudo snap install --classic certbot

# Prepare Certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot certonly --standalone
```

## Setup Agent

```shell
# Install
DD_API_KEY={API_KEY} DD_SITE="datadoghq.com" bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script_agent7.sh)"
```
### Edit /etc/datadog-agent/datadog.yaml and set Proxy URL

```yaml
proxy:
  http: http://127.0.0.1:3834
```

### Reload Datadog Agent

```
sudo datadog-agent stop
sudo datadog-agent start > /dev/null 2>&1 &
```

## Referencias:

- https://docs.datadoghq.com/agent/proxy/?tab=linux
- https://docs.oracle.com/en/operating-systems/oracle-linux/8/balancing/balancing-SettingUpLoadBalancingbyUsingHAProxy.html#haproxy-config
- https://certbot.eff.org/instructions?ws=haproxy&os=centosrhel8
- https://www.digitalocean.com/community/tutorials/how-to-configure-haproxy-logging-with-rsyslog-on-centos-8-quickstart