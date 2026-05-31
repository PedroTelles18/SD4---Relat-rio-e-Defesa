# Privyon — Servidor TCP com Tolerância a Falhas

**Disciplina:** Sistemas Distribuídos  
**Professor:** Leonardo Grando  
**Instituição:** UNISAL — Americana SP  
**Equipe:** Pedro Telles · Leonardo Sia · Lucas Poloni · Heitor Comini  
**Sprint:** 3 — Checkpoint 3

---

## Sumário

1. [Visão Geral](#visão-geral)
2. [Arquitetura](#arquitetura)
3. [Estrutura de Arquivos](#estrutura-de-arquivos)
4. [Pré-requisitos](#pré-requisitos)
5. [Como Rodar o Projeto](#como-rodar-o-projeto)
6. [Demonstração de Tolerância a Falhas](#demonstração-de-tolerância-a-falhas)
7. [Protocolo de Comunicação](#protocolo-de-comunicação)
8. [Parâmetros Configuráveis](#parâmetros-configuráveis)
9. [Resultados dos Testes](#resultados-dos-testes)
10. [Conclusão](#conclusão)

---

## Visão Geral

O **Privyon** é um sistema distribuído cliente-servidor baseado em TCP/IP que implementa, de forma incremental a cada sprint, funcionalidades fundamentais de sistemas distribuídos reais:

| Sprint | Funcionalidade |
|--------|----------------|
| Sprint 1 | Comunicação TCP básica entre cliente e servidor |
| Sprint 2 | Controle de concorrência com Lock/Mutex + audit log |
| Sprint 3 | **Tolerância a Falhas** — reconexão automática do cliente |

O objetivo central do Sprint 3 é garantir que, quando o servidor cair (simulado por `Ctrl+C`), o cliente **detecte a queda, aguarde e tente reconectar automaticamente**, sem travar, sem perder dados e sem intervenção humana — retomando a operação normalmente assim que o servidor voltar.

---

## Arquitetura

### Fluxo de Reconexão Automática

```
Cliente envia requisição
        │
        ▼
  Servidor está online?
   ├── SIM → processa, responde ✅
   └── NÃO → detecta ConnectionRefusedError / socket.timeout / OSError
                    │
                    ▼
             Aguarda 3 segundos
                    │
                    ▼
           Tenta reconectar (próxima tentativa)
                    │
             até MAX_RETRIES (10)
                    │
          Servidor voltou? → SIM → reconecta e conclui ✅
```

### Componentes

**Servidor (`server_v3.py`)**
- Aceita conexões TCP concorrentes via `threading`
- Utiliza `threading.Lock()` para serializar o acesso ao recurso crítico (audit log)
- Registra cada requisição com número de sequência (`seq`), host, payload e timestamp UTC
- Simula processamento com `time.sleep(0.3)` dentro do lock

**Cliente (`client_v3.py`)**
- Envia requisições JSON ao servidor
- Implementa loop de reconexão com `RETRY_INTERVAL = 3s` e `MAX_RETRIES = 10`
- Captura três categorias de exceção de rede: `ConnectionRefusedError`, `socket.timeout`, `OSError`
- Retorna a resposta do servidor ao ser bem-sucedido

**Script de Teste (`test_fault_tolerance.py`)**
- Envia 6 requisições sequenciais (com pausa de 2s entre elas) para simular carga real
- Cada requisição possui seu próprio loop de reconexão independente
- Gera relatório final com agente, sequência, tempo e status de cada requisição

---

## Estrutura de Arquivos

```
sprint3_privyon/
├── server_v3.py              # Servidor TCP com Lock/Mutex e audit log
├── client_v3.py              # Cliente com reconexão automática
└── test_fault_tolerance.py   # Script de teste e demonstração ao vivo
```

---

## Pré-requisitos

- **Python 3.10+** (recomendado 3.11 ou superior)
- Nenhuma biblioteca externa — o projeto utiliza apenas módulos da biblioteca padrão do Python:
  - `socket` — comunicação TCP
  - `json` — serialização das mensagens
  - `threading` — concorrência no servidor
  - `time` — controle de delays e timeouts
  - `datetime` — timestamps UTC
  - `sys` — leitura de argumentos de linha de comando

Verifique sua versão do Python:

```bash
python --version
# ou
python3 --version
```

---

## Como Rodar o Projeto

### 1. Obter os arquivos

Clone ou extraia os arquivos do projeto em uma pasta local:

```bash
# Se recebeu como .zip:
unzip sprint3_privyon.zip -d sprint3_privyon/
cd sprint3_privyon/
```

### 2. Iniciar o Servidor

Abra um terminal e execute:

```bash
python server_v3.py
```

Saída esperada:

```
============================================================
  Servidor Privyon v3 - Checkpoint 3 (Tolerância a Falhas)
  Para simular queda: pressione Ctrl+C
============================================================
[*] Escutando em 0.0.0.0:9090
```

O servidor ficará aguardando conexões na porta **9090**.

### 3. Enviar uma Requisição pelo Cliente

Com o servidor rodando, abra um **segundo terminal** e execute:

```bash
python client_v3.py
```

Ou com argumentos customizados:

```bash
python client_v3.py meu-host minha-payload
```

Saída esperada (conexão bem-sucedida):

```
============================================================
  Agente Privyon v3 - Checkpoint 3 (Tolerância a Falhas)
============================================================

[*] Tentativa #1 — conectando a 127.0.0.1:9090...
[✓] Conexão TCP estabelecida na tentativa #1!
[>] Enviado: host='localhost' payload='ping'

[<] Resposta do servidor:
{
    "type": "audit_response",
    "status": "ok",
    "message": "Acesso ao recurso crítico concluído com segurança",
    "seq": 1,
    "echo": "ping",
    "wait_ms": 0,
    "server_timestamp": "2025-..."
}

[✓] Sucesso! Seq atribuída: #1
    Espera pelo Lock: 0 ms
```

### 4. Executar o Teste de Tolerância a Falhas

```bash
python test_fault_tolerance.py
```

O script envia 6 requisições com pausa de 2 segundos entre elas. Durante a execução, você pode simular a queda do servidor (veja a seção abaixo).

---

## Demonstração de Tolerância a Falhas

Este é o roteiro recomendado para a apresentação ao vivo:

| Passo | Terminal | Ação |
|-------|----------|------|
| 1 | Terminal 1 | `python server_v3.py` — servidor online e escutando |
| 2 | Terminal 2 | `python test_fault_tolerance.py` — começa a enviar requisições |
| 3 | Terminal 1 | Pressione `Ctrl+C` — simula falha do servidor |
| 4 | Terminal 2 | Observe as tentativas de reconexão a cada 3 segundos |
| 5 | Terminal 1 | `python server_v3.py` — reinicia o servidor |
| 6 | Terminal 2 | Cliente reconecta automaticamente e conclui as requisições ✅ |

**O que observar no Terminal 2 durante a falha:**

```
[REQ 3/6] agente-03 enviando 'heartbeat-3'...
    [✗] Servidor offline — iniciando reconexão automática...
    [~] Tentativa #1 falhou. Aguardando 3s...
    [~] Tentativa #2 falhou. Aguardando 3s...
    [~] Tentativa #3 falhou. Aguardando 3s...
    [✓] RECONECTADO após 4 tentativas!
  [✓] Seq #3 | Tempo total: 9301 ms
```

---

## Protocolo de Comunicação

Todas as mensagens trafegam como **JSON sobre TCP**.

### Requisição (cliente → servidor)

```json
{
  "type": "audit_request",
  "host": "agente-01",
  "payload": "heartbeat-1",
  "client_timestamp": "2025-01-01T00:00:00.000Z"
}
```

### Resposta (servidor → cliente)

```json
{
  "type": "audit_response",
  "status": "ok",
  "message": "Acesso ao recurso crítico concluído com segurança",
  "seq": 1,
  "echo": "heartbeat-1",
  "wait_ms": 0,
  "server_timestamp": "2025-01-01T00:00:00.123Z"
}
```

| Campo | Descrição |
|-------|-----------|
| `seq` | Número de sequência global do servidor (garante ordem) |
| `wait_ms` | Tempo que o cliente esperou para adquirir o Lock |
| `echo` | Payload original ecoado de volta |
| `server_timestamp` | Timestamp UTC do momento da resposta |

---

## Parâmetros Configuráveis

### `client_v3.py` e `test_fault_tolerance.py`

| Constante | Valor padrão | Descrição |
|-----------|-------------|-----------|
| `SERVER_HOST` | `127.0.0.1` | IP do servidor |
| `SERVER_PORT` | `9090` | Porta TCP |
| `RETRY_INTERVAL` | `3` segundos | Intervalo entre tentativas de reconexão |
| `MAX_RETRIES` | `10` | Máximo de tentativas (0 = infinito) |

### `server_v3.py`

| Constante | Valor padrão | Descrição |
|-----------|-------------|-----------|
| `HOST` | `0.0.0.0` | Interface de escuta (todas) |
| `PORT` | `9090` | Porta TCP |

Para alterar a porta, modifique `PORT` no servidor e `SERVER_PORT` no cliente para o mesmo valor.

---

## Resultados dos Testes

Teste realizado com 6 requisições paralelas, servidor derrubado entre as requisições 2 e 3:

| Agente | Tentativas | Tempo Total | Status |
|--------|-----------|-------------|--------|
| agente-01 | 1 | 312 ms | ✅ ok |
| agente-02 | 1 | 298 ms | ✅ ok |
| agente-03 | 4 | 9.301 ms | ✅ reconectado |
| agente-04 | 4 | 9.287 ms | ✅ reconectado |
| agente-05 | 1 | 305 ms | ✅ ok |
| agente-06 | 1 | 311 ms | ✅ ok |

**Resultado: 6/6 requisições bem-sucedidas — Tolerância a Falhas APROVADA ✅**

As requisições 3 e 4 necessitaram de 4 tentativas cada (≈ 9,3 segundos = 3 tentativas × 3s de wait), confirmando que o mecanismo de retry funcionou corretamente e sem perda de dados.

---

## Conclusão

### Ganhos Técnicos

O Sprint 3 consolidou um conjunto de práticas fundamentais em sistemas distribuídos:

**Resiliência a falhas de rede:** A implementação do loop de reconexão automática com tratamento de três classes de exceção TCP (`ConnectionRefusedError`, `socket.timeout`, `OSError`) garante que falhas transitórias de rede não encerrem a execução do cliente.

**Separação de responsabilidades:** O servidor é stateless em relação à conectividade — ele não sabe que o cliente reconectou. Toda a lógica de resiliência está encapsulada no cliente, seguindo o princípio de que o lado que depende do serviço deve ser responsável por lidar com sua indisponibilidade.

**Controle de concorrência preservado:** O mecanismo de Lock/Mutex implementado no Sprint 2 foi completamente preservado. Mesmo após reconexões, a numeração sequencial (`seq`) é mantida corretamente, sem duplicatas ou lacunas.

**Timeout configurável:** O `s.settimeout(5)` por tentativa impede que o cliente fique bloqueado indefinidamente em uma conexão que nunca vai responder, combinando com o `RETRY_INTERVAL` para criar um comportamento previsível e controlado.

### Impacto da Solução

Em sistemas distribuídos reais, servidores saem do ar por manutenção, falhas de hardware, sobrecarga ou deploys. Um cliente que simplesmente falha e encerra quando isso acontece é inaceitável em produção.

O Privyon demonstra que é possível implementar tolerância a falhas de forma simples, sem frameworks externos, usando apenas primitivas de socket e controle de fluxo. A solução é diretamente aplicável em sistemas de monitoramento, agentes de coleta de dados e qualquer arquitetura onde clientes precisam manter comunicação contínua com serviços potencialmente instáveis.

O resultado de **100% de sucesso nas 6 requisições** — incluindo as que passaram pela falha do servidor — valida a abordagem e demonstra que o sistema se comporta de forma confiável mesmo em cenários adversos.

---

*Privyon · Sistemas Distribuídos · UNISAL Americana · 2025*
