# CloudFormation

This is the DevOps repository for creating CloudFormation templates from Ruby scripts.  This repository of scripts is
based on the (cloudformation-ruby-dsl)[https://github.com/bazaarvoice/cloudformation-ruby-dsl]


### Installing the cloudformation-ruby-dsl gem and dependencies

To avoid issues with different versions of gems we are using bundler.  To install all the needed dependencies run the following commands:

    cd ~/path/to/cloudformation
    bundle install

You will now have all the gems needed for CloudFormation.  Make sure that that you run any commands with `bundle exec`.

Set the following alias to make sure that the launch script is run with `bundle exec`:

    alias launch='cd /Users/andyboutte/Documents/Shopatron/Repositories/RubyMine/cloudformation/bin/; bundle exec launch'
    
### Add Cloudformation/bin to your path

    vim ~/.bash_profile
    
    export PATH=/Users/dmarkley/Documents/Shopatron/Repositories/cloudformation/bin:$PATH

### Setting up credential file

    vim ~/.aws.credential

    AWSAccessKeyId=<Write your AWS access ID>
    AWSSecretKey=<Write your AWS secret key>

### Adding environment variable with path to credential file

Add the following line to `/Users/andyboutte/.bash_profile`

    export AWS_CREDENTIAL_FILE="/Users/andyboutte/.aws.credential"
    export AWS_DEFAULT_REGION="us-west-2"

# Steps to converting an existing template

### Convert the template to ruby

    cfntemplate-to-ruby tron_vpc.template.json > tron.rb
    chmod +x tron.rb
    ruby ./tron.rb

### Bring in the common components

    require '../../lib/autoloader.rb'
    
    # Variables
    Application='inventory'
    Chef_attribute_directory=Application
    tier='web'
    time = Time.now
    timestamp = "#{time.day}-#{time.month}-#{time.year}"
    iam_policies = []
    user_data_scripts = [ "#{$common_dir}/userdata/00-start.sh", './07-roles-json.sh', "#{$common_dir}/userdata/05-installChef.sh", "#{$common_dir}/userdata/10-chefAliases.sh", "#{$common_dir}/userdata/15-runChef.sh", "#{$common_dir}/userdata/100-end.sh", ]
    user_data_scripts_leader = [ "#{$common_dir}/userdata/00-start.sh", './07-roles-json.sh', "#{$common_dir}/userdata/05-installChef.sh", "#{$common_dir}/userdata/10-chefAliases.sh", "#{$common_dir}/userdata/15-runChef.sh", "#{$common_dir}/userdata/100-end.sh", ]

### Remove all tags

Delete any and all tags that are on SGs, ELB, instances, etc.  We can now globally set tags and they will be automatically added to everything

### Bring in the IAMRole

      resource 'IAMRole', :Type => 'AWS::IAM::Role', :Properties => {
          :AssumeRolePolicyDocument => {
              :Statement => [
                  {
                      :Effect => 'Allow',
                      :Principal => { :Service => 'ec2.amazonaws.com' },
                      :Action => [ 'sts:AssumeRole' ],
                  },
              ],
          },
          :Path => '/',
      }

      resource 'InstanceProfile', :Type => 'AWS::IAM::InstanceProfile', :Properties => {
          :Path => '/',
          :Roles => [ ref('IAMRole') ],
      }

      # Take the requested policies from iam_policies and build them up to one big IAM role
      combine_iam_policies(iam_policies)

This will auto generate an IAM role from policies defined in ./lib/common/iam_policies and any custom ones you need from `iam_policies`

--------------------------------------------------------------


### To get list of commands for the tron.rb script

    ./tron.rb --help

### To get list of parameters for each of the sub commands in tron.rb

    ./tron.rb cfn-validate-template --help
    ./tron.rb cfn-create-stack --help

### Examples

#### Expand ruby script into CloudFormation json

    ./tron.rb expand

#### Template Validation

    ./tron.rb cfn-validate-template --access-key-id <your-access-key> --secret-key <your-secret-key> --region us-west-2

#### Launching a Stack

To use all of the default parameters values:

    ./tron.rb cfn-create-stack tron-dev-$(date '+%m-%d-%y--%H-%M-%S') --aws-credential-file ~/.aws.credential --region us-west-2 --disable-rollback --timeout 20

To modify any of the parameters:

    ./tron.rb cfn-create-stack tron-qa-$(date '+%m-%d-%y--%H-%M-%S') --aws-credential-file ~/.aws.credential --region us-west-2 --disable-rollback --timeout 20 --parameters "Environment=qa;InstanceType=t2.small"

When creating a stack that has IAM resources (ie creating an IAM role) you need to confirm this

    ./manager.rb cfn-create-stack manager-dev-$(date '+%m-%d-%y--%H-%M-%S') --aws-credential-file ~/.aws.credential --region us-west-2 --disable-rollback --timeout 20 --capabilities CAPABILITY_IAM
