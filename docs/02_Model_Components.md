## Model Components

The 5 main components of a `WideDeep` model are:

1. `Wide (Class)`
2. `DeepDense (Class)`
3. `DeepText (Class)`
4. `DeepImage (Class)`
5. `deephead (WideDeep Class parameter)`

The first 4 of them will be collected and combined by the `WideDeep` collector class, while the 5th one can be optionally added to the `WideDeep` model through its corresponding parameters: `deephead` or alternatively `head_layers`, `head_dropout` and `head_batchnorm`

### 1. Wide

The wide component is simply a Linear layer "plugged" into the output neuron(s)


```python
from pytorch_widedeep.models import Wide
```


```python
?Wide
```


```python
wide = Wide(100, 1)
wide
```




    Wide(
      (wide_linear): Linear(in_features=100, out_features=1, bias=True)
    )



### 2. DeepDense

The `DeepDense` component is comprised by a stack of dense layers that receive the embedding representation of the categorical features concatenated with numerical continuous features. For those familiar with Fastai's tabular API, DeepDense is almost identical to their tabular model, although `DeepDense` allows for more flexibility when defining the embedding dimensions. Let's have a look to `DeepDense`:


```python
import torch

from pytorch_widedeep.models import DeepDense
```


```python
# fake dataset
X_deep = torch.cat((torch.empty(5, 4).random_(4), torch.rand(5, 1)), axis=1)
colnames = ['a', 'b', 'c', 'd', 'e']
embed_input = [(u,i,j) for u,i,j in zip(colnames[:4], [4]*4, [8]*4)]
deep_column_idx = {k:v for v,k in enumerate(colnames)}
continuous_cols = ['e']
```


```python
?DeepDense
```


```python
deepdense = DeepDense(hidden_layers=[16,8], dropout=[0.5, 0.5], batchnorm=True, deep_column_idx=deep_column_idx,
                      embed_input=embed_input, continuous_cols=continuous_cols)
```


```python
deepdense
```




    DeepDense(
      (embed_layers): ModuleDict(
        (emb_layer_a): Embedding(4, 8)
        (emb_layer_b): Embedding(4, 8)
        (emb_layer_c): Embedding(4, 8)
        (emb_layer_d): Embedding(4, 8)
      )
      (embed_dropout): Dropout(p=0.0, inplace=False)
      (dense): Sequential(
        (dense_layer_0): Sequential(
          (0): Linear(in_features=33, out_features=16, bias=True)
          (1): LeakyReLU(negative_slope=0.01, inplace=True)
          (2): BatchNorm1d(16, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          (3): Dropout(p=0.5, inplace=False)
        )
        (dense_layer_1): Sequential(
          (0): Linear(in_features=16, out_features=8, bias=True)
          (1): LeakyReLU(negative_slope=0.01, inplace=True)
          (2): BatchNorm1d(8, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          (3): Dropout(p=0.5, inplace=False)
        )
      )
    )




```python
deepdense(X_deep)
```




    tensor([[ 1.9317, -0.0000,  1.3663, -0.3984, -0.0000, -0.0000, -0.0000, -1.2662],
            [ 0.0000, -1.5337, -0.0000,  0.0726, -0.4231,  3.9977, -0.0000, -0.0000],
            [-0.0000, -1.5839,  3.2978, -1.7084, -1.0877, -0.9574,  0.0000, -0.0000],
            [-0.0000,  1.6664, -1.6006,  0.0000, -0.0000, -0.9844, -0.0000, -0.0521],
            [ 2.4249,  0.0000, -0.0000, -0.0000,  0.0000, -0.0000,  2.6460,  0.0000]],
           grad_fn=<MulBackward0>)



###  3. DeepText

The `DeepText` class within the `WideDeep` package is a standard and simple stack of LSTMs on top of word embeddings. You could also add a FC-Head on top of the LSTMs. The word embeddings can be pre-trained. In the future I aim to include some simple pretrained models so that the combination between text and images is fair.  

*While I recommend using the `Wide` and `DeepDense` classes within this package when building the corresponding model components, it is very likely that the user will want to use custom text and image models. That is perfectly possible. Simply, build them and pass them as the corresponding parameters. Note that the custom models MUST return a last layer of activations (i.e. not the final prediction) so that  these activations are collected by WideDeep and combined accordingly. In  addition, the models MUST also contain an attribute `output_dim` with the size of these last layers of activations.*

Let's have a look to the `DeepText` class


```python
import torch
from pytorch_widedeep.models import DeepText
```


```python
?DeepText
```


```python
X_text = torch.cat((torch.zeros([5,1]), torch.empty(5, 4).random_(1,4)), axis=1)
```


```python
deeptext = DeepText(vocab_size=4, hidden_dim=4, n_layers=1, padding_idx=0, embed_dim=4)
```


```python
deeptext
```




    DeepText(
      (word_embed): Embedding(4, 4, padding_idx=0)
      (rnn): LSTM(4, 4, batch_first=True)
    )



You could, if you wanted, add a Fully Connected Head (FC-Head) on top of it


```python
deeptext = DeepText(vocab_size=4, hidden_dim=8, n_layers=1, padding_idx=0, embed_dim=4, head_layers=[8,4], 
                    head_batchnorm=True, head_dropout=[0.5, 0.5])
```


```python
deeptext
```




    DeepText(
      (word_embed): Embedding(4, 4, padding_idx=0)
      (rnn): LSTM(4, 8, batch_first=True)
      (texthead): Sequential(
        (dense_layer_0): Sequential(
          (0): Linear(in_features=8, out_features=4, bias=True)
          (1): LeakyReLU(negative_slope=0.01, inplace=True)
          (2): BatchNorm1d(4, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          (3): Dropout(p=0.5, inplace=False)
        )
      )
    )



Note that since the FC-Head will receive the activations from the last hidden layer of the stack of RNNs, the corresponding dimensions must be consistent.

###  4. DeepImage

The `DeepImage` class within the `WideDeep` package is either a pretrained ResNet (18, 34, or 50. Default is 18) or a stack of CNNs, to which one can add a FC-Head. If is a pretrained ResNet, you can chose how many layers you want to defrost deep into the network with the parameter `freeze`


```python
from pytorch_widedeep.models import DeepImage
```


```python
?DeepImage
```


```python
X_img = torch.rand((2,3,224,224))
```


```python
deepimage = DeepImage(head_layers=[512, 64, 8])
```


```python
deepimage
```




    DeepImage(
      (backbone): Sequential(
        (0): Conv2d(3, 64, kernel_size=(7, 7), stride=(2, 2), padding=(3, 3), bias=False)
        (1): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
        (2): ReLU(inplace=True)
        (3): MaxPool2d(kernel_size=3, stride=2, padding=1, dilation=1, ceil_mode=False)
        (4): Sequential(
          (0): BasicBlock(
            (conv1): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          )
          (1): BasicBlock(
            (conv1): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(64, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          )
        )
        (5): Sequential(
          (0): BasicBlock(
            (conv1): Conv2d(64, 128, kernel_size=(3, 3), stride=(2, 2), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(128, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (downsample): Sequential(
              (0): Conv2d(64, 128, kernel_size=(1, 1), stride=(2, 2), bias=False)
              (1): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            )
          )
          (1): BasicBlock(
            (conv1): Conv2d(128, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(128, 128, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          )
        )
        (6): Sequential(
          (0): BasicBlock(
            (conv1): Conv2d(128, 256, kernel_size=(3, 3), stride=(2, 2), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (downsample): Sequential(
              (0): Conv2d(128, 256, kernel_size=(1, 1), stride=(2, 2), bias=False)
              (1): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            )
          )
          (1): BasicBlock(
            (conv1): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(256, 256, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          )
        )
        (7): Sequential(
          (0): BasicBlock(
            (conv1): Conv2d(256, 512, kernel_size=(3, 3), stride=(2, 2), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(512, 512, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (downsample): Sequential(
              (0): Conv2d(256, 512, kernel_size=(1, 1), stride=(2, 2), bias=False)
              (1): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            )
          )
          (1): BasicBlock(
            (conv1): Conv2d(512, 512, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn1): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
            (relu): ReLU(inplace=True)
            (conv2): Conv2d(512, 512, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1), bias=False)
            (bn2): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
          )
        )
        (8): AdaptiveAvgPool2d(output_size=(1, 1))
      )
      (imagehead): Sequential(
        (dense_layer_0): Sequential(
          (0): Linear(in_features=512, out_features=64, bias=True)
          (1): LeakyReLU(negative_slope=0.01, inplace=True)
          (2): Dropout(p=0.0, inplace=False)
        )
        (dense_layer_1): Sequential(
          (0): Linear(in_features=64, out_features=8, bias=True)
          (1): LeakyReLU(negative_slope=0.01, inplace=True)
          (2): Dropout(p=0.0, inplace=False)
        )
      )
    )




```python
deepimage(X_img)
```




    tensor([[ 8.4865e-02, -3.4401e-03, -9.1973e-04,  3.4269e-01,  3.2816e-02,
              1.9682e-02, -8.0740e-04,  9.4898e-03],
            [ 1.5473e-01, -6.2664e-03, -9.3413e-05,  3.8768e-01, -1.9963e-03,
              1.1729e-01, -2.7111e-03,  1.8670e-01]], grad_fn=<LeakyReluBackward1>)



if `pretrained=False` then a stack of 4 CNNs are used


```python
deepimage = DeepImage(pretrained=False, head_layers=[512, 64, 8])
```


```python
deepimage
```




    DeepImage(
      (backbone): Sequential(
        (0): Sequential(
          (0): Conv2d(3, 64, kernel_size=(3, 3), stride=(1, 1), padding=(1, 1))
          (1): BatchNorm2d(64, eps=1e-05, momentum=0.01, affine=True, track_running_stats=True)
          (2): LeakyReLU(negative_slope=0.1, inplace=True)
          (maxpool): MaxPool2d(kernel_size=2, stride=2, padding=0, dilation=1, ceil_mode=False)
        )
        (1): Sequential(
          (0): Conv2d(64, 128, kernel_size=(1, 1), stride=(1, 1))
          (1): BatchNorm2d(128, eps=1e-05, momentum=0.01, affine=True, track_running_stats=True)
          (2): LeakyReLU(negative_slope=0.1, inplace=True)
        )
        (2): Sequential(
          (0): Conv2d(128, 256, kernel_size=(1, 1), stride=(1, 1))
          (1): BatchNorm2d(256, eps=1e-05, momentum=0.01, affine=True, track_running_stats=True)
          (2): LeakyReLU(negative_slope=0.1, inplace=True)
        )
        (3): Sequential(
          (0): Conv2d(256, 512, kernel_size=(1, 1), stride=(1, 1))
          (1): BatchNorm2d(512, eps=1e-05, momentum=0.01, affine=True, track_running_stats=True)
          (2): LeakyReLU(negative_slope=0.1, inplace=True)
          (adaptiveavgpool): AdaptiveAvgPool2d(output_size=(1, 1))
        )
      )
      (imagehead): Sequential(
        (dense_layer_0): Sequential(
          (0): Linear(in_features=512, out_features=64, bias=True)
          (1): LeakyReLU(negative_slope=0.01, inplace=True)
          (2): Dropout(p=0.0, inplace=False)
        )
        (dense_layer_1): Sequential(
          (0): Linear(in_features=64, out_features=8, bias=True)
          (1): LeakyReLU(negative_slope=0.01, inplace=True)
          (2): Dropout(p=0.0, inplace=False)
        )
      )
    )



###  5. deephead

Note that I do not use uppercase here. This is because, by default, the `deephead` is not defined outside `WideDeep` as the rest of the components. 

When defining the `WideDeep` model there is a parameter called `head_layers` (and the corresponding `head_dropout`, and `head_batchnorm`) that define the FC-head on top of `DeeDense`, `DeepText` and `DeepImage`. 

Of course, you could also chose to define it yourself externally and pass it using the parameter `deephead`. Have a look


```python
from pytorch_widedeep.models import WideDeep
```


```python
?WideDeep
```
