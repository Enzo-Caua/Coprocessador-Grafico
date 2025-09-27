# Sistema de Redimensionamento de Imagens em FPGA

## Sumário
- [Softwares Utilizados](#softwares-utilizados)
- [Hardwares Utilizados](#hardwares-utilizados)
- [Instalação e Configuração do Ambiente](#instalação-e-configuração-do-ambiente)
- [Especificações do Projeto](#especificações-do-projeto)
- [1. Introdução](#1-introdução)
  - [1.1 Requisitos](#11-requisitos)
- [2. Fundamentação Teórica](#2-fundamentação-teórica)
  - [2.1 Fundamentação Matemática](#21-fundamentação-matemática)
  - [2.2 Algoritmos de Ampliação](#22-algoritmos-de-ampliação)
  - [2.3 Algoritmos de Redução](#23-algoritmos-de-redução)
  - [2.4 Pipeline](#24-pipeline)
- [3. Arquitetura e Implementação](#3-arquitetura-e-implementação)
  - [3.1 Organização da Arquitetura](#31-organização-da-arquitetura)
  - [3.2 Estratégias de Implementação](#32-estratégias-de-implementação)
- [4. Testes e Erros](#4-testes-e-erros)
  - [4.1 Testes dos Algoritmos de Zoom](#41-testes-dos-algoritmos-de-zoom)
  - [4.2 Testes de Contadores e Fluxo de Dados](#42-testes-de-contadores-e-fluxo-de-dados)
  - [4.3 Testes de Estabilidade](#43-testes-de-estabilidade)
  - [4.4 Erros e Problemas Identificados](#44-erros-e-problemas-identificados)
- [5. Resultados e Conclusões](#5-resultados-e-conclusões)
- [Autores](#autores)

---

## Especificações do Projeto
- **Tipo de imagem:** Somente imagens em formato **.mif** são aceitas.  
- **Tamanho do zoom:** Fatores de zoom em passo de **2x**, limitados **até 4x**.
- **Controle do zoom:** Selecionado **através** das chaves disponíveis no kit de desenvolvimento.

## Softwares Utilizados
- **Quartus Prime Lite 23.1** – utilizado para **síntese**, compilação e programação da FPGA.  

## Hardwares Utilizados
- **Kit de Desenvolvimento DE1-SoC** com FPGA Intel Cyclone V (5CSEMA5F31C6).  
- **Monitor VGA** com resolução nativa de 640x480 @ 60 Hz.  
- **Computador host** para compilação e programação da FPGA via USB-Blaster II.  

---

## Instalação e Configuração do Ambiente
1. **Instalar o Quartus Prime Lite 23.1** no computador host.  
2. **Criar um novo projeto no Quartus** selecionando o dispositivo *Cyclone V 5CSEMA5F31C6* (DE1-SoC).  
3. **Adicionar os arquivos Verilog** correspondentes aos módulos do sistema (zoom in, zoom out, controlador, driver VGA, etc.).  
4. **Configurar a ROM de entrada**: gerar arquivo `.mif` com a imagem de teste (formato monocromático 8 bits).  
5. **Compilar o projeto** no Quartus e verificar a **ausência** de erros.  
6. **Programar a FPGA** via USB-Blaster II selecionando o arquivo `.sof` gerado.  
7. **Conectar a saída VGA** da placa ao monitor para visualizar os resultados em tempo real.  

---

# 1. Introdução

O projeto propõe o desenvolvimento de um sistema para redimensionamento de imagens em tempo real. A tarefa envolve ampliar ou reduzir imagens em escala de cinza, mantendo a **eficiência** e a qualidade dentro das limitações do hardware.

## 1.1 Requisitos
- Desenvolvimento integralmente em linguagem Verilog, utilizando apenas os componentes disponíveis na placa DE1-SoC.
- Implementação de algoritmos para redimensionamento de imagens, considerando passos de 2X.
- Para ampliação (zoom in), permitir a aplicação dos algoritmos **Vizinho Mais Próximo** e **Replicação de Pixel**.
- Para redução (zoom out), permitir a aplicação dos algoritmos **Decimação/Amostragem** e **Média de Blocos**.
- As imagens são representadas em escala de cinza e cada elemento da imagem (pixel) deverá ser representado por um número inteiro de 8 bits.

# 2. Fundamentação Teórica

## 2.1 Fundamentação Matemática
O funcionamento dos algoritmos de redimensionamento pode ser entendido como um **mapeamento de índices** entre a imagem original e a transformada.

* **Zoom In (ampliação):** cada novo pixel precisa ser associado a uma posição existente na imagem original. Esse mapeamento pode ser feito por **aproximação do índice mais próximo** (vizinho mais próximo) ou pela **replicação direta de valores** em blocos (replicação de pixel). Em ambos os casos, não há criação de novos valores, apenas duplicação ou arredondamento de posições.

* **Zoom Out (redução):** múltiplos pixels da imagem original precisam ser condensados em um único valor. Isso pode ser feito por **seleção direta** (decimação, escolhendo apenas um dos pixels) ou por **cálculo de uma média aritmética** (média de blocos).

Do ponto de vista matemático, todas as operações envolvem apenas **cópia, soma e divisões simples de inteiros de 8 bits**, o que torna sua implementação altamente eficiente em FPGA, já que não exige operações complexas de ponto flutuante.

## 2.2 Algoritmos de Ampliação

*Vizinho Mais Próximo (Nearest Neighbor):* Amplia a imagem duplicando os pixels mais próximos. É simples e rápido, mas pode gerar bordas serrilhadas e perda de suavidade.

*Replicação de Pixel (Pixel Replication):* Cada pixel original é replicado em um bloco proporcional ao fator de escala (ex.: 2x2), aumentando a resolução. Preserva a nitidez dos pixels, mas mantém **aparência “pixelizada”**.

## 2.3 Algoritmos de Redução

*Decimação (Zoom Out Nearest Neighbor):* Reduz a imagem descartando pixels e mantendo apenas alguns pontos de referência. É eficiente, mas pode causar perda significativa de detalhes e **aliasing**.

*Média de Blocos (Block Averaging):* Reduz a imagem calculando a média de **um bloco** de pixels proporcional ao fator de escala (ex.: 2x2). Gera resultado mais suave e com menos **aliasing**, preservando melhor os detalhes em comparação à decimação.

## 2.4 Pipeline
O pipeline é a técnica de dividir uma tarefa complexa em estágios menores, que funcionam de forma paralela. Com isso, vários dados podem ser processados ao mesmo tempo em diferentes etapas, mesmo que a latência de cada operação individual não seja reduzida.

No contexto deste projeto, o **pipeline** foi utilizado para garantir o sincronismo e o cálculo correto dos pixels em todos os algoritmos de redimensionamento. Ao dividir o processamento em estágios, torna-se possível iniciar o cálculo de novos pixels enquanto outros ainda estão em execução. Dessa forma, o sistema consegue produzir um pixel por ciclo após o preenchimento inicial do pipeline, mantendo a compatibilidade com a taxa de atualização do padrão VGA.

# 3. Arquitetura e Implementação

O sistema foi desenvolvido como um **coprocessador** gráfico capaz de executar operações de redimensionamento de imagens na placa DE1-SoC. Sua arquitetura é organizada em módulos independentes, interligados por um controlador principal responsável por coordenar os sinais de controle, selecionar os algoritmos e direcionar o fluxo de dados.

## 3.1 Organização da Arquitetura 

A arquitetura se apoia em duas fases principais: **processamento** e **exibição**.
Na fase de processamento, a imagem é sempre lida da ROM e escrita na RAM, seja em sua forma original ou após a aplicação do algoritmo de zoom selecionado. A RAM atua como memória central do sistema, simplificando o fluxo de dados e garantindo que todas as versões da imagem estejam acessíveis em um único local. Na fase de exibição, o driver VGA acessa exclusivamente a RAM de forma sequencial, convertendo os dados armazenados em sinais compatíveis com monitores padrão. O divisor de frequência assegura que o clock do sistema esteja sincronizado com as exigências temporais do padrão VGA.

A interação do usuário com o sistema foi projetada para ser intuitiva e direta. As chaves da placa DE1-SoC são mapeadas de forma lógica: um conjunto de chaves seleciona o algoritmo desejado, enquanto outras chaves determinam o fator de zoom (1x, 2x ou 4x). Esta combinação permite ao usuário explorar todas as funcionalidades do sistema de forma simples. O sistema detecta automaticamente mudanças na configuração das chaves e reprocessa a imagem imediatamente, permitindo comparações em tempo real entre diferentes algoritmos e fatores de zoom.

O fluxo de dados segue uma sequência bem definida que começa com a detecção de mudanças na configuração das chaves. Quando uma nova configuração é detectada, o sistema reseta todos os contadores e inicia a fase de processamento. A imagem original na ROM é acessada pixel por pixel, seguindo um padrão de varredura linha por linha, similar ao que um monitor faz ao desenhar uma imagem.

Para cada pixel de saída desejado, o seletor de algoritmo calcula quais pixels da imagem original devem ser lidos. No caso de algoritmos simples como Nearest Neighbor, apenas um pixel é lido. Para a média de blocos, até 16 pixels podem ser lidos em sequência, somados e divididos para produzir um único pixel de saída. O resultado é então armazenado na posição correta da RAM, seguindo o padrão de endereçamento linear que facilita a posterior leitura pelo VGA.

Após processar todos os pixels necessários, o sistema automaticamente **transiciona** para a fase de exibição, onde a RAM é lida sequencialmente sincronizada com as demandas do driver VGA. Este fluxo garante que não haja conflitos de acesso à memória e que a saída VGA sempre tenha dados válidos disponíveis.

## 3.2 Estratégias de Implementação

Na implementação, o processamento foi estruturado de forma a garantir desempenho e sincronismo. Durante a fase de preenchimento da RAM, cada pixel da imagem original é lido da ROM, processado pelo algoritmo de zoom selecionado e gravado em sua posição correspondente na RAM. Na fase seguinte, a RAM é lida sequencialmente pelo VGA, assegurando uma saída estável e contínua.

A imagem original é armazenada em uma ROM de 19.200 posições, enquanto a imagem processada é temporariamente salva em uma RAM de até 307.200 posições antes de ser exibida.

Para lidar com algoritmos mais complexos, como a média de blocos, foi utilizado um **pipeline**, que divide os cálculos em estágios. Isso permite iniciar o processamento de novos pixels enquanto outros ainda estão em execução, garantindo que, após o preenchimento inicial, o sistema mantenha a produção de um pixel por ciclo. O pipeline é controlado por uma máquina de estados expandida, capaz de coordenar a leitura de múltiplos pixels de um bloco (até 16 em um bloco 4x4) e acumular suas contribuições até formar o pixel de saída.

Além disso, o sistema conta com multiplexadores que podem fornecer várias coordenadas de leitura simultaneamente, permitindo flexibilidade na execução de algoritmos distintos sem a necessidade de alterar a arquitetura principal. Essa modularidade facilita futuras expansões e simplifica os testes individuais de cada componente.

O driver VGA funciona como a ponte final do sistema, convertendo os dados processados em sinais compatíveis com monitores padrão. O divisor de frequência garante que o clock do sistema esteja sincronizado com os rigorosos requisitos de temporização VGA. Essa organização modular permite que cada componente seja testado independentemente e facilita futuras melhorias sem comprometer a estabilidade do sistema.

# 4. Testes e Erros

O sistema foi testado de forma prática e sistemática, seguindo uma sequência que permitiu validar tanto a funcionalidade técnica quanto a qualidade visual da saída VGA.

## 4.1 Testes dos Algoritmos de Zoom
Os módulos de zoom foram inicialmente testados individualmente **através** de **waveforms** e usando os **LEDs da placa** para verificar se os valores de entrada e saída estavam corretos. Em seguida, a validação foi realizada no monitor VGA, observando a imagem resultante para identificar possíveis erros de leitura ou escrita.

## 4.2 Testes de Contadores e Fluxo de Dados
Os contadores de altura e largura foram validados seguindo **lógica similar** à dos algoritmos, garantindo que se ajustassem automaticamente aos diferentes tamanhos de imagem gerados pelos algoritmos e fatores de zoom. Mudanças dinâmicas nas configurações das chaves foram detectadas corretamente, reinicializando contadores.

## 4.3 Testes de Estabilidade
O sistema foi submetido a operações contínuas e alternância rápida entre algoritmos e fatores de zoom extremos. Em todos os casos, a saída VGA permaneceu estável, sem artefatos, perda de pixels ou problemas de sincronismo. Testes prolongados confirmaram a robustez da implementação, com ausência de deriva de temporização ou falhas de leitura/escrita.

## 4.4 Erros e Problemas Identificados

Durante o desenvolvimento e testes do sistema, alguns tipos de erro foram observados, principalmente relacionados a desempenho e sincronismo. Entre os principais problemas detectados estão:

- **Velocidade de Processamento Insuficiente:** Quando o algoritmo não conseguia processar os pixels em tempo hábil, a saída VGA apresentava atrasos ou atualização incompleta da imagem, gerando efeitos visuais como linhas cortando a imagem ou múltiplas imagens sobrepostas.

- **Leitura ou Escrita Lenta na Memória:** Acesso à RAM ou ROM com latência excessiva resultava em pixels coloridos incorretos, brilho reduzido ou achatamento da imagem, devido à perda de dados durante a transferência.

- **Erros de Sincronismo:** Problemas de alinhamento entre contadores, pipeline e clock do VGA causavam instabilidade visual, incluindo artefatos gráficos, múltiplas imagens exibidas simultaneamente ou linhas horizontais/verticais inesperadas.

- **Cálculos Incorretos ou Sem Limitadores:** Falta de controle nos valores calculados pelos algoritmos podia gerar saturação ou subtração de pixels, causando alterações de cor erradas, baixa luminosidade ou distorção da imagem.

Todos esses erros foram identificados e corrigidos durante o desenvolvimento. O pipeline foi ajustado, os contadores adaptativos foram calibrados e o controle de acesso à memória foi reforçado, garantindo que cada pixel fosse processado e exibido corretamente. Como resultado, o sistema passou a funcionar de forma estável, sem artefatos visuais ou distorções.

# 5. Resultados e Conclusões

O sistema desenvolvido conseguiu atender a todos os objetivos propostos, demonstrando eficácia na execução de operações de redimensionamento de imagens em tempo real na FPGA DE1-SoC.

## 5.1 Resultados Obtidos
- **Zoom In:** Os algoritmos de vizinho mais próximo e replicação de pixels ampliaram corretamente imagens 160x120 para 320x240 e 640x480 pixels, preservando detalhes e apresentando a pixelização esperada.  
- **Zoom Out:** A decimação e a média de blocos reduziram as imagens de forma precisa, mantendo elementos principais reconhecíveis. A média de blocos proporcionou suavidade superior e menor **aliasing** em relação à decimação simples.  
- **Pipeline e Contadores:** O pipeline permitiu processar múltiplos pixels simultaneamente, garantindo um pixel por ciclo após o preenchimento inicial. Contadores adaptativos asseguraram o alinhamento correto entre processamento e exibição, mesmo com diferentes fatores de zoom e algoritmos.  
- **Estabilidade do Sistema:** Testes prolongados e alternância dinâmica entre algoritmos confirmaram a robustez da implementação, sem artefatos visuais, perda de pixels ou falhas de sincronismo.

## 5.2 Conclusões
O projeto demonstrou que é possível implementar um sistema de redimensionamento de imagens eficiente e confiável em FPGA, utilizando apenas operações de inteiros e módulos disponíveis na placa. A arquitetura modular, combinada com o uso de pipeline, contadores adaptativos e memória centralizada (RAM), permitiu que todos os algoritmos fossem executados de forma correta, com saída VGA estável e alta compatibilidade com a taxa de atualização padrão.

A implementação confirmou que problemas críticos como atrasos de processamento, erros de leitura/escrita e falhas de sincronismo podem ser totalmente eliminados com controle rigoroso de pipeline e acesso à memória. Como resultado, o sistema entrega imagens redimensionadas com qualidade visual consistente, mantendo **integridade** e estabilidade mesmo sob condições de operação contínua ou mudanças rápidas de configuração.

O projeto serve como base sólida para futuras expansões, incluindo novos algoritmos de redimensionamento, suporte a cores ou integração com outros sistemas gráficos, demonstrando a viabilidade de processamento de imagens em tempo real em FPGAs de médio porte.

# Autores
- Enzo Cauã da S. Barbosa  
- Jamile Letícia C. da Silva  
- Rafael Sampaio Firmo

Tutoria: Ângelo Amâncio Duarte 
