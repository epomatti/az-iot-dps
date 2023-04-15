# Azure IoT Hub - Device Provisioning Service

Create the IoT Hub:

```sh
# IoT Hub
az group create --name IoTEdgeResources --location westus2
az iot hub create --resource-group IoTEdgeResources --name iothub789 --sku F1 --partition-count 2 --mintls "1.2"

# Upgrade the root to V2 if required
az iot hub certificate root-authority set --hub-name iothub789 --certificate-authority v2
```

Create the DPS:

```sh
# Create the DPS
az iot dps create -n dps789 -g IoTEdgeResources -l westus2

# Link with the IoT Hub
hubConnectionString=$(az iot hub connection-string show -n iothub789 --kt primary --query connectionString -o tsv)
az iot dps linked-hub create --dps-name dps789 --resource-group IoTEdgeResources --connection-string $hubConnectionString

# Verify
az iot dps show -n dps789
```

## References

https://youtu.be/o0xtIweuwdA
https://youtu.be/w1qwfIrUFOQ
https://azure.microsoft.com/en-us/blog/the-blueprint-to-securely-solve-the-elusive-zerotouch-provisioning-of-iot-devices-at-scale/
