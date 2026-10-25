# Fix iPhone USB Hotspot on Windows 11

## Problem

The iPhone Personal Hotspot worked over Wi-Fi, but USB tethering did not work.

- Apple Devices recognized the iPhone.
- Windows detected the iPhone's USB devices.
- The `AppleUSBEthernet` interface had no working driver.
- Installing the correct Ethernet driver fixed USB tethering.

## Environment

- Windows 11
- iPhone connected via USB
- Apple Devices installed
- PowerShell available

## Solution

### 1. Connect and identify the iPhone

Install **Apple Devices** from the Microsoft Store if it is not already installed.

Connect the iPhone using USB, unlock it, and tap **Trust** if prompted.

Open PowerShell and identify the affected device:

```powershell
Get-PnpDevice
```

The affected interface was:

```text
AppleUSBEthernet
USB\VID_05AC&PID_12A8&MI_02
```

The USB device driver was present, but the Ethernet interface had no bound driver.

### 2. Download the Ethernet driver

Download the driver CAB from Microsoft's update server:

[Download Apple Ethernet driver CAB](https://catalog.s.download.windowsupdate.com/c/msdownload/update/driver/drvs/2017/11/netaapl_7503681835e08ce761c52858949731761e1fa5a1.cab)

> Note: Although the CAB was published in 2017, its INF identifies the driver version as `1.8.5.1`, dated July 15, 2013. The package included the missing co-installer DLL. The catalog signature was verified as valid.

Save the CAB to your Downloads folder.

### 3. Extract the driver

Run PowerShell:

```powershell
mkdir C:\AppleEthernetDriver
```

Extract the CAB:

```powershell
expand.exe -F:* "$env:USERPROFILE\Downloads\netaapl_7503681835e08ce761c52858949731761e1fa5a1.cab" "C:\AppleEthernetDriver"
```

The extracted files included:

- `netaapl64.inf` — driver installation information
- `netaapl64.sys` — driver binary
- `netaapl64.cat` — signature catalog
- `WdfCoInstaller01009.dll` — required co-installer
- `netaapl64.pdb` — debugging information

Verify the files:

```powershell
dir C:\AppleEthernetDriver
```

### 4. Verify the hardware ID and driver package

Check the INF:

```powershell
Select-String -Path "C:\AppleEthernetDriver\netaapl64.inf" -Pattern "VID_05AC&PID_12A8&MI_02","DriverVer","CatalogFile","WdfCoInstaller"
```

Confirm that it contains the iPhone Ethernet interface hardware ID:

```text
USB\Vid_05ac&Pid_12A8&Mi_02
```

Check the catalog signature:

```powershell
Get-AuthenticodeSignature "C:\AppleEthernetDriver\netaapl64.cat" | Format-List Status,StatusMessage,SignerCertificate
```

Expected result:

```text
Status : Valid
Signer : Microsoft Windows Hardware Compatibility Publisher
```

Do not rename missing DLLs or manually modify the INF. Use a complete driver package instead.

### 5. Install the Ethernet driver

Open PowerShell **as Administrator** and run:

```powershell
pnputil /add-driver "C:\AppleEthernetDriver\netaapl64.inf" /install
```

Successful output looked like:

```text
Driver package added successfully.
Published Name: oem57.inf
Driver package installed on device:
USB\VID_05AC&PID_12A8&MI_02\...
```

The published name (`oem57.inf`) may differ on another computer.

### 6. Enable USB tethering

1. Connect the iPhone to the PC by USB.
2. Unlock the iPhone.
3. Open **Settings → Personal Hotspot**.
4. Enable **Allow Others to Join**.
5. Tap **Trust** if prompted.

Keep the iPhone connected while Windows detects the network adapter.

### 7. Verify the connection

List network adapters:

```powershell
Get-NetAdapter -IncludeHidden |
    Format-Table Name, InterfaceDescription, Status, LinkSpeed
```

Check IP configuration:

```powershell
ipconfig
```

Look for an Apple Ethernet network adapter and an assigned IPv4 address.

The connection should now work over USB without relying on the PC's Wi-Fi connection to the iPhone hotspot.

## Troubleshooting notes

### Driver package fails to install

- Confirm that the CAB was extracted successfully.
- Check that `netaapl64.inf`, `netaapl64.sys`, `netaapl64.cat`, and `WdfCoInstaller01009.dll` are present.
- Verify that the INF supports `USB\VID_05AC&PID_12A8&MI_02`.
- Check the exact error from `pnputil` before trying other driver packages.

### No Apple Ethernet adapter appears

- Unlock the iPhone and reconnect the USB cable.
- Enable Personal Hotspot.
- Accept the iPhone's Trust prompt.
- Try another USB port or a data-capable cable.
- Check Device Manager for driver errors.

## Notes

- The fix was installing the Ethernet driver directly with `pnputil`.
- The Apple USB device driver was already present.
- An older Apple Mobile Device Support installer had previously failed, so reinstalling the entire package was not necessary.
- Keep `C:\AppleEthernetDriver` as a backup of the extracted package.
- Driver packages and compatibility can change with future Windows and iOS updates. Verify the package and signature before installing it on another computer.
