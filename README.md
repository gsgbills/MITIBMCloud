# MITIBMCloud
Information relevant to IBM Cloud use in MIT

## How can I run code on IBM Cloud [Watson Studio](https://www.ibm.com/cloud/watson-studio) (for free) ?

Here are the steps to setup your account to use Machine Learing (ML) training runs on different numbers and types of GPUs.
**These steps use a Command Line Interface (CLI). There is an alternative browser used interface** 

What we will do as a one time Setup:
0. Confirm that you have an IBM Cloud userid
1. Download [CLI tools](https://console.bluemix.net/docs/cli/index.html#overview) to access and manage resources in the IBM Cloud
2. Login thru the CLI to your IBM Cloud account
3. Configure your IBM Cloud account. 
4. Create a Watson ML Instance
5. Define a [Cloud Object Storage](https://www.ibm.com/cloud/object-storage/faq) Instance to store your data.
 
### Step 0: Confirm that you have a valid account on IBM Cloud. 

Goto [https://console.bluemix.net/](https://console.bluemix.net/) and login
(If you do NOT have a valid account for MIT, please contact TBD@us.ibm.com)

### Step 1: Install CLI tools in your local machine (laptop) to remotely access your Cloud resources

#### 1.1. Download and Install the IBM Cloud CLI: `bx`
'bx' allows you to start and manage resources (e.g., applications, containers, services, ...) in the IBM cloud. 

Goto [bx CLI](https://console.bluemix.net/docs/cli/reference/bluemix_cli/download_cli.html#download_install).<br> 
and download and install it following the instructions for your local machine operating system (OSX, Linux or Windows)

#### 1.2. Install `awscli` using [pip](https://pypi.org/project/pip/)

The [aws CLI](https://docs.aws.amazon.com/cli/latest/userguide/) lets you setup and upload data to your buckets. (Will get to this later)

```
pip install awscli
```

#### 1.3. Install `bx` machine-learning plugin

The `machine-learning` plugin for `bx` lets you start, view, and stop your Machine Learning jobs on Watson.

```
bx plugin install machine-learning
```


#### Step 2. Login to IBM Cloud account:
Now we will create data and service resources in the IBM Cloud. First we login.
```
bx login
```

### 3. Configure your account to access IBM Cloud 

In order to run jobs on Watson, you need an `organization` (also called `org`) and a `space` to hold your jobs. 
`Org` names are also globally unique. 

#### 3.1 Use an existing org
The account owner should have already created an `org` for you (and others) to share assets.
You can find out the organizations available for you with the command:<br>
```
bx account orgs
```
The command will return something like:
```
Name                Region     Account owner      Account ID                         Status   
MITIBMWatsonAiLab   us-south   ailab@us.ibm.com   5eb998dd20e3d7fc0153329e32362d64   active   
```
Select the correct `Name` and save it in a variable, i.e.,
```
org_name="MITIBMWatsonAiLab"
bx target -o $org_name
```

#### 3.2 Use your own space
Now lets find out the name of the `space` for you under the `org`. 
```
bx account spaces
```
This will return something like:
```
Getting spaces under organization MITIBMWatsonAiLab in region us-south as myuserid@mit.edu...
OK

Name   
dev 
```
Select the correct space `Name` and save it in a variable, e.g.,
```
space_name="dev"
```
#### 3.3 Lets set the targeted org and space 
```
bx target -o $org_name -s $space_name
```
### Step 4: Create a Watson ML Instance
Now Lets create an instance of Machine Learning Service

#### 4.1. Setup a Watson ML Instance
```
bx service create pm-20 lite CLI_WML_Instance
bx service key-create CLI_WML_Instance cli_key_CLI_WML_Instance
instance_id=`bx service key-show CLI_WML_Instance cli_key_CLI_WML_Instance | grep "instance_id"| awk -F": " '{print $2}'| cut -d'"' -f2`
username=`bx service key-show CLI_WML_Instance cli_key_CLI_WML_Instance | grep "username"| awk -F": " '{print $2}'| cut -d'"' -f2`
password=`bx service key-show CLI_WML_Instance cli_key_CLI_WML_Instance | grep "password"| awk -F": " '{print $2}'| cut -d'"' -f2`
url=`bx service key-show CLI_WML_Instance cli_key_CLI_WML_Instance | grep "url"| awk -F": " '{print $2}'| cut -d'"' -f2`
```
#### 4.2 Save the relevant ids for later use.
```
export ML_INSTANCE=$instance_id
export ML_USERNAME=$username
export ML_PASSWORD=$password
export ML_ENV=$url
```

### Step 5: Create a bucket in the Cloud to store your data

A [bucket](https://datascience.ibm.com/docs/content/analyze-data/ml_dlaas_object_store.html) is a huge "folder" in the cloud. You use the bucket to put and get any file or folder (e.g., your datasets) using an api-style interface.

#### 5.1. Create a cloud storage instance:

First, lets create your own personal cloud storage instance to hold your bucket(s) and name the instance `my_instance`.

```
bx resource service-instance-create `my_instance` cloud-object-storage lite global
bx resource service-instance `my_instance`
```

#### 5.2. Get security credentials:

We then create and get the credentials to `my_instance` and naming it `my_cli_key` so that you can create and access your bucket.

Create key and print it:

```
bx resource service-key-create "my_cli_key" Writer --instance-name "my_instance" --parameters '{"HMAC":true}' > /dev/null 2>&1
access_key_id=`bx resource service-key my_cli_key | grep "access_key_id"| cut -d\:  -f2`
secret_access_key=`bx resource service-key my_cli_key | grep "secret_access_key"| cut -d\:  -f2`
echo ""; echo "Credentials:"; echo "access_key_id - $access_key_id"; echo "secret_access_key - $secret_access_key"; echo ""
```

#### 3.3 Save your keys in a profile so you can reuse them later

Use `aws` tool to add `access_key_id` and `secret_access_key` to a profile and name it `my_profile` (leave the other fields as None).

```
aws configure --profile my_profile
```

#### 3.4 Copy the keys! You'll need them again a little later to access your resources...

#### 3.5. Create a bucket:
Now, lets make a bucket and name it something unique! Buckets are named globally, which means that only one IBM Cloud account can have a bucket with a particular name. Below we use "my_bucket"
```
bucket_name="my_bucket"
aws --endpoint-url=http://s3-api.us-geo.objectstorage.softlayer.net s3api create-bucket --bucket $bucket_name --profile my_profile 2>&1
```

##Congratulations you are done with the one-time SETUP!

##Now, to test that your setup is working, lets try a simple model.

### Step 0: Upload a dataset to your bucket:

```
aws --endpoint-url=https://s3-api.us-geo.objectstorage.softlayer.net --profile my_profile s3 cp cifar10/  s3://$bucket_name/cifar10 --recursive
```



### Step 7: Modify `pytorch-cifar.yml`

This yaml file holds all the information needed for our job including what bucket, ml framework, and computing instance to use.

#### 7.1. Copy the template manifest:

```
cp pytorch-cifar-template.yml pytorch-cifar.yml
```
#### 7.2. Edit `pytorch-cifar.yml`:

Add your author info and replace the values of `aws_access_key_id`, `aws_secret_access_key`, and `bucket` in `pytorch-cifar.yml` with your storage instance credentials and your chosen bucket name from before for both the data and results references.

```yaml
model_definition:
  framework:
#framework name and version (supported list of frameworks available at 'bx ml list frameworks')
    name: pytorch
    version: 0.3
#name of the training-run
  name: cifar10 in pytorch
#Author name and email
  author:
    name: John Doe
    email: johndoe@ibm.com
  description: This is running cifar training on multiple models
  execution:
#Command to execute -- see script parameters in later section !!
    command: python3 main.py --cifar_path ${DATA_DIR}/cifar10
      --checkpoint_path ${RESULT_DIR} --epochs 10
    compute_configuration:
#Valid values for name - k80/k80x2/k80x4/p100/p100x2/v100/v100x2
      name: k80
training_data_reference:
  name: training_data_reference_name
  connection:
    endpoint_url: "https://s3-api.us-geo.objectstorage.service.networklayer.com"
    aws_access_key_id: < from cloud portal >
    aws_secret_access_key: < from cloud portal >
  source:
    bucket: < bucket name >
  type: s3
training_results_reference:
  name: training_results_reference_name
  connection:
    endpoint_url: "https://s3-api.us-geo.objectstorage.service.networklayer.com"
    aws_access_key_id: < from cloud portal >
    aws_secret_access_key: < from cloud portal >
  target:
    bucket: < bucket name >
  type: s3
```

Notice that under execution in the yaml file, we have already specified a command that will be executed when the job reaches the server.

```
python3 main.py --cifar_path ${DATA_DIR}/cifar10
      --checkpoint_path ${RESULT_DIR} --epochs 10
```

This will execute main.py, which starts a training run of a specified model. Since no model is specified, it will train the default model, vgg16, for 10 epochs using the dataset in the bucket we created.

### Step 8: Send code to run on Watson Studio!

#### 8.1. Zip code into a file:
```
zip model.zip main.py models/*
```

#### 8.2. Send your code and manifest to IBM Watson Studio:
```
bx ml train model.zip pytorch-cifar.yml
```

<img src="img/startrun.jpg" width=100%>

That's it! The command should generate a training ID for you, meaning our model has started training on Watson!


### Step 9: Monitor training (Optional)

#### We can check the status of training using the `bx ml list` command:
```
bx ml list training-runs
```
<img src="img/listruns.jpg" width=100%>

#### We can also continuously monitor a training run by using the `bx ml monitor` command:
```
bx ml monitor training-runs < trainingID >
```
<img src="img/monitorruns.jpg" width=100%>

As training proceeds, you should also see results from the training process being copied to the results bucket specified in your training job `yaml` file - `training_results_references.target bucket`.

<img src="img/cloudstorage.jpg" width=100%>

You can also inspect the status of training by downloading and viewing the training log file which has been copied to the result bucket. This is useful in debugging errors and failed jobs.

#### To do this, we run:
```
aws --endpoint-url=https://s3-api.us-geo.objectstorage.softlayer.net --profile my_profile s3 cp s3://my_bucket/ < trainingID Rig >/learner-1/training-log.txt -
```

### Additional Information

See [https://dataplatform.ibm.com/docs/content/analyze-data/ml_dlaas_working_with_new_models.html](https://dataplatform.ibm.com/docs/content/analyze-data/ml_dlaas_working_with_new_models.html)


## Using the tool
```
usage: main.py [-h] [-l] [--lr LR] [--resume] [--cifar_path CIFAR_PATH]
               [--checkpoint_path CP] [--epochs EPOCHS]
               [--model MODEL]

PyTorch CIFAR10 Training

optional arguments:
  -h, --help            show this help message and exit
  -l                    lists all available models (default: False)
  --lr LR               learning rate (default: 0.1)
  --resume, -r          resume from checkpoint (default: False)
  --cifar_path          path to cifar tar.gz (default: ./cifar10)
  --checkpoint_path CP  checkpoint path (default: .)
  --epochs EPOCHS       no of epochs (default: 10)
  --model MODEL         Model type: vgg16,... (default: vgg16)
```

### Example for a vgg19:

```
# on local machine:
python3 main.py --lr 0.01 --epochs 100 --model vgg19

# in Watson Machine Learning yaml file:
python3 main.py --cifar_path ${DATA_DIR}/cifar10
      --checkpoint_path ${RESULT_DIR}
      --epochs 100 --model vgg19 --lr 0.01
```

### Available Models:

```
DPN26  ( <function DPN26 at 0x1129dbc80> )
DPN92  ( <function DPN92 at 0x112a01048> )
DenseNet121  ( <function DenseNet121 at 0x1129e10d0> )
DenseNet161  ( <function DenseNet161 at 0x1129e16a8> )
DenseNet169  ( <function DenseNet169 at 0x1129e1598> )
DenseNet201  ( <function DenseNet201 at 0x1129e1620> )
GoogLeNet  ( <class 'models.googlenet.GoogLeNet'> )
Inception  ( <class 'models.googlenet.Inception'> )
LeNet  ( <class 'models.lenet.LeNet'> )
MobileNet  ( <class 'models.mobilenet.MobileNet'> )
MobileNetV2  ( <class 'models.mobilenetv2.MobileNetV2'> )
PNASNetA  ( <function PNASNetA at 0x112a018c8> )
PNASNetB  ( <function PNASNetB at 0x112a01ea0> )
PreActBottleneck  ( <class 'models.preact_resnet.PreActBottleneck'> )
PreActResNet101  ( <function PreActResNet101 at 0x1129ed730> )
PreActResNet152  ( <function PreActResNet152 at 0x1129ed7b8> )
PreActResNet18  ( <function PreActResNet18 at 0x1129ed1e0> )
PreActResNet34  ( <function PreActResNet34 at 0x1129ed620> )
PreActResNet50  ( <function PreActResNet50 at 0x1129ed6a8> )
ResNeXt29_2x64d  ( <function ResNeXt29_2x64d at 0x1129eea60> )
ResNeXt29_32x4d  ( <function ResNeXt29_32x4d at 0x1129ed0d0> )
ResNeXt29_4x64d  ( <function ResNeXt29_4x64d at 0x1129eef28> )
ResNeXt29_8x64d  ( <function ResNeXt29_8x64d at 0x1129ed048> )
ResNet101  ( <function ResNet101 at 0x1129ee840> )
ResNet152  ( <function ResNet152 at 0x1129ee8c8> )
ResNet18  ( <function ResNet18 at 0x1129ee2f0> )
ResNet34  ( <function ResNet34 at 0x1129ee730> )
ResNet50  ( <function ResNet50 at 0x1129ee7b8> )
SENet18  ( <function SENet18 at 0x112a01378> )
SepConv  ( <class 'models.pnasnet.SepConv'> )
ShuffleNet  ( <class 'models.shufflenet.ShuffleNet'> )
ShuffleNetG2  ( <function ShuffleNetG2 at 0x1129e1c80> )
ShuffleNetG3  ( <function ShuffleNetG3 at 0x1129ee158> )
VGG11  ( config in  <class 'models.vgg.VGG'> )
VGG13  ( config in  <class 'models.vgg.VGG'> )
VGG16  ( config in  <class 'models.vgg.VGG'> )
VGG19  ( config in  <class 'models.vgg.VGG'> )
```




### Accuracy as reported by `kuangliu`:
| Model             | Acc.        |
| ----------------- | ----------- |
| [VGG16](https://arxiv.org/abs/1409.1556)              | 92.64%      |
| [ResNet18](https://arxiv.org/abs/1512.03385)          | 93.02%      |
| [ResNet50](https://arxiv.org/abs/1512.03385)          | 93.62%      |
| [ResNet101](https://arxiv.org/abs/1512.03385)         | 93.75%      |
| [MobileNetV2](https://arxiv.org/abs/1801.04381)       | 94.43%      |
| [ResNeXt29(32x4d)](https://arxiv.org/abs/1611.05431)  | 94.73%      |
| [ResNeXt29(2x64d)](https://arxiv.org/abs/1611.05431)  | 94.82%      |
| [DenseNet121](https://arxiv.org/abs/1608.06993)       | 95.04%      |
| [PreActResNet18](https://arxiv.org/abs/1603.05027)    | 95.11%      |
| [DPN92](https://arxiv.org/abs/1707.01629)             | 95.16%      |

## Enjoy
Hendrik Strobelt (IBM Research), Evan Phibbs (IBM Research), Victor C. Dibia (IBM Research)

