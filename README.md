# Pipeline de Segmentação e Contagem de Células

## Visão Geral

Este projeto implementa um fluxo automatizado de processamento digital de imagens para análise de células em microscopia fluorescente. O desafio central é lidar com características típicas de imagens biomédicas: ruído de sensor, variação de iluminação de fundo e células que se tocam ou sobrepõem.

O trabalho divide-se em quatro etapas principais:

1. **Melhoria de Contraste:** Aplicação de equalização de histograma para corrigir iluminação desigual.
2. **Filtragem de Ruído:** Comparação entre filtros espaciais (mediana e gaussiano) e filtragem em frequência (FFT), com avaliação quantitativa via MSE e PSNR.
3. **Segmentação:** Uso de operadores de borda (Sobel e Laplaciano), operações morfológicas e algoritmo watershed para separar células adjacentes.
4. **Contagem e Validação:** Quantificação das células detectadas e comparação com o gabarito do dataset para determinar a melhor combinação de técnicas.

## Estrutura de Diretórios

Abaixo está a organização atual do diretório do projeto, refletindo os dados brutos baixados, os dados processados gerados e os scripts de desenvolvimento:

```text
.
├── data
│   ├── raw
│   │   ├── BBBC020_v1_images            # Imagens originais (Canais separados e mesclados)
│   │   ├── BBC020_v1_outlines_cells     # Arquivos TIF individuais com contornos das células
│   │   └── BBC020_v1_outlines_nuclei    # Arquivos TIF individuais com contornos dos núcleos
│   └── processed
│       ├── gt_cells                     # Máscaras de gabarito consolidadas para células (.npy)
│       └── gt_nuclei                    # Máscaras de gabarito consolidadas para núcleos (.npy)
└── notebooks
    ├── 01_eda_preparacao.ipynb          # Exploração e preparação de dados
    ├── 02_filtragem_analise.ipynb       # Comparação de técnicas de filtragem
    └── 03_segmentacao_contagem.ipynb    # Segmentação e validação de contagem

```

## Sobre os Dados

O projeto utiliza o dataset biomédico **BBBC020** (Murine bone-marrow derived macrophages).

* **Características das Imagens:** O conjunto de dados consiste em 25 imagens de macrófagos, disponibilizadas em canais separados.
* **Canal `_c1` (CD11b/APC):** Marca a superfície e o citoplasma da célula.
* **Canal `_c5` (DAPI):** Marca os núcleos.
* **Canal `_(c1+c5)`:** Imagem mesclada de ambos os marcadores para visualização de referência.

* **Desafios do Domínio:** As células apresentam formato irregular e frequentemente se tocam ou sobrepõem. As imagens também contêm ruído natural de sensor e variações de iluminação, exigindo tratamento adequado de pré-processamento.
* **Ground Truth:** O dataset fornece arquivos `.TIF` individuais contendo apenas a linha de contorno (*outline*) para cada célula e cada núcleo identificados. *Nota: Conforme a documentação oficial, os arquivos de gabarito são ausentes/incompletos para algumas das imagens originais.*

## Notebook 01: Análise Exploratória e Preparação dos Dados (`01_eda_preparacao.ipynb`)

Este notebook é responsável por organizar os arquivos brutos e construir as bases que permitirão a validação do algoritmo de contagem no final do projeto. Suas principais funções são:

1. **Mapeamento e Limpeza:** Varredura automática do diretório `raw` para relacionar cada imagem aos seus respectivos arquivos de canais (`c1`, `c5` e mesclado), ignorando arquivos de sistema indesejados que vieram no dataset.
2. **Visualização Inicial:** Inspeção das características visuais dos diferentes canais das imagens usando OpenCV e Matplotlib.
3. **Consolidação do Gabarito (Ground Truth):** Leitura de todos os pequenos arquivos individuais de contorno de uma mesma amostra, preenchimento de suas áreas internas e agregação em uma **única máscara rotulada** (onde cada região conectada recebe um ID numérico único).
4. **Exportação Processada:** Processamento em lote de todo o dataset, ignorando automaticamente as imagens que não possuem gabarito no repositório oficial, e salvamento das máscaras unificadas em formato `.npy` dentro de `data/processed/` para uso nos notebooks de validação.

## Notebook 02: Análise e Comparação de Técnicas de Filtragem (`02_filtragem_analise.ipynb`)

Este notebook concentra-se na avaliação comparativa de estratégias para redução de ruído e melhoria de contraste. Suas etapas principais são:

1. **Pré-processamento Inicial:** Aplicação de equalização de histograma para normalizar a iluminação de fundo em todas as imagens.
2. **Filtragem Espacial:** Implementação e teste de filtros passa-baixas no domínio espacial: filtro de mediana e filtro gaussiano com diferentes tamanhos de kernel. Avaliação visual do suavização versus perda de detalhe.
3. **Filtragem em Frequência:** Transformação das imagens para o domínio da frequência via FFT, aplicação de máscara passa-baixas e transformação inversa. Comparação com abordagens espaciais.
4. **Métricas Quantitativas:** Cálculo de MSE (Erro Quadrático Médio) e PSNR (Razão Sinal-Ruído de Pico) para cada técnica em relação à imagem original, permitindo classificação objetiva do desempenho.
5. **Análise de Impacto:** Visualização lado-a-lado dos resultados de cada filtro e seu efeito subsequente na detecção de bordas, evidenciando quais técnicas preservam melhor as características das células.
6. **Exportação de Resultados:** Salvamento das imagens filtradas e das métricas comparativas para referência e uso no notebook seguinte.

## Notebook 03: Segmentação, Contagem e Validação (`03_segmentacao_contagem.ipynb`)

Este notebook finaliza o pipeline aplicando as melhores estratégias de filtragem identificadas no notebook anterior e executando a segmentação completa com validação. Suas etapas são:

1. **Seleção da Estratégia Ótima:** Carregamento das imagens filtradas e escolha das técnicas que apresentaram melhor desempenho em termos de PSNR e preservação de bordas.
2. **Detecção de Bordas:** Aplicação de operadores de borda Sobel e Laplaciano sobre as imagens filtradas para identificação de limites celulares.
3. **Operações Morfológicas:** Aplicação de operação de fechamento (dilatação seguida de erosão) para consolidar regiões fragmentadas e melhorar a continuidade das bordas.
4. **Segmentação por Watershed:** Uso do algoritmo watershed para separação de células adjacentes ou sobrepostas, transformando regiões de toque em regiões distintas.
5. **Limiarização e Extração de Componentes:** Aplicação de limiarização automática para criar máscara binária e identificação de componentes conectados, cada um representando uma célula.
6. **Contagem de Células:** Enumeração automática de regiões detectadas para cada imagem do dataset.
7. **Validação contra Gabarito:** Comparação entre a contagem obtida e o gabarito consolidado no notebook 01, com cálculo de métricas de acurácia (quantidade correta, falsos positivos, falsos negativos).
8. **Relatório Final:** Geração de visualizações mostrando as máscaras de gabarito, as máscaras segmentadas e um resumo quantitativo dos resultados por imagem.

