# Техническое руководство по воспроизведению

Пошаговые инструкции и команды для полного воспроизведения бенчмаркинга на данных **CAMI III Toy Human Gut** (образец `sample_0`). 

## Требования
*   ОС: Linux (тестировалось на WSL Ubuntu)
*   Менеджер пакетов: Conda (Miniconda/Anaconda)
*   Окружение: `cami_benchmark` `motus_env` 

---

## Шаг 1. Подготовка окружения и баз данных

### 1.1. Установка инструментов

```bash
conda create -n cami_benchmark -c bioconda -c conda-forge kraken2 bracken motus centrifuge taxonkit seqtk -y
conda create -n motus_env -c bioconda -c conda-forge motus=3.1.0 -y
conda activate cami_benchmark
```

### 1.2. Скачивание баз данных

**Kraken2**
```bash
mkdir -p ~/db/kraken2 && cd ~/db/kraken2
wget -c https://genome-idx.s3.amazonaws.com/kraken/k2_standard_08gb_20240605.tar.gz
tar -xzf k2_standard_08gb_20240605.tar.gz
```
**Centrifuge**
```bash
mkdir -p ~/db/centrifuge && cd ~/db/centrifuge
wget -c https://genome-idx.s3.amazonaws.com/centrifuge/p_compressed_2018_4_15.tar.gz
tar -xzf p_compressed_2018_4_15.tar.gz
```

**mOTUs**(встроенная база)
```bash
conda activate motus_env
motus downloadDB
```
**TaxonKit**(для конвертации в формат CAMI для Bracken и Centrifuge)

```bash
mkdir -p ~/.taxonkit
wget -c ftp://ftp.ncbi.nih.gov/pub/taxonomy/taxdump.tar.gz -O ~/.taxonkit/taxdump.tar.gz
tar -xzf ~/.taxonkit/taxdump.tar.gz -C ~/.taxonkit
```

## Шаг 2. Подготовка данных (CAMI III Toy)
```bash
# Извлечение ридов 
Скачано с https://s3.bi.denbi.de/swift/v1/cami/cami3_toydata/human-gut-toy/sample_0_reads.tar.gz
tar -xzf sample_0_reads.tar.gz
cd ~/cami_benchmark/reads/sample_0_reads
seqtk seq -1 anonymous_reads.fq.gz | gzip > sample_0_R1.fastq.gz
seqtk seq -2 anonymous_reads.fq.gz | gzip > sample_0_R2.fastq.gz
```


## Шаг 3. Запуск профилировщиков
**Kraken2**
```bash
kraken2 --db ~/db/kraken2 --threads 4 --paired \
  --report ~/cami_benchmark/results/sample_0_kraken2.report \
  --output ~/cami_benchmark/results/sample_0_kraken2.output \
  ~/cami_benchmark/reads/sample_0_reads/sample_0_R1.fastq.gz \
  ~/cami_benchmark/reads/sample_0_reads/sample_0_R2.fastq.gz
```
**Bracken**(на основе Kraken2)
```bash
bracken -d ~/db/kraken2 \
  -i ~/cami_benchmark/results/sample_0_kraken2.report \
  -o ~/cami_benchmark/results/sample_0_bracken.out \
  -r 150 -l S
```
**Centrifuge**
```bash
centrifuge -x ~/db/centrifuge/p_compressed \
  -p 4 \
  -1 ~/cami_benchmark/reads/sample_0_reads/sample_0_R1.fastq.gz \
  -2 ~/cami_benchmark/reads/sample_0_reads/sample_0_R2.fastq.gz \
  -S ~/cami_benchmark/results/sample_0_centrifuge.out \
  --report-file ~/cami_benchmark/results/sample_0_centrifuge.report
```
**mOTUs**
```bash
motus profile \
  -f ~/cami_benchmark/reads/sample_0_reads/sample_0_R1.fastq.gz \
  -r ~/cami_benchmark/reads/sample_0_reads/sample_0_R2.fastq.gz \
  -t 4 -o ~/cami_benchmark/results/sample_0_motus.cami \
  -C precision
```

## Шаг 4. Конвертация результатов в формат CAMI

***Для OPAL профили Bracken и Centrifuge необходимо привести к единому формату с помощью TaxonKit.***

**Bracken:**
```bash
tail -n +2 ~/cami_benchmark/results/sample_0_bracken.out | cut -f 2,7 > /tmp/bracken.tsv
taxonkit profile2cami \
  --taxid-field 1 \
  --abundance-field 2 \
  --sample-id "sample_0" \
  /tmp/bracken.tsv > ~/cami_benchmark/results/sample_0_bracken.cami
```
**Centrifuge**
```bash
tail -n +2 ~/cami_benchmark/results/sample_0_centrifuge.report | cut -f 2,7 > /tmp/centrifuge.tsv
taxonkit profile2cami \
  --taxid-field 1 \
  --abundance-field 2 \
  --sample-id "sample_0" \
  /tmp/centrifuge.tsv > ~/cami_benchmark/results/sample_0_centrifuge.cami
```
## Шаг 5. Конвертация результатов в формат CAMI

```bash
pip3 install cami-opal
```
**Запуск оценки**
```bash
opal.py \
  -g ~/cami_benchmark/gold/taxonomic_profiles/taxonomic_profile_0.txt \
  -o ~/cami_benchmark/opal_results \
  -p \
  ~/cami_benchmark/results/sample_0_bracken.cami \
  ~/cami_benchmark/results/sample_0_centrifuge.cami \
  ~/cami_benchmark/results/sample_0_motus.cami
```
