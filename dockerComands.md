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

