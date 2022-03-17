# Setup Conjur Enterprise master on RHEL with Podman

### Software Versions
- RHEL 8.5
- Conjur Enterprise 12.5

# 1.0 Setup host prerequisites
## 1.1 Install necessary packages
- Install `podman` and `policycoreutils-python-utils` packages
- The `policycoreutils-python-utils` is required for the `semanage fcontext` command, which is used to allow the Postgres container to access the data directory on the host
```console
yum install -y podman policycoreutils-python-utils
systemctl enable --now podman
```

## 1.2 Load the Conjur appliance image
- Obtain the Conjur container image from CyberArk
- Upload the Conjur container image to the container host
```console
podman load -i conjur-appliance_12.5.0.tar.gz
```
- Clean-up
```console
rm -f conjur-appliance_12.5.0.tar.gz
```

## 1.3 Stage the data directories
- Several directories will be mounted to Conjur appliance
- To allow the Conjur appliance access to the data directory, the SELinux type label needs to be assigned to `svirt_sandbox_file_t`
- SELinux can also simply be disabled, but that is not preferred
```console
mkdir -p /opt/conjur/{security,config,backups,seeds,logs}
semanage fcontext -a -t svirt_sandbox_file_t "/opt/conjur(/.*)?"
restorecon -R -v /opt/conjur
```

## 1.4 Setup Conjur CLI
- Ref: <https://github.com/cyberark/conjur-api-python3/releases>
```console
curl -L -o conjur-cli-rhel-8.tar.gz https://github.com/cyberark/conjur-api-python3/releases/download/v7.1.0/conjur-cli-rhel-8.tar.gz
tar xvf conjur-cli-rhel-8.tar.gz
mv conjur /usr/local/bin/
```
- Clean-up
```console
rm -f conjur-cli-rhel-8.tar.gz
```

## 1.5.1 Method 1: Running Conjur master on the default bridge network
- Podman run command:
```console
podman run --name conjur -d \
--restart=unless-stopped \
--security-opt seccomp=unconfined \
-p "443:443" -p "444:444" -p "5432:5432" -p "1999:1999" \
--log-driver journald \
-v /opt/conjur/config:/etc/conjur/config:Z \
-v /opt/conjur/security:/opt/cyberark/dap/security:Z \
-v /opt/conjur/backups:/opt/conjur/backup:Z \
-v /opt/conjur/seeds:/opt/cyberark/dap/seeds:Z \
-v /opt/conjur/logs:/var/log/conjur:Z \
registry.tld/conjur-appliance:12.5.0
```

## 1.5.2 Method 2: Running Conjur master on the Podman host network
- Podman run command:
```console
podman run --name conjur -d \
--restart=unless-stopped \
--security-opt seccomp=unconfined \
--network host \
--log-driver journald \
-v /opt/conjur/config:/etc/conjur/config:Z \
-v /opt/conjur/security:/opt/cyberark/dap/security:Z \
-v /opt/conjur/backups:/opt/conjur/backup:Z \
-v /opt/conjur/seeds:/opt/cyberark/dap/seeds:Z \
-v /opt/conjur/logs:/var/log/conjur:Z \
registry.tld/conjur-appliance:12.5.0
```
- Add firewall rules on the Podman host
```console
firewall-cmd --add-service https --permanent
firewall-cmd --add-service postgresql --permanent
firewall-cmd --add-port 444/tcp --permanent
firewall-cmd --add-port 1999/tcp --permanent
firewall-cmd --reload
```

# 2.0 Configure the Conjur appliance as master
## 2.1 Initialize the Conjur appliance
- Edit the admin account password in `-p` option and the Conjur account (`cyberark`) according to your environment
```console
podman exec conjur evoke configure master --accept-eula -h conjur.vx --master-altnames "conjur.vx" -p CyberArk123! cyberark
```

## 2.2 Configure container to start on boot
- Run the Conjur container as systemd service and configure it to setup with container host
- Ref: <https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/managing_containers/running_containers_as_systemd_services_with_podman>
```console
podman generate systemd conjur --name --container-prefix="" --separator="" > /etc/systemd/system/conjur.service
systemctl enable conjur
```

## 2.3 Setup Conjur certificates
- The `conjur-certs.tgz` include personal certificate chain for CA, Master and follower, you should generate your own certificates
- Refer to <https://joetanx.github.io/self-signed-ca/> for a guide to generate your own certificates
- **Note**: In event of `error: cert already in hash table`, ensure that the Conjur serverfollower certificates do not contain the CA certificate
```console
curl -L -o conjur-certs.tgz https://github.com/joetanx/conjur-master/raw/main/conjur-certs.tgz
podman exec conjur mkdir -p /opt/cyberark/dap/certificates
podman cp conjur-certs.tgz conjur:/opt/cyberark/dap/certificates/
podman exec conjur tar xvf /opt/cyberark/dap/certificates/conjur-certs.tgz -C /opt/cyberark/dap/certificates/
podman exec conjur evoke ca import -fr /opt/cyberark/dap/certificates/central.pem
podman exec conjur evoke ca import -k /opt/cyberark/dap/certificates/conjur.vx.key -s /opt/cyberark/dap/certificates/conjur.vx.pem
podman exec conjur evoke ca import -k /opt/cyberark/dap/certificates/follower.conjur.svc.cluster.local.key /opt/cyberark/dap/certificates/follower.conjur.svc.cluster.local.pem
```
- Clean-up
```console
podman exec conjur rm -rf /opt/cyberark/dap/certificates
rm -f conjur-certs.tgz
```

## 2.4 Initialize Conjur CLI and login to conjur
```console
conjur init -u https://conjur.vx
conjur login -i admin -p CyberArk123!
```

## 3.0 Staging secret variables
- Integration guides in my GitHub uses the secrets set in this step
- Pre-requisites
  - Setup MySQL database according to this guide: <https://joetanx.github.io/conjur-mysql>
  - Have an AWS IAM user account with programmatic access
- Credentials are configured by `app-vars.yaml` in `world_db` and `aws_api` policies that are defined with the respective secret variables
- Download the Conjur policies
```console
curl -L -o app-vars.yaml https://github.com/joetanx/conjur-master/raw/main/app-vars.yaml
```
- Load the policies to Conjur
```console
conjur policy load -b root -f app-vars.yaml
```
- Populate the variables
```console
conjur variable set -i world_db/username -v cityapp
conjur variable set -i world_db/password -v Cyberark1
conjur variable set -i aws_api/awsakid -v <AWS_ACCESS_KEY_ID>
conjur variable set -i aws_api/awssak -v <AWS_SECRET_ACCESS_KEY>
```
- Clean-up
```console
rm -f app-vars.yaml
```
