# TESTE LOAD BALANCE DOCKER-APACHE-NGINX

Clone este repositório em sua máquina e execute os passos abaixo para simular um ambiente web com dois servidores de conteúdo e um balanceador de carga.

## Construção das duas imagens de servidores Web Apache

Estes servidores WEB são idênticos em seu conteúdo e não tiveram qualquer modificação na sua configuração. Durante a construção foi adicionado um arquivo texto dentro do DocumentRoot que será usado nos testes de requisição.

 docker build -t s1 -f Dockerfile.1 .
 docker build -t s2 -f Dockerfile.2 .

## Contrução da imagem de um servidor Nginx como loadbalancer

Este servidor Nginx teve sua configuração padrão (/etc/nginx/nginx.conf) modificada para servir como laodbalancer ao invés um "servidor de páginas". Consulte o arquivo nginx.conf neste repositório para entender a configuração.

 docker build -t lb -f Dockerfile.lb .

## Criando a rede interna

Esta é rede de comunicação entre o servidor loadbalancer e os servidores web.

Originalmente o Docker usa a rede padrão "bridge" para comunicação entre containeres que tenham especificado o enlace entre si com o parâmetro "--link", no entanto este modo não é mais recomendado devendo então ser criada uma rede interna para comunicação entre containers.

 docker network create load-balance

## Inicia os três servidores

É aqui que a brincadeira começa. Você vai subir os três containeres, respectivamente nas portas 81, 82 e 83.

 docker run -tid --name s1_1 --network load-balance -p 81:80 s1 
 docker run -tid --name s2_1 --network load-balance -p 82:80 s2
 docker run -tid --name lb_1 --network load-balance -p 83:80 lb


## Teste rápido

Depois que executar os três comandos acima, para um teste direto e rápido, você pode abrir seu navegador WEB e apontar para http://localhost:81/teste.txt (deverá ver apenas a mensagem "SERVER 1"), depois aponte para http://localhost:82/teste.txt (deverá ver a mensagem "SERVER 2") e por fim se apontar para http://localhost:83/teste.txt, você deverá ver "SERVER 1" e "SERVER 2" se alternando a cada atualização da página.

## Teste de produção

Você já deve estar com uma janela de comandos aberta, onde está digitando os comando do docker. Mantenha-na aberta, e abra também uma segunda janela. Tente arrumar a sua área de trabalho de forma que cosniga ver ambas as janelas de terminal.

Em uma das janelas de terminal rode o comando abaixo. Ele executará 10000 consultas ao seu serviço web a partir do endereço do loadbalancer. Como ocorreu ao acessar pelo navegador, os nomes dos servidores deverão se alternar. Isso é QUASE como se 10000 visitantes estivessem acessando sua aplicação.

Como os acessos são feitos a partir do loadbalancer, que está distribuindo as requisições entre os dois servidores web, o resultado deverá ser os nomes dos servidores 1 e 2 se alterando na tela.

 for (( i=0 ; i<10000 ; i++ )) ; do echo -n $i " - " ; curl http://localhost:83/teste.txt ; done

## Simulando falhas

Enquanto os "visitantes" estão navegando no seu site vamos simular a falha de cada um dos servidores web. 

Com o comando anterior rodando, em uma janela, na outra cole os comandos abaixo. Se você colar todos eles como estão, você terá a parada do servidor s1, depois de 10s ele será iniciado novamente, após mais 10s segundos o servidor s2 é que será desligado e em seguida ligado novamente.

você deverá notar que enquanto um dos servidores estiver fora o tráfego é todo direcionado para o único servidor ativo no momento, evitando assim que o serviço fique indisponível.

 docker stop s1_1 ; sleep 10s ; docker start s1_1 ; sleep 10s ; docker stop s2_1 ; sleep 10s ; docker start s2_1

## Parando os servidores

Você pode usar os comandos abaixo para interromper os containeres a qualquer momento.

 docker stop s1_1
 docker stop s2_1
 docker stop lb_1

## Destruindo o ambiente

Quando a brincadeira estiver chata você pode quebrar tudo e jogar fora (comandos abaixo) :-) Caso você se arrependa é só reconstruir os containeres novamente.

 docker rm s1_1
 docker rm s2_1
 docker rm lb_1

Se quiser dar uma faxina geral e remover as imagens também use os comandos a seguir:

 docker image rm s1
 docker image rm s2
 docker image rm lb

## Referência

(Apache)[https://hub.docker.com/_/httpd]
(NGinx)[https://hub.docker.com/_/nginx/]
(Load Balacing with Nginx)[http://nginx.org/en/docs/http/load_balancing.html]

