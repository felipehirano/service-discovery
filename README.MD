## Projeto criado para estudo de service discovery utilizando o Consul (HashiCorp)

### Utilizando docker

- Criei um arquivo docker-compose.yml para subir os services;
- Para subir o container rode o comando `'docker-compose up -d '`;
- Para ver se está tudo ok rode o comando `'docker-compose ps'`;
- Para entrar no container utilize o comando `'docker exec -it NAME_CONTAINER sh'`

### Utilizando o consul

- Todas as vezes que subirmos o consul, precisamos indicar se ele vai subir em modo Client, Server ou Dev;
- Rode o comando `'consul agent -dev'` para subir o consul em modo dev;
- Abra outro terminal, entre no mesmo container utilizando o comando `'docker exec -it NAME_CONTAINER sh'`;
- Rode o comando `'consul members'` para listar os nodes que estão de pé;
- Rode o comando `'curl localhost:8500/v1/catalog/nodes'` para acessar a API rest do consul via http;
- O consul possui um servidor de DNS inbutido, que permite buscar informacoes do consul utilizando requisicoes
  http, ou então via DNS;

### Trabalhando com DNS

- A porta default DNS do consul é a 8600;
- Utilize o comando `'apk -U add bind-tools'` para instalar o dig;
- Utilize o comando `'dig @localhost -p 8600'` para fazer a consulta no servidor DNS;
- Utilize o comando `'dig @localhost -p 8600 NAME_CONTAINER.node.consul'` para a consulta dos detalhes de um único node;

### Trabalhando com mais de 1 node

- Alterei o docker-compose.yaml replicando os nodes para consulserver01, consulserver02 e consulserver03;
- Rode o comando `'docker exec -it NAME_CONTAINER sh'` para entrar em um container;
- Rode o comando `'ifconfig'` e peg o inet addr;
- Rode o comando `'mkdir /etc/consul.d'`;
- Rode o comando `consul agent -server -bootstrap-expect=QTD_NODES -node=NAME_CONTAINER -bind=INET_ADDR -data-dir=LOCAL_DATA -config-dir=CONFIG_DIR`
- QTD_NODES = quantidade de servers que iremos subir e
- INET_ADDR é igual o ip do container;
- LOCAL_DATA é a pasta onde o consul irá guardar seus arquivos;
- CONFIG_DIR é o local onde está os arquivos de configuracao do consul;
- Exemplo: `consul agent -server -bootstrap-expect=3 -node=consulserver01 -bind=172.25.0.2 -data-dir=/var/lib/consul -config-dir=/etc/consul.d`;
- Repita o mesmo processo para os outros nodes;
- Rode o comando `'consul join INET_ADDR_OTHER_NODE'` para fazer esses nodes se comunicarem, já que estão rodando em containers separados;
- Rode o comando `'consul members'` para ver os dois nodes se comunicando;

### Trabalhando com o client

- Alterei o docker-compose.yaml para subir o client;
- `'docker-compose down'` para parar os containers;
- `'docker-compose up -d'` para subir o container novamente;
- Entre no container do client utilizando o comando `'docker exec -it NAME_CONTAINER_CLIENT sh'`;
- Rode o comando `'ifconfig'` e peg o inet addr;
- Rode o comando `consul agent -bind=INET_ADDR -data-dir=/var/lib/consul -config-dir=/etc/consul.d`;
  OBS: Não é necessário passar o parâmetro -client porque se não especifica sendo server, ele ja entende que será um type client
- Nesse momento será disparado o seguinte erro: `failed to sync remote state: error="No known Consul servers"` o que indica que não existem servers para ele sincronizar, para resolver isso, basta fazer o join com os outros nodes servers;

### Registro de servicos

- Criei um arquivo chamado services.json dentro de /clients/consul01;
- Dentro do client rode o comando `'consul reload'` para que o servico seja registrado dentro dele;
- Rode o comando `'apk -U add bind-tools'` para instalar o dig;
- Rode o comando `'dig @localhost -p 8600 nginx.service.consul';`
- Com isso, conseguimos ver que o servico já esta registrado, e para verificar, basta utilizar o comando da linha acima dentro de um container de outro server para ver o servico rodando;
- Para fazer uma busca no catalogo de servicos, rode o comando `'curl localhost:8500/v1/catalog/services'`;
- Para verificar em qual node está registrado o servico, rode o comando `consul catalog nodes -service NAME_SERVICE`;
- Rode o comando `'consul catalog nodes -detailed'` para listar todos os detalhes dos nodes dentro do cluster;

### Registrando um segundo servico

- Alterei o docker-compose.yaml adicionando o consulclient02;
- Rodei o comando `'docker-compose up -d'` para subir o consulclient02;
- Criei um arquivo chamado services.json dentro de /clients/consul02;
- Entrei no container do consulclient02;
- Rodei o comando `'mkdir /var/lib/consul'`;
- Rodei o comando `'ifconfig'` para pegar o ip do container;
- Rodei o comando `'consul agent -bind=INET_ADDR -data-dir=/var/lib/consul -config-dir=/etc/consul.d -retry-join=INET_ADDR_SERVER_TO_SYNC'`;
- O parâmetro -retry-join servce para dar o join automático dentro do cluster(posso pegar o ip de qualquer serve que ele irá sincronizar no cluster);
- Em outro terminal, entre dentro do container;
- Rode o comando `'apk -U add bind-tools'` para instalar o dig;
- Rode o comando `'dig @localhost -p 8600 nginx.service.consul'` para verificar os servicos dentro do cluster;

### Health Check

- Adicionei dentro do consul01/services.json o parâmetro check;
- Rodei o comando `'consul reload'`;
- No client estará mostrando o seguinte erro: [WARN] agent: Check is now critical: check=nginx;
- Rode o comando `'curl localhost'` e verá que nada estará retornando;
- E ao listar os servicos utilizando o comando `'dig @localhost -p 8600 nginx.service.consul'` veremos que o servico que está de pé, é apenas o do consul02, pois, nesse momento não temos o nginx rodando;
- Para instalar o nginx rode o comando `'apk add nginx'`;
- Rode o comando `'nginx'` para subir o nginx;
- Verifique o processo do nginx rodando com o comando `'ps'`;
- Rode novamente o comando `'curl localhost'` e verá que agora será retornado um html com error 404;
- Porém, no log do client01 ainda estará retornando [WARN] agent: Check is now critical: check=nginx;
- Rode o comando `'apk add vim'`;
- Rode o comando `'mkdir /usr/share/nginx/html -p'`;
- Rode o comando `'vim /etc/nginx/http.d/default.conf'`
- Altere o arquivo default.conf do nginx para não dar 404 por qualquer coisa;
- No lugar do location / { return 404}; Coloque root /usr/share/nginx/html;
- :wq para salvar e sair no vim;
- Rode o comando `'vim /usr/share/nginx/html/index.html'` e adicione qualquer coisa para mostrar;
- Rode o comando `'nginx -s reload'` para atualizar as confs do nginx;
- Rode novamente o comando `'curl localhost'` e verá que agora será retornado uma mensagem de que está tudo funcionando;

### Otimizando subida de servers

- Criei uma pasta chamada /servers para adicionar os arquivos de subida do servidores;
- São esses arquivos .json que receberão todos os parâmetros para que o server suba;
- Alterei o docker-compose.yaml e adicionei os volumes nos servers;
- Basta rodar o comando `'consul agent -config-dir=/etc/consul.d'` que o servidor irá subir com todas aquelas configs;

### Criptografia

- Adicionei nos arquivos de configuracoes do consul servers o parâmentro encrypt;
- Para gerar a chave que o encrypt espera, rode o comando `'consul keygen'` dentro de um node;
- Essa chave deverá ser o mesmo para todos os nodes;
- Rode o comando `'rm -rf /tmp/*'` para apagar todos os dados antigos dos exemplos anteriores;

### User interface

- Adicionei os parâmetros client_addr e ui_config dentro do server.json;
- A UI do consul roda na porta 8500, então alterei o docker-compose.yaml compartilhando essa porta;
- Para acessar, basta acessar localhost:8500/ui;
