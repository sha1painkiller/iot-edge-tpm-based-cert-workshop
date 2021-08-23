# TPM based certificate
## Deploy a VM with trusted launch enabled
[[official doc link]](https://docs.microsoft.com/en-us/azure/virtual-machines/trusted-launch-portal)

**[important note]** you have to sign in to the Azure **preview** [portal](https://aka.ms/TL_preview)

*Make sure you enabled vTPM in Advanced tab* during VM creation

![](https://i.imgur.com/NY6oxzC.png)

## Create CA root certificate
1. ssh to Azure VM console
1. generate cert
    * use "**1234**" as passphrase all the way to simply the workshop
    * generate CA root RSA key pair
    > openssl genrsa -aes256 -out ca.key.pem 4096
    * use the generated CA root key to generate a X.509 self-signed certificate
    > openssl req -key ca.key.pem -new -x509 -days 3650 -sha256 -out ca.cert.pem
    * generate a RSA key pair as Edge key (will be used in later steps)
    > openssl genrsa -aes256 -out edge.key.pem 2048
1. download the "ca.key.pem" to local desktop

## DPS certificate registration
1. [Set up the IoT Hub Device Provisioning Service](https://docs.microsoft.com/en-us/azure/iot-dps/quick-setup-auto-provision)
2. In the left pane of DPS portal, select "**Certificates**". Then "**Add**" the downloaded certificate (ie. ca.cert.pem) to the portal.
1. Verify the uploaded certificate - click "**Generate verification code**", and then copy the "**Verification code**"
1. Generate verification certificate
    > openssl genrsa -aes256 -out verify.key.pem 2048
    * Use copied **Verification Code** as your CN(Common Name) field entry
    > openssl req -new -sha256 -key verify.key.pem -out verify.csr.pem
    * use CA certificate to sign the verification certificate
    > openssl x509 -req -in verify.csr.pem -CA ca.cert.pem -CAkey ca.key.pem -CAcreateserial -out verify.cert.pem -days 1 -sha256
1. download the "**verify.cert.pem**" to local desktop and upload it DPS portal for verification

### Create Enrollment Group
![](https://i.imgur.com/a1FxWBV.png)

### copy our ‘ca’ files into a known location for later certificate request
sudo mkdir /var/secrets && sudo chmod 777 /var/secrets && sudo cp ca.*.pem /var/secrets

## Install Azure IoT Edge 1.2 on Ubuntu 20.04
> curl https://packages.microsoft.com/config/ubuntu/18.04/multiarch/prod.list > ./microsoft-prod.list
sudo cp ./microsoft-prod.list /etc/apt/sources.list.d/
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo cp ./microsoft.gpg /etc/apt/trusted.gpg.d/
sudo apt update && sudo apt-get install -y moby-engine
sudo apt-get install -y aziot-edge

## Install tpm2-pkcs11
- install the tpm2 software and configure a slot in the TPM from https://azure.github.io/iot-identity-service/pkcs11/tpm2-pkcs11.html
    > wget https://raw.githubusercontent.com/ksaye/IoTDemonstrations/master/installTPM2Tools.sh
    > chmod +x installTPM2Tools.sh 
    > ./installTPM2Tools.sh

- To see the tokens, run the following command.  Note it runs as aziotks, which has permissions.  ‘Token 0:’ is what we want
    > export PKCS11_LIB_PATH=/usr/local/lib/libtpm2_pkcs11.so
    > sudo -u aziotks pkcs11-tool --module "\$PKCS11_LIB_PATH" --label="tpm-rsa-pair" --login --keypairgen --usage-sign --key-type rsa:2048
    > sudo -u aziotks pkcs11-tool --module "\$PKCS11_LIB_PATH" --list-objects --login

- Download our openssl.conf and install libp11 to support to support OpenSSL 
    > cd /var/secrets
    > wget https://raw.githubusercontent.com/ksaye/IoTDemonstrations/master/pkcs11_openssl.conf
    > sudo apt install -y pkgconf libssl-dev
    > mkdir ~/libp11 
    > cd ~/libp11
    > wget https://github.com/OpenSC/libp11/releases/download/libp11-0.4.11/libp11-0.4.11.tar.gz
    > tar -xvf libp11-0.4.11.tar.gz
    > cd libp11-0.4.11
    > ./configure
    > make
    > sudo make install

- generate the private key and make a CSR Request.  Must be aziotks to interact with the TPM
    > sudo -u aziotks OPENSSL_CONF=/var/secrets/pkcs11_openssl.conf openssl req -new -subj /CN=edgetpm -sha256 -engine pkcs11 -keyform engine -key slot_1-label_tpm-rsa-pair -out /var/secrets/edgetpm.csr.pem

- Sign the cert request with our simple OpenSSL CA – this could be to a real CA as well, as this is just a standard cert signing task
    > sudo openssl x509 -req -in /var/secrets/edgetpm.csr.pem -CA /var/secrets/ca.cert.pem -CAkey /var/secrets/ca.key.pem -CAcreateserial -out /var/secrets/edgetpm.cert.pem -days 365 -sha256

- Note in our /var/secretes directory, we do not have an edgetpm.key.pem file, because that is stored in the TPM

## Configure IoT Edge 1.2
- Modify the /etc/aziot/config.toml with your id_scope, identity cert / key and the pkcs11 path as shown below
    > [provisioning]
source = "dps"
global_endpoint = "https://global.azure-devices-provisioning.net/"
id_scope = "<dps scope-id>"	# change this with your scope id
[provisioning.attestation]
method = "x509"
registration_id = "edgetpm" # the device ID appeared in IoT Hub
identity_cert = "file:///var/secrets/edgetpm.cert.pem"
identity_pk = "pkcs11:token=IoTEdgeCert?pin-value=1234"
[aziot_keys]
pkcs11_lib_path = "/usr/local/lib/libtpm2_pkcs11.so"
pkcs11_base_slot = "pkcs11:token=IoTEdgeCert?pin-value=1234"


- Run the following commands to apply the settings, view logs and see the running module
    > sudo iotedge config apply
    > sudo iotedge system logs
    > sudo iotedge module list 

## Check device on target IoT Hub
You can see the device appeared on IoT Hub upon successful registration.
![](https://i.imgur.com/HSKSnM4.png)
