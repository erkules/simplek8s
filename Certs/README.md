# Erstelle eine Root-CA

Key:

~~~
openssl genrsa -out ca-key.pem 4096
~~~

Certificate:

~~~
openssl req -x509 -new -nodes -key ca-key.pem -days 1000 -out  ca.pem -subj "/CN=kube-ca"
~~~

Wir brauchen ein Cert für den API-Server. Die Nodes/Worker und die Clients überprüfen die Echtheit
des API-Servers.

In der `api-openssl.cnf` werden zwei Umgebungsvariablen abgefragt:

* `MASTERDNS` DNS/fqdn des Master/API-Server
* `MASTERIP`  IP-Addresse des Master/API-Server


~~~
openssl genrsa -out apiserver-key.pem 4096
MASTERDNS=... MASTERIP=.. openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config api-openssl.cnf
MASTERDNS=... MASTERIP=.. openssl x509 -req -in apiserver.csr -CA    ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 500 -extensions v3_req -extfile api-openssl.cnf
~~~

Dies Certs alle bitte auf den Master (/etc/kubernetes/ssl) kopieren.


# Host/Kubelet-Keys

Jedes Kubelet verbindet sich mit dem API-Server. 
Damit ein Kubelet sich mit dem API-Server verbinden darf braucht es selbst ein Cert.

Für jedes Kubelet/Host wird ein Cert-Bundle erstellt. Hierfür müssen wir noch je Iteration IP und FQDN setzen.

~~~
export KUBELETIP=
export KUBELETFQDN=
openssl genrsa -out ${KUBELETFQDN}-key.pem 4096
openssl req -new -key ${KUBELETFQDN}-key.pem -out ${KUBELETFQDN}-key.csr -subj "/CN=system:node:${KUBELETFQDN}/O=system:nodes" -config kubelet-openssl.cnf
openssl x509 -req -in    "${KUBELETFQDN}-key.csr" -CA    ca.pem -CAkey ca-key.pem -CAcreateserial -out "${KUBELETFQDN}.pem" -days 500 -extensions v3_req -extfile kubelet-openssl.cnf
~~~
 
Jeder Host bekommt sin Cert-Bundle auch noch nach /etc/kubernetes/ssl kopiert. 
Die Worker brauchen noch das ca.pem kopiert.
