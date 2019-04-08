# Install Jenkins with ansible playbook on container systemd-nspawn 

** You need ansible varsion >= 2.6 **

`ansible-playbook -i inventoryfile playbook/jenkins/deploy-jenkins.yaml`

After playbook finish you can go on http://<systemd-container-ip>:8080
