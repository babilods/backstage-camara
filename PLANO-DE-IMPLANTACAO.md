# Plano de Implantação — Backstage no Kubernetes local (WSL2 + minikube)

Guia autossuficiente para reproduzir este projeto em qualquer máquina Windows
com WSL2 disponível, a partir do código deste repositório. Segue exatamente a
ordem que funcionou, já incluindo as correções para os problemas que
enfrentamos nas instalações anteriores.

**Pré-requisito:** Windows 10 (build recente) ou Windows 11, com permissão de
administrador na máquina (necessário só na Etapa 1, para instalar o WSL).

**Validado em:** Ubuntu 24.04 e Ubuntu 26.04 (WSL2), Docker Engine 29.8,
Node 24.21, minikube 1.39, kubectl 1.37.

---

## Etapa 1 — Instalar o WSL2 e uma distro Linux

Abra o PowerShell **como administrador** e rode:

```powershell
wsl --install -d Ubuntu-24.04
```

Isso baixa e instala o WSL2 (se ainda não estiver instalado) junto com o Ubuntu 24.04.
Na primeira vez, ele vai pedir para você criar um usuário e senha Linux — pode ser
qualquer usuário, é só para uso local.

> Se a máquina já tiver uma distro chamada só `Ubuntu` (versão 24.04 ou mais
> nova, ex: 26.04), ela também serve — o processo inteiro foi validado no 26.04.
> Nesse caso, troque `Ubuntu-24.04` por `Ubuntu` nos comandos `wsl -d` deste guia.
> Para ver o que já está instalado: `wsl -l -v`.

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
# Se já houve uma tentativa anterior de instalação, pode ter sobrado um
# docker.list sem a chave GPG — ele faz o primeiro apt-get update falhar com
# "NO_PUBKEY 7EA0A9C3F273FCD8". Removê-lo é seguro: é recriado logo abaixo.
sudo rm -f /etc/apt/sources.list.d/docker.list

# Instalar pacotes básicos e adicionar o repositório oficial do Docker
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar o Docker Engine
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Deixar o Docker rodar sem precisar de sudo toda vez
sudo usermod -aG docker $USER
```

### Ativar o systemd (necessário para o Docker rodar como serviço)

Confira se já está ativo (nas distros recentes do WSL, normalmente já vem):

```bash
cat /etc/wsl.conf
```

Se não aparecer `systemd=true` na seção `[boot]`, ative:

```bash
sudo bash -c 'printf "[boot]\nsystemd=true\n" >> /etc/wsl.conf'
```

Depois, **feche todo o WSL e reinicie** (rode isso no PowerShell, fora do WSL).
Isso é necessário mesmo que o systemd já estivesse ativo, para o grupo `docker`
passar a valer para o seu usuário:

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
sudo npm install -g yarn   # o sudo é necessário: o Node da NodeSource instala em /usr
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
curl -fLo /tmp/minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install /tmp/minikube /usr/local/bin/minikube

# kubectl (versão estável atual)
curl -fLo /tmp/kubectl "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install /tmp/kubectl /usr/local/bin/kubectl
```

Confirme:
```bash
minikube version
kubectl version --client
```

---

## Etapa 6 — Clonar o repositório e instalar as dependências

O projeto Backstage já está pronto neste repositório — **não é preciso rodar
`create-app`** (veja o Apêndice se quiser criar um projeto novo do zero).

```bash
cd ~
git clone https://github.com/babilods/backstage-camara.git
cd backstage-camara/backstage-app
yarn install
```

O `yarn install` leva alguns minutos e termina com "Done with warnings" — os
avisos são normais.

**Opcional** — validar localmente antes de containerizar:
```bash
yarn start
```
Acesse `http://localhost:3000` no navegador do Windows (o WSL2 encaminha essa
porta automaticamente) e confirme que a tela do Backstage aparece. Pare com
`Ctrl+C` quando confirmar. (Pode pular: o teste da Etapa 14 valida a mesma coisa.)

> Clone o repositório **dentro do sistema de arquivos do Linux** (`~/...`), não
> em `/mnt/c/...` — o `yarn install` e o build ficam muitas vezes mais lentos
> sobre o disco do Windows.

---

## Etapa 7 — Conferir o banco de dados de produção (não precisa editar nada)

O arquivo `backstage-app/app-config.production.yaml` já contém a seção de banco
de dados lendo de variáveis de ambiente — exatamente o que precisamos para o
Kubernetes. Só confirme que ele contém isto:

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

E também a seção de autenticação:

```yaml
auth:
  providers:
    guest:
      dangerouslyAllowOutsideDevelopment: true
```

> ⚠️ **Sem essa opção, a interface abre mas dá erro 401 em tudo.** Dentro do
> cluster o Backstage roda em modo produção, e nesse modo o login de convidado
> é recusado (`/api/auth/guest/refresh` responde 403 com "The guest provider
> cannot be used outside of a development environment"). Sem login, todas as
> chamadas da página (catálogo, notificações, permissões) voltam 401.
>
> Essa opção deixa **qualquer pessoa que acesse o endereço entrar como
> convidado, sem senha** — aceitável só enquanto o Backstage roda em
> `localhost`. Antes de produção, troque por um provedor de login real
> (GitHub, Microsoft etc.).
>
> Atenção à indentação: `guest:` fica 4 espaços para dentro e a linha de baixo
> 6 — com a indentação errada o YAML fica inválido e o Backstage não sobe.

---

## Etapa 8 — Criar o Secret do Postgres

Os manifests do Kubernetes já estão prontos na pasta `k8s/` (na raiz do
repositório) e **devem ser aplicados como estão** — já vêm com:

- tempos de readiness/liveness probe do Backstage mais tolerantes que o padrão
  (o Backstage demora para inicializar todos os módulos internos, e um tempo
  curto demais faz o Kubernetes matar o pod no meio da inicialização);
- limites de memória no Backstage (`300Mi`–`600Mi`, com `NODE_OPTIONS=--max-old-space-size=384`)
  e no Postgres (`150Mi`–`300Mi`).

A única exceção é o `k8s/postgres-secret.yaml`, que **não está no repositório**
de propósito (contém a senha real e está no `.gitignore`). Crie a sua cópia a
partir do modelo, já com uma senha aleatória:

```bash
cd ~/backstage-camara
PW=$(head -c 24 /dev/urandom | base64 | tr -dc 'A-Za-z0-9' | head -c 24)
sed "s/troque-esta-senha/$PW/" k8s/postgres-secret.example.yaml > k8s/postgres-secret.yaml
chmod 600 k8s/postgres-secret.yaml
git status --short   # o postgres-secret.yaml NÃO deve aparecer aqui
```

> A senha não precisa ser igual à de outra máquina — cada cluster começa com um
> banco vazio. Só importa se você for migrar dados de um banco existente.

---

## Etapa 9 — Gerar os arquivos que o Dockerfile espera encontrar

> ⚠️ **Passo que passa despercebido, mas é obrigatório.** O Dockerfile do
> backend (`packages/backend/Dockerfile`) não builda o código TypeScript —
> ele só empacota arquivos **já compilados**, que precisam existir antes. Sem
> este passo, o `docker build` da Etapa 10 falha procurando por
> `packages/backend/dist/skeleton.tar.gz`.

Dentro da pasta `backstage-app/`:

```bash
yarn tsc
yarn build:backend
```

Confirme que os arquivos foram gerados:
```bash
ls packages/backend/dist/
# deve mostrar: bundle.tar.gz  skeleton.tar.gz
```

---

## Etapa 10 — Buildar a imagem Docker

Ainda dentro da pasta `backstage-app/`:

```bash
docker build -t backstage:latest -f packages/backend/Dockerfile .
```

Isso pode demorar alguns minutos na primeira vez (baixa a imagem base e instala
as dependências de produção). Confirme que a imagem foi criada:

```bash
docker images backstage
```

> As Etapas 6, 9 e 10 vêm **antes** de subir o minikube de propósito: o build
> não depende do cluster, e assim o `yarn install`/`docker build` (pesados) não
> disputam memória com ele.

---

## Etapa 11 — Subir o cluster minikube

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
Deve aparecer `Running`/`Ready` em tudo (logo após o start o nó pode aparecer
`NotReady` por alguns segundos — é normal).

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
kubectl wait -n backstage --for=condition=available deploy/postgres --timeout=300s
kubectl get pods -n backstage
```
`postgres-...` deve mostrar `1/1 Running` (pode reiniciar uma vez sozinho, é normal).

Agora aplique o Backstage:
```bash
kubectl apply -f k8s/backstage-deployment.yaml
kubectl wait -n backstage --for=condition=available deploy/backstage --timeout=400s
kubectl get pods -n backstage
```

Aguarde até `backstage-...` mostrar `1/1 Running`.

> O `k8s/ingress.yaml` é opcional (pensado para produção) e exige
> `minikube addons enable ingress` — não é necessário para o teste local.

---

## Etapa 14 — Testar

```bash
kubectl port-forward -n backstage svc/backstage 7007:7007
```
Deixe esse terminal aberto e, no navegador do Windows, acesse:
```
http://localhost:7007
```
Deve aparecer a tela do Backstage **com o catálogo listando os exemplos**
(API, Component, System etc.). Só a página abrir não basta — se o catálogo
aparecer vazio ou com erro, veja se o login de convidado está funcionando:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -H 'X-Requested-With: XMLHttpRequest' localhost:7007/api/auth/guest/refresh
# deve mostrar 200; se mostrar 403, falta a configuração de auth da Etapa 7
```

Pra ver visualmente (painel do Kubernetes):
```bash
minikube dashboard
```

Pra acessar o banco de dados diretamente:
```bash
kubectl exec -it -n backstage deploy/postgres -- psql -U backstage -d backstage
```

---

## Depois de reiniciar o computador (ou um `wsl --shutdown`)

O cluster e o port-forward caem junto, mas a imagem, os manifests aplicados e
os dados do Postgres continuam lá. Para voltar:

```bash
minikube start
kubectl get pods -n backstage   # aguarde tudo 1/1 Running
kubectl port-forward -n backstage svc/backstage 7007:7007
```

Se você mudar o código ou a configuração (`app-config*.yaml`) do Backstage,
refaça as Etapas 9, 10 e 12 e rode
`kubectl rollout restart -n backstage deploy/backstage`. (Se a mudança foi só
em `app-config*.yaml`, a Etapa 9 pode ser pulada — o Dockerfile copia esses
arquivos direto.)

> ⚠️ Depois de um `rollout restart`, o `kubectl port-forward` que estava aberto
> continua ocupando a porta 7007, mas apontando para o pod antigo, que já não
> existe. Pare-o com `Ctrl+C` (ou `pkill -f "kubectl port-forward"`) e rode o
> port-forward de novo.

---

## Checklist rápido — o que NÃO fazer (aprendido com erros anteriores)

- ❌ Não instale o Docker Desktop — use Docker Engine nativo dentro do WSL.
- ❌ Não rode `sudo <comando>` sem a flag `-n` em scripts automatizados — se o
  NOPASSWD não for reconhecido, ele pode ficar esperando uma senha para sempre.
- ❌ Não confie em `minikube image load` — use o método manual da Etapa 12.
- ❌ Não confie em `docker cp` para copiar arquivos grandes para dentro do node
  do minikube — use o redirecionamento via pipe (`cat arquivo | docker exec -i ...`).
- ❌ Não reduza os tempos dos probes do Backstage para o padrão (10s/20s) — são
  curtos demais; aplique `k8s/backstage-deployment.yaml` como está.
- ❌ Não versione o `k8s/postgres-secret.yaml` — ele contém a senha real.
- ❌ Não considere o teste concluído só porque a página abriu — confira se o
  catálogo carrega (o erro 401 do login de convidado só aparece aí).
- ❌ Não reaproveite um port-forward antigo depois de reiniciar o pod do
  Backstage — reinicie o port-forward também.
- ❌ Evite rodar `yarn install` pesado ao mesmo tempo que o minikube está de pé
  em máquinas com poucos núcleos — rode `minikube stop` antes, se notar tudo
  muito lento, e `minikube start` depois.

---

## O que isso NÃO inclui (pendências antes de produção real)

Este plano cobre até um **ambiente de teste local completo e funcional**. Antes
de produção de verdade, ainda falta: segredos gerenciados de verdade (não em
texto puro), HTTPS, endereço público real (não `localhost`), um cluster
Kubernetes real da organização, um registry de imagens real, e uma esteira
(pipeline de CI/CD) para automatizar as Etapas 9 a 13.

---

## Apêndice — Criar um projeto Backstage novo do zero

Só necessário se você quiser começar um projeto novo em vez de usar o deste
repositório (foi assim que o `backstage-app/` foi criado originalmente):

```bash
npx @backstage/create-app@latest --path backstage-app
```

> Rode esse comando dentro do **Bash do WSL**, não em outro terminal — em alguns
> ambientes, o pipe de entrada de outros shells injeta caracteres invisíveis que
> quebram a validação do prompt interativo ("App name must be lowercase...").

O `create-app` já gera o `app-config.production.yaml` com a seção de banco da
Etapa 7. Se alguma versão futura gerar diferente, ajuste manualmente.
