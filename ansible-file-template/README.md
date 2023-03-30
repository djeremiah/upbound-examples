# Upbound Example - Using provider-ansible to deploy a file template to an existing VM

## Running the demo
- Install UXP, provider-aws, and provider-ansible
- Configure provider-aws as normal
- Create a public/private keypair in id_rsa/id_rsa.pub
- Copy the public key into instance.yaml so that the EC2 instance will have it available for ansible ssh access
- `kubectl apply -f instance.yaml` to provision the EC2 environment
- Get the public ip from the EC2 instance and insert it into ansiblerun.yaml
- Create a connection secret for the ansible provider `kubectl create secret generic ansible-ssh-access -n upbound-system --from-file=ssh-privatekey=id_rsa`
- Configure provider-ansible `kubectl apply -f ansible.ProviderConfig.yaml`
- Deploy the ansible run `kubectl apply -f ansiblerun.yaml`
- You can confirm that the file `/etc/foo.conf` has been created by connecting to the EC2 instance and listing the directory
