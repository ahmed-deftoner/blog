+++ 
draft = false
date = 2024-05-02T12:53:37+05:00
title = "Cronjobs using Helm Charts"
description = "cron jobs using helm charts"
slug = "cronjobs-using-helm-charts"
authors = ["Ahmed Nadeem"]
tags = ["cronjobs", "kubernetes", "devops"]
categories = []
+++

Imagine you're debugging your application, and you've defined your cronjobs within your code - you are relying on a library/package to schedule your jobs - and then it runs locally without you realizing and your co-workers are left wondering why certain actions got triggered so quickly. In some instances this might not be a huge issue, but the project I worked on had cron jobs to track payments and update their states, hence it wasn't a good idea to have a library schedule our cron jobs. Moreover, most libraries also don't provide concurrency control, which could lead to race conditions. Let's see some code (I am using NestJS for scheduling cron jobs) 

```Typescript
@Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT, { name: 'test' })
async test() {
    this.logger.log("cron job started");
}
```

Also add the scheduler package in the app.module.ts

```Typescript
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

Underneath, the nestjs scheduler uses the NodeJS cron package to schedule tasks. This configuration is fine, but if the cron job is scheduled at lower time intervals, it becomes an issue while debugging your app, especially when your app is a monolith. You're testing an endpoint, and the cron job starts and makes some crucial state changes which you did not intend. Also there is the hassle of continuously changing the cron expression to lower intervals to test it as well.

Generally, it is a good practice to manage your jobs on the infrastructure level, this could mean, for example, using an AWS Lambda function for your cron job and using CloudWatch Events or AWS EventBridge to trigger the cron job, or if you're using Kubernetes, you could use native K8s cron jobs, or you could use certain SaaS services like Inngest or Trigger.dev that invoke your jobs on the cloud. These services solve many common problems faced with cron jobs, like concurrency control, third party app integrations, retry capability etc.

Now that we've laid the foundation of why we want to manage our jobs on the infrastructure level instead of using scheduling libraries, let's start. Currently, the app is deployed as a Kubernetes pod, so we'll use native Kubernetes cron jobs. There are two approaches, either we can invoke our functions using npm commands, and then run then provide the script as an argument to the cron job, or we could create a controller, and hit the endpoint using curl.

I prefer the second appraoch, since littering your package.json with tons of npm commands isn't ideal. There are drawbacks with the curl approach like having your cronjobs being exposed as endpoints to the outisde world is a potential security vulnerability. However, this could be mitigated with additional configuration or by just implementing some basic auth, depending on the severity of the vulnerability. Let's create a template for the cron job using Helm charts:

```Yaml
{{- range .Values.cronjobs }}
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ .name }}
spec:
  schedule: {{ .schedule | quote }}
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: {{ .name }}
            image: {{ .image }}
            command: ["{{ .command }}"]
            args: {{ .args | toJson }}
          restartPolicy: OnFailure
---
{{- end }}
```

Helm Charts aren't just useful as package managers, but they can also improve code reusability. Since you won't have to create separate manifest files for each cron job. You could just create a template like the one above, and plug in the values in the values.yaml file.

Let's create a controller for our test function:

```Typescript
@Controller('cronjob')
export class CronjobController {
    constructor(
        private readonly cronjobService: CronJobsService,
    ) { }

    @Get('check')
    async Test() {
        this.cronjobService.test();
        return { message: "Cron started" };
    }
}
```

Now, we'll be hitting this endpoint, lets create a values.yaml file which will use the cron job template:

```Yaml
cronjobs:
  - name: test
    successfulJobsHistoryLimit: 2
    failedJobsHistoryLimit: 1
    schedule: "*/5 * * * *"
    image: "{my_image}"
    command: "curl"
    args: ["{my_app_url}/cronjob/check"]
```

This cron job will hit this endpoint every 5 minutes, we can append more cron jobs to this list and reuse the same template. I've added some additional parameters, the successfulJobsHistoryLimit allows us to retain a certain number of successful jobs and we can view them using kubectl. Furthermore, we can also specify how our cron job behaves when there are concurrent instances of the cron jobs running. We can specify a concrrency Policy, which can be "Allow", "Forbid", or "Replace".

Finally, since for my project we have separate environments, we can specify separate URL's or configurations by having environment specific values.yaml files. So my values-dev.yaml file looks like this:

```Yaml
cronjobs:
  - name: test
    successfulJobsHistoryLimit: 2
    failedJobsHistoryLimit: 1
    schedule: "*/1 * * * *"
    image: "{my_image}"
    command: "curl"
    args: ["{my_dev_app_url}/cronjob/check"]
```
