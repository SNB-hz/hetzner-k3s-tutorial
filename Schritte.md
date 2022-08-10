## Hcloud Vorbereitung

diese Schritte erfolgen im hcloud GUI: https://console.hetzner.cloud/

- Account erstellen

- account zum projekt hinzufügen lassen

- Zugänge einrichten:
  - ssh key im account erstellen
    -> ein key pro Person
    -> Namen des key merken, brauchen wir für server-Erstellung später
  - api token "api-token-nutzer" für nutzer im account erstellen
    -> tokeninhalt aufbewahren
  - api token "api-token-controller" für cloud controller erstellen
    -> tokeninhalt aufbewahren

## Hcloud tool

- hcloud tool installieren
  - https://github.com/hetznercloud/cli

- hcloud tool für das projekt konfigurieren: 
  
  `hcloud context create <contextname, frei wählbar>`

   - anschließend Inhalt von "api-token-nutzer" angeben

- hcloud tool testen
  
  `hcloud server list`

## Infrastruktur vorbereiten

- internes Netzwerk erstellen

  `hcloud network create --name network-kubernetes --ip-range 10.0.0.0/16`
 
  `hcloud network add-subnet network-kubernetes --network-zone eu-central --type server --ip-range 10.0.0.0/16`
    
- placement group erstellen

  stellt sicher, dass nicht alle VMs auf dem gleichen physikalischen host landen

  'hcloud placement-group create --name group-spread --type spread'

## Initiales k3s setup

### VM erstellen
  
- ersten server erstellen

  diese VM wird als "control-plane" dienen, mit der wir den k3s cluster verwalten
  bei der Erstellung können ssh keys angegeben werden, die zur VM hinzugefügt werden sollen. Ein automatisches hinzufügen via hcloud ist zu einem späteren Zeitpunkt nicht möglich.

  `hcloud server create --type cx21 --name server-1 --image ubuntu-20.04 --ssh-key < key-name > --network network-kubernetes --placement-group group-spread`
 
- prüfen ob server richtig erstellt wurde

  `hcloud server list` 

### k3s setup

- per SSH auf den server verbinden

  `ssh root@< server-1 public ip > -i < Speicherort ssh key >`

- k3s auf server installieren
  
  hier geben wir ein secret token unserer Wahl an, mit dem sich später weitere k3s nodes authentifizieren wenn sie dem cluster beitreten wollen


  ```
  curl -sfL https://get.k3s.io | sh -s - server \
                --disable-cloud-controller \
                --disable metrics-server \
                --write-kubeconfig-mode=644 \
                --node-name="$(hostname -f)" \
                --cluster-cidr="10.244.0.0/16" \
                --kubelet-arg="cloud-provider=external" \
                --token="< k3s-secret-token >" \
                --tls-san="$(hostname -I | awk '{print $2}')" \
                --flannel-iface=ens10
  ```

- cluster status prüfen

  es kann wenige Minuten dauern bis k3s startet

  `kubectl get nodes`

  `kubectl get pods -A`

    --> einige pods stehen auf `Pending`, da wir den standard cloud-controller disabled haben, aber noch keinen Ersatz konfiguriert haben. Das machen wir jetzt:


- hetzner cloud controller manager konfigurieren

  erlaubt k8s, mit unserem Hetzner Cloud Projekt zu kommunizieren

  - Ein Kubernetes secret erstellen, welches das api-token für den Controller enthält, welches wir zu Beginn erstellt haben:

    `kubectl -n kube-system create secret generic hcloud --from-literal=token=< controller-api-token > --from-literal=network=network-kubernetes`
  
  - das manifest für den Hetzner Cloud Controller Manager herunterladen und installieren
      
    `kubectl apply -f https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm.yaml`

- pods nochmals prüfen

  `kubectl get pods -A`

  --> nach kurzer Zeit sollten alle Pods im Zustand `Running` stehen


- bash completion für k8s Befehle einrichten:
        
  echo 'source <(kubectl completion bash)' >>~/.bashrc
  
  mehr Infos: https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/


Wir haben jetzt einen initialen k3s cluster, der aber nur aus der control-plane besteht und noch nicht Anwendungen betreiben kann.
Deshalb fügen wir jetzt noch 2 worker hinzu, die unsere Anwendungen betreiben.
Anschließend setzen wir einen Hetzner Cloud Loadbalancer ein, um traffic richtig zu unserem cluster zu schicken

## den k3s cluster vervollständigen

### worker nodes / "agents" zum cluster hinzufügen

- agent VMs erstellen

  `hcloud server create --type cx21 --name agent-1 --image ubuntu-20.04 --ssh-key < key-name > --network network-kubernetes --placement-group group-spread`

  `hcloud server create --type cx21 --name agent-2 --image ubuntu-20.04 --ssh-key < key-name > --network network-kubernetes --placement-group group-spread

- prüfen ob VMs richtig erstellt wurden

  `hlcoud server list`


 - k3s agents in den cluster joinen:

   Diese Schritte wiederholen wir für jeden der agent server

   wir geben hier wieder das k3s-secret-token an, das wir beim erstellen von server-1 angegeben haben. Damit können sich die agents bei server-1 authentifizieren

   server-1 sollte die IP 10.0.0.2 erhalten haben. Falls nicht, ggf. die IP hier ändern.

   ```
   curl -sfL https://get.k3s.io | K3S_URL=https://10.0.0.2:6443 K3S_TOKEN=< k3s-secret-token > sh -s - agent \
    --node-name="$(hostname -f)" \
    --kubelet-arg="cloud-provider=external" \
    --flannel-iface=ens10
   ```   

- nachdem agent-1 und agent-2 dem cluster beigetreten sind, den cluster status prüfen

  - per ssh auf server-1 verbinden

  - den cluster status auf server-1 prüfen:

    `kubectl get nodes`

    --> server-1, agent-1 und agent-2 sollten als `Ready` gelistet sein

Wir haben jetzt einen cluster mit control-planes und worker nodes in einem internen Netzwerk, aber noch keinen Weg eingehenden traffic ordentlich zu behandeln.

### Traefik für proxy protocol konfigurieren

Proxy protocol aktivieren, und vorerst allen IPs vertrauen.
Normalerweise würden wir hier nur den Loadbalancer als trusted IP angeben, aber das funktioniert aufgrund eines bugs momentan nicht.

```
cat <<EOF > /var/lib/rancher/k3s/server/manifests/traefik-config.yaml
      - "--entryPoints.web.proxyProtocol.insecure"
      - "--entryPoints.web.forwardedHeaders.insecure"
EOF
```

### Loadbalancer anlegen

- den loadbalancer erstellen, und zu unserem privaten Netzwerk hinzufügen

```
hcloud load-balancer create --type lb11 --location nbg1 --name lb-kubernetes'
hcloud load-balancer attach-to-network --network network-kubernetes --ip 10.0.0.254 lb-kubernetes`
```

- dem Loadbalancer unsere k3s nodes als "targets" hinzufügen

```
hcloud load-balancer add-target lb-kubernetes --server master-1 --use-private-ip
hcloud load-balancer add-target lb-kubernetes --server node-1 --use-private-ip
hcloud load-balancer add-target lb-kubernetes --server node-2 --use-private-ip
```

### SSL Zertifikat anlegen 

Hetzner Cloud erlaubt die automatische Verwaltung von kostenlosen Lets Encrypt Zertifikaten.
Einzige Voraussetzung ist, dass DNS für die Domain von Hetzner's DNS Panel verwaltet wird.

Entweder diese Schritte für jede Subdomain wiederholen, oder ein Wildcard Zertifikat für "*.eigene.domain" anlegen


- das Zertifikat anlegen
  
  `hcloud certificate create --domain <example.com> --type managed --name cert-t1`
  
  das erstellen des Zertifikats kann wenige Minuten dauern

- das Zertifikat prüfen und die Zertifikats-ID notieren

  `hcloud certificate list`

### finale Loadbalancer konfiguration

`hcloud load-balancer add-service lb-kubernetes --protocol https --http-redirect-http --proxy-protocol --http-certificates <certificate_id>`

`hcloud load-balancer update-service lb-kubernetes --listen-port 443 --health-check-http-domain <example.com>`


Damit ist das k3s setup komplett: Der LB nimmt unter der angegebenen domain traffic via https an, und leitet ihn über http an unseren cluster weiter.



## Anwendungen deployen

Zum testen des setups deployen wir eine einfache Hallo-Welt Anwendung.

- sicherstellen dass im DNS ein A record für die gewünschte testdomain existiert, der auf die public Loadbalancer IP zeigt

- Manifest herunteladen

  `wget https://raw.githubusercontent.com/SNB-hz/hetzner-k3s-tutorial/main/manifests/hello-kubernetes/deployment.yaml`

- im Manifest die domain ersetzen, für die das Zertifikat und DNS eingestellt wurde

- das Manifest anwenden

  `kubectl apply -f deployment.yaml`

- pods prüfen

  `kubectl get pods`

- Test-domain aufrufen
