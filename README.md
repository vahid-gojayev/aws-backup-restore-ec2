# aws-backup-restore-ec2
AWS ec2 backup and recovery automation solution

![architecture](https://user-images.githubusercontent.com/123200995/213848962-ea7d597f-945a-453e-95ac-d502591f3cd0.PNG)


I shared how I wrote a lambda boto3 python script for a solution that automates the backup and restore of an ec2 instance.

I used 2 classes for this solution: the first class for preparing the image, the second for deploying the image.
>

class PrepareAmi:



        def __init__(self, stack_instance_name, ssm_parameter, stack_name, region, aminame, timeout):
                self.stack_instance_name = stack_instance_name
                self.ssm_parameter = ssm_parameter
                self.stack_name = stack_name
                self.region = region
                self.aminame = aminame
                self.timeout = timeout



        def get_parameter_version(self):
                ssm = boto3.client('ssm')
                parameter = ssm.get_parameter(Name=f"{self.ssm_parameter}", WithDecryption=True)
                parameter_response = parameter['Parameter']['Version']
                return parameter_response 



        def get_old_image_parameters(self,y):
                ec2 = boto3.resource('ec2')
                for instance in list(ec2.instances.filter(Filters=[{'Name': 'tag:aws:cloudformation:logical-id', 'Values': [f"*{self.stack_instance_name}*"]},{'Name': 'instance-state-name', 'Values': ['running']}])):
                            def wrapper(*args):
                                if y == 1:
                                   return instance.id
                                if y == 2:
                                   return instance.image.id
                                if y == 3:
                                   return instance.image.name
                            return wrapper



        def create_new_ami(self, instance_id, new_ami_name):
                client = boto3.client('ec2', region_name=f"{self.region}")
                ami_response = client.create_image(InstanceId=f"{instance_id}",Name=f"{new_ami_name}", NoReboot=True)
                return ami_response
                


        def check_state_new_ami(self, new_image_id):
                count = 0
                while True:
                    client = boto3.client('ec2', region_name=f"{self.region}")
                    ami_state_response = client.describe_images(ImageIds=new_image_id)
                    time.sleep(self.timeout)
                    print(ami_state_response["Images"][0]["State"])
                    if ami_state_response["Images"][0]["State"] == "available":
                        return "available"
                    if ami_state_response["Images"][0]["State"] != "available":
                        count += 1
                        if count == 15:
                            logger.error("Connection Timeout Error You can Increase timout")
                            break

>
class CreateAmi(PrepareAmi):


        def __init__(self, stack_instance_name, ssm_parameter, stack_name, region, aminame, timeout):
              super().__init__(stack_instance_name, ssm_parameter, stack_name, region, aminame, timeout)


        def create_ami_vars(self):

                ssm_client = boto3.client('ssm', region_name=f"{region}")
                ec2_client = boto3.client('ec2', region_name=f"{region}")
                clf_client = boto3.client('cloudformation', region_name=f"{region}")

                

                try:
                        version_new = self.get_parameter_version() + 1
                except json.decoder.JSONDecodeError as err:
                        logger.error("Parameter Not Found Please Fix it")
                        exit(1)


                version_old = self.get_parameter_version()
                new_ami_name = aminame + f"-{version_new}"
                old_ami_name = aminame + f"-{version_old}"


                @self.get_old_image_parameters(1)
                def old_instance_id():
                        return "Return Decorator Instace"
                print(old_instance_id)


                @self.get_old_image_parameters(2)
                def old_image_id():
                        return "Return Decorator Image Variable"
                print(old_image_id)


                @self.get_old_image_parameters(3)
                def old_image_name():
                        return "Return Decorator Old Image ID"
                print(old_image_name)


                new_ami_image_id = self.create_new_ami(old_instance_id,new_ami_name)["ImageId"]
                convert = []
                result = ''.join(map(lambda x: x, new_ami_image_id[0:]))
                convert.append(result)
                check_new_ami_avaibility = self.check_state_new_ami(convert)


                if "available" in  check_new_ami_avaibility:

                        
                        parameter = ssm_client.put_parameter(Name=f"{self.ssm_parameter}", Overwrite=True, Value=new_ami_image_id)
                        logger.info("Value of {} is updated to {} version".format(ssm_parameter,parameter["Version"]))
                        


                        try:
                                check_resp = ec2_client.describe_images(ImageIds=[old_image_id])
                                owner_id = check_resp["Images"][0]["OwnerId"]
                                print(owner_id)
                        except ClientError:
                                print("Invalid Image ID Please use correct image id")


                        ami_response = ec2_client.deregister_image(ImageId=old_image_id, DryRun=False)
                        print(ami_response)
                        snapshot_response = ec2_client.describe_snapshots(OwnerIds=[owner_id])


                        for i in range(len(snapshot_response["Snapshots"])):
                        
                                if old_image_id in snapshot_response["Snapshots"][i]["Description"]:
                                        snapid = snapshot_response["Snapshots"][i]["SnapshotId"]
                                        response = ec2_client.delete_snapshot(SnapshotId=snapid)
                                        if response["ResponseMetadata"]["HTTPStatusCode"] == 200:
                                                print(logger.info(f"Backup Successful Created: Deleted Old Image ID and Name - {old_image_id}--{old_ami_name}  Created New Image ID and Name - {new_ami_image_id}--{new_ami_name} "))
                                

                
                response = clf_client.delete_stack(StackName=stack_name)
                print(logger.info(f"Stack {stack_name} Successful Deleted"))
