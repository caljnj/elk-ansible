
# intro


the idea of this is to create the pretend http certificates and keys for the test environments based on the servers in the inventory. 

it should do the following: 

| task | tech | details | 
|----|-----|-----|
| create the certs | ansible | |
| add certs to vault | python | |


# create the certs

this will use the same inventory  used for the main playbook

it will then create all the certs and place them into `vault/http-certs`

    ANSIBLE_GATHERING=explicit ANSIBLE_HOST_KEY_CHECKING=False ANSIBLE_FORKS=10 ansible-playbook -i inventories/test-inventory2.yml test-setup/add-http-to-vault/elasticsearch-certs-html-test-v2.yml --user=ec2-user --become --ask-vault-pass

for a non-AWS machines 

ANSIBLE_GATHERING=explicit ANSIBLE_HOST_KEY_CHECKING=False ANSIBLE_FORKS=10 ansible-playbook -i inventories/test/test-inventory.yml test-setup/add-http-to-vault/elasticsearch-certs-html-test-v2.yml --ask-vault-pass

note that `gather_facts` is disabled using the 

# encrypt the certs

tried to find an automated way to do this but its too difficult to mess around with. 

cd into the directory `vault/http-certs` and run this one line:

    ansible-vault encrypt *

it will prompt for a vault password, so add one and remember it. 

for testing just use something silly like "1234"

all files in this dir are now encrypted

