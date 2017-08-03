---
title: Convert a Windows virtual machine from unmanaged disks to managed disks - Azure Managed Disks | Microsoft Docs
description: How to convert a Windows VM from unmanaged disks to managed disks by using PowerShell in the Resource Manager deployment model
services: virtual-machines-windows
documentationcenter: ''
author: cynthn
manager: timlt
editor: ''
tags: azure-resource-manager

ms.assetid: 
ms.service: virtual-machines-windows
ms.workload: infrastructure-services
ms.tgt_pltfrm: vm-windows
ms.devlang: na
ms.topic: article
ms.date: 06/23/2017
ms.author: cynthn
---

# Convert a Windows virtual machine from unmanaged disks to managed disks

If you have existing Windows virtual machines (VMs) that use unmanaged disks, you can convert the VMs to use managed disks through the [Azure Managed Disks](../../storage/storage-managed-disks-overview.md) service. This process converts both the OS disk and any attached data disks.

This article shows you how to convert VMs by using Azure PowerShell. If you need to install or upgrade it, see [Install and configure Azure PowerShell](/powershell/azure/install-azurerm-ps.md).

## Before you begin


* Review [Plan for the migration to Managed Disks](on-prem-to-azure.md#plan-for-the-migration-to-managed-disks).

[!INCLUDE [virtual-machines-common-convert-disks-considerations](../../../includes/virtual-machines-common-convert-disks-considerations.md)]




## Convert single-instance VMs
This section covers how to convert single-instance Azure VMs from unmanaged disks to managed disks. (If your VMs are in an availability set, see the next section.) 

1. Deallocate the VM by using the [Stop-AzureRmVM](/powershell/module/azurerm.compute/stop-azurermvm) cmdlet. The following example deallocates the VM named `myVM` in the resource group named `myResourceGroup`: 

  ```powershell
  $rgName = "myResourceGroup"
  $vmName = "myVM"
  Stop-AzureRmVM -ResourceGroupName $rgName -Name $vmName -Force
  ```

2. Convert the VM to managed disks by using the [ConvertTo-AzureRmVMManagedDisk](/powershell/module/azurerm.compute/convertto-azurermvmmanageddisk) cmdlet. The following process converts the previous VM, including the OS disk and any data disks:

  ```powershell
  ConvertTo-AzureRmVMManagedDisk -ResourceGroupName $rgName -VMName $vmName
  ```

3. Start the VM after the conversion to managed disks by using [Start-AzureRmVM](/powershell/module/azurerm.compute/start-azurermvm). The following example restarts the previous VM:

  ```powershell
  Start-AzureRmVM -ResourceGroupName $rgName -Name $vmName
  ```


## Convert VMs in an availability set

If the VMs that you want to convert to managed disks are in an availability set, you first need to convert the availability set to a managed availability set.

1. Convert the availability set by using the [Update-AzureRmAvailabilitySet](/powershell/module/azurerm.compute/update-azurermavailabilityset) cmdlet. The following example updates the availability set named `myAvailabilitySet` in the resource group named `myResourceGroup`:

  ```powershell
  $rgName = 'myResourceGroup'
  $avSetName = 'myAvailabilitySet'

  $avSet = Get-AzureRmAvailabilitySet -ResourceGroupName $rgName -Name $avSetName
  Update-AzureRmAvailabilitySet -AvailabilitySet $avSet -Sku Aligned 
  ```

  If the region where your availability set is located has only 2 managed fault domains but the number of unmanaged fault domains is 3, this command shows an error similar to "The specified fault domain count 3 must fall in the range 1 to 2." To resolve the error, update the fault domain to 2 and update `Sku` to `Aligned` as follows:

  ```powershell
  $avSet.PlatformFaultDomainCount = 2
  Update-AzureRmAvailabilitySet -AvailabilitySet $avSet -Sku Aligned
  ```

2. Deallocate and convert the VMs in the availability set. The following script deallocates each VM by using the [Stop-AzureRmVM](/powershell/module/azurerm.compute/stop-azurermvm) cmdlet, converts it by using [ConvertTo-AzureRmVMManagedDisk](/powershell/module/azurerm.compute/convertto-azurermvmmanageddisk), and restarts it by using [Start-AzureRmVM](/powershell/module/azurerm.compute/start-azurermvm):

  ```powershell
  $avSet = Get-AzureRmAvailabilitySet -ResourceGroupName $rgName -Name $avSetName

  foreach($vmInfo in $avSet.VirtualMachinesReferences)
  {
     $vm = Get-AzureRmVM -ResourceGroupName $rgName | Where-Object {$_.Id -eq $vmInfo.id}
     Stop-AzureRmVM -ResourceGroupName $rgName -Name $vm.Name -Force
     ConvertTo-AzureRmVMManagedDisk -ResourceGroupName $rgName -VMName $vm.Name
     Start-AzureRmVM -ResourceGroupName $rgName -Name $vmName
  }
  ```


## Convert standard managed disks to premium
After you've converted your VM to managed disks, you can switch between the storage types. You can also have a mixture of disks that use standard and premium storage. 

In the following example, we show how to switch from standard to premium storage. To use premium managed disks, your VM must use a [VM size](sizes.md) that supports premium storage. This example also switches to a size that supports premium storage.

```powershell
$rgName = 'myResourceGroup'
$vmName = 'YourVM'
$size = 'Standard_DS2_v2'
$vm = Get-AzureRmVM -Name $vmName -resourceGroupName $rgName

# Stop and deallocate the VM before changing the size
Stop-AzureRmVM -ResourceGroupName $rgName -Name $vmName -Force

# Change the VM size to a size that supports premium storage
$vm.HardwareProfile.VmSize = $size
Update-AzureRmVM -VM $vm -ResourceGroupName $rgName

# Get all disks in the resource group of the VM
$vmDisks = Get-AzureRmDisk -ResourceGroupName $rgName 

# For disks that belong to the selected VM, convert to premium storage
foreach ($disk in $vmDisks)
{
	if ($disk.OwnerId -eq $vm.Id)
	{
		$diskUpdateConfig = New-AzureRmDiskUpdateConfig –AccountType PremiumLRS
		Update-AzureRmDisk -DiskUpdate $diskUpdateConfig -ResourceGroupName $rgName `
		-DiskName $disk.Name
	}
}

Start-AzureRmVM -ResourceGroupName $rgName -Name $vmName
```

## Troubleshooting

If there is an error during conversion, or if a VM is in a failed state because of issues in a previous conversion, run the `ConvertTo-AzureRmVMManagedDisk` cmdlet again. A simple retry usually unblocks the situation.


## Managed disks and Azure Storage Service Encryption

You can't use the preceding steps to convert an unmanaged disk into a managed disk if the unmanaged disk is in a storage account that has ever been encrypted through [Azure Storage Service Encryption](../../storage/storage-service-encryption.md?toc=%2fazure%2fvirtual-machines%2fwindows%2ftoc.json). The following steps detail how to copy and use unmanaged disks that have been in an encrypted storage account:

1. Copy the VHD by using [AzCopy](../../storage/storage-use-azcopy.md) to a storage account that has never been enabled for Azure Storage Service Encryption.

2. Use the copied VM in one of the following ways:

   * Create a VM that uses managed disks and specify that VHD file during creation by using `New-AzureRmVm`.

   * Attach the copied VHD by using `Add-AzureRmVmDataDisk` to a running VM that uses managed disks.


## Next steps

Take a read-only copy of a VM by using [snapshots](snapshot-copy-managed-disk.md).

