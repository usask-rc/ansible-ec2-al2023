# ansible-ec2-al2023
Ansible deployment to EC2.

This is a guide to running Ansible in Ubuntu under WSL, deploying to AWS EC2 instances. It would be nice if ansible could run on Windows as a control host, however it cannot, so Ubuntu under Windows is the next best thing for users that are unable to boot into Ubuntu natively.  Unfortunately, it comes with some caveats as explained below.

This guide is written for University of Saskatchewan researchers.  It may not work in your environment.

You should already have at least one EC2 instance up and running, which may have been installed using terraform if you already followed the instructions here:
https://github.com/usask-rc/ec2-sample-single-al2023

1. Install WSL

Open Software Center on your Usask managed Windows computer. Search for wsl. One of the options in the list should be "Microsoft Virtual Machine Platform". Click to open this item and then click Install.  A reboot of your computer will be required.

2. Verify that WSL is installed

Run the following in a Windows terminal or Power Shell prompt:

```wsl --list --online```

You should see a list of distributions that could be installed.  If you see anything else, you will need to work with IT Support to fix your WSL installation before proceeding.

3. Install Ubuntu

```wsl --install Ubuntu```

Set a password for your user that you will remember, because you will need to type it a few times in the future.

4. Allow Ubuntu to set file permissions

As currently installed, unix file permissions cannot be changed.  To fix this, run the following at a Windows command prompt:

```wsl -u root```

Now that you are in Ubuntu at a command prompt, edit the following file with `vi` or `nano`:

```vi /etc/wsl.conf```

Add a new section to this file and then save the file:
```
[automount]
options = "metadata"
```
5. Change your home directory [OPTIONAL]

It can be convenient to use the same home directory on Windows and Unbuntu, so that ssh keys are easily found in both (ie: at a path like `~/.ssh/key`). If you want to do this, then while you are still within Ubuntu as root change the home directory for your user (use your NSID here in place of abc123):

```usermod -d /mnt/c/Users/abc123 abc123```

If you get an error that the user process is in use it means you already started Ubuntu as yourself. You can restart and try again, but this time do not start Ubuntu as yourself, just go in as root first. 

You can also delete your default home directory in Ubuntu so you don't accidentally put anything there in the future **assuming that you have not already saved files there**:

```cd /home; rm -rf abc123```

If you do have your own data in your default ubuntu home directory, either skip this step or copy the data elsewhere first.

6. Run Ubuntu as yourself

If you are still running as root in Ubuntu, exit out:

```exit```

At a Windows command prompt:

```wsl```

This will start Ubuntu again, and you will be logged in as your own non-root user.

7. Update Ubuntu and install Ansible

At the Ubuntu command prompt:
```
sudo apt update
sudo apt upgrade -y
sudo apt install unzip ansible -y
```

You should be able to run `ansible --version` now and see it installed correctly.

8. Install the AWS CLI in Ubuntu

See this guide, ensuring that you follow the instructions for "Command line installer - Linux x86 (64 bit)":
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

When you are finished you should be able to run `aws --version` and see it installed correctly.

9. Install the AWS Python libraries in Ubuntu

Run the following command in Ubuntu:

```sudo apt install python3-botocore python3-boto3 -y```

10. Configure an AWS SSO session [OPTIONAL]

If you have already configured AWS SSO within Windows then you will already have a file at `~/.aws/config` otherwise you can run the following:

```aws configure sso --profile <profilename>```

Answer the questions.  The Start URL is the same one that you use to log into your Usask AWS account, but remember to omit all characters after the question mark.

Then log in to your profile:

```aws sso login --profile <profilename>```

You will need to control-click the URL that shows up and then use your normal browser to log in and authorize the CLI to your AWS account.

11. Set the AWS profile for Ansible

Edit the file `aws_ec2.yml` and set the variable for `profile` to be the same as `<profilename>` that you used when setting up AWS SSO.

12. Fix directory permissions

Still in Ubuntu at a command prompt, change directory to where you cloned this repo, then set permissions:
```
cd ansible-ec2-al2023
chmod 0755 .
```
Now when you list the directory contents, you should see the correct permissions:

```ls -al```

If this step is not done then the file `ansible.cfg` will be ignored by Ansible since it believes the project directory is world writable and it does not trust the configuration file.

13. Fix ssh private key permissions

Assuming that your ssh private key is in `~/.ssh/ec2sample` then make sure ssh in Ubuntu thinks the key is not world writable:

```chmod 0600 ~/.ssh/ec2sample```

If you are using a different private key then you need to update the key file path in `update.yml`.

14. Run ansible and show the EC2 hosts

Run the following command and see that it shows the EC2 hosts:

```ansible-inventory -i aws_ec2.yml --graph```

15. Check ssh into the target host

Get the public host name from the inventory list above.  It will be something like `ec2-11-22-33-44.ca-central-1.amazonaws.com`. Using this host name, run a command like the following:

```ssh -i ~/.ssh/ec2sample ec2-user@ec2-11-22-33-44.ca-central-1.amazonaws.com```

16. Run the playbook

```ansible-playbook -i aws_ec2.yml update.yml```

If all goes well the Ansible output should not contain any red text.

#### ansible.cfg

This repo contains a file `ansible.cfg` that sets some ssh properties for ansible. This is not normally needed, however Ubuntu under WSL behaves in a similar fashion to running Ansible [within a Docker container](https://github.com/semaphoreui/semaphore/issues/309#issuecomment-432515181).

#### ssh-agent

If you are trying to get ssh agent working in Ubuntu under WSL, the syntax is slightly different than normal:
```
eval `ssh-agent -s`
ssh-add ~/.ssh/ec2sample
```

