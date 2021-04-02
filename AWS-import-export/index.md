# Migrating a VM to AWS using VM Import/Export

This is a small lab to learn AWS VM Import/Export service for a friend. It is possible needed in a lot of cloud migration scenarios but not the only one. AWS documentation has provided a good guide about this migration but I wanted to actually do it a few times and see the outputs, specially the import process and know more how a usual Linux (and perhaps a Windows server later) VM with dynamic IP address with no “cloud init” would behaviour. 

So for the process mind the requirements in terms of OS and versions, kernels, disk type interfaces, etc… AWS is pretty broad supporting a lot of Linux and Windows combinations but sometimes the latest versions aren’t. Also, FreeBSD is complete absent here but I’d bet most admins would find fun in launching it from the marketplace and move the load manually!

## AWS instructions site

Main documentation page is here \> [https://docs.aws.amazon.com/vm-import/latest/userguide/what-is-vmimport.html?shortFooter=true](https://docs.aws.amazon.com/vm-import/latest/userguide/what-is-vmimport.html?shortFooter=true)

It is important to know supported OS and kernel versions before doing it as I stumbled with an “incompatible kernel” version much later on first tries when adding an Ubuntu 17.x as the most updated one supported is v16. I retried using a CentOS 7.4 .

And the instructions and method are here \> [https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html?shortFooter=true](https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html?shortFooter=true)

I won’t detail here but we need to use AWS CLI to create a role with associated policy to be able to run the import command and a ready S3 bucket for that. Any bucket should do and for lab purposes you can make it less expensive using one with less redundancy (new in 2018) and with RRS settings.

I used VMWare Fusion with a regular CentOS 7.4 installed with just server package to make the OVA smaller before upload. I am suspecting I could make it smaller with minimal set of software installed, but I was unsure if lacking network package groups would make AWS reset failing. It should not as the requirements pages would tell us, right? 

So for the first try, I’ve got a Ubuntu, made an OVF package/folder instead of OVA and of course it failed much later in the process (uploading 1GiB took 6 hours using SCP). I changed to CentOS expecting the installation to be smaller but it was little bit bigger, 50MiB approximately. So using CentOS 7.4 and exporting as OVA has worked fine. I will retry with Ubuntu later but I need to install v16.x instead of v17 that is not supported.

## Moving image to S3 for import

I used SCP to copy the OVA image to a temp Linux AMI instance I had it running in **us-east-1** region. So using a simple _scp**_ copy got the file to the instance, where I could then copy to the S3 bucket, third command below. Note that I have not used an S3 role for this instance but instead I’ve installed a set of IAM access key and access key ID user with read/write rights to the bucket. Both ways should work.

iMac:Downloads rodrigo$ **scp -i EC2-US-EAST-1.pem** /Desktop/CentOS1.ova/ ec2-user@54.152.35.169: 
CentOS1.ova                                                                                                                   100% 1044MB  28.5KB/s 10:26:16   
iMac:Downloads rodrigo$ 
iMac:Downloads rodrigo$ **ssh -i EC2-US-EAST-1.pem ec2-user@54.152.35.169**
Last login: Fri Apr 27 19:22:31 2018 from 177.141.138.65

   __|  __|_  )
   _|  (     /   Amazon Linux AMI
  ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2018.03-release-notes/
1 package(s) needed for security, out of 5 available
Run "sudo yum update" to apply all updates.
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
[ec2-user@ip-172-31-59-187 ](#)$ **aws s3 cp CentOS1.ova s3://besparked-vm-import/**
Completed 1.0 GiB/1.0 GiB (62.8 MiB/s) with 1 file(s) remaining   

upload: ./CentOS1.ova to s3://besparked-vm-import/CentOS1.ova     
[ec2-user@ip-172-31-59-187 ](#)$ 
[ec2-user@ip-172-31-59-187 ](#)$ 

### Editing the _containers.json_ file

The document is clear here and before running the import command, a JSON file is needed to point the bucket and OVA file. It seems it could be done in a single command line but I haven’t found references to it, something to review later. For now make sure using correct bucket name and file name and prefix.

[ec2-user@ip-172-31-59-187 ](#)$** nano containers.json **

[](#)
  
"Description": "Ubuntu OVA",
"Format": "ova",
"UserBucket": 
"S3Bucket": "**besparked-vm-import**",
"S3Key": "**CentOS1.ova**"
}
}]



### Importing the OVA archive as an AMI image

The following command will get the archive as specified in the JSON document and start the import process. Later we can check the import process through a describe option. Output at several states are pasted below.

[ec2-user@ip-172-31-59-187 ](#)$ aws ec2 import-image --description "CentOS" --license-type BYOL --disk-containers file://containers.json

"Status": "active", 
"LicenseType": "BYOL", 
"Description": "CentOS", 
"SnapshotDetails": [](#)

"UserBucket": 
"S3Bucket": "besparked-vm-import", 
"S3Key": "CentOS1.ova"
}, 
"DiskImageSize": 0.0, 
"Format": "OVA"
}
], 
"Progress": "2", 
"StatusMessage": "pending", 
"ImportTaskId": "import-ami-0306b6f28c3708613"
}
[ec2-user@ip-172-31-59-187 ](#)$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-0306b6f28c3708613
'
"ImportImageTasks": [](#)

"Status": "active", 
"LicenseType": "BYOL", 
"Description": "CentOS", 
"SnapshotDetails": [](#), 
"Progress": "2", 
"StatusMessage": "pending", 
"ImportTaskId": "import-ami-0306b6f28c3708613"
}
]
}
[ec2-user@ip-172-31-59-187 ](#)$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-0306b6f28c3708613

"ImportImageTasks": [](#)

"Status": "active", 
"LicenseType": "BYOL", 
"Description": "CentOS", 
"SnapshotDetails": [](#)

"Status": "active", 
"UserBucket": 
"S3Bucket": "besparked-vm-import", 
"S3Key": "CentOS1.ova"
}, 
"DiskImageSize": 652312064.0, 
"Format": "VMDK"
}
], 
"Progress": "28", 
"StatusMessage": "converting", 
"ImportTaskId": "import-ami-0306b6f28c3708613"
}
]
}
[ec2-user@ip-172-31-59-187 ](#)$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-0306b6f28c3708613

"ImportImageTasks": [](#)

"Status": "active", 
"LicenseType": "BYOL", 
"Description": "CentOS", 
"SnapshotDetails": [](#)

"Status": "completed", 
"UserBucket": 
"S3Bucket": "besparked-vm-import", 
"S3Key": "CentOS1.ova"
}, 
"DiskImageSize": 652312064.0, 
"Format": "VMDK"
}
], 
"Progress": "34", 
"StatusMessage": "updating", 
"ImportTaskId": "import-ami-0306b6f28c3708613"
}
]
}
[ec2-user@ip-172-31-59-187 ](#)$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-0306b6f28c3708613

"ImportImageTasks": [](#)

"Status": "active", 
"LicenseType": "BYOL", 
"Description": "CentOS", 
"Platform": "Linux", 
"Architecture": "x86_64", 
"SnapshotDetails": [](#)

"Status": "completed", 
"DeviceName": "/dev/sda1", 
"DiskImageSize": 652312064.0, 
"UserBucket": 
"S3Bucket": "besparked-vm-import", 
"S3Key": "CentOS1.ova"
}, 
"Format": "VMDK"
}
], 
"Progress": "58", 
"StatusMessage": "booting", 
"ImportTaskId": "import-ami-0306b6f28c3708613"
}
]
}
[ec2-user@ip-172-31-59-187 ](#)$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-0306b6f28c3708613

"ImportImageTasks": [](#)

"Status": "active", 
"LicenseType": "BYOL", 
"Description": "CentOS", 
"Platform": "Linux", 
"Architecture": "x86_64", 
"SnapshotDetails": [](#)

"Status": "completed", 
"DeviceName": "/dev/sda1", 
"Format": "VMDK", 
"DiskImageSize": 652312064.0, 
"SnapshotId": "snap-01907d3b13638c26c", 
"UserBucket": 
"S3Bucket": "besparked-vm-import", 
"S3Key": "CentOS1.ova"
}
}
], 
"Progress": "84", 
"StatusMessage": "preparing ami", 
"ImportTaskId": "import-ami-0306b6f28c3708613"
}
]
}
[ec2-user@ip-172-31-59-187 ](#)$ 
[ec2-user@ip-172-31-59-187 ](#)$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-0306b6f28c3708613

"ImportImageTasks": [](#)

"Status": "completed", 
"LicenseType": "BYOL", 
"Description": "CentOS", 
"ImageId": "ami-0c93e58db64cb1d23", 
"Platform": "Linux", 
"Architecture": "x86_64", 
"SnapshotDetails": [](#)

"Status": "completed", 
"DeviceName": "/dev/sda1", 
"Format": "VMDK", 
"DiskImageSize": 652312064.0, 
"SnapshotId": "snap-01907d3b13638c26c", 
"UserBucket": 
"S3Bucket": "besparked-vm-import", 
"S3Key": "CentOS1.ova"
}
}
], 
"ImportTaskId": "import-ami-0306b6f28c3708613"
}
]
}
[ec2-user@ip-172-31-59-187 ](#)$ 

The message above indicates process is done and a new private AMI is under the account list.

### Testing access to the instance

Go to the **EC2** section of the AWS console, under the AMI, check if the new image is there and with the button **Launch** create a small instance base on the imported OVA. Here I would use most of default settings first time but I would like to explore larger instances and with more disks, instance based or EBS ones. 

By the last page the security key can be used, but it won’t work as it does with regular Amazon images, reasoning is because the process won’t “cloud init” the instance if I am not missing anything. Anyway, if you have a regular account that is allowed to SSH you can try using public or private IP address on the instance.

![](Screen%20Shot%202018-04-28%20at%2012.19.33.png)

[ec2-user@ip-172-31-59-187 ](#)$ ssh  rmonteiro@54.158.55.32
rmonteiro@54.158.55.32's password: 
Last login: Sat Apr 28 08:33:01 2018 from 54.152.35.169
[rmonteiro@centos1 ](#)$ 
[rmonteiro@centos1 ](#)$ 


## To where now

Having this completed, there is a lot variations one can do, such as using more VM dishes before packing the OVA, using fixed IP address, having IPv6, etc… and I’d say try with a Windows Server supported in the requirements.

Last, there are other methods to use such as vCenter and Hyper-V assisted migrations, or live, which are more complex but useful with production environments with a large serve fleet. On the same main documentation page you can find main instructions.

