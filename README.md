# ArganExtractedBacteria.md
Analysis of PacBio sequencing on bacteria extracted from root nodules under argon gaz condition

1. Evaluate reads quality

               for FILE in $(ls *.fastq.gz); do echo $FILE; sbatch --partition=pshort_el8 --job-name=$(echo $FILE | cut -d'.' -f1)fastp --time=0-02:00:00 --mem-per-cpu=24G --ntasks=1 --cpus-per-task=1 --output=$(echo $FILE | cut -d'.' -f1)_fastp.out --error=$(echo $FILE | cut -d'.' -f1)_fastp.error --mail-type=END,FAIL --wrap " cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/01_RawData ; module load FastQC;fastqc -t 4 $FILE"; done 


2. Clean and filter reads

                for FILE in $(ls *.fastq.gz); do echo $FILE; sbatch --partition=pshort_el8 --job-name=$(echo $FILE | cut -d'.' -f1)fastp --time=0-02:00:00 --mem-per-cpu=24G --ntasks=1 --cpus-per-task=1 --output=$(echo $FILE | cut -d'.' -f1)_fastp.out --error=$(echo $FILE | cut -d'.' -f1)_fastp.error --mail-type=END,FAIL --wrap " cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/01_RawData ; module load fastp/0.23.4-GCC-10.3.0; fastp -i $FILE -W 10 -5 -3 -M 10 -l 3000 -G -A -w 20 --qualified_quality_phred 30 -o $(echo $FILE | cut -d'.' -f1)_Clean.fastq.gz"; done 

3. Move to new folder

               mkdir ../02_CleanedReads; mv *Clean.fastq.gz ../02_CleanedReads

5. Evaluate quality after fastp

            for FILE in $(ls *.fastq.gz); do echo $FILE; sbatch --partition=pshort_el8 --job-name=$(echo $FILE | cut -d'.' -f1)fastp --time=0-02:00:00 --mem-per-cpu=24G --ntasks=1 --cpus-per-task=1 --output=$(echo $FILE | cut -d'.' -f1)_fastp.out --error=$(echo $FILE | cut -d'.' -f1)_fastp.error --mail-type=END,FAIL --wrap " cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/02_CleanedReads ; module load FastQC;fastqc -t 4 $FILE"; done 


6. Map cleaned reads to reference with minimap2


                  mkdir ../03_MappedReads;

                for FILE in $(ls *.fastq.gz); do echo $FILE; sbatch --partition=pibu_el8 --job-name=minimap1 --time=0-10:00:00 --mem-per-cpu=50G --ntasks=12 --cpus-per-task=1 --output=$(echo $FILE | cut -d'_' -f2)_minimap.out --error=$(echo $FILE | cut -d'_' -f2)_minimap.error --mail-type=END,FAIL --wrap "module load minimap2/2.20-GCCcore-10.3.0; cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/02_CleanedReads; minimap2 -ax map-hifi /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/00_Ref/FribourgSMeliloti_Prokka.fna $FILE > ../03_MappedReads/$(echo $FILE | cut -d'_' -f1,2)_Fribourg.sam; module load SAMtools;samtools sort -o ../03_MappedReads/$(echo $FILE | cut -d'_' -f1,2)_Fribourg.bam ../03_MappedReads/$(echo $FILE | cut -d'_' -f1,2)_Fribourg.sam; samtools index ../03_MappedReads/$(echo $FILE | cut -d'_' -f1,2)_Fribourg.bam"; done

7.  Call snp with freebayes

a. Fix bam files with picard AddOrReplaceReadGroups Picard tools

           for FILE in $(ls *.bam); do echo $FILE; sbatch --partition=pshort_el8 --job-name=$(echo $FILE | cut -d'_' -f1,2)picard --time=0-02:00:00 --mem-per-cpu=64G --ntasks=8 --cpus-per-task=1 --output=$(echo $FILE | cut -d'_' -f1,2)_picard.out --error=$(echo $FILE | cut -d'_' -f1,2)_picard.error --mail-type=END,FAIL --wrap "cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/03_MappedReads; module load picard/2.25.5-Java-13;java -jar $EBROOTPICARD/picard.jar AddOrReplaceReadGroups I=$FILE O=$(echo $FILE | cut -d'.' -f1)_Fixed.bam RGID=4 RGLB=lib1 RGPL=illumina RGPU=unit1 RGSM=20"   ; sleep 1; done

b. Variant calling freebayes

            for FILE in $(ls *_Fixed.bam); do echo $FILE; sbatch --partition=pibu_el8 --job-name=$(echo $FILE | cut -d'_' -f1,2)_FB --time=0-03:00:00 --mem-per-cpu=128G --ntasks=1 --cpus-per-task=1 --output=$(echo $FILE | cut -d'_' -f1,2)_FB.out --error=$(echo $FILE | cut -d'_' -f1,2)_FB.error --mail-type=END,FAIL --wrap "module load freebayes; cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/03_MappedReads;  freebayes -f /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/00_Ref/FribourgSMeliloti_Prokka.fna -C 10 $FILE > $(echo $FILE | cut -d'_' -f1,2)_FreeBayes.vcf"   ; sleep 1; done

c. filter variant calling

           for FILE in $(ls *_FreeBayes.vcf); do echo $FILE; sbatch --partition=pshort_el8 --job-name=$(echo $FILE | cut -d'_' -f1,2)_FB --time=0-02:00:00 --mem-per-cpu=64G --ntasks=8 --cpus-per-task=1 --output=$(echo $FILE | cut -d'_' -f1,2)_FB.out --error=$(echo $FILE | cut -d'_' -f1,2)_FB.error --mail-type=END,FAIL --wrap "module load VCFtools; cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/03_MappedReads;  vcftools --vcf $FILE --minQ 20 --recode --recode-INFO-all --out $(echo $FILE | cut -d'_' -f1,2)_FreeBayes_q20.vcf"   ; sleep 1; done





# 10 denovo assembly
module load hifiasm/0.16.1-GCCcore-10.3.0

## 1. denovo 1E

### 1. Hifiasm

    
    sbatch --partition=pibu_el8 --job-name=1E_hifiasm7 --time=0-4:00:00 --mem-per-cpu=64G --ntasks=12 --cpus-per-task=1 --output=hifiasm1.out --error=hifiasm1.error --mail-type=END,FAIL --wrap "module load hifiasm/0.16.1-GCCcore-10.3.0; cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/10_deNovo; hifiasm -o 01_hifiasm/deNovo_Argon1E -t 12 /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/02_CleanedReads/Argon_1E_Clean.fastq.gz"

### 2. Transform gfa. fasta

            awk '/^S/{print ">"$2;print $3}' /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/10_deNovo/01_hifiasm/deNovo_Argon1E.bp.p_ctg.gfa > deNovo_Argon1E.p_ctg.fa

            
## 2. circularize


### 1. Map reads to contigs


       sbatch --partition=pibu_el8 --job-name=minimap2 --time=0-2:00:00 --mem-per-cpu=64G --ntasks=12 --cpus-per-task=1 --output=minimap.out --error=minimap.error --mail-type=END,FAIL --wrap "module load minimap2/2.20-GCCcore-10.3.0; module load SAMtools/1.13-GCC-10.3.0; cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/10_deNovo/01_hifiasm/; minimap2 -t 12 -ax map-hifi deNovo_Argon1E.p_ctg.fa /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/02_CleanedReads/Argon_1E_Clean.fastq.gz > Argon_1E_Aligned.sam | samtools sort -o Argon_1E_Sorted.bam; samtools index Argon_1E_Sorted.bam "


### 2. Circularize


      singularity pull /data/users/imateusgonzalez/SOFTS/circlator.sif docker://sangerpathogens/circlator
      export CIRCLATOR_SIF=/data/users/imateusgonzalez/SOFTS/circlator.sif
      singularity exec ${CIRCLATOR_SIF} circlator all /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/10_deNovo/01_hifiasm/deNovo_Argon1E.p_ctg.fa /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/02_CleanedReads/Argon_1E_Clean.fastq.gz /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/10_deNovo/01_hifiasm/01_Circularized


### 3. map reads to circular sequence..

          sbatch --partition=pibu_el8 --job-name=minimap2 --time=0-2:00:00 --mem-per-cpu=64G --ntasks=12 --cpus-per-task=1 --output=minimap.out --error=minimap.error --mail-type=END,FAIL --wrap "module load minimap2/2.20-GCCcore-10.3.0; module load SAMtools/1.13-GCC-10.3.0; cd /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/10_deNovo/01_hifiasm/01_Circularized; minimap2 -t 12 -ax map-hifi 06.fixstart.fasta /data/projects/p495_SinorhizobiumMeliloti/50_ArgonExtractedBacteria/02_CleanedReads/Argon_1E_Clean.fastq.gz > Argon_1E_Aligned_denovo.sam;  samtools sort Argon_1E_Aligned_denovo.sam -o Argon_1E_Sorted_denovo.bam; samtools index Argon_1E_Sorted_denovo.bam "

