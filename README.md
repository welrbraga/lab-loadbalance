# TESTE LOAD BALANCE DOCKER-APACHE-NGINX

Clone este repositório em sua máquina e execute os passos abaixo para simular um ambiente web com dois servidores de conteúdo e um balanceador de carga.

 git clone https://github.com/welrbraga/lab-loadbalance.git
 cd lab-loadbalance

## Construção das duas imagens de servidores Web Apache

Estes servidores WEB são idênticos em seu conteúdo e não tiveram qualquer modificação na sua configuração. Durante a construção foi adicionado um arquivo texto dentro do DocumentRoot que será usado nos testes de requisição.

 docker build -t s1 -f Dockerfile.1 .
 
 docker build -t s2 -f Dockerfile.2 .

Na verdade eu poderia ter criado apenas um Dockerfile para construir ambas as imagens, mas eu optei pela simplicidade para que aspirantes a usuários de Docker se sintam mais confortáveis ao ler o conteúdo destes arquivos.

Também poderia ter criado 3, 10 100 ou mil imagens da mesma forma, o que aumentará a disponibilidade do meu serviço WEB.

## Construção da imagem de um servidor Nginx como loadbalancer

Este servidor Nginx teve sua configuração padrão (/etc/nginx/nginx.conf) modificada para servir como loadbalancer ao invés um "servidor de páginas". Consulte o arquivo nginx.conf neste repositório para entender a configuração.

 docker build -t lb -f Dockerfile.lb .

Aqui estamos trabalhando de forma bem rudimentar em modo HTTP, mas seria exatamente aqui que colocariamos o nosso certificado SSL, caso fosse necessário operar em HTTPS.

Se você criar mais do que os dois containeres WEB, edite o arquivo nginx.conf e adicione-os aqui. 

## Criando a rede interna

Originalmente o Docker usa a rede padrão "bridge" para comunicação entre containeres que tenham especificado o enlace entre si com o parâmetro "--link", no entanto este modo não é mais recomendado devendo então ser criada uma rede interna para comunicação entre containers.

 docker network create load-balance

Eu nomeei a rede como "load-balance", mas o nome não importa, poderia ser qualquer outro termo, palavbra ou sigla, desde que adiante eu eu altere este nome também quando for subir os meus conteineres.

## Inicia os três servidores

É aqui que a brincadeira começa. Você vai subir os três containeres, respectivamente nas portas 81, 82 e 83. Se você criou mais do que 3 imagens deverá adicionar todas elas a esta rede.

 #Servidor WEB 1 (porta 81)
 docker run -tid --name s1_1 --network load-balance -p 81:80 s1
 
 #Servidor WEB 2 (porta 82)
 docker run -tid --name s2_1 --network load-balance -p 82:80 s2
 
 #Balanceador de carga (porta 83)
 docker run -tid --name lb_1 --network load-balance -p 83:80 lb

## Teste rápido

Depois que executar os três comandos acima, para um teste direto e rápido, você pode abrir seu navegador WEB e apontar para http://localhost:81/teste.txt (deverá ver apenas a mensagem "SERVER 1"), depois aponte para http://localhost:82/teste.txt (deverá ver a mensagem "SERVER 2").

Por fim, se apontar para o loadbalancer em http://localhost:83/teste.txt, você deverá ver "SERVER 1" e "SERVER 2" se alternando a cada atualização da página (tecle F5 várias vezes para verificar). Parabéns seu loa-balancer está funcionando.

## Teste pesado

Você já deve estar com uma janela de comandos aberta, onde está digitando os comando do docker. Mantenha-na aberta, e abra também uma segunda janela. Tente arrumar a sua área de trabalho de forma que consiga ver ambas as janelas de terminal.

Agora, em uma das janelas de terminal rode o comando abaixo:

 for (( i=0 ; i<10000 ; i++ )) ; do echo -n $i " - " ; curl http://localhost:83/teste.txt ; done

Isso vai simular 10000 consultas aos seus servidores web a partir do loadbalancer. Observe como "SERVER1" e "SERVER2" se alternam. Isso mostra que o seu load-balancer está distribuindo as conexões entre os dois servidores web, como era de se esperar.

Isso é QUASE como se 10000 visitantes estivessem acessando sua aplicação. Só não é de fato, porque os acessos estão acontecendo em sequencia, e não paralelamente, mas se quiser mudar o script acima para fazer este exercício, fique a vontade.

## Simulando falhas

Enquanto os "visitantes" estão navegando no seu site vamos tirar um servidor do ar e simular a falha de cada um dos servidores web. 

Com o comando anterior rodando, em uma janela de terminal, na outra cole o comandos abaixo e observe na janela anterior o que vai acontecer com as conexões:

 docker stop s1_1 ; sleep 10s ; docker start s1_1

Nós paramos o servidor 1 por 10 segundos. Com ele fora do ar, nosso loadbalancer redireciona todos os acesso para o unico servidor disponível que é o SERVER 2.

Aguarde que o ambiente normalize os servidores voltem a responder alternadamente e agora vamos fazer o mesmo com o outro servidor:

 docker stop s2_1 ; sleep 10s ; docker start s2_1

Desta vez tiramos do ar o servidor 2, então todo o tráfego deve estar sendo direcionado ao servidor 1, até que ele volte a ativa.

Nos testes de laboratório talvez você não veja nenhuma mensagem de perda de conexão, mas na prática talvez um ou outro usuário sinta uma indisponibilidade porque a sua conexão foi interrompida no meio de sua sessão.

Se você acessa muitos sites, provavelmente já passou por isso, xingou o sysadmin, teclou F5 e tudo estava normal. Isso semrpe vai acontecer, mas o importante é: Apenas um acesso foi projedicado, talvez "alguns" dependendo do volume de acesso, mas não todos.

## Escalando pra cima ou para baixo

Obviamente quanto maior a sua estrutura de balanceamento (10, 20, 30  servidores WEB pendurados no load-balance) menos clientes vão te xingar na parada de um serviço.

Não vou descrever aqui o processo completo, mas experimente copiar e editar o arquivo "Dockerfile.1" para "Dockerfile.3", adicione este novo servidor na seção upstream do arquivo nginx.conf, reconstrua as imagens s3_1, e lb_1 e refaça os testes. 
 
## Parando os servidores

Você pode usar os comandos abaixo para interromper os containeres a qualquer momento. Com isso você estará liberando recursos da sua máquina.

 docker stop s1_1
 
 docker stop s2_1
 
 docker stop lb_1

## Destruindo o ambiente

Quando a brincadeira estiver chata você pode quebrar tudo e jogar fora (comandos abaixo) :-) Isso vai remover os containeres de sua máquina, mas não se preocupe, caso você se arrependa é só reconstrui-los novamente.

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

