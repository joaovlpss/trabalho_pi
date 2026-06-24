# Pipeline de Segmentação e Contagem de Células

## Visão Geral

Este projeto implementa um fluxo automatizado de processamento digital de imagens para análise de células em microscopia fluorescente. O desafio central é lidar com características típicas de imagens biomédicas: ruído de sensor, variação de iluminação de fundo e células que se tocam ou se sobrepõem.

O trabalho divide-se em quatro etapas:

1. **Melhoria de contraste:** equalização de histograma (HE global e CLAHE) para corrigir iluminação desigual.
2. **Filtragem de ruído:** comparação entre filtros espaciais (mediana e gaussiano) e filtragem em frequência (passa-baixas via FFT), com avaliação quantitativa por MSE e PSNR sob ruído controlado, mais uma métrica de ruído residual sem referência.
3. **Segmentação:** operadores de borda (Sobel e Laplaciano), operações morfológicas (fechamento e abertura) e algoritmo *watershed* para separar células adjacentes.
4. **Contagem e validação:** quantificação das células detectadas e comparação contra o gabarito do dataset, para determinar qual combinação de técnicas entrega o melhor resultado final.

## Estrutura de Diretórios

```text
.
├── data
│   ├── raw
│   │   ├── BBBC020_v1_images            # Imagens originais (canais separados e mesclado)
│   │   ├── BBC020_v1_outlines_cells     # TIFs individuais com contornos das células
│   │   └── BBC020_v1_outlines_nuclei    # TIFs individuais com contornos dos núcleos
│   ├── processed
│   │   ├── 01_ground_truth
│   │   │   ├── gt_cells                 # Máscaras rotuladas de células (.npy)
│   │   │   └── gt_nuclei                # Máscaras rotuladas de núcleos (.npy)
│   │   ├── 02_filtered_images
│   │   │   ├── mediana                  # Imagens CLAHE + mediana 3×3 (.npy)
│   │   │   ├── gaussiano                # Imagens CLAHE + gaussiano 5×5 (.npy)
│   │   │   └── fft                      # Imagens CLAHE + FFT passa-baixas (.npy)
│   │   └── 03_segmented_masks
│   │       ├── mediana                  # Máscaras segmentadas por pipeline (.npy)
│   │       ├── gaussiano
│   │       └── fft
│   └── metadata
│       ├── ground_truth_manifest.json   # Lista de imagens que possuem gabarito
│       ├── metricas_filtragem.csv       # Métricas de filtragem por imagem/filtro
│       └── metricas_comparacao_pipelines.csv  # Métricas de segmentação por imagem/filtro
└── notebooks
    ├── 01_eda_preparacao.ipynb
    ├── 02_preprocessamento_filtragem.ipynb
    └── 03_segmentacao_contagem.ipynb
```

## Sobre os Dados

O projeto utiliza o dataset biomédico **BBBC020** (*Murine bone-marrow derived macrophages*).

- **Imagens:** 25 amostras de macrófagos, em canais separados.
- **Canal `_c1` (CD11b/APC):** marca a superfície e o citoplasma da célula. É o canal usado em todo o pipeline.
- **Canal `_c5` (DAPI):** marca os núcleos.
- **Canal `_(c1+c5)`:** imagem mesclada, para visualização de referência.
- **Desafios do domínio:** células de formato irregular que frequentemente se tocam ou se sobrepõem, além de ruído de sensor e variação de iluminação.
- **Ground truth:** o dataset fornece um arquivo `.TIF` por célula/núcleo, contendo apenas a linha de contorno (*outline*). Conforme a documentação oficial, os gabaritos são incompletos: não há contorno para células na borda da imagem nem para células com sobreposição/borrão muito forte. Das 25 imagens, **20 possuem gabarito** e são as usadas na validação. Como o gabarito exclui justamente os casos difíceis, ele tende a ser um **limite inferior** do número real de células.

## Notebook 01: Análise Exploratória e Preparação (`01_eda_preparacao.ipynb`)

Organiza os arquivos brutos e constrói as bases de validação:

1. **Mapeamento e limpeza:** varredura do diretório `raw` relacionando cada imagem aos seus canais (`c1`, `c5`, mesclado), ignorando arquivos de sistema (`Thumbs.db`).
2. **Visualização inicial:** inspeção dos diferentes canais com OpenCV e Matplotlib.
3. **Consolidação do gabarito:** leitura dos contornos individuais de cada amostra, preenchimento das áreas internas e agregação em uma máscara rotulada única (cada célula recebe um ID; a contagem do gabarito é o maior ID). Gerado tanto para células quanto para núcleos.
4. **Exportação:** processamento em lote, ignorando automaticamente imagens sem gabarito, com salvamento das máscaras em `.npy` em `data/processed/01_ground_truth/` e de um *manifest* (`ground_truth_manifest.json`) com as imagens válidas.

## Notebook 02: Pré-processamento e Filtragem (`02_preprocessamento_filtragem.ipynb`)

Avalia as estratégias de contraste e de redução de ruído.

1. **Estimativa de ruído:** quantifica o ruído nativo do fundo (σ do fundo via Otsu) e o SNR. As imagens têm ruído real (σ ≈ 14,6; SNR ≈ 5,9).
2. **Equalização de histograma:** comparação entre HE global e **CLAHE** (`clip=2.0`, `tile=8×8`). O CLAHE é adotado como base do pipeline por realçar contraste local sem estourar o histograma.
3. **Filtros espaciais:** mediana e gaussiano em kernels 3×3 e 5×5, aplicados sobre a base CLAHE.
4. **Filtro em frequência:** passa-baixas via FFT (máscara circular ideal, `r=50`), com visualização do espectro antes/depois.
5. **Avaliação quantitativa (PSNR/MSE):** como **não existe versão limpa** das imagens nativas, a comparação correta usa **ruído sintético controlado** — adiciona-se ruído (gaussiano e sal-e-pimenta) à base, filtra-se e compara-se com a base **antes** do ruído. (Medir PSNR contra a própria imagem de entrada seria incorreto, pois premiaria o filtro que menos altera a imagem, e não o que melhor remove ruído.)
6. **Processamento em lote e exportação:** salva as três versões filtradas de cada imagem (`02_filtered_images/`) e grava, por imagem/filtro, duas métricas honestas: PSNR/MSE sob ruído sintético (*full-reference*) e o σ do fundo após filtrar (*no-reference*). Ao final, agrega tudo em um comparativo global espacial × frequência.

**Resultado da filtragem.** Sob ruído gaussiano (perfil compatível com sensor de fluorescência), o gaussiano vence o PSNR médio (32,2 dB) em todas as 20 imagens; sob sal-e-pimenta, a mediana domina (39,4 dB). A métrica sem-referência (σ do fundo) ficou pouco discriminativa entre filtros. **Observação importante:** o filtro melhor por PSNR não é necessariamente o melhor para a contagem — isso é testado no Notebook 03.

## Notebook 03: Segmentação, Contagem e Validação (`03_segmentacao_contagem.ipynb`)

Aplica a segmentação completa e valida contra o gabarito.

1. **Detecção de bordas e impacto dos filtros:** Sobel e Laplaciano aplicados sobre as três versões filtradas. A magnitude do gradiente (Sobel) é o relevo sobre o qual o *watershed* separa as células. A qualidade da borda por filtro é medida quantitativamente pelo contraste de borda (gradiente sobre as bordas reais do gabarito ÷ gradiente no fundo).
2. **Operações morfológicas:** fechamento (para fechar falhas no anel da membrana e pequenos vãos) seguido de preenchimento de buracos e abertura (remove respingos isolados), construindo a máscara que alimenta o *watershed*.
3. **Segmentação por watershed:** sementes obtidas pela transformada de distância (um pico por célula) e relevo dado pelo gradiente; inclui uma etapa de fusão de fragmentos com borda fraca e filtro por área.
4. **Contagem e validação:** para cada imagem conta-se o número de regiões e compara-se com o gabarito, com erro de contagem, IoU binária e métricas por célula (TP/FP/FN com IoU > 0,5, e F1 derivado).
5. **Comparação de pipelines:** os três filtros são rodados pelo mesmo *watershed* e comparados pela acurácia de contagem.

## Resultados

### Comparação dos três pipelines (média sobre as 20 imagens com gabarito)

| Pipeline (CLAHE + filtro + Watershed) | Erro de contagem | IoU média | Precisão | Recall | F1 (por célula) |
|---|---|---|---|---|---|
| **Mediana** | 39,9% | 0,254 | 0,318 | 0,188 | **0,236** |
| Gaussiano | 38,3% | 0,254 | 0,312 | 0,188 | 0,235 |
| FFT | 38,2% | 0,219 | 0,227 | 0,140 | 0,173 |

### Qualidade de borda por filtro (contraste = borda/fundo, Sobel)

| Filtro | Gradiente na borda | Gradiente no fundo | Contraste |
|---|---|---|---|
| Gaussiano | 49,9 | 15,4 | **3,46** |
| Mediana | 38,4 | 12,1 | 3,35 |
| FFT | 77,4 | 31,8 | 2,62 |

### Conclusões

- **A escolha do filtro pouco afeta o resultado final.** Mediana e gaussiano empatam na prática (F1 0,236 vs 0,235); o FFT fica atrás. O erro de contagem permanece em ~38–40% para os três. A melhor combinação é **CLAHE + mediana + Watershed (com fechamento)**, mas por margem mínima.
- **Qualidade de imagem ≠ qualidade de segmentação.** O PSNR (Notebook 02) elegeu o gaussiano em 20/20 imagens; pela acurácia de contagem, o vencedor é a mediana. A comparação por contagem foi o que revelou essa diferença.