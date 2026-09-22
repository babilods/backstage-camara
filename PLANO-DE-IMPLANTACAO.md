# Plano de Implantação — Backstage no Kubernetes local (WSL2 + minikube)

Guia autossuficiente para refazer este projeto do zero, em qualquer máquina Windows
com WSL2 disponível. Segue exatamente a ordem que funcionou, já incluindo as
correções para os problemas que enfrentamos na primeira vez.

**Pré-requisito:** Windows 10 (build recente) ou Windows 11, com permissão de
administrador na máquina (necessário só na Etapa 1, para instalar o WSL).

---

## Etapa 1 — Instalar o WSL2 e uma distro Linux

Abra o PowerShell **como administrador** e rode:

```powershell
wsl --install -d Ubuntu-24.04
```

Isso baixa e instala o WSL2 (se ainda não estiver instalado) junto com o Ubuntu 24.04.
Na primeira vez, ele vai pedir para você criar um usuário e senha Linux — pode ser
qualquer usuário, é só para uso local.

Depois de instalado, confirme que está funcionando:

```powershell
wsl -d Ubuntu-24.04 -- echo "ok"
```

Se aparecer "ok", está tudo certo.

> **Não instale o Docker Desktop.** Vamos usar o Docker Engine nativo dentro do
> WSL, que é mais leve e não trava com disco cheio (foi exatamente esse o motivo
> de abandonarmos o Docker Desktop da primeira vez).

---

## Etapa 2 — Instalar o Docker Engine dentro do WSL

A partir daqui, todos os comandos rodam **dentro do WSL** (abra um terminal Ubuntu,
ou use `wsl -d Ubuntu-24.04` a partir do PowerShell).

```bash
# Instalar pacotes básicos e adicionar o repositório oficial do Docker
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar o Docker Engine
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Deixar o Docker rodar sem precisar de sudo toda vez
sudo usermod -aG docker $USER
```

### Ativar o systemd (necessário para o Docker rodar como serviço)

Edite (ou crie) o arquivo `/etc/wsl.conf`:

```bash
sudo bash -c 'printf "[boot]\nsystemd=true\n" > /etc/wsl.conf'
```

Depois, **feche todo o WSL e reinicie** (rode isso no PowerShell, fora do WSL):

```powershell
wsl --shutdown
```

Abra o WSL de novo e confirme que o Docker está de pé:

```bash
docker run --rm hello-world
```

Se aparecer a mensagem "Hello from Docker!", deu certo.

> ⚠️ **Importante:** depois de qualquer `wsl --shutdown`, o cluster minikube (se
> já estiver rodando) é derrubado junto. É normal — só precisa dar `minikube start`
> de novo depois.

---

## Etapa 3 — Instalar build-essential ANTES de instalar o Node.js

Este passo é importante fazer **nesta ordem** — instalar o compilador C/C++ antes
de instalar as dependências do projeto evita ter que reinstalar tudo depois
(alguns pacotes do Backstage precisam compilar código nativo).

```bash
sudo apt-get install -y build-essential python3-dev
```

---

## Etapa 4 — Instalar Node.js e Yarn

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo bash -
sudo apt-get install -y nodejs
npm install -g yarn
```

Confirme as versões:
```bash
node --version   # deve mostrar v24.x
yarn --version
```

---

## Etapa 5 — Instalar minikube e kubectl

```bash
# minikube
curl -Lo /tmp/minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install /tmp/minikube /usr/local/bin/minikube

# kubectl (ajuste a versão se necessário, ou consulte https://dl.k8s.io/release/stable.txt)
curl -Lo /tmp/kubectl https://dl.k8s.io/release/v1.37.0/bin/linux/amd64/kubectl
sudo install /tmp/kubectl /usr/local/bin/kubectl
```

---

## Etapa 6 — Subir o cluster minikube

```bash
minikube start --driver=docker --memory=2200mb --cpus=2
```

> As flags `--memory` e `--cpus` explícitas evitam um aviso de "não sobra memória
> para o sistema" que aparece com a detecção automática em máquinas mais simples.

Confirme:
```bash
minikube status
kubectl get nodes
```
Deve aparecer `Running`/`Ready` em tudo.

---

## Etapa 7 — Criar o projeto Backstage

```bash
npx @backstage/create-app@latest --path backstage-app
```

> Rode esse comando dentro do **Bash do WSL**, não em outro terminal — em alguns
> ambientes, o pipe de entrada de outros shells injeta caracteres invisíveis que
> quebram a validação do prompt interativo ("App name must be lowercase...").

Depois de criado, entre na pasta e rode localmente para validar antes de
containerizar:
```bash
cd backstage-app
yarn install
yarn start
```
Acesse `http://localhost:3000` no navegador do Windows (o WSL2 encaminha essa
porta automaticamente) e confirme que a tela do Backstage aparece. Pare com
`Ctrl+C` quando confirmar.

---

## Etapa 8 — Conferir o banco de dados de produção (não precisa editar nada)

O `create-app` já gera automaticamente o arquivo `app-config.production.yaml`
(na raiz do `backstage-app`) com a seção de banco de dados pronta, lendo de
variáveis de ambiente — exatamente o que precisamos para o Kubernetes. **Não é
necessário criar nem editar esse arquivo**, só confirmar que ele já contém isto:

```yaml
backend:
  baseUrl: http://localhost:7007
  listen: ':7007'
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
```

Se por acaso alguma versão futura do `create-app` gerar diferente, é só ajustar
manualmente para ficar assim.

---

## Etapa 9 — Criar os manifests do Kubernetes

Crie uma pasta `k8s/` com os seguintes arquivos (na raiz do projeto, fora do
`backstage-app/`):

**`k8s/namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: backstage
```

**`k8s/postgres-secret.yaml`** (troque a senha por uma sua)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: backstage
type: Opaque
stringData:
  POSTGRES_USER: backstage
  POSTGRES_PASSWORD: troque-esta-senha
  POSTGRES_DB: backstage
```

**`k8s/backstage-config.yaml`**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backstage-config
  namespace: backstage
data:
  POSTGRES_HOST: postgres
  POSTGRES_PORT: "5432"
```

**`k8s/postgres-pvc.yaml`**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: backstage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

**`k8s/postgres-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: backstage
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgres-credentials
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "backstage"]
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "backstage"]
            initialDelaySeconds: 15
            periodSeconds: 10
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-data
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: backstage
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

**`k8s/backstage-deployment.yaml`**

> Repare nos tempos de `initialDelaySeconds` abaixo — já vêm ajustados (mais
> tolerantes que o padrão) porque o Backstage demora para inicializar todos os
> seus módulos internos, e um tempo curto demais faz o Kubernetes matar o pod
> achando que travou, no meio da inicialização.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: backstage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      containers:
        - name: backstage
          image: backstage:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 7007
          envFrom:
            - configMapRef:
                name: backstage-config
            - secretRef:
                name: postgres-credentials
          readinessProbe:
            httpGet:
              path: /healthcheck
              port: 7007
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 6
          livenessProbe:
            httpGet:
              path: /healthcheck
              port: 7007
            initialDelaySeconds: 60
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backstage
  namespace: backstage
spec:
  type: NodePort
  selector:
    app: backstage
  ports:
    - port: 7007
      targetPort: 7007
```

---

## Etapa 10 — Gerar os arquivos que o Dockerfile espera encontrar

> ⚠️ **Passo que passa despercebido, mas é obrigatório.** O Dockerfile do
> backend (`packages/backend/Dockerfile`) não builda o código TypeScript —
> ele só empacota arquivos **já compilados**, que precisam existir antes. Sem
> este passo, o `docker build` da Etapa 11 falha procurando por
> `packages/backend/dist/skeleton.tar.gz`.

Dentro da pasta `backstage-app/`:

```bash
yarn install
yarn tsc
yarn build:backend
```

Confirme que os arquivos foram gerados:
```bash
ls packages/backend/dist/
# deve mostrar: bundle.tar.gz  skeleton.tar.gz
```

---

## Etapa 11 — Buildar a imagem Docker

Ainda dentro da pasta `backstage-app/`:

```bash
docker build -t backstage:latest -f packages/backend/Dockerfile .
```

Isso pode demorar alguns minutos na primeira vez (baixa a imagem base e instala
as dependências de produção). Confirme que a imagem foi criada:

```bash
docker images | grep backstage
```

---

## Etapa 12 — Carregar a imagem no minikube

> ⚠️ **Atenção — este é o passo que mais deu problema da primeira vez.** O
> comando oficial `minikube image load` trava indefinidamente em clusters que
> usam containerd como runtime (que é o padrão atual do minikube). **Não perca
> tempo tentando esse comando** — vá direto para o método abaixo, que sempre
> funcionou:

```bash
# 1. Salvar a imagem como arquivo
docker save backstage:latest -o /tmp/backstage.tar

# 2. Copiar o arquivo para dentro do node do minikube (NÃO use "docker cp" — ele
#    falha silenciosamente nesse cenário; use o redirecionamento via pipe abaixo)
cat /tmp/backstage.tar | docker exec -i minikube sh -c "cat > /tmp/backstage.tar"

# 3. Importar a imagem diretamente no containerd do node
docker exec minikube ctr --namespace k8s.io images import /tmp/backstage.tar

# 4. Confirmar que a imagem está disponível
docker exec minikube ctr --namespace k8s.io images list | grep backstage
```

---

## Etapa 13 — Aplicar os manifests no cluster

Na raiz do projeto (onde está a pasta `k8s/`):

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres-secret.yaml -f k8s/backstage-config.yaml -f k8s/postgres-pvc.yaml -f k8s/postgres-deployment.yaml
```

Aguarde o Postgres ficar pronto antes de continuar:
```bash
kubectl get pods -n backstage --watch
```
Espere até `postgres-...` mostrar `1/1 Running` (pode reiniciar uma vez sozinho,
é normal). Aperte `Ctrl+C` para sair do modo `--watch`.

Agora aplique o Backstage:
```bash
kubectl apply -f k8s/backstage-deployment.yaml
```

Aguarde de novo até `backstage-...` mostrar `1/1 Running`.

---

## Etapa 14 — Testar

```bash
# Testar de dentro do cluster
kubectl port-forward -n backstage svc/backstage 7007:7007
```
Em outra aba/terminal, ou no navegador do Windows, acesse:
```
http://localhost:7007
```
Deve aparecer a tela do Backstage.

Pra ver visualmente (painel do Kubernetes):
```bash
minikube dashboard
```

Pra acessar o banco de dados diretamente:
```bash
kubectl exec -it -n backstage deploy/postgres -- psql -U backstage -d backstage
```

---

## Checklist rápido — o que NÃO fazer (aprendido com erros anteriores)

- ❌ Não instale o Docker Desktop — use Docker Engine nativo dentro do WSL.
- ❌ Não rode `sudo <comando>` sem a flag `-n` em scripts automatizados — se o
  NOPASSWD não for reconhecido, ele pode ficar esperando uma senha para sempre.
- ❌ Não confie em `minikube image load` — use o método manual da Etapa 12.
- ❌ Não confie em `docker cp` para copiar arquivos grandes para dentro do node
  do minikube — use o redirecionamento via pipe (`cat arquivo | docker exec -i ...`).
- ❌ Não deixe os tempos padrão dos probes do Backstage (10s/20s) — são curtos
  demais; use os valores já ajustados na Etapa 9.
- ❌ Evite rodar `yarn install` pesado ao mesmo tempo que o minikube está de pé
  em máquinas com poucos núcleos — rode `minikube stop` antes, se notar tudo
  muito lento, e `minikube start` depois.

---

## O que isso NÃO inclui (pendências antes de produção real)

Este plano cobre até um **ambiente de teste local completo e funcional**. Antes
de produção de verdade, ainda falta: segredos gerenciados de verdade (não em
texto puro), HTTPS, endereço público real (não `localhost`), um cluster
Kubernetes real da organização, um registry de imagens real, e uma esteira
(pipeline de CI/CD) para automatizar as Etapas 11 a 13.
