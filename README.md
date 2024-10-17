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
3. **Mount the Azure File Share**: Mount the Azure File Share on the head node VM to make the databases available.
   ```bash
       cd ~
       ./mount_annotationdb.sh
   ```

5. **Configure nextflow.config**: Create an appropriate `nextflow.config` file if one is not already available. Note that this repository contains a default configuration file sufficient for getting started. Enter your Azure batch and storage account information. Do not change anything other than the account specific information. Doing so may change the way the pipeline manages resources and lead to unexpected costs.
    ```ini
    process {
    executor = 'azurebatch'
    }

    azure {
        activeDirectory {
            servicePrincipalId = <YOUR SERVICE PRINCIPAL ID>
            servicePrincipalSecret = <YOUR SERVICE PRINCIPAL SECRET>
            tenantId = <YOUR TENANT ID>
        }
    storage {
        accountName = <YOUR ACCOUNT NAME>
        accountKey = <YOUR ACCOUNT KEY>
        fileShares {
            annotationdb {
                mountPath = "/mnt/"
            }
        }
    }
    batch {

        location = 'eastus'
        accountName = 'bioinformaticsbatch'
        autoPoolMode = true
        deletePoolsOnCompletion = true
        pools {
            auto {
                    vmType = 'Standard_D8_v3'
                    vmCount = 10
                    privileged = true
            }
        }
    }

   }
    ```
6. **Upload Genome Assemblies**: Upload the genome assemblies you wish to annotate into a folder within your Azure blob storage account. Ensure it is the blob storage account associated with the Azure Batch service.

7. **Create a Sample Sheet**: The sample sheet must be a CSV file specifying each sample’s name and its corresponding input assembly location. A sample sheet template is available in this repository. Example samplesheet.csv contents:
```ini
sample,fasta
JBS9815,az://bioinfobatchblob/20240812_medium_batch_annotation/AB-UMGC-Kin-b02-JBS9815_S417_L006_assembly.scaffold.fna
JBS1497,az://bioinfobatchblob/20240812_medium_batch_annotation/AB-UMGC-Kin-b02-JBS1497_S473_L006_assembly.scaffold.fna
JBS3673,az://bioinfobatchblob/20240812_medium_batch_annotation/AB-UMGC-Kin-b01-JBS3673_S178_L005_assembly.scaffold.fna
JBS3330,az://bioinfobatchblob/20240812_medium_batch_annotation/AB-UMGC-Kin-b02-JBS3330_S345_L006_assembly.scaffold.fna
```
8. **Execute the Pipeline**: Run the pipeline using the appropriate Nextflow command. Monitor the initial execution to familiarize yourself with the pipeline's operation and be prepared to handle errors. The pipeline usually deallocates VMs gracefully upon failure, but you must monitor the Azure Batch account resource pools to confirm.

### Monitoring Pipeline Execution

Each line of the pipeline's output reports the progress of a separate subworkflow. The ID in brackets indicates the work directory, which is useful for troubleshooting individual tools if needed. Subworkflows often depend on earlier ones, so certain subworkflows may appear paused while waiting for others.

### Best Practices

- **Use a Terminal Multiplexer**: We recommend using a terminal multiplexer, such as `screen`, to allow pipeline execution without terminal session interruption.

### Running the Pipeline with `screen`

Using `screen` allows you to run the pipeline and disconnect from the terminal while keeping the process running in the background. Here are the steps to use `screen` for running the pipeline:

1. **Start a new `screen` session**: First, start a `screen` session by running the following command in your SSH terminal:

    ```bash
    screen -S pipeline-session
    ```

    This command opens a new terminal session within `screen`, allowing you to safely disconnect later without killing the pipeline process.

2. **Run the pipeline**: Inside the `screen` session, run your Nextflow pipeline command (Note: this is only an example command used for brevity, your command will be different):

    ```bash
    nextflow run nf-core/funcscan --input samplesheet.csv --output results/
    ```

3. **Disconnect from the `screen` session**: After initiating the pipeline, you can safely disconnect from the `screen` session by pressing the following key combination:

    ```
    Ctrl + A, then D
    ```

    This will "detach" the `screen` session, keeping the pipeline running in the background while allowing you to close the SSH connection.

4. **Reattach to the `screen` session later (optional)**: To check the status of the pipeline or re-enter the `screen` session, log back into the VM via SSH and run:

    ```bash
    screen -r pipeline-session
    ```

5. **Terminate the `screen` session**: Once the pipeline has completed, you can exit the `screen` session permanently by typing `exit` inside the `screen` session or terminating it by pressing `Ctrl + D`.

---

> **Tip**: If your SSH connection drops unexpectedly, you can always reattach to your running session by using `screen -r` as shown above.


### Resource Utilization

Nextflow, with Azure Batch, dynamically allocates resources. We initiate the pipeline using a lower-resource head node, specifically the ‘anitest’ VM (4 vCPUs). Ensure the head node is deallocated after execution. Plan ahead, as the ‘anitest’ VM is scheduled for auto-shutdown at a specific time (7:00:00 PM UTC). Disable auto-shutdown or respond promptly to Azure’s notification email to avoid disruptions.

We have restricted the pipeline to use 'Standard_D8_v3' VMs (8 vCPU, 32 GB RAM), and a VM count of 10. This has worked well on batches as large as 300 assemblies. Our vCPU quota should allow us to go higher in VM count, however it has not yet been necessary as the pipeline still completes in under 24 hours of operation. 

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
