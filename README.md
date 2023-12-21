This script is to clone a VM that is not in AV Zone, to same region same RG with AV Zone

# Login to Azure
Connect-AzAccount

# Get the list of subscriptions
$subscriptions = Get-AzSubscription

# Display subscriptions and ask the user to select one
Write-Host "Please select a subscription:"
$subscriptions | ForEach-Object { Write-Host $_.Name }
$selectedSubscriptionName = Read-Host "Enter Subscription Name"
Set-AzContext -SubscriptionName $selectedSubscriptionName

# Ask for the source VM details and target settings
$sourceVmName = Read-Host "Enter the Source VM Name"
$resourceGroupName = Read-Host "Enter the Resource Group Name of the Source VM"
$availabilityZone = Read-Host "Enter the Availability Zone for the new VM"
$newVmName = Read-Host "Enter the New VM Name"

# Proceed with the rest of the script

# Step 1: Get the source VM details
$sourceVm = Get-AzVM -Name $sourceVmName -ResourceGroupName $resourceGroupName
Write-Host "Stopping the source VM: $sourceVmName..."
Stop-AzVM -Name $sourceVmName -ResourceGroupName $resourceGroupName -Force -AsJob | Wait-Job
Write-Host "Source VM stopped."


# Step 2: Create and copy snapshots for the OS disk
$timestamp = Get-Date -Format "yyyyMMddHHmmss"
$snapshotName = $sourceVmName + "-OSDisk-Snapshot-" + $timestamp
$osDisk = Get-AzDisk -ResourceGroupName $resourceGroupName -DiskName $sourceVm.StorageProfile.OsDisk.Name
$snapshotConfig = New-AzSnapshotConfig -SourceUri $osDisk.Id -Location $sourceVm.Location -CreateOption Copy 
New-AzSnapshot -Snapshot $snapshotConfig -SnapshotName $snapshotName -ResourceGroupName $resourceGroupName



do {
    Start-Sleep -Seconds 10
    $snapshotStatus = Get-AzSnapshot -SnapshotName $snapshotName -ResourceGroupName $resourceGroupName
} while ($snapshotStatus.ProvisioningState -ne "Succeeded")


# Step 3: Create a new managed disk in the same region from the snapshot
$timestamp = Get-Date -Format "yyyyMMddHHmmss"
$newDiskName = $sourceVmName + "-OSDisk-Copy-" + $timestamp
$diskConfig = New-AzDiskConfig -Location $sourceVm.Location -SourceResourceId (Get-AzSnapshot -ResourceGroupName $resourceGroupName -SnapshotName $snapshotName).Id -CreateOption Copy -Zone $availabilityZone
$newDisk = New-AzDisk -Disk $diskConfig -DiskName $newDiskName -ResourceGroupName $resourceGroupName

do {
    Start-Sleep -Seconds 10
    $diskStatus = Get-AzDisk -DiskName $newDiskName -ResourceGroupName $resourceGroupName
} while ($diskStatus.ProvisioningState -ne "Succeeded")

# Step 4: Create a new network interface in the specified Availability Zone
$nicName = $newVmName + "-nic"
$subnetId = (Get-AzVirtualNetwork -ResourceGroupName $resourceGroupName).Subnets[0].Id
$nic = New-AzNetworkInterface -Name $nicName -ResourceGroupName $resourceGroupName -Location $sourceVm.Location -SubnetId $subnetId



# Step 5: Create the new VM configuration and set the OS disk
$newVm = New-AzVMConfig -VMName $newVmName -VMSize $sourceVm.HardwareProfile.VmSize -Zone $availabilityZone
$newVm = Set-AzVMOSDisk -VM $newVm -ManagedDiskId $newDisk.Id -CreateOption Attach -Windows
$newVm = Add-AzVMNetworkInterface -VM $newVm -Id $nic.Id


# Add a 2-minute pause before creating the VM
Write-Host "Pausing for 2 minutes to ensure all resources are ready..."
Start-Sleep -Seconds 120


# Step 6: Create the new VM in Azure
New-AzVM -VM $newVm -ResourceGroupName $resourceGroupName -Location $sourceVm.Location


do {
    Start-Sleep -Seconds 15
    $vmStatus = Get-AzVM -Name $newVmName -ResourceGroupName $resourceGroupName -Status
} while ($vmStatus.Statuses[1].Code -ne "PowerState/running")
