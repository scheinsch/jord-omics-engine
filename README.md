# jord-omics-engine

# Pipeline Guide for Bacterial Genome Annotation at JBS

The base bacterial genome annotation pipeline used at JBS utilizes the pipeline management software Nextflow. The pipeline integrates several open-source tools to predict genes, biosynthetic gene clusters (BGCs), antibiotic resistance profiles, and antimicrobial peptides.

## Tools Used

- **BAKTA** - Gene finding and annotation
- **antiSMASH** - BGC prediction
- **Gecco** - BGC prediction
- **ABRicate** - Antibiotic resistance
- **fARGene** - Antibiotic resistance
- **AMPlify** - Antimicrobial peptides
- **ampir** - Antimicrobial peptides

The `nf-core/funcscan` pipeline manages the flow of data through these tools, providing summary output and a log of the tool versions used.

## Azure Setup

Due to the large volume of genome assemblies processed in batches, the pipeline runs on Azure. Nextflow has built-in capability to use an Azure Batch account to provision and manage virtual machines for pipeline execution.

### Requirements

- **Azure Batch account** (see Nextflow documentation for setup)
- **Azure storage account**
- **Azure File Share** - Holds databases required by the pipeline
- **Azure Blob Storage** - Holds input (genome assemblies) and pipeline output
- **Azure VM** to serve as the head node - We use ‘anitest’, a 4 vCPU VM
- Genome assemblies in FASTA format (see note below regarding assembly quality)
- BAKTA full database (URL placeholder)
- antiSMASH database (URL placeholder)

### Optional, but Useful

- **Azure Storage Explorer** - Graphical file explorer for transferring data between local computers, Azure File Share, and Azure Blob Storage.

## Running the Pipeline

1. **Start the VM**: Navigate to the Azure portal and go to the Virtual Machines section. Click on the virtual machine ‘anitest’ and click the play button to start it. It should take at most 2-3 minutes.

2. **Connect via SSH**: Connect to the VM using SSH. Ensure you have an SSH key stored on your computer. If not, contact Stephen or Sarija for assistance.

3. **Configure nextflow.config**: Create an appropriate `nextflow.config` file if one is not already available. Note that this repository contains a default configuration file sufficient for getting started. Enter your Azure batch and storage account information.

4. **Upload Genome Assemblies**: Upload the genome assemblies you wish to annotate into a folder within your Azure blob storage account. Ensure it is the blob storage account associated with the Azure Batch service.

5. **Create a Sample Sheet**: The sample sheet must be a CSV file specifying each sample’s name and its corresponding input assembly location. A sample sheet template is available in this repository.

6. **Execute the Pipeline**: Run the pipeline using the appropriate Nextflow command. Monitor the initial execution to familiarize yourself with the pipeline's operation and be prepared to handle errors. The pipeline usually deallocates VMs gracefully upon failure, but you must monitor the Azure Batch account resource pools to confirm.

### Monitoring Pipeline Execution

Each line of the pipeline's output reports the progress of a separate subworkflow. The ID in brackets indicates the work directory, which is useful for troubleshooting individual tools if needed. Subworkflows often depend on earlier ones, so certain subworkflows may appear paused while waiting for others.

### Best Practices

- **Use a Terminal Multiplexer**: We recommend using a terminal multiplexer, such as `screen`, to allow pipeline execution without terminal session interruption.

### Resource Utilization

Nextflow, with Azure Batch, dynamically allocates resources. We initiate the pipeline using a lower-resource head node, specifically the ‘anitest’ VM (4 vCPUs). Ensure the head node is deallocated after execution. Plan ahead, as the ‘anitest’ VM is scheduled for auto-shutdown at a specific time (XX:XX:XX UTC). Disable auto-shutdown or respond promptly to Azure’s notification email to avoid disruptions.

## Expected Output

The pipeline output folder is organized by workflow, grouping outputs by subworkflow rather than isolate. Below is a table of critical output files:

| File                                   | Description                                                          |
| ------------------------------------- | -------------------------------------------------------------------- |
| `annotation/bakta/*.gbff`             | Annotation in GenBank flatfile format                                |
| `annotation/bakta/*.ffn`              | Coding sequences in FASTA format                                     |
| `annotation/bakta/*.fna`              | Assembly sequence in FASTA format                                    |
| `annotation/bakta/*.faa`              | Translated amino acid sequences in FASTA format                      |
| `annotation/bakta/*.gff3`             | Gene annotations in GenBank flat file (GFF3 format)                  |
| `bgc/antismash/<sample>/<sample>.gbk` | Summary antiSMASH annotation in GenBank format                       |
| `bgc/antismash/<sample>/*region*.gbk` | antiSMASH annotation for individual regions                          |
| `bgc/gecco/<sample>/*.clusters.tsv`   | Tab-delimited report of detected BGCs                                |
| `arg/abricate`                        | Description placeholder                                              |
| `arg/fargene`                         | Description placeholder                                              |
| `arg/rgi`                             | Description placeholder                                              |
| `arg/hamronization`                   | Description placeholder                                              |
| `amp/ampir`                           | Description placeholder                                              |
| `amp/amplify`                         | Description placeholder                                              |
| `amp/macrel`                          | Description placeholder                                              |

> **Note**: The above table is not exhaustive.
