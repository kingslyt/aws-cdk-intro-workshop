+++
title = "Polish Pipeline"
weight = 5000
+++

## Get Endpoints
Stepping back, we can see a problem now that our app is being deployed by our pipeline. There is no easy way to find the endpoints of our application (the `TableViewer` and `APIGateway` endpoints), so we can't call it! Let's add a little bit of code to expose these more obviously.

First edit `infra/workshop-stack.go` to get these values and expose them as properties of our stack:

{{<highlight go "hl_lines=18-30 48 52 57-63 65 68-74">}}
package infra

import (
	"cdk-workshop/hitcounter"

	"github.com/aws/aws-cdk-go/awscdk/v2"
	"github.com/aws/aws-cdk-go/awscdk/v2/awslambda"
	"github.com/aws/aws-cdk-go/awscdk/v2/awsapigateway"
	"github.com/aws/constructs-go/constructs/v10"
	"github.com/aws/jsii-runtime-go"
	"github.com/cdklabs/cdk-dynamo-table-viewer-go/dynamotableviewer"
)

type CdkWorkshopStackProps struct {
	awscdk.StackProps
}

type cdkWorkshopStack struct {
	awscdk.Stack
	hcViewerUrl awscdk.CfnOutput
	hcEndpoint  awscdk.CfnOutput
}

type CdkWorkshopStack interface {
	awscdk.Stack
	HcViewerUrl() awscdk.CfnOutput
	HcEndpoint() awscdk.CfnOutput
}

func NewCdkWorkshopStack(scope constructs.Construct, id string, props *CdkWorkshopStackProps) CdkWorkshopStack {
	var sprops awscdk.StackProps
	if props != nil {
		sprops = props.StackProps
	}
	stack := awscdk.NewStack(scope, &id, &sprops)

	helloHandler := awslambda.NewFunction(stack, jsii.String("HelloHandler"), &awslambda.FunctionProps{
		Code: awslambda.Code_FromAsset(jsii.String("lambda"), nil),
		Runtime: awslambda.Runtime_NODEJS_16_X(),
		Handler: jsii.String("hello.handler"),
	})

	hitcounter := hitcounter.NewHitCounter(stack, "HelloHitCounter", &hitcounter.HitCounterProps{
		Downstream: helloHandler,
		ReadCapacity: 10,
	})

	gateway := awsapigateway.NewLambdaRestApi(stack, jsii.String("Endpoint"), &awsapigateway.LambdaRestApiProps{
		Handler: hitcounter.Handler(),
	})

	tv := dynamotableviewer.NewTableViewer(stack, jsii.String("ViewHitCounter"), &dynamotableviewer.TableViewerProps{
		Title: jsii.String("Hello Hits"),
		Table: hitcounter.Table(),
	})

	hcViewerUrl := awscdk.NewCfnOutput(stack, jsii.String("GatewayUrl"), &awscdk.CfnOutputProps{
		Value: gateway.Url(),
	})

	hcEndpoint := awscdk.NewCfnOutput(stack, jsii.String("TableViewerUrl"), &awscdk.CfnOutputProps{
		Value: tv.Endpoint(),
	})

	return &cdkWorkshopStack{stack, hcViewerUrl, hcEndpoint}
}

func (s *cdkWorkshopStack) HcViewerUrl() awscdk.CfnOutput {
	return s.hcViewerUrl
}

func (s *cdkWorkshopStack) HcEndpoint() awscdk.CfnOutput {
	return s.hcEndpoint
}
{{</highlight>}}

By adding outputs `hcViewerUrl` and `hcEnpoint`, we expose the necessary endpoints to our HitCounter application. We are using the core construct `CfnOutput` to declare these as Cloudformation stack outputs (we will get to this in a minute).

Let's commit these changes to our repo (`git commit -am "MESSAGE" && git push`), and navigate to the [Cloudformation console](https://console.aws.amazon.com/cloudformation). You can see there are three stacks.

* `CDKToolkit`: The first is the integrated CDK stack (you should always see this on bootstrapped accounts). You can ignore this.
* `WorkshopPipelineStack`: This is the stack that declares our pipeline. It isn't the one we need right now.
* `Deploy-WebService`: Here is our application! Select this, and under details, select the `Outputs` tab. Here you should see four endpoints (two pairs of duplicate values). Two of them, `EndpointXXXXXX` and `ViewerHitCounterViewerEndpointXXXXXXX`, are defaults generated by Cloudformation, and the other two are the outputs we declared ourselves.

![](./stack-outputs.png)

If you click the `TableViewerUrl` value, you should see our pretty hitcounter table that we created in the initial workshop.

## Add Validation Test
Now we have our application deployed, but no CD pipeline is complete without tests!

Let's start with a simple test to ping our endpoints to see if they are alive.
Return to `infra/pipeline-stack.go` and add the following:

{{<highlight go "hl_lines=40-61">}}
package infra

import (
	"github.com/aws/aws-cdk-go/awscdk/v2"
	"github.com/aws/aws-cdk-go/awscdk/v2/awscodecommit"
	"github.com/aws/aws-cdk-go/awscdk/v2/pipelines"
	"github.com/aws/constructs-go/constructs/v10"
	"github.com/aws/jsii-runtime-go"
)

type PipelineStackProps struct {
	awscdk.StackProps
}

func NewPipelineStack(scope constructs.Construct, id string, props *PipelineStackProps) awscdk.Stack {
	var sprops awscdk.StackProps
	if props != nil {
		sprops = props.StackProps
	}
	stack := awscdk.NewStack(scope, &id, &sprops)

	repo := awscodecommit.NewRepository(stack, jsii.String("WorkshopRepo"), &awscodecommit.RepositoryProps{
		RepositoryName: jsii.String("WorkshopRepo"),
	})

	pipeline := pipelines.NewCodePipeline(stack, jsii.String("Pipeline"), &pipelines.CodePipelineProps{
		PipelineName: jsii.String("WorkshopPipeline"),
		Synth: pipelines.NewCodeBuildStep(jsii.String("SynthStep"), &pipelines.CodeBuildStepProps{
			Input: pipelines.CodePipelineSource_CodeCommit(repo, jsii.String("main"), nil),
			Commands: jsii.Strings(
				"npm install -g aws-cdk",
				"goenv install 1.18.3",
				"goenv local 1.18.3",
				"npx cdk synth",
			),
		}),
	})

	deploy := NewWorkshopPipelineStage(stack, "Deploy", nil)
	deployStage := pipeline.AddStage(deploy, nil)

	deployStage.AddPost(
		pipelines.NewCodeBuildStep(jsii.String("TestViewerEndpoint"), &pipelines.CodeBuildStepProps{
			ProjectName: jsii.String("TestViewerEndpoint"),
			EnvFromCfnOutputs: &map[string]awscdk.CfnOutput{
				"ENDPOINT_URL": //TBD
			},
			Commands: jsii.Strings("curl -Ssf $ENDPOINT_URL"),
		}),
		pipelines.NewCodeBuildStep(jsii.String("TestAPIGatewayEndpoint"), &pipelines.CodeBuildStepProps{
			ProjectName: jsii.String("TestAPIGatewayEndpoint"),
			EnvFromCfnOutputs: &map[string]awscdk.CfnOutput{
				"ENDPOINT_URL": //TBD
			},
			Commands: jsii.Strings(
				"curl -Ssf $ENDPOINT_URL",
				"curl -Ssf $ENDPOINT_URL/hello",
				"curl -Ssf $ENDPOINT_URL/test",
			),
		}),
	)

	return stack
}
{{</highlight>}}

We add post-deployment steps via `deployStage.AddPost(...)` from CDK Pipelines. We add two actions to our deployment stage: to test our TableViewer endpoint and our APIGateway endpoint, respectively.

> Note: We submit several `curl` requests to the APIGateway endpoint so that when we look at our tableviewer, there are several values already populated.

You may notice that we have not yet set the URLs of these endpoints. This is because they are not yet exposed to this stack!

With a slight modification to `infra/pipeline-stage.go` we can expose them:

{{<highlight go "hl_lines=12-22 24 31 33 36-46">}}
package infra

import (
	"github.com/aws/aws-cdk-go/awscdk/v2"
	"github.com/aws/constructs-go/constructs/v10"
)

type WorkshopPipelineStageProps struct {
	awscdk.StageProps
}

type workshopPipelineStage struct {
	stage awscdk.Stage
	hcViewerUrl awscdk.CfnOutput
	hcEndpoint  awscdk.CfnOutput
}

type WorkshopPipelineStage interface {
	Stage() awscdk.Stage
	HcViewerUrl() awscdk.CfnOutput
	HcEndpoint() awscdk.CfnOutput
}

func NewWorkshopPipelineStage(scope constructs.Construct, id string, props *WorkshopPipelineStageProps) WorkshopPipelineStage {
	var sprops awscdk.StageProps
	if props != nil {
		sprops = props.StageProps
	}
	stage := awscdk.NewStage(scope, &id, &sprops)

	workshopStack := NewCdkWorkshopStack(stage, "WebService", nil)

	return &workshopPipelineStage{stage, workshopStack.HcViewerUrl(), workshopStack.HcEndpoint()}
}

func (s *workshopPipelineStage) Stage() awscdk.Stage {
	return s.stage
}

func (s *workshopPipelineStage) HcViewerUrl() awscdk.CfnOutput {
	return s.hcViewerUrl
}

func (s *workshopPipelineStage) HcEndpoint() awscdk.CfnOutput {
	return s.hcEndpoint
}
{{</highlight>}}

Now we can add those values to our actions in `infra/pipeline-stack.go` by getting the `stackOutput` of our pipeline stack:
{{<highlight go "hl_lines=2 8 15">}}
	// CODE HERE...
	deployStage := pipeline.AddStage(deploy.Stage(), nil)

	deployStage.AddPost(
		pipelines.NewCodeBuildStep(jsii.String("TestViewerEndpoint"), &pipelines.CodeBuildStepProps{
			ProjectName: jsii.String("TestViewerEndpoint"),
			EnvFromCfnOutputs: &map[string]awscdk.CfnOutput{
				"ENDPOINT_URL": deploy.HcViewerUrl(),
			},
			Commands: jsii.Strings("curl -Ssf $ENDPOINT_URL"),
		}),
		pipelines.NewCodeBuildStep(jsii.String("TestAPIGatewayEndpoint"), &pipelines.CodeBuildStepProps{
			ProjectName: jsii.String("TestAPIGatewayEndpoint"),
			EnvFromCfnOutputs: &map[string]awscdk.CfnOutput{
				"ENDPOINT_URL": deploy.HcEndpoint(),
			},
			Commands: jsii.Strings(
				"curl -Ssf $ENDPOINT_URL",
				"curl -Ssf $ENDPOINT_URL/hello",
				"curl -Ssf $ENDPOINT_URL/test",
			),
		}),
	)
{{</highlight>}}

## Commit and View!
Commit those changes, wait for the pipeline to re-deploy the app, and navigate back to the [CodePipeline Console](https://console.aws.amazon.com/codesuite/codepipeline/pipelines) and you can now see that there are two test actions contained within the `Deploy` stage!

![](./pipeline-tests.png)

Congratulations! You have successfully created a CD pipeline for your application complete with tests and all! Feel free to explore the console to see the details of the stack created, or check out the [API Reference](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-construct-library.html) section on CDK Pipelines and build one for your application.
