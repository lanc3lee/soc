
--------

upgrade AWS account to deploy t3.medium EC2 instance for Wazuh installation
(it will cost you up to 3-5 sgd a day if you keep it running, terminate resources after practice)

![[AWS-upgrade-account.png]]

![[AWS-upgrade-account-to-access-bigger-EC2.png]]

----------

Practice installing Wazuh via tofu (terraform) followed by install assistant

![[tofu-apply-wazuh-01.png]]

![[tofu-apply-wazuh-02.png]]

note 3.81.49.222 is the EC2 instance IP (and also the Wazuh dashboard IP later)

You will get a different IP as you practise

![[AWS-EC2-wazuh-initializing.png]]


EC2 initializing takes about 3-5 minutes. 

You can find EC2 IP address in console too

![[AWS-EC2-wazuh-running.png]]


-----

![[wazuh-installation-using-assistant.png]]

--------

![[wazuh-newly-installed-login-page.png]]


![[wazuh-newly-installed.png]]



-------


Terminating EC2 instance after practice to avoid incurring costs

![[AWS-EC2-terminate-01.png]]

![[AWS-EC2-terminate-02.png]]