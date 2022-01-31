# 4. Setup Conjur Master
Conjur Master deployment options:
1. On RHEL directly - Conjur on RHEL is under Controlled Availability, contact CyberArk for this deployment option
2. As a container
## 4.1. RHEL Based Master
- The Conjur-RHEL installer comes with an older version of keyutils in its repository which will fail to install if your RHEL is updated or if you're installing on RHEL 8.5. Manually installing keyutils resolves this.
```console
yum -y install http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/keyutils-1.5.10-9.el8.x86_64.rpm
```
- Download the Conjur CLI and add execute permissions
- Ref: https://github.com/cyberark/conjur-api-python3/releases
```console
curl -L -o conjur-cli-rhel-8.tar.gz https://github.com/cyberark/conjur-api-python3/releases/download/v7.1.0/conjur-cli-rhel-8.tar.gz
tar xvf conjur-cli-rhel-8.tar.gz
mv conjur /usr/local/bin/
```
- Clean-up
```console
rm -f conjur-cli-rhel-8.tar.gz
```
- Obtain the Conjur installation package from CyberArk
- Unpack and install Conjur node
```console
mkdir conjur && cd $_
# upload Conjur-Enterprise-RHELinux-Intel-Rls-v12.4.0+Conjur.RHEL.CA.tar.gz to conjur folder
tar xvf Conjur-Enterprise-RHELinux-Intel-Rls-v12.4.0+Conjur.RHEL.CA.tar.gz
sed -i '$ s/accept_eula\:/accept_eula\: true/' conjur_enterprise_node_config.yml
./conjur_enterprise_setup.sh --install-node
```
- Setup environment variables and allow Conjur communications through firewalld
```console
source /etc/profile.d/conjur_enterprise.sh
firewall-cmd --add-service http --permanent
firewall-cmd --add-service https --permanent
firewall-cmd --add-service ldaps --permanent
firewall-cmd --add-port 1999/tcp --permanent
firewall-cmd --add-service postgresql --permanent
firewall-cmd --reload
```
- Clean-up
```console
cd .. && rm -rf conjur
```
- Setup the Conjur node as master
- Edit the admin account password in `-p` option and the Conjur account (`cyberark`) according to your environment
```console
evoke configure master --accept-eula -h conjur.vx --master-altnames conjur.vx -p CyberArk123! cyberark
```
- Setup Conjur certificates
- The `conjur-certs.tgz` include CA, Master and follower certificates for my lab use, you should generate your own certificates
- Refer to https://github.com/joetanx/conjur-k8s/blob/main/generate-conjur-certificates.md for a guide to generate your own certificates
> Note: In event of "error: cert already in hash table", ensure that conjur/follower certificates do not contain the CA certificate
```console
curl -L -o conjur-certs.tgz https://github.com/joetanx/conjur-k8s/raw/main/conjur-certs.tgz
tar xvf conjur-certs.tgz
evoke ca import --root central.pem
evoke ca import --key follower.default.svc.cluster.local.key follower.default.svc.cluster.local.pem
evoke ca import --key conjur.vx.key --set conjur.vx.pem
```
- Clean-up
```console
rm -f *.key
rm -f *.pem
```
- Initialize Conjur CLI and login to conjur
```console
conjur init -u https://conjur.vx
conjur login -i admin -p CyberArk123!
```
## 4.2. Container Based Master
- Install podman
```console
yum -y installl podman
```
- Obtain the Conjur container image from CyberArk
- Upload the Conjur container image to the container host
```console
podman load -i conjur-appliance_12.4.1.tar.gz
```
- Clean-up
```console
rm -f conjur-appliance_12.4.1.tar.gz
```
- Stage the volume mounts and download the conjur configuration file
```console
mkdir -p /opt/cyberark/dap/{security,config,backups,seeds,logs}
curl -L -o /opt/cyberark/dap/config/conjur.yml https://github.com/joetanx/conjur-k8s/raw/main/conjur.yml
```
- Download the Conjur CLI and add execute permissions
- Ref: https://github.com/cyberark/conjur-api-python3/releases
```console
curl -L -o conjur-cli-rhel-8.tar.gz https://github.com/cyberark/conjur-api-python3/releases/download/v7.1.0/conjur-cli-rhel-8.tar.gz
tar xvf conjur-cli-rhel-8.tar.gz
mv conjur /usr/local/bin/
```
- Clean-up
```console
rm -f conjur-cli-rhel-8.tar.gz
```
- Run the Conjur container
```console
podman run --name conjur -d \
--restart=unless-stopped \
--security-opt seccomp=unconfined \
-p "443:443" -p "444:444" -p "5432:5432" -p "1999:1999" \
--log-driver journald \
-v /opt/cyberark/dap/config:/etc/conjur/config:Z \
-v /opt/cyberark/dap/security:/opt/cyberark/dap/security:Z \
-v /opt/cyberark/dap/backups:/opt/conjur/backup:Z \
-v /opt/cyberark/dap/seeds:/opt/cyberark/dap/seeds:Z \
-v /opt/cyberark/dap/logs:/var/log/conjur:Z \
registry.tld/conjur-appliance:12.4.1
```
- Setup the Conjur container as master
- Edit the admin account password in `-p` option and the Conjur account (`cyberark`) according to your environment
```console
podman exec conjur evoke configure master --accept-eula -h conjur.vx --master-altnames "conjur.vx" -p CyberArk123! cyberark
```
- Run the Conjur container as systemd service and configure it to setup with container host
- Ref: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/managing_containers/running_containers_as_systemd_services_with_podman
```console
podman generate systemd conjur --name --container-prefix="" --separator="" > /etc/systemd/system/conjur.service
systemctl enable conjur
```
- Setup Conjur certificates
- The `conjur-certs.tgz` include CA, Master and follower certificates for my lab use, you should generate your own certificates
- Refer to https://github.com/joetanx/conjur-k8s/blob/main/generate-conjur-certificates.md for a guide to generate your own certificates
> Note: In event of "error: cert already in hash table", ensure that conjur/follower certificates do not contain the CA certificate
```console
curl -L -o conjur-certs.tgz https://github.com/joetanx/conjur-k8s/raw/main/conjur-certs.tgz
podman cp conjur-certs.tgz conjur:/tmp/
podman exec conjur tar xvf /tmp/conjur-certs.tgz -C /tmp/
podman exec conjur evoke ca import --root /tmp/central.pem
podman exec conjur evoke ca import --key /tmp/follower.default.svc.cluster.local.key /tmp/follower.default.svc.cluster.local.pem
podman exec conjur evoke ca import --key /tmp/conjur.vx.key --set /tmp/conjur.vx.pem
```
- Clean-up
```console
podman exec conjur /bin/sh -c "rm -f /tmp/conjur-certs.tgz /tmp/*.pem /tmp/*key"
rm -f conjur-certs.tgz
```
- Initialize Conjur CLI and login to conjur
```console
conjur init -u https://conjur.vx
conjur login -i admin -p CyberArk123!
```
