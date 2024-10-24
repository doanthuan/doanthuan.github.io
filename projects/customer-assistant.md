---
layout: page
title: 
permalink: /projects/customer-assistant/
---

<h3>
Project Information
</h3>

**Date**: 2022

**Overview:**
This project focuses on the development of an integrated system for call center customer support using a two-pronged approach: voice recognition for analyzing audio calls and a rule-based chatbot for automated customer service interactions. The goal is to streamline customer support operations by automating tasks, improving response accuracy, and reducing wait times, all while enhancing the overall customer experience.


![Image](/assets/customer-assistant.png)

**Key Components:**
1.	Voice Recognition Module:
	- The voice recognition system processes and analyzes audio from customer calls in real-time.
	- Using state-of-the-art speech-to-text technology, the module transcribes spoken language into text, identifies key customer inquiries, and flags potential issues that require further human attention.
	- The system also tracks key metrics like call duration, sentiment analysis, and intent recognition to improve customer service outcomes and optimize agent performance.
2.	Rule-Based Chatbot (RASA):
	- A RASA-based chatbot is implemented to handle common customer inquiries through predefined rules and decision trees.
	- The chatbot interacts with customers via a text interface, providing instant responses to frequently asked questions, offering troubleshooting advice, and guiding users through standard procedures.
	- The chatbot can escalate complex issues to a human agent when necessary and continuously improves its decision-making ability through feedback loops and rule updates.

**Impact:**
The system will allow call centers to improve operational efficiency by automating routine tasks, ensuring faster response times, and reducing the workload on human agents. Through the integration of voice recognition and chatbot technologies, the customer support experience will be enhanced, leading to higher satisfaction and better resource utilization.

**Technologies:** 
 - Python, Transformers, Kaldi, wav2vec, VAD, PhoBERT, vncorenlp, RASA, Trankit


## Call Center Voice Recognition
### Speech To Text with Kaldi and Lookahead
- [https://kaldi-asr.org/](https://kaldi-asr.org/)
- [https://alphacephei.com/nsh/2020/03/27/lookahead.html](https://alphacephei.com/nsh/2020/03/27/lookahead.html)
![Image](/assets/speech_model_classic.png)


### Speech To Text with wav2vec
#### 1. Train a self-supervised model on unlabeled data (Pretrain)

- 1.1 Prepare unlabeled audios

    Collect unlabel audios and put them all together in a single directory. Audio format requirements:\
    Format: wav, PCM 16 bit, single channel\
    Sampling_rate: 16000\
    Length: 5 to 30 seconds\
    Content: silence should be removed from the audio. Also, each audio should contain only one person speaking.\
    Please look at examples/unlabel_audio directory for reference.

- 1.2 Download an initial model
    Instead of training from scratch, we download and use english wav2vec model for weight initialization. This pratice can be apply to all languages.
    ```
    wget https://dl.fbaipublicfiles.com/fairseq/wav2vec/wav2vec_small.pt
    ```

- 1.3 Run Pre-training
    ```
    python3 pretrain.py --fairseq_path path/to/libs/fairseq --audio_path path/to/audio_directory --init_model path/to/wav2vec_small.pt
    ```
    Where:
    - fairseq_path: path to installed fairseq library, after install [instruction](https://github.com/mailong25/self-supervised-speech-recognition/blob/master/Dependencies.md)
    - audio_path: path to unlabel audio directory
    - init_model: downloaded model from step 1.2

    Logs and checkpoints will be stored at outputs directory\
    Log_file path: outputs/date_time/exp_id/hydra_train.log.  You should check the loss value to decide when to stop the training process.\
    Best_checkpoint path: outputs/date_time/exp_id/checkpoints/checkpoint_best.pt\
    In my casse, it took ~ 4 days for the model to converge, train on 100 hours of data using 2 NVIDIA Tesla V100.

#### 2. Finetune the self-supervised model on the labeled data

- 2.1 Prepare labeled data
    -- Transcript file ---\
    One trainng sample per line with format "audio_absolute_path \tab transcript"\
    Example of a transcript file:
    ```
    /path/to/1.wav AND IT WAS A MATTER OF COURSE THAT IN THE MIDDLE AGES WHEN THE CRAFTSMEN
    /path/to/2.wav AND WAS IN FACT THE KIND OF LETTER USED IN THE MANY SPLENDID MISSALS PSALTERS PRODUCED BY PRINTING IN THE FIFTEENTH CENTURY
    /path/to/3.wav JOHN OF SPIRES AND HIS BROTHER VINDELIN FOLLOWED BY NICHOLAS JENSON BEGAN TO PRINT IN THAT CITY
    /path/to/4.wav BEING THIN TOUGH AND OPAQUE
    ```
    Some notes on transcript file:
    - One sample per line
    - Upper case
    - All numbers should be transformed into verbal form.
    - All special characters (eg. punctuation) should be removed. The final text should contain words only
    - Words in a sentence must be separated by whitespace character


    -- Labeled audio file ---\
    Format: wav, PCM 16 bit, single channel, Sampling_rate: 16000.\
    Silence should be removed from the audio.\
    Also, each audio should contain only one person speaking.\

- 2.2 Generate dictionary file
    ```
    python3 gen_dict.py --transcript_file path/to/transcript.txt --save_dir path/to/save_dir
    ```
    The dictionary file will be stored at save_dir/dict.ltr.txt. Use the file for fine-tuning and inference.

- 2.3 Run Fine-tuning on the pretrain model
    ```
    python3 finetune.py --transcript_file path/to/transcript.txt --pretrain_model path/to/pretrain_checkpoint_best.pt --dict_file path/to/dict.ltr.txt
    ```
    Where:
    - transcript_file: path to transcript file from step 2.1
    - pretrain_model: path to best model checkpoint from step 1.3
    - dict_file: dictionary file generated from step 2.2

    Logs and checkpoints will be stored at outputs directory\
    Log_file path: outputs/date_time/exp_id/hydra_train.log. You should check the loss value to decide when to stop the training process.\
    Best_checkpoint path: outputs/date_time/exp_id/checkpoints/checkpoint_best.pt\
    In my casse, it took ~ 12 hours for the model to converge, train on 100 hours of data using 2 NVIDIA Tesla V100.

#### 3. Train a language model
- 3.1 Prepare text corpus
    Collect all texts and put them all together in a single file. \
    Text file format:
    - One sentence per line
    - Upper case
    - All numbers should be transformed into verbal form.
    - All special characters (eg. punctuation) should be removed. The final text should contain words only
    - Words in a sentence must be separated by whitespace character

    Example of a text corpus file for English case:
    ```
    AND IT WAS A MATTER OF COURSE THAT IN THE MIDDLE AGES WHEN THE CRAFTSMEN
    AND WAS IN FACT THE KIND OF LETTER USED IN THE MANY SPLENDID MISSALS PSALTERS PRODUCED BY PRINTING IN THE FIFTEENTH CENTURY
    JOHN OF SPIRES AND HIS BROTHER VINDELIN FOLLOWED BY NICHOLAS JENSON BEGAN TO PRINT IN THAT CITY
    BEING THIN TOUGH AND OPAQUE
    ...
    ```


- 3.2 Train the language model
    ```
    python3 train_lm.py --kenlm_path path/to/libs/kenlm --transcript_file path/to/transcript.txt --additional_file path/to/text_corpus.txt --ngram 3 --output_path ./lm
    ```
    Where:
    - kenlm_path: path to installed kenlm library, after install [instruction](https://github.com/mailong25/self-supervised-speech-recognition/blob/master/Dependencies.md)
    - transcript_file: path to transcript file from step 2.1
    - additional_file: path to text corpus file from step 3.1

    The LM model and the lexicon file will be stored at output_path


**Pre-trained models (Pretrain + Fine-tune + LM)**
- [Vietnamese](https://drive.google.com/file/d/1kZFdvMQt-R7fVebTbfWMk8Op7I9d24so/view?usp=sharing)

**Reference:**
    Paper: wav2vec 2.0: A Framework for Self-Supervised Learning of Speech Representations: https://arxiv.org/abs/2006.11477 \
    Source code: https://github.com/pytorch/fairseq/blob/master/examples/wav2vec/README.mdYoutube/Podcast is a great place to collect the data for your own language


### Sentiment Analysis and Name Entity Recognition
- Base model: PhoBert
- Steps:
    - Standardize text data
    - Word segmentation with VnCoreNLP or Underthesea
    - Bert tokenizer
    - Padding
    - Extract features from Bert
    - SVM or BertForSequenceClassification


## Customer Support Chatbot

### RASA

This project aims to develop an intelligent customer support chatbot using the RASA framework, a robust open-source platform for building conversational AI. The chatbot will be designed to provide automated customer assistance, reducing the need for human intervention in handling frequent and repetitive queries while enhancing customer experience through prompt responses, contextual understanding, and seamless integration with customer support systems.

**Objectives**

- **Develop a Conversational Agent**: Build a chatbot capable of understanding and responding to customer queries across a wide range of topics, leveraging RASA’s natural language understanding (NLU) and dialogue management capabilities.
- **Multi-channel Support**: Integrate the chatbot with multiple communication channels such as websites, messaging apps (e.g., WhatsApp, Facebook Messenger), and live chat platforms.
- **Contextual and Personalized Interactions**: Enhance the chatbot's ability to handle contextual conversations and provide personalized responses based on user history and preferences.
- **FAQ Automation**: Automate responses to Frequently Asked Questions (FAQs) to reduce customer service response times and improve operational efficiency.
- **Human Hand-off Integration**: Implement smooth transitions to human agents when the chatbot is unable to handle complex queries or when requested by the customer.
- **Data Analytics and Insights**: Incorporate analytics tools to track chatbot performance, customer satisfaction, and key metrics such as response time, issue resolution rate, and interaction quality.

### Technical Approach

**1. Intent and Entity Recognition**
- Use RASA’s NLU component to train models that accurately classify customer intents and extract relevant entities from user input. This will enable the chatbot to understand a wide variety of customer queries.
- Example intents: _Check order status_, _Refund request_, _Product inquiry_.

**2. Dialogue Management**
- Leverage RASA’s dialogue management system to create flexible and dynamic conversations. Use RASA’s stories and rules to determine appropriate chatbot actions based on user inputs.
- Implement multi-turn dialogues to maintain conversational flow and improve the user experience.

**3. Custom Actions and APIs**
- Develop custom actions to connect the chatbot with external systems such as CRMs, order management, or product databases through API calls. This will allow the bot to perform tasks like retrieving order status, updating user details, or scheduling appointments.

**4. Fallback and Human Agent Hand-off**
- Use fallback mechanisms to handle unknown intents or provide helpful suggestions when the chatbot does not understand the user's query. 
- Implement hand-off capabilities to human agents by integrating with customer support tools (e.g., Zendesk, Freshdesk) when the chatbot is unable to resolve the issue.

**5. Multi-channel Integration**
- Deploy the chatbot on multiple channels using RASA’s connectors, ensuring that customers can access support through their preferred platforms, whether web, mobile, or social media.

**6. Testing and Continuous Improvement**
- Use RASA’s interactive learning capabilities to iteratively improve the chatbot's performance by correcting model predictions and improving intent classification and entity extraction over time.
- Collect customer feedback to refine the bot’s conversational abilities and add more functionality as needed.


![Image](/assets/rasa_architecture.png)

- **NLU Pipeline**: RASA NLU modules
- **Dialog Policies**: RASA Core’s policies
- **Action Server**: Actions that respond to users, written in Python (the default action is a text-based action)
- **Tracker Store**: A module that stores slots, entities, and conversations (This is essentially the chatbot’s memory storage module)
- **Lock Store**: A module that ensures conversations sent to the chatbot are processed sequentially, avoiding race conditions
- **Filesystem**: A module that stores and manages the chatbot’s files
- **Agent**: A module that handles overall processing and connects the other components

Among these components, we mainly work with the NLU Pipeline, Dialog Policies, Action Server, and use the Tracker to handle data in the Action Server.

#### RASA NLU Pipeline

![Image](/assets/rasa_details.webp)

The RASA NLU Pipeline consists of multiple modules to handle the user’s input text: tokenizer, feature extractor (featurizer), entity extractor (regex entity extractor, etc.), and classifier. In the diagram above, the text input is processed in the following order:

- Tokenizer
- CountVectorFeaturizer 1
- CountVectorFeaturizer 2
- DIETClassifier
- RegexEntityExtractor

In practice, it is not necessary to strictly follow this order, nor do you need to include all these components (You will see the configuration of these modules in the config.yml file). You can add or remove modules, or extract entities first using the RegexEntityExtractor before proceeding to feature extraction and intent classification.

![Image](/assets/rasa_pipe.webp)

#### RASA Core pipeline (Dialog Policies)
RASA Core determines what action is taken in response to a dialog from the user, the action can be a text dialog (Hard-defined in domain.yml) or a more complex calculation before responding (eg: Request to search for a list of nearby restaurants) (Action Server written in python)
RASA Core divides the dialog flow based on the 3 policies mentioned above: RulePolicy, MemoizationPolicy, TEDPolicy. In which:

- RulePolicy: Uses rules to determine the next action
- MemoizationPolicy: Uses Story to determine the next action
- TEDPolicy: Uses deep learning techniques to determine
To use these Policies, we configure in the config.yml file

![Image](/assets/rasa_policy.webp)

**Policy Order:**

![Image](/assets/rasa_policy_order.webp)

