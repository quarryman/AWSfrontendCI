# Continuous deployment of React websites to Amazon S3
This sample includes a continuous deployment pipiline for websites built with React. We use AWS CodePipeline, CodeBuild, and [SAM](https://github.com/awslabs/serverless-application-model) to deploy the application. To deploy the application to S3 using SAM we use a custom CloudFormation resource.

# Files included
* `buildspec.yml`: YAML configuration for CodeBuild, this file should be in the root of your code repository
* `configure.js`: Script executed in the build step to generate a config.json file for the application, this is used to include values exported by other CloudFormation stacks (separate services of the same application).
* `index.js`: Custom CloudFormation resource that publishes the website to an S3 bucket. As you can see from the buildspec and SAM template, this function is located in a `s3-deployment-custom-resource` sub-folder of the repo
* `app-sam.yaml`: Serverless Application model YAML file. This configures the S3 bucket and the custom resource
* `serverless-pipeline.yaml`: The pipeline template to deploy the whole caboodle. The pipeline is currently configured to pick up the source from GitHub

The pipeline workflow includes the following stages:

## Source
* Pick up source files from git repository, the source include the unmodified react code

## Beta
### CodeBuild
Using a CodeBuild container (`aws/codebuild/nodejs:6.3.1`) we crate the deployment packages for the application

* Pull down all dependencies with `npm` for the website and the custom CloudFormation resource
* Run a `configure.js` script, this script reads output values from other CloudFormation stacks and generates a config.json file used by the read app - an example could be the Cognito User Pool Id. I use the stage name in the stack, this is passed to the script when started and is accessed as an environment variable of the CodeBuild container - this is why we have one build step per stage of the pipeline
* Run `npm start build` command
* Run `cloudformation package` command to prepare our SAM template for deployment
* Zip up the build directory from the website
* Outputs from the container are the processed SAM template and the zip file with the website content

> The buidspec.yml file included in this gist contains all of the commands above. I've also included a sample of the configure.js script (code needs a lot of cleanup)

### CreateChangeSet
CloudFormation step that uses our SAM template to start the deployment of the website

* This step creates the CloudFormation ChangeSet required to deploy our SAM template. 
* The template receives two additional parameters: The source artifact bucket and source artifact key in the bucket. These parameters are needed because our custom CloudFormation resource will read the CodeBuild output file, extract the zipped contents from the React build folder, and copy them to an S3 bucket

> To override the parameters passed to CloudFormation we use the `ParameterOverrides` property in CodePipelines with the following JSON
> 
> ```
> {
>   "SourceBucket" : { "Fn::GetArtifactAtt" : ["BetaBuiltZip", "BucketName"]},
>   "SourceArtifact" : { "Fn::GetArtifactAtt" : ["BetaBuiltZip", "ObjectKey"]}
> }
> ```

### ExecuteChangeSet
This step actually runs the template we created in the previous step

## Gamma
Rinse and repeat. For gamma and prod we simply repeat the steps above.