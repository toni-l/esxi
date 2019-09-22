Step 1 - Download the ZIP file for the specific version of your ESXi host and upload to ESXi host using SCP or Datastore Browser

Step 2 - Place the ESXi host into Maintenance Mode using the vSphere UI or CLI (e.g. esxcli system maintenanceMode set -e true)

Step 3 - Install the Offline Bundle by running the following command on ESXi Shell:
esxcli software vib install -d /path/to/the offline bundle
Step 4 - Plug-in the USB NIC and reboot for the change to go into effect. Once the host has rebooted, ESXi should automatically pickup and claim the USB NIC (e.g. vusb0)

Note: Secure Boot can not be enabled if you decide to use the USB NIC as your primary NIC for Management Network. Since the settings do not persist, you will need to create a startup script (see Instructions for more details) and this is not allowed when Secure Boot is enabled. If USB NIC is not your primary NIC for the Management Network, then you do not have to disable Secure Boot

Persisting USB NIC Bindings
Currently there is a limitation in ESXi where USB NIC bindings are picked up much later in the boot process and to ensure settings are preserved upon a reboot, the following needs to be added to /etc/rc.local.d/local.sh based on your configurations.

Standard Virtual Switch (VSS)
Here is an example which binds the physical USB NIC (vsub0) to Standard vSwitch and the respective portgroups that are also attached on the VSS
vusb0_status=$(esxcli network nic get -n vusb0 | grep 'Link Status' | awk '{print $NF}')
count=0
while [[ $count -lt 20 && "${vusb0_status}" != "Up" ]] ]
do
    sleep 10
    count=$(( $count + 1 ))
    vusb0_status=$(esxcli network nic get -n vusb0 | grep 'Link Status' | awk '{print $NF}')
done

if [ "${vusb0_status}" = "Up" ]; then
    esxcfg-vswitch -L vusb0 vSwitch0
    esxcfg-vswitch -M vusb0 -p "Management Network" vSwitch0
    esxcfg-vswitch -M vusb0 -p "VM Network" vSwitch0
fi
Distributed Virtual Switch (VDS)
Here is an example which binds the physical USB NIC (vsub0) to a Distributed Virtual Switch (VDS). You will need to update both VDS_NAME and VDS_PORT_ID variable to match your enviornment
VDS_NAME=VDS
VDS_PORT_ID=8

vusb0_status=$(esxcli network nic get -n vusb0 | grep 'Link Status' | awk '{print $NF}')
count=0
while [[ $count -lt 20 && "${vusb0_status}" != "Up" ]] ]
do
    sleep 10
    count=$(( $count + 1 ))
    vusb0_status=$(esxcli network nic get -n vusb0 | grep 'Link Status' | awk '{print $NF}')
done

if [ "${vusb0_status}" = "Up" ]; then
    esxcfg-vswitch -P vusb0 -V ${VDS_PORT_ID} ${VDS_NAME}
fi
