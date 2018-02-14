# SimpleK8s

Das ist ein einfaches Howto um ein Kubernetes aufzubauen.

* Controlplane mit einem Master
* Eine beliebige Nummer von Worker Nodes


## Elemente

### Controlplane/Master

Die Controlplane besteht aus

* docker
* Kubelet (systemd)
* etcd als Pod
* apiserver als Pod
* controller als Pod
* Scheduler als Pod

### Worker

Ein Worker besteht aus

* docker
* Kubelet (systemd)



# Installation

Wir gehen davon aus, dass alle Rechner schon existieren.

Als Filesystem sollte ext4 verwendet werden und Docker zur Verwendung von Overlay2 konfiguriert sein. (ungetestet)

~~~
$ cat /etc/docker/daemon.json 
{
   "storage-driver": "overlay2"
}
~~~


* firewalld soll disabled sein.
* selinux nicht scharf geschaltet


Bevor wir beginnen brauchen wir noch ein paar Cert-Bundles.
Dazu setzen wir eine eigene CA auf.
Jeder Worker/Kubelet braucht ein Bundle um sich zu authentifizieren.
Der API-Server braucht auch ein Bundle um von den Clients verifiziert zu werden.
(Auch wird sein Bundle verwendet um ServiceAccount-Tokens zu erstellen. )

Siehe dazu das README.md im Certs Verzeichnis.

## Controlplane/Master

Wenn alle Certs nach /etc/kubernetes/ssl kopiert sind. können wir mit dem Master loslegen.

Alle meint:
ca*.pem
apiserver*pem
$hostname*pem

### Docker

Wir installieren docker:

~~~
yum install -y  docker
systemctl enable docker
systemctl start docker
~~~

Swap muss ausgeschaltet werden.

~~~
swapoff -a
~~~

### Kubelet

Als nächstes starten wir das Kubelet als systemd-Unit. 
Damit wird auch gleich das Image für Kubernetes heruntergeladen.

Als Image verwenden wir hier: `gcr.io/google-containers/hyperkube:v1.9.2`

Dazu werden zwei Dateien benötigt:

~~~
/etc/systemd/system/kubelet.service
/etc/kubernetes/ssl/kubeconfig-node.yml
~~~

Dann noch den Service starten:

~~~
systemclt start kubelet
systemclt enable kubelet
~~~

### Controlplane


Nun werden folgende Dateien in das Verzeichnis `/etc/kubernetes/manifests` des Masters kopiert.

* etcd.pod            (Standalone etcd-server)
* apiserver.pod       (Unser API-Server)
* controller.pod
* scheduler.pod

Im etcd.pad

Das Kubelet liest die Datei und rollt die Pods aus. Bitte bei jeder Datei mit `docker ps` oder `docker ps -a` den Fortschritt prüfen.
Jedes Pod spawnt 2 Container. (Pause und hyperkube $option) 

Wenn es am Ende ein:

~~~
$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
~~~

ist bis jetzt alles gut.


# Install Worker.

Ein Worker benötigt docker,  sein Cert-Bundle und

~~~
/etc/systemd/system/kubelet.service
/etc/kubernetes/ssl/kubeconfig-node.yml
~~~

wobei kubelet.service angepasst werden muss. 
So muss der Taint raus und das label auf node gesetzt werden.


Kontrollieren ob sich der nodes angemeldet hat:

~~~
kubectl get nodes
~~~

# Hier ein paar ungetestete :/ Beispiele

Jetzt wo der Cluster "steht" kann nun via dem API-Server deployt werden.

Wir setzen die Rechte des ServiceAccounts des Namespaces kube-system.

Ganz bösartig wird der Serviceaccount (default) des Namespaces kube-system zum cluster-admin.

~~~
kubectl -f  Deploy/10-kube-system-serviceaccount.yml
~~~

## Kube-proxy

Der DaemonSet Kube-proxy rollt den Container aus, welcher die Iptables-Regeln (Servicenetz) auf allen Hosts schreibt.

~~~
kubectl -f  Deploy/20-kube-proxy.yaml
~~~

Den Status des DaemonSet ansehen:

~~~
kubectl get ds kube-proxy -n kube-system
~~~

# Netzwerk


Folgend wir Canal deployed

~~~
kubectl -f  Deploy/30-canal.yaml
~~~
