# Prebuilt Sim enclaves for SGX DCAP Quote Generation and Verification

The documented [installation of DCAP](https://www.intel.com/content/www/us/en/developer/articles/guide/intel-software-guard-extensions-data-center-attestation-primitives-quick-install-guide.html)
is meant for a hardware enabled SGX platform.  This causes problems for those wishing to develop on a non SGX platform.

During the installation some pre built HW enclaves signed by Intel will be installed, 
[prebuilt_dcap_1.13.tar.gz](https://download.01.org/intel-sgx/sgx-dcap/1.13/linux/prebuilt_dcap_1.13.tar.gz).  
This repo provides versions of these enclaves that work in an `SGX_MODE=SIM` environment.

# Installation

Follow the normal installation steps for installing the Intel DCAP libraries and then copy
over the HW enclaves in `/usr/lib/x86_64/` with the ones in this repo.

For example:
> Note: Ubuntu 22.04 currently doesn't have the SDK binaries available one will need to manually build those.
```
wget https://download.01.org/intel-sgx/sgx-dcap/1.13/linux/distro/ubuntu$(source /etc/os-release; echo $VERSION_ID)-server/sgx_debian_local_repo.tgz
wget https://download.01.org/intel-sgx/sgx-dcap/1.13/linux/distro/ubuntu$(source /etc/os-release; echo $VERSION_ID)-server/sgx_linux_x64_sdk_2.16.100.4.bin
sudo /bin/bash ./sgx_linux_x64_sdk_2.16.100.4.bin --prefix=/opt/intel
source /opt/intel/sgxsdk/environment
tar -xzf sgx_debian_local_repo.tgz 
sudo mv sgx_debian_local_repo /usr/local/
sudo dpkg-scanpackages /usr/local/sgx_debian_local_repo | gzip -9c > /usr/local/Packages.gz
sudo echo "deb [trusted=yes] file:/usr/local/sgx_debian_local_repo/ $(source /etc/os-release; echo $VERSION_CODENAME) main" >> /etc/apt/sources.list
sudo apt-get update
sudo apt-get -y install libsgx-quote-ex-dev libsgx-dcap-ql-dev
git clone https://github.com/nick-mobilecoin/prebuild_dcap_sim.git
cd prebuilt_dcap_sim/$(source /etc/os-release; echo $VERSION_CODENAME)
sudo cp libsgx_id_enclave.signed.so.1.11.101.1 /usr/lib/x86_64-linux-gnu/libsgx_id_enclave.signed.so.1.11.101.1
sudo cp libsgx_pce.signed.so.1.16.100.0 /usr/lib/x86_64-linux-gnu/libsgx_pce.signed.so.1.16.100.0
sudo cp libsgx_qe3.signed.so.1.11.101.1 /usr/lib/x86_64-linux-gnu/libsgx_qe3.signed.so.1.11.101.1
```

## Manuall building Intel SGX SDK (for Ubuntu 22.04)
```
sudo apt-get -y install build-essential ocaml ocamlbuild automake autoconf libtool wget python-is-python3 libssl-dev git cmake perl libssl-dev libcurl4-openssl-dev protobuf-compiler libprotobuf-dev debhelper cmake reprepro unzip
git clone --branch sgx_2.16 https://github.com/intel/linux-sgx
cd linux-sgx
make preparation
CXXFLAGS=-Wno-deprecated-declarations make sdk_install_pkg
sudo ./linux/installer/bin/sgx_linux_x64_sdk_2.16.100.4.bin --prefix=/opt/intel
make deb_psw_pkg
sudo apt-get install -y ./linux/installer/deb/sgx-aesm-service/libsgx-ae-qe3_1.13.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/sgx-aesm-service/libsgx-ae-id-enclave_1.13.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/libsgx-urts/libsgx-urts_2.16.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/sgx-aesm-service/libsgx-qe3-logic_1.13.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/sgx-aesm-service/libsgx-ae-pce_2.16.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/sgx-aesm-service/libsgx-pce-logic_1.13.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/sgx-aesm-service/libsgx-dcap-ql_1.13.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/sgx-aesm-service/libsgx-dcap-ql-dev_1.13.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/libsgx-quote-ex/libsgx-quote-ex_2.16.100.4-$(lsb_release -cs)1_amd64.deb
sudo apt-get install -y ./linux/installer/deb/libsgx-quote-ex/libsgx-quote-ex-dev_2.16.100.4-$(lsb_release -cs)1_amd64.deb
```

# Rebuilding the Enclaves

The enclaves are built from the https://github.com/intel/linux-sgx repo.

The repo is not ready to build the enclaves in SIM mode. A few of the make files need to be patched.
This patches are based on the 2.16 release.  The patches are provided in this repo as bundles:
* [pce_makefile.bundle](pce_makefile.bundle) patches the `linux-sgx` repo directly
* [sim_quoted_enclave.bundle](sim_quoted_enclave.bundle) patches an external repo brought in by `make preparation`

> These bundles will patch the repos in the commands below
> 
> Note: there are 3 entries to fill out in the below commands, 
> the `git clone` command and the `git fetch` commands.


```
git clone --branch <release-version> https://github.com/intel/linux-sgx.git
cd linux-sgx
make preparation
git fetch <path/to/this/repo>/pce_makefile.bundle
git cherry-pick FETCH_HEAD
cd external/dcap_source
git fetch <path/to/this/repo>/sim_quote_enclave.bundle
git cherry-pick FETCH_HEAD
cd -
git clone --branch sgx_2.16 https://github.com/intel/linux-sgx.git
cd linux-sgx
make preparation
openssl genrsa -out sim_private_key.pem -3 3072
cp sim_private_key.pem psw/ae/pce/pce_sim_private_key.pem
APPLY_PATCH psw/ae/pce/Makefile
(cd psw/ae/pce && make SGX_MODE=SIM)
cp psw/ae/pce/libsgx_pce.signed.so .
cp sim_private_key.pem external/dcap_source/QuoteGeneration/quote_wrapper/quote/enclave/linux/qe3_sim_private_key.pem
(cd external/dcap_source/QuoteGeneration/quote_wrapper/quote/enclave/linux && make SGX_MODE=SIM)
cp external/dcap_source/QuoteGeneration/quote_wrapper/quote/enclave/linux/libsgx_qe3.signed.so .
cp sim_private_key.pem external/dcap_source/QuoteGeneration/quote_wrapper/quote/id_enclave/linux/id_sim_private_key.pem
(cd external/dcap_source/QuoteGeneration/quote_wrapper/quote/id_enclave/linux && make SGX_MODE=SIM)
cp external/dcap_source/QuoteGeneration/quote_wrapper/quote/id_enclave/linux/libsgx_id_enclave.signed.so .
```

After this one should be left with the following artifacts at the root of the repo
* `sim_private_key.pem`
* `libsgx_pce.signed.so`
* `libsgx_qe3.signed.so`
* `libsgx_id_enclave.signed.so`

These files can be copied over the hardware versions that were installed to `/usr/lib/x86_64/`.  
Suggest one copies over the real files and *not* the links this means the copy command will be something like:
```
cp libsgx_id_enclave.signed.so /usr/lib/x86_64-linux-gnu/libsgx_id_enclave.signed.so.1.11.101.1
cp libsgx_pce.signed.so /usr/lib/x86_64-linux-gnu/libsgx_pce.signed.so.1.16.100.0
cp libsgx_qe3.signed.so /usr/lib/x86_64-linux-gnu/libsgx_qe3.signed.so.1.11.101.1
```

The files commited to the repo were named on the HW version to avoid ambiguity if one just needs to install these.
