Ansible playbook to install and configure amq as service
Download to your directory
1. change the ipaddress or hostname in the hosts file to the server you are targeting for installation
2. run the command ansible-playbook -vvvv --user root --ask-pass playbook.yml
3. it will ask for the root password on the server 
4. if it is the first time then the script will propmt to add the private key
5. A vagrant file is provided in case you want to create the virtual box ( I have used rhel , change to centos if you do not have subscription  or do not wish to use rhel ).
