```
docker container run hello-world
```

1. O comando "docker" se comunica com o daemon do Docker informando a ação desejada.
2. O daemon do Docker verifica se a imagem "hello-world" existe em seu host; caso ainda não, o Docker faz o download da imagem diretamente do Docker Hub.
3. O daemon do Docker cria um novo container utilizando a imagem que você acabou de baixar.
4. O daemon do Docker envia a saída para o comando "docker", que imprime a mensagem em seu terminal.

```
docker image ls
```

- REPOSITORY -- O nome da imagem.
- TAG -- A versão da imagem.
- IMAGE ID -- Identificação da imagem.
- CREATED -- Quando ela foi criada.
- SIZE -- Tamanho da imagem.

```
docker container ls -a
```

Com o "docker container ls", você consegue visualizar todos os containers em execução e ainda obter os detalhes sobre eles. 

- CONTAINER ID -- Identificação única do container.
- IMAGE -- A imagem que foi utilizada para a execução do container.
- COMMAND -- O comando em execução.
- CREATED -- Quando ele foi criado.
- STATUS -- O seu status atual.
- PORTS -- A porta do container e do host que esse container utiliza.
- NAMES -- O nome do container.

```
- t
- i
- d
```

- -t -- Disponibiliza um TTY (console) para o nosso container.
- -i -- Mantém o STDIN aberto mesmo que você não esteja conectado no container.
- -d -- Faz com que o container rode como um daemon, ou seja, sem a interatividade que os outros dois parâmetros nos fornecem.

##  Modo interativo

Na maior parte das vezes você vai subir um container a partir de uma imagem que já está pronta, toda ajustadinha. 
Porém, há alguns casos em que você precisa interagir com o seu container -- isso pode acontecer, por exemplo, na hora 
de montar a sua imagem personalizada. Nesse caso, usar o modo interativo é a melhor opção. Para isso, 
basta passar os parâmetros "-ti" ao comando "docker container run".

## Daemonizando o container

Utilizando o parâmetro "-d" do comando "docker container run", é possível daemonizar o container, fazendo com que 
o container seja executado como um processo daemon. Isso é ideal quando nós já possuímos um container que não 
iremos acessar (via shell) para realizar ajustes. Imagine uma imagem já com a sua aplicação e tudo que precisa configurado; 
você irá subir o container e somente irá consumir o serviço entregue por sua aplicação. Se for uma aplicação web, basta 
acessar no browser passando o IP e a porta onde o serviço é disponibilizado no container. Sensacional, não?
Ou seja, se você quer subir um container para ser utilizado como uma máquina Linux convencional com shell e 
que necessita de alguma configuração ou ajuste, utilize o modo interativo, ou seja, os parâmetros "-ti".
Agora, se você já tem o container configurado, com sua aplicação e todas as dependências sanadas, não tem a 
necessidade de usar o modo interativo -- nesse caso utilizamos o parâmetro "-d", ou seja, o container daemonizado. 
Vamos acessar somente os serviços que ele provê, simples assim.

