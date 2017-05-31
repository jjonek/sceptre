.. highlight:: shell
..  _template:

=========
Templates
=========

Sceptre uses CloudFormation or Troposphere templates to launch AWS Stacks. Conventionally, templates are stored in a directory named templates, in the same directory as the config directory::

  .
  ├── config
  │   └── dev
  │       ├── config.yaml
  │       └── vpc.yaml
  └── templates
    └── vpc.py

Note that as a path to the template is supplied in a stack's Stack Config file, templates may be stored at any arbitrary location on disk.


CloudFormation
--------------

Templates with ``.json`` or ``.yaml`` extensions are treated as CloudFormation templates. Sceptre supports the use of templating in CloudFormation template files.

Internally, Sceptre uses Jinja2 for templating, so any valid Jinja2 syntax should work with Sceptre templating.

Templating can use `sceptre_user_data` defined in the stack's Stack Config file.

Example
```````

Create a S3 bucket policy for CloudTrail with configurable list of allowed accounts:

.. code-block:: yaml

  AWSTemplateFormatVersion: '2010-09-09'
  Resources:
  CloudTrailBucketPolicy:
      Type: 'AWS::S3::BucketPolicy'
      Properties:
      Bucket: !Ref CloudTrailBucket
      PolicyDocument:
          Version: '2012-10-17'
          Statement:
          - Sid: 'AWSCloudTrailWrite'
              Effect: 'Allow'
              Principal:
              Service: 'cloudtrail.amazonaws.com'
              Action: 's3:PutObject'
              Resource:
              {% for account_id in sceptre_user_data.cloudtrail_accounts %}
              - !Sub 'arn:aws:s3:::${CloudTrailBucket}/AWSLogs/{{ account_id }}/*'
              {% endfor %}


Troposphere
-----------

Templates with a ``.py`` extension are treated as Troposphere templates. They should implement a function named ``sceptre_handler(sceptre_user_data)`` which returns the CloudFormation template as a ``string``. Sceptre User Data is passed to this function as an argument. If Sceptre User Data is not defined in the Stack Config file, Sceptre passes an empty ``dict``.


Example
```````

.. code-block:: python

  from troposphere import Template, Parameter, Ref
  from troposphere.ec2 import VPC


  class Vpc(object):
      def __init__(self, sceptre_user_data):
          self.template = Template()
          self.sceptre_user_data = sceptre_user_data
          self.add_vpc()

      def add_vpc(self):
          self.vpc = self.template.add_resource(VPC(
              "VirtualPrivateCloud",
              CidrBlock=self.sceptre_user_data["cidr_block"]
          ))


  def sceptre_handler(sceptre_user_data):
      vpc = Vpc(sceptre_user_data)
      return vpc.template.to_json()
