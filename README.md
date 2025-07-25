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

           for FILE in $(ls *Q30.bam); do echo $FILE; sbatch --partition=pall --job-name=$(echo $FILE | cut -d'_' -f1,2)ST2 --time=0-03:00:00 --mem-per-cpu=64G --ntasks=8 --cpus-per-task=1 --output=$(echo $FILE | cut -d'_' -f1,2)_ST.out --error=$(echo $FILE | cut -d'_' -f1,2)_FB.error --mail-type=END,FAIL --wrap "cd /data/projects/p495_SinorhizobiumMeliloti/03_MasterSummerProject/03_pascal_reanalysis; module add UHTS/Analysis/picard-tools/1.127;picard-tools AddOrReplaceReadGroups I=$FILE O=$(echo $FILE | cut -d'.' -f1)_Fixed.bam RGID=4 RGLB=lib1 RGPL=illumina RGPU=unit1 RGSM=20"   ; sleep 1; done

b. Variant calling freebayes

            for FILE in $(ls *Q30_Fixed.bam); do echo $FILE; sbatch --partition=pall --job-name=$(echo $FILE | cut -d'_' -f1,2)_FB --time=0-03:00:00 --mem-per-cpu=64G --ntasks=8 --cpus-per-task=1 --output=$(echo $FILE | cut -d'_' -f1,2)_FB.out --error=$(echo $FILE | cut -d'_' -f1,2)_FB.error --mail-type=END,FAIL --wrap "module add UHTS/Analysis/freebayes/1.2.0; cd /data/projects/p495_SinorhizobiumMeliloti/03_MasterSummerProject/03_pascal_reanalysis;  freebayes --fasta-reference p_ctg_oric.fasta -C 10 $FILE > $(echo $FILE | cut -d'_' -f1,2,3)_FreeBayes.vcf"   ; sleep 1; done

c. filter variant calling

           for FILE in $(ls *_FreeBayes.vcf); do echo $FILE; sbatch --partition=pall --job-name=$(echo $FILE | cut -d'_' -f1,2)_FB --time=0-03:00:00 --mem-per-cpu=64G --ntasks=8 --cpus-per-task=1 --output=$(echo $FILE | cut -d'_' -f1,2)_FB.out --error=$(echo $FILE | cut -d'_' -f1,2)_FB.error --mail-type=END,FAIL --wrap "module load UHTS/Analysis/vcftools/0.1.15; cd /data/projects/p495_SinorhizobiumMeliloti/03_MasterSummerProject/03_pascal_reanalysis;  vcftools --vcf $FILE --minQ 20 --recode --recode-INFO-all --out $(echo $FILE | cut -d'_' -f1,2,3,4)_q20.vcf"   ; sleep 1; done


