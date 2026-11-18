# Projeto_SD__cofre_DE2-115


## Diagrama de Estados
![Diagrama de estados](docs/diagrama_estados.png)

## Waveform Simulação
![Waveform Simulação](docs/waveform.png)

## Principais Bugs e Limitações
**Ausência de filtro de debounce mecânico nos botões (KEY[3:1]):** O código implementa um detector de borda síncrono (btn_pos & ~btn_prev) amostrado a 50 MHz (a cada 20 ns), mas não aplica um temporizador de filtragem (tipicamente 10 ms a 20 ms). Na simulação o sinal é ideal, mas na placa física o repique mecânico (bouncing) do botão pode fazer um único clique ser interpretado como dois ou mais pulsos seguidos, pulando números ou posições.

**Senha fixa no código-fonte (1-2-3-4):** Por compatibilidade com o testbench de validação, a senha está gravada como parâmetros fixos (localparam) no hardware, não permitindo que o usuário redefina uma nova senha diretamente pelos botões da placa.

**Tentativas ilimitadas sem bloqueio (lockout):** O sistema não contabiliza falhas consecutivas; após errar a senha e aguardar os 3 segundos de sinalização nos LEDs vermelhos, o usuário pode tentar novamente sem restrição.

**Bloqueio de entradas durante os temporizadores de feedback:** Durante os estados WAIT_GREEN_LED_TIME (5 s) e WAIT_RED_LED_TIME (3 s), os botões de edição e confirmação ficam inativos, sendo possível interromper a contagem apenas pelo reset global (KEY[0]).

## Descrição detalhada de como os requisitos foram implementados
**Visão geral:** Para o SafeCrack Pro, montamos uma máquina de estados com cinco estados: START, CURRENT_POSITION, CHECK_PASSWORD, WAIT_GREEN_LED_TIME e WAIT_RED_LED_TIME. Junto dela, o módulo safecrackpro_fsm tem um detector de borda para os botões e um contador regressivo que controla por quanto tempo os LEDs ficam acesos. O módulo recebe KEY[3:1] pela entrada btn e usa o KEY[0] como reset (rstn). Ele entrega para fora os quatro dígitos, o índice do dígito que está sendo editado e os LEDs verdes e vermelhos. Já a conversão dos dígitos para os displays de 7 segmentos fica no top-level.

**Os quatro dígitos e os displays:** A senha tem quatro dígitos, cada um guardado em 4 bits no array digits. O primeiro dígito aparece no HEX3 e o quarto no HEX0. Para saber qual dígito está sendo editado, usamos o registrador current_position_idx, que vai de 0 a 3. O top-level mostra esse índice no HEX4 como um número de 1 a 4, e é por ele que o usuário sabe em que posição está. Os dígitos nunca passam de 9 nem ficam abaixo de 0, porque tratamos o wrap-around de forma explícita.

**Estado inicial e reset (KEY[0]):** Todos os registradores do módulo têm reset assíncrono. Então, ao apertar o KEY[0], em qualquer momento, tudo volta para o começo na hora: os quatro dígitos viram 0, o primeiro dígito fica ativo e os LEDs se apagam. Isso vale tanto no meio da digitação quanto com os LEDs acesos. O estado START serve exatamente como esse ponto de partida: ele zera tudo de novo por segurança e, no ciclo seguinte, passa para CURRENT_POSITION, onde o usuário começa a digitar. Quando o tempo dos LEDs acaba, a FSM volta para o START pelo mesmo caminho.

**Um clique, uma ação:** Os botões da DE2-115 são ativos em nível baixo, então a primeira coisa que fazemos é inverter o sinal, para que “1” signifique “pressionado”. Depois, guardamos o valor do botão no ciclo anterior (btn_prev) e comparamos com o valor atual. Só consideramos que houve um clique quando o botão estava solto e agora está pressionado, ou seja, na borda do pressionamento. Se o usuário continuar segurando o botão, essa condição deixa de ser verdadeira e nada mais acontece. Por isso, mesmo segurando um botão por vários segundos, só uma ação é executada.

Os botões ficam assim: KEY[1] confirma, KEY[2] incrementa e KEY[3] decrementa. Se dois botões forem apertados exatamente no mesmo ciclo de clock, a FSM atende só um deles, com prioridade para o incremento, depois o decremento e por último a confirmação.

**Mudando o valor do dígito:** Com a FSM em CURRENT_POSITION, o KEY[2] soma 1 ao dígito ativo, e se ele estava em 9, volta para 0. O KEY[3] subtrai 1, e se ele estava em 0, vai para 9. Só o dígito ativo é alterado, e os outros três continuam como estavam.

**Confirmando e avançando:** Quando o usuário aperta o KEY[1], o valor do dígito ativo fica fixado e o índice avança para o dígito seguinte. Se aquele já era o quarto dígito, a FSM vai para a verificação da senha. Em nenhum ponto o índice volta para trás, então não dá para corrigir um dígito já confirmado. Para recomeçar a inserção, o usuário precisa apertar o reset, como pede o enunciado.

**Verificando a senha:** O estado CHECK_PASSWORD dura apenas um ciclo de clock. Nele, a FSM compara os quatro dígitos digitados com a senha correta, que está definida no código pelos parâmetros PWD_0 a PWD_3 (a senha é 1-2-3-4). Se bater, o contador é carregado com o valor de 5 segundos e a FSM vai para WAIT_GREEN_LED_TIME. Se não bater, o contador recebe o valor de 3 segundos e a FSM vai para WAIT_RED_LED_TIME.

**LEDs e volta ao início:** Com a senha certa, acendemos os 9 LEDs verdes da placa ao mesmo tempo. Com a senha errada, acendemos os 18 LEDs vermelhos. Em ambos os casos, o contador diminui a cada ciclo do clock de 50 MHz, começando em 250.000.000 para os 5 segundos e em 150.000.000 para os 3 segundos. Quando ele chega a zero, a FSM zera os dígitos e o índice e volta ao START, com os displays em 0 e o primeiro dígito ativo, pronta para uma nova tentativa.

Enquanto os LEDs estão acesos, os botões KEY[1], KEY[2] e KEY[3] não fazem nada, porque esses estados simplesmente não olham para eles. A única forma de interromper a contagem antes do fim é apertar o reset.
