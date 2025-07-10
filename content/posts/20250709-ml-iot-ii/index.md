---
title: "Part II - Scenario - Machine Learning for Defect Detection"
description: "Showcasing a prediction scenario"
summary: "Showcasing a prediction scenario"
categories: ["Machine Learning"]
tags: ["Machine Learning", "CDM", "IoT", "Dataset"]
##externalUrl: ""
date: 2025-07-09
draft: false
authors:
  - Roque
---

For this use case we will have a two part blog post. The goal will be to showcase how we can use the machine learning module from Critical Manufacturing to predict material defects.

## Overview

The full use case will be to train a machine learning module with data from a machine and correlate it with an MES material defect that happens in an inspection machine.

![Scenario](https://image.j-roque.com/posts/20250625-sqlite-iii/scenario.png)

We want to be able to predict defects and notify the employees that something was wrong.

![Scenario Final](https://image.j-roque.com/posts/20250625-sqlite-iii/scenario_final.png)

For this second part we will focus on:
- Creating and Training a Machine Learning model
- Simulator tool
- Making Predictions

## Creating and Training a Machine Learning Model

We have already a prepared dataset from the first blog post.

In the MES UI we can now simply create a machine learning model, feed it the dataset and specify the type of model we want.

![Creating an ML Model](https://image.j-roque.com/posts/20250709-ml-iot-ii/creatingmlmodel.gif)

In our use case we want the model to classify if a certain set of machine will generate a defect. So we are using classification and our classifier is the feature classification which will be marked as label.

We can then transform/normalize our data and train our model, specifying what percentages of our dataset will be used to train, validate and test. We can also specify what should our model optimize for, each one will impact how the model behaves.

![Training our model](https://image.j-roque.com/posts/20250709-ml-iot-ii/trainingmodel.gif)

Now our model is ready to predict!

## Simulator Tool

In order to create this scenario we created a simulator tool based on actual machine data. This tool will be an IPC-CFX simulator and will mimic all the events a Reflow Oven does in an SMT shopfloor. This tool will also prepare the MES scenario and will be able to also record defects.

![Simulation](https://image.j-roque.com/posts/20250709-ml-iot-ii/simulation.jpg)

In detail what will happen is that we will have a live instance of the MES system, a live instance of a Connect IoT Automation Manager and then the simulator. The simulator will call the MES API and also interface with the Automation Manager as if it was an IPC-CFX machine. In order to perform this we leverage the IoT Test Orchestrator which is a testing tool provided by Critical Manufacturing (more in-depth explanation [here](https://j-roque.com/posts/20250516-testinglowcode/)).

![Simulation Detail](https://image.j-roque.com/posts/20250709-ml-iot-ii/simulation-detail.png)

For this scenario we know that there is a defect type `Solder Balling` that happens if the temperature of the PreHeat zones of the oven is too high. All simulator will then simulate an SMT line and every X amount of times it will simulate that a panel has been exposed to high temperatures in the PreHeat. Then when it reaches the optical inspection `AOI` it will record a defect.

## Making Predictions

Let's now change our workflow so as to use the `ML Prediction` task to predict if we will have defects.

![ML Prediction Workflow](https://image.j-roque.com/posts/20250709-ml-iot-ii/mlpredictioniot.png)

We send our post telemetry event and then call the ML Prediction task. If the prediction is true, we will create a notification.

![ML Prediction](https://image.j-roque.com/posts/20250709-ml-iot-ii/mlmodelprediction.gif)

We can later check the material to see if it did create a material defect.

![Defect View](https://image.j-roque.com/posts/20250709-ml-iot-ii/materialdefectview.gif)

So we can actually validate that our model prediction was correct.

{{< alert "circle-info" >}}
**Info:** Currently, there is a limitation where the task ML Prediction can only be executed by a automation manager running in the same stack as the MES.
{{< /alert >}}

## Final Thoughts

This is a very simple scenario which is very delimited. We area only looking at temperatures and not to a whole host of other parameters. In a real life scenario we can use the real events of a shopfloor to either train our model or run it in unsupervised mode to perform predictions. In this example we have passive control, by just creating a notification, but we can perform actions like putting the material on hold.

Hopefully this two part blog post gives you the tools so you can try by yourself, adjusting it to your real life use case.