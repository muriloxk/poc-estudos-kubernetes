# Criando e entendendo pods

## Imperativa: 

`> kubectl run nome-do-pod --image=nginx:latest`

`> kubectl get pods || kubectl get pods -o wide`

`> kubectl describe pod nome-do-pod`

`> kubectl edit pod nome-do-pod`

`> kubectl delete pod nome-do-pod || kubectl delete -f primeiro-pod.yaml`

`> kubectl delete pods --all`

`> kubectl delete svc --all`

`> kubectl exec -it nome-do-pod -- bash`

## Declarativa: 

> cat \>  primeiro-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata: 
 name: primeiro-pod-declarativo
spec: 
  containers:
    -name: nginx-container
    image: nginx:latest
```

`> kubectl apply -f primeiro-pod.yaml`

-- Para validar o arquivo yaml (https://github.com/instrumenta/kubeval)

# Expondo pods com services (svc) 

1. Abstrações para expor applicações executando um ou mais pods
2. Proveem IP's fixos para comunicação
3. Proveem um DNS para um ou mais pods
4. São capazes de fazer balanceamento de carga

## **Existem três tipos de serviço: ClusterIP, NodePort e LoadBalancer**

### 1. **ClusterIP**: Fornece apenas comunicação interna do cluster. 

Exemplo:
Vamos criar dois pods (pod-1 e pod-2) e vamos criar um service para o pod-2 
através de labels

> cat \> portal-noticias.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: portal-noticias
  labels:
    app: portal-noticias
spec:
  containers:
    - name: container-portal-noticias
      image: nginx:latest
      ports:
        - containerPort: 80
```

> cat \> pod-1.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
spec:
  containers:
    - name: container-pod-1
      image: nginx:latest
      ports:
        - containerPort: 80
```

> cat \> pod-2.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-2
spec:
  containers:
    - name: container-pod-2
      image: nginx:latest
      ports:
        - containerPort: 80
```

> cat \> svc-pod-2.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-pod-2
spec:
  type: ClusterIP
  selector: 
    app: segundo-pod
  ports: 
    - port: 80  # Porta em que o service está ouvindo
    targetPort: 80 # Onde vai ser dispachado.
```

`> kubectl apply -f pod-1.yaml`

`> kubectl apply -f pod-2.yaml`

`> kubectl apply -f svc-pod-2.yaml`

`> kubectl get svc` 

`> kubectl get pods`

`> kubectl exec -it pod-1 -- bash`

`> curl <ip do svc-pod-2>:80`

**Perguntas:** 
1. E se eu tiver dois pods que possui a label para dar match com o selector do service? <br/>
R: Parece que ele faz um Load balancing decidindo entre um dos dois pods. 

2. E se eu derrubar o pod que está ouvindo o que eu acontece? <br/>
R: O service continua ativo, mas da erro de conexão por ninguém estar ouvindo.

### NodePort

Abre a comunicação do nó com o mundo externo e também funcionam como ClusterIP.

<!-- ![Image of NodePort](node_port.png) -->

> cat \> svc-pod-1.yaml

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: svc-pod-1
spec:
  type: NodePort 
  ports:
    - port: 80  
    # Como definimos apenas o port, ele define automaticamente
    # o targer port para 80;
    nodePort: 3000 #between 30000-32767
  selector:
    app: primeiro-pod
```

> cat \> pod-1.yaml 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    app: primeiro-pod # inclui a label para ser selecionado.
spec:
  containers:
    - name: container-pod-1
      image: nginx:latest
      ports:
        - containerPort: 80
```

`> kubectl apply -f svc-pod-1.yaml`

`> kubectl apply -f portal-noticias.yaml`

`> kubectl apply -f pod-1.yaml`

`> kubectl get svc`

`> kubectl exec -it portal-noticias -- bash`

`> curl <ip do node port/svc-pod-1>:80`


### Load Balancer 

Mesma funcionalidade que o NodePort, mas também faz balanceamento de carga e ele se integra com o cloud provider (Ex: AWS, Azure, Google Cloud..)

> cat \> svc-pod-1-loadbalancer.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-pod-1-loadbalancer
spec:
  type: Loadbalancer
  ports:
    - port: 80
      nodePort: 3000
  selector:
    app: primeiro-pod
```

## ConfigMap e Environment Variables

Exemplo de variavéis de ambiente direto no arquivo yaml: 

> cat \> exemplo.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  labels:
    app: db-pod
spec:
  containers:
    -name: db-pod-container
    image: enderecoregistry/mysql-db:1
    ports:
      - containerPort: 3306
    env:
      - name: "MYSQL_ROOT_PASSWORD"
      value: "exemplo"
      - name: "MYSQL_DATABASE"
      value: "exemplo"
      - name: "MYSQL_PASSWORD"
      value: "exemplo"
```

### Configurando ConfigMaps

Um pod pode ter um ou mais config maps, os config maps podem ser reutilizado entre os pods.

docs: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/

Exemplo de ConfigMap: 

> cat \> db-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-configmap
data:
   MYSQL_ROOT_PASSWORD: "exemplo"
   MYSQL_DATABASE: "exemplo"
   MYSQL_PASSWORD: "exemplo"
```


> cat \> app-configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-configmap
data:
  HOST_DB: "exemplo"
  USER_DB: "root"
  PASS_DB: "exemplo"
```

> cat \> exemplo.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-pod
  labels:
    app: db-pod
spec:
  containers:
    -name: db-pod-container
    image: enderecoregistry/mysql-db:1
    ports:
      - containerPort: 3306
    # Modo uma a uma:
    #env:
      # - name: "MYSQL_ROOT_PASSWORD"
      # valueFrom: 
      #   configMapKeyRef:
      #     name: db-configmap
      #     key: MYSQL_ROOT_PASSWORD
      # - name: "MYSQL_DATABASE"
      # valueFrom: 
      #   configMapKeyRef:
      #     name: db-configmap
      #     key: MYSQL_DATABASE

      #Todas de uma vez:
      envFrom:
        - configMapRef:
          name: db-configmap
        - configMapRef:
          name: app-configmap
```

> Todos os exemplos até aqui estão na pasta "basico"

## ReplicaSets e Deployments

 ![Image of NodePort](imgs/replica_set_deployments.png)

### ReplicaSet

Caso um pod falhe, por ser efemero, como criar outro para assumir o lugar de maneira automatica? 

Para isso utilizamos o **Replica Set**, ele é uma estrutura que pode encapsular um ou mais pods. Logo um desses pods podem falhar e o replica set vai criar um novo automaticamente.

### Deployment

É uma camada acima do replica set, quando definimos um Deployment também definimos um replica set. 

Exemplo: 
> cat \> nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      name: nginx-pod
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx-container
          image: nginx:stable
          ports:
            - containerPort: 80
  selector:
    matchLabels:
      app: nginx-pod
```

`> kubectl apply -f nginx-deployment.yaml`

`> kubectl get rs`

`> kubectl get deployments`

### History/Versionamento do deployment

Com o deployment ganhamos um histórico e versionamento das versões, exemplo:

1. Executar a alteração e gravar no histórico:

`> kubectl apply -f nome-deployment.yaml` --record

2. Para criar um comentário sobre a causa de alteração

`> kubectl annotate deployment <nome do deployment> kubernetes.io/change-cause="<Comentário sobre a causa da mudança>"`

3. Comando para listar:

`> kubectl rollout history deployment <nome do deployment>`

Com isso podemos fazer um *"rollback"* de versões, chamando de **undo** 

`> kubectl rollout undo deployment <nome do deployment> --to-revision=<identificador da revisão>`

## Persistindo dados com volumes (vol, pv, pvc sc)

docs: https://kubernetes.io/docs/concepts/storage/volumes/
microsoft docs: https://docs.microsoft.com/pt-br/azure/aks/concepts-storage#volumes
video explicação aprofundada e cases: youtube.com/watch?v=0swOh5C3OVM

### volumes

Volumes possuem ciclos de vida independente dos containers, porém são dependentes do pod.

Exemplo: Um pod com dois containers, se um container falhar o pod ainda está vivo, logo o volume ainda resiste, já se os dois containers falharem, o pod vai ser destruido e o volume junto com ele.

Possuimos várias integrações de volumes:

- awsElasticBlockStore
- azureDisk
- azureFile
- cephfs
- cinder
- configMap
- downwardAPI
- emptyDir
- fc (fibre channel)
- flocker (deprecated)
- gcePersistentDisk
- gitRepo (deprecated)
- glusterfs
- hostPath
- iscsi
- local
- nfs
- persistentVolumeClaim
- portworxVolume
- projected
- quobyte
- rbd
- scaleIO (deprecated)
- secret
- storageOS
- vsphereVolume

Exemplo com hostPath, criando um pod com dois containers compartilhando um mesmo volume com binding para a maquina do host. 

> cat \> pod-volume.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-volume
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - mountPath: /volume-dentro-do-container
          name: primeiro-volume
    - name: jenkins-container
      image: jenkins:alpine
      volumeMounts:
        - mountPath: /volume-dentro-do-container
          name: primeiro-volume
  volumes:
    - name: primeiro-volume
      hostPath:
        path: <algum diretorio do seu computador>
        type: DirectoryOrCreate
```

*Observação: Nesse caso, caso seja deletado o pod, o volume também será excluido, porem, os arquivos que estão na maquina do host não.*

### Persistent Volume and Persistent Volume Claim (pv, pvc) 

O volume separa o armazenamento do recipiente. Seu ciclo de vida é acoplado a um pod. Ele permite reinicializações seguras de contêineres e compartilhamento de dados entre contêineres em um pod.

O **Persistent Volume** separa o armazenamento do pod. Seu ciclo de vida é independente. Ele permite reinicializações seguras de pods e compartilhamento de dados entre eles.

**PVC** é uma declaração de necessidade de armazenamento que pode em algum momento tornar-se disponível - como em vinculado a alguns reais PV.

É um pouco como o conceito de programação assíncrona de um promessa ou uma interface. PVC promete que em algum momento"traduza" para o volume de armazenamento que seu aplicativo poderá usar e uma das características definidas como classe, tamanho e modo de acesso.

![Image of PersistentVolumes](imgs/pv-pvc2.png)

E para que todas essas abstrações? 

Desacoplamento da aplicação com a infraestrutura e posso separar os papeis responsaveis de administrador da infre e devops, um faz a claim e o administrador (especialista) cria a estrutura do volume.

Exemplo:

1. Pod com a necessidade de um storage/volume linkado ao um pvc.

> cat \> pod-pv.yaml

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pv
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - mountPath: /volume-dentro-do-container
          name: primeiro-pv
  volumes:
    - name: primeiro-pv
      persistentVolumeClaim:
        claimName: pvc-1
```

2. PVC declarando um *"contrato como se fosse uma interface"*

> cat \> pvc.yaml

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

3. PV declarado para atender o pvc e linkado ao um disco criado na google cloud

> cat \> pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-1
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  gcePersistentDisk:
    pdName: pv-disk # nome do disco na google cloud
  storageClassName: standard
```

4. Aplicando:

`> kubectl apply -f pv.yaml` 

`> kubectl apply -f pvc.yaml` 

`> kubectl apply -f pod-pv.yaml` 

5. Assim temos um pod utilizando como volume um disco da google cloud.

### Utilizando Storage Classes

Storage classes são utilizados pelo pvc para criar um disco e um pv dinamicamente.

> cat \> sc.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  fstype: ext4
  replication-type: none
```

> cat \> pvc-sc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-sc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: slow #linkando com o storage class criado.
```

> cat \> pod-sc.yaml

```yaml 
apiVersion: v1
kind: Pod
metadata:
  name: pod-sc
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      volumeMounts:
        - mountPath: /volume-dentro-do-container
          name: primeiro-pv
  volumes:
    - name: primeiro-pv
      persistentVolumeClaim:
        claimName: pvc-sc
```

`> kubectl apply -f sc.yaml`

`> kubectl apply -f pvc-sc.yaml`

`> kubectl apply -f pod-sc.yaml`


















