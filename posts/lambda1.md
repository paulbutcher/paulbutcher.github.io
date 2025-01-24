Title: Quick and Easy Clojure on AWS Lambda in 2025
Date: 2025-01-23
Tags: clojure

I recently found myself starting a new project and was looking for the quickest and easiest way to get something up and running. In the past I might have used [Heroku](https://www.heroku.com), and I looked briefly at [Fly.io](https://fly.io), but it turns out that Clojure now runs much better on Lambda than it used to (cold starts are no longer an issue), and it's easy to get up and running with [AWS SAM](https://aws.amazon.com/serverless/sam/) which gives us simple serverless Infrastructure as Code.

This article describes how to get a simple Clojure Ring application running. A [followup article](lambda2.html) shows how to connect it to a database. The code is available  [here](https://github.com/paulbutcher/example-lambda-app).

## The Code

Let's start with a simple Clojure Ring app with a sprinkling of interactivity through [HTMX](https://htmx.org):

```clojure
(ns example.lambda-app
  (:require [compojure.core :refer [defroutes GET]]
            [compojure.route :as route]
            [hiccup2.core :refer [html]]
            [ring.logger :refer [wrap-with-logger]]
            [ring.middleware.defaults :refer [site-defaults wrap-defaults]]
            [ring.middleware.params :refer [wrap-params]]))

(defn index-page
  []
  (str (html [:head [:title "HTMX Example"]
              [:script {:src "https://unpkg.com/htmx.org@2.0.4"}]]
             [:body [:h1 "HTMX Example"]
              [:div#greeting {:hx-get "/greet" :hx-trigger "load"}]])))

(defn greet [] (str (html [:div "Hello, World!"])))

(defroutes app-routes
  (GET "/" [] (index-page))
  (GET "/greet" [] (greet))
  (route/not-found "Not Found"))

(def app
  (-> app-routes
      wrap-params
      (wrap-defaults site-defaults)
      wrap-with-logger))
```

This is all entirely standard: nothing different from any other Ring app. We're going to implement _two_ different ways to serve this app, one using `ring-jetty-adapter` for local development, and one using `ring-lambda-adapter` for deployment to AWS Lambda.

For local development, we'll create `dev/user.clj`:

```clojure
(ns user
  (:require [ring.adapter.jetty :as jetty]
            [example.lambda-app :refer [app]]))

(defn -main [& _]
  (jetty/run-jetty app {:port 8080 :host "0.0.0.0" :join? false}))
```

And for deployment to AWS Lambda, we'll create `lambda.clj`:

```clojure
(ns example.lambda
  (:gen-class :implements
              [com.amazonaws.services.lambda.runtime.RequestStreamHandler])
  (:require [paulbutcher.ring-lambda-adapter :refer [handle-request]]
            [example.lambda-app :refer [app]]))

(defn -handleRequest [_ is os _] (handle-request app is os))
```

This implements the `RequestStreamHandler` interface defined by the AWS Java Lambda runtime. The `handle-request` function is provided by `ring-lambda-adapter` and does the work of converting the input and output streams to and from Ring requests and responses.

Finally, here's our `deps.edn`:

```clojure
{:paths ["src" "resources"]
 :deps {com.amazonaws/aws-lambda-java-core {:mvn/version "1.2.3"}
        com.amazonaws/aws-xray-recorder-sdk-slf4j {:mvn/version "2.18.2"}
        com.paulbutcher/ring-lambda-adapter {:mvn/version "1.0.7"}
        compojure/compojure {:mvn/version "1.7.1"}
        hiccup/hiccup {:mvn/version "2.0.0-RC4"}
        org.clojure/clojure {:mvn/version "1.12.0"}
        ring-logger/ring-logger {:mvn/version "1.1.1"}
        ring/ring-core {:mvn/version "1.13.0"}
        ring/ring-defaults {:mvn/version "0.5.0"}}
 :aliases {:run {:main-opts ["-m" "user"]}
           :build {:deps {io.github.clojure/tools.build {:mvn/version "0.10.6"}}
                   :ns-default build}
           :dev {:extra-paths ["dev"]
                 :extra-deps {org.slf4j/slf4j-simple {:mvn/version "2.0.16"}
                              ring/ring-jetty-adapter {:mvn/version "1.13.0"}}}}}
```

This is all very standard apart from:

* `aws-lambda-java-core` which provides the AWS Java Lambda runtime.
* `aws-xray-recorder-sdk-slf4j` which turns SLF4J logs into AWS X-Ray traces.
* `ring-lambda-adapter` which provides a Ring adapter for AWS Lambda.

You should now be able to run locally with `clojure -M:dev:run`.

## Deployment

To deploy to AWS, you'll need to have the [AWS SAM CLI installed](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html). This sits on top of AWS CloudFormation and simplifies the process of deploying serverless applications. Our application is described via a `template.yaml` file:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31

Resources:
  Function:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: target/standalone.jar
      Handler: example.lambda::handleRequest
      Runtime: java21
      FunctionUrlConfig:
        AuthType: NONE
      AutoPublishAlias: live
      SnapStart:
        ApplyOn: PublishedVersions
      Timeout: 20
      MemorySize: 512
      Tracing: Active
    Metadata:
      SkipBuild: true

Outputs:
  Endpoint:
    Value: !GetAtt FunctionUrl.FunctionUrl
```

* `CodeUri` points to an uberjar file we're going to build (the `SkipBuild` metadata tells SAM not to try to build the jar file itself).
* `Handler` is our Lambda function entry point.
* `AuthType` is `NONE` because this is a public web app (not an API that's sitting behind some kind of authentication).
* We're only using a single alias (`live`), but we could have multiple aliases for different environments (e.g. staging etc.).
* [`SnapStart`](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) takes a snapshot of our function which reduces cold-start times to less than a second.
* `Tracing` enables AWS X-Ray tracing.

To deploy, first build the jar file with `clojure -T:build uberjar` (see [`build.clj` in GitHub](https://github.com/paulbutcher/example-lambda-app/blob/main/build.clj)), then deploy with `sam deploy --guided`. This will ask you some questions such as which region you want to deploy to and then eventually output the URL of your function's endpoint. Connect to that URL, and you should see exactly what you saw when you ran locally.

## Monitoring

There are a number of different ways to keep an eye on how your Lambda function is performing.

* You can watch logs locally in real time with `sam logs --tail`.
  * You'll need to specify the region and stack name you already specified when deploying. You can avoid having to do so every time by adding a `[default.global.parameters]` section to your `samconfig.toml` file.
* From the AWS console, you can see logs in CloudWatch. Most interestingly, you can use X-Ray traces to see detailed timing information for each request.

## Development

Local development works just like any other Ring app. To deploy a new version, just build a new jar file and run `sam deploy`.

## Conclusion

The combination of SAM, which makes Lambda function deployment so simple, and SnapStart, which removes the cold start problem, means that AWS Lambda is my new default for getting started quickly.

In the [next article](lambda2.html), we'll look at how to connect our Lambda function to a database.

## Credits

This was all heavily inspired by [A Recipe for Plain Clojure Lambdas](https://www.juxt.pro/blog/plain-clojure-lambda).
