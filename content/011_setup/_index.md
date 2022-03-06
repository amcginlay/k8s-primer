---
title: "1. Setup"
chapter: false
weight: 11
draft: false
---

## Purpose

Prior to working with *containers* and *Kubernetes* it is helpful to have a dedicated workspace you can use to work, learn, and play with these things.
This chapter guides you through the process of provisioning an isolated environment in the cloud that you can use independently of your local computer(s).
The rest of this workshop assumes you have created a **Cloud9** environment in which you can perform the tasks and learn at your own pace.
You may choose to delete one environment whenever you are done learning or set up a new one when you wish to start over.

## Prerequisites

- These instructions build a [Cloud9](https://aws.amazon.com/cloud9/) environment and assume you have access to a sufficiently privileged [AWS](https://aws.amazon.com/) account in order to do so.
- To ensure reproducibility we will create all our infrastructure in the `us-west-2` region. 
- **CloudShell** is used here as a temporary bootstrap to provision your Cloud9 environment.

## Configure CloudShell

{{< step >}}Once signed into your AWS account, begin by navigating to https://us-west-2.console.aws.amazon.com/cloudshell.{{< /step >}} 

The CloudShell environment includes the AWS CLI but it regularly tracks behind the latest version.

{{< step >}}To ensure you are running the latest version, from **within your CloudShell environment**, update the CLI as follows.{{< /step >}}
```bash
rm -rf /usr/local/bin/aws 2> /dev/null
rm -rf /usr/local/aws-cli 2> /dev/null
rm -rf aws awscliv2.zip 2> /dev/null
curl --silent "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install --update
```

## Build a Cloud9 environment

{{< step >}}From **within your CloudShell environment**, run the following snippet to create a Cloud9 instance.{{< /step >}}
```bash
subnet_id=$( \
  aws ec2 describe-subnets \
    --filters "Name=availability-zone,Values=${AWS_DEFAULT_REGION}a" "Name=default-for-az,Values=true" \
    --query "Subnets[].SubnetId" \
    --output text \
)
env_id=$(
  aws cloud9 create-environment-ec2 \
    --name k8s-primer-$(date +"%Y%m%d%H%M") \
    --instance-type m5.large \
    --image-id amazonlinux-2-x86_64 \
    --subnet-id ${subnet_id} \
    --automatic-stop-time-minutes 1440 \
    --query "environmentId" \
    --output text \
)
```

{{% notice note %}}
NOTE: The assignment of `subnet_id` here assumes that you have a *Default VPC* in the default region with a subnet in availability zone `a`.
If this is not the case, either: a) use an alternative region which still has a *Default VPC* or b) use a different account with a *Default VPC* in `us-west-2` or c) modify the subnet query accordingly to choose an adequate subnet for the Cloud9 instance.
{{% /notice %}}

## Configure your Cloud9 environment

{{< step >}}From **within your CloudShell environment**, execute the following command then navigate your browser to the Cloud9 URL it prints out.{{< /step >}}
```bash
echo "https://${AWS_DEFAULT_REGION}.console.aws.amazon.com/cloud9/ide/${env_id}"
```

If the Cloud9 URL is unresponsive, wait 30 seconds and try again.

{{% notice note %}}
Henceforth, your terminal interactions will exclusively take place via your **Cloud9** environment.
You may initiate new command line sessions inside **Cloud9** at any time using `Window -> New Terminal`
{{% /notice %}}

## Extend your root volume

{{< step >}}From **within your Cloud9 environment**, to ensure we don't exhaust disk space, extend the root volume storage to 30gb.{{< /step >}}
```bash
df -T # check disk use percentage before (typically ~80%) ...

region=$(curl --silent http://169.254.169.254/latest/meta-data/placement/region)
instance_id=$(curl --silent http://169.254.169.254/latest/meta-data/instance-id)

volume_id=$(aws ec2 describe-instances \
  --region ${region} \
  --instance-id ${instance_id} \
  --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" \
  --output text
)

aws ec2 modify-volume \
  --region ${region} \
  --volume-id ${volume_id} \
  --size 30

while [ \
  $(aws ec2 describe-volumes-modifications \
    --region ${region} \
    --volume-id ${volume_id} \
    --filters Name=modification-state,Values="optimizing","completed" \
    --query "length(VolumesModifications)" \
    --output text) != 1 ]; do
  sleep 1
done

sudo growpart /dev/nvme0n1 1
sudo xfs_growfs -d /

df -T # ... check disk use percentage has been reduced
```

## Dispose of your pre-loaded Docker images

Each Cloud9 instance has the [Docker](https://www.docker.com/) build/runtime tooling.
Docker provides a local cache feature so it can make efficient re-use of downloaded container images.
Cloud9 pre-loads the cache with images we will not need in this tutorial.
{{< step >}}Flush out the container image cache as follows.{{< /step >}}
```bash
for i in $(docker ps -q); do docker kill $i; done
docker system prune --all --force
```

## Reconfigure your terminal prompt

The Cloud9 terminal prompt is configured to do some pretty smart things but it can also misbehave, so dumb it down now.

{{< step >}}Append a shorter prompt to the shell configuration and apply it.{{< /step >}}
```
echo "export PS1='$ '" >> ~/.bashrc
source ~/.bashrc
```

## Setup Quiz

Please take the following quiz to review your knowledge of your environment setup.

{{< quizdown >}}

---
primary_color: orange
secondary_color: lightgray
text_color: black
shuffle_questions: false
---

## Workshop Environment

---
shuffle_answers: false
---

What environment will you be using to work with containers and Kubernetes?

> Scroll up or switch to your other tab.

- [ ] AWS CloudShell
- [x] AWS Cloud9
- [ ] AWS CodePipeline
- [ ] Amazon SageMaker Studio
- [ ] Microsoft VSCode

## Setup time

How long did it take to set up this environment?

> What time was it when you started the setup?

- [ ] Instantaneous. No time at all.
- [x] Less than 5 minutes.
- [ ] 10 minutes.
- [ ] 15 minutes.

## Redoing this

If you delete this practice environment, can you set it up again any time?

> What if you want to repeat this next week? Next month?

- [x] Yes, I can do this again any time.
- [ ] No, I can only do this today.

{{< /quizdown >}}

## Success

You have now completed the prerequisite of creating a **Cloud9** environment in which you can explore *containers* and *Kubernetes*.
Now you are ready to go. Please continue to the next section.
