# {book}: Descomplicando Docker

## \Container

Container é, em português claro, o agrupamento de uma aplicação junto com suas dependências, que compartilham o 
kernel do sistema operacional do host, ou seja, da máquina (virtual ou física) onde está rodando.

Containers são bem similares às máquinas virtuais, porém mais leves e mais integrados ao sistema operacional 
da máquina host, uma vez que, como já dissemos, compartilha o seu kernel, o que proporciona melhor desempenho 
por conta do gerenciamento único dos recursos.

> 
Lembre-se: na máquina virtual você emula um novo sistema operacional dentro do sistema operacional do host. Já no container você emula somente as aplicações e suas dependências tornando-o portátil.


##  Fun fact about containers

Apesar de o termo ter se tornado hype nos últimos anos, durante décadas já utilizávamos containers em sistemas Unix através do comando chroot. Sim, bem mais simplório, é verdade, pois era apenas uma forma de isolar o filesystem, mas já era o começo!

## Copy-On-Write (COW) e Docker

It's a little bit like having a book. You can make notes in that book if you want, but each time you approach the pen to the page, suddenly someone shows up and takes the page and makes a xerox copy and hand it back to you, that's exactly how copy on write works.

Basicamente, significa que um novo recurso, seja ele um bloco no disco ou uma área em memória, só é alocado quando for modificado. 

Docker usa um esquema de camadas, ou layers, e para montar essas camadas são usadas técnicas de Copy-On-Write. Um container é basicamente uma pilha de camadas compostas por N camadas read-only e uma, a superior, read-write.

## Storage drivers

Apesar de um container possuir uma camada de escrita, na maior parte do tempo você não quer escrever dados diretamente nele, por vários motivos, dentre eles a sua natureza volátil. Em situações onde sua aplicação gera dados, você vai preferir usar volumes "atachados" ao container e escrever neles (veremos mais à frente como fazer isso). Porém, em algumas situações é, sim, necessária a escrita local no container, e é aí que o storage driver entra na história. Storage driver é o mecanismo utilizado pela engine do Docker para ditar a forma como esses dados serão manipulados no filesystem do container. A seguir, os principais storage drivers e suas peculiaridades.

### AUFS (Another Union File System)

O primeiro filesystem disponível para o Docker foi o AUFS, um dos mais antigos Copy-On-Write filesystems, e inicialmente teve que passar por algumas modificações a fim de melhorar a estabilidade.

O AUFS funciona no nível de arquivos (não em bloco), e a ideia é ter múltiplos diretórios (camadas) que ele apresenta para o SO como um ponto único de montagem.

Quando você tenta ler um arquivo, a busca é iniciada pela camada superior, até achar o arquivo ou concluir que ele não existe. Para escrever em um arquivo, este precisa primeiro ser copiado para a camada superior (writable) -- e, sim, você adivinhou: escrever em arquivos grandes pode causar certa degradação da performance, já que o arquivo precisaria ser copiado completamente para a primeira camada, mesmo quando uma parte bem pequena vai sofrer alteração.

### Device Mapper

Em se tratando de Docker, o Device Mapper e o AUFS são bem similares: a grande diferença entre eles é que, no Device Mapper, quando você precisa escrever no arquivo, a cópia é feita em nível de blocos, que era um problema lá no AUFS, e com isso você ganha uma granularidade bem maior. Em teoria, o problema que você tinha quando escrevia um arquivo grande desaparece. Por padrão, Device Mapper escreve em arquivos de loopback, o que deixa as coisas mais lentas, mas agora na versão 1.17+ você já pode configurá-lo em modo direct-lvm, que escreve em blocos e, em teoria, resolveria esse problema. É um pouco mais chatinho de configurar, mas é uma solução mais elegante para ambientes em produção.

Além de AUFS e Device Mapper, você também pode usar BRTFS e OverlayFS como storage driver. Por serem tecnologias relativamente jovens, aprecie com moderação.

### OverlayFS e OverlayFS2

A bola da vez. Uma versão melhorada do AUFS, o OverlayFS e sua versão seguinte e oficialmente recomendada pelo Docker, o Overlay2, são ambos other union filesystems, mas dessa vez muito mais eficientes, rápidos e com uma implementação muito mais simples.

Por serem union file systems, também compartilham da ideia de juntar vários diretórios em um único ponto de montagem como nosso amigo AUFS, porém, no caso do OverlayFS, apenas dois diretórios são suportados, o que não acontece no Overlay2, que tem suporte multi-layer. Ambos suportam page caching sharing, ou seja, múltiplos containers acessando o mesmo arquivo dividem a mesma entrada no arquivo de paginação, o que é um uso mais eficiente de memória.

Aquele problema antigo do AUFS de ter de copiar todo o arquivo para a camada de cima para escrever nele ainda persiste, porém no OverlayFS ele só é copiado uma vez e fica lá para que as outras escritas no mesmo arquivo possam acontecer mais rápido, então tem uma pequena vantagem.

Nota-se um consumo excessivo de inodes quando se usa OverlayFS. Esse é um problema resolvido no Overlay2, então sempre que possível busque usá-lo -- até porque, no geral, tem uma performance superior. Lembrando que 
kernel 4.0+ é pré-requisito para usar OverlayFS2.

### BTRFS 

Geração seguinte de union filesystem. Ele é muito mais space-efficient, suporta muitas tecnologias avançadas de storage e já está incluso no mainline do kernel. O BTRFS, diferentemente do AUFS, realiza operações a nível de bloco e usa um esquema de thin provision parecido com o do Device Mapper e suporta copy-on-write snapshots. Você pode inclusive combinar vários devices físicos em um único BTRFS filesystem, algo como um LVM.

O BTRFS é suportado atualmente na versão CE apenas em distribuições debian-like e na versão EE apenas em SLES (Suse Linux Enterprise Server).

IMPORTANTE: alterar o storage drive fará com que qualquer container já criado se torne inacessível ao sistema local. Cuidado!

## Docker Internals

O Docker utiliza algumas features básicas do kernel Linux para seu funcionamento. A seguir temos um diagrama no qual é possível visualizar os módulos e features do kernel de que o Docker faz uso:

![Docker Internals](/images/docker-internals.png)

### Nasmespaces 

Namespaces foram adicionados no kernel Linux na versão 2.6.24 e são eles que permitem o isolamento de processos quando estamos utilizando o Docker. São os responsáveis por fazer com que cada container possua seu próprio environment, ou seja, cada container terá a sua árvore de processos, pontos de montagens, etc., fazendo com que um container não interfira na execução de outro. Vamos saber um pouco mais sobre alguns namespaces utilizados pelo Docker.

#### PID Namespace

O PID namespace permite que cada container tenha seus próprios identificadores de processos. Isso faz com que o container possua um PID para um processo em execução -- e quando você procurar por esse processo na máquina host o encontrará; porém, com outra identificação, ou seja, com outro PID.

#### Net Namespace

O Net Namespace permite que cada container possua sua interface de rede e portas. Para que seja possível a comunicação entre os containers, é necessário criar dois Net Namespaces diferentes, um responsável pela interface do container (normalmente utilizamos o mesmo nome das interfaces convencionais do Linux, por exemplo, a eth0) e outro responsável por uma interface do host, normalmente chamada de veth* (veth + um identificador aleatório). Essas duas interfaces estão linkadas através da bridge Docker0 no host, que permite a comunicação entre os containers através de roteamento de pacotes.

#### Mnt namespace

É evolução do chroot. Com o Mnt Namespace cada container pode ser dono de seu ponto de montagem, bem como de seu sistema de arquivos raiz. Ele garante que um processo rodando em um sistema de arquivos não consiga acessar outro sistema de arquivos montado por outro Mnt Namespace.

#### IPC namespace

Ele provê um SystemV IPC isolado, além de uma fila de mensagens POSIX própria.

#### UTS namespace

Responsável por prover o isolamento de hostname, nome de domínio, versão do SO, etc.

#### User namespace

O mais recente namespace adicionado no kernel Linux, disponível desde a versão 3.8. É o responsável por manter o mapa de identificação de usuários em cada container.

### Cgroups

É o cgroups o responsável por permitir a limitação da utilização de recursos do host pelos containers. Com o cgroups você consegue gerenciar a utilização de CPU, memória, dispositivos, I/O, etc.

### Netfilter

A já conhecida ferramenta iptables faz parte de um módulo chamado netfilter. Para que os containers consigam se comunicar, o Docker constrói diversas regras de roteamento através do iptables; inclusive utiliza o NAT, que veremos mais adiante no livro.

## Importancia do Docker

O Docker é muito bom para os desenvolvedores, pois com ele você tem liberdade para escolher a sua linguagem de programação, seu banco de dados e sua distribuição predileta. Já para os sysadmins é melhor ainda, pois, além da liberdade de escolher a distribuição, não precisamos preparar o servidor com todas as dependências da aplicação. Também não precisamos nos preocupar se a máquina é física ou virtual, pois o Docker suporta ambas.

A empresa como um todo ganha, com a utilização do Docker, maior agilidade no processo de desenvolvimento de aplicações, encurtando o processo de transição entre os ambientes de QA STAGING e PROD, pois é utilizada a mesma imagem. Traz menos custos com hardware por conta do melhor gerenciamento e aproveitamento dos recursos, além do overhead, que é bem menor se comparado com outras soluções, como a virtualização.

Com Docker fica muito mais viável a criação de microservices (microsserviços, a ideia de uma grande aplicação ser quebrada em várias pequenas partes e estas executarem tarefas específicas), um assunto que tem ganhado cada vez mais espaço no mundo da tecnologia e que vamos abordar com mais detalhes no final deste livro.

## Executando Docker

Para que possamos executar um container, utilizamos o parâmetro "run" do subcomando "container" do comando "docker". Simples, não? :D

[Comandos](./dockerComands.md)

