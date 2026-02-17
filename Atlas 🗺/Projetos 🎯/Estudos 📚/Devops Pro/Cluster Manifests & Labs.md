## Cheatsheet Kubernetes Deployment

### Template de Deployment “completo” (com itens que você sempre esquece)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
  namespace: default
  labels:
    app: minha-app
spec:
  replicas: 2                       # escala horizontal simples (HPA faz isso melhor)
  revisionHistoryLimit: 5           # quantos ReplicaSets antigos manter
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0             # garante zero downtime (se tiver capacidade)
      maxSurge: 1
  selector:
    matchLabels:
      app: minha-app
  template:
    metadata:
      labels:
        app: minha-app
      annotations:
        prometheus.io/scrape: "true"      # exemplo: scraping simples
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: default          # troque se usar RBAC/IRSA/etc
      terminationGracePeriodSeconds: 30    # tempo para finalizar com elegância
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: minha-app
          image: minhaorg/minha-app:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080

          # (A) Variáveis de ambiente (ConfigMap/Secret)
          env:
            - name: APP_ENV
              value: "prod"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minha-app-secret
                  key: DB_PASSWORD
          envFrom:
            - configMapRef:
                name: minha-app-config

          # (B) Recursos (sempre definir)
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          # (C) Probes (padrão saudável)
          startupProbe:                     # só use se o boot for demorado
            httpGet:
              path: /healthz
              port: http
            failureThreshold: 30            # 30 tentativas
            periodSeconds: 2                # a cada 2s => ~60s para “subir”
          readinessProbe:                   # controla entrada no Service
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:                    # reinicia se travar
            httpGet:
              path: /live
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3

          # (D) Lifecycle hooks (shutdown elegante)
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]

          # (E) Segurança do container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          # (F) Volumes (se precisar)
          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}
```

### O que cada opção “faz de verdade”

- **replicas**: número de pods desejados (alta disponibilidade básica).
    
- **strategy RollingUpdate**: atualiza sem derrubar tudo.
    
    - `maxUnavailable`: quantos pods podem ficar fora ao atualizar.
        
    - `maxSurge`: quantos pods extras podem nascer durante a atualização.
        
- **requests/limits**:
    
    - `requests`: “reserva” para o scheduler decidir onde cabe.
        
    - `limits`: teto de consumo (CPU “throttle”; memória pode OOMKill).
        
- **readinessProbe**: pod só recebe tráfego quando estiver pronto.
    
- **livenessProbe**: se falhar, Kubernetes reinicia o container.
    
- **startupProbe**: protege apps que demoram a subir (evita liveness matar cedo).
    
- **terminationGracePeriodSeconds + preStop**: dá tempo para drenar conexões antes de matar.
    
- **securityContext**: endurece o runtime (boa prática pra DevSecOps).
    
- **env / envFrom**: use `envFrom` pra ConfigMap grande; use `secretKeyRef` pra valores sensíveis.
    
- **revisionHistoryLimit**: evita acumular ReplicaSet velho demais.
    

---

## Checklist rápido antes do `kubectl apply`

### Identidade e seleção

- [ ]  `namespace` correto?
    
- [ ]  `labels` e `selector.matchLabels` batendo (mesmo `app:`)?
    
- [ ]  Nome do container e `image` com tag fixa (evitar `latest`)?
    

### Escalabilidade e rollout

- [ ]  `replicas` >= 2 (se precisar HA)?
    
- [ ]  `strategy` definido e coerente com o tráfego (ex.: `maxUnavailable: 0`)?
    
- [ ]  `revisionHistoryLimit` ajustado?
    

### Probes (quase sempre)

- [ ]  `readinessProbe` existe e aponta para endpoint leve (`/ready`)?
    
- [ ]  `livenessProbe` existe e não é agressiva demais?
    
- [ ]  `startupProbe` só se a app demora (migrações, warmup, etc)?
    

### Recursos e estabilidade

- [ ]  `requests` e `limits` definidos (CPU e memória)?
    
- [ ]  Porta do container nomeada (`name: http`) pra usar em probes e Service?
    
- [ ]  `terminationGracePeriodSeconds` definido (ex.: 30)?
    
- [ ]  `preStop` se precisa drenar?
    

### Config e segredos

- [ ]  `ConfigMap` e `Secret` existem e têm as keys corretas?
    
- [ ]  Variáveis sensíveis só via Secret?
    
- [ ]  `imagePullSecrets` se registry privado?
    

### Segurança

- [ ]  `runAsNonRoot`, `allowPrivilegeEscalation: false`, `drop ALL`?
    
- [ ]  `readOnlyRootFilesystem` (se app suportar)?
    
- [ ]  volumes só quando necessário?
    

### Observabilidade

- [ ]  logs no stdout/stderr?
    
- [ ]  anotação/ServiceMonitor para Prometheus se aplicável?
    
- [ ]  endpoints de health expostos e consistentes?
    

---

## Cheatsheet Services (ClusterIP, NodePort, LoadBalancer) + Ingress

### 1) ClusterIP (padrão: tráfego interno no cluster)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-app
spec:
  type: ClusterIP
  selector:
    app: minha-app
  ports:
    - name: http
      port: 80
      targetPort: http
```

### 2) NodePort (abre uma porta em cada node)

Bom pra laboratório (k3d/kind/minikube), rápido pra testar.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-app
spec:
  type: NodePort
  selector:
    app: minha-app
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30080   # opcional (senão o cluster escolhe)
```

### 3) LoadBalancer (cloud; em lab pode ser via add-on / metallb / túnel)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minha-app
spec:
  type: LoadBalancer
  selector:
    app: minha-app
  ports:
    - name: http
      port: 80
      targetPort: http
```

### 4) Ingress (rota por host/path)

Precisa de um Ingress Controller (Nginx, Traefik etc.).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minha-app
spec:
  rules:
    - host: minha-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: minha-app
                port:
                  number: 80
```

**Quando usar cada tipo**

- **ClusterIP**: comunicação interna (backend, APIs internas).
    
- **NodePort**: expor rápido pra fora em ambiente local.
    
- **LoadBalancer**: expor em cloud ou com solução de LB no cluster.
    
- **Ingress**: expor HTTP/HTTPS com múltiplas rotas/hosts e TLS.
    

---

## Passos de criação de cluster

### k3d (K3s no Docker)

**Criar cluster simples**

```bash
k3d cluster create lab
kubectl cluster-info
```

**Com porta publicada (ótimo pra NodePort/Ingress)**

```bash
k3d cluster create lab \
  -p "8080:80@loadbalancer" \
  -p "8443:443@loadbalancer"
```

**Ver clusters / apagar**

```bash
k3d cluster list
k3d cluster delete lab
```

---

### kind (Kubernetes “puro” em containers)

**Criar cluster default**

```bash
kind create cluster --name lab
kubectl cluster-info
```

**Criar com config (expondo NodePort de forma previsível)**  
Crie `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP
```

Depois:

```bash
kind create cluster --name lab --config kind-config.yaml
```

**Apagar**

```bash
kind delete cluster --name lab
```

---

### minikube (VM/Container local, bem “bateria inclusa”)

**Iniciar**

```bash
minikube start
kubectl cluster-info
```

**Habilitar addons comuns**

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

**Acessar um Service facilmente**

```bash
minikube service minha-app --url
```

**Apagar**

```bash
minikube delete
```

---

## Mini-guia de decisão (bem rápido)

- Quero lab rápido e prático com “cara de cluster real”: **k3d**
    
- Quero simular Kubernetes em containers de forma bem padronizada: **kind**
    
- Quero “tudo pronto” com addons e UX boa pra aprender: **minikube**
    


#DevOps #Estudos #Tecnologia #Recorrente #Dicas #Comandos #Linux