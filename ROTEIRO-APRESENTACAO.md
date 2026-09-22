# 🎤 Backstage no Kubernetes — Roteiro Completo

## Tópico 1 — Recapitulando de onde viemos
Na apresentação anterior, mostramos que trocamos o Docker Desktop por Docker Engine + WSL, porque o disco do computador vivia cheio e travava o ambiente inteiro. A partir dali, o Docker, o minikube e o próprio projeto do Backstage passaram a rodar dentro do "Linux de dentro do Windows" (WSL).

Hoje o objetivo era: **pegar o Backstage e colocar ele rodando de verdade dentro de um cluster Kubernetes, com banco de dados real**, do jeito que ficaria em produção — só que ainda testando localmente, no próprio computador, antes de levar pra empresa.

---

## Tópico 2 — Configurando e validando o cluster minikube

**O que é o minikube:** é uma versão "de bolso" do Kubernetes que roda inteira dentro de um único computador, só para testes. O Kubernetes de verdade normalmente é um conjunto de vários servidores trabalhando juntos; o minikube simula esse conjunto todo dentro de um container só, pra você poder testar sem precisar de uma infraestrutura real.

**Passo a passo:**
1. Rodei o comando `minikube start --driver=docker` — isso diz pro minikube: "suba o cluster, usando o Docker que já temos instalado como base".
2. O minikube criou e configurou as três peças que fazem o Kubernetes funcionar:
   - **control-plane**: o "cérebro" que toma as decisões (onde rodar cada aplicação, etc.)
   - **kubelet**: o "braço" que efetivamente liga e desliga os containers em cada máquina
   - **apiserver**: a "porta de entrada" por onde todos os comandos (inclusive os meus) conversam com o cluster
3. Validei que tudo subiu certo com `minikube status` — as três peças apareceram como `Running`.
4. Confirmei também com `kubectl get nodes`, que mostrou o node do cluster com status `Ready`.

**Resultado:** cluster funcionando e pronto para receber aplicações.

---

## Tópico 3 — Construindo a imagem Docker do Backstage e entregando pro minikube

**O que é uma imagem Docker:** é como empacotar o aplicativo inteiro (código + todas as bibliotecas que ele precisa pra funcionar) dentro de uma "caixa fechada" padronizada, que pode ser executada em qualquer lugar sem precisar instalar nada manualmente ali.

**Passo a passo:**
1. Usei o `Dockerfile` que o próprio Backstage já gera automaticamente quando o projeto é criado (`create-app`). Esse arquivo descreve, passo a passo, como montar a imagem: copiar o código, instalar as dependências de produção, empacotar tudo.
2. Rodei `docker build -t backstage:latest` — isso executa o Dockerfile e gera a imagem. O processo reaproveitou o cache de uma instalação anterior de dependências, o que acelerou bastante o build.
3. A imagem final ficou com **202 MB**.
4. Depois de pronta, precisei **entregar essa imagem pro minikube** — porque o minikube roda isolado, num "mundo próprio", e não enxerga as imagens que ficam só no Docker do computador. Isso é feito configurando o Deployment do Kubernetes com `imagePullPolicy: Never`, avisando: "não tente baixar essa imagem de nenhum lugar da internet, ela já está aqui dentro".

**Resultado:** imagem construída e disponível dentro do ambiente do minikube, pronta pra ser usada por um Deployment.

---

## Tópico 4 — Deploy do banco de dados (Postgres) e armazenamento

**Por que precisa de um banco:** o Backstage precisa guardar informações permanentes (catálogo de serviços, usuários, configurações) em algum lugar. Isso é feito com um banco de dados Postgres rodando dentro do próprio cluster.

**Passo a passo:**
1. Criei um **Deployment** do Postgres — a "receita" que diz ao Kubernetes: "mantenha sempre 1 cópia rodando desse banco de dados".
2. Criei um **Secret** — um recurso do Kubernetes específico para guardar informação sensível (usuário, senha e nome do banco) separado do resto da configuração.
3. Criei um **PersistentVolumeClaim (PVC)** de 2Gi — um "espaço de armazenamento reservado" que existe independente do container. Isso é essencial: sem isso, se o pod do Postgres reiniciasse por qualquer motivo, todos os dados guardados seriam perdidos junto.
4. Apliquei tudo no cluster e acompanhei o pod subir: ele inicializou o banco pela primeira vez, reiniciou uma vez sozinho (isso é comportamento normal da imagem oficial do Postgres, não é erro) e ficou saudável (`1/1 Ready`).

**Resultado:** banco de dados rodando dentro do cluster, com armazenamento persistente garantido.

---

## Tópico 5 — Executando o Backstage dentro do cluster

**Passo a passo:**
1. Criei o **Deployment** do Backstage, apontando para a imagem que construímos no Tópico 3.
2. Conectei esse Deployment ao banco de dados usando dois recursos do Kubernetes:
   - Um **ConfigMap**, com o endereço e a porta do Postgres (`POSTGRES_HOST`, `POSTGRES_PORT`)
   - O mesmo **Secret** do Tópico 4, com usuário e senha
3. Configurei o Backstage para escutar na porta **7007** (a porta padrão do backend dele).
4. Criei um **Service** do tipo `NodePort` — isso expõe essa porta pra fora do cluster, permitindo acesso externo (não só de dentro do "mundo" do Kubernetes).

**Resultado:** Backstage rodando dentro do cluster, conectado ao banco real.

---

## Tópico 6 — Testando a aplicação no Kubernetes

Não bastava o pod aparecer como "rodando" — precisava confirmar que a aplicação realmente funcionava de ponta a ponta.

**Passo a passo dos testes:**
1. Testei **de dentro do cluster**, usando `kubectl port-forward` (uma espécie de túnel temporário direto pro pod) e chamando o endereço de verificação de saúde (`/healthcheck`) → respondeu **200 OK**.
2. Testei **de fora do cluster**, como um usuário real acessaria — usando o IP do node do minikube junto com a porta exposta pelo `NodePort` → respondeu **200 OK** também, tanto no `/healthcheck` quanto na tela inicial (UI).

**Resultado:** confirmado que a aplicação, o banco de dados e a rede do Kubernetes estão todos funcionando juntos corretamente — não só "no ar", mas realmente operando.

---

## Tópico 7 — Integração do Backstage com o Kubernetes

Vale mencionar: o Backstage tem um plugin próprio que permite mostrar, dentro da própria interface dele, informações sobre o que está rodando no cluster Kubernetes (uma espécie de painel de monitoramento integrado).

Essa configuração específica **ainda não foi feita** — aparece um aviso nos logs (`Failed to initialize kubernetes backend`), mas isso **não impede nada** do que já está funcionando. Fica marcado como uma melhoria futura, não um bloqueio.

---

## Tópico 8 — Problemas e dificuldades enfrentadas

**Problema 1 — Processo de build "fantasma"**
Uma sessão de trabalho anterior foi interrompida bem no meio da construção da imagem Docker. Quando tentei rodar de novo, descobri que o processo antigo **continuava rodando escondido** em segundo plano, competindo com o processo novo pela mesma imagem — os dois travados. *Solução: identifiquei e encerrei o processo antigo manualmente, deixando só o novo terminar.*

**Problema 2 — Comando de carregar imagem no minikube trava sem terminar**
Depois de pronta, a imagem precisava ser entregue pro minikube usar. O comando oficial pra isso (`minikube image load`) simplesmente **ficou parado, sem nunca terminar** — uma limitação conhecida quando o motor interno usado pelo Kubernetes (chamado containerd) é diferente do que esse comando espera. *Solução: contornei manualmente, salvando a imagem como um arquivo e importando ela diretamente dentro do node do cluster.*

**Problema 3 — Cópia de arquivo falhando silenciosamente**
Na tentativa de contornar o Problema 2, o comando usado pra copiar o arquivo (`docker cp`) **dizia que tinha funcionado, mas o arquivo simplesmente não chegava** do outro lado. *Solução: troquei a estratégia de cópia, usando um redirecionamento direto de dados (pipe) em vez do comando de cópia.*

**Problema 4 — Pod do Backstage entrando em loop de reinício**
Ao subir o Backstage pela primeira vez no cluster, o pod ficava sendo **morto e recriado repetidamente**. O motivo: o Kubernetes tem um "verificador de saúde" automático que bate na porta da aplicação pra ver se ela está viva; esse verificador estava configurado pra ser mais impaciente do que o Backstage precisa pra inicializar todos os seus módulos internos — principalmente numa máquina com recursos mais limitados. O Kubernetes achava que a aplicação tinha travado e matava ela bem no meio da inicialização. *Solução: aumentei o tempo de tolerância desse verificador de saúde.*

---

## Tópico 9 — Resultado final / status atual

- Deployment do Backstage: **1/1 Ready**
- Deployment do Postgres: **1/1 Ready**
- Volume de armazenamento do banco: **Bound** (reservado e funcionando)
- Testes de ponta a ponta: **aprovados**

**Importante deixar claro na apresentação:** isso é um **ambiente de teste completo e funcional**, não é produção ainda.

---

## Tópico 10 — Documentação para reprodução: o Plano de Implantação

Pra garantir que esse trabalho não fique só na minha máquina, criei um **Plano de Implantação** completo — um passo a passo detalhado, testado, que permite refazer esse ambiente do zero em qualquer outra máquina, **sem precisar de ajuda de IA**.

**A técnica usada para montar esse plano — não foi só "escrever de memória":**

1. **Baseado no histórico real de comandos executados**, não em teoria — cada passo do plano corresponde a um comando que realmente rodei e validei nesta máquina, com o resultado esperado documentado ao lado.

2. **Auditoria cruzada contra os arquivos reais do projeto.** Antes de considerar o plano pronto, comparei cada trecho de configuração (os manifests do Kubernetes, o Dockerfile) **linha por linha com os arquivos de verdade** do repositório — não apenas assumi que estava certo.

3. **Essa auditoria encontrou e corrigiu 3 problemas reais no primeiro rascunho:**
   - Faltava uma etapa inteira e obrigatória (gerar os arquivos de build antes do `docker build` — sem isso, o processo falharia numa máquina nova, mas passou despercebido aqui porque esses arquivos já existiam de um trabalho anterior)
   - Uma etapa dava a entender que precisava editar um arquivo de configuração manualmente, quando na verdade ele já vem pronto por padrão
   - Referências cruzadas entre etapas ficaram desalinhadas depois de uma correção, e precisaram ser recontadas uma por uma

4. **Os 4 problemas reais enfrentados (Tópico 8) foram incorporados como avisos preventivos** no próprio plano — quem for seguir esse passo a passo já sabe, antes de tentar, quais comandos vão travar e qual é o caminho alternativo certo.

**Por que isso importa destacar:** um plano de implantação não serve de nada se ele descreve o que "deveria" funcionar em teoria — o valor real está em documentar o que **de fato** funciona, incluindo os erros já resolvidos, validado contra a realidade do sistema, não contra a memória de quem escreveu.

Esse documento serve tanto para eu reproduzir localmente quanto como **referência técnica pronta** para quem for montar o ambiente de homologação.

> Arquivos relacionados no projeto: `PLANO-DE-IMPLANTACAO.md` (e sua versão em PDF, `PLANO-DE-IMPLANTACAO.pdf`).

---

## Tópico 11 — Versionamento e segurança do código

O projeto foi publicado num repositório Git (GitHub), com um cuidado importante de segurança: **a senha real do banco de dados não foi enviada** ao repositório. No lugar, ficou apenas um arquivo de exemplo, sem nenhum dado sensível — quem for usar o projeto sabe o formato esperado, mas precisa criar sua própria senha.

Isso já demonstra uma boa prática básica de segurança desde o início — algo que só vai ficar mais rigoroso conforme o projeto avança para homologação e produção.

> Repositório: `https://github.com/babilods/backstage-camara`

---

## Tópico 12 — O que falta antes de ir para homologação (e o que fica com a infra depois)

Conversamos com a liderança e ficou definido: **produção fica sob coordenação do time de infraestrutura**. Nosso papel é entregar um **ambiente de homologação validado**, que será usado por um período prolongado.

**O que ainda precisamos resolver, antes de homologação:**
- Confirmar com a liderança **quem disponibiliza o ambiente de homologação** — se a infra fornece um servidor/cluster já pronto, ou se o próprio time tem autonomia para subir isso
- Hoje tudo roda só no meu notebook (depende da minha sessão continuar aberta) — homologação precisa ser algo **permanente e compartilhado**, acessível por outras pessoas

**O que já sabemos que fica com a infra, mais adiante (produção):**
- 🔴 Gerenciamento de segredos por um cofre corporativo de verdade (hoje é um Secret simples do Kubernetes)
- 🔴 Corrigir o endereço interno da aplicação (hoje ainda aponta para "localhost")
- 🔴 Um cluster Kubernetes de produção real da empresa
- 🔴 Um registry de imagens corporativo de verdade
- 🟡 HTTPS/certificado de segurança
- 🟡 Uma **esteira** (pipeline de CI/CD) definitiva — automatizar o build e publicação da imagem, hoje feito manualmente
- 🟡 Catálogo do Backstage conectado aos repositórios reais da empresa no GitHub
- 🟢 Configurar a integração do Backstage com o Kubernetes (Tópico 7)
- 🟢 Monitoramento e alertas automáticos

---

## Tópico 13 — Resumo executivo

✅ **Feito e validado:** ambiente de teste completo, com Backstage rodando dentro do Kubernetes, conectado a um banco de dados real, testado de ponta a ponta, documentado para reprodução e versionado com segurança no GitHub.

⏭️ **Próxima etapa:** entregar este ambiente validado para **homologação**. **Produção fica sob coordenação do time de infraestrutura** — nosso papel é entregar algo testado e documentado, pronto para ser usado em homologação por um período prolongado.

---

# 📚 O que você precisa saber de cor

## Os conceitos-base (se perguntarem "o que é X", responda assim)

- **Docker** = empacota o código + tudo que ele precisa pra rodar, numa "caixa padronizada" que funciona igual em qualquer máquina.
- **Imagem Docker** = a caixa fechada e pronta (o "molde"). **Container** = a caixa em execução (o "molde rodando de verdade").
- **Kubernetes** = o "gerente" que decide onde rodar cada container, reinicia quem trava, garante que a quantidade certa de cópias está sempre no ar.
- **Minikube** = um Kubernetes "de bolso", rodando dentro de uma máquina só, usado só para testar — não é pra produção.
- **Pod** = a menor unidade que o Kubernetes gerencia (geralmente 1 container dentro).
- **Deployment** = a "receita" que diz ao Kubernetes quantas cópias de um pod manter rodando.
- **Service** = um "nome fixo" pra acessar um pod, mesmo que o pod reinicie e troque de IP.
- **Namespace** = uma "gaveta" dentro do Kubernetes que separa e organiza os recursos de um projeto dos outros (o nosso se chama `backstage`).
- **ConfigMap** = configuração não-sensível (tipo endereço do banco).
- **Secret** = informação sensível (senha, chave) — mas atenção: hoje só está **codificada**, não **criptografada de verdade**, isso é uma pendência real pra produção.
- **PVC (PersistentVolumeClaim)** = "reserva de disco" que sobrevive mesmo se o pod reiniciar ou morrer.
- **WSL** = Linux rodando dentro do Windows — usamos porque o Docker roda nativamente em Linux, e ferramentas do ecossistema Kubernetes se comportam melhor lá do que direto no Windows.
- **Esteira** = termo popular pra **pipeline de CI/CD** — o processo automatizado que builda e publica a aplicação sozinho, sem comandos manuais. Hoje ainda não temos, o processo é manual.
