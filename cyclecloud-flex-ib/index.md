# Leveraging Azure CycleCloud and Virtual Machine Scale Set in Flexible Orchestration Mode to Scale HPC Workloads with Over a Thousand InfiniBand-Connected Nodes

## I. Introduction

In the world of high-performance computing (HPC), scaling out clusters to meet demanding computational needs is a crucial aspect. Azure CycleCloud, a cloud-based HPC management solution, provides a powerful platform for orchestrating and scaling HPC workloads. In this blog post, we will explore how to scale out a CycleCloud SLURM cluster on Azure VMSS Flex, leveraging InfiniBand connectivity. This configuration not only enhances scalability but also improves deployment performance and reliability, making it an ideal choice for embarrasingly parallel HPC and AI worloads.

## II. Setting Up the Environment for CycleCloud Cluster

To begin, let's outline the steps involved in setting up the environment for our CycleCloud SLURM cluster. These steps ensure you have a working CycleCloud VM host before setting up a cluster to leverage InfiniBand with VMSS Flexible.

The variables include:

```shell
set -e

RG=$1
LOCATION=$2
SUBSCRIPTION="REPLACE_WITH_AZURE_SUBSCRIPTION_ID"
IMAGE="azurecyclecloud:azure-cyclecloud:cyclecloud8:latest"
SKU="REPLACE_WITH_AZURE_VM_SIZE"
HOST_VM_NAME="$RG-host-vm"
VMSS_NAME="$RG-flex-vmss"
STORAGE_ACCOUNT="ccstoragelocker"
```

For CycleCloud cluster deployment environment to be up and running, you need the entire script to run successfully. Hence, `set -e` ensures that the script exit whenever any of the commands fail.

Optionally, you can add control flow statements *prior to the variables declaration block* to check whether the first (`$1`) and second (`$2`) arguments have been omitted.

```shell
if [ -z "$1" ]; then
  echo "Please provide the resource group name as the first argument."
  exit 1
fi

if [ -z "$2" ]; then
  echo "Please provide the location as the second argument."
  exit 1
fi
```

### II. 1. CycleCloud Host VM Deployment Steps

1 - Creation of the Resource Group: The resource group will include the host VM, the VMSS, and all the shared resources (e.g., network, storage, etc.).

  ```bash
  az group create \
      -n $RG \
      -l $LOCATION \
      --subscription $SUBSCRIPTION
  ```

2 - Provisioning the Virtual Network: This will facilitate network connectivity for your CycleCloud cluster.

  ```bash
  az network vnet create \
      --name $RG-vnet \
      --address-prefixes "10.0.0.0/16" \
      --resource-group $RG
  ```

3 - Adding a Subnet to the Virtual Network: The default subnet will help with IP address allocation for the host VM and subsequent nodes in the CycleCloud cluster.

  ```bash
  az network vnet subnet create \
    --address-prefixes "10.0.0.0/22" \
    --name "default" \
    --vnet-name $RG-vnet \
    --resource-group $RG
  ```

4 - Creating the CycleCloud Host VM: Set up the CycleCloud host VM, which serves as the control or management node for the cluster.

  ```bash
  az vm create \
    -n $HOST_VM_NAME \
    -g $RG \
    --image $IMAGE \
    --public-ip-sku Standard \
    --size $SKU \
    --admin-username "azureuser" \
    --ssh-key-value "/path/to/.ssh/id_rsa.pub" \
    --vnet-name $RG-vnet \
    --subnet "default" \
    --assign-identity \
    --plan-name "cyclecloud8" \
    --plan-publisher "azurecyclecloud" \
    --plan-product "azure-cyclecloud"
  ```

5 - Deploying an Empty VMSS Flex: The CycleCloud cluster will later scale out from within this VMSS using RDMA-enabled VM sizes to ensure InfiniBand connectivity between the nodes.

  ```bash
  az vmss create \
    -n "$VMSS_NAME" \
    -g $RG \
    --platform-fault-domain-count 1 \
    --orchestration-mode Flexible \
    --single-placement-group false
  ```

6 - Assigning Contributor Role to the host VM Managed Identity: This grants the necessary permissions allowing the control node to create compute nodes and associated resources.

  ```bash
  # Get Host VM Principal ID
  hostVMPrincipalID=$(az vm show \
                            -g $RG \
                            -n "$HOST_VM_NAME" \
                            --query "identity.principalId" \
                            -o tsv)

  # Assign contributor role to the host VM
  az role assignment create \
      --assignee-principal-type ServicePrincipal \
      --assignee-object-id $hostVMPrincipalID \
      --role "Contributor" \
      --scope "/subscriptions/$SUBSCRIPTION"   
  ```

7 - Setting Network Security Group Rules: Enable web access to CycleCloud through the WebUI by defining network security group rules.

  ```bash
  # Get the NSG Name
  nsgName=$(az network nsg list  \
                  -g $RG \
                  --query '[0].name' \
                  -o json | jq -r '.')

  # Set NSG Rule to Allow Web Requests
  az network nsg rule create \
          --nsg-name $nsgName  \
          -g $RG \
          -n "allow-web-req" \
          --destination-port-ranges 80 443 \
          --access Allow \
          --protocol Tcp \
          --priority 107
  ```

8 - Creating the Storage Account: Establish a storage account that will serve as the CycleCloud storage locker, ensuring data persistence and accessibility across the nodes in the cluster.

  ```bash
  az storage account create \
          -n $STORAGE_ACCOUNT \
          -g $RG \
          --sku Standard_LRS
  ```

### II. 2. Accessing the Host VM Securely through Bastion Service

To ensure secure access to the CycleCloud host VM terminal, we'll deploy a bastion service. This service enables convenient and protected SSH connectivity to the host VM, without the need for a public IP address or VPN connection.

```bash
# Create bastion public IP
az network public-ip create \
  --name $RG-bastion-public-ip \
  --resource-group $RG \
  --sku Standard \
  --allocation-method Static

# Create bastion subnet
az network vnet subnet create \
  --address-prefixes "10.0.4.0/24" \
  --name "AzureBastionSubnet" \
  --vnet-name $RG-vnet \
  --resource-group $RG

# Deploy bastion
az network bastion create \
  --name $RG-bastion \
  --public-ip-address $RG-bastion-public-ip \
  --resource-group $RG \
  --vnet-name $RG-vnet \
  --enable-tunneling true
```

At this stage, you are ready to connect to the CycleCloud host VM using bastion. The command below utilizes the `az network bastion ssh` command to establish an SSH connection to the host VM within the specified resource group (`$RG`) using Azure Bastion. It authenticates with an SSH key by pointing to the location of the private key associated with the public key used when creating CycleCloud host VM and using the username `azureuser`.

```bash
# Get the host VM resource ID
cc_host_vm_id=$(az vm show \
                    --name $HOST_VM_NAME \
                    -g $RG \
                    --query 'id' \
                    -o json | jq -r '.')

# SSH into the VM
az network bastion ssh \
  --name $RG-bastion \
  --resource-group $RG \
  --target-resource-id $cc_host_vm_id \
  --auth-type ssh-key \
  --username azureuser \
  --ssh-key "/path/to/.ssh/id_rsa.pem"
```

## III. Installing and Configuring CycleCloud

Now that the environment is prepared, we can proceed with installing and configuring CycleCloud on the CycleCloud host VM. This step lays the foundation for managing and scaling HPC workloads effectively.

11. Installing CycleCloud and Configuring the Account: Utilize the public IP of the CycleCloud host VM to install and configure CycleCloud via the Edge browser. Set up an account using the WebUI.

## Connecting CycleCloud to Azure Resources

To leverage the power of Azure resources, we'll establish connections between CycleCloud and the necessary components.

12. Adding Storage Account Key to the Config File: Enable access to the storage account by adding the storage account key to the CycleCloud config file.
13. Setting Up Private Key for Authentication: Generate a private key file and configure it for authentication with the CycleCloud host VM.

## Scaling Out the SLURM Cluster with VMSS Flex

14. Importing the SLURM Template: Import the SLURM template into CycleCloud, specifying the required parameters such as the CycleCloud subscription, subnet ID, and VMSS Flex ID. This template enables CycleCloud to scale out the SLURM cluster using VMSS Flex.

15. Launching and Monitoring the Cluster: Launch the SLURM cluster and monitor its progress. Verify that the cluster enters the allocation mode, indicating that it's ready for scaling.

16. Adding Nodes to the Cluster: Use the CycleCloud command-line interface (CLI) to add nodes to the cluster. Specify the desired partition and observe the new nodes starting up and joining the cluster.

## Monitoring and Verifying the Cluster

17. Checking Cluster Status: Monitor the status of the cluster to ensure successful provisioning of the nodes. You can use the Azure portal or the CLI to check the status and verify that all nodes are operational.

18. Finalizing the Configuration: Tailor the cluster configuration based on your specific workload requirements. This could involve custom configurations, software installations, or job scheduling setups. Fine-tune the configuration to maximize the efficiency of your HPC workflows.

## Conclusion

Scaling out a CycleCloud SLURM cluster with InfiniBand connectivity on Azure VMSS Flex provides a robust solution for handling demanding HPC workloads. By leveraging the flexibility of VMSS and the performance benefits of InfiniBand, HPC developers, scientists, and business decision-makers can achieve unprecedented scalability, deployment performance, and reliability.

The combination of CycleCloud's powerful orchestration capabilities, Azure's scalable infrastructure, and InfiniBand's high-speed interconnect technology opens up new possibilities for tackling complex computational tasks efficiently. Whether it's running large-scale simulations, data analytics, or scientific research, this setup empowers users to harness the full potential of their HPC workloads.

As HPC continues to evolve and demand grows for more computational power, leveraging technologies like CycleCloud, Azure, and InfiniBand becomes increasingly critical. Embracing these advanced solutions can enable organizations to stay at the forefront of scientific research, engineering simulations, and data-driven decision-making.

So, embrace the power of CycleCloud, leverage Azure's scalability, and harness the benefits of InfiniBand connectivity to unlock the full potential of your HPC workflows. Empower your organization to push the boundaries of computational capabilities and accelerate innovation in your field.

Thank you for joining us on this journey to scale out CycleCloud SLURM clusters with InfiniBand on Azure VMSS Flex. Happy computing, and may your HPC endeavors yield remarkable discoveries and breakthroughs!

