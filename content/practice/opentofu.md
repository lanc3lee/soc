
![[tofu-install.png]]



-----



```

lance@LANC3 tofu % tofu destroy

**data.aws_ami.ubuntu: Reading...**

**aws_security_group.siem_practice: Refreshing state... [id=sg-020a191d083bfd6ae]**

**data.aws_ami.ubuntu: Read complete after 0s [id=ami-0d0d141bde51e80ef]**

**aws_instance.siem_practice: Refreshing state... [id=i-06047c22187ee63a5]**

  

OpenTofu used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:

  - destroy

  

OpenTofu will perform the following actions:

  

  **# aws_instance.siem_practice** will be **destroyed**
  ...
  
```


```
**Do you really want to destroy all resources?**

  OpenTofu will destroy all your managed infrastructure, as shown above.

  There is no undo. Only 'yes' will be accepted to confirm.

  

  **Enter a value:** yes

  

**aws_instance.siem_practice: Destroying... [id=i-06047c22187ee63a5]**

**aws_instance.siem_practice: Still destroying... [id=i-06047c22187ee63a5, 10s elapsed]**

**aws_instance.siem_practice: Still destroying... [id=i-06047c22187ee63a5, 20s elapsed]**

**aws_instance.siem_practice: Still destroying... [id=i-06047c22187ee63a5, 30s elapsed]**

**aws_instance.siem_practice: Still destroying... [id=i-06047c22187ee63a5, 40s elapsed]**

**aws_instance.siem_practice: Destruction complete after 40s**

**aws_security_group.siem_practice: Destroying... [id=sg-020a191d083bfd6ae]**

**aws_security_group.siem_practice: Destruction complete after 1s**

  

**Destroy complete! Resources: 2 destroyed.**

lance@LANC3 tofu % 

lance@LANC3 tofu % tofu show

The state file is empty. No resources are represented.

lance@LANC3 tofu % 

lance@LANC3 tofu % aws ec2 describe-instances --instance-ids i-06047c22187ee63a5 --region ap-southeast-1 --query 'Reservations[].Instances[].State.Name'

[

    "terminated"

]

lance@LANC3 tofu %
```