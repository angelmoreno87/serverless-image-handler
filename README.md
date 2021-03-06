# AWS Serverless Image Handler Lambda wrapper for Thumbor
A solution to dynamically handle images on the fly, utilizing Thumbor (thumbor.org).
Published version, additional details and documentation are available here: https://aws.amazon.com/answers/web-applications/serverless-image-handler/

_Note:_ it is recommend to build the application binary on Amazon Linux.

## Running unit tests for customization
* Clone the repository, then make the desired code changes
* Next, run unit tests to make sure added customization passes the tests
```
cd ./deployment
chmod +x ./run-unit-tests.sh  \n
./run-unit-tests.sh \n
```

## Building distributable for customization
* Configure the bucket name of your target Amazon S3 distribution bucket
```
export TEMPLATE_OUTPUT_BUCKET=my-bucket-name # bucket where cfn template will reside
export DIST_OUTPUT_BUCKET=my-bucket-name # bucket where customized code will reside
export VERSION=my-version # version number for the customized code
```
_Note:_ You would have to create 2 buckets, one named 'my-bucket-name' and another regional bucket named 'my-bucket-name-<aws_region>'; aws_region is where you are testing the customized solution. Also, the assets  in bucket should be publicly accessible.

* OS/Python Environment Setup
```bash
yum install yum-utils epel-release -y
sudo yum-config-manager --enable epel
sudo yum update -y
sudo yum install zip wget git libpng-devel libcurl-devel gcc python-devel libjpeg-devel -y
alias sudo='sudo env PATH=$PATH'
sudo pip install setuptools==39.0.1
sudo pip install virtualenv==15.2.0
```
* Clone the github repo
```bash
git clone https://github.com/awslabs/serverless-image-handler.git
```

* Navigate to the deployment folder
```bash
cd serverless-image-handler/deployment
```

* Now build the distributable
```bash
sudo ./build-s3-dist.sh $DIST_OUTPUT_BUCKET $VERSION
```

* Deploy the distributable to an Amazon S3 bucket in your account. Note: you must have the AWS Command Line Interface installed.
```bash
aws s3 cp ./dist/ s3://$DIST_OUTPUT_BUCKET-[region_name]/serverless-image-handler/$VERSION/ --recursive --exclude "*" --include "*.zip"
aws s3 cp ./dist/serverless-image-handler.template s3://$TEMPLATE_OUTPUT_BUCKET/serverless-image-handler/$VERSION/
```
_Note:_ In the above example, the solution template will expect the source code to be located in the my-bucket-name-[region_name] with prefix serverless-image-handler/my-version/serverless-image-handler.zip

* Get the link of the serverless-image-handler.template uploaded to your Amazon S3 bucket.
* Deploy the Serverless Image Handler solution to your account by launching a new AWS CloudFormation stack using the link of the serverless-image-handler.template
```bash
https://s3.amazonaws.com/my-bucket-name/serverless-image-handler/my-version/serverless-image-handler.template
```

## SafeURL hash calculation
* For hash calculation with safe URL, use following snippet to find signed_path value
```bash
http_key='mysecuritykey' # security key provided to lambda env variable
http_path='200x200/smart/sub-folder/myimage.jpg' # sample options for myimage
hashed = hmac.new(str(http_key),str(http_path),sha1)
encoded = base64.b64encode(hashed.digest())
signed_path = encoded.replace('/','_').replace('+','-')
```

## Saving output image to S3

It is possible to save the output image to a S3 bucket adding extra HTTP Headers.
In order to enable output saving:

- Open the AWS console and go to the Lambda section
- Open the *-ImageHandlerFunction-* lambda
- Add S3_SAVE_SECRET to the Lambda environment. Choose a secret key long enough, e.g. 32 characters, digits and letters in mixed 
  case. This is the key that must be supplied by the invoker to save on S3.
- (optional) add S3_SAVE_BUCKET to the Lambda environment. If missing, TC_AWS_LOADER_BUCKET is used as fallback.
  This is the target bucket.
- edit the Lambda role to add "s3:PutObject" permission to the policy on the target bucket (S3_SAVE_BUCKET).
- save the changes

Now you are ready to request save output. You need to set `X-save-s3-key` to the request header with the value of the
S3 path (key) where the file must be saved. In this case you must also set the `X-save-secret` request header with a string
that matches the S3_SAVE_SECRET.

Now you can issue two different type of requests, depending on your needs:

- GET request: the output image to be returned in the body of the request. HTTP status will be 200 OK;
- POST request: the output image to be returned in the body of the request. HTTP status will be 201 Created.

Example:

    # image returned in the body
	curl -H "X-save-s3-key: my/output/image.jpg" -H "X-save-secret: 123456789" https://a12345.execute-api.us-east-1.amazonaws.com/image/300x300/source.jpg

    # image not returned in the body
	curl -I -X POST -H "X-save-s3-key: out/image.jpg" -H "X-save-secret: ayzd5BEk7znTCMgNRQMkfyfTa54A6vA6" https://a12345.execute-api.us-east-1.amazonaws.com/image/300x300/source.jpg

If you just the POST request, the 6 MB limit for the output image does not apply.

***

Copyright 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.

Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License. A copy of the License is located at

    http://aws.amazon.com/asl/

or in the "license" file accompanying this file. This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied. See the License for the specific language governing permissions and limitations under the License.
