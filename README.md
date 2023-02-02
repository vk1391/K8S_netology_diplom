### Создание Kubernetes кластера
 - На подготовленной инфраструктуре был развернут Kubernetes по средствам Kubespray(K8s+calico+nginx-ingress-controller)
 - kubespray/inventory/mycluster/hosts.yaml
```
all:
  hosts:
    node1:
      ansible_host: 158.160.54.93
      ip: 192.168.15.15
      ansible_user: ubuntu
    node2:
      ansible_host: 158.160.50.3
      ip: 192.168.15.16
      ansible_user: ubuntu
    node3:
      ansible_host: 158.160.35.120
      ip: 192.168.15.17
      ansible_user: ubuntu
  children:
    kube_control_plane:
      hosts:
        node1:
    kube_node:
      hosts:
        node2:
        node3:       
    etcd:
      hosts:
        node1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
 ```
  - kubectl get nodes -o wide
 ```
NAME    STATUS   ROLES           AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node1   Ready    control-plane   6d19h   v1.24.6   192.168.15.15   <none>        Ubuntu 20.04.3 LTS   5.4.0-97-generic   containerd://1.6.8
node2   Ready    <none>          6d19h   v1.24.6   192.168.15.16   <none>        Ubuntu 20.04.3 LTS   5.4.0-97-generic   containerd://1.6.8
node3   Ready    <none>          6d19h   v1.24.6   192.168.15.17   <none>        Ubuntu 20.04.3 LTS   5.4.0-97-generic   containerd://1.6.8
```
### Подготовка cистемы мониторинга и деплой приложения
Созданы следующие неймспейсы с приложениями:
1. Content. В данном неймспейсе развернуто тестовое приложение по средствам deployment контроллера.Доступ из вне организован через NodePort service
```
kubectl get pods -n content -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
nginx-static-content-6d4fbdfd9b-5nkzs   1/1     Running   0          91m   10.233.75.50   node2   <none>           <none>
nginx-static-content-6d4fbdfd9b-5smk8   1/1     Running   0          91m   10.233.71.62   node3   <none>           <none>
```
```
kubectl get svc -n content -o wide
NAME                           TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE   SELECTOR
nginx-static-content-service   NodePort   10.233.57.152   <none>        80:30009/TCP   40h   app=nginx-static-content
```
![](https://github.com/vk1391/devops-netology/blob/5bc467b77417bcf2c47ea940c4af84ccd62834d0/content.jpg)
2. devops-tools. В данном неймспейсе развернут Jenkins по средствам deployment контроллера.Доступ из вне организован через NodePort service
```
NAME                      READY   STATUS    RESTARTS      AGE     IP             NODE    NOMINATED NODE   READINESS GATES
jenkins-fd5fdf49f-s5xjg   1/1     Running   0             2d20h   10.233.75.44   node2   <none>           <none>
```
```
kubectl get svc -n devops-tools -o wide
NAME              TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE     SELECTOR
jenkins-service   NodePort   10.233.0.179   <none>        8080:32000/TCP   2d20h   app=jenkins-server
```
![](https://github.com/vk1391/devops-netology/blob/5bc467b77417bcf2c47ea940c4af84ccd62834d0/k8s_jenkins.jpg)
3. atlantis. В данном неймспейсе развернут Pull-request automation tools Atlantis по средствам StatefullSet контроллера.Доступ из вне организован через NodePort service
```
NAME         READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
atlantis-0   1/1     Running   0          13m   10.233.75.51   node2   <none>           <none>
```
```
kubectl get svc -n atlantis -o wide
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE   SELECTOR
atlantis   NodePort   10.233.54.86   <none>        80:30003/TCP   21m   app=atlantis
```
![](https://github.com/vk1391/devops-netology/blob/5bc467b77417bcf2c47ea940c4af84ccd62834d0/k8s_atlantis.jpg)
4. monitoring.  В данном неймспейсе развернут kube-prometheus по средствам deployment контроллера.Доступ из вне организован через NodePort service
```
kubectl get pods -n monitoring -o wide
NAME                                   READY   STATUS    RESTARTS        AGE   IP              NODE    NOMINATED NODE   READINESS GATES
alertmanager-main-0                    2/2     Running   4 (15h ago)     20h   10.233.71.58    node3   <none>           <none>
alertmanager-main-1                    2/2     Running   4 (15h ago)     20h   10.233.75.45    node2   <none>           <none>
alertmanager-main-2                    2/2     Running   4 (15h ago)     20h   10.233.71.59    node3   <none>           <none>
blackbox-exporter-58c9c5ff8d-jhmn8     3/3     Running   6 (15h ago)     20h   10.233.71.52    node3   <none>           <none>
grafana-6b4547d9b8-5vhxk               1/1     Running   2 (15h ago)     20h   10.233.71.57    node3   <none>           <none>
kube-state-metrics-6d454b6f84-sbfw2    3/3     Running   8 (15h ago)     20h   10.233.71.51    node3   <none>           <none>
node-exporter-k52lf                    2/2     Running   4 (15h ago)     20h   192.168.15.16   node2   <none>           <none>
node-exporter-l2z72                    2/2     Running   8 (3h40m ago)   20h   192.168.15.15   node1   <none>           <none>
node-exporter-q6dt8                    2/2     Running   4 (15h ago)     20h   192.168.15.17   node3   <none>           <none>
prometheus-adapter-678b454b8b-64sxp    1/1     Running   4 (15h ago)     20h   10.233.71.50    node3   <none>           <none>
prometheus-adapter-678b454b8b-6nw22    1/1     Running   4 (15h ago)     20h   10.233.75.42    node2   <none>           <none>
prometheus-k8s-0                       2/2     Running   4 (15h ago)     20h   10.233.75.46    node2   <none>           <none>
prometheus-k8s-1                       2/2     Running   4 (15h ago)     20h   10.233.71.56    node3   <none>           <none>
prometheus-operator-7bb9877c54-kjmjt   2/2     Running   5 (15h ago)     20h   10.233.71.53    node3   <none>           <none>
```
```
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE   SELECTOR
alertmanager-main       ClusterIP   10.233.11.24    <none>        9093/TCP,8080/TCP            20h   app.kubernetes.io/component=alert-router,app.kubernetes.io/instance=main,app.kubernetes.io/name=alertmanager,app.kubernetes.io/part-of=kube-prometheus
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   20h   app.kubernetes.io/name=alertmanager
blackbox-exporter       ClusterIP   10.233.4.13     <none>        9115/TCP,19115/TCP           20h   app.kubernetes.io/component=exporter,app.kubernetes.io/name=blackbox-exporter,app.kubernetes.io/part-of=kube-prometheus
grafana                 NodePort    10.233.35.173   <none>        3000:30010/TCP               20h   app.kubernetes.io/component=grafana,app.kubernetes.io/name=grafana,app.kubernetes.io/part-of=kube-prometheus
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP            20h   app.kubernetes.io/component=exporter,app.kubernetes.io/name=kube-state-metrics,app.kubernetes.io/part-of=kube-prometheus
node-exporter           ClusterIP   None            <none>        9100/TCP                     20h   app.kubernetes.io/component=exporter,app.kubernetes.io/name=node-exporter,app.kubernetes.io/part-of=kube-prometheus
prometheus-adapter      ClusterIP   10.233.44.6     <none>        443/TCP                      20h   app.kubernetes.io/component=metrics-adapter,app.kubernetes.io/name=prometheus-adapter,app.kubernetes.io/part-of=kube-prometheus
prometheus-k8s          ClusterIP   10.233.43.233   <none>        9090/TCP,8080/TCP            20h   app.kubernetes.io/component=prometheus,app.kubernetes.io/instance=k8s,app.kubernetes.io/name=prometheus,app.kubernetes.io/part-of=kube-prometheus
prometheus-operated     ClusterIP   None            <none>        9090/TCP                     20h   app.kubernetes.io/name=prometheus
prometheus-operator     ClusterIP   None            <none>        8443/TCP                     20h   app.kubernetes.io/component=controller,app.kubernetes.io/name=prometheus-operator,app.kubernetes.io/part-of=kube-prometheus
```
![](https://github.com/vk1391/devops-netology/blob/5bc467b77417bcf2c47ea940c4af84ccd62834d0/k8s_grafana.jpg)
