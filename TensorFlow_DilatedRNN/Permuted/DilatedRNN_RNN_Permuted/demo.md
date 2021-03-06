
# Permuted Pixel MNIST Demo 
Light weighted demo of our DilatedRNN on Pixel MNist with permutation.


```python
import sys
sys.path.append("./models")
import numpy as np
import tensorflow as tf
from tensorflow.examples.tutorials.mnist import input_data
from classification_models import drnn_classification
```


```python
# configurations
data_dir = "./MNIST_data"
n_steps = 28*28
input_dims = 1
n_classes = 10 

# model config
cell_type = "RNN"
assert(cell_type in ["RNN", "LSTM", "GRU"])
hidden_structs = [20] * 9
dilations = [1, 2, 4, 8, 16, 32, 64, 128, 256]
assert(len(hidden_structs) == len(dilations))

# learning config
batch_size = 128
learning_rate = 1.0e-3
training_iters = batch_size * 30000
testing_step = 5000
display_step = 100

# permutation seed 
seed = 92916
```


```python
mnist = input_data.read_data_sets(data_dir, one_hot=True)
if 'seed' in globals():
    rng_permute = np.random.RandomState(seed)
    idx_permute = rng_permute.permutation(n_steps)
else:
    idx_permute = np.random.permutation(n_steps)
```

    Extracting ./MNIST_data/train-images-idx3-ubyte.gz
    Extracting ./MNIST_data/train-labels-idx1-ubyte.gz
    Extracting ./MNIST_data/t10k-images-idx3-ubyte.gz
    Extracting ./MNIST_data/t10k-labels-idx1-ubyte.gz



```python
# build computation graph
tf.reset_default_graph()
x = tf.placeholder(tf.float32, [None, n_steps, input_dims])
y = tf.placeholder(tf.float32, [None, n_classes])    
global_step = tf.Variable(0, name='global_step', trainable=False)

# build prediction graph
print ("==> Building a dRNN with %s cells" %cell_type)
pred = drnn_classification(x, hidden_structs, dilations, n_steps, n_classes, input_dims, cell_type)

# build loss and optimizer
cost = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(logits=pred, labels=y))
optimizer = tf.train.RMSPropOptimizer(learning_rate, 0.9).minimize(cost, global_step=global_step)

# evaluation model
correct_pred = tf.equal(tf.argmax(pred,1), tf.argmax(y,1))
accuracy = tf.reduce_mean(tf.cast(correct_pred, tf.float32))
```

    ==> Building a dRNN with RNN cells
    Building layer: multi_dRNN_dilation_1, input length: 784, dilation rate: 1, input dim: 1.
    =====> Input length for sub-RNN: 784
    Building layer: multi_dRNN_dilation_2, input length: 784, dilation rate: 2, input dim: 20.
    =====> Input length for sub-RNN: 392
    Building layer: multi_dRNN_dilation_4, input length: 784, dilation rate: 4, input dim: 20.
    =====> Input length for sub-RNN: 196
    Building layer: multi_dRNN_dilation_8, input length: 784, dilation rate: 8, input dim: 20.
    =====> Input length for sub-RNN: 98
    Building layer: multi_dRNN_dilation_16, input length: 784, dilation rate: 16, input dim: 20.
    =====> Input length for sub-RNN: 49
    Building layer: multi_dRNN_dilation_32, input length: 784, dilation rate: 32, input dim: 20.
    =====> 16 time points need to be padded. 
    =====> Input length for sub-RNN: 25
    Building layer: multi_dRNN_dilation_64, input length: 784, dilation rate: 64, input dim: 20.
    =====> 48 time points need to be padded. 
    =====> Input length for sub-RNN: 13
    Building layer: multi_dRNN_dilation_128, input length: 784, dilation rate: 128, input dim: 20.
    =====> 112 time points need to be padded. 
    =====> Input length for sub-RNN: 7
    Building layer: multi_dRNN_dilation_256, input length: 784, dilation rate: 256, input dim: 20.
    =====> 240 time points need to be padded. 
    =====> Input length for sub-RNN: 4
    WARNING:tensorflow:From <ipython-input-4-7aeba9471f6b>:12: softmax_cross_entropy_with_logits (from tensorflow.python.ops.nn_ops) is deprecated and will be removed in a future version.
    Instructions for updating:
    
    Future major versions of TensorFlow will allow gradients to flow
    into the labels input on backprop by default.
    
    See tf.nn.softmax_cross_entropy_with_logits_v2.
    



```python
sess = tf.Session()
init = tf.global_variables_initializer()
sess.run(init)
```


```python
step = 0
train_results = []
validation_results = []
test_results = []

while step * batch_size < training_iters:
    batch_x, batch_y = mnist.train.next_batch(batch_size)    
    batch_x = batch_x[:, idx_permute]
    batch_x = batch_x.reshape([batch_size, n_steps, input_dims])

    feed_dict = {
        x : batch_x,
        y : batch_y
    }
    cost_, accuracy_, step_,  _ = sess.run([cost, accuracy, global_step, optimizer], feed_dict=feed_dict)    
    train_results.append((step_, cost_, accuracy_))    

    if (step + 1) % display_step == 0:
        print ("Iter " + str(step + 1) + ", Minibatch Loss: " + "{:.6f}".format(cost_) \
        + ", Training Accuracy: " + "{:.6f}".format(accuracy_))
             
    if (step + 1) % testing_step == 0:
        
        # validation performance
        batch_x = mnist.validation.images
        batch_y = mnist.validation.labels

        # permute the data
        batch_x = batch_x[:, idx_permute]        
        batch_x = batch_x.reshape([-1, n_steps, input_dims])
        feed_dict = {
            x : batch_x,
            y : batch_y
        }
        cost_, accuracy__, step_ = sess.run([cost, accuracy, global_step], feed_dict=feed_dict)
        validation_results.append((step_, cost_, accuracy__))
        
        # test performance
        batch_x = mnist.test.images
        batch_y = mnist.test.labels
        batch_x = batch_x[:, idx_permute]        
        batch_x = batch_x.reshape([-1, n_steps, input_dims])
        feed_dict = {
            x : batch_x,
            y : batch_y
        }
        cost_, accuracy_, step_ = sess.run([cost, accuracy, global_step], feed_dict=feed_dict)
        test_results.append((step_, cost_, accuracy_))        
        print ("========> Validation Accuarcy: " + "{:.6f}".format(accuracy__) \
        + ", Testing Accuarcy: " + "{:.6f}".format(accuracy_)) 
    step += 1
```

    Iter 100, Minibatch Loss: 1.217873, Training Accuracy: 0.593750
    Iter 200, Minibatch Loss: 0.648894, Training Accuracy: 0.804688
    Iter 300, Minibatch Loss: 0.575670, Training Accuracy: 0.781250
    Iter 400, Minibatch Loss: 0.409198, Training Accuracy: 0.875000
    Iter 500, Minibatch Loss: 0.582663, Training Accuracy: 0.812500
    Iter 600, Minibatch Loss: 0.390323, Training Accuracy: 0.882812
    Iter 700, Minibatch Loss: 0.337956, Training Accuracy: 0.898438
    Iter 800, Minibatch Loss: 0.370990, Training Accuracy: 0.898438
    Iter 900, Minibatch Loss: 0.425197, Training Accuracy: 0.867188
    Iter 1000, Minibatch Loss: 0.313392, Training Accuracy: 0.906250
    Iter 1100, Minibatch Loss: 0.196125, Training Accuracy: 0.953125
    Iter 1200, Minibatch Loss: 0.238387, Training Accuracy: 0.914062
    Iter 1300, Minibatch Loss: 0.295062, Training Accuracy: 0.906250
    Iter 1400, Minibatch Loss: 0.301192, Training Accuracy: 0.906250
    Iter 1500, Minibatch Loss: 0.233151, Training Accuracy: 0.929688
    Iter 1600, Minibatch Loss: 0.252008, Training Accuracy: 0.906250
    Iter 1700, Minibatch Loss: 0.207863, Training Accuracy: 0.914062
    Iter 1800, Minibatch Loss: 0.201175, Training Accuracy: 0.960938
    Iter 1900, Minibatch Loss: 0.300831, Training Accuracy: 0.929688
    Iter 2000, Minibatch Loss: 0.141390, Training Accuracy: 0.960938
    Iter 2100, Minibatch Loss: 0.241431, Training Accuracy: 0.929688
    Iter 2200, Minibatch Loss: 0.245881, Training Accuracy: 0.937500
    Iter 2300, Minibatch Loss: 0.190820, Training Accuracy: 0.976562
    Iter 2400, Minibatch Loss: 0.326672, Training Accuracy: 0.914062
    Iter 2500, Minibatch Loss: 0.316938, Training Accuracy: 0.875000
    Iter 2600, Minibatch Loss: 0.234397, Training Accuracy: 0.945312
    Iter 2700, Minibatch Loss: 0.228057, Training Accuracy: 0.929688
    Iter 2800, Minibatch Loss: 0.268731, Training Accuracy: 0.921875
    Iter 2900, Minibatch Loss: 0.212340, Training Accuracy: 0.937500
    Iter 3000, Minibatch Loss: 0.302604, Training Accuracy: 0.929688
    Iter 3100, Minibatch Loss: 0.185972, Training Accuracy: 0.929688
    Iter 3200, Minibatch Loss: 0.197821, Training Accuracy: 0.937500
    Iter 3300, Minibatch Loss: 0.159962, Training Accuracy: 0.953125
    Iter 3400, Minibatch Loss: 0.161457, Training Accuracy: 0.945312
    Iter 3500, Minibatch Loss: 0.119377, Training Accuracy: 0.984375
    Iter 3600, Minibatch Loss: 0.195043, Training Accuracy: 0.945312
    Iter 3700, Minibatch Loss: 0.221710, Training Accuracy: 0.921875
    Iter 3800, Minibatch Loss: 0.166305, Training Accuracy: 0.953125
    Iter 3900, Minibatch Loss: 0.148652, Training Accuracy: 0.960938
    Iter 4000, Minibatch Loss: 0.145465, Training Accuracy: 0.968750
    Iter 4100, Minibatch Loss: 0.185895, Training Accuracy: 0.929688
    Iter 4200, Minibatch Loss: 0.219336, Training Accuracy: 0.937500
    Iter 4300, Minibatch Loss: 0.149134, Training Accuracy: 0.968750
    Iter 4400, Minibatch Loss: 0.210364, Training Accuracy: 0.929688
    Iter 4500, Minibatch Loss: 0.131500, Training Accuracy: 0.976562
    Iter 4600, Minibatch Loss: 0.294027, Training Accuracy: 0.914062
    Iter 4700, Minibatch Loss: 0.260792, Training Accuracy: 0.945312
    Iter 4800, Minibatch Loss: 0.227699, Training Accuracy: 0.914062
    Iter 4900, Minibatch Loss: 0.168146, Training Accuracy: 0.953125
    Iter 5000, Minibatch Loss: 0.186329, Training Accuracy: 0.937500
    ========> Validation Accuarcy: 0.944200, Testing Accuarcy: 0.942700
    Iter 5100, Minibatch Loss: 0.156607, Training Accuracy: 0.945312
    Iter 5200, Minibatch Loss: 0.210934, Training Accuracy: 0.945312
    Iter 5300, Minibatch Loss: 0.120905, Training Accuracy: 0.960938
    Iter 5400, Minibatch Loss: 0.167477, Training Accuracy: 0.953125
    Iter 5500, Minibatch Loss: 0.175255, Training Accuracy: 0.945312
    Iter 5600, Minibatch Loss: 0.146085, Training Accuracy: 0.960938
    Iter 5700, Minibatch Loss: 0.175093, Training Accuracy: 0.960938
    Iter 5800, Minibatch Loss: 0.104758, Training Accuracy: 0.968750
    Iter 5900, Minibatch Loss: 0.171855, Training Accuracy: 0.929688
    Iter 6000, Minibatch Loss: 0.059026, Training Accuracy: 0.984375
    Iter 6100, Minibatch Loss: 0.110763, Training Accuracy: 0.976562
    Iter 6200, Minibatch Loss: 0.128894, Training Accuracy: 0.953125
    Iter 6300, Minibatch Loss: 0.180169, Training Accuracy: 0.929688
    Iter 6400, Minibatch Loss: 0.269718, Training Accuracy: 0.929688
    Iter 6500, Minibatch Loss: 0.152102, Training Accuracy: 0.960938
    Iter 6600, Minibatch Loss: 0.222117, Training Accuracy: 0.945312
    Iter 6700, Minibatch Loss: 0.282523, Training Accuracy: 0.890625
    Iter 6800, Minibatch Loss: 0.115102, Training Accuracy: 0.960938
    Iter 6900, Minibatch Loss: 0.133874, Training Accuracy: 0.968750
    Iter 7000, Minibatch Loss: 0.064283, Training Accuracy: 0.976562
    Iter 7100, Minibatch Loss: 0.250485, Training Accuracy: 0.929688
    Iter 7200, Minibatch Loss: 0.099579, Training Accuracy: 0.976562
    Iter 7300, Minibatch Loss: 0.267421, Training Accuracy: 0.953125
    Iter 7400, Minibatch Loss: 0.145545, Training Accuracy: 0.960938
    Iter 7500, Minibatch Loss: 0.203254, Training Accuracy: 0.937500
    Iter 7600, Minibatch Loss: 0.223166, Training Accuracy: 0.945312
    Iter 7700, Minibatch Loss: 0.274043, Training Accuracy: 0.914062
    Iter 7800, Minibatch Loss: 0.151518, Training Accuracy: 0.960938
    Iter 7900, Minibatch Loss: 0.170241, Training Accuracy: 0.968750
    Iter 8000, Minibatch Loss: 0.225683, Training Accuracy: 0.921875
    Iter 8100, Minibatch Loss: 0.218552, Training Accuracy: 0.953125
    Iter 8200, Minibatch Loss: 0.109398, Training Accuracy: 0.945312
    Iter 8300, Minibatch Loss: 0.102874, Training Accuracy: 0.976562
    Iter 8400, Minibatch Loss: 0.224485, Training Accuracy: 0.945312
    Iter 8500, Minibatch Loss: 0.041486, Training Accuracy: 0.984375
    Iter 8600, Minibatch Loss: 0.112970, Training Accuracy: 0.953125
    Iter 8700, Minibatch Loss: 0.125093, Training Accuracy: 0.953125
    Iter 8800, Minibatch Loss: 0.144896, Training Accuracy: 0.937500
    Iter 8900, Minibatch Loss: 0.124289, Training Accuracy: 0.968750
    Iter 9000, Minibatch Loss: 0.146495, Training Accuracy: 0.953125
    Iter 9100, Minibatch Loss: 0.147829, Training Accuracy: 0.937500
    Iter 9200, Minibatch Loss: 0.069774, Training Accuracy: 0.968750
    Iter 9300, Minibatch Loss: 0.086232, Training Accuracy: 0.976562
    Iter 9400, Minibatch Loss: 0.082313, Training Accuracy: 0.960938
    Iter 9500, Minibatch Loss: 0.095243, Training Accuracy: 0.976562
    Iter 9600, Minibatch Loss: 0.133992, Training Accuracy: 0.945312
    Iter 9700, Minibatch Loss: 0.091497, Training Accuracy: 0.976562
    Iter 9800, Minibatch Loss: 0.122139, Training Accuracy: 0.960938
    Iter 9900, Minibatch Loss: 0.066346, Training Accuracy: 0.976562
    Iter 10000, Minibatch Loss: 0.109506, Training Accuracy: 0.960938
    ========> Validation Accuarcy: 0.941400, Testing Accuarcy: 0.942100
    Iter 10100, Minibatch Loss: 0.153582, Training Accuracy: 0.953125
    Iter 10200, Minibatch Loss: 0.123443, Training Accuracy: 0.984375
    Iter 10300, Minibatch Loss: 0.178095, Training Accuracy: 0.937500
    Iter 10400, Minibatch Loss: 0.105871, Training Accuracy: 0.968750
    Iter 10500, Minibatch Loss: 0.075538, Training Accuracy: 0.968750
    Iter 10600, Minibatch Loss: 0.107752, Training Accuracy: 0.968750
    Iter 10700, Minibatch Loss: 0.093539, Training Accuracy: 0.976562
    Iter 10800, Minibatch Loss: 0.207272, Training Accuracy: 0.921875
    Iter 10900, Minibatch Loss: 0.100761, Training Accuracy: 0.960938
    Iter 11000, Minibatch Loss: 0.121660, Training Accuracy: 0.968750
    Iter 11100, Minibatch Loss: 0.087808, Training Accuracy: 0.968750
    Iter 11200, Minibatch Loss: 0.110900, Training Accuracy: 0.960938
    Iter 11300, Minibatch Loss: 0.066796, Training Accuracy: 0.984375
    Iter 11400, Minibatch Loss: 0.198495, Training Accuracy: 0.921875
    Iter 11500, Minibatch Loss: 0.191371, Training Accuracy: 0.960938
    Iter 11600, Minibatch Loss: 0.214099, Training Accuracy: 0.929688
    Iter 11700, Minibatch Loss: 0.117431, Training Accuracy: 0.976562
    Iter 11800, Minibatch Loss: 0.108032, Training Accuracy: 0.968750
    Iter 11900, Minibatch Loss: 0.087474, Training Accuracy: 0.968750
    Iter 12000, Minibatch Loss: 0.151373, Training Accuracy: 0.953125
    Iter 12100, Minibatch Loss: 0.103026, Training Accuracy: 0.953125
    Iter 12200, Minibatch Loss: 0.168945, Training Accuracy: 0.937500
    Iter 12300, Minibatch Loss: 0.135219, Training Accuracy: 0.937500
    Iter 12400, Minibatch Loss: 0.064339, Training Accuracy: 0.976562
    Iter 12500, Minibatch Loss: 0.091187, Training Accuracy: 0.953125
    Iter 12600, Minibatch Loss: 0.078632, Training Accuracy: 0.968750
    Iter 12700, Minibatch Loss: 0.102996, Training Accuracy: 0.968750
    Iter 12800, Minibatch Loss: 0.073673, Training Accuracy: 0.968750
    Iter 12900, Minibatch Loss: 0.186820, Training Accuracy: 0.921875
    Iter 13000, Minibatch Loss: 0.073278, Training Accuracy: 0.976562
    Iter 13100, Minibatch Loss: 0.215133, Training Accuracy: 0.921875
    Iter 13200, Minibatch Loss: 0.092290, Training Accuracy: 0.968750
    Iter 13300, Minibatch Loss: 0.119452, Training Accuracy: 0.976562
    Iter 13400, Minibatch Loss: 0.229319, Training Accuracy: 0.937500
    Iter 13500, Minibatch Loss: 0.093270, Training Accuracy: 0.968750
    Iter 13600, Minibatch Loss: 0.102054, Training Accuracy: 0.960938
    Iter 13700, Minibatch Loss: 0.158294, Training Accuracy: 0.968750
    Iter 13800, Minibatch Loss: 0.099532, Training Accuracy: 0.953125
    Iter 13900, Minibatch Loss: 0.098910, Training Accuracy: 0.960938
    Iter 14000, Minibatch Loss: 0.111413, Training Accuracy: 0.968750
    Iter 14100, Minibatch Loss: 0.103388, Training Accuracy: 0.945312
    Iter 14200, Minibatch Loss: 0.128575, Training Accuracy: 0.945312
    Iter 14300, Minibatch Loss: 0.070368, Training Accuracy: 0.960938
    Iter 14400, Minibatch Loss: 0.035028, Training Accuracy: 0.984375
    Iter 14500, Minibatch Loss: 0.115245, Training Accuracy: 0.968750
    Iter 14600, Minibatch Loss: 0.084187, Training Accuracy: 0.960938
    Iter 14700, Minibatch Loss: 0.061045, Training Accuracy: 0.984375
    Iter 14800, Minibatch Loss: 0.140908, Training Accuracy: 0.953125
    Iter 14900, Minibatch Loss: 0.237368, Training Accuracy: 0.945312
    Iter 15000, Minibatch Loss: 0.113584, Training Accuracy: 0.984375
    ========> Validation Accuarcy: 0.955200, Testing Accuarcy: 0.950200
    Iter 15100, Minibatch Loss: 0.070407, Training Accuracy: 0.992188
    Iter 15200, Minibatch Loss: 0.112176, Training Accuracy: 0.968750
    Iter 15300, Minibatch Loss: 0.071044, Training Accuracy: 0.968750
    Iter 15400, Minibatch Loss: 0.113018, Training Accuracy: 0.976562
    Iter 15500, Minibatch Loss: 0.095223, Training Accuracy: 0.968750
    Iter 15600, Minibatch Loss: 0.094758, Training Accuracy: 0.968750
    Iter 15700, Minibatch Loss: 0.153763, Training Accuracy: 0.968750
    Iter 15800, Minibatch Loss: 0.161246, Training Accuracy: 0.953125
    Iter 15900, Minibatch Loss: 0.156115, Training Accuracy: 0.953125
    Iter 16000, Minibatch Loss: 0.073605, Training Accuracy: 0.968750
    Iter 16100, Minibatch Loss: 0.110485, Training Accuracy: 0.968750
    Iter 16200, Minibatch Loss: 0.109032, Training Accuracy: 0.960938
    Iter 16300, Minibatch Loss: 0.097022, Training Accuracy: 0.984375
    Iter 16400, Minibatch Loss: 0.078908, Training Accuracy: 0.976562
    Iter 16500, Minibatch Loss: 0.157971, Training Accuracy: 0.960938
    Iter 16600, Minibatch Loss: 0.113900, Training Accuracy: 0.960938
    Iter 16700, Minibatch Loss: 0.044644, Training Accuracy: 1.000000
    Iter 16800, Minibatch Loss: 0.103692, Training Accuracy: 0.968750
    Iter 16900, Minibatch Loss: 0.050414, Training Accuracy: 0.984375
    Iter 17000, Minibatch Loss: 0.092936, Training Accuracy: 0.976562
    Iter 17100, Minibatch Loss: 0.147109, Training Accuracy: 0.960938
    Iter 17200, Minibatch Loss: 0.102026, Training Accuracy: 0.976562
    Iter 17300, Minibatch Loss: 0.112156, Training Accuracy: 0.976562
    Iter 17400, Minibatch Loss: 0.141811, Training Accuracy: 0.945312
    Iter 17500, Minibatch Loss: 0.114926, Training Accuracy: 0.960938
    Iter 17600, Minibatch Loss: 0.117075, Training Accuracy: 0.953125
    Iter 17700, Minibatch Loss: 0.109597, Training Accuracy: 0.945312
    Iter 17800, Minibatch Loss: 0.091628, Training Accuracy: 0.968750
    Iter 17900, Minibatch Loss: 0.150276, Training Accuracy: 0.976562
    Iter 18000, Minibatch Loss: 0.080136, Training Accuracy: 0.976562
    Iter 18100, Minibatch Loss: 0.137448, Training Accuracy: 0.953125
    Iter 18200, Minibatch Loss: 0.076299, Training Accuracy: 0.984375
    Iter 18300, Minibatch Loss: 0.094828, Training Accuracy: 0.960938
    Iter 18400, Minibatch Loss: 0.092209, Training Accuracy: 0.984375
    Iter 18500, Minibatch Loss: 0.097151, Training Accuracy: 0.968750
    Iter 18600, Minibatch Loss: 0.079569, Training Accuracy: 0.953125
    Iter 18700, Minibatch Loss: 0.120651, Training Accuracy: 0.960938
    Iter 18800, Minibatch Loss: 0.118920, Training Accuracy: 0.945312
    Iter 18900, Minibatch Loss: 0.158290, Training Accuracy: 0.953125
    Iter 19000, Minibatch Loss: 0.068884, Training Accuracy: 0.968750
    Iter 19100, Minibatch Loss: 0.122161, Training Accuracy: 0.937500
    Iter 19200, Minibatch Loss: 0.073587, Training Accuracy: 0.976562
    Iter 19300, Minibatch Loss: 0.096551, Training Accuracy: 0.976562
    Iter 19400, Minibatch Loss: 0.103746, Training Accuracy: 0.992188
    Iter 19500, Minibatch Loss: 0.126605, Training Accuracy: 0.945312
    Iter 19600, Minibatch Loss: 0.146229, Training Accuracy: 0.953125
    Iter 19700, Minibatch Loss: 0.075341, Training Accuracy: 0.968750
    Iter 19800, Minibatch Loss: 0.063588, Training Accuracy: 0.984375
    Iter 19900, Minibatch Loss: 0.024030, Training Accuracy: 1.000000
    Iter 20000, Minibatch Loss: 0.089634, Training Accuracy: 0.960938
    ========> Validation Accuarcy: 0.957400, Testing Accuarcy: 0.953300
    Iter 20100, Minibatch Loss: 0.070332, Training Accuracy: 0.976562
    Iter 20200, Minibatch Loss: 0.154724, Training Accuracy: 0.968750
    Iter 20300, Minibatch Loss: 0.117457, Training Accuracy: 0.960938
    Iter 20400, Minibatch Loss: 0.115784, Training Accuracy: 0.976562
    Iter 20500, Minibatch Loss: 0.068846, Training Accuracy: 0.976562
    Iter 20600, Minibatch Loss: 0.041990, Training Accuracy: 0.992188
    Iter 20700, Minibatch Loss: 0.066883, Training Accuracy: 0.976562
    Iter 20800, Minibatch Loss: 0.058559, Training Accuracy: 0.968750
    Iter 20900, Minibatch Loss: 0.100186, Training Accuracy: 0.976562
    Iter 21000, Minibatch Loss: 0.089232, Training Accuracy: 0.968750
    Iter 21100, Minibatch Loss: 0.082767, Training Accuracy: 0.976562
    Iter 21200, Minibatch Loss: 0.137770, Training Accuracy: 0.960938
    Iter 21300, Minibatch Loss: 0.103774, Training Accuracy: 0.968750
    Iter 21400, Minibatch Loss: 0.103693, Training Accuracy: 0.960938
    Iter 21500, Minibatch Loss: 0.074418, Training Accuracy: 0.976562
    Iter 21600, Minibatch Loss: 0.017874, Training Accuracy: 0.992188
    Iter 21700, Minibatch Loss: 0.072561, Training Accuracy: 0.968750
    Iter 21800, Minibatch Loss: 0.249178, Training Accuracy: 0.929688
    Iter 21900, Minibatch Loss: 0.109021, Training Accuracy: 0.976562
    Iter 22000, Minibatch Loss: 0.100376, Training Accuracy: 0.953125
    Iter 22100, Minibatch Loss: 0.103923, Training Accuracy: 0.960938
    Iter 22200, Minibatch Loss: 0.075507, Training Accuracy: 0.976562
    Iter 22300, Minibatch Loss: 0.069339, Training Accuracy: 0.976562
    Iter 22400, Minibatch Loss: 0.107979, Training Accuracy: 0.953125
    Iter 22500, Minibatch Loss: 0.220197, Training Accuracy: 0.937500
    Iter 22600, Minibatch Loss: 0.099401, Training Accuracy: 0.976562
    Iter 22700, Minibatch Loss: 0.065230, Training Accuracy: 0.984375
    Iter 22800, Minibatch Loss: 0.093145, Training Accuracy: 0.976562
    Iter 22900, Minibatch Loss: 0.098992, Training Accuracy: 0.968750
    Iter 23000, Minibatch Loss: 0.084726, Training Accuracy: 0.960938
    Iter 23100, Minibatch Loss: 0.083829, Training Accuracy: 0.976562
    Iter 23200, Minibatch Loss: 0.241455, Training Accuracy: 0.929688
    Iter 23300, Minibatch Loss: 0.060781, Training Accuracy: 0.984375
    Iter 23400, Minibatch Loss: 0.113883, Training Accuracy: 0.953125
    Iter 23500, Minibatch Loss: 0.037564, Training Accuracy: 0.984375
    Iter 23600, Minibatch Loss: 0.055533, Training Accuracy: 0.984375
    Iter 23700, Minibatch Loss: 0.082160, Training Accuracy: 0.968750
    Iter 23800, Minibatch Loss: 0.042996, Training Accuracy: 0.976562
    Iter 23900, Minibatch Loss: 0.038416, Training Accuracy: 0.992188
    Iter 24000, Minibatch Loss: 0.108771, Training Accuracy: 0.976562
    Iter 24100, Minibatch Loss: 0.129464, Training Accuracy: 0.968750
    Iter 24200, Minibatch Loss: 0.203121, Training Accuracy: 0.968750
    Iter 24300, Minibatch Loss: 0.021505, Training Accuracy: 1.000000
    Iter 24400, Minibatch Loss: 0.044558, Training Accuracy: 0.984375
    Iter 24500, Minibatch Loss: 0.119700, Training Accuracy: 0.968750
    Iter 24600, Minibatch Loss: 0.017815, Training Accuracy: 1.000000
    Iter 24700, Minibatch Loss: 0.045727, Training Accuracy: 0.976562
    Iter 24800, Minibatch Loss: 0.095012, Training Accuracy: 0.953125
    Iter 24900, Minibatch Loss: 0.123142, Training Accuracy: 0.960938
    Iter 25000, Minibatch Loss: 0.063438, Training Accuracy: 0.960938
    ========> Validation Accuarcy: 0.953000, Testing Accuarcy: 0.951000
    Iter 25100, Minibatch Loss: 0.060641, Training Accuracy: 0.984375
    Iter 25200, Minibatch Loss: 0.037531, Training Accuracy: 0.992188
    Iter 25300, Minibatch Loss: 0.051003, Training Accuracy: 0.968750
    Iter 25400, Minibatch Loss: 0.089018, Training Accuracy: 0.976562
    Iter 25500, Minibatch Loss: 0.101826, Training Accuracy: 0.968750
    Iter 25600, Minibatch Loss: 0.070767, Training Accuracy: 0.976562
    Iter 25700, Minibatch Loss: 0.096660, Training Accuracy: 0.976562
    Iter 25800, Minibatch Loss: 0.097090, Training Accuracy: 0.968750
    Iter 25900, Minibatch Loss: 0.054532, Training Accuracy: 0.984375
    Iter 26000, Minibatch Loss: 0.049698, Training Accuracy: 0.984375
    Iter 26100, Minibatch Loss: 0.066166, Training Accuracy: 0.968750
    Iter 26200, Minibatch Loss: 0.149927, Training Accuracy: 0.929688
    Iter 26300, Minibatch Loss: 0.056125, Training Accuracy: 0.976562
    Iter 26400, Minibatch Loss: 0.036056, Training Accuracy: 0.992188
    Iter 26500, Minibatch Loss: 0.026118, Training Accuracy: 0.984375
    Iter 26600, Minibatch Loss: 0.097521, Training Accuracy: 0.968750
    Iter 26700, Minibatch Loss: 0.087847, Training Accuracy: 0.976562
    Iter 26800, Minibatch Loss: 0.029292, Training Accuracy: 1.000000
    Iter 26900, Minibatch Loss: 0.041632, Training Accuracy: 0.992188
    Iter 27000, Minibatch Loss: 0.063614, Training Accuracy: 0.976562
    Iter 27100, Minibatch Loss: 0.089533, Training Accuracy: 0.976562
    Iter 27200, Minibatch Loss: 0.087505, Training Accuracy: 0.968750
    Iter 27300, Minibatch Loss: 0.048620, Training Accuracy: 0.976562
    Iter 27400, Minibatch Loss: 0.107318, Training Accuracy: 0.968750
    Iter 27500, Minibatch Loss: 0.107944, Training Accuracy: 0.953125
    Iter 27600, Minibatch Loss: 0.154255, Training Accuracy: 0.960938
    Iter 27700, Minibatch Loss: 0.100852, Training Accuracy: 0.976562
    Iter 27800, Minibatch Loss: 0.128153, Training Accuracy: 0.945312
    Iter 27900, Minibatch Loss: 0.055775, Training Accuracy: 0.976562
    Iter 28000, Minibatch Loss: 0.119425, Training Accuracy: 0.953125
    Iter 28100, Minibatch Loss: 0.080018, Training Accuracy: 0.976562
    Iter 28200, Minibatch Loss: 0.099598, Training Accuracy: 0.976562
    Iter 28300, Minibatch Loss: 0.060295, Training Accuracy: 0.992188
    Iter 28400, Minibatch Loss: 0.082797, Training Accuracy: 0.984375
    Iter 28500, Minibatch Loss: 0.013474, Training Accuracy: 1.000000
    Iter 28600, Minibatch Loss: 0.075964, Training Accuracy: 0.968750
    Iter 28700, Minibatch Loss: 0.097607, Training Accuracy: 0.945312
    Iter 28800, Minibatch Loss: 0.060283, Training Accuracy: 0.984375
    Iter 28900, Minibatch Loss: 0.071996, Training Accuracy: 0.953125
    Iter 29000, Minibatch Loss: 0.044768, Training Accuracy: 0.984375
    Iter 29100, Minibatch Loss: 0.050186, Training Accuracy: 0.984375
    Iter 29200, Minibatch Loss: 0.131764, Training Accuracy: 0.945312
    Iter 29300, Minibatch Loss: 0.137931, Training Accuracy: 0.960938
    Iter 29400, Minibatch Loss: 0.041017, Training Accuracy: 0.984375
    Iter 29500, Minibatch Loss: 0.080616, Training Accuracy: 0.968750
    Iter 29600, Minibatch Loss: 0.046189, Training Accuracy: 0.976562
    Iter 29700, Minibatch Loss: 0.033803, Training Accuracy: 0.992188
    Iter 29800, Minibatch Loss: 0.090986, Training Accuracy: 0.960938
    Iter 29900, Minibatch Loss: 0.146621, Training Accuracy: 0.960938
    Iter 30000, Minibatch Loss: 0.071132, Training Accuracy: 0.976562
    ========> Validation Accuarcy: 0.947400, Testing Accuarcy: 0.950000



```python

```
