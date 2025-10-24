🧩 Airbyte EKS Deployment — Autenticação com NGINX Basic Auth

Este repositório contém a configuração de deploy automatizado do Airbyte Self-Managed em um cluster EKS (AWS), com autenticação HTTP via NGINX Basic Auth opcional.

🚀 Estrutura geral

O fluxo de acesso com autenticação habilitada é:

Usuário → ALB (Ingress AWS)
        → NGINX Auth Proxy (Basic Auth)
        → Airbyte Server (Helm Chart)

🧱 Estrutura dos arquivos
cicd/environments/staging/
│
├── values.yaml                        # Configuração principal do Airbyte via Helm
│
└── k8s/
    ├── ingress.yaml                   # Ingress (ALB → nginx-auth-proxy)
    ├── nginx-basic-auth.yaml          # Deployment, ConfigMap e Service do NGINX proxy
    ├── nginx-basic-auth-secret.yaml   # Secret com arquivo .htpasswd (Base64)
    ├── service-account.yaml
    └── gp3-storageclass.yaml


O pipeline GitHub Actions aplica tudo automaticamente, incluindo os secrets e a autenticação.

🔐 Autenticação HTTP (Basic Auth)

A autenticação é implementada por um NGINX proxy reverso, que exige usuário e senha antes de redirecionar o tráfego para o Airbyte.

🧰 Como gerar o primeiro usuário

Gere um arquivo .htpasswd localmente:

htpasswd -c ./auth admin


Ele pedirá uma senha (exemplo: airbyte123) # essa airbyte123 sera a real senha de logar no airbyte com o usuario admin feito assim no htpasswd

Converta o conteúdo para Base64:

cat auth | base64 -w0


Copie o valor gerado e adicione como secret no GitHub:

Vá em Settings → Secrets and variables → Actions

Nome: BASIC_AUTH_FILE_BASE64

Valor: cole o conteúdo Base64

Rode o pipeline normalmente.
O GitHub Actions aplicará o secret e criará o proxy NGINX automaticamente.

➕ Adicionar novos usuários

Para adicionar outro usuário (ex: andre), execute:

htpasswd ./auth andre


Verifique o arquivo:

cat auth


Saída:

admin:$apr1$Xz4w...
andre:$apr1$Tn3b...


Converta novamente o arquivo inteiro para Base64:

cat auth | base64 -w0


Atualize o secret BASIC_AUTH_FILE_BASE64 no GitHub
e rode o pipeline novamente.

Os novos usuários poderão logar imediatamente.

----------------

🧩 Como desativar a autenticação

Caso deseje desativar o Basic Auth e voltar ao acesso direto ao Airbyte:

No arquivo ingress.yaml, mude o backend:

backend:
  service:
    name: airbyte-airbyte-server-svc
    port:
      number: 8001


(em vez de nginx-auth-proxy:8080)

Comente ou remova as etapas abaixo do deploy.yaml:

- name: Apply Basic Auth Secret
- name: Deploy NGINX Auth Proxy


(Opcional) Remova os manifests:

kubectl delete deployment nginx-auth-proxy -n airbyte
kubectl delete svc nginx-auth-proxy -n airbyte
kubectl delete configmap nginx-auth-config -n airbyte
kubectl delete secret airbyte-basic-auth -n airbyte


Rode novamente o pipeline.
O ALB passará a enviar o tráfego direto para o Airbyte, sem autenticação.


---------------

⚙️ Atualizar senha de um usuário existente

Gere novamente o .htpasswd com a mesma conta:

htpasswd ./auth admin


Converta para Base64:

cat auth | base64 -w0


Atualize o Secret BASIC_AUTH_FILE_BASE64 no GitHub
e execute o pipeline.

A senha é atualizada automaticamente — sem precisar reiniciar pods.

✅ Verificação

Após o deploy, acesse:

https://airbyte-staging.data.caju.app


O navegador exibirá um popup de autenticação:

Authentication Required - Airbyte


Digite o usuário e senha definidos no .htpasswd.

🧪 Teste interno (via kubectl)

Você também pode testar dentro do cluster:

kubectl run -i --tty debug --image=curlimages/curl --restart=Never -n airbyte -- sh


E dentro do pod:

curl -u admin:airbyte123 http://nginx-auth-proxy:8080


Se retornar HTML da UI do Airbyte, a autenticação está funcionando.

💡 Observações

O Basic Auth é ideal para ambientes internos, staging e testes.

Para produção, recomenda-se OAuth2 (Google, Okta, Cognito, etc).

O secret .htpasswd é versionado apenas como Base64, não exponha em texto plano.

O NGINX proxy pode ser facilmente substituído por oauth2-proxy se desejar upgrade futuro.

----------

Alguns pontos do airbyte:

# o values.yaml funciona assim

- Blocos
EX: server

se voce colcoar uma variavel aqui, por exemplo:
 env_vars:
    AIRBYTE_URL: "http://airbyte-server.airbyte.svc.cluster.local:8000/"  # Ajuste para Kubernetes
 extraEnv:
    - name: AIRBYTE_MANIFEST_SERVER_API_BASE_PATH
      value: "http://airbyte-connector-builder-server:80"

Só vai valer para os PODs do SERVER

# Agora se colocar no bloco global:

global:
  env_vars:
    STORAGE_BUCKET_AUDIT_LOGGING: "my-bucket/audit-logging" # com essa global sim foi no deploy de worker

aqui vale para TODOS.

-----------------------

a politica do service account tem essas permissoes
no trust relantionship:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<MY-ACCOUNT>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<OIDC-CLUSTER-NUMBER>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-1.amazonaws.com/id/<OIDC-CLUSTER-NUMBER>:sub": "system:serviceaccount:airbyte:airbyte-sa",
                    "oidc.eks.us-east-1.amazonaws.com/id/<OIDC-CLUSTER-NUMBER>:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}

# ABAIXO A POLITICA PARA ACESSO NA BUCKET:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*"
            ]
        }
    ]
}