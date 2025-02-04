![chronograph-logo-no-icon-normal](https://user-images.githubusercontent.com/16158417/62494113-b45aa300-b7a0-11e9-964b-e657d377f444.png)


# lambda-OCRmyPDF

Adapting the python library [OCRmyPDF](https://github.com/jbarlow83/OCRmyPDF/) to run as an AWS Lambda Function.

From the OCRmyPDF readme:

> OCRmyPDF adds an OCR text layer to scanned PDF files, allowing them to be searched or copy-pasted.

## Purpose

The purpose of this application was to adapt the **OCRmyPDF** application/library to be run on AWS servers to serve as a dynamic document OCR processing service.  We loved the implementation of OCRmyPDF but felt it would work well if adapted to an Amazon Lambda Function.

## What's in the repo?

This repository contains all external libraries required by OCRmyPDF compiled on and extracted from an Amazon Linux EC2 instance.  It also contains all python packages compiled on and extracted from an Amazon Linux EC2 instance.  Lastly, it features some minor changes to the OCRmyPDF source itself to make it Lambda friendly.  The resulting zipfile extracts to around `261,459,415` bytes, which comes just under the `262,144,000` byte limit set by AWS.

## Calling the Event

The event currently supports only a few, basic, parameters, which we intend on expanding.  The parameters are:

| Key | Description|
|---------|----------|
| awsRegion | The region where the S3 bucket is located |
| s3.bucket.name | The name of the S3 bucket |
| s3.object.key | The key for the object in the S3 bucket |
| pages | The pages parameter for OCRmyPDF.  Ex: "1", "3-5", "1,3-5,8" |
| doBackup | Creates a mirror of the S3 object appended with `.bak` |


## Installation
### Download Latest Release

Download the [latest release](https://github.com/chronograph-pe/lambda-OCRmyPDF/releases) from this repository's releases page.

[Optional]: You can also download the source here and compress it to your own .zip file.  However you may need to update the permissions of the files within the /bin and /lib folders, as discovered in [this issue](https://github.com/chronograph-pe/lambda-OCRmyPDF/issues/2).

### Create the Function

- Go to S3 and upload the downloaded zip file to an S3 bucket of your choosing.  Keep track of the url of the file for later.
- Access the AWS Lambda dashboard and click on Create Function.
- Ensure `Author from scratch` is selected
- Name your function as you please.
- Make sure that your Runtime is set to `Python 3.6`
- If you do not have an IAM Role set up for S3 access, set one up with Read, Write access on S3.  I used AWS's `AWSLambdaExecute` policy as a base.

### Setup the Function

#### manually

- Under the Function code section:
    - Set the **Code entry type** field to `Upload a file from Amazon S3`
    - Set the **Runtime** field to `Python 3.6`
    - Set the handler field to `apply-ocr-to-s3-object.apply_ocr_to_document_handler`
    - Copy and paste the S3 object URL from above into the **Amazon S3 link URL** field
    - Increase the timeout to max (900 sec)
    - Set memory size to your desired size
     - Required for OCRmyPDF is ~256MB. lambda max is 3008MB
     - Denote - incresing the memory size may reduce the runtime drastically and increases the lambda per-time cose see [aws pricing](https://aws.amazon.com/lambda/pricing/)
- Under the Environment Variables section:
    - Set the below environment variables to their respective paths
    - **PATH** : `/var/task/bin`
    - **PYTHONPATH** : `/var/task/python`
    - **TESSDATA_PREFIX** : `/var/task/tessdata`
    
#### Or by script

```bash

lambda_name="your_lambda_name"
s3_bucket="your_bucket"
s3_file_key="your_s3_file_key.zip"

zip_file_name="lambda-ocrtopdf.zip"
download_url="https://github.com/chronograph-pe/lambda-OCRmyPDF/releases/download/v1.0-alpha/lambda-ocrtopdf.zip"

wget -O $zip_file_name -L $download_url

aws s3 cp  $zip_file_name s3://$s3_bucket/$s3_file_key
aws lambda update-function-code --function-name $lambda_name --s3-bucket $s3_bucket --s3-key $s3_file_key
aws lambda update-function-configuration --function-name $lambda_name \
                                        --handler apply-ocr-to-s3-object.apply_ocr_to_document_handler \
                                        --environment "Variables={PATH=/var/task/bin,PYTHONPATH=/var/task/python,TESSDATA_PREFIX=/var/task/tessdata}" \
                                        --timeout 900 \
                                        --memory-size 3008 \
                                        --runtime "python3.6"
```

### Test the Function

The following test configuration can be added to lambda to test the functionality.  Upload any pdf called `input.pdf` to an S3 bucket and run this test configuration:

**input_test_configuration.json**
```json
{
  "pages": "1",
  "awsRegion": "[YOUR AWS REGION HERE]",
  "s3": {
    "bucket": {
      "name": "[YOUR BUCKET NAME HERE]"
    },
    "object": {
      "key": "input.pdf"
    }
  }
}
```

# To Do:
- Add additional language support
- Continue to trim down python packages
