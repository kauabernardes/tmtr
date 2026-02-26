# Documentação Técnica do Sistema - II TMTR 2026

## 1. Visão Geral da Arquitetura
O sistema de gerenciamento do II Torneio e Mostra Tecnológica de Robótica (TMTR 2026) possui uma arquitetura centralizada, orientada a eventos e focada na sincronização de dados em tempo real. O módulo da **Mesa** atua como o controlador principal, gerenciando as rotas de exibição e os estados lógicos das telas secundárias (TV e Projeções).

O ecossistema divide-se em três interfaces principais:

* **1. Interface do Juiz (Mobile/Tablet):** Módulo de entrada de dados na arena. Contém componentes de controle de tempo (cronômetros) e de registro de eventos (pontuação e infrações).
* **2. Interface da Mesa (Desktop):** Painel de controle administrativo e auditoria. O operador gerencia o chaveamento de modalidades e os estados de exibição (ex: comutação entre `MODO_PARTIDA` e `MODO_INTERVALO`).
* **3. Telas de Exibição (TV e Projeção no Chão):** Interfaces passivas (somente leitura) que consomem os dados do sistema em tempo real. O layout e as informações exibidas são definidos dinamicamente pelas requisições oriundas da Mesa.

---

## 2. Controle de Estados e Sincronização
As modalidades do torneio ocorrem de forma sequencial. Para evitar conflito de dados, o sistema utiliza um **Controle de Modalidade Ativa** parametrizado pela Mesa. A alteração deste parâmetro atualiza instantaneamente a interface de todos os clientes conectados (Juízes, TV e Projeção).

### Modos de Operação Global

* **Seleção de Categoria:** A Mesa define a constante de ambiente (`SEGUIDOR_DE_LINHA`, `SUMO` ou `FUTEBOL`). As requisições de outras modalidades são temporariamente bloqueadas nas interfaces dos juízes.
* **Estado `MODO_PARTIDA`:**
    * A TV e a Projeção do Chão renderizam os componentes referentes à modalidade ativa (ex: cronômetro e placar simultâneos no caso do Futebol e Sumô).
    * Os dados são espelhados localmente sem sobreposição de informações de competições inativas.
* **Estado `MODO_INTERVALO` (Transição e Pausas):**
    * Durante pausas, a Mesa comuta a exibição da TV para o componente de **Dashboard**, que consome e exibe um carrossel iterativo de tabelas de classificação, chaves (Fase de Grupos) e rankings das modalidades recentes ou em andamento.
    * A Projeção do Chão entra em modo de espera (tela de descanso/logo do TMTR), mitigando distrações visuais na arena durante os reparos de hardware das equipes.

---

## 3. Especificações e Regras por Modalidade

### Módulo A: Robô Seguidor de Linha
A avaliação baseia-se na execução de um trajeto autônomo. O sistema deve focar no registro preciso de tempo (timestamps) e na somatória de pontuações de percurso.

* **Fluxo de Execução:** Cada equipe possui o limite de 3 (três) rodadas totais. A Mesa inicializa a rodada, espelhando o relógio nas telas de TV e Projeção. O juiz aciona o gatilho de "Start", sincronizando o cronômetro em todos os endpoints.
* **Condições de Interrupção:** O sistema deve permitir a parada imediata do tempo pelo juiz caso o robô saia completamente da linha. Se o robô permanecer inativo por mais de 10 segundos, o juiz registrará o encerramento antecipado da rodada, com interrupção do tempo.
* **Cômputo de Pontuação:** Após a interrupção do relógio, o juiz seleciona os obstáculos concluídos. O backend processa a somatória de acordo com a tabela de pesos (ex: Linha reta = 5 pontos, Mudança de direção de 90° = 20 pontos, Zigue-Zague = 50 pontos).
* **Ranqueamento:** O ranking na TV é ordenado priorizando a maior pontuação no menor tempo possível. Em caso de empate por pontuação, o critério do menor tempo será o desempate.

### Módulo B: Sumô de Robôs
Partidas de combate em um Dojô, exigindo atualização de estado em baixa latência para os eventos da arena.

* **Fluxo da Partida:** As partidas são disputadas em até 3 rodadas. A duração máxima de uma rodada é restrita a 1 minuto.
* **Pontuação (Yukô) e Condição de Vitória:** O juiz possui controles para incrementar pontos de Yukô (ex: registrar quando um robô desloca o oponente para fora, ou quando o oponente se move para fora por conta própria ). A partida é encerrada com vitória assim que uma equipe atinge 2 pontos de Yukô.
* **Tratamento de Penalidades:** O juiz deve registrar infrações, como inatividade superior a 10 segundos, que encerra a rodada imediatamente. O sistema implementa um gatilho automático: a cada acúmulo de 2 penalidades pela mesma equipe ao longo da partida, é creditado 1 ponto de Yukô à equipe adversária.
* **Fase de Grupos:** As equipes são ordenadas pelo número de partidas vencidas. O algoritmo de desempate segue a hierarquia: confronto direto (quando envolver exatamente duas equipes), seguido pelo menor número total de penalidades acumuladas, e por fim, sorteio. As duas equipes melhor classificadas de cada grupo avançam para a Fase Final.

### Módulo C: Futebol de Robôs
Partidas teleguiadas em dinâmica de tempo corrido, requerendo o gerenciamento de placares, saldo de gols e controle de faltas.

* **Fluxo da Partida e Temporização:** A partida consiste em dois tempos regulamentares de 3 minutos cada, separados por um intervalo máximo de 2 minutos. TV, Projeção e dispositivos dos Juízes mantêm cronômetros e placares sincronizados.
* **Controle de Faltas e Pênaltis:** O juiz efetua os registros de faltas (ex: inatividade por mais de 10 segundos)[cite: 528]. [cite_start]O sistema conta com um contador de advertências por partida; ao registrar 2 faltas de uma equipe, o sistema alerta visualmente para uma cobrança de pênalti[cite: 530].
* **Classificação da Fase de Grupos:** O módulo atualiza a tabela injetando 3 pontos para cada Vitória, 1 ponto para Empate e 0 pontos para Derrota.
* **Critérios de Desempate (Tabela):** Na constatação de empates em pontos no grupo, o sistema ordena a tabela baseando-se no Saldo de Gols (SG), seguido por maior número de gols marcados e confronto direto. As duas equipes melhor classificadas de cada grupo avançarão para a fase eliminatória.
