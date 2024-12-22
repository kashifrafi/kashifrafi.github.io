---
title: Amazon Bedrock and Key Concept
dates: 2024-10-18
categories: [AWS, Amazon Bedrock, AI]
tags: [AWS, Amazon Bedrock, AI]
---

<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-5684321510124560"
     crossorigin="anonymous"></script>
# Understanding Amazon Bedrock

Amazon Bedrock is a fully managed service that makes high-performing foundation models (FMs) from leading AI companies and Amazon available through a unified API. With Amazon Bedrock, you can choose from a wide range of foundation models to find the one best suited for your use case. It offers robust capabilities to build generative AI applications with a focus on security, privacy, and responsible AI practices. 

Amazon Bedrock provides a serverless experience, enabling you to quickly get started, privately customize foundation models with your own data, and seamlessly integrate and deploy them into your applications using AWS tools—without managing any infrastructure.

---

## **What You Can Do with Amazon Bedrock**

### **1. Experiment with Prompts and Configurations**
Submit prompts and generate responses by experimenting with various configurations and foundation models. Use the API or graphical interfaces like text, image, and chat playgrounds in the AWS console. Once ready, you can set up your application to make requests using the `InvokeModel` API.

---

### **2. Augment Response Generation with Data Sources**
Enhance foundation model responses by creating knowledge bases. Upload your data sources to query and provide context or additional information for the model’s responses.

---

### **3. Build Intelligent Applications**
Create applications capable of reasoning through customer requests. Build agents that utilize foundation models, make API calls, and optionally query knowledge bases to execute tasks efficiently.

---

### **4. Customize Models for Specific Tasks**
Adapt foundation models to specific domains or tasks using training data. Techniques like fine-tuning or continued pretraining can improve the model’s performance for your unique use cases.

---

### **5. Improve Efficiency with Provisioned Throughput**
Boost the efficiency of your foundation model-based applications by purchasing Provisioned Throughput. This allows you to run model inference more cost-effectively and efficiently at scale.

---

### **6. Evaluate and Select the Best Model**
Compare outputs of different models using built-in or custom prompt datasets. This helps determine which model is best suited for your application.

---

### **7. Ensure Content Safety**
Implement guardrails to prevent inappropriate or unwanted content in your generative AI applications.

---

### **8. Optimize Latency**
Enhance response times for your AI applications with latency-optimized inference, ensuring faster and more responsive interactions.

---

## **Key Concept

---

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
