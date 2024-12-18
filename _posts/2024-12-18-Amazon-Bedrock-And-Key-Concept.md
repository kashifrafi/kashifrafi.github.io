---
title: VPC Lattice Setup
dates: 2024-10-12
categories: [AWS, VPC, EKS, VPC Lattice]
tags: [AWS, VPC, EKS, VPC Lattice]
---

# Key Concepts for AI: Understanding Amazon Bedrock

This post explains key terminology to help you understand what Amazon Bedrock offers and how it works. Familiarizing yourself with these concepts will enhance your ability to leverage generative AI and Amazon Bedrock's capabilities.

## **Foundation Model (FM)**
A foundation model is an AI model with a large number of parameters, trained on a massive amount of diverse data. These models can generate responses for a wide range of use cases, including text and image generation, as well as converting inputs into embeddings. Access to foundation models in Amazon Bedrock requires a request.  
_For more details, see: [Supported foundation models in Amazon Bedrock](#)._

---

## **Base Model**
A base model is a foundation model packaged by a provider and ready to use. Amazon Bedrock offers a variety of industry-leading foundation models from top providers.  
_For more details, see: [Supported foundation models in Amazon Bedrock](#)._

---

## **Model Inference**
Model inference is the process where a foundation model generates an output (response) from a given input (prompt).  
_For more details, see: [Submit prompts and generate responses with model inference](#)._

---

## **Prompt**
A prompt is the input provided to a model to guide it in generating a response. Prompts can be simple queries or detailed instructions, enabling tasks such as classification, question answering, code generation, and creative writing.  
_For more details, see: [Prompt engineering concepts](#)._

---

## **Token**
A token represents a sequence of characters interpreted as a single unit of meaning. It can be a word, part of a word, a punctuation mark, or even a phrase.

---

## **Model Parameters**
These are values defining a model's behavior in interpreting inputs and generating responses. Providers update these parameters, and they can be adjusted to create custom models.

---

## **Inference Parameters**
Values that can be tuned during model inference to influence the model's response. They affect the variety, length, and specific patterns of the output.  
_For more details, see: [Influence response generation with inference parameters](#)._

---

## **Playground**
A user-friendly graphical interface in the AWS Management Console to experiment with Amazon Bedrock. The Playground allows you to test different models, configurations, and prompts.  
_For more details, see: [Generate responses in the console using playgrounds](#)._

---

## **Embedding**
The process of converting inputs into numerical vectors for comparing similarities between objects. Examples include text-to-text similarity, visual similarity, or text-to-image relevance.  
_For more details, see: [Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases](#)._

---

## **Orchestration**
The coordination of foundation models, enterprise data, and applications to carry out tasks.  
_For more details, see: [Automate tasks in your application using AI agents](#)._

---

## **Agent**
An application that interprets inputs and produces outputs using a foundation model to handle customer requests.  
_For more details, see: [Automate tasks in your application using AI agents](#)._

---

## **Retrieval Augmented Generation (RAG)**
The process of querying and retrieving data to augment a model's response to a prompt.  
_For more details, see: [Retrieve data and generate AI responses with Amazon Bedrock Knowledge Bases](#)._

---

## **Model Customization**
Adjusting model parameters using training data to create a custom model. Techniques include:
- **Fine-tuning**: Uses labeled data (inputs and outputs).
- **Continued Pre-training**: Uses unlabeled data (inputs only).  
_For more details, see: [Customize your model to improve its performance for your use case](#)._

---

## **Hyperparameters**
Values that control the training process during model customization, affecting the final custom model's output.  
_For more details, see: [Custom model hyperparameters](#)._

---

## **Model Evaluation**
The process of comparing model outputs to select the best-suited model for a use case.  
_For more details, see: [Evaluate the performance of Amazon Bedrock resources](#)._

---

## **Provisioned Throughput**
A purchased level of throughput for a model, allowing increased token processing during inference. Provisioned throughput creates a dedicated model for high-capacity tasks.  
_For more details, see: [Increase model invocation capacity with Provisioned Throughput in Amazon Bedrock](#)._

---

Understanding these key concepts will help you make the most of Amazon Bedrock's generative AI capabilities. Dive deeper into each topic to unleash the full potential of your AI applications!
