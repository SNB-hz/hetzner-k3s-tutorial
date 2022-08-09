## Hcloud Vorbereitung
    - Account erstellen
    - account zum projekt hinzufügen lassen
    - Zugänge einrichten:
        - ssh key im account erstellen
            -> ein key pro Person
            -> Namen des key merken, brauchen wir für server-Erstellung später
        - api token für nutzer im account erstellen
            -> tokeninhalt aufbewahren
        - api token für cloud controller erstellen: "controller-api-token"
            -> tokeninhalt aufbewahren
## Hcloud tool
- hcloud tool installieren
  - https://github.com/hetznercloud/cli
- hcloud tool für das projekt konfigurieren: 
  
  `hcloud context create <contextname, frei wählbar>`
   - anschließend token Inhalt von oben eingeben
    - hcloud tool testen
        - hcloud server list
## Server erstellen
- internes Netzwerk erstellen

 `hcloud network create --name network-kubernetes --ip-range 10.0.0.0/16`
 
 `hcloud network add-subnet network-kubernetes --network-zone eu-central --type server --ip-range 10.0.0.0/16`
      
 - ersten server erstellen
        - `hcloud server create --type cx21 --name server-1 --image ubuntu-20.04 --ssh-key < key-name > --network network-kubernetes --placement-group group-spread`
 
 
 - k3s auf server installieren
        - curl -sfL https://get.k3s.io | sh -s - server \
                --disable-cloud-controller \
                --disable metrics-server \
                --write-kubeconfig-mode=644 \
                --node-name="$(hostname -f)" \
                --cluster-cidr="10.244.0.0/16" \
                --kubelet-arg="cloud-provider=external" \
                --token="< hetzner-api-token >" \
                --tls-san="$(hostname -I | awk '{print $2}')" \
                --flannel-iface=ens10
        #- `curl -sfL https://get.k3s.io | sh -s - --disable-cloud-controller --no-deploy servicelb --kubelet-arg="cloud-provider=external"`
        --> pods stehen auf Pending wegen fehlendem ccm
    - hetzner cloud controller manager konfigurieren:
        - `kubectl -n kube-system create secret generic hcloud --from-literal=token=< controller-api-token > --from-literal=network=network-kubernetes`
        - kubectl apply -f https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm.yaml
        --> pods springen auf Running
    - bash completion für k8s befehle:
        - echo 'source <(kubectl completion bash)' >>~/.bashrc
            - https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/
    - k3s agents in den cluster joinen:
        - curl -sfL https://get.k3s.io | K3S_URL=https://65.108.148.75:6443 K3S_TOKEN=<k3s token> sh -s - --kubelet-arg="cloud-provider=external"
    - hello kubernetes deployment erstellen
        - https://raw.githubusercontent.com/SNB-hz/hetzner-k3s-tutorial/main/manifests/hello-kubernetes/deployment.yaml

                          
    - CSI driver installieren: 
        - https://github.com/hetznercloud/csi-driver/blob/main/README.md
