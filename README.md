# Lab 01

**Disciplina:** Sistemas Operacionais

**Aluno:** Victor Paulo

**Ambiente:** Ubuntu em VirtualBox, Python 

## Arquivos

| Arquivo | Descrição |
|---|---|
| `conta_bancaria_insegura.py` | Parte 1: 2 threads depositando sem sincronização |
| `conta_bancaria_segura.py` | Parte 2: mesma lógica protegida por `threading.Lock` |
| `conta_bancaria_bonus.py` | Bônus: 2 depósitos e 1 saque |

Nas versões insegura e segura, além do código do roteiro, foi acrescentada apenas a medição de tempo (`time.time()`), necessária para a Questão 2.

## Resultados

### Parte 1: versão insegura (3 execuções)

| Execução | Saldo esperado | Saldo obtido | Tempo (s) |
|---|---|---|---|
| 1 | 200000 | 200000 | 0,0150 |
| 2 | 200000 | 200000 | 0,0162 |
| 3 | 200000 | 200000 | 0,0150 |

**Observação:** a condição de corrida não se manifestou nesta máquina. O roteiro diz que o valor final "quase nunca" atinge 200.000, e aqui ele atingiu nas três execuções. O resultado depende do escalonador, da versão do Python e da carga do sistema.

<img width="770" height="458" alt="Captura de tela 2026-09-23 213302" src="https://github.com/user-attachments/assets/c825c73e-0e57-41c1-8e85-06f4d295fe96" />

### Parte 2: versão segura com Lock (3 execuções)

| Execução | Saldo esperado | Saldo obtido | Tempo (s) |
|---|---|---|---|
| 1 | 200000 | 200000 | 0,0341 |
| 2 | 200000 | 200000 | 0,0316 |
| 3 | 200000 | 200000 | 0,0335 |

<img width="692" height="313" alt="Captura de tela 2026-09-23 213457" src="https://github.com/user-attachments/assets/e9ad760a-eb4e-4524-9af0-c1d9b7676332" />

### Bônus: 2 depósitos + 1 saque (3 execuções)

| Execução | Saldo esperado | Saldo obtido | Tempo (s) |
|---|---|---|---|
| 1 | 100000 | 100000 | 0,0465 |
| 2 | 100000 | 100000 | 0,0491 |
| 3 | 100000 | 100000 | 0,0597 |

<img width="803" height="452" alt="Captura de tela 2026-09-23 213805" src="https://github.com/user-attachments/assets/42e92682-1309-47d7-b7be-81f8c4df11b5" />

## Questão 1: Troca de contexto e atomicidade

A operação `temp = saldo_conta; temp = temp + 1; saldo_conta = temp` parece simples, mas não é atômica. A CPU a executa em várias instruções: carregar o valor da memória para um registrador, somar 1 e gravar o resultado de volta na memória. O escalonador pode interromper a thread entre quaisquer dessas instruções (troca de contexto).

Exemplo: a Thread A lê 100 e é interrompida. A Thread B lê 100, soma e grava 101. A Thread A retoma com o `temp` antigo (100), soma e grava 101. Duas somas resultaram em apenas +1, e uma atualização foi perdida. Isso é uma condição de corrida, e o trecho que acessa a variável compartilhada é a **seção crítica**.

Nas execuções deste laboratório o erro não apareceu. No CPython, o GIL permite que apenas uma thread execute bytecode por vez, e a troca de thread só ocorre após um intervalo (cerca de 5 ms). Como cada thread terminou seu laço em poucos milissegundos, raramente foi interrompida entre a leitura e a escrita. Isso **não** torna o código seguro: a corrida continua possível e depende do escalonador, da carga da máquina e da versão do Python.

## Questão 2: Custo do Lock

O tempo médio da versão insegura foi de cerca de 0,0154 s, e o da segura, cerca de 0,0331 s, ou seja, aproximadamente 2,1 vezes mais. O Lock adiciona overhead porque:

1. **Custo da sincronização:** cada `with lock_bancario` faz uma aquisição e uma liberação do Lock, 200.000 vezes no total. Mesmo sem disputa, isso tem um custo que a versão insegura não tem.
2. **Bloqueio e troca de contexto:** quando uma thread tenta pegar o Lock e ele está ocupado, ela precisa esperar, o que pode envolver suspender e acordar a thread, gerando trocas de contexto adicionais.
3. **Serialização:** apenas uma thread por vez executa a seção crítica, então o trecho protegido não roda em paralelo. Em Python o GIL já limita o paralelismo, então aqui a diferença vem em boa parte do custo das chamadas ao Lock. Em linguagens sem GIL, a serialização pesa mais.

O preço em tempo garante a correção: com o Lock, o saldo foi 200.000 em todas as execuções.

## Desafio extra: 3 threads (2 depósitos + 1 saque)

Duas threads depositam R$ 1,00 e uma saca R$ 1,00, cada uma com 100.000 operações. O saldo esperado é 100.000 + 100.000 − 100.000 = **100.000**, e foi o obtido nas três execuções.

- As três threads usam o **mesmo** `lock_bancario`. Como depósitos e saque alteram a mesma variável, precisam disputar o mesmo Lock. Locks diferentes não dariam exclusão mútua entre depósito e saque.
- Cada operação é aplicada por inteiro, uma de cada vez, então nenhuma atualização se perde.
- O saque não verifica saldo, então a conta pode ficar negativa durante a execução. Num sistema real, essa validação deveria ser feita dentro da seção crítica.

## Como executar

```bash
python3 conta_bancaria_insegura.py
python3 conta_bancaria_segura.py
python3 conta_bancaria_bonus.py
```
