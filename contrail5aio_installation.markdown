- [All-in-one openstack-helm with contrail cluster (NON-HA)](#all-in-one-openstack-helm-with-contrail-cluster--non-ha-)
  * [Tested with](#tested-with)
  * [Resource spec (used for internal validation)](#resource-spec--used-for-internal-validation-)
  * [Pre-req packages](#pre-req-packages)
  * [Installation steps](#installation-steps)
  * [Git clone the necessary repo's using below command](#git-clone-the-necessary-repo-s-using-below-command)
  * [Export variables needed by below procedure](#export-variables-needed-by-below-procedure)
  * [change `external_dns_nameservers` in `vars.yaml`](#change--external-dns-nameservers--in--varsyaml-)
  * [Installing necessary packages and deploying kubernetes](#installing-necessary-packages-and-deploying-kubernetes)
  * [Install openstack and heat client](#install-openstack-and-heat-client)
  * [Deploy openstack-helm related charts](#deploy-openstack-helm-related-charts)
  * [deploy opencontrail charts](#deploy-opencontrail-charts)
  * [Deploy heat charts](#deploy-heat-charts)
  * [all commands](#all-commands)

# All-in-one openstack-helm with contrail cluster (NON-HA)

Using below step you can bring an all-in-one cluster with openstack and contrail

## Tested with

1. Operating system: Ubuntu 16.04.3 LTS
2. Kernel: 4.4.0-87-generic
3. docker: 1.13.1
4. helm: v2.7.2
5. kubernetes: v1.9.3
6. openstack: ocata

## Resource spec (used for internal validation)

1. CPU: 8
2. RAM: 32 GB
3. HDD: 120 GB

## Pre-req packages

Install below packages on your setup

```bash
git
ntp
```


## Installation steps

   ```bash
   sed -i '/HISTCONTROL=ignoredups:ignorespace/s/^/#/g' ~/.bashrc
   ```

exit and ssh into node again.

## Git clone the necessary repo's using below command

   ```bash
   # Download openstack-helm code
   git clone https://github.com/Juniper/openstack-helm.git
   # Download openstack-helm-infra code
   git clone https://github.com/Juniper/openstack-helm-infra.git
   # Download contrail-helm-deployer code
   git clone https://github.com/Juniper/contrail-helm-deployer.git
   ```

## Export variables needed by below procedure

   ```bash
   export BASE_DIR=$(pwd)
   export OSH_PATH=${BASE_DIR}/openstack-helm
   export OSH_INFRA_PATH=${BASE_DIR}/openstack-helm-infra
   export CHD_PATH=${BASE_DIR}/contrail-helm-deployer
   ``` 

## change `external_dns_nameservers` in `vars.yaml`

   ```bash
   export varsyaml="${BASE_DIR}/openstack-helm-infra/tools/images/kubeadm-aio/assets/opt/playbooks/vars.yaml"
   sed -i '/8.8.8.8/d' $varsyaml
   sed -i 's/8.8.4.4/172.22.199.11/g' $varsyaml
   ```

## Installing necessary packages and deploying kubernetes

   ```bash
   cd ${OSH_PATH}
   ./tools/deployment/developer/common/001-install-packages-opencontrail.sh
   ./tools/deployment/developer/common/010-deploy-k8s.sh
   ```

Edit `${OSH_INFRA_PATH}/tools/gate/devel/local-vars.yaml` if you would want to
install a different version of kubernetes, cni, calico. This overrides the
default values given in `${OSH_INFRA_PATH}/playbooks/vars.yaml`

Sample `${OSH_INFRA_PATH}/tools/gate/devel/local-vars.yaml` file

```yaml
version:
  kubernetes: v1.9.3
  helm: v2.7.2
  cni: v0.6.0

docker:
  #list of insecure_registries, from where you will be pulling container images
  insecure_registries:
    - "10.87.65.243:5000"
  #list of private secure docker registry auth info, from where you will be pulling container images
  #private_registries:
  #  - name: <docker-registry-name>
  #    username: username@abc.xyz
  #    email: username@abc.xyz
  #    password: password
  #    secret_name: contrail-image-secret
  #    namespace: openstack
kubernetes:
  network:
    default_device: docker0
  cluster:
    cni: calico
    pod_subnet: 192.168.0.0/16
    domain: cluster.local
nodes:
  labels:
    all:
    - name: openstack-control-plane
      value: enabled
    - name: openstack-compute-node
      value: enabled
    - name: linuxbridge
      value: enabled
    - name: ceph-mon
      value: enabled
    - name: ceph-osd
      value: enabled
    - name: ceph-mds
      value: enabled
    - name: ceph-rgw
      value: enabled
    - name: ceph-mgr
      value: enabled
    - name: opencontrail.org/controller
      value: enabled
    - name: opencontrail.org/vrouter-kernel
      value: enabled
```

## Install openstack and heat client

   ```bash
   ./tools/deployment/developer/common/020-setup-client.sh
   ```

## Deploy openstack-helm related charts

   ```bash
   ./tools/deployment/developer/nfs/031-ingress-opencontrail.sh
   ./tools/deployment/developer/nfs/040-nfs-provisioner.sh
   ./tools/deployment/developer/nfs/050-mariadb.sh
   ./tools/deployment/developer/nfs/060-rabbitmq.sh
   ./tools/deployment/developer/nfs/070-memcached.sh
   ./tools/deployment/developer/nfs/080-keystone.sh
   ./tools/deployment/developer/nfs/100-horizon.sh
   ./tools/deployment/developer/nfs/120-glance.sh
   ./tools/deployment/developer/nfs/151-libvirt-opencontrail.sh
   # Edit ${OSH_PATH}/tools/overrides/backends/opencontrail/nova.yaml and
   # ${OSH_PATH}/tools/overrides/backends/opencontrail/neutron.yaml
   # to make sure that you are pulling the right opencontrail init container image
   ./tools/deployment/developer/nfs/161-compute-kit-opencontrail.sh
   ```

## deploy opencontrail charts

1. compile charts

   ```bash
   cd $CHD_PATH
   make
   ```

2. prepare `contrail.yaml` file

   ```bash
   #Set the IP of your CONTROL_NODES (specify your control data ip, if you have one)
   export CONTROL_NODES=10.169.25.25
   export VROUTER_GATEWAY=10.169.25.1
   export CONTROL_DATA_NET_LIST="10.169.25.0/24"
   export CONTRAIL_REGISTRY="opencontrailnightly"
   export CONTRAIL_TAG="5.0-latest"
   kubectl replace -f ${CHD_PATH}/rbac/cluster-admin.yaml
   ```

   ```
   tee /tmp/contrail.yaml << EOF
   global:
     contrail_env:
       CONTROLLER_NODES: 172.17.0.1
       CONTROL_NODES: ${CONTROL_NODES}
       VROUTER_GATEWAY: ${VROUTER_GATEWAY}
       CONTROL_DATA_NET_LIST: ${CONTROL_DATA_NET_LIST}
       AAA_MODE: rbac
       CLOUD_ORCHESTRATOR: openstack
       BGP_ASN: 60227
       DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 5
       CONFIG_NODEMGR__DEFAULTS__minimum_diskGB: 5
     images:
       tags:
         kafka: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-external-kafka:${CONTRAIL_TAG:-latest}"
         cassandra: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-external-cassandra:${CONTRAIL_TAG:-latest}"
         redis: "redis:4.0.2"
         zookeeper: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-external-zookeeper:${CONTRAIL_TAG:-latest}"
         contrail_control: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-control-control:${CONTRAIL_TAG:-latest}"
         control_dns: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-control-dns:${CONTRAIL_TAG:-latest}"
         control_named: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-control-named:${CONTRAIL_TAG:-latest}"
         config_api: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-api:${CONTRAIL_TAG:-latest}"
         config_devicemgr: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-devicemgr:${CONTRAIL_TAG:-latest}"
         config_schema_transformer: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-schema:${CONTRAIL_TAG:-latest}"
         config_svcmonitor: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-svcmonitor:${CONTRAIL_TAG:-latest}"
         webui_middleware: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-webui-job:${CONTRAIL_TAG:-latest}"
         webui: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-webui-web:${CONTRAIL_TAG:-latest}"
         analytics_api: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-api:${CONTRAIL_TAG:-latest}"
         contrail_collector: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-collector:${CONTRAIL_TAG:-latest}"
         analytics_alarm_gen: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-alarm-gen:${CONTRAIL_TAG:-latest}"
         analytics_query_engine: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-query-engine:${CONTRAIL_TAG:-latest}"
         analytics_snmp_collector: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-snmp-collector:${CONTRAIL_TAG:-latest}"
         contrail_topology: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-topology:${CONTRAIL_TAG:-latest}"
         build_driver_init: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-kernel-build-init:${CONTRAIL_TAG:-latest}"
         vrouter_agent: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-agent:${CONTRAIL_TAG:-latest}"
         vrouter_init_kernel: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-kernel-init:${CONTRAIL_TAG:-latest}"
         vrouter_dpdk: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-agent-dpdk:${CONTRAIL_TAG:-latest}"
         vrouter_init_dpdk: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-kernel-init-dpdk:${CONTRAIL_TAG:-latest}"
         nodemgr: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-nodemgr:${CONTRAIL_TAG:-latest}"
         contrail_status: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-status:${CONTRAIL_TAG:-latest}"
         node_init: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-node-init:${CONTRAIL_TAG:-latest}"
         dep_check: quay.io/stackanetes/kubernetes-entrypoint:v0.2.1
   EOF
   ```

3. install:

   ```bash
   helm install --name contrail ${CHD_PATH}/contrail \
   --namespace=contrail --values="/tmp/contrail.yaml" \
   ${CONTRAIL_REGISTRY_ARG}
   cp /tmp/contrail.yaml ~/
   ```

## Deploy heat charts

   ```bash
   cd ${OSH_PATH}
   ./tools/deployment/developer/nfs/091-heat-opencontrail.sh
   ```

## appendix

### all commands

```bash
git clone https://github.com/Juniper/openstack-helm.git
git clone https://github.com/Juniper/openstack-helm-infra.git
git clone https://github.com/Juniper/contrail-helm-deployer.git

export BASE_DIR=$(pwd)
export OSH_PATH=${BASE_DIR}/openstack-helm
export OSH_INFRA_PATH=${BASE_DIR}/openstack-helm-infra
export CHD_PATH=${BASE_DIR}/contrail-helm-deployer
export varsyaml="${BASE_DIR}/openstack-helm-infra/tools/images/kubeadm-aio/assets/opt/playbooks/vars.yaml"

sed -i '/8.8.8.8/d' $varsyaml
sed -i 's/8.8.4.4/172.22.199.11/g' $varsyaml

cd ${OSH_PATH}
./tools/deployment/developer/common/001-install-packages-opencontrail.sh
./tools/deployment/developer/common/010-deploy-k8s.sh
./tools/deployment/developer/common/020-setup-client.sh
./tools/deployment/developer/nfs/031-ingress-opencontrail.sh
./tools/deployment/developer/nfs/040-nfs-provisioner.sh
./tools/deployment/developer/nfs/050-mariadb.sh
./tools/deployment/developer/nfs/060-rabbitmq.sh
./tools/deployment/developer/nfs/070-memcached.sh
./tools/deployment/developer/nfs/080-keystone.sh
./tools/deployment/developer/nfs/100-horizon.sh
./tools/deployment/developer/nfs/120-glance.sh
./tools/deployment/developer/nfs/151-libvirt-opencontrail.sh
# Edit ${OSH_PATH}/tools/overrides/backends/opencontrail/nova.yaml and
# ${OSH_PATH}/tools/overrides/backends/opencontrail/neutron.yaml
# to make sure that you are pulling the right opencontrail init container image
./tools/deployment/developer/nfs/161-compute-kit-opencontrail.sh

#Set the IP of your CONTROL_NODES (specify your control data ip, if you have one)
export CONTROL_NODES=10.169.25.25
export VROUTER_GATEWAY=10.169.25.1
export CONTROL_DATA_NET_LIST="10.169.25.0/24"
export CONTRAIL_REGISTRY="opencontrailnightly"
export CONTRAIL_TAG="5.0-latest"
kubectl replace -f ${CHD_PATH}/rbac/cluster-admin.yaml

tee /tmp/contrail.yaml << EOF
global:
  contrail_env:
    CONTROLLER_NODES: 172.17.0.1
    CONTROL_NODES: ${CONTROL_NODES}
    VROUTER_GATEWAY: ${VROUTER_GATEWAY}
    CONTROL_DATA_NET_LIST: ${CONTROL_DATA_NET_LIST}
    AAA_MODE: rbac
    CLOUD_ORCHESTRATOR: openstack
    BGP_ASN: 60227
    DATABASE_NODEMGR__DEFAULTS__minimum_diskGB: 5
    CONFIG_NODEMGR__DEFAULTS__minimum_diskGB: 5
  images:
    tags:
      kafka: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-external-kafka:${CONTRAIL_TAG:-latest}"
      cassandra: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-external-cassandra:${CONTRAIL_TAG:-latest}"
      redis: "redis:4.0.2"
      zookeeper: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-external-zookeeper:${CONTRAIL_TAG:-latest}"
      contrail_control: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-control-control:${CONTRAIL_TAG:-latest}"
      control_dns: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-control-dns:${CONTRAIL_TAG:-latest}"
      control_named: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-control-named:${CONTRAIL_TAG:-latest}"
      config_api: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-api:${CONTRAIL_TAG:-latest}"
      config_devicemgr: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-devicemgr:${CONTRAIL_TAG:-latest}"
      config_schema_transformer: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-schema:${CONTRAIL_TAG:-latest}"
      config_svcmonitor: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-config-svcmonitor:${CONTRAIL_TAG:-latest}"
      webui_middleware: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-webui-job:${CONTRAIL_TAG:-latest}"
      webui: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-controller-webui-web:${CONTRAIL_TAG:-latest}"
      analytics_api: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-api:${CONTRAIL_TAG:-latest}"
      contrail_collector: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-collector:${CONTRAIL_TAG:-latest}"
      analytics_alarm_gen: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-alarm-gen:${CONTRAIL_TAG:-latest}"
      analytics_query_engine: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-query-engine:${CONTRAIL_TAG:-latest}"
      analytics_snmp_collector: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-snmp-collector:${CONTRAIL_TAG:-latest}"
      contrail_topology: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-analytics-topology:${CONTRAIL_TAG:-latest}"
      build_driver_init: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-kernel-build-init:${CONTRAIL_TAG:-latest}"
      vrouter_agent: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-agent:${CONTRAIL_TAG:-latest}"
      vrouter_init_kernel: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-kernel-init:${CONTRAIL_TAG:-latest}"
      vrouter_dpdk: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-agent-dpdk:${CONTRAIL_TAG:-latest}"
      vrouter_init_dpdk: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-vrouter-kernel-init-dpdk:${CONTRAIL_TAG:-latest}"
      nodemgr: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-nodemgr:${CONTRAIL_TAG:-latest}"
      contrail_status: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-status:${CONTRAIL_TAG:-latest}"
      node_init: "${CONTRAIL_REGISTRY:-opencontrailnightly}/contrail-node-init:${CONTRAIL_TAG:-latest}"
      dep_check: quay.io/stackanetes/kubernetes-entrypoint:v0.2.1
EOF

cp /tmp/contrail.yaml ~/
helm install --name contrail ${CHD_PATH}/contrail \
--namespace=contrail --values=/tmp/contrail.yaml \

cd ${OSH_PATH}
./tools/deployment/developer/nfs/091-heat-opencontrail.sh
```

### installation logs

https://gist.github.com/pinggit/2498c59fa0be4d892ee333dc286e7346
