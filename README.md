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
    └── 01_eda_preparacao.ipynb  # Notebook inicial de exploração e preparação de dados

```

## Sobre os Dados

O projeto utiliza o dataset biomédico **BBBC020** (Murine bone-marrow derived macrophages).

* **Características das Imagens:** O conjunto de dados consiste em 25 imagens de macrófagos, disponibilizadas em canais separados.
* **Canal `_c1` (CD11b/APC):** Marca a superfície e o citoplasma da célula.
* **Canal `_c5` (DAPI):** Marca os núcleos.
* **Canal `_(c1+c5)`:** Imagem mesclada de ambos os marcadores para visualização de referência.

* **Desafios do Domínio:** As células apresentam formato irregular e frequentemente se tocam ou sobrepõem. As imagens também contêm ruído natural de sensor e variações de iluminação, exigindo tratamento adequado de pré-processamento.
* **Ground Truth:** O dataset fornece arquivos `.TIF` individuais contendo apenas a linha de contorno (*outline*) para cada célula e cada núcleo identificados. *Nota: Conforme a documentação oficial, os arquivos de gabarito são ausentes/incompletos para algumas das imagens originais.*

## Notebook 01: Análise Exploratória e Preparação dos Dados (`01_ead_preparacao.ipynb`)

Este notebook é responsável por organizar os arquivos brutos e construir as bases que permitirão a validação do algoritmo de contagem no final do projeto. Suas principais funções são:

1. **Mapeamento e Limpeza:** Varredura automática do diretório `raw` para relacionar cada imagem aos seus respectivos arquivos de canais (`c1`, `c5` e mesclado), ignorando arquivos de sistema indesejados que vieram no dataset.
2. **Visualização Inicial:** Inspeção das características visuais dos diferentes canais das imagens usando OpenCV e Matplotlib.
3. **Consolidação do Gabarito (Ground Truth):** Leitura de todos os pequenos arquivos individuais de contorno de uma mesma amostra, preenchimento de suas áreas internas e agregação em uma **única máscara rotulada** (onde cada região conectada recebe um ID numérico único).
4. **Exportação Processada:** Processamento em lote de todo o dataset, ignorando automaticamente as imagens que não possuem gabarito no repositório oficial, e salvamento das máscaras unificadas em formato `.npy` dentro de `data/processed/` para uso nos notebooks de validação.
