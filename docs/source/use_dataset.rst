Train with 🤗 Datasets
======================

So far, you loaded a dataset from the Hugging Face Hub and learned how to access the information stored inside the dataset. Now you will tokenize and use your dataset with a framework such as PyTorch or TensorFlow. By default, all the dataset columns are returned as Python objects. But you can bridge the gap between a Python object and your machine learning framework by setting the format of a dataset. Formatting casts the columns into compatible PyTorch or TensorFlow types.

.. important::
    
   Often times you may want to modify the structure and content of your dataset before you use it to train a model. For example, you may want to remove a column or cast it as a different type. 🤗 Datasets provides the necessary tools to do this, but since each dataset is so different, the processing approach will vary individually. For more detailed information about preprocessing data, take a look at our `guide <https://huggingface.co/transformers/preprocessing.html#>`_ from the 🤗 Transformers library. Then come back and read our :doc:`How-to Process <./process>` guide to see all the different methods for processing your dataset.

Tokenize
--------

Tokenization divides text into individual words called tokens. Tokens are converted into numbers, which is what the model receives as its input. 

The first step is to install the 🤗 Transformers library:

.. code::

   pip install transformers

Next, import a tokenizer. It is important to use the tokenizer that is associated with the model you are using, so the text is split in the same way. In this example, load the `BERT tokenizer <https://huggingface.co/transformers/model_doc/bert.html#berttokenizerfast>`_ because you are using the `BERT <https://huggingface.co/bert-base-cased>`_ model:

.. code-block::

   >>> from transformers import BertTokenizerFast
   >>> tokenizer = BertTokenizerFast.from_pretrained('bert-base-cased')

Now you can tokenize ``sentence1`` field of the dataset:

.. code-block::

   >>> encoded_dataset = dataset.map(lambda examples: tokenizer(examples['sentence1']), batched=True)
   >>> encoded_dataset.column_names
   ['sentence1', 'sentence2', 'label', 'idx', 'input_ids', 'token_type_ids', 'attention_mask']
   >>> encoded_dataset[0]
   {'sentence1': 'Amrozi accused his brother , whom he called " the witness " , of deliberately distorting his evidence .',
   'sentence2': 'Referring to him as only " the witness " , Amrozi accused his brother of deliberately distorting his evidence .',
   'label': 1,
   'idx': 0,
   'input_ids': [  101,  7277,  2180,  5303,  4806,  1117,  1711,   117,  2292, 1119,  1270,   107,  1103,  7737,   107,   117,  1104,  9938, 4267, 12223, 21811,  1117,  2554,   119,   102],
   'token_type_ids': [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
   'attention_mask': [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
   }

The tokenization process creates three new columns: ``input_ids``, ``token_type_ids``, and ``attention_mask``. These are the inputs to the model.

Use in PyTorch or TensorFlow
----------------------------

Next, format the dataset into compatible PyTorch or TensorFlow types.

PyTorch
^^^^^^^

If you are using PyTorch, set the format with :func:`datasets.Dataset.set_format`, which accepts two main arguments:

1. ``type`` defines the type of column to cast to. For example, ``torch`` returns PyTorch tensors.
   
2. ``columns`` specify which columns should be formatted.

After you set the format, wrap the dataset with ``torch.utils.data.DataLoader``. Your dataset is now ready for use in a training loop!

.. code-block::

   >>> import torch
   >>> from datasets import load_dataset
   >>> from transformers import AutoTokenizer
   >>> dataset = load_dataset('glue', 'mrpc', split='train')
   >>> tokenizer = AutoTokenizer.from_pretrained('bert-base-cased')
   >>> dataset = dataset.map(lambda e: tokenizer(e['sentence1'], truncation=True, padding='max_length'), batched=True)
   >>> dataset.set_format(type='torch', columns=['input_ids', 'token_type_ids', 'attention_mask', 'label'])
   >>> dataloader = torch.utils.data.DataLoader(dataset, batch_size=32)
   >>> next(iter(dataloader))
   {'attention_mask': tensor([[1, 1, 1,  ..., 0, 0, 0],
                            ...,
                            [1, 1, 1,  ..., 0, 0, 0]]),
   'input_ids': tensor([[  101,  7277,  2180,  ...,     0,     0,     0],
                       ...,
                       [  101,  1109,  4173,  ...,     0,     0,     0]]),
   'label': tensor([1, 0, 1, 0, 1, 1, 0, 1]),
   'token_type_ids': tensor([[0, 0, 0,  ..., 0, 0, 0],
                            ...,
                            [0, 0, 0,  ..., 0, 0, 0]])}

TensorFlow
^^^^^^^^^^

If you are using TensorFlow, set the format with ``to_tf_dataset``, which accepts several arguments:

1. ``columns`` specify which columns should be formatted (includes the inputs and labels).

2. ``shuffle`` determines whether the dataset should be shuffled.

3. ``batch_size`` specifies the batch size.

4. ``collate_fn`` specifies a data collator that will batch each processed example and apply padding. If you are using a ``DataCollator``, make sure you set ``return_tensors="tf"`` when you initialize it to return ``tf.Tensor`` outputs.

.. code-block::

   >>> import tensorflow as tf
   >>> from datasets import load_dataset
   >>> from transformers import AutoTokenizer
   >>> dataset = load_dataset('glue', 'mrpc', split='train')
   >>> tokenizer = AutoTokenizer.from_pretrained('bert-base-cased')
   >>> dataset = dataset.map(lambda e: tokenizer(e['sentence1'], truncation=True, padding='max_length'), batched=True)
   >>> data_collator = DataCollatorWithPadding(tokenizer=tokenizer, return_tensors="tf")
   >>> train_dataset = dataset["train"].to_tf_dataset(
   ...   columns=['input_ids', 'token_type_ids', 'attention_mask', 'label'],
   ...   shuffle=True,
   ...   batch_size=16,
   ...   collate_fn=data_collator,
   ... )
   >>> next(iter(train_dataset))
   {'attention_mask': <tf.Tensor: shape=(16, 512), dtype=int64, numpy=
    array([[1, 1, 1, ..., 0, 0, 0],
         ...,
         [1, 1, 1, ..., 0, 0, 0]])>,
    'input_ids': <tf.Tensor: shape=(16, 512), dtype=int64, numpy=
     array([[  101, 11336, 11154, ...,     0,     0,     0],
         ..., 
         [  101,   156, 22705, ...,     0,     0,     0]])>,
    'labels': <tf.Tensor: shape=(16,), dtype=int64, numpy=
     array([1, 1, 0, 1, 0, 1, 1, 1, 0, 0, 1, 1, 0, 0, 1, 0])>,
    'token_type_ids': <tf.Tensor: shape=(16, 512), dtype=int64, numpy=
     array([[0, 0, 0, ..., 0, 0, 0],
          ...,
         [0, 0, 0, ..., 0, 0, 0]])>
   }

.. tip::

   ``to_tf_dataset`` is the easiest way to create a TensorFlow compatible dataset. If you are looking for additional options for constructing a TensorFlow dataset, take a look at the :ref:`format` section!

Your dataset is now ready for use in a training loop!