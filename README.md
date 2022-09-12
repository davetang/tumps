# tumps
tumor purity simulation

will only work on an HPC system. Can be easily adapted

## Sambamba

[Sambamba](https://github.com/biod/sambamba) is used to subsample and merge a tumour and normal BAM file. A binary can be downloaded from the [releases page](https://github.com/biod/sambamba/releases).

```bash
wget https://github.com/biod/sambamba/releases/download/v0.8.2/sambamba-0.8.2-linux-amd64-static.gz
gunzip sambamba-0.8.2-linux-amd64-static.gz
chmod 755 sambamba-0.8.2-linux-amd64-static
ln -s sambamba-0.8.2-linux-amd64-static sambamba
```

Below is the relevant part of `tumps.sh` that performs the mixing.

```bash
PURITY=50
SAMBAMBA=${HOME}/bin/sambamba
TUMOR=tumour.bam
NORMAL=normal.bam
TUMOR_COVERAGE=70
NORMAL_COVERAGE=30
PURBAM=purity${PURITY}.bam

echo "Calculating ${PURITY}% purity"
RATIO=`echo "${PURITY} / 100" | bc -l`
TCOV=`echo "${TUMOR_COVERAGE} * ${RATIO}" | bc -l`
echo "Subsample tumor BAM to a ${PURITY}%. That is ${TCOV}x coverage"
echo "$SAMBAMBA view -f bam -s $RATIO -o tumorOnly${PURITY}.bam.tmp $TUMOR"

NCOV=`echo "${TUMOR_COVERAGE} - ${TCOV}" | bc -l`
NPUR=`echo "${NCOV} / ${NORMAL_COVERAGE}" | bc -l`
echo "Subsample ${NCOV}x coverage from normal BAM. That is $NPUR of the original"
echo "$SAMBAMBA view -f bam -s $NPUR -o normalOnly${PURITY}.bam.tmp $NORMAL"

echo "Merge both"
echo "$SAMBAMBA merge $PURBAM tumorOnly${PURITY}.bam.tmp normalOnly${PURITY}.bam.tmp"
```

The output from above.

```
Calculating 50% purity
Subsample tumor BAM to a 50%. That is 35.00000000000000000000x coverage
/home/dtang/bin/sambamba view -f bam -s .50000000000000000000 -o tumorOnly50.bam.tmp tumour.bam
Subsample 35.00000000000000000000x coverage from normal BAM. That is 1.16666666666666666666 of the original
/home/dtang/bin/sambamba view -f bam -s 1.16666666666666666666 -o normalOnly50.bam.tmp normal.bam
Merge both
/home/dtang/bin/sambamba merge purity50.bam tumorOnly50.bam.tmp normalOnly50.bam.tmp
```

For `sambamba view` the options are:

* `-f`, --format=sam|bam|json|unpack specify which format to use for output
(default is SAM); unpack streams unpacked BAM
* `-s`, --subsample=FRACTION subsample reads (read pairs)
* `-o`, --output-filename specify output filename

Note that `-s` can only subsample, i.e. accept a value up to 1.

For `sambamba merge` the usage is  `<output.bam> <input1.bam> <input2.bam>`.

## Improvements

- [ ] Calculate coverage within the script.
- [ ] Script assumes that normal and tumour samples have similar coverage.
- [ ] Option to down-sample or up-size such that both BAM files have similar coverage.
- [ ] Option to output back to FASTQ.

## Basic usage
Required parameters are: 
- Tumor BAM file
- Normal BAM file
- Tumor Coverage
- Normal Coverage

Example basic usage:

```
./tumps.sh -t tumor.bam -n normal.bam -tc 90 -nc 30
```
Purities are specified providing the -p | --purities argument with a comma separated list. By default, purities are 0,10,20,25,50,75,100. 0 corresponds to the normal BAM file and 100 to the tumor BAM file.
Somatic SV calling is performed using GRIDSS by default. Other supported options are NanoSV or PBSV. SV calling can be disabled with --nosv.
I.e. for a run to get tumor purities of 30% and 60% and no somatic SV calling:

```
./tumps.sh -t tumor.bam -n normal.bam -tc 90 -nc 30 -p 30,60 --nosv
```

## Advanced usage

```
./tumps.sh --help

Parameters:
    -t|--tumor                    Path to tumor BAM file, must be indexed (Required)
    -n|--normal                    Path to normal BAM file, must be indexed (Required)
    -tc|--tumor_coverage                    Average tumor coverage (Required)
    -nc|--normal_coverage                    Average normal coverage for mixing (Required)
    -nsv|--normal_sv                    Path to normal BAM file to use for SV calling (if different than normal above), must be indexed [equal to --normal]
    -p|--purities                    Merge Tumor and Normal with the following tumor purities (comma separated): [0,10,20,25,50,75,100]
    -d|--output_dir                    OUTPUT DIRECTORY [.]
    --nosv                     Don't perform SV calling and stop after mixing the BAMs [FALSE]
    --nooverlap                     Don't perform truth and stop after SV calling [FALSE]
    -m|--mode                    SV calling mode, choose from: gridss, nanopore, pbsv [gridss]
    -g|--truthset                    Path to TRUTHSET VCF file for comparison [files/COLO829.somatic.vcf]
    --bed                    BED file for coverage calculations [files/human_hg19.bed]
    --sambamba                    Path to SAMBAMBA [/hpc/local/CentOS7/cog_bioinf/sambamba_v0.6.5/sambamba]
    --samtools                    Path to SAMTOOLS [/hpc/local/CentOS7/cog_bioinf/samtools-1.7/samtools]
    --mail                    Mail for HPC jobs [jespejov@umcutrecht.nl]
    --libgridss                    Path to LIBGRIDSS directory [scripts/libgridss/]
    --gridss_pon                    Path to GRIDSS PON directory (too big to share here) [/hpc/cog_bioinf/cuppen/project_data/Roel_pipeline_validation/gridss]
    --sniffles                    Path to Sniffles executable [/hpc/cog_bioinf/cuppen/personal_data/jvalleinclan/tools_kloosterman/Sniffles-1.0.8/bin/sniffles-core-1.0.8/sniffles]
    --survivor                    Path to SURVIVOR executable (for nanopore and pbsv) [/hpc/cog_bioinf/cuppen/personal_data/jvalleinclan/tools_kloosterman/SURVIVOR-1.0.6/Debug/SURVIVOR]
    --pbsv                    Path to PBSV executable [/hpc/cog_bioinf/cuppen/personal_data/jvalleinclan/bin/miniconda3/bin/pbsv]
    --ref                    Path to reference genome (for PBSV) [/hpc/cog_bioinf/GENOMES/Homo_sapiens.GRCh37.GATK.illumina/Homo_sapiens.GRCh37.GATK.illumina.fasta]
    --nanosv_venv                    Path to NanoSV VENV folder [/hpc/cog_bioinf/cuppen/personal_data/jvalleinclan/bin/NanoSV/bin/NanoSV]
    --nanosv_config                    Path to NanoSV config file [files/config_COLO829_NGMLR.ini]

```
