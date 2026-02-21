
# Guia Completo: Como rodar seu app Python no Kubernetes com Kind e Ingress

## Conceitos Básicos

- **Kind**: Ferramenta para criar clusters Kubernetes locais usando containers Docker. Permite simular um ambiente Kubernetes real no seu computador.
- **kubectl**: Ferramenta de linha de comando para interagir com o Kubernetes (criar, listar, deletar recursos, etc).
- **Docker**: Usado para criar imagens do seu app Python, que serão executadas dentro do cluster.
- **Deployment**: Recurso do Kubernetes que garante que seu app esteja sempre rodando, gerenciando réplicas e atualizações.
- **Service**: Expõe seu app para acesso interno (dentro do cluster) ou externo (fora do cluster). Faz o balanceamento de carga entre pods.
- **Ingress**: Permite expor múltiplos serviços HTTP/HTTPS sob o mesmo IP/porta, com regras de roteamento. Precisa de um Ingress Controller (como o NGINX) instalado no cluster.

---

## Ordem Lógica dos Componentes

1. **Configuração do Kind (kind-config.yaml)**: Define como o cluster será criado e quais portas do host serão mapeadas para o cluster.
2. **Criação do cluster Kind**: Inicializa o cluster local com as configurações desejadas.
3. **Instalação do Ingress Controller**: Necessário para que o recurso Ingress funcione.
4. **Criação e carregamento da imagem Docker**: Garante que o app Python estará disponível para o cluster.
5. **Deploy, Service e Ingress**: Deploy cria o pod, Service expõe o pod, Ingress faz o roteamento externo.
6. **Testes e validações**: Garante que tudo está funcionando e acessível.

---

## Passo a Passo Didático

### 1. Instale as ferramentas necessárias

- Docker: https://docs.docker.com/get-docker/
- Kind: https://kind.sigs.k8s.io/docs/user/quick-start/
- kubectl: https://kubernetes.io/docs/tasks/tools/

---

### 2. Crie o arquivo de configuração do Kind (kind-config.yaml)

Esse arquivo define o mapeamento de portas do cluster para o seu computador. Exemplo para expor a porta 80 do cluster na porta 31384 do host:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 31384
```

> **Atenção:** Se quiser expor HTTPS, adicione também o mapeamento da porta 443.

---

### 3. Crie o cluster Kind

```sh
kind create cluster --name primeiro-python --config kind-config.yaml
```
Esse comando cria o cluster local com o nome "primeiro-python" e mapeia a porta 80 do cluster para a 31384 do seu computador.

---

### 4. Instale o NGINX Ingress Controller

O Ingress só funciona se houver um Ingress Controller rodando no cluster. O mais comum é o NGINX:

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/kind/deploy.yaml
```

Depois, adicione o label necessário ao nó:
```sh
kubectl get nodes
kubectl label node <nome-do-no> ingress-ready=true
```
Exemplo:
```sh
kubectl label node primeiro-python-control-plane ingress-ready=true
```

Verifique se o pod ingress-nginx-controller está Running:
```sh
kubectl get pods -n ingress-nginx
```

---

### 5. Construa e carregue a imagem Docker do app

No terminal, navegue até a pasta do seu projeto e rode:
```sh
docker build -t python-app:v1 .
kind load docker-image python-app:v1 --name primeiro-python
```
Assim, a imagem estará disponível para o cluster Kind.

---

### 6. Crie e aplique os manifestos Kubernetes

#### a) k8s/deploy.yaml
Define o pod e a imagem a ser usada:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
  labels:
    app: python-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: python-app
  template:
    metadata:
      labels:
        app: python-app
    spec:
      containers:
      - name: python-app
        image: python-app:v1
        ports:
        - containerPort: 5000
```

#### b) k8s/service.yaml
Expõe o app na porta 8080 do cluster e encaminha para a porta 5000 do pod:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: python-app
spec:
  selector:
    app: python-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 5000
```

#### c) k8s/ingress.yaml
Roteia o tráfego externo para o serviço:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: python-app
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: python-app
            port:
              number: 8080
```

---

### 7. Aplique os manifestos no cluster

```sh
kubectl apply -f k8s/deploy.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```
Esses comandos criam o Deployment, o Service e o Ingress no cluster.

---

### 8. Verifique se o app está rodando

```sh
kubectl get pods
kubectl get svc
kubectl get ing
```
Procure pelo nome "python-app" em cada recurso. O pod deve estar Running.

---

### 9. Teste o acesso externo

No navegador ou via curl:
```sh
curl -v http://localhost:31384/api/v1/info
```
Se tudo estiver correto, você verá o JSON de resposta do seu app.

---

## Atenção às Portas e Possíveis Problemas

- **Porta NodePort (ex: 31384):** O Kind só expõe essa porta para o host se você configurar o kind-config.yaml corretamente. Se tentar acessar uma porta alta e não funcionar, revise o mapeamento.
- **Porta 80:** Só será acessível no host se você mapear explicitamente no kind-config.yaml.
- **Port-forward:** Alternativamente, use `kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80` e acesse http://localhost:8080/.
- **Pod Pending ou CrashLoopBackOff:** Verifique se a imagem foi carregada corretamente e se o nome da imagem no deploy.yaml está igual ao nome local.
- **Ingress não funciona:** Confirme se o Ingress Controller está instalado e rodando, e se o label ingress-ready=true está no nó.
- **Service não encontrado pelo Ingress:** O nome do serviço no ingress.yaml deve ser igual ao do service.yaml.

---

## Resumo dos arquivos a editar/criar

- kind-config.yaml (define o mapeamento de portas do cluster)
- Dockerfile (já existe)
- k8s/deploy.yaml (confirme a imagem)
- k8s/service.yaml (confirme porta e nome)
- k8s/ingress.yaml (confirme nome do serviço e porta)

---

## Dicas Finais

- Sempre siga a ordem: configure o Kind → crie o cluster → instale o Ingress Controller → carregue a imagem → aplique os manifestos.
- Use `kubectl get pods`, `kubectl get svc`, `kubectl get ing` para monitorar o status.
- Se der erro de conexão, revise o mapeamento de portas e o status dos pods.
- Para dúvidas ou problemas, revise os logs com `kubectl logs` e descreva os recursos com `kubectl describe`.

---

Se quiser, posso revisar ou editar os arquivos para você. Basta pedir!
