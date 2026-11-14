# Simulador de escalonador de processos

Trabalho 1 da disciplina de Sistemas Operacionais, feito por Rafael Lehnen Medeiros e Gustavo Pies Ternus.

O programa simula, tick a tick, como um sistema operacional reveza a CPU entre processos. Ele cria três processos com cargas sorteadas, escalona com duas filas de prontos (prioridade alta e baixa) e três filas de dispositivos (disco, fita e impressora), imprime cada evento e termina com as estatísticas de turnaround e de espera.

Tudo fica em um único arquivo Java, sem dependências. As filas são circulares, implementadas sobre arrays, e os dados de cada processo (o "PCB") ficam em arrays paralelos.

## Regras de escalonamento

| Fila de prontos | Quantum | Quem entra nela |
|---|---|---|
| Alta (`HIGH`) | 4 ticks | Processos novos e quem volta da fita ou da impressora |
| Baixa (`LOW`) | 2 ticks | Quem volta do disco, quem estoura o quantum e quem sofre preempção |

- **Carga de trabalho.** Cada processo recebe de 1 a 10 surtos de CPU (bursts), de 2 a 4 ticks cada. Entre dois surtos há uma operação de E/S em um dispositivo sorteado: disco (1 a 4 ticks), fita (1 a 3) ou impressora (2 a 4). Todos os processos chegam em `t = 0`, na fila alta. O sorteio não usa semente fixa, então cada execução é diferente.
- **Seleção.** Com a CPU livre, entra o primeiro processo da fila alta. Se ela estiver vazia, entra o primeiro da fila baixa.
- **Preempção.** Se o processo em execução veio da fila baixa e aparece alguém na fila alta, ele sai da CPU e volta para o fim da fila baixa. A troca custa um tick: o processo da fila alta só começa a executar no tick seguinte.
- **Fim de quantum.** O processo volta para a fila baixa e guarda o tempo de CPU que falta. Como o maior surto (4 ticks) cabe no quantum da fila alta, na prática o quantum só estoura na fila baixa.
- **Fim de surto.** Se ainda há E/S, o processo fica bloqueado na fila do dispositivo. Se era o último surto, ele termina.
- **Dispositivos.** Cada dispositivo atende um processo por vez, na ordem de chegada. Ao terminar, o processo volta para a fila indicada na tabela.
- **Estados.** Pronto (`READY`), executando (`RUN`), bloqueado (`BLOCK`) e finalizado (`FIN`).
- **Parada.** A simulação acaba quando os três processos terminam, ou ao atingir 500 ticks.

A cada tick, nesta ordem: escolhe um processo se a CPU está livre, avança a E/S dos três dispositivos, executa um tick de CPU (ou faz a preempção) e soma um tick de espera para quem está pronto.

No fim, o programa mostra por processo o turnaround (o tick em que terminou, já que todos chegam em `t = 0`), o tempo de espera em prontos, o tempo total de CPU e o número de surtos, além das médias de turnaround e de espera.

## Como rodar

Precisa de um JDK 11 ou mais novo. Testado com o Temurin 11 e o 21. Na raiz do repositório:

```
java trabalho_1_so_gupo_9/trabalho_1_so_grupo9.java
```

Ou compilando antes:

```
javac -d out trabalho_1_so_gupo_9/trabalho_1_so_grupo9.java
java -cp out sistemas_operacionais_I.trabalho_1_so_gupo_9.trabalho_1_so_grupo9
```

O programa não recebe argumentos. O número de processos, os quanta e as faixas de tempo são constantes no início do arquivo.

## Exemplo de saída

Trecho de uma execução real. As linhas `[...]` marcam cortes.

```
== Carga de Trabalho ==
PID 0 (PPID = -1, prioridade = HIGH): CPU 3 depois DISK 3 CPU 2 depois DISK 2 CPU 2 depois DISK 2 CPU 3
PID 1 (PPID = -1, prioridade = HIGH): CPU 2 depois PRINTER 3 CPU 4 depois TAPE 1 CPU 3
PID 2 (PPID = -1, prioridade = HIGH): CPU 3
t = 0 | Escalonador: selecionou PID 0 da fila HIGH
t = 0 | Executando PID 0 (tempo de CPU restante = 2, quantum restante = 3)
t = 1 | Executando PID 0 (tempo de CPU restante = 1, quantum restante = 2)
t = 2 | Executando PID 0 (tempo de CPU restante = 0, quantum restante = 1)
t = 2 | Início de I/O em DISK para PID 0 (duração = 3)
[...]
t = 11 | Executando PID 1 (tempo de CPU restante = 0, quantum restante = 0)
t = 11 | Início de I/O em TAPE para PID 1 (duração = 1)
t = 12 | Escalonador: selecionou PID 0 da fila LOW
t = 12 | I/O concluído em TAPE para PID 1 (retorna para HIGH)
t = 12 | Preempção: removeu PID 0 (LOW) porque há processo em HIGH
t = 13 | Escalonador: selecionou PID 1 da fila HIGH
[...]
t = 24 | Escalonador: selecionou PID 0 da fila LOW
t = 24 | Executando PID 0 (tempo de CPU restante = 2, quantum restante = 1)
t = 25 | Executando PID 0 (tempo de CPU restante = 1, quantum restante = 0)
t = 25 | Fim de quantum: PID 0 retorna para LOW (tempo de CPU restante = 1)
t = 26 | Escalonador: selecionou PID 0 da fila LOW
t = 26 | Executando PID 0 (tempo de CPU restante = 0, quantum restante = 1)
t = 26 | Finalização: PID 0 terminou a execução
--- Fim da simulação --- tempo = 27 | processos finalizados = 3 de 3
== Estatísticas ==
PID 0 (PPID = -1, prioridade final = LOW) | turnaround = 26 | espera em prontos = 14 | total de tempo de CPU = 10 | quantidade de bursts = 4
PID 1 (PPID = -1, prioridade final = HIGH) | turnaround = 15 | espera em prontos = 5 | total de tempo de CPU = 9 | quantidade de bursts = 3
PID 2 (PPID = -1, prioridade final = HIGH) | turnaround = 7 | espera em prontos = 5 | total de tempo de CPU = 3 | quantidade de bursts = 1
média de turnaround = 16.0
média de espera em prontos = 8.0
```

## Estrutura

```
trabalho_1_so_gupo_9/
  trabalho_1_so_grupo9.java   o simulador inteiro
README.md
```

Dentro do arquivo: `makeProc` sorteia a carga de cada processo, `schedule` escolhe quem entra na CPU, `stepIO` avança os dispositivos, `stepCPU` executa um tick e trata preempção, fim de surto e fim de quantum, e `stats` imprime as estatísticas.
