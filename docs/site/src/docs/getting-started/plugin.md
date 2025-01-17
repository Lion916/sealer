# Plugins Usage

## Apply Plugins metadata in Clusterfile

For example, set node label after install kubernetes cluster:

```yaml
apiVersion: sealer.aliyun.com/v1alpha1
kind: Cluster
metadata:
  name: my-cluster
spec:
  image: kubernetes:v1.19.8
  provider: BAREMETAL
  ssh:
    passwd:
    pk: xxx
    pkPasswd: xxx
    user: root
  network:
    podCIDR: 100.64.0.0/10
    svcCIDR: 10.96.0.0/22
  certSANS:
    - aliyun-inc.com
    - 10.0.0.2

  masters:
    ipList:
     - 172.20.126.4
     - 172.20.126.5
     - 172.20.126.6
  nodes:
    ipList:
     - 172.20.126.8
     - 172.20.126.9
     - 172.20.126.10
---
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
  name: LABEL
spec:
  type: LABEL
  data: |
     172.20.126.8 ssd=false,hdd=true
```

```shell script
sealer apply -f Clusterfile
```

## hostname plugin

HOSTNAME plugin will help you to change all the hostnames

```yaml
---
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
  name: HOSTNAME
spec:
  type: HOSTNAME # should not change this name
  data: |
     192.168.0.2 master-0
     192.168.0.3 master-1
     192.168.0.4 master-2
     192.168.0.5 node-0
     192.168.0.6 node-1
     192.168.0.7 node-2
```

## shell plugin

You can exec any shell command on specify node in any phase.

```yaml
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
  name: PostInstall.sh #Script file name
spec:
  type: SHELL
  action: PostInstall # PreInit PreInstall PostInstall
  on: 192.168.0.2-192.168.0.4 #or 192.168.0.2,192.168.0.3,192.168.0.7
  data: |
     kubectl taint nodes node-role.kubernetes.io/master=:NoSchedule
```

action: the phase of command.

* PreInit: before init master0.
* PreInstall: before join master and nodes.
* PostInstall: after join all nodes.

on: exec on witch node.

## label plugin

Help you set label after install kubernetes cluster

```yaml
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
  name: LABEL
spec:
  type: LABEL
  data: |
     192.168.0.2 ssd=true
     192.168.0.3 ssd=true
     192.168.0.4 ssd=true
     192.168.0.5 ssd=false,hdd=true
     192.168.0.6 ssd=false,hdd=true
     192.168.0.7 ssd=false,hdd=true
```

## Etcd backup

```yaml
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
  name: ETCD_BACKUP
spec:
  action: Manual
```

Etcd backup plugin is triggered manually: `sealer plugin -f etcd_backup.yaml`

## Define the default plugin in Kubefile to build the image and run it

In many cases it is possible to use plugins without using Clusterfile, essentially sealer stores the Clusterfile plugin configuration in the Rootfs/Plugin directory before using it, so we can define the default plugin when we build the image.

Plugin configuration shell.yaml:

```
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
name: taint
spec:
type: SHELL
action: PostInstall
on: role=master
data: |
kubectl taint nodes node-role.kubernetes.io/master=:NoSchedule
---
apiVersion: sealer.aliyun.com/v1alpha1
kind: Plugin
metadata:
  name: SHELL
spec:
  action: PostInstall
  on: role=node
  data: |
    if type yum >/dev/null 2>&1;then
    yum -y install iscsi-initiator-utils
    systemctl enable iscsid
    systemctl start iscsid
    elif type apt-get >/dev/null 2>&1;then
    apt-get update
    apt-get -y install open-iscsi
    systemctl enable iscsid
    systemctl start iscsid
    fi
```

Kubefile:

```shell script
FROM kubernetes:v1.19.8
COPY shell.yaml plugin
```

Build a cluster image that contains a taint plugin (or more plugins):

```shell script
sealer build -m lite -t kubernetes-taint:v1.19.8 .
```

Run the image and the plugin will also be executed without having to define the plug-in in the Clusterfile:
`sealer run kubernetes-taint:v1.19.8 -m x.x.x.x -p xxx`

## develop you own plugin

The plugin [interface](https://github.com/alibaba/sealer/blob/main/plugin/plugin.go)

```golang
type Interface interface {
	Run(context Context, phase Phase) error
}
```

[Example](https://github.com/alibaba/sealer/blob/main/plugin/labels.go):

```golang
func (l LabelsNodes) Run(context Context, phase Phase) error {
	if phase != PhasePostInstall {
		logger.Debug("label nodes is PostInstall!")
		return nil
	}
	l.data = l.formatData(context.Plugin.Spec.Data)

	return err
}
```

Then regist you [plugin](https://github.com/alibaba/sealer/blob/main/plugin/plugins.go):

```golang
func (c *PluginsProcesser) Run(cluster *v1.Cluster, phase Phase) error {
	for _, config := range c.Plugins {
		switch config.Name {
		case "LABEL":
			l := LabelsNodes{}
			err := l.Run(Context{Cluster: cluster, Plugin: &config}, phase)
			if err != nil {
				return err
			}
        // add you plugin here
		default:
			return fmt.Errorf("not find plugin %s", config.Name)
		}
	}
	return nil
}
```
