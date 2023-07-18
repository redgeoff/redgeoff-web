---
title: "Running MosaicML's MPT-7B, a ChatGPT Competitor, on AWS SageMaker"
date: 2023-07-18T08:48:01-07:00
draft: false
description: "In this blog post, I’m going to take you step-by-step through the process of running MosaicML’s ChatGPT competitor, MPT-7B, on your own AWS SageMaker instance."
images:
  - /posts/mpt-7b-on-aws-sagemaker/mpt-7b-sagemaker.png
categories:
  - Programming
tags:
  - chatgpt
  - aws
  - mosaicml
  - mpt-7b
  - mpt-30b
  - openai
  - ai
  - gpt4
  - llm
---

In this blog post, I’m going to take you step-by-step through the process of running MosaicML’s ChatGPT competitor, MPT-7B, on your own AWS SageMaker instance.

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/mpt-7b-sagemaker.png" alt="MPT-7B and AWS SageMaker" align="center" attr="MPT-7B and AWS SageMaker. Image credit: [MosaicML](https://www.mosaicml.com/blog/mpt-30b) and Author" >}}

Are you excited about the capabilities of ChatGPT, but have concerns about exposing your sensitive data to OpenAI? Luckily, there are alternatives that you can run on your own infrastructure. One such alternative is MosaicML's MPT-7b, a competitor to ChatGPT, which we'll explore in this blog post.

### Introduction to MosaicML and MPT-7B

MosaicML, recently [acquired by Databricks for $1.3 billion](https://www.reuters.com/markets/deals/databricks-strikes-13-bln-deal-generative-ai-startup-mosaicml-2023-06-26), has been making waves in the ML community with their MPT-7B model, a supposed competitor to ChatGPT. Despite its promise, running this model can be daunting due to sparse documentation and its heavy resource requirements. However, one can run MPT-7B on AWS SageMaker in a Jupyter notebook, an environment that is beginner-friendly and highly flexible for rapid iteration. This setup allows you to test the model’s feasibility and hardware requirements before deciding to move into production.

### Running MPT-7B on AWS SageMaker

Running MPT-7B in a Jupyter notebook on AWS SageMaker provides several benefits. Not only can you pay just for what you use and turn it off when you're done, but the ability to easily rerun portions of your code without having to reload the model saves time during iterative development. But do beware! If you forget to stop your notebook instances, the charges can quickly add up. 

While this method is relatively convenient, there are some considerations you must take into account. Firstly, loading the model can take up to 20 minutes even on a high-performance GPU, making this process somewhat time-consuming. Also, the cost is a factor to consider, as the running cost is at least $4 per hour. You'll need to run MPT-7B on at least a _p3.2xlarge_ instance; anything smaller does not seem feasible. If you opt for EC2 instead of SageMaker, you'll have to ask AWS for permission to use a _p3.2xlarge_ instance.

In the next sections I’ll take you step-by-step through how to run the MPT-7B model in your very own SageMaker jupyter notebook:

#### Step 1 - Open the SageMaker Console

Fire up the AWS Console and search for SageMaker:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/sage-maker.png" alt="Search for SageMaker" align="center" attr="Search for SageMaker. Image credit: Author" >}}

#### Step 2 - Create a notebook instance

From the left-side menu, select _Notebook->Notebook instances_:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/notebook-instances.png" alt="Notebook instances" align="center" attr="Notebook instances. Image credit: Author" >}}

Click the _Create notebook instance_ button:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/create-notebook-instance.png" alt="Create notebook instance" align="center" attr="Create notebook instance. Image credit: Author" >}}

Specify an instance name. Choose the instance type _m1.p3.2xlarge_. Unfortunately, it seems that an instance as powerful as _m1.p3.2xlarge_ is required, or else your instance may run out of memory or take an excessive amount of time to respond to even the simplest questions. However, please note that this instance will cost approximately $4/hr, so it's important to monitor your usage carefully.

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/create-instance-2.png" alt="Specify notebook instance details" align="center" attr="Specify notebook instance details. Image credit: Author" >}}

Create a new IAM role:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/create-role.png" alt="Create a new role" align="center" attr="Create a new role. Image credit: Author" >}}

If your test environment doesn’t have any particularly sensitive data in it then you can grant access to _Any S3 bucket_. Otherwise, you’ll need to be more explicit.

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/create-role-s3.png" alt="Specify S3 access" align="center" attr="Specify S3 access. Image credit: Author" >}}

Click the _Create notebook instance_ button:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/create-instance-3.png" alt="Create notebook instance" align="center" attr="Create notebook instance. Image credit: Author" >}}

The notebook will then be in a _Pending_ status. This will likely last for about 10 minutes:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/notebook-pending.png" alt="Pending notebook instance" align="center" attr="Pending notebook instance. Image credit: Author" >}}

In the meantime, we’ll download a notebook so that we can upload it after the AWS SageMaker instance has finished provisioning.

#### Step 3 - Download the notebook

Head on over to the notebook at [MPT-7B on AWS SageMaker.ipynb](https://colab.research.google.com/drive/1kJr2LHHLKYkbnNutVYEkt2vrYsbO38aw) and download it:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/colab-notebook.png" alt="The notebook on Google Colab" align="center" attr="The notebook on Google Colab. Image credit: Author" >}}

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/download-colab-notebook.png" alt="Download the notebook" align="center" attr="Download the notebook. Image credit: Author" >}}

In this notebook you’ll notice two main code blocks. The first block loads the MPT-7B tokenizer and model:

```python
from torch import cuda, bfloat16
from transformers import AutoTokenizer, AutoModelForCausalLM, AutoConfig

device = f'cuda:{cuda.current_device()}' if cuda.is_available() else 'cpu'

tokenizer = AutoTokenizer.from_pretrained("mosaicml/mpt-7b-chat",
                                          trust_remote_code=True)

config={"init_device": "meta"}
model = AutoModelForCausalLM.from_pretrained("mosaicml/mpt-7b-chat",
                                             trust_remote_code=True,
                                             config=config,
                                             torch_dtype=bfloat16)

print(f"device={device}")
print('model loaded')
```

The tokenizer is used to encode the question sent to the model and decode the response from the model. Additionally, we obtain the device specification for our GPU so that we can configure the model to utilize it later:

```python
import time
from IPython.display import Markdown
import torch
from transformers import StoppingCriteria, StoppingCriteriaList

# mtp-7b is trained to add "<|endoftext|>" at the end of generations
stop_token_ids = [tokenizer.eos_token_id]

# Define custom stopping criteria object.
# Source: https://github.com/pinecone-io/examples/blob/master/generation/llm-field-guide/mpt-7b/mpt-7b-huggingface-langchain.ipynb
class StopOnTokens(StoppingCriteria):
def __call__(self, input_ids: torch.LongTensor,scores: torch.FloatTensor,
             **kwargs) -> bool:
  for stop_id in stop_token_ids:
    if input_ids[0][-1] == stop_id:
      return True
    return False

stopping_criteria = StoppingCriteriaList([StopOnTokens()])

def ask_question(question, max_length=100):
  start_time = time.time()

  # Encode the question
  input_ids = tokenizer.encode(question, return_tensors='pt')

  # Use the GPU
  input_ids = input_ids.to(device)

  # Generate a response
  output = model.generate(
    input_ids,
    max_new_tokens=max_length,
    temperature=0.9,
    stopping_criteria=stopping_criteria
  )

  # Decode the response
  response = tokenizer.decode(output[:, input_ids.shape[-1]:][0],
                              skip_special_tokens=True)

  end_time = time.time()
  duration = end_time - start_time

  display(Markdown(response))

  print("Function duration:", duration, "seconds")
```

Note the use of `stopping_critera`, which is needed or else the model will just start babbling on, even after it has answered our question.

See [model generate parameters](https://huggingface.co/transformers/v4.12.5/main_classes/model.html#transformers.generation_utils.GenerationMixin.generate) if you want to explore the different options.

Now, let’s upload this notebook to SageMaker.

#### Step 4 - Upload the notebook

Hopefully by this time your SageMaker notebook instance has finished being provisioned. When it has, click the _Open Jupyter_ link:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/created-notebook-instances.png" alt="Open Jupyter" align="center" attr="Open Jupyter. Image credit: Author" >}}

Then, click the _Upload_ button in the top-right corner of your screen and select the notebook that you just downloaded:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/upload-notebook.png" alt="Upload the notebook" align="center" attr="Upload the notebook. Image credit: Author" >}}

Set the kernel to _conda_python3_:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/set-kernel.png" alt="Set the kernel" align="center" attr="Set the kernel. Image credit: Author" >}}

#### Step 5 - Run the notebook

Select _Cell -> Run All_:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/run-all.png" alt="Run all cells" align="center" attr="Run all cells. Image credit: Author" >}}

An hourglass logo will then appear in the browser tab:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/wait-for-notebook.png" alt="Wait for notebook" align="center" attr="Wait for notebook. Image credit: Author" >}}

You’ll then need to wait about 10 minutes for the model to be downloaded:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/notebook-downloads-model.png" alt="Download model" align="center" attr="Download model. Image credit: Author" >}}

After it runs, you’ll see the answer to the question _Explain to me the difference between nuclear fission and fusion_:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/fission-vs-fusion.png" alt="Explain to me the difference between nuclear fission and fusion" align="center" attr="Explain to me the difference between nuclear fission and fusion. Image credit: Author" >}}

Since the model and tokenizer have already been loaded above, you can simply modify the _ask_question_ code block and click the _Run_ button to ask any other questions. This will save you from spending 10 minutes each time you want to test a new question.

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/capital-of-france.png" alt="What is the capital of France?" align="center" attr="What is the capital of France?. Image credit: Author" >}}

#### Step 6 - Stop the notebook

As soon as you have finished testing the model, you’ll want to head back to your list of notebook instances and stop it. If you don’t, $4/hr will add up very quickly 💸

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/stop-notebook.png" alt="Stop the notebook" align="center" attr="Stop the notebook. Image credit: Author" >}}

### Performance Comparison

In terms of performance, my preliminary tests suggest that MPT-7B's results might not be as good as ChatGPT’s. It does a decent job answering questions like _What is the capital of France?_, _Explain to me the difference between nuclear fission and fusion_, and _Write python code that converts a csv into pdf_. But for questions like_What is the capital of Belize?_ it fails pretty horribly:

{{< figure src="/posts/mpt-7b-on-aws-sagemaker/capital-of-belize.png" alt="What is the capital of Belize?" align="center" attr="What is the capital of Belize? Image credit: Author" >}}

I am currently collecting more data and will conduct a comprehensive comparative analysis in a follow-up blog post. In that post, I will compare the question and answer performance of MPT-7B, MPT-30B, Falcon-40b, and ChatGPT using actual conversation history.

### From Testing to Production

Once you're ready to transition from testing to production, SageMaker offers an additional benefit - the ability to create an endpoint for your model. With SageMaker, you can auto-scale based on demand to the endpoint, optimizing your resources.

### Additional Tips

Be mindful that it's easy for your process to get forked while running in a Jupyter notebook and run out of memory. If this happens, simply shut down the kernel and run all the commands again.

If you're curious about running this model on a platform other than AWS, Google Colab Pro is another viable option at $9/month. However, based on our testing, we found that we exhausted the provided credits within just a few hours. 😳

Another challenge you may face is the inability to utilize the [Triton optimization](https://huggingface.co/mosaicml/mpt-7b#how-to-use) on SageMaker due to a CUDA version incompatibility. Unfortunately, AWS's current P3 instances do not include a recent CUDA version. Therefore, if you wish to utilize the Triton optimization, you will need to create an EC2 container with command line access. However, it's important to note that you will also require special permission from AWS Support to run an instance with 8 VCPUs. In a future post, I will provide a detailed guide on how to integrate Triton and utilize a more cost-effective GPU cloud provider, such as [Lambda Labs](lambdalabs.com).

### Final Thoughts

While MosaicML's MPT-7B offers a viable alternative to OpenAI's ChatGPT, it presents its own set of challenges. Running the model can be time-consuming, expensive, and the available documentation is lacking. However, the ability to keep the model in-house and protect your data from being exposed to OpenAI can be compelling for certain use cases.

SageMaker offers great convenience for quickly testing the model and provides the flexibility to transition to production when you're ready. Whether you're just starting with MPT-7B or have been using it for a while, we hope this guide has provided valuable insights.

Stay tuned for our next blog post, where we'll delve deeper into the performance comparisons between MPT-7B, MPT-30B, Falcon-40b, and ChatGPT.

See the following links if you are interested in learning more about [MPT-7B](https://www.mosaicml.com/blog/mpt-7b) or its larger variant, [MPT-30B](https://www.mosaicml.com/blog/mpt-30b).

And remember, whether you're working with ChatGPT or MPT-7B, the key is to ensure your use case is served without compromising data privacy and cost-effectiveness. Happy tinkering!

### Want a Turn-Key Solution for Using MPT or ChatGPT with Your Data?

At [MindfulDataAI.com](https://mindfuldataai.com?utm_source=redgeoff_com&utm_campaign=mpt-7b), we offer ChatGPT for businesses. If you're interested in leveraging ChatGPT, MPT, or other models with your company's data, please get in touch with us.
