---
layout: post
title:  "Dynamic Parameters in YAML Pipelines"
date:   2023-05-21 11:25:42 -0500
category: Azure DevOps
---
Parameters can be used in an Azure DevOps YAML pipeline in ways that variables cannot. For example, you can use the [each keyword](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/expressions?view=azure-devops#each-keyword) to loop over parameters. The usefulness of this technique is limited because parameter values must be set at an early stage of pipeline execution, before any of the pipeline tasks run. It is not possible to have one task in a pipeline set the value of a parameter to be used in a later task.

For example, I worked on a project where developers would add new Azure Functions apps on a somewhat regular basis. The developers were not able to modify the pipeline YAML themselves. I developed a pipeline that took the list of function app names as a parameter, looped through them, and built and deployed each one. This still required adding the new function app name to the parameter value each time, a step that was sometimes forgotten. While I could add a PowerShell task to the pipeline to find all of the Azure Functions projects within a particular folder and set the list of project names as a variable, I could not find a way to loop over the values in a variable (if you know of a way to do it, let me know!)

There is a workaround to this limitation: use two pipelines. The first pipeline computes the values for the dynamic parameter(s), and then calls the second pipeline, passing in the dynamically generated parameter values.

### Triggering the Second Pipeline
There are a couple of options for triggering one pipeline from another: 
1. call the Azure DevOps API directly. I haven't done this myself.

2. use the [Trigger Build task](https://marketplace.visualstudio.com/items?itemName=benjhuser.tfs-extensions-build-tasks) from the Visual Studio Marketplace.


It might seem easier to use a [pipeline resource](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/resources-pipelines-pipeline?view=azure-pipelines) to trigger the second pipeline, but as far a I can tell, there is no way to pass parameters to the second pipeline this way.

### Example YAML

Here is a simple example of the relevant pieces of the two pipelines. I have a [complete example](https://github.com/mharriger/AzDOExamples/tree/main/DynamicParameters) on GitHub as well.

First pipeline:
{% highlight yaml %}
# Store the parameter value in a variable
- script: value=42; echo "##vso[task.setvariable variable=generatedVariable]$value"
# Trigger the second pipeline, passing the parameter
- task: TriggerBuild@3
    inputs:
      buildDefinition: 'pipeline2'
      templateParameters: 'dynamicParameter:$(generatedVariable)'
{% endhighlight %}

Second pipeline:
{% highlight yaml %}
parameters:
  - name: dynamicParameter
    type: number

steps:{% raw %}
  - script: echo ${{ parameters.dynamicParameter }}{% endraw %}
{% endhighlight %}

### Alternate Approach
Although you can't loop over a variable in a pipeline using the each keyword, it is possible to loop over a variable in a limited way using the [matrix strategy](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/jobs-job-strategy?view=azure-pipelines#strategy-matrix-maxparallel) on a job. This approach does require you to repeat every step in a job, and is not support for [deployment jobs](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs?view=azure-devops), which is what prevented me from using it in the situation mentioned above.