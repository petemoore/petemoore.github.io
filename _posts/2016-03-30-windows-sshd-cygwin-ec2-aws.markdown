---
layout: post
title:  "Walkthrough installing Cygwin SSH Daemon on AWS EC2 instances"
date:   2016-03-30 11:33:34
categories: general taskcluster
---

One of the challenges we face at Mozilla is supporting Windows in an
organisational environment which is predominantly \*nix oriented. Furthermore,
historically our build and test infrastructure has only provided a very limited
ssh daemon, with an antiquated shell, and outdated unix tools.

With the move to hosting Windows environments in AWS EC2, the opportunity arose
to review our current SSH daemon, and see if we couldn't do something a little
bit better.

When creating Windows environments in EC2, it is possible to launch a "vanilla"
Windows instance, from an AMI created by Amazon. This instance is based on a
standard installation of a given version of Windows, with a couple of AWS EC2
tools preinstalled.

One of the features of the preinstalled tools, is that they allow you to
specify powershell and/or batch script snippets inside the instance User Data,
that will be executed upon launch.

This makes it quite trivial to customise a Windows environment, by providing
all of the customisation steps as a PowerShell snippet in the instance User
Data.

In this Walkthrough, we will set up a Windows 2012 R2 Windows machine, with the
cygwin ssh daemon preinstalled. In order to follow this walkthrough, you will
need an AWS account, and the ability to spawn an instance.

# Install AWS CLI

Although all of these steps can be performed via the web console, typically we
would want to automate them. Therefore in this walkthrough, I'm using the AWS
CLI to perform all of the actions, to make it easier should you want to script
any of the setup.

### Windows installation

Download and run the [64-bit](https://s3.amazonaws.com/aws-cli/AWSCLI64.msi) or
[32-bit](https://s3.amazonaws.com/aws-cli/AWSCLI32.msi) Windows installer.

### Mac and Linux installation

Requires Python 2.6.5 or higher.

Install using pip.

{% highlight bash %}
$ pip install awscli
{% endhighlight %}

### Further help

See the AWS [CLI guide](https://aws.amazon.com/cli/) if you get stuck.

### Configuring AWS credentials

If this is your first time running the AWS CLI tool, configure your credentials
with:

{% highlight bash %}
$ aws configure
{% endhighlight %}

See the AWS [credentials configuration
guide](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)
if you need more help.

# Locate latest Windows Server 2012 R2 AMI (64bit)

The following command line will find you the latest Windows 2012 R2 stock
image, provided by AWS, in your default region.

{% highlight bash %}
$ AMI="$(aws ec2 describe-images --owners self amazon --filters \
"Name=platform,Values=windows" \
"Name=name,Values=Windows_Server-2012-R2_RTM-English-64Bit-Base*" \
--query 'Images[*].{A:CreationDate,B:ImageId}' --output text \
| sort -u | tail -1 | cut -f2)"
{% endhighlight %}

Now we can see what the current AMI is, in our default region, with:

{% highlight bash %}
$ echo "Windows AMI: ${AMI}"
Windows AMI: ami-1719f677
{% endhighlight %}

Note, the actual AMI provided by AWS changes from week to week, and from region
to region, so don't be surprised if you get a different result to the one
above.

# Create a Security Group

We need our instance to be in a security group that allows us to SSH onto it.

First create a security group:

{% highlight bash %}
$ SECURITY_GROUP="$(aws ec2 create-security-group --group-name ssh-only \
--description "SSH only" --output text)"
{% endhighlight %}

And then update it to only allow inbound SSH traffic:

{% highlight bash %}
$ [ -n "${SECURITY_GROUP}" ] && aws ec2 authorize-security-group-ingress \
--group-id "${SECURITY_GROUP}" \
--ip-permissions '[{"IpProtocol": "tcp", "FromPort": 22, "ToPort": 22,
"IpRanges": [{"CidrIp": "0.0.0.0/0"}]}]'
{% endhighlight %}

# Create a unique Client Token

We should create a unique client token that will allow us to make idempotent
requests, should there be any failures. We will also use this as our "name"
for the instance until we get the real instance name back.

{% highlight bash %}
$ TOKEN="$(date +%s)"
{% endhighlight %}

# Create a dedicated Key Pair

We'll need to specify a key pair in order to retrieve the Windows Password.
Let's create a dedicated one just for this instance.

{% highlight bash %}
$ aws ec2 create-key-pair --key-name "${TOKEN}" --query 'KeyMaterial' \
--output text > "${TOKEN}.pem" && chmod 400 "${TOKEN}.pem"
{% endhighlight %}

# Create custom post-installation script

Typically, you'll want to customise the cygwin environment, for example:

* Changing the bash prompt
* Setting vim options
* Adding ssh authorized keys
* ....

Let's do this in a post installation bash script, which we can download as
part of the installation.

In order to be able to authenticate with our new key, we'll need to get
the public part. Note, we could generate separate keys for ssh'ing to
our machine, but we might as well reuse the key we just created.

{% highlight bash %}
$ PUB_KEY="$(ssh-keygen -y -f "${TOKEN}.pem")"
{% endhighlight %}

# Create User Data

The [AWS Windows
Guide](http://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ec2-instance-metadata.html#user-data-execution)
advises us that Windows PowerShell commands can be executed if supplied as part
of the EC2 User Data. We'll use this userdata to install cygwin and the ssh
daemon from scratch.

Create a file `userdata` to store the User Data:

{% highlight bash %}
$ cat > userdata << 'EOF'
<powershell>

# needed for making http requests
$client = New-Object system.net.WebClient

# download cygwin
$client.DownloadFile("https://www.cygwin.com/setup-x86_64.exe", `
"C:\cygwin-setup-x86_64.exe")

# install cygwin
# complete package list: https://cygwin.com/packages/package_list.html
Start-Process "C:\cygwin-setup-x86_64.exe" -ArgumentList ("--quiet-mode " +
"--wait --root C:\cygwin --site http://cygwin.mirror.constant.com " +
"--packages openssh,vim,curl,tar,wget,zip,unzip,diffutils,bzr") -wait `
-NoNewWindow -PassThru -RedirectStandardOutput "C:\cygwin_install.log" `
-RedirectStandardError "C:\cygwin_install.err"

# open up firewall for ssh daemon
New-NetFirewallRule -DisplayName "Allow SSH inbound" -Direction Inbound `
-LocalPort 22 -Protocol TCP -Action Allow

# workaround for https://www.cygwin.com/ml/cygwin/2015-10/msg00036.html
# see:
#   1) https://www.cygwin.com/ml/cygwin/2015-10/msg00038.html
#   2) https://goo.gl/EWzeVV
$env:LOGONSERVER = "\\" + $env:COMPUTERNAME

# configure sshd
Start-Process "C:\cygwin\bin\bash.exe" -ArgumentList "--login
-c `"ssh-host-config -y -c 'ntsec mintty' -u 'cygwinsshd' \
-w 'qwe123QWE!@#'`"" -wait -NoNewWindow -PassThru -RedirectStandardOutput `
"C:\cygrunsrv.log" -RedirectStandardError "C:\cygrunsrv.err"

# start sshd
Start-Process "net" -ArgumentList "start sshd" -wait -NoNewWindow -PassThru `
-RedirectStandardOutput "C:\net_start_sshd.log" `
-RedirectStandardError "C:\net_start_sshd.err"

# download bash setup script
$client.DownloadFile(
"https://raw.githubusercontent.com/petemoore/myscrapbook/master/setup.sh",
"C:\cygwin\home\Administrator\setup.sh")

# run bash setup script
Start-Process "C:\cygwin\bin\bash.exe" -ArgumentList `
"--login -c 'chmod a+x setup.sh; ./setup.sh'" -wait -NoNewWindow -PassThru `
-RedirectStandardOutput "C:\Administrator_cygwin_setup.log" `
-RedirectStandardError "C:\Administrator_cygwin_setup.err"

# add SSH key
Add-Content "C:\cygwin\home\Administrator\.ssh\authorized_keys" "%{SSH-PUB-KEY}%"
</powershell>
EOF
{% endhighlight %}

# Fix SSH key

We need to replace the SSH public key placeholder we just referenced in userdata
with the actual public key

{% highlight bash %}
$ USERDATA="$(cat userdata | sed "s_%{SSH-PUB-KEY}%_${PUB_KEY}_g")"
{% endhighlight %}

# Launch new instance

We're now finally ready to launch the instance. We can do this with the
following commands:

{% highlight bash %}
$ {
echo "Please be patient, this can take a long time."
INSTANCE_ID="$(aws ec2 run-instances --image-id "${AMI}" --key-name "${TOKEN}" \
--security-groups 'ssh-only' --user-data "${USERDATA}" \
--instance-type c4.2xlarge --block-device-mappings \
DeviceName=/dev/sda1,Ebs='{VolumeSize=75,DeleteOnTermination=true,VolumeType=gp2}' \
--instance-initiated-shutdown-behavior terminate --client-token "${TOKEN}" \
--output text --query 'Instances[*].InstanceId')"
PUBLIC_IP="$(aws ec2 describe-instances --instance-id "${INSTANCE_ID}" --query \
'Reservations[*].Instances[*].NetworkInterfaces[*].Association.PublicIp' \
--output text)"
unset PASSWORD
until [ -n "$PASSWORD" ]; do
    PASSWORD="$(aws ec2 get-password-data --instance-id "${INSTANCE_ID}" \
    --priv-launch-key "${TOKEN}.pem" --output text \
    --query PasswordData)"
    sleep 10
    echo -n "."
done
echo
echo "SSH onto your new instance (${INSTANCE_ID}) with:"
echo "    ssh -i '${TOKEN}.pem' Administrator@${PUBLIC_IP}"
echo
echo "Note, the Administrator password is \"${PASSWORD}\", but it"
echo "should not be needed when connecting with the ssh key."
echo
}
{% endhighlight %}

You should get some output similar to this:

{% highlight bash %}
Please be patient, this can take a long time.
................
SSH onto your new instance (i-0fe79e45ffb2c34db) with:
    ssh -i '1459795270.pem' Administrator@54.200.218.155

Note, the Administrator password is "PItDM)Ph*U", but it
should not be needed when connecting with the ssh key.
{% endhighlight %}
