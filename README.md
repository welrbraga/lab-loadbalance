TESTE LOAD BALANCE DOCKER-APACHE-NGINX

Clone este repositório em sua máquina e execute os passos abaixo

## Constroi duas imagens de servidores Web Apache

Estes servidores WEB são identicos em seu conteúdo e não tiveram qualquer modificação na sua configuração. Durante a construção foi adicionado um arquivo texto dentro do DocumentRoot que será usado nos testes de requisição.

 docker build -t s1 -f Dockerfile.1 .
 docker build -t s2 -f Dockerfile.2 .

## Controi a imagem de um servidor Nginx como loadbalance

Este servidor Nginx teve sua configuração padrão (/etc/nginx/nginx.conf) modificado para servir como laodbalance ao inves um "servidor de páginas". Consulte o arquivo nginx.conf neste repositório para entender a configuração.

 docker build -t lb -f Dockerfile.lb .

## Criando a rede interna

Esta é rede de comunicação entre o servidor Loadbalance e os servidores web.

Originalemnte o Docker usa a rede padrão "bridge" para comunicação
entre containeres que tenha especificado o enace entre si com o parâmetro "--link", no entanto este modo não é mais recomendado devendo então ser criada uma rede interna para comunicação entre containers.

 docker network create load-balance

## Inicia os três servidores

 docker run -tid --name s1_1 --network load-balance -p 81:80 s1 
 docker run -tid --name s2_1 --network load-balance -p 82:80 s2
 docker run -tid --name lb_1 --network load-balance -p 83:80 lb


## Rotina de teste

Em uma janela de terminal a parte rode o seguinte comando que executará 10000 consultas ao seu loadbalance (os nomes dos servidores deverão se alternar). Isso é como se 10000 visitantes estivessem acessando sua aplicação.

Como os acessos são feitos pelo Loadbalance, que está distribuindo as requisições entre os servidores, o resultado deverá ser os nomes dos servidores 1 e 2 se alterando na tela.

 for (( i=0 ; i<10000 ; i++ )) ; do echo -n $i " - " ; curl http://localhost:83/teste.txt ; done

Enquanto os "visitantes" estão navegando no seu site vamos simular a falha de cada um dos servidores web.

 docker stop s1_1 ; sleep 10s ; docker start s1_1 ; sleep 10s ; docker stop s2_1 ; sleep 10s ; docker start s2_1

## Parando os servidores

 docker stop s1_1
 docker stop s2_1
 docker stop lb_1

## Referência

https://hub.docker.com/_/httpd
https://hub.docker.com/_/nginx/
http://nginx.org/en/docs/http/load_balancing.html

