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
2. [Descrição da Implementação](#descrição-da-implementação)
3. [Ferramentas Utilizadas](#ferramentas-utilizadas)
4. [Resultados dos Testes](#resultados-dos-testes)
5. [Dificuldades Encontradas](#dificuldades-encontradas)
6. [Lições Aprendidas](#lições-aprendidas)
7. [Referências Bibliográficas](#referências-bibliográficas)

---

# Política de Colaboração no Trabalho

O trabalho foi desenvolvido de forma colaborativa por **Gabriel** e **Luiz**, com participação ativa de ambos em todas as etapas do projeto, mantendo comunicação constante e revisão mútua do trabalho.

**Gabriel** concentrou seus esforços principalmente na parte técnica do projeto, sendo responsável pela implementação do algoritmo Raft, desenvolvimento das funções e estruturas necessárias, resolução de problemas relacionados à concorrência e sincronização, e configuração do ambiente de desenvolvimento.

**Luiz** dedicou-se especialmente à documentação e organização do trabalho, elaborando o relatório técnico, consolidando as informações sobre ferramentas utilizadas, dificuldades encontradas e lições aprendidas, além de estruturar e formatar o documento final.

Ambos os membros trabalharam em conjunto no estudo do algoritmo Raft, execução e validação dos testes, discussão das soluções para os problemas encontrados, e revisão final tanto do código quanto da documentação. Todo o código implementado foi escrito pelos autores, com exceção do código base fornecido pelo laboratório, seguindo rigorosamente a política de colaboração estabelecida na especificação do trabalho.

---

# Descrição da Implementação

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

---

# Ferramentas Utilizadas

A entrega foi feita utilizando exclusivamente funcionalidades da linguagem Go, com as seguintes ferramentas e bibliotecas:

### Linguagem e Ambiente

-   **Go (Golang)**: Linguagem de programação principal
-   **Sistema Operacional**: macOS
-   **Editor**: Visual Studio Code

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

### Configuração do Ambiente

Durante a configuração inicial do ambiente de desenvolvimento, foi necessário ajustar a forma de execução dos testes. A especificação do trabalho sugeria configurar as variáveis de ambiente da seguinte forma:

```bash
$ export GOROOT=<Local de instalação do Go>
$ export PATH=$PATH:$GOROOT/bin
$ export GOPATH=$HOME/451
$ cd $GOPATH
$ cd src/raft
$ go test -run 2A
```

Entretanto, esta abordagem não funcionou diretamente no ambiente macOS utilizado. Foi necessário criar um script `setenv.sh` customizado e ajustar o `GOPATH` para o diretório correto do projeto:

```bash
$ cd <diretório-do-projeto>
$ export GO111MODULE=off
$ export GOPATH="$(pwd)"
$ cd src/raft
$ go test -run 2A
```

Esta configuração foi necessária porque o projeto utiliza a estrutura de workspace Go tradicional (modo `GOPATH`) ao invés do sistema de módulos Go mais recente (`GO111MODULE`), e o caminho precisava apontar para a raiz do repositório clonado.

---

# Resultados dos Testes

O programa foi testado utilizando o teste 2A, que verifica o mecanismo de eleição do líder e envio de heartbeats.

## Execução dos Testes

**[INSERIR IMAGEM: Screenshot da execução do teste]**
_Figura: Captura de tela mostrando a execução bem-sucedida dos testes do Lab 2A._
_Comando executado: `go test -run 2A`_

O programa passou em todas as verificações de testes do Lab 2A:

```
Test (2A): initial election ...
  ... Passed
Test (2A): election after network failure ...
  ... Passed
PASS
ok  raft  8.225s
```

Os resultados "Passed" indicam que o algoritmo está implementado corretamente de acordo com as especificações do Raft para eleição de líder. O teste verifica:

-   **Initial election**: Se um único líder é eleito corretamente quando o sistema é iniciado
-   **Election after network failure**: Se um novo líder assume o controle quando o líder anterior falha ou perde comunicação

O tempo de execução ficou entre 7-9 segundos, demonstrando que os timeouts estão adequadamente calibrados para satisfazer os requisitos do laboratório.

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

8. **Ajuste de Parâmetros**: Encontrar valores adequados para timeouts que satisfizessem as restrições (não enviar mais que 10 heartbeats por segundo para não sobrecarregar a rede, e garantir eleição completa em menos de 5 segundos).

## Como as Dificuldades Foram Contornadas

Para contornar as dificuldades, foram adotadas as seguintes estratégias:

-   **Estudo da Linguagem Go**: Busca de conhecimento através da documentação oficial [GO], tutoriais e guias [W3] para entender suas principais funcionalidades, especialmente relacionadas a concorrência e sincronização.

-   **Estudo do Algoritmo**: Uso intensivo dos materiais disponibilizados na descrição do trabalho [RA], incluindo o artigo original do Raft e guias ilustrados disponíveis online. Com isso, foi possível compreender o algoritmo de maneira didática e aplicá-lo na prática.

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
    - Importância de seguir rigorosamente as especificações do algoritmo

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
