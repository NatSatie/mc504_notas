# Capítulo 37: Hard Disk Drive
## Hard Drive Interface
tem uma interface de uma **coleção de setores de 512-blocos** e podemos interpretar como um array de setores.
Quando esse bloco é escrito, vai completar inteiramente ou não vai, ou seja é atomico.
**torn white:** situação em que 	acontece uma perda de energia e somente uma porção da escrita é completada.
**unwritten contact**: é mais rápido acessar dois blocos próximos um do outro do que dois blocos distantes. Ao acessar blocos em pedaços (blocos armazenados sequencialmente) é o método mais rápido de acesso e muito mais rápido que o método de acesso aleatório.
## Geometria do Hard Drive
## Fases de procura
Lembre-se que o harddrive é um mecanismo mecânico e físico, então tem suas fases .
 1. Aceleração
 2. Coasting: movimento do braço do hard drive em velocidade máxima
 3. Desaceleração
 4. Parando (settling): o head é posicionado no lugar desejado (varia de 0.5ms a 0.2ms)

## Fases de I/O
 1. Procurar
 2. Delay de rotação
 3. Transferência
## Track Skew
## Cache ou Track Buffer
## Write on Cache
**writeback**: consciência de que a operação de escrita foi realizada *e que os dados foram inseridos* na memória. Apesar da velocidade é arriscado, por perda de energia, ou um erro.
**write through**: consciência da operação write já completou *depois do dado* ser inserido no disco.

## Disk Scheduling
Um escalonador de disco decide qual o próximo I/O request .
### SSTF: Shortest Seek Time First
Dedide a ordem de I/O por trilhas e busca requests na trilha mais próxima para completar.
#### Problemas

 1. **Geometria do hard drive não está disponível ao OS**; Solução: aplicar NBF, nearest block first ou primeiro bloco primeiro.
 2. **Starvation**: as requests de trilhas internas ganham prioridade em cima das trilhas externas.
### Elevator ou SCAN ou C-SCAN
O elevador garante que todos os requests sejam atendidos a partir de visitas em todas as trilhas.
 1. Sweep (Varrer): Como diz no nome, percorre todo a trilha do disco. Caso apareça alguma request dessa trilha e já terminou a varredura, a request fica numa fila esperando sua vez na próxima visita.
 2. F-SCAN: F é de freeze, de congelar, nesse caso, a fila é congelada quando realiza-se uma varredura/sweep
 3. C-SCAN: C é de circular, fazer uma varredura de trilhas de dentro para fora e de fora para dentro.
### Como calcular o custo?

### Outros métodos de varredura
> Written with [StackEdit](https://stackedit.io/).
