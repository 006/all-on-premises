# [Helm](https://helm.sh/)

## Install via get_helm.sh

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Install from Binary Releases

Download your [desired version from github](https://github.com/helm/helm/releases)

```bash
# Unpack it
tar -zxvf helm-v3.0.0-linux-amd64.tar.gz

# Find the helm binary in the unpacked directory, and move it to its desired destination
sudo mv /zzz/helm /usr/local/bin/

# or create a link
sudo ln -s /zzz/helm /usr/local/bin/

helm version
# Output should be similar to following:
version.BuildInfo{Version:"v3.17.1", GitCommit:"980d8ac1939e39138101364400756af2bdee1da5", GitTreeState:"clean", GoVersion:"go1.23.5"}
```
