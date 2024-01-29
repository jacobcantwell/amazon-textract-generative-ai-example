# Amazon Textract Generative AI Example

I wrote a lambda function that takes an image in Amazon S3 as a input and outputs JSON content so you can see the query responses.

With the combination of being able to improve the annotations and potentially add human reviews, if the training model is refined with a variety of images then you will be able to create highly accurate predictions with this Amazon Textract method.

## AWS Lambda Code

* Function name: read-ultrasound-with-textract-v1
* Description: read-ultrasound-with-textract-v1
* Runtime: Python 3.12

```python3
import json
import boto3
import urllib

textract = boto3.client('textract', region_name='ap-southeast-2')

TEXTRACT_ADAPTER_ID = "a#####e#####"
TEXTRACT_ADAPTER_VERSION = "1"

queries = [
    { 'Text': "Did patient procedure matching occur?", 'Alias': "PATIENT_PROCEDURE_MATCHING" },
    { 'Text': "What were the sonographer's notes?", 'Alias': "SONOGRAPHERS_NOTES" },
    { 'Text': "What was the Liver size?", 'Alias': "LIVER_SIZE_VALUE" },
    { 'Text': "What was the Liver texture Normal or Abnormal?", 'Alias': "LIVER_TEXTURE" },
]

def lambda_handler(event, context):

    # Get the object from the event and show its content type
    s3_bucket = event['Records'][0]['s3']['bucket']['name']
    s3_object_key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')

    # Call Amazon Textract
    response = textract.analyze_document(
        Document={
            'S3Object': {
                'Bucket': s3_bucket,
                'Name': s3_object_key
            }
        },
        FeatureTypes=['QUERIES'],
        QueriesConfig={
            'Queries': queries
        })

    # Print out text, forms, tables found
    response_blocks = []
    for block in response['Blocks']:
        if block['BlockType'] == 'QUERY':
            # print(block)
            query_question_id = block['Id']
            query_question_alias = block['Query']['Alias']
            query_question_text = block['Query']['Text']
            query_answer_id = block['Relationships'][0]['Ids'][0]
            
            answer_block = next((item for item in response['Blocks'] if item["Id"] == query_answer_id), None)
            print(answer_block)
            query_answer_text = answer_block['Text']
            query_answer_confidence = answer_block['Confidence']

            response_blocks.append({
              'alias': query_question_alias,
              'question': query_question_text,
              'answer': query_answer_text,
              'confidence': query_answer_confidence,
            })
    print(response_blocks)
    return response_blocks
```

## Invoking Custom Adapter in the AWS Management Console

* Select the adapter
* Select the version
* Select Try Adapter
* Choose a document
* Download results

This will download a JSON file and a CSV files. The queryAnswers.csv file includes the query questions and responses.

## Invoking Custom Adapter in an AWS Lambda function

* A sample python AWS Lamdba calls Amazon Textract
* Open AWS Lambda
* Open read-ultrasound-with-textract-v1
* Create an Amazon S3 trigger test event that points to the image to analyze
* Response is JSON list of the queries - output can be converted to whatever format is needed or saved in a database

### Test S3 Trigger event

```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "name": "textract-ultrasound-forms"
        },
        "object": {
          "key": "ultrasound-image-01.png"
        }
      }
    }
  ]
}
```

## Improving Annotations

When annotating your documents, you can choose to auto-label your documents using the pretrained Queries feature and then edit the labels where needed.

When auto-labeling your documents, you specify the appropriate queries for your document. When you finish adding queries to your documents, Amazon Textract attempts to extract the proper elements from your documents, generating annotations. You must then verify the accuracy of these annotations, correcting any that are incorrect. By linking queries to answers, you teach the model what information is important in your documents.

* https://docs.aws.amazon.com/textract/latest/dg/textract-preparing-training-testing.html

## Using Amazon Augmented AI for Human Review

Amazon Augmented AI lets you add human review of low-confidence predictions or random prediction samples from Amazon Textract. Amazon Augmented AI (Amazon A2I) is a service that brings human review of ML predictions to all developers by removing the heavy lifting associated with building human review systems or managing large numbers of human reviewers. 

* https://docs.aws.amazon.com/sagemaker/latest/dg/a2i-use-augmented-ai-a2i-human-review-loops.html
* https://docs.aws.amazon.com/textract/latest/dg/a2i-textract.html







