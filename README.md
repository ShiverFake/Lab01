# Laboratório 01 — Concorrência, Threads e Condição de Corrida

**Disciplina:** Sistemas Operacionais — 4º Semestre / 2026.2
**Docente:** Prof. Esp. Rodrigo Martins Sousa
**Aluno(s):** Victor Paulo Arueira Silva
**Ambiente de execução:** Ubuntu _[versão]_ em máquina virtual (VirtualBox, host Windows 10) — Python _[saída de `python3 --version`]_

---

## Estrutura do repositório

```
.
├── conta_bancaria_insegura.py   # Parte 1 — sem sincronização
├── conta_bancaria_segura.py     # Parte 2 — com Mutex (Lock)
├── desafio_bonus.py             # Bônus — 3 threads (2 depósitos + 1 saque)
├── evidencias/                  # capturas de tela das execuções
└── README.md
```

---

## 1. Execução prática

### Parte 1 — versão insegura (3 execuções)

| Execução | Saldo esperado | Saldo obtido | Operações perdidas | Tempo (s) |
|----------|----------------|--------------|--------------------|-----------|
| 1ª       | 40000          | 20000        | _[valor]_          | 7,8469s   |
| 2ª       | 40000          | 20002        | _[valor]_          | 7.5189s   |
| 3ª       | 40000          | 20000        | _[valor]_          | 8,4192s   |

O resultado varia a cada execução porque depende do escalonamento das threads pelo
sistema operacional, que não é determinístico.

<img width="614" height="275" alt="image" src="https://github.com/user-attachments/assets/06a77a22-c4eb-43d9-b771-176f63040410" />

### Parte 2 — versão segura

| Saldo esperado | Saldo obtido | Tempo (s) |
|----------------|--------------|-----------|
| 200000         | 200000       | _[valor]_ |

![Execução da versão segura](evidencias/segura.png)

### Bônus — 3 threads

| Saldo esperado | Saldo obtido | Tempo (s) |
|----------------|--------------|-----------|
| 100000         | 100000       | _[valor]_ |

![Execução do desafio bônus](evidencias/bonus.png)

---

## 2. Análise teórica

### Questão 1 — Troca de contexto e atomicidade

A instrução `temp = temp + 1` parece uma única ação para quem lê o código-fonte, mas
não é assim que ela chega ao processador. Em nível de hardware, a CPU não consegue
somar um valor diretamente na memória RAM: ela precisa de três etapas distintas,
conhecidas como ciclo **Read-Modify-Write**:

1. **LOAD** — copiar o conteúdo do endereço de memória de `saldo_conta` para um
   registrador do processador;
2. **ADD** — somar 1 ao valor que está no registrador;
3. **STORE** — gravar o conteúdo do registrador de volta no endereço de memória.

No Python isso fica visível ao desmontar a função com o módulo `dis`: uma única linha
de código vira várias instruções de bytecode (`LOAD_FAST`, `LOAD_CONST`, `BINARY_OP`,
`STORE_FAST`, `STORE_GLOBAL`). A operação, portanto, é **não-atômica**: ela pode ser
interrompida no meio.

O escalonador do sistema operacional trabalha com **preempção**. Quando o *quantum*
(fatia de tempo) de uma thread termina, o SO realiza uma **troca de contexto**: salva o
estado dos registradores daquela thread no seu TCB (Thread Control Block) e carrega o
estado de outra. Essa interrupção pode acontecer exatamente entre os passos 1 e 3.

Cenário concreto de perda de dados, com `saldo_conta = 50`:

| Tempo | Thread-Caixa-1        | Thread-App-2          | saldo_conta (memória) |
|-------|-----------------------|-----------------------|-----------------------|
| t1    | lê 50 → registrador   | —                     | 50                    |
| t2    | soma: registrador = 51| —                     | 50                    |
| t3    | **troca de contexto** | —                     | 50                    |
| t4    | (suspensa, valor 51 salvo no TCB) | lê 50 → registrador | 50        |
| t5    | —                     | soma: registrador = 51| 50                    |
| t6    | —                     | escreve 51            | **51**                |
| t7    | volta e escreve 51    | —                     | **51**                |

Duas operações de depósito foram executadas, mas o saldo subiu apenas uma unidade. A
segunda escrita simplesmente sobrescreveu a primeira — é a chamada **atualização
perdida** (*lost update*). Multiplicado por 100.000 iterações em cada thread, isso
explica a diferença observada na tabela da Parte 1.

Vale registrar uma particularidade do CPython: o **GIL** (Global Interpreter Lock)
permite que apenas uma thread execute bytecode por vez, mas ele é liberado
periodicamente (a cada ~5 ms, valor definido por `sys.getswitchinterval()`) e também
entre instruções de bytecode. Ou seja, o GIL **não** transforma a sequência
leitura-modificação-escrita em uma operação atômica — por isso a condição de corrida
ocorre mesmo em Python.

### Questão 2 — Custo do Lock

| Versão   | Tempo de execução | Diferença |
|----------|-------------------|-----------|
| Insegura | _[valor]_ s       | —         |
| Segura   | _[valor]_ s       | _[Nx mais lenta]_ |

A versão protegida é mais lenta por quatro motivos, todos ligados ao funcionamento
do SO:

1. **Chamadas adicionais a cada iteração.** Cada passagem pelo laço executa um
   `acquire()` e um `release()`. São 200.000 pares de operações que simplesmente não
   existiam na versão insegura.

2. **Serialização forçada.** A exclusão mútua elimina o paralelismo dentro da seção
   crítica por definição: o que antes podia ser executado de forma entrelaçada agora
   precisa acontecer em fila indiana. Ganha-se integridade, perde-se concorrência.

3. **Bloqueio e desbloqueio de threads.** Quando uma thread tenta adquirir um lock
   já ocupado, ela sai do estado *running* e vai para *blocked*. O SO precisa
   removê-la da CPU, colocá-la na fila de espera do mutex e, quando o lock é
   liberado, movê-la de volta para a fila de prontos. Cada uma dessas transições é
   uma troca de contexto, que custa salvamento/restauração de registradores e
   invalidação de linhas de cache e da TLB.

4. **Contenção.** Com a seção crítica muito curta e disputadíssima, as threads passam
   proporcionalmente mais tempo negociando o acesso ao lock do que fazendo trabalho
   útil. Esse é o pior caso para um mutex.

A conclusão prática é que sincronização não é de graça: o programador deve manter a
seção crítica **a menor possível**, protegendo apenas o que de fato é compartilhado.

### Questão 3 — Desafio extra: 3 threads

A implementação está em `desafio_bonus.py`. Duas threads executam `depositar()` e uma
executa `sacar()`, todas sobre a mesma variável `saldo_conta`.

O ponto central é que as três threads compartilham **o mesmo objeto `Lock`**. Um mutex
só garante integridade se *todos* os caminhos de acesso ao recurso compartilhado
passarem por ele — bastaria a função de saque não usar o lock (ou usar um lock
próprio) para que a corrida voltasse a ocorrer, mesmo com os depósitos protegidos.

Como cada operação vale R$ 1,00:

```
saldo_final = (2 × 100000) − (1 × 100000) = 100000
```

O resultado obtido confirmou o valor esperado, comprovando que a exclusão mútua se
mantém consistente independentemente do número de threads ou do tipo de operação
realizada sobre o recurso.

---

## 3. Como executar

```bash
python3 conta_bancaria_insegura.py   # rodar 3 vezes
python3 conta_bancaria_segura.py
python3 desafio_bonus.py
```
