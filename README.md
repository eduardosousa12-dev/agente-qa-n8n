# 🚀 Projeto: Pipeline de QA Automatizado para Agentes de IA (Meta-Agente)

Este projeto é um pipeline de automação ponta-a-ponta desenhado para resolver um dos maiores gargalos no desenvolvimento de Agentes de IA: **o teste manual, lento e inconsistente**.

Esta ferramenta atua como um "Meta-Agente", um sistema de IAs que testa, avalia e gera relatórios de performance sobre outros agentes, garantindo qualidade e acelerando o ciclo de desenvolvimento.

### 🎬 Demonstração Rápida

**Parte 1: Geração dos Testes e Execução da Simulação: https://www.loom.com/share/9ebde053df9e48d79ae143305faf2299**
**Parte 2: Análise, Relatório e Score Final: https://www.loom.com/share/96794539c87b4d9491396bd65eef76c0**

---

## 1. O Problema (A Dor)

Testar agentes de IA (chatbots, assistentes de vendas, etc.) é um processo complexo:
* **Demorado:** Requer que um humano crie dezenas de cenários e prompts manualmente.
* **Inconsistente:** Um testador humano pode avaliar de forma diferente de outro, ou esquecer de testar "casos extremos" (edge cases).
* **Superficial:** É difícil para um humano simular 20 testes diferentes e depois analisar *padrões de erro* entre todas as conversas.

O resultado é que gastávamos horas em testes manuais para cada pequena alteração, e mesmo assim, bugs passavam para a produção.

## 2. A Solução (O Remédio)

Para resolver isso, construí um **pipeline de 3 estágios** no n8n que gerencia o ciclo de QA de forma 100% autônoma.

O sistema é acionado por um formulário, onde o desenvolvedor apenas fornece o "contexto" do que deve ser testado. A partir daí, o pipeline:
1.  **Cria** uma bateria de testes complexos.
2.  **Executa** cada teste simulando um usuário real.
3.  **Avalia** individualmente cada conversa.
4.  **Consolida** todas as avaliações em um relatório final com score (0-100) e análise de padrões.

**Principais Vantagens:**
* **Economia Drástica de Tempo:** Reduz o tempo de teste de horas para minutos.
* **Consistência Total:** Todos os testes são gerados e avaliados com os mesmos critérios rigorosos.
* **Análise Profunda:** O "Agente Gerente" final identifica padrões de erro que um humano jamais veria analisando individualmente.
* **Recurso de Depuração (Re-teste):** O fluxo permite que o desenvolvedor escolha entre "Gerar novos testes" ou **"Repetir os mesmos testes"**. Isso é crucial para depurar (debugar) um agente, permitindo rodar a mesma bateria de testes após uma correção para verificar se o bug foi resolvido.

---

## 3. Arquitetura da Solução (Como Funciona)

O sistema é composto por 3 workflows independentes no n8n que são acionados em sequência por webhooks.

### Estágio 1: Agente Criador de Testes
Este workflow é o ponto de partida.
1.  **Gatilho:** Um formulário (`On form submission`) ou Webhook. O dev insere o `System prompt` (contexto do teste), a `URL da planilha` e o `Path do webhook` do agente-alvo.
2.  **Seleção de Modo (IF):** O dev escolhe "Novo teste" ou "Repete o mesmo".
    * **Se "Novo teste":** O workflow apaga os dados de *todas* as planilhas (testes antigos e resultados antigos).
    * **Se "Repete o mesmo":** O workflow apaga *apenas* os resultados antigos ("Análises", "Revisão", "Resultado"), preservando os testes já criados na planilha "Testes Detalhados".
3.  **Geração (IA):** (Apenas se for "Novo teste") Um Agente de IA (`Basic LLM Chain` + `Grok-4 Fast`) usa o contexto para gerar a bateria de testes.
4.  **Parse (Python):** Um nó de Código Python (`Divide cada teste`) "quebra" o texto da IA em itens JSON individuais para cada teste.
5.  **Ação:** Salva os novos casos de teste na planilha **"Testes Detalhados"**.
6.  **Próximo Estágio:** Dispara um Webhook (`Envia pra testar`) para acionar o Estágio 2.

### Estágio 2: Agente Testador de LLMs (O Simulador)
Este workflow simula as conversas.
1.  **Gatilho:** Recebe o Webhook do Estágio 1.
2.  **Leitura:** Puxa os casos de teste da planilha **"Testes Detalhados"**.
3.  **Loop:** Para cada caso de teste, ele inicia um loop (`Loop Over Items`).
4.  **Execução (IA + Ferramenta):** Um `AI Agent` (o "Test Runner") simula o usuário. Ele usa uma ferramenta customizada (`agente_principal`) que é um nó de código (JavaScript/Axios) que faz a chamada de API real para o "Agente-Alvo" que está sendo testado (usando o `Path do webhook` fornecido no Estágio 1).
5.  **Memória:** Uma memória (`Postgres Chat Memory`) é usada para manter o contexto da conversa, turno após turno, para cada teste individual.
6.  **Ação:** Ao final de cada teste, o workflow puxa o histórico completo da conversa do Postgres (`Puxa histórico`) e salva o log na planilha **"Análises"**.
7.  **Próximo Estágio:** Ao final de *todos* os testes, dispara um Webhook (`Envia pra revisão`) para acionar o Estágio 3.

### Estágio 3: Agente Revisador (O Gerente de QA)
Este workflow é o cérebro analítico e possui duas fases.
1.  **Gatilho:** Recebe o Webhook do Estágio 2.
2.  **Fase A: Revisão Individual**
    * Lê todos os logs de conversa da planilha **"Análises"**.
    * Entra em um loop (`Loop Over Items`).
    * Para cada conversa, um `AI Agent` (o "Crítico Individual") a avalia e gera um `score` e uma `justificativa`.
    * Salva essa revisão individual na planilha **"Revisão"**.
3.  **Fase B: Relatório Consolidado (Após o loop)**
    * Lê *todas* as revisões individuais da planilha **"Revisão"**.
    * Agrega (`Aggregate3`) todos os dados em um único bloco de texto.
    * Envia este bloco para um `AI Agent` final (o "Gerente de QA"), que usa um modelo poderoso (`Gemini 2.5 Pro`) e um prompt massivo de "Análise Consolidada".
    * Este agente gera o relatório final, calculando um **Score Geral (0-100)**, analisando **Padrões de Erro** e **Padrões de Acerto**.
    * **Resultado Final:** Salva o relatório completo na planilha **"Resultado"**.

---

## 4. Como Usar (Setup Essencial)

Para que este pipeline funcione, o usuário (desenvolvedor) precisa configurar o "alvo" do teste.

1.  **Tenha seu Agente-Alvo:** O seu Agente de IA (o que você quer testar) deve estar rodando e acessível através de um **endpoint de Webhook** (POST).
2.  **Configure a Conexão:** O seu workflow de Agente-Alvo deve:
    * Receber o `chatInput` (a mensagem do usuário) e o `sessionId` do webhook.
    * Conectar esse input ao seu próprio Agente de IA.
    * Retornar a resposta da IA no formato JSON.
3.  **Forneça o Endpoint:** No formulário do **Estágio 1**, você deve fornecer o `Path do webhook` (o final da URL do seu agente-alvo) para que o "Agente Testador" (Estágio 2) saiba para onde enviar as mensagens de teste.

O nó `agente_principal` dentro do Estágio 2 é a ferramenta que faz essa chamada (`axios.post`) para o seu webhook.

---

## 5. Ferramentas Utilizadas (Tech Stack)

* **Orquestração:** **n8n** (workflows, gatilhos, loops)
* **Inteligência (LLMs):**
    * **LangChain Nodes** (`AI Agent`, `Basic LLM Chain`, `Structured Output Parser`)
    * **OpenRouter** (para acesso a múltiplos modelos)
    * **Grok-4 Fast** (para geração rápida de testes e revisão individual)
    * **Google Gemini 2.5 Pro** (para a análise consolidada final)
* **Banco de Dados:**
    * **Google Sheets** (usado como banco de dados de pipeline para mover dados entre os estágios)
    * **PostgreSQL** (usado para a memória de chat (`Postgres Chat Memory`) durante a execução dos testes)
* **Lógica Customizada:**
    * **Python** (no nó `Code` para parsear o output de texto do LLM)
    * **JavaScript** (no nó `ToolCode` para criar a ferramenta `agente_principal` que chama o agente-alvo via `axios`)
* **Comunicação:**
    * **Webhooks** e **HTTP Request** (para acionar os workflows em sequência)

---

## 6. Arquivos do Workflow

* `Agente Criador de Testes.json`: Workflow do Estágio 1 (Geração de Testes).
* `Agente Testador de LLMs.json`: Workflow do Estágio 2 (Execução do Teste).
* `Agente Revisador do teste.json`: Workflow do Estágio 3 (Análise e Relatório).

# Contato
Whatsapp: 5534998557386
Instagram: eduardosousa.12
