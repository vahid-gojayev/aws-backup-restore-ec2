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
