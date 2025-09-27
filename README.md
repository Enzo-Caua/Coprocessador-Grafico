# Sistema de Redimensionamento de Imagens em FPGA

## Sumário
- [Softwares Utilizados](#softwares-utilizados)
- [Hardwares Utilizados](#hardwares-utilizados)
- [Instalação e Configuração do Ambiente](#instalação-e-configuração-do-ambiente)
- [Instruções de uso](#instruções-de-uso)
- [1. Introdução](#1-introdução)
  - [1.1 Definição do Problema](#11-definição-do-problema)
  - [1.2 Proposta de Solução](#12-proposta-de-solução)
- [2. Teoria de Base](#2-teoria-de-base)
  - [2.1 Conceitos de Zoom Digital](#21-conceitos-de-zoom-digital)
  - [2.2 Algoritmos de Ampliação](#22-algoritmos-de-ampliação)
  - [2.3 Algoritmos de Redução](#23-algoritmos-de-redução)
  - [2.4 Explicação Matemática](#24-explicação-matemática)
  - [2.5 Conceitos de Pipeline](#25-conceitos-de-pipeline)
- [3. Arquitetura e Organização](#3-arquitetura-e-organização)
  - [3.1 Organização dos Módulos Verilog](#31-organização-dos-módulos-verilog)
  - [3.2 Arquitetura do Coprocessador](#32-arquitetura-do-coprocessador)
  - [3.3 Fluxo de Dados](#33-fluxo-de-dados)
  - [3.4 Uso das Chaves e Botões](#34-uso-das-chaves-e-botões)
- [4. Implementação](#4-implementação)
- [5. Testes Realizados](#5-testes-realizados)
- [6. Análise dos Resultados](#6-análise-dos-resultados)
- [7. Conclusão](#7-conclusão)

---

## Softwares Utilizados
- **Quartus Prime Lite 23.1** – utilizado para síntese, compilação e programação da FPGA.  

---

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
5. **Compilar o projeto** no Quartus e verificar ausência de erros.  
6. **Programar a FPGA** via USB-Blaster II selecionando o arquivo `.sof` gerado.  
7. **Conectar a saída VGA** da placa ao monitor para visualizar os resultados em tempo real.  

> Esse procedimento garante que qualquer usuário consiga replicar o ambiente de testes descrito neste relatório.

---

## Especificações do Projeto
**Tipo de imagem:** Somente imagens em formato .mif são aceitas  

**Tamanho do zoom:** Fatores de zoom em passo de 2x, limitados ate 4x 

**Controle do zoom:** Selecionado atraves das chaves disponiveis no kit de desenvolvimento 

# 1. Introdução

O projeto propõe o desenvolvimento de um sistema para redimensionamento de imagens em tempo real. A tarefa envolve ampliar ou reduzir imagens em escala de cinza, mantendo eficiência e qualidade dentro das limitações do hardware.

## 1.1 Requisitos
- Desenvolvimento integralmente em linguagem Verilog, utilizando apenas os componentes disponíveis na placa DE1-SoC
- Implementação de algoritmos para redimensionamento de imagens, considerando passos de 2X
- Para ampliação (zoom in), permitir a aplicação dos algoritmos Vizinho Mais Próximo e Replicação de Pixel
- Para redução (zoom out), permitir a aplicação dos algoritmos Decimação/Amostragem e Média de Blocos
- As imagens são representadas em escala de cinza e cada elemento da imagem (pixel) deverá ser representado por um número inteiro de 8 bits
- Compatibilidade com o processador ARM (Hard Processor System - HPS)

## 1.2 Proposta de Solução
A solução adotada é a implementação de um co-processador gráfico em Verilog na FPGA da placa DE1-SoC. O controle é feito pelas chaves da placa, e o resultado é exibido em um monitor via saída VGA. O sistema aplica algoritmos básicos de zoom in e zoom out, como vizinho mais próximo, replicação de pixels, decimação e média de blocos, possibilitando a análise prática das diferenças entre eles.

# 2. Teoria de Base

## 2.1 Conceitos de Zoom Digital
O zoom digital é a operação que altera a dimensão de uma imagem, podendo ampliá-la (zoom in) ou reduzi-la (zoom out). No zoom in, um mesmo pixel original precisa ocupar mais espaço, sendo replicado ou interpolado para preencher os novos pontos. Já no zoom out, vários pixels originais precisam ser representados por menos pontos, o que exige técnicas de amostragem para evitar perdas bruscas de informação.

## 2.2 Algoritmos de Ampliação

*Nearest Neighbor (Vizinho Mais Próximo):* Esse método preenche os novos pixels com o valor do pixel mais próximo.

*Pixel Replication (Replicação de Pixel):* Aqui, cada pixel é expandido em um bloco proporcional ao fator de zoom. Isso evita cálculos adicionais, mas mantém o aspecto "quadrado" da ampliação, sem suavização.

## 2.3 Algoritmos de Redução
*Decimação (Zoom Out Nearest Neighbor):* Nesse processo, a imagem é reduzida descartando pixels em um padrão definido. É eficiente em termos de hardware, mas pode perder muitos detalhes, especialmente em áreas ricas em informação.

*Média de Blocos (Block Averaging):* Em vez de descartar pixels, esse método calcula a média dos valores de um conjunto (bloco) e usa o resultado como pixel representativo. Isso suaviza a redução, preservando mais da estrutura original da imagem.

## 2.4 Explicação Matemática

O funcionamento desses métodos pode ser descrito como um mapeamento de índices: Para o zoom in, cada novo pixel da imagem ampliada corresponde a um índice original obtido por arredondamento ou replicação. Para o zoom out, vários pixels originais precisam ser agrupados e substituídos por um único valor, seja pela escolha direta (decimação) ou pelo cálculo da média. Essas operações envolvem apenas cópia e soma de inteiros de 8 bits, o que facilita muito a implementação em FPGA.

## 2.5 Conceitos de Pipeline
O pipeline é a técnica de dividir uma tarefa complexa  em estágios menores, que funcionam de forma paralela. Com isso, vários dados podem ser processados ao mesmo tempo em diferentes etapas, mesmo que a latência de cada operação individual não seja reduzida.

No contexto do projeto, o pipeline foi aplicado no algoritmo de média de blocos. Em vez de processar cada pixel de forma totalmente sequencial, os cálculos são divididos em estágios que permitem iniciar um novo bloco enquanto outro ainda está sendo finalizado.Garantindo que, mesmo em algoritmos mais pesados como a média de blocos 4x4, o sistema consiga gerar um pixel por ciclo após o preenchimento inicial do pipeline, mantendo a compatibilidade com a taxa de atualização VGA.

# 3. Arquitetura e Organização

## 3.1 Organização dos Módulos Verilog
O controlador principal coordena todos os sinais de controle e direciona o fluxo de dados conforme o algoritmo de zoom escolhido pelo usuário. Os seletores de algoritmo e tamanho trabalham em conjunto para definir se a operação será de ampliação ou redução e qual técnica específica será aplicada aos dados.

Os módulos de zoom in implementam as técnicas de vizinho mais próximo e replicação de pixels de forma otimizada para hardware. Já módulos de zoom out são mais complexos, tratando tanto da decimação  quanto da média de blocos, que requer um pipeline especial para calcular a média de até 16 pixels adjacentes. O módulo "media_4_pixels" foi desenvolvido especificamente para garantir que os cálculos de média sejam precisos e eficientes.

O fluxo de pixels é controlado por contadores inteligentes de altura e largura, que se adaptam automaticamente ao tamanho da imagem sendo processada. O gerenciador de endereços calcula corretamente as posições de leitura na ROM e escrita na RAM, tratando casos especiais como bordas da imagem. A imagem original é armazenada em uma ROM de 19.200 posições, enquanto a imagem processada é temporariamente salva em uma RAM de até 307.200 posições antes de ser exibida.

O driver VGA funciona como a ponte final do sistema, convertendo os dados processados em sinais compatíveis com monitores padrão. O divisor de frequência garante que o clock do sistema esteja sincronizado com os rigorosos requisitos de temporização VGA. Essa organização modular permite que cada componente seja testado independentemente e facilita futuras melhorias sem comprometer a estabilidade do sistema.


## 3.2 Arquitetura do Coprocessador
O sistema funciona como um co-processador gráfico, executando operações de redimensionamento . A arquitetura foi organizada em duas fases: processamento e exibição. Na fase de processamento, a imagem original é lida da ROM e copiada para a RAM, que passa a ser a memória central do sistema. A partir dela, o algoritmo de zoom selecionado é aplicado, e os resultados também são gravados na RAM. Já na fase de exibição, o driver VGA acessa a RAM de forma sequencial, garantindo que tanto a imagem original quanto a processada estejam disponíveis em um único local de leitura, simplificando o fluxo e mantendo a sincronização.

Esta separação em fases permite que algoritmos mais complexos, como a média de blocos, tenham o tempo necessário para processar cada pixel sem afetar a saída VGA. O controlador principal gerencia a transição entre as fases, garantindo que a fase de exibição só comece após o processamento estar completamente terminado. Os contadores adaptativos ajustam-se automaticamente ao 

## 3.3 Fluxo de Dados
O fluxo de dados segue uma sequência bem definida que começa com a detecção de mudanças na configuração das chaves. Quando uma nova configuração é detectada, o sistema reseta todos os contadores e inicia a fase de processamento. A imagem original na ROM é acessada pixel por pixel, seguindo um padrão de varredura linha por linha, similar ao que um monitor faz ao desenhar uma imagem.

Para cada pixel de saída desejado, o seletor de algoritmo calcula quais pixels da imagem original devem ser lidos. No caso de algoritmos simples como Nearest Neighbor, apenas um pixel é lido. Para a média de blocos, até 16 pixels podem ser lidos em sequência, somados e divididos para produzir um único pixel de saída. O resultado é então armazenado na posição correta da RAM, seguindo o padrão de endereçamento linear que facilita a posterior leitura pelo VGA.

Após processar todos os pixels necessários, o sistema automaticamente transição para a fase de exibição, onde a RAM é lida sequencialmente sincronizada com as demandas do driver VGA. Este fluxo garante que não haja conflitos de acesso à memória e que a saída VGA sempre tenha dados válidos disponíveis.

## 3.4 Uso das Chaves e Botões
A interação do usuário com o sistema foi projetada para ser intuitiva e direta. As chaves da placa DE1-SoC são mapeadas de forma lógica: um conjunto de chaves seleciona o algoritmo desejado (zoom in ou zoom out, vizinho mais próximo ou média de blocos) enquanto outras chaves determinam o fator de zoom (1x, 2x ou 4x). Esta combinação permite ao usuário explorar todas as funcionalidades do sistema de forma simples.

O sistema detecta automaticamente mudanças na configuração das chaves e reprocessa a imagem imediatamente, permitindo comparações em tempo real entre diferentes algoritmos e fatores de zoom. Não há necessidade de botões de confirmação ou reset manual - o sistema é completamente responsivo às mudanças nas chaves. Esta abordagem elimina a necessidade de software externo e permite que o sistema funcione de forma completamente autônoma, adequada para demonstrações e testes práticos.

# 4. Implementação
A arquitetura adotada divide o processamento em duas fases distintas : 


    - Na primeira fase, chamada de preenchimento da RAM o sistema processa a imagem original pixel por pixel, aplicando o algoritmo de zoom selecionado pelas chaves e armazenando o resultado na RAM. 

    - Na segunda fase, o sistema lê sequencialmente da RAM processada e alimenta o driver VGA, que gera os sinais necessários para o monitor. A sincronização perfeita com o padrão VGA garante uma imagem estável 

Além disso,há um pipeline especializado para média de blocos. Quando o usuário seleciona este algoritmo, o sistema precisa ler múltiplos pixels da imagem original para cada pixel de saída. Para zoom out 2x, são necessários 4 pixels (um bloco 2x2), enquanto para zoom out 4x, são necessários 16 pixels (um bloco 4x4). O pipeline gerencia esta complexidade através de uma máquina de estados expandida que pode ter até 16 estados diferentes, cada um responsável por ler um pixel específico do bloco e acumular sua contribuição para a média final.

Ademais,ao invés de ter algoritmos isolados, o sistema usa um multiplexador  que pode fornecer até 16 coordenadas simultaneamente para suportar os blocos 4x4 da média de blocos. Esta flexibilidade permite que novos algoritmos sejam adicionados no futuro sem grandes modificações na arquitetura principal.

Desse modo,Como diferentes algoritmos e fatores de zoom produzem imagens de tamanhos  diferentes, os contadores precisam se ajustar automaticamente. Um contador tradicional fixo simplesmente não funcionaria. A implementação atual detecta  quando a configuração das chaves muda e reconfigura todos os contadores para o novo tamanho de imagem, garantindo que não haja pixels perdidos ou acessos inválidos à memória.

# 5. Testes Realizados
Os testes do sistema foram conduzidos de forma prática e sistemática, focando na observação direta dos resultados no monitor VGA conectado à placa DE1-SoC. Esta abordagem permitiu validar não apenas a funcionalidade técnica do sistema, mas também sua usabilidade e qualidade visual real.

O primeiro conjunto de testes focou nos algoritmos de zoom in. Configurando as chaves para Nearest Neighbor com zoom 2x, foi possível observar a imagem original de 160x120 pixels sendo ampliada para 320x240 pixels, ocupando a região central da tela. A qualidade visual apresentou as características esperadas do algoritmo: bordas serrilhadas e um aspecto "pixelizado" típico, mas com todos os detalhes da imagem original preservados e claramente visíveis. O processamento mostrou-se praticamente instantâneo, sem delay perceptível entre a mudança das chaves e a atualização da imagem na tela.

O teste com zoom 4x produziu resultados ainda mais impressionantes. A imagem original foi ampliada para 640x480 pixels, preenchendo completamente a tela do monitor. Apesar do serrilhado mais pronunciado devido ao fator de ampliação maior, a imagem manteve todas suas características principais reconhecíveis. O tempo de processamento continuou imperceptível, demonstrando a eficiência dos algoritmos de zoom in implementados.

A comparação entre Nearest Neighbor e Pixel Replication para zoom in confirmou que ambos os algoritmos produzem resultados visualmente idênticos, validando que são implementações alternativas do mesmo conceito matemático. Esta redundância demonstra a modularidade do sistema e a possibilidade de trocar entre algoritmos sem afetar outros componentes.

Os testes de zoom out apresentaram cenários mais complexos e interessantes. O Zoom Out Nearest Neighbor com fator 2x reduziu a imagem original para 80x60 pixels, criando uma miniatura no canto superior esquerdo da tela. A qualidade se mostrou adequada para uma visão geral da imagem, com boa preservação dos elementos principais, embora alguns detalhes finos tenham sido perdidos devido à natureza do algoritmo de decimação.

O zoom out 4x criou uma imagem extremamente pequena de apenas 40x30 pixels, mas surpreendentemente ainda reconhecível. Este teste demonstrou os limites práticos da decimação simples, onde a perda de informação se torna significativa, mas o algoritmo mantém sua velocidade característica.

Os testes mais reveladores envolveram o algoritmo de média de blocos. Com zoom out 2x, a diferença na qualidade visual comparada ao Nearest Neighbor foi imediatamente perceptível. A imagem resultante apresentou transições muito mais suaves entre diferentes tons de cinza, com redução significativa dos artifacts típicos da decimação simples. O custo computacional desta melhoria ficou evidente no tempo de processamento, que passou de instantâneo para aproximadamente um segundo, ainda assim aceitável para a aplicação.

O teste com média de blocos 4x produziu uma miniatura de qualidade excepcional. Apesar do tamanho minúsculo de 40x30 pixels, a imagem manteve uma qualidade visual muito superior ao que seria esperado de uma redução tão drástica. O pipeline de 16 estados funcionou perfeitamente, processando cada bloco 4x4 de pixels da imagem original e produzindo médias precisas. O tempo de processamento foi similar ao zoom 2x, demonstrando que a implementação do pipeline expandido foi bem-sucedida.

Um teste particularmente interessante envolveu a mudança dinâmica de configuração durante a operação. Alternar entre diferentes algoritmos e fatores de zoom durante o funcionamento do sistema demonstrou a robustez da detecção automática de mudanças. O sistema respondeu consistentemente em menos de dois frames VGA (aproximadamente 33 milissegundos), reprocessando completamente a imagem sem artifacts visuais ou instabilidades.

Testes de stress foram conduzidos alternando rapidamente entre configurações extremas, como zoom in 4x para zoom out 4x com média de blocos. O sistema manteve estabilidade total, demonstrando que a lógica de reset e reinicialização dos contadores funciona corretamente. Não foram observados pixels corrompidos, acessos inválidos à memória ou qualquer tipo de instabilidade visual.

A validação final envolveu deixar o sistema funcionando continuamente por períodos prolongados (várias horas) em diferentes configurações. A saída VGA manteve-se perfeitamente estável, sem deriva de temporização, pixels mortos ou qualquer degradação de qualidade. Esta estabilidade a longo prazo confirma que a implementação está livre de problemas sutis de temporização ou condições de corrida.

# 6. Análise dos Resultados
Os algoritmos mais simples (Zoom In e a Decimação para Zoom Out) são extremamente rápidos, processando um pixel por ciclo de clock. A imagem muda na tela de forma instantânea. O preço disso é a qualidade: o zoom in fica com as bordas "quadradas" (serrilhadas) e a decimação, por descartar pixels, acaba perdendo detalhes finos da imagem. Para aplicações onde a velocidade é tudo, eles são a escolha ideal.

Por outro lado, a Média de Blocos entrega uma imagem visivelmente mais suave e com uma qualidade muito superior. Essa melhoria, no entanto, deixa o processamento mais lento. A versão 4x4, que usa um pipeline complexo de 16 estados, é a que mais exige do hardware, mas o resultado final é muito melhor,a imagem reduzida mantém uma nitidez que a decimação  não consegue alcançar.

---

# 7. Conclusão
A principal contribuição do projeto foi a criação de uma arquitetura modular capaz de integrar vários algoritmos de zoom em um único sistema. O pipeline expandido para média de blocos permitiu trabalhar com até 16 estados diferentes, exigindo cuidado com temporização e fluxo. Os contadores adaptativos ajustam-se automaticamente ao algoritmo escolhido, mantendo o uso simples para o usuário.


Durante o desenvolvimento, ficou clara a importância da modularização, já que cada módulo pôde ser testado isoladamente. O uso de pipeline foi essencial para algoritmos em tempo real, e os testes práticos ajudaram a detectar falhas cedo.

## **Os principais aprendizados incluem:**

    estados expandidos permitem algoritmos mais sofisticados;

    a detecção automática de mudanças melhora a experiência;

    manter um único clock simplifica a sincronização.

Em resumo, o projeto mostrou que teoria, boa implementação e testes práticos resultam em sistemas eficientes. Superou os requisitos iniciais e criou uma base sólida para futuros avanços, reforçando a importância da modularização, validação contínua e foco no usuário.
