# Traefik e Kubernetes com o Elasticsearch Stack

Este repositório contem as configurações dos componentes:

 - Elastic
 - Kibana
 - Filebeat
 - HttpBin

Quando implantado como um controlador de entrada do Kubernetes, o Traefik pode processar e rotear milhares de solicitações sem reclamação. 

Ainda assim, para a equipe de operações, a visibilidade do que está acontecendo nos bastidores é essencial. 

O aplicativo está íntegro? 

Está funcionando conforme o planejado? 

O monitoramento de sistemas distribuídos é um dos princípios básicos do conjunto de práticas conhecido como engenharia de confiabilidade de site (SRE).

Este primeiro de uma série de postagens sobre as técnicas do Traefik e SRE explora como os recursos de registro integrados do Traefik podem ajudar a fornecer a visibilidade necessária.
Quando combinado com o conjunto de projetos de código aberto conhecido como Elastic Stack - incluindo Elasticsearch, Kibana e Filebeat, entre outros.

- o Traefik torna-se parte de um rico conjunto de ferramentas para análise e visualização de log de rede.

# Pré-requisitos

# O kubectl

Ferramenta de linha de comando instalada e configurada para acessar seu cluster. (Se você criou seu cluster usando K3d e as instruções acima, isso já terá sido feito para você.)

Você também precisará do conjunto de arquivos de configuração que acompanham este artigo, que estão disponíveis no GitHub :

git clone https://github.com/efcunha/ELK-K8s.git

Configurar Elastic Logging

O Traefik gera logs de acesso automaticamente, mas na forma bruta eles são de utilidade limitada. 
Este tutorial demonstra como ingerir esses logs no Elasticsearch para pesquisa e agregação, o que, por sua vez, permitirá que você crie visualizações usando gráficos Kibana.

Isso significa que você deve primeiro implantar Elasticsearch, Kibana e Filebeat em seu cluster, o que pode ser feito usando seus respectivos comandos Helm.

# Implantar Elasticsearch

Adicione e atualize o repositório de gráficos Elasticsearch Helm usando os seguintes comandos:
```sh
helm repo add elastic https://helm.elastic.co
```
```sh
helm repo update
```
```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "traefik" chart repository
...Successfully got an update from the "elastic" chart repository
Update Complete. ⎈Happy Helming!⎈
```
Elasticsearch requer um volume para armazenar logs. 

Vamos antes criar um Secrets para conexão entre Elasticsearch / Kibana / Filebeat com usuario e senha.

```sh
kubectl apply -f elastic-credentials.yaml
```
A configuração padrão do Helm especifica um volume de 30 GiB usando standard como storageClassName. 
Infelizmente, embora o standardStorageClass esteja disponível no Google Cloud Platform, ele não está disponível no K3s por padrão. 
Para encontrar uma alternativa, faça uma pesquisa para determinar qual StorageClass está disponível:
```sh
kubectl get storageClass
```
```
NAME                                               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  97m
```
A configuração a seguir define Elasticsearch para usar o local-path StorageClass com os seguintes atributos:
Caso vc tenha um StorageClass implantado no seu cluster substitua pelo seu storageClassName.

```
# Request smaller persistent volumes.
volumeClaimTemplate:
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: "local-path" # Adicionar seu storageClassName
  resources:
    requests:
      storage: 30G
```
Implante o Elasticsearch com a configuração acima usando o Helm:
```sh
helm install elasticsearch elastic/elasticsearch -f ./elastic-values.yaml
```
```
NAME: elasticsearch
LAST DEPLOYED: Sun Jan 10 12:23:30 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Watch all cluster members come up.
  $ kubectl get pods --namespace=default -l app=elasticsearch-master -w
2. Test cluster health using Helm test.
  $ helm test elasticsearch
```
Observe que pode levar vários minutos para que os pods do Elasticsearch fiquem disponíveis, então seja paciente.

# Implantar Kibana

O repositório Elastic também fornece gráficos Helm para Kibana. Assim como no Elasticsearch, você deseja configurar Kibana com os seguintes valores:

Implante o Kibana com a configuração acima usando o Helm:
```sh
helm install kibana elastic/kibana -f ./kibana-values.yaml
```
```
NAME: kibana
LAST DEPLOYED: Sun Jan 10 14:50:50 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
Depois que todos os pods estiverem funcionando, antes de acessar o painel do Elastic, você precisará implantar um IngressRoute para expô-lo em seu cluster:
```sh
kubectl apply -f kibana-ingress.yaml
```
```
ingressroute.traefik.containo.us/kibana created
```
Para testar a configuração, tente acessar o painel com seu navegador da web em kibana.localhost:

<img width="1000" alt="kibana-1" src="https://user-images.githubusercontent.com/52961166/116609713-b5335100-a902-11eb-9b72-bf34ab7a917e.png">

Implementar Filebeat

Em seguida, implante o Filebeat como um DaemonSet para encaminhar todos os logs para o Elasticsearch. 
Assim como com os outros componentes, você configurará o Filebeat com os seguintes valores:

Implante o Filebeat com as opções de configuração acima usando o Helm:
```sh
helm install filebeat elastic/filebeat -f ./filebeat-values.yaml
```
```
NAME: filebeat
LAST DEPLOYED: Sun Jan 10 16:23:55 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Watch all containers come up.
  $ kubectl get pods --namespace=default -l app=filebeat-filebeat -w
```
# Aplicativo de demonstração

Finalmente, agora que os componentes do Elastic Stack estão instalados em seu cluster, você precisará de um aplicativo para monitorar. 
O serviço HttpBin fornece muitos terminais que você pode usar para gerar vários tipos de tráfego, o que pode ser útil para gerar visualizações. 

Você pode implantar o serviço e o IngressRoute apropriado usando um único arquivo de configuração:
```sh
kubectl apply -f httpbin.yaml
```
```
deployment.apps/httpbin created
service/httpbin created
ingressroute.traefik.containo.us/httpbin created
```
Depois que os pods são criados, você pode acessar o aplicativo com seu navegador em httpbin.localhost e tentar algumas solicitações:

<img width="1000" alt="httpbin" src="https://user-images.githubusercontent.com/52961166/116609821-d431e300-a902-11eb-8a30-e36afd52a933.png">

# Conecte Traefik e Kibana

Agora é hora de vincular Traefik e Kibana para que você possa interpretar os registros do Traefik de uma forma significativa. 

Nessas próximas etapas, você configurará os dois aplicativos para extrair as informações que deseja do Traefik e prepará-las para visualizar como gráficos Kibana.

Configurar registros de acesso do Traefik

Os registros de acesso do Traefik contêm informações detalhadas sobre cada solicitação que ele trata. 
Por padrão, esses logs não estão habilitados. 
Quando eles estão habilitados, o Traefik grava os logs stdoutpor padrão, o que mistura os logs de acesso com os logs de aplicativos gerados pelo Traefik.

Para resolver esse problema, você deve atualizar a implantação para gerar logs /data/access.loge garantir que eles sejam gravados no formato JSON. 

Volte ao projeto https://github.com/efcunha/Traefik-v2.2.git

E altere o arquivo "daemon-set.yaml"

Esta é a aparência da configuração dos args do daemon-set.yaml:

```sh
args:
# Enable the dashboard without requiring a password. Not recommended
# for production.
- --api.insecure
- --api.dashboard=true
- --ping=true
# Specify that we want to use Traefik as an Ingress Controller.
- --providers.kubernetesingress
# Define two entrypoint ports, and setup a redirect from HTTP to HTTPS.
#- --entrypoints.traefik.address=:8080
- --entryPoints.web.address=:80
- --entryPoints.websecure.address=:443
#- --entrypoints.web.http.redirections.entryPoint.to=websecure
#- --entrypoints.web.http.redirections.entryPoint.scheme=https
# Enable debug logging. Useful to work out why something might not be
# working. Fetch logs of the pod.
# Let's Encrypt Configurtion.
- --certificatesresolvers.default.acme.email=<Seu E-mail>
- --certificatesresolvers.default.acme.storage=acme.json
- --certificatesresolvers.default.acme.tlschallenge
# Use the staging ACME server. Uncomment this while testing to prevent
# hitting rate limits in production.
# Habilitar a linha abaixo somente quando for gerar certificado para produção.
#- --certificatesresolvers.default.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
#- --certificatesresolvers.default.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
- --accesslog=true
- --log=true
- --accesslog.format=json
- --accesslog.filepath=/data/access.log
- --metrics=true
- --metrics.prometheus=true
#- --metrics.prometheus.entryPoint="web-secure"    
```

Corrija a implantação do Traefik para fazer todas as alterações acima usando o arquivo de configuração fornecido:
```sh
kubectl apply -f daemon-set.yaml
```
```
deployment.apps/traefik patched
```

# Painel Kibana

Em seguida, para começar a construir seu painel em Kibana, você precisará configurar os padrões de índice. 

Você pode fazer isso com as seguintes etapas.

Primeiro, abra o menu com três linhas no canto superior esquerdo da tela e escolha:

Kibana > Overview. 

Na página Visão geral do Kibana:

Selecione "Add your data" e clique em "Create index pattern":

<img width="1000" alt="kibana-create-index" src="https://user-images.githubusercontent.com/52961166/116609901-ead83a00-a902-11eb-9621-19a35d462bf7.png">

Defina o padrão de índice nomeado "filebeat-**" para corresponder aos filebeat "indexes":

<img width="1000" alt="kibana-define-index" src="https://user-images.githubusercontent.com/52961166/116611264-29222900-a904-11eb-85a7-10b1fc6ffc40.png">

Clique em "Next step" e selecione "@timestamp" como o campo de hora principal no menu suspenso:

<img width="1000" alt="kibana-define-index-timestamp" src="https://user-images.githubusercontent.com/52961166/116611423-58d13100-a904-11eb-937b-a6ac2372aec5.png">

Ao clicar em "Create index pattern", a página de resumo do índice mostrará os campos atualizados. 

Você poderá usar estes campos em consultas do Kibana na página do painel:

<img width="1000" alt="kibana-index-summary" src="https://user-images.githubusercontent.com/52961166/116611475-671f4d00-a904-11eb-838b-bf66a30ce3fe.png">

Agora, se você clicar no menu com três linhas no canto superior esquerdo da tela e escolher:

Kibana > Discover 

Deverá ver um gráfico preliminar de todos os logs ingeridos.

Para restringi-los apenas aos úteis, escolha "Add filter", entre kubernetes.pod.name no menu suspenso Campo, escolha "is" no menu suspenso Operador e selecione o traefik nome do pod apropriado no menu suspenso "Value" para veja apenas as entradas de registro criadas por ele:

<img width="1000" alt="kibana-logs" src="https://user-images.githubusercontent.com/52961166/116611523-7605ff80-a904-11eb-99af-09bd7384f4c8.png">

Nesse estágio, no entanto, se você expandir qualquer campo de mensagem determinado (clicando na seta à esquerda de seu carimbo de data/hora), verá que a entrada de registro JSON é armazenada como um único item message, o que não serve ao propósito de analisar logs do Traefik.

Para corrigir isso, você precisará fazer com que o Filebeat ensira a mensagem completa como campos JSON separados. 
Há muitas maneiras de fazer isso, mas uma delas é atualizar o plug-in Filebeat para usar o decode-json processador, da seguinte forma:
```sh
# filebeat-chain-values.yaml
- decode_json_fields:
    fields: ["message"]
    process_array: false
    max_depth: 1
    target: ""
    overwrite_keys: false
```
Você pode atualizar a cadeia do processador com as opções de configuração acima usando helm upgrade o arquivo de configuração fornecido:
```sh
helm upgrade filebeat elastic/filebeat -f ./filebeat-chain-values.yaml
```
```
Release "filebeat" has been upgraded. Happy Helming!
NAME: filebeat
LAST DEPLOYED: Sun Jan 10 18:04:54 2021
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
1. Watch all containers come up.
  $ kubectl get pods --namespace=default -l app=filebeat-filebeat -w
```

Agora, os logs no Kibana projetarão cada campo JSON como um campo de consulta separado. 
Mas ainda há um problema! 
Você notará triângulos amarelos ao lado dos campos e, ao passar o cursor sobre eles, verá uma mensagem de aviso de que "No Cache mapping exists for the field":

<img width="1000" alt="kibana-traefik-fields" src="https://user-images.githubusercontent.com/52961166/116611598-8f0eb080-a904-11eb-9992-068a58e7d268.png">

Para corrigir isso, volte para filebeat-** a "Index summary" da pagina

Management > Stack Management > Kibana > Index Patterns.

No canto superior esquerdo, ao lado do ícone de lata de lixo vermelho, há um ícone de atualização. 

Clique nele para atualizar "Click it to refresh" a lista de campos.

<img width="1000" alt="kibana-refresh-index" src="https://user-images.githubusercontent.com/52961166/116611654-a057bd00-a904-11eb-8c85-4d0c8ad8cce6.png">

Como você verá quando retornar à 

Kibana > Discover 

Agora todos os campos de registro gerados pelo Traefik estão disponíveis em Kibana para consulta.

Simular Carga

Os registros não fazem sentido se não tiverem eventos para registrar, então vá em frente e brinque com o serviço HttpBin que você instalou anteriormente, acessando-o httpbin.localhost para gerar algum tráfego, ou tente executar scripts como estes para acessar o serviço em loops:
```sh
for ((i=1;i<=10;i++)); do curl -s -X GET "http://localhost/get" -H "accept: application/json" -H "host: httpbin.localhost" > /dev/null; done
for ((i=1;i<=10;i++)); do curl -s -X POST "http://localhost/post" -H "accept: application/json" -H "host: httpbin.localhost" > /dev/null; done
for ((i=1;i<=20;i++)); do curl -s -X PATCH "http://localhost/patch" -H "accept: application/json" -H "host: httpbin.localhost" > /dev/null; done
```
Kibana Charts

Agora você pode começar a criar algumas visualizações. Os registros de acesso gerados pelo Traefik contêm um conjunto diversificado de campos. 
Para gerar um gráfico do detalhamento da carga geral da solicitação, navegue até 

Kibana > Visualize

Escolha "Create new visualization" e clique em "Go to lens.".

A partir daí, encontre o campo "RequestPath" nas seleções à esquerda e arraste-o para o quadrado no meio da tela. Escolha "Donut" como estilo de gráfico no menu suspenso e você verá um gráfico parecido com este:

<img width="1000" alt="dashboard-all-requests" src="https://user-images.githubusercontent.com/52961166/116611784-c0877c00-a904-11eb-8c84-5c29dad51a8d.png">

Este gráfico mostra todas as solicitações tratadas pelo Traefik. Se quiser restringi-lo, você pode adicionar filtros. 

Por exemplo, escolha "RouterName", selecione "exists" como sua operadora e clique em "Save". 

Em seguida, escolha "RequestHost", selecione "is" como seu operador, filtre httpbin.localhost no menu suspenso e clique em "Save" novamente. 

Agora seu gráfico será parecido com este:

<img width="1000" alt="dashboard-httpbin" src="https://user-images.githubusercontent.com/52961166/116611935-ef055700-a904-11eb-9d04-1e68d62d2405.png">

O Traefik também gerou tempos médios de duração para todas as solicitações atendidas pelo aplicativo HttpBin. 

Tente arrastar e soltar o campo "Duration" em seu gráfico e selecionar "Bar Graph" como seu tipo de gráfico:

<img width="1000" alt="dashboard-avg-times" src="https://user-images.githubusercontent.com/52961166/116613158-596ac700-a906-11eb-8f51-52b43b4948d7.png">

# Resumo:

Este exemplo simples serve para demonstrar como os recursos de registro abrangentes do Traefik, combinados com o Elastic Stack de código aberto, podem ser uma ferramenta poderosa para visualizar e compreender a integridade e o desempenho dos serviços em execução nos clusters do Kubernetes. 

Muitos mais gráficos são possíveis do que os mostrados aqui, então mergulhe e explore.

E se você quiser se aprofundar em como suas instâncias do Traefik estão operando, verifique o Traefik Pilot , nova plataforma de monitoramento e gerenciamento SaaS.

Compartilhe
