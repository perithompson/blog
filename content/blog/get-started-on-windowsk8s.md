---
title: "Get Started on k8s with Windows with Clusterctl"
date: 2021-02-24T08:49:35Z
draft: true
---

## Background

## What we need to get going

To stop this becoming too long I won't go through installing each of the tools but listed below are the minimum you need to get started, it is worth noting that clusterctl can consume a large number of CPUs in azure so you may find you need to increase yoou allocation if you want to start there.  I am going to be using a preconfigured vsphere environment but the premise is the same!

- Image-builder
- Clusterctl
- ISOs
    - Windows Server 2019 *or later*
    - Vmware tools
- Docker
- Kind
- Vsphere Environment or Public cloud account (for this blog we are going to use vsphere)


# Building a Windows Image with ImageBuilder
Before you can build a windows cluster you need an template image loaded with all the tools Clusterctl needs in order to work.  Fortunately this is made easy by the use of image-builder[^1].

1. Create your variables file

2. Run Docker

## Clusterctl

``` bash 
export KIND_EXPERIMENTAL_DOCKER_NETWORK=bridge

kind create cluster

# The username used to access the remote vSphere endpoint
export VSPHERE_USERNAME="administrator@vsphere.local"
# The password used to access the remote vSphere endpoint
# You may want to set this in ~/.cluster-api/clusterctl.yaml so your password is not in
# bash history
export VSPHERE_PASSWORD="VMWare1\!"

# Finally, initialize the management cluster
clusterctl init --infrastructure vsphere

clusterctl config cluster capi-win \
    --infrastructure vsphere:v0.7.4 \
    --kubernetes-version v1.20.1 \
    --control-plane-machine-count 1 \
    --worker-machine-count 1 > cluster.yaml

```

```yaml
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: VSphereMachineTemplate
metadata:
  name: capi-win-windows
  namespace: default
spec:
  template:
    spec:
      cloneMode: linkedClone
      datacenter: metal-datacenter
      datastore: esx2_ds
      diskGiB: 80
      folder: perit
      memoryMiB: 8192
      network:
        devices:
        - dhcp4: true
          networkName: VM Network
      numCPUs: 4
      resourcePool: perit
      server: 10.176.37.10
      template: Windows-2019-kube-v1.20.1
      thumbprint: D7:B5:BD:2D:5D:5B:2D:1A:64:E2:10:5E:F2:D6:0F:45:71:82:BB:54
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
kind: KubeadmConfigTemplate
metadata:
  name: capi-win-md-win0
  namespace: default
spec:
  template:
    spec:
      jjoinConfiguration:
        nodeRegistration:
          criSocket: npipe:////./pipe/containerd-containerd
          kubeletExtraArgs:
            cloud-provider: external
            register-with-taints: os=windows:NoSchedule
          name: '{{ ds.meta_data.hostname }}'
      files:
      - path: 'c:\k\antrea\antrea-startup.ps1'
        content: |
          $service = Get-Service -Name ovs-vswitchd -ErrorAction SilentlyContinue
          Push-Location C:\k\antrea
          if($service -eq $null) {
            & ./Install-OVS.ps1
            Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
            & nssm install kube-proxy "c:/k/kube-proxy.exe" "--proxy-mode=userspace --kubeconfig=C:/etc/kubernetes/kubelet.conf --log-dir=c:/var/log/kube-proxy --logtostderr=false --alsologtostderr"
            & nssm install antrea-agent "c:/k/antrea/bin/antrea-agent.exe" "--config=c:/k/antrea/etc/antrea-agent.conf --logtostderr=false --log_dir=c:/k/antrea/logs --alsologtostderr --log_file_max_size=100 --log_file_max_num=4"
            & nssm set antrea-agent DependOnService kube-proxy ovs-vswitchd
            & nssm set antrea-agent Start SERVICE_DELAYED_START
            start-service kube-proxy
            start-service antrea-agent
          }
      - path: 'C:\Temp\antrea.ps1'
        content: |
          $service = Get-Service -Name ovs-vswitchd -ErrorAction SilentlyContinue
          if($service -ne $null) {
            exit
          }
          invoke-expression "bcdedit /set TESTSIGNING ON"
          New-Item -ItemType Directory -Force -Path C:\k\antrea
          New-Item -ItemType Directory -Force -Path C:\k\antrea\logs
          New-Item -ItemType Directory -Force -Path C:\k\antrea\bin
          New-Item -ItemType Directory -Force -Path C:\var\log\kube-proxy
          [Environment]::SetEnvironmentVariable("NODE_NAME", (hostname).ToLower())
          $trigger = New-JobTrigger -AtStartup
          $options = New-ScheduledJobOption -RunElevated
          Register-ScheduledJob -Name PrepareAntrea -Trigger $trigger -FilePath 'c:\k\antrea\antrea-startup.ps1' -ScheduledJobOption $options
          $env:HostIP = (
              Get-NetIPConfiguration |
              Where-Object {
                  $_.IPv4DefaultGateway -ne $null -and $_.NetAdapter.Status -ne "Disconnected"
              }
          ).IPv4Address.IPAddress
          $file = 'C:\var\lib\kubelet\kubeadm-flags.env'
          $newstr="--node-ip=" + $env:HostIP
          $raw = Get-Content -Path $file -TotalCount 1
          $raw = $raw -replace ".$"
          $new = "$($raw) $($newstr)`""
          Set-Content $file $new
          $nssm = (Get-Command nssm).Source
          $serviceName = 'Kubelet'
          & $nssm set $serviceName start SERVICE_AUTO_START
          cd c:\k\antrea
          $TAG = "v0.13.0"
          curl.exe -LO "https://raw.githubusercontent.com/vmware-tanzu/antrea/${TAG}/hack/windows/Install-OVS.ps1"
          curl.exe -LO "https://raw.githubusercontent.com/vmware-tanzu/antrea/${TAG}/hack/windows/Helper.psm1"
          curl.exe -LO "https://github.com/vmware-tanzu/antrea/releases/download/${TAG}/antrea-agent-windows-x86_64.exe"
          mv antrea-agent-windows-x86_64.exe c:/k/antrea/bin/antrea-agent.exe
          Import-Module ./helper.psm1
          & Install-AntreaAgent -KubernetesVersion "v1.20.1" -KubernetesHome "c:/k" -KubeConfig "C:/etc/kubernetes/kubelet.conf" -AntreaVersion "${TAG}" -AntreaHome "c:/k/antrea"
          New-KubeProxyServiceInterface
          Restart-Computer -Force
      postKubeadmCommands:
        - powershell C:/Temp/antrea.ps1 -ExecutionPolicy Bypass
      users:
      - name: capv
        groups: Administrators
        sshAuthorizedKeys:
        - 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCeWhZAGsKC7VezEIpVhzX9fDfxZ80/oIbO03DRVrLBlhgGdewRiHcZQDvOsHaEReKsM2HOFFH108s1e2bRbf+ZMyAlSLgq+Q3gTWzUD8Lu3UuW7ANk2CAaiAI4+3cPG+eCUsvKngWliZ4dDaTFv+kJ8mgiAZmfIJK7KRzVUM52V48pWp3+ve+IEG5NkHVvIwbOYtKj5UiQ+/MPKCh7WOyFuvT6bMSRHgfreCOeuu2AydiqkhQIKFBjtqrwutaKs8wXOlK1RzSXJttv31hQGN9+mFanDJNIziqZe9BJztE0p2hFdNt4WqkGsaC/Yqt8eDGW0UqnWn+MuGl7awTQDBoP4JV5tjZ5N9AF00Jgp/zYSL4o7m1jrIAN0Nx0/2GT/UmhdMiCgRL7vxKVNrKFEbkcMxmrgsE/MhMYLnKB9L5amZ3uNvZpNZRyCRjjiXqQAzOm0OPS8MS0G9xBmrNx+joNYK+z5q+1PMmKd9/8h2t96/yTJLBWynKWzXzyj4ra3gscGpmqf+fzkVixziW7pbyi2+e/B+idAS/d3oGq5xixHYn+/PcbUSnMitOmJ1rmQf4HfcHYokxgNnJ5Zeikn7hYjy+tjEKtk8eX2WNtLcuOmJ0IVo5fUWfeph6nWSPwNUCRElVSR26j7xW0+bm0swmZkhnUzl76PTrmfXuyXg+AtQ== perit@vmware.com'
        sudo: ALL=(ALL) NOPASSWD:ALL
---
apiVersion: cluster.x-k8s.io/v1alpha3
kind: MachineDeployment
metadata:
  labels:
    cluster.x-k8s.io/cluster-name: capi-win
  name: capi-win-md-win0
  namespace: default
spec:
  clusterName: capi-win
  replicas: 1
  selector:
    matchLabels: {}
  template:
    metadata:
      labels:
        cluster.x-k8s.io/cluster-name: capi-win
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
          kind: KubeadmConfigTemplate
          name: capi-win-md-win0
      clusterName: capi-win
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
        kind: VSphereMachineTemplate
        name: capi-win-windows
      version: v1.20.1
```

[^1]: https://github.com/kubernetes-sigs/image-builder