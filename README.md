# TESTE LOAD BALANCE DOCKER-APACHE-NGINX

Clone este repositório em sua máquina e execute os passos abaixo para simular um ambiente web com dois servidores de conteúdo e um balanceador de carga.

```
 git clone https://github.com/welrbraga/lab-loadbalance.git
 cd lab-loadbalance
```

## Construção das duas imagens de servidores Web Apache

Estes servidores WEB são idênticos em seu conteúdo e não tiveram qualquer modificação na sua configuração. Durante a construção foi adicionado um arquivo texto dentro do DocumentRoot que será usado nos testes de requisição.

```
 docker build -t s1 -f Dockerfile.1 .
 docker build -t s2 -f Dockerfile.2 .
```

Neste ponto é importante deixa algumas notas:
1 - Os arquivos textos mencionandos são propositalmente diferentes em cada nós para podermos ver a transição de nós quando testarmos falhas, no entanto em um ambiente real eles seriam substituídos pelos arquivos da sua página/aplicação web;

2 - Eu poderia ter criado apenas um Dockerfile para construir ambas as imagens, e então jogar o mesmo arquivo texto, a ser provido pelo Apache, em todos os nós, mas eu optei pela simplicidade do processo para que aspirantes a usuários de Docker se sintam mais confortáveis ao ler o conteúdo destes dockerfiles e vê-los da forma mais básica possível;

3 - Também poderia ter usado o mesmo processo para criar 3, 10, 100 ou mil imagens da mesma forma, o que aumentará a disponibilidade do meu serviço WEB;

4 - Outra opção (esta talvez seja a mais profissional) seria criar apenas uma única imagem e usá-la como base tantos containeres quanto fossem necessários, mas neste caso usariamos um sistema de arquivos compartilhado (NFS, AWS EFS etc) para hospedar o mesmo DocumentRoot do Apache.

## Construção da imagem de um servidor Nginx como loadbalancer

Este servidor Nginx teve sua configuração padrão (/etc/nginx/nginx.conf) modificada para servir como loadbalancer ao invés um "servidor de páginas". Consulte o arquivo nginx.conf neste repositório para entender a configuração que está bem simples.

```
 docker build -t lb -f Dockerfile.lb .
```

Aqui estamos trabalhando de forma bem rudimentar em modo HTTP, mas seria exatamente aqui que colocariamos o nosso certificado SSL, caso fosse necessário operar em HTTPS.

Se você criar mais do que os dois containeres WEB, edite o arquivo nginx.conf e adicione-os aqui. 

## Criando a rede interna

Originalmente o Docker usa a rede padrão "bridge" para comunicação entre containeres que tenham especificado o enlace entre si com o parâmetro "--link", no entanto este modo não é mais recomendado devendo então ser criada uma rede interna para comunicação entre containers.

```
 docker network create load-balance
```

Eu nomeei a rede como "load-balance", mas o nome não importa, poderia ser qualquer outro termo, palavra ou sigla, desde que eu altere este nome também quando for subir os meus conteineres no próximo passo.

## Iniciando os três servidores

É aqui que a brincadeira começa. Você vai subir os três containeres, respectivamente nas portas 81, 82 e 83. Se você criou mais do que 3 imagens deverá adicionar todas elas a esta rede.

```
 #Servidor WEB 1 (porta 81)
 docker run -tid --name s1_1 --network load-balance -p 81:80 s1
 
 #Servidor WEB 2 (porta 82)
 docker run -tid --name s2_1 --network load-balance -p 82:80 s2
 
 #Balanceador de carga (porta 83)
 docker run -tid --name lb_1 --network load-balance -p 83:80 lb
```
Depois que executar os três comandos acima, você terá todos os containeres rodando em sua máquina. É importante destacara aqui que todos os containeres responderão as consultas no endereço 127.0.0.1 (ou localhost) já que estão todas em sua máquina. A distinção entre cada container se dará pela porta TCP onde faremos a requisição.

## Teste rápido

Agora com tudo no ar podemos fazer uns testes para vermos a estrutura funcionando.

### Teste do servidor 1
Abra seu navegador WEB e aponte para http://localhost:81/teste.txt

Você deverá ver apenas a mensagem "SERVER 1" que é o conteúdo do arquivo teste.txt que colocamos no DocumentRoot do Apache. você pode atualizar a página quantas vezes quiser que ele vai estar lá como uma aplicação "web normal".

### Teste do servidor 2
Agora aponte o seu navegador para http://localhost:82/teste.txt

Um pouco diferente do teste anterior, agora você deverá ver a mensagem "SERVER 2". Como eu disse lá no início, estes arquivos foram criados propositalmetne distintos para que possamos ver o load-balance funcionando adiante, mas no "mundo real" eles seriam subistituídos pela sua aplicação que é exatametne igual em todos os nós.

### Teste do Loadbalancer

Por fim, vamos apontar o navegador web para o endereço do nosso loadbalancer em http://localhost:83/teste.txt

Agora você deverá ver "SERVER 1" e "SERVER 2" se alternando a cada atualização da página (tecle F5 várias vezes para verificar). Parabéns seu load-balancer está funcionando muito bem.

## Teste pesado

Você ainda deve estar com uma janela de comandos aberta, onde digitou os comando para iniciar os containeres. Mantenha-na aberta, mas abra também uma segunda janela. Tente arrumar a sua área de trabalho de forma que consiga ver ambas, seja lado a lado ou empilhada ... como quiser.

Agora, em uma das janelas de terminal rode o comando abaixo:

```
 for (( i=0 ; i<10000 ; i++ )) ; do echo -n $i " - " ; curl http://localhost:83/teste.txt ; done
```

Para quem não entende de shell, ao invés de ficarmos teclando F5 dez mil vezes, nós usamos o comando "curl" dentro de um laço para simular todo esse número de acessos ao nosso servidor de loadbalance. Isso poupa nossos dedos e agiliza os testes de acesso.

Observe como "SERVER1" e "SERVER2" se alternam. Isso mostra que o seu load-balancer está distribuindo as conexões entre os dois servidores web, como era de se esperar.

Essa simulação só não é perfeita para dizermos que temos 10000 visitantes acessando a aplicação porque os acessos estão acontecendo em sequencia, e não paralelamente, mas para simplificar nossos testes isso atende e se quiser mudar o script acima para fazer este exercício, fique a vontade.

## Simulando falhas

Enquanto o comando anterior roda na primeira janela de terminal simulando os "visitantes" que navegam em seu site, vamos provocar uma falha tirando um dos servidores web do ar e ver o que acontece.

Na segunda janela de terminal cole ou digite os comandos abaixo.

```
 docker stop s1_1 ; sleep 10s ; docker start s1_1
```

Estes comandos vão parar o servidor 1 e após dez segundos o subirá novamente.

Após teclar ENTER e confirmar o comando, observe na janela anterior o que acontece com as conexões que se alternavam entre "SERVER1" e "SERVER2".
 
Com o "servidor 1" fora do ar, nosso loadbalancer redireciona todos os acesso para o único servidor disponível que é o SERVER 2.

Aguarde que o ambiente normalize os servidores voltem a responder alternadamente e agora vamos fazer o mesmo com o outro servidor:

```
 docker stop s2_1 ; sleep 10s ; docker start s2_1
```

Desta vez tiramos do ar o servidor 2, então todo o tráfego deve estar sendo direcionado ao servidor 1, até que ele volte a ativa.

Nos testes de laboratório talvez você não veja nenhuma mensagem de perda de conexão, mas na prática talvez um ou outro usuário sinta uma indisponibilidade porque a sua conexão foi interrompida no meio de sua sessão.

Provavelmente já passou por essa situação de estar acessando um serviço sem qualquer problema de repente ele fica indisponível. Mas após assegurar que sua conexão está OK e teclar F5 e tudo está normal como se nada tivesse ocorrido. 

Isso sempre vai acontecer, mas o importante é: Apenas um acesso foi prejudicado, talvez "alguns" dependendo do volume de acesso, mas não todos.

## Ao infinito e além

Obviamente quanto maior a sua estrutura de balanceamento (10, 20, 30  servidores WEB pendurados no load-balance) menos clientes vão te xingar na parada de um serviço.

Não vou descrever aqui o processo completo, mas experimente copiar e editar o arquivo "Dockerfile.1" para "Dockerfile.3", adicione este novo servidor na seção upstream do arquivo nginx.conf, reconstrua as imagens s3_1, e lb_1 e refaça os testes. 
 
## Parando os servidores

Você pode usar os comandos abaixo para interromper os containeres a qualquer momento. Com isso você estará liberando recursos da sua máquina.

```
 docker stop s1_1
 docker stop s2_1
 docker stop lb_1
```

## Destruindo o ambiente

Quando a brincadeira estiver chata você pode quebrar tudo e jogar fora (comandos abaixo) :-) Isso vai remover os containeres de sua máquina, mas não se preocupe, caso você se arrependa é só reconstrui-los novamente.

```
 docker rm s1_1
 docker rm s2_1
 docker rm lb_1
```

Se quiser dar uma faxina geral e remover as imagens também use os comandos a seguir:

```
 docker image rm s1
 docker image rm s2
 docker image rm lb
```

## Sempre tem algo mais a entender

Este texto não esgota o assunto. 

### Sessões e dados compartilhados

Eu aqui só mostro como funciona o balanceamento a nível de entrega dos documentos ao usuário. Em uma aplicação real nós temos que lidar com autenticação, sessão, dados compartlihados, cookies etc. Sem isso, caso a sessão do usuário caia e ele retorne em outro servidor, ee deverá refazer o login preencher o cadastro novametne etc e ninguém fica satisfeito com isso.

Para estas questoes é necessário que seu desenvolvedor saiba como modelar a aplicação, usando serviços de cache compartilhado como memcache, redis etc, mas isso está fora do nosso escopo.

### Pilhas de containeres

Nós subimos os containeres individualmente, mas na prática para esta atividade é mais saudável para sua sanidade mental usar alguma ferrametna de gestão de pilhas de cotainers como o Docker Swarm, Docker-Compose, Kubernetes entre outras opções, inclusive algumas especificas de ambiente de cloud como o AWS ECS.

## Resiliência do balanceador de carga

Nós abordamos a resiliência dos nós WEB. Nas nossas simulações nós tiramos um e depois o outro do ar mostrando que o serviço está sempre disponível, mas e se o loadbalancer falhar?

Não fizemos este teste porque o resultado é o que você imagina. Ele não tem redundância e por isso na sua falta todo o ambiente ficará fora do ar.

Quando montar sua estrutura redundante para produção não esqueça de levar isso em consideração. É possível criar uma estrutura de balanceadores de carga redundantes mas isso vai envolver muito mais do que apenas dois ou três containeres, então a minha sugestão para você que usa um ambiente de cloud é usar os serviços de balanceamento de carga oferecidos pelos principais provedores de nuvem, que já contam com alta disponibilidade, assim sua única preocupação será de fato com a redundância dos nós de serviço WEB.

## Referência

(Apache)[https://hub.docker.com/_/httpd]

(NGinx)[https://hub.docker.com/_/nginx/]

(Load Balacing with Nginx)[http://nginx.org/en/docs/http/load_balancing.html]

