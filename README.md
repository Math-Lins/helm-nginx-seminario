# Seminário de Helm: chart próprio simples com Nginx

Tema T07 · Helm. Renderizar e instalar um chart próprio simples, alterando um parâmetro de configuração.

## Sumário

1. [Objetivo](#objetivo)
2. [Arquitetura](#arquitetura)
3. [Ambiente e versões](#ambiente-e-versões)
4. [Preparação](#preparação)
5. [Executar a demonstração](#executar-a-demonstração)
6. [O que explicar durante a demonstração](#o-que-explicar-durante-a-demonstração)
7. [Solução rápida de problemas](#solução-rápida-de-problemas)
8. [Limpeza](#limpeza)
9. [Vantagens, desvantagens e panorama](#vantagens-desvantagens-e-panorama)
10. [Limitações](#limitações)
11. [Fontes e créditos](#fontes-e-créditos)

## Objetivo

Demonstrar, com um chart criado pela equipe, como o Helm renderiza e instala instruções para o Kubernetes e como uma configuração pode ser alterada. O parâmetro escolhido é `replicaCount`: primeiro executamos uma cópia do Nginx e depois mudamos para duas.

Todos os dados são fictícios: o Nginx serve apenas a página padrão.

### O que cada parte faz

- **Nginx** é o programa de exemplo que entrega uma página web.
- **Chart** é a pasta com as instruções e configurações da aplicação.
- **Helm** lê o chart, mostra o resultado da configuração e envia as instruções ao Kubernetes.
- **Kubernetes** inicia e acompanha os programas. Neste projeto, ele mantém a quantidade de cópias indicada no Deployment.
- **Pod** é uma unidade em execução no cluster; neste exemplo, cada pod executa uma cópia do Nginx.
- **Release** é o nome de uma instalação feita pelo Helm; aqui, `demo`.

Quem acessa a página normalmente não percebe a mudança de uma para duas cópias: ela ocorre nos bastidores. Duas cópias podem ajudar a atender mais acessos, mas este exemplo local não mede capacidade nem garante alta disponibilidade.

## Arquitetura

O chart foi escrito para este seminário; não é um chart pronto baixado de um repositório.

```text
chart/
  Chart.yaml                  Nome e versão do chart (nginx-seminario, 0.1.0)
  values.yaml                 Configurações padrão, incluindo replicaCount: 1
  templates/
    deployment.yaml           Instrui o Kubernetes a executar o Nginx
    service.yaml              Fornece um endereço interno para acessar o Nginx
```

Fluxo da demonstração:

```text
values.yaml + templates/  -->  Helm (lint, template, install, upgrade)
                                   |
                                   v
                      manifestos: Deployment + Service
                                   |
                                   v
                      API do Kubernetes (estado atual x desejado)
                                   |
                                   v
                      Pods do Nginx (1 e depois 2 cópias)
```

O valor `replicaCount` de `values.yaml` é usado pelo template do Deployment: onde o modelo escreve `{{ .Values.replicaCount }}`, o Helm coloca o valor na renderização. O nome da release (`{{ .Release.Name }}`) dá nome aos recursos, por isso o Deployment se chama `demo-nginx`. O Service seleciona os pods do Nginx pela label `app.kubernetes.io/instance` e leva a porta 80 à porta `http` do container. Como o tipo é `ClusterIP`, o Service só é acessível de dentro do cluster.

| Valor (`values.yaml`) | Padrão | Onde é usado |
|---|---|---|
| `replicaCount` | `1` | `deployment.yaml`: número de réplicas |
| `image.repository` / `image.tag` | `nginx` / `1.27.5-alpine` | `deployment.yaml`: imagem do container |
| `image.pullPolicy` | `IfNotPresent` | `deployment.yaml`: política de download |
| `service.type` / `service.port` | `ClusterIP` / `80` | `service.yaml`: tipo e porta do Service |

### Precedência dos valores

O `values.yaml` traz o valor padrão e o `--set`, na linha de comando, sobrescreve esse padrão. Usamos `--set` porque a mudança fica visível no próprio comando e o padrão do chart continua intacto.

### Decisões e alternativas

| Decisão | Alternativa descartada | Motivo |
|---|---|---|
| Chart próprio | Chart pronto de repositório | Controle de cada arquivo e explicação de cada escolha |
| `replicaCount` como parâmetro | Mudar a versão da imagem | Efeito simples de verificar: um Pod ou dois |
| Service `ClusterIP` | `NodePort`, `LoadBalancer` | Só atende dentro do cluster; o foco é configuração, não exposição |
| Tag `1.27.5-alpine` | `latest` | Mesma versão do Nginx em toda execução |
| `--set` no comando | Editar o `values.yaml` | A mudança aparece no comando e o padrão do chart fica intacto |

## Ambiente e versões

A execução foi realizada em Windows 11 Pro 25H2, com Minikube usando o driver Docker Desktop.

| Componente | Versão registrada |
|---|---|
| Helm | 4.3.1 |
| Minikube | 1.39.0 |
| kubectl (cliente) | 1.36.1 |
| Kubernetes (cluster Minikube) | 1.37.0 |
| Imagem do Nginx | `nginx:1.27.5-alpine` |
| Docker Engine | 29.8.0 |

O cluster criado para a demonstração informou 2 CPUs e 8000 MB de memória. A primeira inicialização baixa imagens e pode levar alguns minutos.

## Preparação

### Dependências

1. **Docker Desktop para Windows**, configurado para contêineres Linux. Ele fornece o ambiente Docker usado pelo Minikube.
2. **kubectl**, para consultar e controlar o cluster Kubernetes.
3. **Helm**, para renderizar, instalar e atualizar o chart.
4. **Minikube**, para criar um cluster Kubernetes local no computador.
5. **Internet na primeira execução**, para baixar a imagem do Nginx e as imagens usadas pelo Minikube.

### Instalar o Docker Desktop (se ainda não estiver instalado)

1. Abra a página oficial de [instalação do Docker Desktop para Windows](https://docs.docker.com/desktop/setup/install/windows-install/) e baixe o instalador.
2. Execute o instalador e escolha o backend **WSL 2** quando essa opção aparecer. Conclua a instalação e reinicie o Windows se o instalador solicitar.
3. Abra o Docker Desktop pelo menu Iniciar e aguarde até ele terminar de iniciar, no modo de contêineres Linux.
4. No PowerShell, confirme que o mecanismo está disponível:

```powershell
docker version
```

Se o WSL 2 ainda não estiver instalado, abra o PowerShell como administrador e siga as instruções oficiais do [WSL](https://learn.microsoft.com/windows/wsl/install), reiniciando o computador quando solicitado. Para verificar o WSL, use `wsl --version`; para atualizar uma instalação existente, use `wsl --update`.

### Instalar kubectl, Helm e Minikube

Abra o PowerShell e confira se o `winget` está disponível:

```powershell
winget --version
```

Execute uma linha por vez:

```powershell
winget install -e --id Kubernetes.kubectl --source winget
winget install -e --id Helm.Helm --source winget
winget install -e --id Kubernetes.minikube --source winget
```

Se o PowerShell perguntar se você aceita os termos, digite `Y` enquanto a pergunta estiver aparecendo e pressione Enter. Depois que o comando terminar, não digite `Y` de novo como se fosse um comando.

Feche e abra novamente o PowerShell para atualizar o PATH. Inicie o Docker Desktop, aguarde até ele estar pronto e confira as ferramentas:

```powershell
docker version
helm version
minikube version
kubectl version --client
```

A versão do Docker Engine registrada neste teste é `29.8.0`. Para registrar também a versão do aplicativo Docker Desktop, abra **About/Sobre** no próprio aplicativo. O comando `docker desktop version --short` mostra a versão do plugin de linha de comando, não a do aplicativo.

### Preparar o cluster local

Confirme que o Docker Desktop terminou de iniciar e usa contêineres Linux:

```powershell
docker version
docker info --format '{{.OSType}}'
```

O primeiro comando deve mostrar **Client** e **Server**; o segundo deve imprimir `linux`. Se aparecer `windows`, mude o Docker Desktop para contêineres Linux.

Inicie o cluster e confira:

```powershell
minikube start --driver=docker
kubectl get nodes
```

O nó `minikube` deve aparecer com status `Ready`. Se o computador for reiniciado, abra o Docker Desktop e execute `minikube start --driver=docker` novamente.

### Checklist antes da demonstração

- [ ] Docker Desktop aberto, em contêineres Linux.
- [ ] `kubectl get nodes` mostra o nó `minikube` como `Ready`.
- [ ] `helm list` não mostra a release `demo` (se mostrar, execute `helm uninstall demo`).
- [ ] PowerShell aberto na pasta do projeto, onde estão este README e a pasta `chart`.
- [ ] Imagem do Nginx já baixada em um ensaio (evita espera durante a apresentação).
- [ ] Ensaio completo feito: conferir a saída real de `helm lint` e dos comandos `kubectl`.
- [ ] Prints e gravação de uma execução completa guardados como plano B.

## Executar a demonstração

Extraia o arquivo ZIP do projeto e, no PowerShell, entre na pasta principal (a que contém este README e a pasta `chart`). Ajuste o caminho ao local onde extraiu:

```powershell
cd "C:\caminho\para\helm-nginx-seminario"
```

Os resultados abaixo descrevem o que deve aparecer; a formatação exata das tabelas pode variar entre versões. Confirme-os em um ensaio antes da apresentação.

### 1. Conferir o chart

```powershell
helm lint .\chart
```

Resultado esperado: `1 chart(s) linted, 0 chart(s) failed`. O aviso de que um ícone é recomendado é opcional e não indica falha.

### 2. Renderizar sem instalar

```powershell
helm template demo .\chart --set replicaCount=2
```

O Helm imprime os manifestos Kubernetes gerados a partir do chart. Procure `kind: Deployment` e `replicas: 2`, e também `kind: Service`. Nesta etapa nada foi instalado no cluster.

### 3. Instalar com uma cópia

```powershell
helm install demo .\chart --set replicaCount=1
kubectl rollout status deployment/demo-nginx
kubectl get deployment demo-nginx
kubectl get pods -l app.kubernetes.io/instance=demo
```

Resultado esperado: `READY 1/1` no Deployment e um pod com status `Running`.

### 4. Alterar para duas cópias

```powershell
helm upgrade demo .\chart --set replicaCount=2
kubectl rollout status deployment/demo-nginx
kubectl get deployment demo-nginx
kubectl get pods -l app.kubernetes.io/instance=demo
```

Resultado esperado: `READY 2/2` no Deployment e dois pods com status `Running`. O `helm upgrade` atualiza a instalação chamada `demo`; não é necessário instalar o chart outra vez.

Para mostrar o valor que ficou salvo na instalação:

```powershell
helm get values demo
```

O comando deve mostrar `replicaCount: 2`.

### 5. Abrir a página do Nginx

```powershell
kubectl port-forward service/demo-nginx 8080:80
```

Mantenha o comando em execução e abra [http://localhost:8080](http://localhost:8080). Deve aparecer a página padrão do Nginx, igual com uma ou com duas cópias: a prova do número de cópias está no `kubectl`, não no navegador. Para parar o encaminhamento, pressione `Ctrl+C` no PowerShell.

## O que explicar durante a demonstração

1. `helm lint` verifica a estrutura do chart.
2. `helm template` mostra os manifestos que seriam enviados ao Kubernetes; `--set replicaCount=2` muda o valor usado na renderização (o `--set` vence o `values.yaml`).
3. `helm install` cria a release `demo` com uma cópia do Nginx.
4. `helm upgrade` altera a mesma release para duas cópias.
5. `kubectl get deployment` e `kubectl get pods` confirmam o resultado real no cluster.
6. O navegador mostra a página servida pelo Nginx. A página pode parecer igual; a diferença comprovada é haver duas cópias em execução.

## Solução rápida de problemas

- **`helm`, `kubectl` ou `minikube` não é reconhecido:** feche e abra o PowerShell para atualizar o PATH; se persistir, reinstale a ferramenta com `winget`.
- **`helm install` diz que a release `demo` já existe:** ela já foi instalada. Para mudar sua configuração, use `helm upgrade demo .\chart --set replicaCount=2`. Para começar do zero, execute `helm uninstall demo` e instale novamente.
- **`kubectl get nodes` não mostra `Ready`:** abra o Docker Desktop e execute `minikube start --driver=docker` novamente.
- **Pod em `ContainerCreating` ou ainda não `Running`:** aguarde alguns segundos e repita `kubectl get pods`. A imagem pode estar sendo baixada.
- **Falha ao baixar imagens:** confira a conexão com a internet e se o Docker Desktop está aberto.
- **A porta 8080 já está ocupada:** use outra porta local, por exemplo `kubectl port-forward service/demo-nginx 8081:80`, e abra `http://localhost:8081`.

## Limpeza

Depois da apresentação ou dos testes, remova a release e confira:

```powershell
helm uninstall demo
helm list
kubectl get pods -l app.kubernetes.io/instance=demo
```

`helm list` não deve mais mostrar `demo`, e a consulta de pods deve informar que nenhum recurso foi encontrado. Para parar também o cluster local (opcional):

```powershell
minikube stop
```

Para apagar o cluster por completo (opcional; a próxima inicialização baixará tudo de novo):

```powershell
minikube delete
```

## Vantagens, desvantagens e panorama

**Vantagens**

- Instalação repetível: modelos e valores ficam reunidos no chart.
- Configuração por valores, sem editar cada manifesto.
- Atualização com um único comando e validação antes de instalar (`helm lint`, `helm template`).
- O mesmo chart pode ser usado em vários ambientes.

**Desvantagens**

- Curva de aprendizado: além do YAML, é preciso entender a sintaxe dos modelos.
- Erros em modelos são mais difíceis de depurar (por isso validamos antes de instalar).
- É uma camada a mais sobre o `kubectl`.
- Para uma aplicação mínima, de poucos arquivos e sem variações entre ambientes, aplicar o YAML direto pode bastar.

| Critério | Avaliação |
|---|---|
| Custo | Gratuito e de código aberto, com licença Apache 2.0. O custo real é o tempo de aprendizado e a infraestrutura do cluster; neste projeto, um Minikube local, sem gasto. |
| Popularidade | Na pesquisa anual da CNCF, 75% dos respondentes apontaram o Helm como gerenciador de pacotes preferido para Kubernetes, segundo a cobertura da imprensa; é projeto graduado da CNCF. A amostra é de profissionais, não do mercado inteiro. |
| Acesso | Instalação por `winget` e documentação oficial gratuita. |
| Usabilidade | Comandos principais simples (`lint`, `template`, `install`, `upgrade`); os modelos exigem aprender uma sintaxe própria. |
| Praticidade | Alta para repetir e atualizar instalações; menor para uma aplicação única e minúscula. |

## Limitações

Este exemplo usa um cluster Minikube local de um único nó. As duas cópias do Nginx demonstram uma alteração de configuração, e a atualização da release, mas não representam um teste de carga nem uma arquitetura de alta disponibilidade. Se o nó local parar, as duas cópias podem ser afetadas. O Service é do tipo `ClusterIP`, portanto interno; o `port-forward` serve apenas para a demonstração. O chart não configura domínio, HTTPS, balanceador externo, armazenamento persistente ou monitoramento.

Próximos passos possíveis: executar em vários nós, expor a aplicação com segurança (por exemplo, Ingress com HTTPS), adicionar monitoramento e fazer testes de carga.

## Fontes e créditos

- [Documentação do Helm](https://helm.sh/docs/)
- [Arquivos de valores](https://helm.sh/docs/chart_template_guide/values_files/)
- [História do Helm](https://helm.sh/community/history)
- [CNCF: Helm Project Journey Report](https://www.cncf.io/reports/cncf-helm-project-journey-report/)
- [Documentação do Kubernetes](https://kubernetes.io/docs/)
- [Instalação do kubectl no Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- [Instalação do Helm](https://helm.sh/docs/intro/install/)
- [CNCF Annual Survey 2024](https://www.linuxfoundation.org/hubfs/Research%20Reports/cncf_annual_survey24_031225a.pdf) e [cobertura da Cloud Native Now](https://cloudnativenow.com/editorial-calendar/best-of-2025/cncf-survey-surfaces-steady-pace-of-increased-cloud-native-technology-adoption-2/) (75% de preferência)
- [Helm no GitHub](https://github.com/helm/helm) (licença Apache 2.0)
- [Início rápido do Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [Instalação do Docker Desktop no Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- Imagem de contêiner do Nginx: `nginx:1.27.5-alpine`, distribuída pelo Docker Hub.

## Integrantes

| Integrante | Matrícula |
|---|---|
| Laryssa Rayanne Souza Martins | 01612424 |
| Pedro Arthur Rodrigues | 01522993 |
| Matheus de Oliveira Lins Mendes Simes | 01618966 |
| Rafael Ferreira dos Anjos | 01579531 |
| Elizeu Leoncio Ferreira Junior | 01576238 |
