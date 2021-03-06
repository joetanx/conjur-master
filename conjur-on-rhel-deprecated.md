> Conjur on RHEL was a CyberArk internal Controlled Availability feature which was deprecated before it even got released.
>
> This document is just an archive.
## 1. RHEL Based Master
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
- Refer to https://github.com/joetanx/conjur-master/blob/main/generate-conjur-certificates.md for a guide to generate your own certificates
> Note: In event of "error: cert already in hash table", ensure that conjur/follower certificates do not contain the CA certificate
```console
curl -L -o conjur-certs.tgz https://github.com/joetanx/conjur-master/raw/main/conjur-certs.tgz
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
> In situation where a single-node Kubernetes cluster is deployed on the same machine as the Conjur-on-RHEL, the Conjur CLI .netrc file will cause the follower deployment to fail.
>
> Logout and delete the .netrc before deploying the follower.
```console
conjur logout
rm -f .netrc
```
