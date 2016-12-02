
Install pip
easy_install pip

install docker
echo deb http://ftp.unicamp.br/pub/ppc64el/ubuntu/16_04/docker-1.12.0-ppc64el/ xenial main > /etc/apt/sources.list.d/xenial-docker.list
apt-get update
apt-get install docker-engine

Install dependencies
apt-get install python-dev libffi-dev gcc libssl-dev  git

set mountflag option for docker
mkdir -p /etc/systemd/system/docker.service.d
vi /etc/systemd/system/docker.service.d/kolla.conf
[Service]
MountFlags=shared

set proxy in docker if required 
/etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=https://web-proxy.corp.xxxxxx.com:8080/"
Environment="HTTPS_PROXY=https://web-proxy.corp.xxxxxx.com:8080/"
Environment="NO_PROXY=localhost,127.0.0.1,localaddress,.localdomain.com"

enabled insecure access to registry
vi /etc/default/docker 
DOCKER_OPTS="--insecure-registry 192.168.155.12:4000"

vi /lib/systemd/system/docker.service
[Service]
ExecStart=/usr/bin/docker -d -H fd:// $DOCKER_OPTS
EnvironmentFile=-/etc/default/docker
systemctl daemon-reload
systemctl restart docker

mount 
mount --make-shared /run

pip install -U python-openstackclient python-neutronclient python-novaclient docker-py
pip install ansible


git clone https://github.com/persistentgroups/kolla-power.git
pip install kolla-power/
cd kolla-power/
cp -r etc/kolla /etc/

apt-get install ntp

Install  docker registry 
apt-get install docker-registry -y
vi /etc/docker/registry/config.yml
http: 
	address: :4000
	headers:
		X-Content-Type-Options: [nosniff]

nohup /usr/bin/docker-registry 	/etc/docker/registry/config.yml &

Build kolla images 
kolla-build --base ubuntu --base-arch ppc64le --type source --registry  192.168.155.12:4000 --push

Deploy kolla
edit /et/kolla/passwords.yml and /et/kolla/globals.yml

/etc/kolla/globals.yml
	config_strategy: "COPY_ALWAYS"
	kolla_base_distro: "ubuntu"
	kolla_install_type: "source"
	kolla_internal_vip_address: " 192.168.155.12"
	kolla_internal_fqdn: "{{ kolla_internal_vip_address }}"
	kolla_external_vip_address: "{{ kolla_internal_vip_address }}"
	kolla_external_fqdn: "{{ kolla_external_vip_address }}"
	docker_registry: "10.154.54.19:4000"
	network_interface: "vmbr0"
	neutron_external_interface: "veth1"
	openstack_logging_debug: "False"
	nova_console: "novnc"
	enable_cinder: "yes"
	enable_heat: "no"
	enable_magnum: "no"
	enable_haproxy: "no"
	enable_cinder_backend_lvm: "yes"
	
kolla-ansible post deploy
kolla-ansible  deploy
source /etc/kolla/admin-openrc.sh
