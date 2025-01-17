# Root Volume Encryption
Despite all the planning and preparing we do to architect flawless systems, we may eventually run into issues with our design. Changing business needs will mean you need to quickly reassess your design and find reliable solutions. For example, say you spin up several EC2 instances with unencrypted root volumes, thinking you would not need to store any sensitive data. Requirements change and you now need to encrypt those volumes. This post will walk through the steps to encrypt a root volume for a given EC2 instance.

## Tech Overview
Amazon provides APIs for many different languages. This example will be written in Python. Python's SDK is called Boto 3. There is a great GitHub quick start that can get you going with Boto3 for further research beyond these instructions. https://github.com/boto/boto3

## AWS CLI Configuration
1. Install the AWS CLI. http://docs.aws.amazon.com/cli/latest/userguide/installing.html
2. Configure your client. http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html

These instructions are well documented and will walk through setting up your client with your AWS keys for credentials. When configuring the AWS CLI it will set up your default profile. You can set up multiple 'Named Profiles' with different settings. Simply set the '--profile' flag and it prompt for input. 

```
aws configure --profile profilename
```

You can check what your configuration is by looking at a file located in 'home_directory/.aws' called 'config.' You can also see your credentials in the same location but in a file called 'credentials.'

## IAM Permissions
These actions are required in order to run this script. This is an example policy you can attach to a user to give them the proper IAM authorization.
```json
{
    "Version": "2010-10-10",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:StopInstances",
                "ec2:StartInstances",
                "ec2:CopySnapshot",
                "ec2:CreateSnapshot",
                "ec2:CreateVolume",
                "ec2:DeleteSnapshot",
                "ec2:AttachVolume",
                "ec2:DetachVolume"
            ],
            "Resource": "*"
        }
    ]
}
```

## The Script:

First we will layout the overview of the script including parameters it can receive.

```python
#!/bin/python

# Overview:
#    Take unencrypted root volume and encrypt it for EC2.
# Params:
#    ID for EC2 instance
#    Customer Master Key (CMK) (optional)
#    Profile
# Conditions:
#    Return if volume already encrypted
#    Use named profiles from credentials file
```

This script will receive the ID of the EC2 instance. We dont want to encrypt a volume if it is already encrypted, we will return if that is the case. Also, we want to take advantage of profiles so we will receive the profile the user wants to use. The Customer Master Key will be used for the encryption, but is not required. This will be explained later.

Lets import what we need for the script and set up our argument parser.

```python
import sys
import boto3
import argparse

def main(argv):
    parser = argparse.ArgumentParser(description='Encrypts EC2 root volume.')
    parser.add_argument('-i', '--instance', help='Instance to encrypt volume on.',required=True)
    parser.add_argument('-key','--customer_master_key',help='Customer master key', required=False)
    parser.add_argument('-p','--profile',help='Profile to use', required=False)
    args = parser.parse_args()
```


Setting up our session:
The next step is to get our session set up. If we received a given profile, we can pass that profile in when we create our session. If no arguments are passed it will grab our default profile.

```python
# Set up AWS Client + Resources + Waiters
if args.profile:
  # Create custom session
  print('Using Profile {}'.format(args.profile))
  session = boto3.session.Session(profile_name=args.profile)
else:
  # Use default session
  session = boto3.session.Session()
```


Two key features of Boto3 are high-level object-oriented resources and low-level service connections. Low-level services map closly to the AWS service APIs whereas the high-level are abstracted from them.

Another feature are waiters, these are helpful for blocking until desired states are reached. Two waiters are stored, one for waiting until a snapshot is completed and the other for waiting until a volume is available. We get these waiters from the low-level client.

```python
client = session.client('ec2')
ec2 = session.resource('ec2')
waiter_snapshot_complete = client.get_waiter('snapshot_completed')
waiter_volume_available = client.get_waiter('volume_available')
```

If the optional CMK was passed in, we will store it.
```python
# Get CMK
customer_master_key = args.customer_master_key
```


Check instance state:
Before we proceed to far into the script we will make sure that the instance id we received can be used to retrieve an instance. If its not, we will exit the script.

```python
# Get Instance
instance_id = args.instance
print('---Instance {}'.format(instance_id))
instance = ec2.Instance(instance_id)

if not instance:
    print('No instance found with ID {}'.format(instance_id))
    sys.exit()
```


### Steps:
We have an instance and the resources we need, lets continue the script and encrypt the volume.

#### 1.Shut down if running
We wont be able to work with this instance how we want if its running. You can check the status of instance through its state property. The different codes are:

- 0 : pending
- 16 : running
- 32 : shutting-down
- 48 : terminated
- 64 : stopping
- 80 : stopped

```python
if instance.state['Code'] is 16:
    instance.stop()
```


#### 2.Take snapshot
You can access the volumes of the instances. These return a Boto3 'Collection' which is an iterable of resources. We can iterate through it to get access to the actual instance of the volume. Note, if the volume is encrypted, we will exit our script.

```python
for v in instance.volumes.all():
    volume_id = v.id
    if v.encrypted:
        print('**Volume already is encrypted')
        sys.exit()
```

Now that we have the id for the volume we will  use that to create a snapshot of its current state. Immediately after we will use the waiter we stored earlier to wait for the snapshot to be complete. You can pass multiple ids into the waiter. In this case, we will just wait for the one we created to be complete.

```python
print('---Create snapshot of volume {}'.format(volume_id))
snapshot = ec2.create_snapshot(
    VolumeId=volume_id,
    Description='Snapshot of {}'.format(volume_id),
)

waiter_snapshot_complete.wait(
    SnapshotIds=[
        snapshot.id,
    ]
)
```

#### 3.Create new encrypted volume
To create the encrypted volumes we can simply create a copy of the snapshot of the unencrypted volume and set the 'Encrypted' flag to true. In addition to the encrypted flag, we can set other parameters for this action. If the user passed their customer master key, meaning they dont want to use the Amazons default key managment system, we will use that specific key for the encryption. Again, once the snapshot begins to be copied, we will wait for the snapshot to be complete before proceeding.

```python
print('---Create Encrypted Snapshot Copy')
if customer_master_key:
    # Use custom key
    snapshot_encrypted = snapshot.copy(
        SourceRegion=session.region_name,
        Description='Encrypted Copied Snapshot of {}'.format(snapshot.id),
        KmsKeyId=customer_master_key,
        Encrypted=True,
    )
else:
    # Use default key
    snapshot_encrypted = snapshot.copy(
        SourceRegion=session.region_name,
        Description='Encrypted Copied Snapshot of {}'.format(snapshot.id),
        Encrypted=True,
    )

waiter_snapshot_complete.wait(
    SnapshotIds=[
        snapshot_encrypted['SnapshotId'],
    ],
)
```

The snapshot is complete so we can now take that and create an encrypted volume. Because the snapshot is encrypted, the volume will be too.

```python
print('---Create Encrypted Volume from snapshot')
volume_encrypted = ec2.create_volume(
    SnapshotId=snapshot_encrypted['SnapshotId'],
    AvailabilityZone=instance.placement['AvailabilityZone']
)
```

#### 4.Detach current root volume
Before we can attach the new encrypted volume we need to detach the old volume.

```python
print('---Deatch Volume {}'.format(volume_id))
instance.detach_volume(
    VolumeId=volume_id,
    Device=instance.root_device_name,
)
```

#### 5.Attach new volume
Once the encrypted volume is complete, we are ready to attach it. Pass in the volume id and the keep the device type constant by pulling it from the original instances property.

```python
print('---Attach Volume {}'.format(volume_encrypted.id))
waiter_volume_available.wait(
    VolumeIds=[
        volume_encrypted.id,
    ],
)

instance.attach_volume(
    VolumeId=volume_encrypted.id,
    Device=instance.root_device_name
)
```

#### 6.Restart instance
The volume is attached so we want to bring our instance back up.

```python
print('---Restart Instance')
if instance.state['Code'] is 80 or instance.state['Code'] is 64:
    instance.start()
```

#### Clean up
We no longer need the snapshot because we have extracted our volume already. You can do clean up as needed.
```python
snapshot.delete()
```
