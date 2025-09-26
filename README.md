# hello-app

## Descrição:

O objetivo desse projeto é automatizar o ciclo completo de desenvolvimento, build, deploy e execução de uma aplicação FastAPI simples, usando GitHub Actions para CI/CD, Docker Hub como registry, e ArgoCD para entrega contínua em Kubernetes local com Rancher Desktop.

## Pré-requisitos

- Conta no GitHub
- Conta no DockerHub
- Rancher Desktop com Kubernetes habilitado
- kubectl configurado corretamente (kubectl get nodes)
- ArgoCD instalado no cluster local
- Git instalado
- Python 3 e Docker instalados

## Criando nossa aplicação

Para fins de exemplo, nossa aplicação será simples
Crie um repositório com uma pasta `apps` e crie tambem o arquivo `main.py`

Dentro de `main.py` adicione o seguinte código:

```python
from fastapi import FastAPI


app = FastAPI()


@app.get("/")

async def root():
	return {"message": "Hello, World!"}
```

Devemos, então criar um Dockerfile para poder buildar a imagem do nosso pequeno aplicativo:

Volte uma pasta e crie um `Dockerfile`. Seu conteúdo será:

```Dockerfile
FROM python:3.9

WORKDIR /code

COPY ./requirements.txt /code/requirements.txt

RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt

COPY ./app /code/app

CMD ["fastapi", "run", "app/main.py", "--port", "80", "--proxy-headers"]
```

Perceba que ele utiliza um arquivo chamado `requirements.txt` para instalar as dependências do app. Vamos criá-lo então.

Na mesma pasta que o arquivo `Dockerfile` crie o `requirements.txt`.
Dentro dele, insira as dependências necessarias:

```requirements.txt
fastapi[standard]>=0.113.0,<0.114.0
```

> Essa estratégia é vantajosa para manter a organização e também o controle das dependências do app.

Pronto, nosso app está feito e o Dockerfile ajustado. Para otimizarmos o deploy utilizaremos as funcionalidades do Github Actions.

## Criando um fluxo de deploy utilizando Github Actions

Para o script do Github Actions funcionar devemos criar um ambiente apropriado.

Ainda no mesmo diretório, crie uma pasta chamada `.github`, nela crie outra pasta chamada de `workstation`. É nela onde criaremos nosso arquivos yamls que servirão como scripts de automação. Nesse exemplo estarei nomeando como `build_and_push.yaml`

O conteudo de build_and_push.yaml será:

> Mude o valor das variáveis em `env:` para o nome do seu repositório no DockerHub e o do repositório de manifestos que iremos criar posteriormente

```build_and_push.yaml
name: ci

on:
  push:
    tags:
      - "v*.*"

env:
  DOCKER_REPO: vitorialeda/projeto-4
  MANIFEST_REPO: vitorialeda/hello-manifest

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ env.DOCKER_REPO }}:${{ github.ref_name }}

      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          repository: ${{env.MANIFEST_REPO}}
          ref: main
          token: ${{ secrets.PAT }}

      - name: Update image tag in deployment file
        run: |
          sudo apt-get install -y yq
          yq -i -y '.spec.template.spec.containers[0].image = "${{ env.DOCKER_REPO }}:${{ github.ref_name }}"' k8s/deployment.yaml

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v5
        with:
          token: ${{ secrets.PAT }}
          commit-message: Update image tag to ${{ github.ref_name }}
          title: Update image tag to ${{ github.ref_name }}
          body: This PR updates the image tag in the deployment file to ${{ github.ref_name }}.
          branch: update-image-tag-${{ github.ref_name }}
          base: main

```

- Referências:
  - [Documentação Docker](https://docs.docker.com/build/ci/github-actions/)
  - [Documentação da Action create-pull-request@v5](https://github.com/peter-evans/create-pull-request)

> A estrutura final desse repositório ficará assim:
> o README.md não é fundamental mas é recomendado :))
> ![estrutura_hello-main](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/1_estrutura_hello-main.png)

## Criando um repositório remoto

No seu perfil, clique em `Repositories`e depois em `New`.
![Criando novo repo](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/2_Criando%20novo%20repo.png)

Nomeie o repositório e clique em `Create repository`. Neste exemplo estarei chamando de hello-app
![nomeando repositorio](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/3_nomeando%20repositorio.png)

Copie o link `HTTPS` em `<> Code`

![copiando_link](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/4_copiando_link.png)

No terminal, dentro da pasta em que criamos nossos arquivos, cole o seguinte, substituindo pelo seu link:

```bash
git init
git remote add origin https://github.com/vitorialeda/hello-app.git

git add .
git commit -m "Aplicao"
git push -u origin main
```

Pronto, nosso repositório tá no jeito. Mas ainda tem um detalhe.
Nosso arquivo `build_and_build.yaml` utiliza secrets para funcionar, por isso devemos configurar no nosso repositório remoto.

## Criando os secrets

Utilizaremos no total 3 secrets:

- DOCKER_USERNAME
- DOCKER_PASSWORD
- PAT

Vá até o repositório onde está nosso build_and_push.yaml (nesse exemplo é o hello-app), clique em settings, depois em `Secrets and variables` , depois em `Actions`.

![settando primeiro secret](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/5_settando%20primeiro%20secret.png)

Clique em "New repository secret"
![new repo secret](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/6_new%20repo%20secret.png)

### DOCKER_USERNAME:

![docker username](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/7_docker%20username.png)

### DOCKER_PASSWORD

Por questões de segurança, vamos criar um Personal Acess Token no DockerHub para o secret DOCKER_PASSWORD:

Acesse o DockerHub, clique na sua foto de perfil no canto superior direito e depois em "Account settings"
![acessando configuracoes dockerhub](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/8_acessando%20configuracoes%20dockerhub.png)

Na barra lateral, clique em Personal access tokens:
![pat docker hub](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/9_pat%20docker%20hub.png)

Clique em "Generate new token":
![generate new token](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/10_generate%20new%20token.png)

Nomeie a chave, escolha o tempo de vida da chave, escolha `Read & Write` e clique em "Generate"
![config token](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/11_config%20token.png)

Copie a senha gerada:

![copiando senha](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/12_copiando%20senha.png)

Crie um novo secret, nomeie como DOCKER_PASSWORD, cole a senha gerada na aba "secret" e clique em "Add secret"

![finalizando docker password](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/13_finalizando%20docker%20password.png)

### PAT

Clique no [link](https://github.com/settings/apps), clique em "Generate new token" e depois em "Generate new tocken (classic)"
![PAT classico](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/14_PAT%20classico.png)

Escreva uma descrição, marque a opção de `repo` e `workflow` e depois clique em "Generate token"
![workflow](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/15_workflow.png)

Copie a chave e crie um novo Secret no repositório hello-app
![copia token](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/16_copia%20token.png)

![cola pat](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/17_cola%20pat.png)

Pronto! Essa parte do projeto está concluida. Vamos para a automação com ArgoCD.

## Criando um repositório com manifetos do ArgoCD

Crie um novo repositório com uma pasta `k8s`que comporta os arquivos `deployment.yaml`e `service.yaml`. Nesse exemplo o repositório remoto será chamado de `hello-manifest`

Nele nos teremos a seguinte estrutura:

![repo manifest](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/18_repo%20manifest.png)

O conteúdo dos arquivos yaml serão respectivamente o seguinte:

- `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
  labels:
    app: projeto-4
spec:
  replicas: 1
  selector:
    matchLabels:
      app: projeto-4
  template:
    metadata:
      labels:
        app: projeto-4
    spec:
      containers:
        - name: projeto-4
          image: vitorialeda/projeto-4:latest
          ports:
            - containerPort: 80
```

> mude conforme sua realidade

- `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: projeto-4-service
  labels:
    app: projeto-4
spec:
  type: ClusterIP
  selector:
    app: projeto-4
  ports:
    - port: 80
      targetPort: 80
```

Referências:

- [Doc Kubernetes - Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Doc Kubernetes - Service](https://kubernetes.io/docs/concepts/services-networking/service/)

Commite e dê push no repositório remoto. Vamos para o ArgoCD agora :))

## Criando a aplicação no ArgoCD

[Tutorial de como instalar, e acessar o ArgoCD](https://github.com/vitorialeda/Projeto-3-Kubernetes?tab=readme-ov-file#4-instalando-o-argocd)

Clique em `Create Application`, dê um nome ao seu aplicativo, selecione o projeto e hablite a sincronização automatica
![create app](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/19_create%20app.png)

Copie o link do repositório hello-manifest na aba `Code` no GitHub, e cole na área `Repository URL` e em `path` digite `k8s` presentes na seção `Source`
![source](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/20_source.png)

Preencha a seção `Destination` da seguinte forma:
![Destination](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/21_Destination.png)

e clique em `Create app` no canto superior esquerdo, e nossa aplicação será iniciada:
![aplicacao rodando](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/22_aplicacao%20rodando.png)

## Testando a aplicação

Digite em seu terminal:

```bash
kubectl get services -n hello-app
```

Você deverá ver algo como:

```bash
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
projeto-4-service   ClusterIP   10.43.83.145   <none>        80/TCP    6m16s
```

Faça um Port-forward:

```bash
kubectl port-forward svc/projeto-4-service -n hello-app 3000:80
```

Acesse no se navegador:
`localhost:30000`

Você deverá ver algo assim:

![visualizando no navegador](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/23_visualizando%20no%20navegador.png)

## Criando e testando uma nova versão da aplicação:

No arquivo `main.py` mude algo como:

![Modificando app](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/24_Modificando%20app.png)

Commite e dê push para o repositório remoto e relacione à uma nova tag:

```bash
git add .
git commit -m "modificando aplicacao"
git push

git tag v1.0
git push origin v1.0
```

Criação de uma nova tag a partir do Github Actions
![nova tag](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/25_nova%20tag.png)

Novo pull request criado no repositório de manifestos
![novo pull request](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/26_novo%20pull%20request.png)

> Faça o merge desse pull request para que o ArgoCD capte a mudança

Clique em `Refresh` para atualizar o ArgoCD
![Refresh argo](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/27_refresh%20argo.png)

Agora a imagem foi de latest para v1.0
![checando imagem](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/28_checando%20imagem.png)

O conteúdo da resposta também foi atualizado:

![att conteudo](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/29_att%20conteudo.png)

Pods rodando:

![pods rodando](https://github.com/vitorialeda/hello-app/blob/main/doc/imgs/30_pods%20rodando.png)

E é isso! Obrigada por acompanhar até aqui!
