---
layout: post
title:  "Creating AWS Lambda Container Image with custom Golang base image"
date:   2021-01-16 12:22:42 +0800
categories: aws lambda containers golang
---

AWS supports creating AWS Lambda function using containers. While the [AWS documentation][aws-doc] does pretty good of explaining how to build Lambda container image using [AWS base image for Go][aws-go-image-doc], it is a bit confusing for images built using custom/alternative Golang base image. So, in this post, we will discuss how to create Golang container image for AWS Lambda and test it on local machine.


First of all, Create a Golang app and implement the Handler function as per your business logic. Here is the sample app code taken from [ this AWS doc][aws-lambda-golang-function]. Please refer doc for supported Handler types.
{% highlight golang %}
package main

import (
    "fmt"
    "context"
    "github.com/aws/aws-lambda-go/lambda"
)

type MyEvent struct {
    Name string `json:"name"`
}

func HandleRequest(ctx context.Context, name MyEvent) (string, error) {
    return fmt.Sprintf("Hello %s!", name.Name ), nil
}

func main() {
    lambda.Start(HandleRequest)
}
{% endhighlight %}

Secondly, write a Dockerfile to specify how to build the container image. For example:
{% highlight dockerfile %}
FROM golang:1.15

# include AWS RIE
ADD https://github.com/aws/aws-lambda-runtime-interface-emulator/releases/latest/download/aws-lambda-rie /usr/bin/aws-lambda-rie
RUN chmod 755 /usr/bin/aws-lambda-rie
COPY entry.sh /
RUN chmod 755 /entry.sh

# Add your app dependencies here and specify how to build your app binary. e.g.
# WORKDIR /usr/src/app
# COPY main.go .
# go build -o main main.go

ENTRYPOINT [ "/entry.sh"]
CMD [<your-binary-image>]
# CMD ["/usr/src/app/main"]
{% endhighlight %}


Now, build, run and test your Lambda function locally as follows:

{% highlight bash %}
$ docker build -t <image-name> .

$ docker run -p 9000:8080 <image-name>

# In a separate terminal, you can then locally invoke the function using cURL:
$ curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{"payload":"hello world!"}'
{% endhighlight %}

PS: The container exposes endpoint `/2015-03-31/functions/function/invocations` on port `8080`. AFAIK we can not modify this setting.

After testing the image locally, upload the container image to your AWS ECR repo. Now you can create a Lambda function using the uploaded containter image.

Thanks!

[aws-doc]: https://docs.aws.amazon.com/lambda/latest/dg/go-image.html 
[aws-go-image-doc]: https://docs.aws.amazon.com/lambda/latest/dg/go-image.html#go-image-base
[aws-lambda-golang-function]: https://docs.aws.amazon.com/lambda/latest/dg/golang-handler.html 