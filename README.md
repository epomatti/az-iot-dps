# Azure IoT Hub - Device Provisioning Service

This demonstration generally follows the [this example](https://learn.microsoft.com/en-us/azure/iot-dps/tutorial-custom-hsm-enrollment-group-x509?tabs=linux&pivots=programming-language-nodejs).

## 1 - IoT Hub

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

## 2 - Root Certificate

Prepare the OpenSSL structure:

```
cd openssl

mkdir certs csr newcerts private
touch index.txt
openssl rand -hex 16 > serial
```

Create the root CA private key:

```
openssl genrsa -aes256 -passout pass:1234 -out ./private/azure-iot-test-only.root.ca.key.pem 4096
```

Create the root CA certificate:

```
openssl req -new -x509 -config ./openssl_root_ca.cnf -passin pass:1234 -key ./private/azure-iot-test-only.root.ca.key.pem -subj '/CN=Azure IoT Hub CA Cert Test Only' -days 30 -sha256 -extensions v3_ca -out ./certs/azure-iot-test-only.root.ca.cert.pem
```

Examine the certificate:

```
openssl x509 -noout -text -in ./certs/azure-iot-test-only.root.ca.cert.pem
```

## 3 - Intermediate CA certificate

Create the CA private key:

```
openssl genrsa -aes256 -passout pass:1234 -out ./private/azure-iot-test-only.intermediate.key.pem 4096
```

Create the CSR:

```
openssl req -new -sha256 -passin pass:1234 -config ./openssl_device_intermediate_ca.cnf -subj '/CN=Azure IoT Hub Intermediate Cert Test Only' -key ./private/azure-iot-test-only.intermediate.key.pem -out ./csr/azure-iot-test-only.intermediate.csr.pem
```

Sign the intermediate certificate with the root CA certificate:

```
openssl ca -batch -config ./openssl_root_ca.cnf -passin pass:1234 -extensions v3_intermediate_ca -days 30 -notext -md sha256 -in ./csr/azure-iot-test-only.intermediate.csr.pem -out ./certs/azure-iot-test-only.intermediate.cert.pem
```

Examine:

```
openssl x509 -noout -text -in ./certs/azure-iot-test-only.intermediate.cert.pem
```

## 4 - Device Certificates

Create the Device-01 private key:

```
openssl genrsa -out ./private/device-01.key.pem 4096
```

Create the Device-01 CSR:

> CN must follow standard

```
openssl req -config ./openssl_device_intermediate_ca.cnf -key ./private/device-01.key.pem -subj '/CN=device-01' -new -sha256 -out ./csr/device-01.csr.pem
```

Sign the certificate:

```
openssl ca -batch -config ./openssl_device_intermediate_ca.cnf -passin pass:1234 -extensions usr_cert -days 30 -notext -md sha256 -in ./csr/device-01.csr.pem -out ./certs/device-01.cert.pem
```

Examine the certificate:

```
openssl x509 -noout -text -in ./certs/device-01.cert.pem
```

Create the certificate chain for Device-01:

```
cat ./certs/device-01.cert.pem ./certs/azure-iot-test-only.intermediate.cert.pem ./certs/azure-iot-test-only.root.ca.cert.pem > ./certs/device-01-full-chain.cert.pem
```

## 5 - IoT DPS Config

Upload and verify the certificate:

```
az iot dps certificate create -n "Test-Only-Root" --dps-name dps789 -g IoTEdgeResources -p certs/azure-iot-test-only.root.ca.cert.pem -v true
```

Create the enrollment group:

```sh
az iot dps enrollment-group create -n dps789 -g IoTEdgeResources\
    --root-ca-name "Test-Only-Root" \
    --secondary-root-ca-name "Test-Only-Root" \
    --enrollment-id "DefaultGroup" \
    --provisioning-status "enabled" \
    --reprovision-policy "reprovisionandmigratedata" \
    --iot-hubs "iothub789.azure-devices.net" \
    --allocation-policy "hashed" \
    --edge-enabled false \
    --tags '{ "Environment": "Staging"}' \
    --props '{ "Debug": "false"}'
```

## Device Config

Copy the full chain cert and the private key to the device:

```
mkdir device/certs
cp openssl/certs/device-01-full-chain.cert.pem device/config/full-chain.cert.pem
cp openssl/private/device-01.key.pem device/config/key.pem
```

For development purposes, create the `.env` file in the "device" directory:

```sh
# Global DPS hostname
PROVISIONING_HOST="global.azure-devices-provisioning.net"

# ID Scope from DPS
PROVISIONING_IDSCOPE="0ne009ECC61"

# Device ID (must match certificate)
PROVISIONING_REGISTRATION_ID="device-01"

# Public Cert (Full Chain)
CERTIFICATE_FILE="config/full-chain.cert.pem"

# Private Key
KEY_FILE="config/key.pem"
```

Register the device:

```sh
node register_x509.js
```

## References

https://youtu.be/o0xtIweuwdA
https://youtu.be/w1qwfIrUFOQ
https://azure.microsoft.com/en-us/blog/the-blueprint-to-securely-solve-the-elusive-zerotouch-provisioning-of-iot-devices-at-scale/
https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-x509ca-concept
https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-x509ca-overview
https://www.youtube.com/watch?v=szagwwSLbXo