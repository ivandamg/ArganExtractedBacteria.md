# ArganExtractedBacteria.md
Analysis of PacBio sequencing on bacteria extracted from root nodules under argon gaz condition



1. Map reads to reference with minimap2
map reads with minimap2
Sort and compress to bam with samtools


      sbatch --partition=pibu_el8 --job-name=minimap1 --time=0-10:00:00 --mem-per-cpu=50G --ntasks=12 --cpus-per-task=1 --output=minimap1.out --error=minimap1.error --mail-type=END,FAIL --wrap "module load minimap2/2.20-GCCcore-10.3.0; cd /data/projects/p782_RNA_seq_Argania_spinosa/40_S_spinosum_FinalFinal/06_RemapReads/Hap1; minimap2 -ax map-hifi /data/projects/p782_RNA_seq_Argania_spinosa/40_S_spinosum_FinalFinal/01_Assembly/01_hap1/S_spinosum_hap1.fa /data/projects/p782_RNA_seq_Argania_spinosa/40_S_spinosum_FinalFinal/01_Assembly/Combined_clean.fq > CombinedClean_hap1.sam; module load SAMtools;samtools sort -o CombinedClean_hap1.bam CombinedClean_hap1.sam; samtools index CombinedClean_hap1.bam"
