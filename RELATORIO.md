# UNIVERSIDADE FEDERAL FLUMINENSE

## CURSO DE CIÊNCIA DA COMPUTAÇÃO

**GABRIEL BREVILIERI**

**LUIZ FELIPE ALVES**

# SEGUNDO TRABALHO

## Sistemas Distribuídos

**NITERÓI, 2025**

---

# Sumário

1. [Política de Colaboração no Trabalho](#política-de-colaboração-no-trabalho)
2. [O Algoritmo Raft](#o-algoritmo-raft)
3. [Objetivo do Trabalho](#objetivo-do-trabalho)
4. [Descrição do Código](#descrição-do-código)
5. [Ferramentas Utilizadas](#ferramentas-utilizadas)
6. [Testes](#testes)
7. [Dificuldades Encontradas](#dificuldades-encontradas)
8. [Lições Aprendidas](#lições-aprendidas)
9. [Referências Bibliográficas](#referências-bibliográficas)

---

# Política de Colaboração no Trabalho

O trabalho foi desenvolvido de forma colaborativa por **Luiz** e **Gabriel**, com participação ativa de ambos em todas as etapas do projeto.

Gabriel focou principalmente nas atividades de **implementação do código**, incluindo a estruturação do algoritmo Raft, desenvolvimento das funções de eleição de líder, heartbeats, e resolução de problemas técnicos relacionados a concorrência e race conditions.

Luiz concentrou seus esforços na **escrita e formatação do documento**, elaborando as seções teóricas, documentação técnica, análise de resultados e consolidação das lições aprendidas.

Ambos os membros contribuíram ativamente para:

-   Configuração do ambiente de desenvolvimento
-   Estudo do algoritmo Raft e materiais de referência
-   Execução e análise dos testes
-   Identificação e resolução de dificuldades
-   Revisão final do código e do relatório

---

# O Algoritmo Raft

Um algoritmo de consenso com a proposta de ser mais compreensível que o algoritmo proposto por Lamport, o Paxos, descrito originalmente na dissertação "In Search of an Understandable Consensus Algorithm" publicada por Diego Ongaro e John Ousterhout, da Universidade de Stanford.

**"Raft"** é uma sigla derivada do inglês que significa "Replicated And Fault Tolerant" ("Replicado e tolerante a falhas", em tradução livre). Apesar do uso do Paxos em diversas aplicações, o Raft se propõe a oferecer uma base melhor para a construção e educação em Sistemas Distribuídos, com um algoritmo mais simples, descrito de forma a facilitar o entendimento dos procedimentos.

Sendo um algoritmo de consenso, tem o objetivo de fazer com que os nós de um sistema concordem com uma sequência de eventos (entries) e mantenham um estado consistente de logs pela troca de mensagens via chamadas remotas de procedimento (RPC). É projetado para resistir à perda eventual de mensagens na transmissão, falhas de servidores, garantindo a replicação dos dados e a tolerância a falhas.

O Raft possui três fundamentos: a **eleição do líder**, a **replicação de logs** e a **segurança**. Esses princípios, embora interconectados, podem ser estudados separadamente, contribuindo para a facilidade de aprendizado e aplicação do algoritmo.

## Funcionamento Básico

No Raft, há vários servidores que podem estar em um dos três estados não fixos: **líder**, **seguidor** ou **candidato**. A transição entre os estados é recorrente e essencial à integridade do sistema. Inicialmente, todos os servidores começam como seguidores. Cada nó possui um temporizador aleatório e distinto chamado timeout, que expira se não receber mensagens de um líder ativo. Se expirar, o nó se torna candidato e solicita votos dos outros nós para se tornar líder. Caso um nó receba a maioria dos votos primeiro, ele é eleito como o novo líder do sistema.

**[INSERIR IMAGEM 1: Diagrama de Estados do Raft]**
_Figura 1: Estados dos servidores no Raft (Follower, Candidate, Leader) e transições entre eles._
_Fonte: Artigo "In Search of an Understandable Consensus Algorithm (Extended Version)" - Figura 4_

Um líder é responsável por enviar mensagens periódicas, chamadas de **heartbeats**, aos seguidores. Essas mensagens confirmam que o líder está ativo e funcionando adequadamente. Além disso, o líder recebe solicitações de clientes, armazena-as no log local e inicia o processo de replicação para os seguidores. Os seguidores respondem aos heartbeats confirmando que estão sincronizados.

## Terms e Comunicação

Cada período de liderança no Raft é identificado por um número chamado **term**, que funciona como um marcador lógico de tempo. Esse número é incrementado sempre que um novo líder é eleito. Se um líder ou candidato descobre que outro nó possui um term mais recente, ele abdica de sua posição e se torna seguidor. Isso assegura que haja no máximo um líder ativo por vez.

**[INSERIR IMAGEM 2: Timeline de Terms]**
_Figura 2: Timeline mostrando a evolução de terms e eleições ao longo do tempo._
_Fonte: Artigo "In Search of an Understandable Consensus Algorithm (Extended Version)" - Figura 5_

A comunicação entre os servidores é realizada através de duas chamadas via RPC: **RequestVote** e **AppendEntries**. A primeira é usada durante a eleição para solicitar votos dos seguidores, enquanto a segunda é utilizada para replicar entradas de log e enviar os heartbeats.

Além disso, o algoritmo garante que as operações sejam aplicadas de forma consistente em todos os nós. Para isso, o líder só confirma a execução de uma operação após receber confirmações de que a maioria dos nós atualizou seus logs. Essa abordagem garante que, se a maioria dos nós estiver operando normalmente, o sistema funciona de maneira consistente e coesa.

Com esses mecanismos, o Raft simplifica a implementação de sistemas distribuídos enquanto mantém robustez, tornando-se uma alternativa eficaz e compreensível ao Paxos.

---

# Objetivo do Trabalho

Baseado nos conhecimentos de aula e nos materiais disponibilizados sobre o algoritmo Raft, deseja-se estudar o algoritmo e propor funções na linguagem Go que implementam as funcionalidades do algoritmo Raft no caso de eleição de um líder, levando em consideração possíveis falhas que possam ocorrer em algum componente do sistema. Reportando, ao final, os resultados obtidos neste documento.

## Tarefa

Implementar eleição de líder e heartbeats (RPCs AppendEntries sem entradas no log) conforme a especificação do Raft. O objetivo é que um único líder seja eleito, que o líder continue sendo o líder se não houver falhas e que um novo líder assuma o controle se o antigo líder falhar ou se os pacotes de/para o antigo líder forem perdidos.

Deve-se executar `go test -run 2A` para testar o código.

---

# Descrição do Código

## Estruturas Adicionadas

```go
// Log entry structure
type LogEntry struct {
    Term    int
    Command interface{}
}

// AppendEntries RPC arguments
type AppendEntriesArgs struct {
    Term         int
    LeaderId     int
    PrevLogIndex int
    PrevLogTerm  int
    Entries      []LogEntry
    LeaderCommit int
}

// AppendEntries RPC reply
type AppendEntriesReply struct {
    Term    int
    Success bool
}

// RequestVote RPC arguments
type RequestVoteArgs struct {
    Term         int
    CandidateId  int
    LastLogIndex int
    LastLogTerm  int
}

// RequestVote RPC reply
type RequestVoteReply struct {
    Term        int
    VoteGranted bool
}
```

## Campos Adicionados à Struct Raft

```go
type Raft struct {
    mu        sync.Mutex
    peers     []*labrpc.ClientEnd
    persister *Persister
    me        int
    dead      int32

    // Persistent state
    CurrentTerm int        // último termo que o servidor viu
    VotedFor    int        // id do candidato que recebeu voto no termo atual
    Log         []LogEntry // entradas de log

    // Volatile state
    CommitIndex int
    LastApplied int

    // Volatile state on leaders
    NextIndex  []int
    MatchIndex []int

    // Additional state
    State         int          // papel do servidor (Follower/Candidate/Leader)
    ElectionTimer *time.Timer  // timer de eleição do nó
}
```

## Principais Funções Implementadas

### `func (rf *Raft) GetState() (int, bool)`

Retorna o termo atual e se o servidor acredita ser o líder.

### `func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply)`

Trata os casos de recebimento de uma solicitação de voto: se o nó recebe um termo maior que o seu, ele se torna follower. Verifica se o log do candidato está atualizado antes de conceder voto.

### `func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply)`

Trata os casos de recebimento de uma mensagem de heartbeat ou replicação de log: se o nó recebe um termo maior que o seu, ele se torna follower e reseta seu timeout de eleição.

### `func (rf *Raft) initiateElection()`

Começa uma eleição de acordo com as regras do algoritmo Raft: transforma o nó em candidato, incrementa o termo, envia mensagens RequestVote para peers e verifica se possui mais da metade dos votos.

### `func (rf *Raft) sendHeartbeats(leaderTerm int)`

Simula comportamento do líder, utilizando-se de paralelismo para envio de heartbeats periódicos através de AppendEntries.

### `func (rf *Raft) electionTicker()`

Gerencia o timer de eleição, fazendo com que um nó inicie uma eleição caso não tenha recebido nenhuma mensagem de um líder de acordo com o timeout.

### Funções Auxiliares

-   `func (rf *Raft) transitionToFollower(term int)`: Transição para estado de follower
-   `func (rf *Raft) isLogUpToDate(candidateLastIdx, candidateLastTerm int) bool`: Verifica se o log do candidato está atualizado
-   `func (rf *Raft) getLastLogInfo() (int, int)`: Retorna índice e termo da última entrada do log
-   `func getRandomTimeout() time.Duration`: Gera timeout aleatório para eleição
-   `func (rf *Raft) resetElectionTimeout()`: Reseta o timer de eleição
-   `func (rf *Raft) broadcastHeartbeat()`: Envia heartbeat para todos os peers (usado por sendHeartbeats)

**[INSERIR IMAGEM 3: Figura 2 do Paper - Resumo do Algoritmo Raft]**
_Figura 3: Resumo condensado do algoritmo de consenso Raft com todas as regras de implementação._
_Fonte: Artigo "In Search of an Understandable Consensus Algorithm (Extended Version)" - Figura 2_

---

# Ferramentas Utilizadas

A entrega foi feita utilizando exclusivamente funcionalidades da linguagem Go, com as seguintes ferramentas e bibliotecas:

### Linguagem e Ambiente

-   **Go (Golang)**: Linguagem de programação principal
-   **Sistema Operacional**: macOS
-   **Editor**: Visual Studio Code com extensão Go

### Bibliotecas Importadas

-   **labrpc**: Pacote RPC fornecido pelo laboratório para comunicação entre peers
    -   Simula rede com possibilidade de perda de pacotes
    -   Permite atrasos e reordenação de mensagens
-   **sync**: Pacote padrão do Go para sincronização
    -   `sync.Mutex` para proteção de seções críticas
    -   `sync.Once` para inicialização única do gerador aleatório
-   **time**: Pacote padrão do Go para temporização
    -   `time.Timer` para timeouts de eleição
    -   `time.Ticker` para envio periódico de heartbeats
-   **math/rand**: Geração de números aleatórios para definição e cálculo de election timeouts dos nós
-   **sync/atomic**: Operações atômicas para flag de shutdown

### Ferramentas de Teste

-   **go test**: Framework de testes nativo do Go
    -   Execução dos testes fornecidos (2A)
    -   Testes com `-race` para detecção de race conditions

### Código Base

Aproveitou-se as implementações já presentes no código base referentes ao protocolo de comunicação RPC. Implementou-se as funções que foram disponibilizadas de maneira incompleta, para que o código pudesse atender aos requisitos do teste 2A.

---

# Testes

O teste alvo é o 2A, que trata do mecanismo de eleição do líder e disparo de heartbeats.

## Resultados Obtidos

O programa passou em todas as verificações de testes do Lab 2A:

**[INSERIR IMAGEM 4: Screenshot dos Testes 2A Passando]**
_Figura 4: Captura de tela mostrando a execução bem-sucedida dos testes do Lab 2A._
_Comando executado: `go test -run 2A`_

```
Test (2A): initial election ...
  ... Passed
Test (2A): election after network failure ...
  ... Passed
PASS
ok  raft  8.225s
```

### Execução com Race Detection

**[INSERIR IMAGEM 5: Screenshot dos Testes com Race Detection]**
_Figura 5: Captura de tela mostrando a execução dos testes com detecção de race conditions._
_Comando executado: `go test -run 2A -race`_

```
Test (2A): initial election ...
  ... Passed
Test (2A): election after network failure ...
  ... Passed
PASS
ok  raft  9.314s
```

### Análise dos Resultados

Os retornos "Passed" indicam que o programa passou em cada verificação de testes. Todas as execuções atingiram 100% de aceitação pelo teste 2A descrito, indicando que o algoritmo está implementado de acordo com as especificações do funcionamento da eleição de um líder no algoritmo.

**Estatísticas**:

-   **Taxa de Sucesso**: 100% em múltiplas execuções consecutivas (10+ execuções)
-   **Tempo de Execução**: Intervalo entre 7,4s a 9,3s
-   **Race Conditions**: 0 detectadas após correções

---

# Dificuldades Encontradas

Dentre as dificuldades encontradas, as principais foram:

1. **Falta de Conhecimento Prévio sobre Go**: A linguagem Go era nova, exigindo tempo para compreender suas particularidades e funcionalidades específicas para programação concorrente.

2. **Compreensão do Algoritmo Raft**: Os detalhamentos das etapas do algoritmo devido aos possíveis cenários indesejados que podem existir dentro do contexto de troca de mensagem, timeouts e possibilidade dos nós poderem falhar.

3. **Compreensão do Projeto Base**: Entender a estrutura do projeto fornecido e a maneira como os demais elementos interagem com o algoritmo desenvolvido.

4. **Race Conditions**: Múltiplas goroutines acessando variáveis compartilhadas sem proteção adequada causavam condições de corrida detectadas pelo `go test -race`.

5. **Problemas com Timers**: Uso incorreto de `time.Timer` do Go causava deadlocks ou disparos inesperados de timeouts.

6. **Timeouts Simultâneos**: Quando todos os servidores iniciavam eleição ao mesmo tempo, causava empates constantes e nenhum líder era eleito.

7. **Múltiplos Líderes**: Em alguns casos, mais de um servidor se declarava líder no mesmo termo devido a condições de corrida.

8. **Ajuste de Parâmetros**: Encontrar valores adequados para timeouts que satisfizessem as restrições (máximo 10 heartbeats por segundo e eleição completa em menos de 5 segundos).

## Como as Dificuldades Foram Contornadas

Para contornar as dificuldades, foram adotadas as seguintes estratégias:

-   **Estudo da Linguagem Go**: Busca de conhecimento através da documentação oficial [GO], tutoriais e guias [W3] para entender suas principais funcionalidades, especialmente relacionadas a concorrência e sincronização.

-   **Estudo do Algoritmo**: Uso intensivo dos materiais disponibilizados na descrição do trabalho [RA], em especial a **Figura 2 da versão estendida do artigo sobre Raft** e o guia ilustrado. Com isso, foi possível compreender o algoritmo de maneira muito didática, funcionando na prática.

-   **Proteção de Seções Críticas**: Garantia de que todos os acessos a variáveis compartilhadas ocorram dentro de seções protegidas por `mutex`.

-   **Implementação Correta de Timers**: Seguir o padrão correto de reset de timers do Go, incluindo drenagem do canal quando necessário.

-   **Timeouts Aleatórios**: Implementação de timeouts aleatórios (450-850ms) usando `rand.Int63n()` com seed inicializado por `sync.Once`.

-   **Verificações de Estado**: Verificação rigorosa do estado e termo antes de qualquer transição de estado, garantindo consistência.

-   **Testes Iterativos**: Execução múltipla dos testes com `-race` para detectar e corrigir race conditions.

---

# Lições Aprendidas

As lições aprendidas estão relacionadas a:

1. **Maior Compreensão da Linguagem Go**:

    - Modelo de concorrência com goroutines e channels
    - Uso de mutexes e operações atômicas
    - Padrões corretos para timers e tickers
    - Importância do `defer` para garantir liberação de recursos

2. **Entendimento Profundo do Protocolo Raft**:

    - Política de eleição e transições de estado
    - Gerenciamento de timeouts aleatórios
    - Envio de heartbeats periódicos
    - Maneiras de garantir consistência de dados entre diferentes nós
    - Importância de seguir rigorosamente a Figura 2 do paper

3. **Programação Concorrente**:

    - Proteção adequada de estado compartilhado é fundamental
    - Race conditions podem ser sutis e não aparecer em execuções simples
    - Ferramenta `-race` do Go é invaluável para detectar problemas
    - Verificações duplas de estado são necessárias em ambientes concorrentes

4. **Sistemas Distribuídos**:

    - Consenso é complexo mesmo em algoritmos "simples"
    - Lidar com falhas parciais onde alguns nós falham mas outros continuam
    - Comunicação assíncrona exige cuidado com verificações de estado
    - Timeouts bem calibrados são cruciais para performance e correção

5. **Testes e Debugging**:
    - Testes automatizados são essenciais em cenários complexos
    - Múltiplas execuções revelam bugs não-determinísticos
    - Análise de traces ajuda a entender sequência de eventos

---

# Referências Bibliográficas

-   **[GO]** Google. **Documentation - The Go Programming**. Disponível em: https://go.dev/doc/. Acesso em: 22/11/2025.

-   **[W3]** W3Schools. **Go Tutorial**. Disponível em: https://www.w3schools.com/go/. Acesso em: 22/11/2025.

-   **[RA]** ONGARO, Diego; OUSTERHOUT, John. **In Search of an Understandable Consensus Algorithm (Extended Version)**. Stanford University, 2014. Disponível em: https://raft.github.io/raft.pdf.

-   **[SG]** **Students' Guide to Raft**. Disponível em: https://thesquareplanet.com/blog/students-guide-to-raft/. Acesso em: 22/11/2025.

-   **[VIS]** **The Secret Lives of Data - Raft Visualization**. Disponível em: http://thesecretlivesofdata.com/raft/. Acesso em: 22/11/2025.

-   **[LAB]** **Lab 2: Raft - CS451/651 Distributed Systems**. Disponível em: http://www.cs.bu.edu/~jappavoo/jappavoo.github.com/451/labs/lab-raft.html. Acesso em: 22/11/2025.
