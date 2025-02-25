Quick Start
===========

The quick start is intended for developers who are ready to dive in to the code, and see an end-to-end example of how they can integrate 🤗 Datasets into their model training workflow. For beginners who are looking for a gentler introduction, we recommend you begin with the :doc:`tutorials <./tutorial>`.

In the quick start, you will walkthrough all the steps to fine-tune `BERT <https://huggingface.co/bert-base-cased>`_ on a paraphrase classification task. Depending on the specific dataset you use, these steps may vary, but the general steps of how to load a dataset and process it are the same.

.. tip::

   For more detailed information on loading and processing a dataset, take a look at `Chapter 3 <https://huggingface.co/course/chapter3/1?fw=pt>`_ of the Hugging Face course! It covers additional important topics like dynamic padding, and fine-tuning with the Trainer API. 

Get started by installing 🤗 Datasets:

.. code::

   pip install datasets

Load the dataset and model
--------------------------

Begin by loading the `Microsoft Research Paraphrase Corpus (MRPC) <https://huggingface.co/datasets/viewer/?dataset=glue&config=mrpc>`_ training dataset from the `General Language Understanding Evaluation (GLUE) benchmark <https://huggingface.co/datasets/glue>`_. MRPC is a corpus of human annotated sentence pairs used to train a model to determine whether sentence pairs are semantically equivalent.

.. code-block::

   >>> from datasets import load_dataset
   >>> dataset = load_dataset('glue', 'mrpc', split='train')

Next, import the pre-trained BERT model and its tokenizer from the `🤗 Transformers <https://huggingface.co/transformers/>`_ library:

.. tab:: PyTorch

   >>> from transformers import AutoModelForSequenceClassification, AutoTokenizer
   >>> model = AutoModelForSequenceClassification.from_pretrained('bert-base-cased')
   Some weights of the model checkpoint at bert-base-cased were not used when initializing BertForSequenceClassification: ['cls.predictions.bias', 'cls.predictions.transform.dense.weight', 'cls.predictions.transform.dense.bias', 'cls.predictions.decoder.weight', 'cls.seq_relationship.weight', 'cls.seq_relationship.bias', 'cls.predictions.transform.LayerNorm.weight', 'cls.predictions.transform.LayerNorm.bias']
   - This IS expected if you are initializing BertForSequenceClassification from the checkpoint of a model trained on another task or with another architecture (e.g. initializing a BertForSequenceClassification model from a BertForPretraining model).
   - This IS NOT expected if you are initializing BertForSequenceClassification from the checkpoint of a model that you expect to be exactly identical (initializing a BertForSequenceClassification model from a BertForSequenceClassification model).
   Some weights of BertForSequenceClassification were not initialized from the model checkpoint at bert-base-cased and are newly initialized: ['classifier.weight', 'classifier.bias']
   You should probably TRAIN this model on a down-stream task to be able to use it for predictions and inference.
   >>> tokenizer = AutoTokenizer.from_pretrained('bert-base-cased')

.. tab:: TensorFlow

   >>> from transformers import TFAutoModelForSequenceClassification, AutoTokenizer
   >>> model = TFAutoModelForSequenceClassification.from_pretrained("bert-base-cased")
   Some weights of the model checkpoint at bert-base-cased were not used when initializing TFBertForSequenceClassification: ['nsp___cls', 'mlm___cls']
   - This IS expected if you are initializing TFBertForSequenceClassification from the checkpoint of a model trained on another task or with another architecture (e.g. initializing a BertForSequenceClassification model from a BertForPretraining model).
   - This IS NOT expected if you are initializing TFBertForSequenceClassification from the checkpoint of a model that you expect to be exactly identical (initializing a BertForSequenceClassification model from a BertForSequenceClassification model).
   Some weights of TFBertForSequenceClassification were not initialized from the model checkpoint at bert-base-cased and are newly initialized: ['dropout_37', 'classifier']
   You should probably TRAIN this model on a down-stream task to be able to use it for predictions and inference.
   >>> tokenizer = AutoTokenizer.from_pretrained('bert-base-cased')

Tokenize the dataset
--------------------

The next step is to tokenize the text in order to build sequences of integers the model can understand. Encode the entire dataset with :func:`datasets.Dataset.map`, and truncate and pad the inputs to the maximum length of the model. This ensures the appropriate tensor batches are built.

.. code-block::

   >>> def encode(examples):
   ...     return tokenizer(examples['sentence1'], examples['sentence2'], truncation=True, padding='max_length')
   
   >>> dataset = dataset.map(encode, batched=True)
   >>> dataset[0]
   {'sentence1': 'Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
   'sentence2': 'Referring to him as only " the witness " , Amrozi accused his brother of deliberately distorting his evidence .',
   'label': 1,
   'idx': 0,
   'input_ids': array([  101,  7277,  2180,  5303,  4806,  1117,  1711,   117,  2292, 1119,  1270,   107,  1103,  7737,   107,   117,  1104,  9938, 4267, 12223, 21811,  1117,  2554,   119,   102, 11336,  6732, 3384,  1106,  1140,  1112,  1178,   107,  1103,  7737,   107, 117,  7277,  2180,  5303,  4806,  1117,  1711,  1104,  9938, 4267, 12223, 21811,  1117,  2554,   119,   102]),
   'token_type_ids': array([0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]),
   'attention_mask': array([1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1])}

Notice how there are three new columns in the dataset: ``input_ids``, ``token_type_ids``, and ``attention_mask``. These columns are the inputs to the model.

Format the dataset
------------------

Depending on whether you are using PyTorch, TensorFlow, or JAX, you will need to format the dataset accordingly. There are three changes you need to make to the dataset:

1. Rename the ``label`` column to ``labels``, the expected input name in `BertForSequenceClassification <https://huggingface.co/transformers/model_doc/bert.html#transformers.BertForSequenceClassification.forward>`__ or `TFBertForSequenceClassification <https://huggingface.co/transformers/model_doc/bert.html#tfbertforsequenceclassification>`__:
   
.. code::

   >>> dataset = dataset.map(lambda examples: {'labels': examples['label']}, batched=True)

2. Retrieve the actual tensors from the Dataset object instead of using the current Python objects.
3. Filter the dataset to only return the model inputs: ``input_ids``, ``token_type_ids``, and ``attention_mask``.
   
:func:`datasets.Dataset.set_format` completes the last two steps on-the-fly. After you set the format, wrap the dataset in ``torch.utils.data.DataLoader`` or ``tf.data.Dataset``:

.. tab:: PyTorch

   >>> import torch
   >>> dataset.set_format(type='torch', columns=['input_ids', 'token_type_ids', 'attention_mask', 'labels'])
   >>> dataloader = torch.utils.data.DataLoader(dataset, batch_size=32)
   >>> next(iter(dataloader))
   {'attention_mask': tensor([[1, 1, 1,  ..., 0, 0, 0],
                         [1, 1, 1,  ..., 0, 0, 0],
                         [1, 1, 1,  ..., 0, 0, 0],
                         ...,
                         [1, 1, 1,  ..., 0, 0, 0],
                         [1, 1, 1,  ..., 0, 0, 0],
                         [1, 1, 1,  ..., 0, 0, 0]]),
   'input_ids': tensor([[  101,  7277,  2180,  ...,     0,     0,     0],
                   [  101, 10684,  2599,  ...,     0,     0,     0],
                   [  101,  1220,  1125,  ...,     0,     0,     0],
                   ...,
                   [  101, 16944,  1107,  ...,     0,     0,     0],
                   [  101,  1109, 11896,  ...,     0,     0,     0],
                   [  101,  1109,  4173,  ...,     0,     0,     0]]),
   'label': tensor([1, 0, 1, 0, 1, 1, 0, 1]),
   'token_type_ids': tensor([[0, 0, 0,  ..., 0, 0, 0],
                        [0, 0, 0,  ..., 0, 0, 0],
                        [0, 0, 0,  ..., 0, 0, 0],
                        ...,
                        [0, 0, 0,  ..., 0, 0, 0],
                        [0, 0, 0,  ..., 0, 0, 0],
                        [0, 0, 0,  ..., 0, 0, 0]])}

.. tab:: TensorFlow

   >>> import tensorflow as tf
   >>> dataset.set_format(type='tensorflow', columns=['input_ids', 'token_type_ids', 'attention_mask', 'labels'])
   >>> features = {x: dataset[x].to_tensor(default_value=0, shape=[None, tokenizer.model_max_length]) for x in ['input_ids', 'token_type_ids', 'attention_mask']}
   >>> tfdataset = tf.data.Dataset.from_tensor_slices((features, dataset["labels"])).batch(32)
   >>> next(iter(tfdataset))
   ({'input_ids': <tf.Tensor: shape=(32, 512), dtype=int32, numpy=
   array([[  101,  7277,  2180, ...,     0,     0,     0],
     [  101, 10684,  2599, ...,     0,     0,     0],
     [  101,  1220,  1125, ...,     0,     0,     0],
     ...,
     [  101,  1109,  2026, ...,     0,     0,     0],
     [  101, 22263,  1107, ...,     0,     0,     0],
     [  101,   142,  1813, ...,     0,     0,     0]], dtype=int32)>, 'token_type_ids': <tf.Tensor: shape=(32, 512), dtype=int32, numpy=
   array([[0, 0, 0, ..., 0, 0, 0],
     [0, 0, 0, ..., 0, 0, 0],
     [0, 0, 0, ..., 0, 0, 0],
     ...,
     [0, 0, 0, ..., 0, 0, 0],
     [0, 0, 0, ..., 0, 0, 0],
     [0, 0, 0, ..., 0, 0, 0]], dtype=int32)>, 'attention_mask': <tf.Tensor: shape=(32, 512), dtype=int32, numpy=
   array([[1, 1, 1, ..., 0, 0, 0],
     [1, 1, 1, ..., 0, 0, 0],
     [1, 1, 1, ..., 0, 0, 0],
     ...,
     [1, 1, 1, ..., 0, 0, 0],
     [1, 1, 1, ..., 0, 0, 0],
     [1, 1, 1, ..., 0, 0, 0]], dtype=int32)>}, <tf.Tensor: shape=(32,), dtype=int64, numpy=
   array([1, 0, 1, 0, 1, 1, 0, 1, 0, 0, 0, 0, 1, 1, 0, 0, 0, 1, 0, 1, 1, 1,
     0, 1, 1, 1, 0, 0, 1, 1, 1, 0])>)

Train the model
---------------

Lastly, create a simple training loop and start training:

.. tab:: PyTorch

   >>> from tqdm import tqdm
   >>> device = 'cuda' if torch.cuda.is_available() else 'cpu' 
   >>> model.train().to(device)
   >>> optimizer = torch.optim.AdamW(params=model.parameters(), lr=1e-5)
   >>> for epoch in range(3):
   ...     for i, batch in enumerate(tqdm(dataloader)):
   ...         batch = {k: v.to(device) for k, v in batch.items()}
   ...         outputs = model(**batch)
   ...         loss = outputs[0]
   ...         loss.backward()
   ...         optimizer.step()
   ...         optimizer.zero_grad()
   ...         if i % 10 == 0:
   ...             print(f"loss: {loss}")

.. tab:: TensorFlow
  
   >>> loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(reduction=tf.keras.losses.Reduction.NONE, from_logits=True)
   >>> opt = tf.keras.optimizers.Adam(learning_rate=3e-5)
   >>> model.compile(optimizer=opt, loss=loss_fn, metrics=["accuracy"])
   >>> model.fit(tfdataset, epochs=3)

What's next?
------------

This completes the basic steps of loading a dataset to train a model. You loaded and processed the MRPC dataset to fine-tune BERT to determine whether sentence pairs have the same meaning.

For your next steps, take a look at our :doc:`How-to guides <./how_to>` and learn how to achieve a specific task (e.g. load a dataset offline, add a dataset to the Hub, change the name of a column). Or if you want to deepen your knowledge of 🤗 Datasets core concepts, read our :doc:`Conceptual Guides <./about_arrow>`.