# Desafio: Fundamentos de Kubernetes na Prática

## Ferramenta de cluster local

Rancher Desktop (containerd + k3s), integrado via WSL2 (Ubuntu).

## Ambiente

- Windows 11 Pro
- WSL2 + Ubuntu
- kubectl v1.36.4 (via Rancher Desktop)

## Como aplicar

Antes de aplicar, copiar o arquivo de exemplo do Secret e preencher com valores reais:

```bash
cp manifests/01-postgres-secret.yaml.example manifests/01-postgres-secret.yaml
```

Editar `manifests/01-postgres-secret.yaml` com usuário, senha e URI reais.

Aplicar os manifests:
```bash
kubectl apply -f manifests/
```

Criar a tabela no banco de dados:

```bash
kubectl exec -it -n desafio-k8s deploy/postgres -- psql -U desafio -d desafio_db
```

Dentro do prompt:
```sql
CREATE TABLE itens (
  id SERIAL PRIMARY KEY,
  nome TEXT NOT NULL
);
```

## Como testar

Port-forward da API:

```bash
kubectl port-forward -n desafio-k8s svc/pgrest 3000:3000
```

Inserir um dado:

```bash
curl -X POST http://localhost:3000/itens -H "Content-Type: application/json" -d '{"nome": "teste"}'
```

Consultar dados:

```bash
curl http://localhost:3000/itens

# Saída
[{"id":1,"nome":"teste"}]
```

### Persistência

Deletar o Pod do Postgres e confirmar que o dado sobrevive:

```bash
kubectl delete pod -n desafio-k8s -l app=postgres
```

Consultar de novo (mesmo dado, Pod novo):

```bash
curl http://localhost:3000/itens

# Saída
[{"id":1,"nome":"teste"}]
```

## Escalonamento automático (HPA)

Gerar carga na API:

```bash
kubectl run carga --image=busybox -n desafio-k8s -it --rm -- /bin/sh -c "while true; do wget -q -O- http://pgrest:3000/itens; done"
```

Observar o HPA escalar:

```bash
kubectl get hpa -n desafio-k8s -w

# Saída
NAME         REFERENCE           TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
pgrest-hpa   Deployment/pgrest   cpu: 4%/50%     2         5         2          29m
pgrest-hpa   Deployment/pgrest   cpu: 5%/50%     2         5         2          29m
pgrest-hpa   Deployment/pgrest   cpu: 73%/50%    2         5         2          29m
pgrest-hpa   Deployment/pgrest   cpu: 126%/50%   2         5         3          30m
pgrest-hpa   Deployment/pgrest   cpu: 122%/50%   2         5         5          30m
pgrest-hpa   Deployment/pgrest   cpu: 106%/50%   2         5         5          30m
pgrest-hpa   Deployment/pgrest   cpu: 59%/50%    2         5         5          30m
pgrest-hpa   Deployment/pgrest   cpu: 62%/50%    2         5         5          31m
```

Confirmar os Pods extras criados:

```bash
kubectl get pods -n desafio-k8s

# Saída
NAME                        READY   STATUS    RESTARTS   AGE
pgrest-6f84799976-7bv7v     1/1     Running   0          2m47s
pgrest-6f84799976-9ch7r     1/1     Running   0          36m
pgrest-6f84799976-bj6bk     1/1     Running   0          2m47s
pgrest-6f84799976-wggxh     1/1     Running   0          3m3s
pgrest-6f84799976-xh5mz     1/1     Running   0          36m
postgres-6d78c9b7fc-2gqw8   1/1     Running   0          39m
```

Depois de parar a carga, o HPA reduz de volta para o mínimo:

```bash
kubectl get pods -n desafio-k8s

# Saída
NAME                        READY   STATUS    RESTARTS   AGE
pgrest-6f84799976-9ch7r     1/1     Running   0          53m
pgrest-6f84799976-xh5mz     1/1     Running   0          53m
postgres-6d78c9b7fc-2gqw8   1/1     Running   0          56m
```

## Evidências

```bash
kubectl get all -n desafio-k8s

# Saída
NAME                            READY   STATUS    RESTARTS        AGE
pod/pgrest-6f84799976-9ch7r     1/1     Running   1 (2m22s ago)   76m
pod/pgrest-6f84799976-xh5mz     1/1     Running   1 (2m22s ago)   76m
pod/postgres-6d78c9b7fc-2gqw8   1/1     Running   1 (2m22s ago)   78m

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/pgrest     ClusterIP   10.43.117.92    <none>        3000/TCP   4h9m
service/postgres   ClusterIP   10.43.153.156   <none>        5432/TCP   4h9m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pgrest     2/2     2            2           4h9m
deployment.apps/postgres   1/1     1            1           4h9m

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/pgrest-6f84799976     2         2         2       76m
replicaset.apps/pgrest-6fcdb5bddd     0         0         0       82m
replicaset.apps/pgrest-9745b5d97      0         0         0       4h9m
replicaset.apps/postgres-6d78c9b7fc   1         1         1       4h9m

NAME                                             REFERENCE           TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/pgrest-hpa   Deployment/pgrest   cpu: 6%/50%   2         5         2          72m
```

## Limpeza

```bash
kubectl delete namespace desafio-k8s
```