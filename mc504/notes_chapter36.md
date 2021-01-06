# Capítulo 36: I/O Devices
## Arquitetura de Sistemas
ALU já evolui bastante e nessa aula vamos verificar sobre a a evolução. Vamos considerar os *buses*; memory bus, I/O bus, peripheral bus. Bus pode ser traduzido como barramento então fica barramento de memória (CPU  e Memória depende do fabricante), barramento de I/O (i = input e o = output) e periféricos.

### Canonical device
Temos a divisão em hardware interface (ou interface) e device internal structure (internals).  
A interface tem registradores de estado, comando e de dados. 
### Canonical Protocol
Um protocolo básico é simples e verifica os estados da máquina, no código abaixo, temos a verificação de um estado e executa uma determinada rotina.

```javascript
While (STATUS == BUSY)  
	; // wait until device is not busy  
Write data to DATA register  
Write command to COMMAND register  //Doing so starts the device and executes the command)  
While (STATUS == BUSY)  
	; // wait until device is done with your request
```
Uma crítica é que tem dois busy-waits, e quando comandamos o dispositivo, não são eficientes, causando lentidão.
**Podemos evitar o custo do polling?** 
Como o OS precisa testar o tempo do dispositivo sme fazer consultas repetidas e diminuir o overhead?

Considere 1 é o número do processo e p é pooling e o disco está na linha de baixo.

![](https://raw.githubusercontent.com/NatSatie/mc504_notas/main/img/chapter36_02.jpg)

![](https://raw.githubusercontent.com/NatSatie/mc504_notas/main/img/chapter36_03.jpg)

> **Chaveamento de contexto**
Quando um processo entra em estado de espera o processador obtem outro processo e passa a executa-lo. Essa troca entre processos é denominada chaveamento de contexto, ou seja, um contexto de execução é substituido por outro.
Um contexto possui diversas informações contidas no descritor de processo, esses dados permitem que um contexto seja medido como i/o bound, memory bound ou cpu bound. Esses termos indicam onde existe um possível gargalo no processo executado, se pegarmos um processo i/o bound a maior parte de suas operações são do tipo i/o
[Fonte](https://computersciencestudies.wordpress.com/2011/04/12/descritor-de-processo-e-chaveamento-de-contexto/)

Usar uma interrupção não é necessariamente melhor que uma i/o programada (PIO - programmed i/o), pois é capaz de atrasar certos dispositivos, o custo de interromper e trocar de contexto pode ser mais maior que seus benefícios.
Se temos muitas interrupções pode levar a **livelock** com trocas de chaveamento que é uma grande quantidade de interrupções causando atrasos.
Uma forma de otimizarmos é coalescing no qual um dispositivo espera um pouco antes de entregar a interrupção da CPU que permite uma fusão.
> It then initiates the I/O, which must copy the data from memory  
to the device explicitly, one word at a time (marked c in the diagram).When the copy is complete, the I/O begins on the disk and the CPU can  finally be used for something else

Dada a imagem abaixo, vamos economizar o tempo de cópia em C.

![](https://raw.githubusercontent.com/NatSatie/mc504_notas/main/img/chapter36_04.jpg)

Quando usamos DMA - direct memory access, podemos abrir mais espaço e permitindo que 2 realize a sua respectiva tarefa.
![](https://raw.githubusercontent.com/NatSatie/mc504_notas/main/img/chapter36_05.jpg)

## Comunicação de hardware com dispositivos

 - instruções privilegiadas (kernel) I/O 
 - memory-mapped i/o

