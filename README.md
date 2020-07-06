TESTE LOAD BALANCE DOCKER-APACHE-NGINX


## Constroi duas imagens de servidores Web Apache

 docker build -t s1 -f Dockerfile.1 .
 docker build -t s2 -f Dockerfile.2 .

## Controi a imagem de um servidor Nginx como loadbalance

 docker build -t lb -f Dockerfile.lb .


## Inicia os três servidores

 docker run -tid --name s1_1 -p 81:80 s1 
 docker run -tid --name s2_1 -p 82:80 s2
 docker run -ti -p 83:80 --name lb_1 --link s1_1 --link s2_1 lb


## Rotina de teste

Em uma janela de terminal a parte rode o seguinte comando que simulará 10000 consultas ao seu loadbalance (os nomes dos servidores deverão se alternar). Isso é como se 10000 visitantes estivessem acessando sua aplicação e o Loadbalance está distribuindo os acessos entre os seus dois servidores.

 for (( i=0 ; i<10000 ; i++ )) ; do echo -n $i " - " ; curl http://localhost:83/teste.txt ; done


Enquanto os "visitantes" estão navegando no seu site vamos simular a falha de um deles.

 docker stop s1_1 ; sleep 10s ; docker start s1_1 ; sleep 10s ; docker stop s2_1 ; sleep 10s ; docker start s2_1


## Referência

https://hub.docker.com/_/httpd
https://hub.docker.com/_/nginx/
http://nginx.org/en/docs/http/load_balancing.html

