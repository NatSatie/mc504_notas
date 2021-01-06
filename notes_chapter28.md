# MC504: Capítulo 28: LOCKS
Locks são mutex que são usados para garantir exclusão mútua.
```javascript
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
Pthread_mutex_lock(&lock); // wrapper; exits on failure
balance = balance + 1;
Pthread_mutex_unlock(&lock);
```
Temos algumas abordagens para o lock:

 1. coarse-grained: protege os dados com um lock toda vez que a região crítica é acessada
 2. fine grained: protege dados com locks diferentes permite que locks tranquem mais threads

## Critérios para locks
O que precisamos assegurar em um lock:

 1. Exclusão mútua
 2. Justiça/fairness: garantimos que nenhuma thread morra de fome
 3. Performance

## Proposta 1: Interrupções
Uma das primeiras soluções de exclusão mútua foi desabilitar interrupções em regiões críticas.
```javascript
void lock():
	DisableInterrupts()
void unlock():
	EnableInterrupts()
```
### Considerações
Estamos rodando um único processador. Quando interrompemos as interrupções( precisamos de uma instrução de hardware que faça) e que essa parada seja realizada **antes** de entrar na região crítica. Para retornar, basta fazer uma instrução de harware para retomar a rotina.
### Pontos negativos

 1. Temos o problema grave de confiar tudo isso em uma operação de alta prioridade e temos que rezar para que esse sistema não seja abusado.
 2.  Infelizmente essa mecânica não funciona em multi-processadores
 3. Pior: quais são os erros fatais em ligar e desligar interrupções?
## Proposta 2: Usando Flags
**spoilers: não funciona também**
```javascript
typedef struct __lock_t { int flag; } lock_t;  

void init(lock_t *mutex) // 0 -> lock is available, 1 -> held  
	mutex->flag = 0

void lock(lock_t *mutex) 
	while (mutex->flag == 1) // TEST the flag
		// spin-wait  (do nothing)
	mutex->flag = 1 // now SET it!
 
void unlock(lock_t *mutex) 
	mutex->flag = 0
```
### Pontos negativos

 1. Corretude: Considere a seguinte situação: *Thread 1 chama o lock() e retém o lock (flag = 1) então ele fica no spin-wait. Mas se ocorrer alguma interupção inesperada, e a Thread também é chamada para pear o lock, a flag é 1 então ela fica em spin-wait. Nenhum dos dois devolve o lock*
 2. Perfomance: Para um único processador o spin-wait é um desperdício de tempo, porque a thread fica numa fila que não pode sair
## Proposta 3: Spin Locks & Test and Set
Test and Set retorna o o valor antigo para o *old_ptr* e simultaneamente atualiza o valor para o *new*. Essa sequência de operações acontece atomicamente, ou seja, ela funciona uma única vez para todas as threads.
Seu nome vem de testar o o valor antigo (que foi retornado) e configuramos a memória para o novo valor.

```javascript
int TestAndSet(int *old_ptr, int new) {  
	int old = *old_ptr; // fetch old value at old_ptr  
	*old_ptr = new;   // store ’new’ into old_ptr  
	return old; // return the old value
```
```javascript
typedef struct __lock_t {
	int flag;  
} lock_t;  
 
void init(lock_t *lock) 
	//0: lock is available, 1: lock is held  
	lock->flag = 0;  
void lock(lock_t *lock)
	while (TestAndSet(&lock->flag, 1) == 1)  
		// spin-wait (do nothing)  
void unlock(lock_t *lock)
	lock->flag = 0;  
```
### Caso 1
Considere a seguinte situação:
 1. uma thread chama lock();  Nenhuma outra threas tem direito de segurar esse lock
 2. A thread chama TestAndSet(flag, 1); retorna *flag*, em seu valor antigo que é zero
 3. Se o valor retornado é zero, então não ficamos presos no spin-wait
 4. a thread é automaticamente configurada para 1 em TestAndSet para um novo valor

### Caso 2
Considere a seguinte situação:
 1. Uma thread tem o lock e chama TestAndSet(flag, 1); 
 2. a thread fica no spin-lock até terminar

A parte boa é que o lock é devolvido, mas se a thread nunca terminar, ele nunca vai devolver o lock, então precisamos de um **preemptive scheduler / um escalonador preventivo**

## Proposta 4: Compare and swap
Depois do test and set vamos verifiar o compare and sawp, que verifica se o valor no ponteiro é igual ao esperado.
```javascript
int CompareAndSwap(int *ptr, int expected, int new)  
	int original = *ptr;  
	if (original == expected)  
		*ptr = new;  
	return original;
```
Agora substituímos o lock do TestandSet por esse logo abaixo.
```javascript
void lock(lock_t *lock) 
	while (CompareAndSwap(&lock->flag, 0, 1) == 1)  
		// spin  
```
Como podemos ver, o CompareAndSwap é mais efiente e resolve parte das complicações do TestAndSet. Com ele introduzimos o conceito de **lock-free synchronization**
## Proposta 5: Load-Linked and Store-Conditional
Na arquitetura de hardware, podemos encontrar no MIPS as instruções load-linked e store-conditional que controlam locks e problemas de concorrência.
Load-linked funciona como uma instrução de load, ele busca o value da memória e coloca no registrador. A diferença está em store-conditional, que quando a operação é sucedida só e somente se não há interferência no store no endereço desejado. Caso seja um sucesso, store-conditional retorna 1 e atualiza o *ptr* para *value*, caso contrário retorna 0 e *ptr* não é atualizado. 
```javascript
void lock(lock_t *lock) 
	while (1) 
		while (LoadLinked(&lock->flag) == 1)
			; // spin until it’s zero  
		if (StoreConditional(&lock->flag, 1) == 1)  
			return; // if set-it-to-1 was a success: all done
					// otherwise: try it all over again

void unlock(lock_t *lock) 
	lock->flag = 0;  
```
Alguns anos depois conseguimos uma versão mais concisa:
```javascript
void lock(lock_t *lock) 
	while (LoadLinked(&lock->flag)||!StoreConditional(&lock->flag, 1))  
		; // spin
```
Uma das falhas desse sistema é que ainda tempos a possibilidade de :
>One thread calls  lock() and executes the load-linked, returning 0 as the lock is not held.  Before it can attempt the store-conditional, it is interrupted and another  thread enters the lock code, also executing the load-linked instruction,  and also getting a 0 and continuing. At this point, two threads have  each executed the load-linked and each are about to attempt the storeconditional

## Proposta 6: Fetch and Add
```javascript
int FetchAndAdd(int *ptr) 
	int old = *ptr;  
	*ptr = old + 1;  
	return old;
```
Fetch and Add é uma instrução que atomicamente aumenta o valor enquanto retorna um endereço em particular. 
```javascript
typedef struct __lock_t {  
	int ticket;  
	int turn;  
} lock_t;
```
Para isso precisamos saber como funciona o ticket lock, no qual o ticket transforma uma variavel em uma compinação de locks. A opração funciona desta forma: 
1. Quando a thread quer pegar um lock: chama fetch-and-add para um ticket. Esse valor corresponde ao turno da thread.
2. globalmente o lock->turn é usado para verificar se é o turno da thread (myturn == turn) e assim permite acesso à região crítica.
3. O unlock é adquirido quando incrementamos o turno da próxima thread (se existir)

```javascript
void lock_init(lock_t *lock) 
	lock->ticket = 0;  
	lock->turn = 0;  

void lock(lock_t *lock) 
	int myturn = FetchAndAdd(&lock->ticket);  
	while (lock->turn != myturn)  
		; // spin  
void unlock(lock_t *lock) 
	lock->turn = lock->turn + 1;
```
### Desistir?
Depois de vários usos de spin-wait. O que podemos melhorar?
É um conceito que pode ser chamado de *yield()*, que chama a thread que tem o lock e falar "eu desisto e passo para a próxima thread". Então a própria thread se desencalona. 
## Proposta 7: "just yield, baby"


## Proposta 8: 
Finalmente podemos dar uma parada de threads presas no spin-wait. Agora o escalonador pode fazer **park()** que é por para dormir e **unpark()** que é acordar uma thread a partir de seu ID.
Então ao observamos nosso lock ele tem:

```javascript
typedef struct __lock_t {  
	int flag;  
	int guard;  
	queue_t *q;  
} lock_t;  
 ```

  ```javascript
void lock_init(lock_t *m) {  
	m->flag = 0;  
	m->guard = 0;  
	queue_init(m->q);  

void lock(lock_t *m) {  
	while(TestAndSet(&m->guard, 1) == 1)
		; //acquire guard lock by spinning  
		if (m->flag == 0) 
			m->flag = 1; // lock is acquired
			m->guard = 0;  
		else  
			queue_add(m->q, gettid());  
			m->guard = 0;  
			park();  
	  
void unlock(lock_t *m) {  
	while (TestAndSet(&m->guard, 1) == 1)  
		; //acquire guard lock by spinning  
		if (queue_empty(m->q))  
			m->flag = 0; // let go of lock; no one wants it
		else  
			unpark(queue_remove(m->q)); // hold lock  
			// (for next thread!)  
		m->guard = 0;  
```


> Written with [StackEdit](https://stackedit.io/).
