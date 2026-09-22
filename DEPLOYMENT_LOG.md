# Backstage no Kubernetes (minikube) — Passo a passo

Documento de acompanhamento do projeto: integrar o Backstage rodando dentro de um cluster Kubernetes local (minikube). Vai sendo atualizado conforme as etapas avançam.

Referência oficial usada como base: https://backstage.io/docs/deployment/k8s

## Decisões tomadas

- Ambiente pensado já com produção em mente (Postgres com volume persistente, Secrets separados, Ingress incluído desde já), mas o acesso local no dia a dia será via `minikube service` (mais simples que Ingress).
- Catálogo do Backstage vai começar só com os exemplos estáticos gerados pelo `create-app`; integração com GitHub fica para uma etapa futura.
- Ordem de execução combinada: **minikube up → criar o Backstage → rodar localmente e validar → decidir o banco com a equipe → só então criar os manifests de banco/deploy no k8s.**

## Ambiente da máquina

| Ferramenta | Situação inicial | Situação atual |
|---|---|---|
| Node.js | Quebrado (`npm`/`npx` falhavam com `EPERM`) | OK — Node LTS `v24.19.0` |
| npm | Quebrado | OK — `11.17.0` |
| Yarn | Não instalado | OK — `1.22.22` (clássico, via `npm install -g yarn`) |
| Docker Desktop | Instalado, daemon rodando | **Desinstalado** (10/09) — substituído por Docker Engine `29.8.0` dentro do WSL2 (ver Decisão abaixo) |
| minikube | Instalado, cluster não iniciado | OK — recriado dentro do WSL, `v1.39.0`, node `Ready`, todos os pods de sistema `Running` |
| kubectl | Não instalado como binário separado | OK — `v1.37.0` instalado dentro da distro WSL |
| WSL2 | Só a distro interna do Docker Desktop | OK — distro própria `Ubuntu-24.04`, com Docker Engine + minikube + kubectl |

### Problema 1 — Node/npm com erro `EPERM`

**Sintoma:** qualquer comando `npm`/`npx` falhava com:
```
Error: EPERM: operation not permitted, lstat 'C:\Users\tec.mauro\AppData'
```

**Causa raiz:** o Node estava instalado via `nvm4w` (nvm for Windows) em `C:\nvm4w\nodejs`, mas a pasta `node_modules\npm` dentro dele era um **link simbólico** apontando para o perfil de outro usuário do domínio (`tec.mauro`), a quem a conta atual (`cldf\barbara.santos`) não tem permissão de acesso. Aparentemente essa instalação foi feita/compartilhada por outro usuário na mesma máquina.

**Correção aplicada:** em vez de mexer nas permissões do perfil de outro usuário (arriscado e fora do escopo), foi instalado o Node.js LTS oficial via `winget`:
```powershell
winget install --id OpenJS.NodeJS.LTS -e --accept-package-agreements --accept-source-agreements --silent
```
Isso instalou o Node em `C:\Program Files\nodejs`, independente do `nvm4w`.

**Pendência conhecida:** o `PATH` do sistema ainda tem `C:\nvm4w\nodejs` **antes** de `C:\Program Files\nodejs`, então todo comando `node`/`npm`/`npx`/`yarn` rodado nesta sessão precisa priorizar o caminho novo, por exemplo:
```powershell
$env:Path = "C:\Program Files\nodejs;" + $env:Path
```
Para uso fora do Claude Code (terminal normal do Windows), o ideal é reordenar o `PATH` do sistema (Variáveis de Ambiente) colocando `C:\Program Files\nodejs` antes de `C:\nvm4w\nodejs`, ou remover a entrada quebrada do `nvm4w`.

### Problema 2 — Yarn ausente

O `@backstage/create-app` exige Yarn como pré-requisito. Instalado via:
```powershell
npm install -g yarn
```

### Docker Desktop

Já estava instalado e funcionando (`docker --version` → `29.7.2`, daemon respondendo). Não precisou de nenhuma ação.

### kubectl

Não há um binário `kubectl` separado instalado. Em vez de instalar mais uma ferramenta, todos os comandos de Kubernetes são feitos através do wrapper embutido do minikube:
```
minikube kubectl -- <comando>
```

### Problema 3 — Disco cheio derruba o Docker Desktop (09/09 e 10/09)

**Sintoma:** durante o `docker build` da imagem do backend, o build falhou com erro de sistema de arquivos somente leitura (`EROFS`). Depois disso o daemon do Docker parou de responder por completo (`docker version` travava/dava timeout).

**Causa raiz:** o disco `C:` (só ~111 GB no total) ficou com **0,4 GB livres**. O Docker Desktop (via WSL2) não tolera bem ficar sem espaço: o disco de dados dele (`docker_data.vhdx`) ficou num estado inconsistente, e ao tentar subir de novo o log mostrava:
```
detected no file system. Formatting
mke2fs: /dev/sdf is apparently in use by the system; will not make a filesystem here!
```

**Correção aplicada:**
1. Liberado espaço no disco (limpeza manual feita pela usuária — Lixeira, temporários, etc. Uma limpeza mais profunda via DISM/Liberador de Espaço em Disco exige permissão de administrador, que a sessão do Claude Code não tem).
2. `wsl --shutdown` para forçar a liberação do disco de dados travado.
3. Docker Desktop reaberto — dessa vez subiu normalmente, sem repetir o erro de formatação.

**Recaída:** ao tentar o `docker build` de novo, o disco voltou a cair rapidamente (de ~6 GB para ~1,4 GB livres em poucos minutos) — o build continuou consumindo espaço no daemon mesmo depois do comando ser cancelado do lado do terminal. Foi necessário parar o Docker Desktop e o WSL de novo (`wsl --shutdown`) para estancar antes de zerar o disco outra vez.

**Decisão tomada (10/09): sair do Docker Desktop.** Combinado com a usuária substituir Docker Desktop + a forma atual do minikube por **Docker Engine rodando direto dentro de uma distro WSL2 (Ubuntu)**, mantendo o minikube (não abrindo mão do Kubernetes). Motivo: o Docker Desktop soma uma camada extra de processos, telemetria e duas distros WSL próprias (`docker-desktop` / `docker-desktop-data`) por cima do que já seria necessário só com WSL2 + Docker Engine — nesta máquina, com pouca margem de disco, esse peso extra foi o que directamente causou as duas quedas acima. Escopo combinado: mover **todo** o fluxo (Docker, minikube, kubectl e o próprio projeto Node/Yarn) para dentro do WSL, em vez de uma mistura Windows/WSL (mais frágil e pior suportada sem a ponte que o Docker Desktop fazia).

Passos da migração (concluída em 10/09):
1. `winget uninstall --id Docker.DockerDesktop` — desinstalado com sucesso, e isso já removeu as distros `docker-desktop`/`docker-desktop-data`, liberando o disco de volta para ~16 GB livres.
2. Instalada uma distro Ubuntu própria via `wsl --install -d Ubuntu-24.04 --no-launch` (o `--no-launch` evita o assistente interativo de criação de usuário, que travaria uma sessão não-interativa como esta).
3. Usuário Linux `barbara.santos` criado via `useradd` (não-interativo), com `sudo` sem senha liberado **diretamente em `/etc/sudoers`** (uma entrada em `/etc/sudoers.d/` foi tentada primeiro mas, por algum motivo não identificado, não estava sendo respeitada por `sudo -n` — editar o arquivo principal resolveu). Usuário definido como padrão da distro via `/etc/wsl.conf` (`[user] default=...`).
4. **`systemd` ativado** na distro via `/etc/wsl.conf` (`[boot] systemd=true`) — necessário para gerenciar o `dockerd` como serviço (`systemctl enable --now docker`). Sem isso, `systemctl` falha com "System has not been booted with systemd as init system".
5. Docker Engine instalado via repositório oficial (apt): `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`. Usuário adicionado ao grupo `docker` (`usermod -aG docker`) para rodar `docker` sem `sudo`. Testado com `docker run hello-world` — funcionando.
6. minikube (`v1.39.0`) e kubectl (`v1.37.0`) instalados manualmente (binários baixados via `curl`, não pacotes apt) dentro da distro.
7. Cluster minikube recriado do zero com `minikube start --driver=docker --memory=2200mb --cpus=2` (memória/CPU explícitas para evitar um aviso de "não sobra memória para o sistema" que aparecia com a detecção automática). Resultado: node `Ready`, todos os pods de `kube-system` (`etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kube-proxy`, `coredns`, `kindnet`, `storage-provisioner`) `1/1 Running`.
8. Projeto copiado do Windows (`/mnt/c/Users/barbara.santos/backstage`) para dentro do filesystem nativo do WSL (`~/backstage`) via `rsync`, **excluindo `node_modules` e `.git`** (módulos nativos compilados para Windows não funcionam no Linux; melhor gerar de novo).
9. Node.js `v24.21.0` e Yarn `1.22.22`/`4.13.0` (o projeto usa Yarn 4 via `packageManager`/`.yarnrc.yml`) instalados nativamente dentro da distro (repositório oficial NodeSource + `npm install -g yarn`).
10. `build-essential` (`gcc`, `g++`, `make`) instalado — necessário para compilar dependências nativas do projeto (`better-sqlite3`, `tree-sitter`, `@swc/core`, `esbuild`, `keytar`, `cpu-features`, etc.). **Sem isso, `yarn install` completa mas com pacotes nativos falhando silenciosamente** (`couldn't be built successfully`) — não é erro fatal do install como um todo, mas quebra funcionalidades em runtime.
11. `yarn install` rodado dentro do WSL: primeira tentativa levou **38min41s** (sem `build-essential` ainda, então `cpu-features` falhou ao compilar; o resto funcionou). Segunda tentativa, já com `build-essential` instalado, levou só **3min35s** (cache do fetch reaproveitado) e terminou **sem nenhuma falha de build**.

**Pegadinhas descobertas durante a migração (documentando para não repetir):**
- **`sudo` sem `-n` trava para sempre**: rodando comandos via `wsl.exe -d <distro> -- sudo <cmd>` sem a flag `-n`, se por algum motivo o NOPASSWD não é reconhecido, o `sudo` tenta pedir senha interativamente e **trava indefinidamente** em vez de falhar rápido (não há terminal para prompt nesta sessão). Sempre usar `sudo -n`.
- **Processos em `&`/`disown` dentro do WSL não sobrevivem**: rodar `comando &` dentro de um `wsl.exe -d <distro> -- bash -c "..."` não mantém o processo vivo depois que aquele `wsl.exe` retorna — a sessão do WSL é encerrada e derruba os processos filhos junto, mesmo com `nohup`/`disown`. Isso derrubou o cluster minikube uma vez pouco depois de ele subir (container saiu com exit 130/SIGINT).
- **Correção aplicada**: manter uma sessão `wsl.exe -d <distro> -- sleep 3600` rodando em segundo plano (via backgrounding do próprio Claude Code, não do WSL) durante todo o trabalho, pra WSL não derrubar a distro entre um comando e outro.
- Downloads via `apt`/`curl` de dentro do WSL funcionam normalmente (rede OK, sem bloqueio de proxy corporativo).
- **Máquina com poucos núcleos: minikube inteiro (etcd, apiserver, kubelet, controller-manager, scheduler, coredns, kube-proxy) rodando ao mesmo tempo que um `yarn install` pesado disputa CPU e deixa tudo muito lento** (chegou a quase parar por completo). Rodar `minikube stop` antes de operações pesadas de build/install e religar depois ajuda bastante.
- **Instale `build-essential` (`gcc`/`g++`/`make`) antes do primeiro `yarn install`**, não depois — isso evita ter que rodar o install duas vezes por causa de módulos nativos que falham silenciosamente na primeira tentativa.

## Como usar o Docker/minikube agora (dentro do WSL)

Todos os comandos abaixo devem ser rodados **dentro da distro `Ubuntu-24.04`**, não no PowerShell/Windows:
```bash
wsl -d Ubuntu-24.04
docker ps
minikube status
kubectl get pods -A
```

## Etapas executadas

### 1. Subir o cluster minikube

```powershell
minikube start --driver=docker
```
Resultado: cluster `Running` (control-plane, kubelet e apiserver todos `Running`, kubeconfig configurado). Confirmado com:
```powershell
minikube status
```

### 2. Criar a aplicação Backstage

```bash
npx @backstage/create-app@latest --path backstage-app
```
> Nota técnica: rodar esse comando via **Bash**, não via PowerShell — o PowerShell injeta um BOM (`﻿`, U+FEFF) ao fazer pipe de string para stdin, o que quebra a validação do prompt interativo do `create-app` ("App name must be lowercase..."). No Bash isso não acontece.

Resultado: app criado com sucesso em `backstage-app/` (yarn install completo, `node_modules`, `packages/`, `plugins/`, `yarn.lock` presentes).

## Etapas 3-8: do local ao Kubernetes (concluídas em 21/09)

3. **Rodar o Backstage localmente** (dentro do WSL, após a migração) e validar que a UI sobe corretamente. **(concluído)**
   > Nota: o script correto é `yarn start`, não `yarn dev` (esse script não existe no `create-app`).
   Resultado: backend subiu na porta `:7007` (`200 OK` em `/healthcheck`), frontend compilado com sucesso na `:3000`. Único warning (esperado nesta fase): `Failed to initialize kubernetes backend: valid kubernetes config is missing`.
4. ~~Decidir com a equipe o padrão de banco de dados~~ **(decidido)** — Postgres rodando em pod dentro do próprio cluster Kubernetes (não será um serviço gerenciado externo).
5. `app-config.production.yaml` já estava configurado (banco via env vars `POSTGRES_HOST/PORT/USER/PASSWORD`, lidos do ConfigMap + Secret do k8s).
6. **Build da imagem Docker e carga no minikube.** **(concluído, com dois problemas sérios pelo caminho)**
7. **Manifests Kubernetes** (`k8s/namespace.yaml` + stack do Postgres + stack do Backstage + Ingress) — já existiam, só precisou ajustar os tempos do `readinessProbe`/`livenessProbe` do Backstage (ver abaixo).
8. **Deploy e verificação** — `kubectl apply -f k8s/...`, `kubectl get pods -n backstage`, `kubectl logs`. **(concluído, tudo saudável)**

### Problema 4 — Sessão interrompida no meio do `docker build` deixou um processo órfão

Uma sessão anterior do Claude Code foi encerrada bem no meio do `docker build` (na etapa final de exportar a imagem). Ao reiniciar e rodar o build de novo, o **processo antigo continuou rodando sozinho dentro do WSL** (o WSL não mata processos só porque a ferramenta que os iniciou fechou) e competia com o build novo pela mesma tag de imagem `backstage:latest`, travando os dois. Resolvido matando o processo órfão (`kill -9`) antes de deixar o build novo prosseguir. Boa notícia: o Docker reaproveitou o cache das camadas já construídas, então o build "novo" levou só ~2 minutos em vez de repetir os ~10 minutos do `yarn install` de produção.

### Problema 5 — `minikube image load` trava silenciosamente (containerd + driver docker)

Depois do build (imagem `backstage:latest`, 202 MB), o comando padrão `minikube image load backstage:latest` ficou "rodando" sem nunca terminar — o processo ficava preso num `futex`/`epoll_wait` esperando uma transferência via SSH para dentro do node que nunca avançava (confirmado checando que nenhum arquivo grande chegava em `/tmp` dentro do node). Isso é uma limitação conhecida do `minikube image load` quando o node usa **containerd** como runtime (em vez de dockerd) — o próprio `minikube docker-env` já havia avisado sobre isso ser "highly experimental".

**Correção aplicada** (contornando o comando quebrado):
1. `docker save backstage:latest -o /tmp/backstage.tar` — salvar a imagem como arquivo.
2. Copiar esse arquivo para dentro do container do node via `cat arquivo | docker exec -i minikube sh -c "cat > /tmp/backstage.tar"` (o `docker cp` direto também falhava silenciosamente — reportava sucesso mas o arquivo não aparecia do outro lado).
3. Importar a imagem direto no containerd do node: `docker exec minikube ctr --namespace k8s.io images import /tmp/backstage.tar`.

Resultado: imagem reconhecida pelo Kubernetes (`io.cri-containerd.image=managed`), pronta para os pods usarem com `imagePullPolicy: Never`.

### Problema 6 — `livenessProbe` matando o pod do Backstage antes dele terminar de inicializar

Na primeira aplicação do `backstage-deployment.yaml`, o pod ficou preso num ciclo de `0/1` → reinício → `0/1`. O evento do Kubernetes mostrou: `Liveness probe failed: ... context deadline exceeded`. Causa: o backend do Backstage, nesta máquina (poucos núcleos, disco pequeno), demora mais para inicializar todos os plugins do que os tempos padrão do manifest previam (`initialDelaySeconds: 20`). O Kubernetes matava o processo achando que travou, exatamente no meio da inicialização.

**Correção aplicada:** aumentados os tempos em `k8s/backstage-deployment.yaml`:
- `readinessProbe`: `initialDelaySeconds` de 10 → 30, `failureThreshold` 6
- `livenessProbe`: `initialDelaySeconds` de 20 → 60, `failureThreshold` 5

Depois do ajuste, o pod subiu limpo, sem reinícios.

## Resultado final (21/09, antes de qualquer preparação de produção)

```
kubectl get all,pvc -n backstage
```
- `deployment.apps/backstage`   1/1 Ready
- `deployment.apps/postgres`    1/1 Ready
- `persistentvolumeclaim/postgres-data`  Bound, 2Gi
- Serviço `backstage` do tipo `NodePort`, testado com sucesso via `curl` tanto por `kubectl port-forward` quanto direto no IP do node do minikube (`/healthcheck` e `/` retornando `200`).

## Como acessar

```powershell
# dentro da distro WSL (wsl -d Ubuntu-24.04):
minikube service backstage -n backstage
# ou, para pegar a URL sem abrir o navegador:
kubectl get svc backstage -n backstage
```

## Pendências antes de produção real (ainda não feitas)

Documentado, mas **fora do escopo até aqui**:
- Secrets via SealedSecrets/External Secrets (hoje é um Secret simples do k8s, senha em texto no arquivo)
- Ingress com TLS (o `ingress.yaml` já existe mas usa `backstage.local` sem certificado, e precisa do addon `ingress` habilitado no minikube)
- Corrigir `backend.baseUrl`/`app.baseUrl` em `app-config.production.yaml` — hoje aponta pra `localhost:7007`, o que o próprio Backstage já avisa nos logs como "misconfiguration" para ambientes reais (precisa ser uma URL roteável de verdade, ex. via Ingress)
- Integração do catálogo com GitHub (hoje só tem os exemplos estáticos do `create-app`)
- Pipeline de CI para build/push de imagem em um registry de verdade (hoje é tudo manual e local, com `imagePullPolicy: Never`)

## Atualizações (22/09)

**Definição de escopo com a liderança:** confirmado que **produção fica sob coordenação
do time de infraestrutura**. O papel deste time é entregar um **ambiente de
homologação validado**, que será usado por um período prolongado. Pendência em
aberto: confirmar quem disponibiliza o ambiente de homologação em si (infra
fornece servidor/cluster, ou o time sobe por conta própria) — hoje tudo ainda
roda só nesta máquina (notebook pessoal), o que não é suficiente para
homologação (precisa ser permanente e compartilhado).

**Documentação de reprodução criada:**
- `PLANO-DE-IMPLANTACAO.md` (+ versão `PLANO-DE-IMPLANTACAO.pdf`) — passo a
  passo autossuficiente para refazer o ambiente do zero em outra máquina, sem
  depender de IA. Auditado linha por linha contra os manifests reais do
  projeto antes de finalizar; a auditoria encontrou e corrigiu 3 problemas no
  primeiro rascunho (uma etapa obrigatória faltando — gerar `dist/*.tar.gz`
  via `yarn tsc && yarn build:backend` antes do `docker build` —, uma etapa
  que sugeria edição manual desnecessária, e referências cruzadas
  desalinhadas após a correção).
- `ROTEIRO-APRESENTACAO.md` — roteiro completo para apresentação interna,
  cobrindo todas as etapas técnicas, os 4 problemas enfrentados, e um
  glossário de termos.

**Projeto publicado no GitHub:** `https://github.com/babilods/backstage-camara`.
Cuidado de segurança aplicado antes do primeiro commit:
- `k8s/postgres-secret.yaml` (senha real) adicionado ao `.gitignore` do
  repositório raiz — nunca foi versionado.
- Criado `k8s/postgres-secret.example.yaml` como modelo, sem dado sensível.
- Detectado e corrigido um repositório git aninhado dentro de `backstage-app/`
  (criado automaticamente pelo `create-app`) — o `.git` interno foi removido
  para que os arquivos fossem versionados normalmente pelo repositório
  principal, em vez de aparecerem como um submódulo vazio.

**Incidente (22/09):** o arquivo `DEPLOYMENT_LOG.md` e o `PLANO-DE-IMPLANTACAO.pdf`
desapareceram do disco entre uma mensagem e outra (causa não identificada —
possivelmente ação externa ao Claude Code, já que não houve nenhum comando de
exclusão correspondente no histórico desta sessão). O `.md` foi recuperado via
`git restore` (já estava commitado); o `.pdf` não tinha sido commitado ainda e
precisou ser regenerado. Lição: considerar comitar no git logo após gerar
qualquer artefato importante (como o PDF), em vez de deixar só no disco.
