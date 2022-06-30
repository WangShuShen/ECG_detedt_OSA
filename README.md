# ECG_detect_OSA

------------------------------------------
This program is designed to recognize **Obstructive sleep apnea**. Using I²C connect MAX86150 with ARC EM9D board GMA303KU on board and Convolutional Neural Networks to recognize these diease.

* [Introduction](#introduction)
* [Hardware and Software Setup](#hardware-and-software-setup)
	* [Required Hardware](#required-hardware)
	* [Required Software](#required-software)
	* [Hardware Connection](#hardware-connection)
* [User Manual](#user-manual)
	* [Compatible Model](#compatible-model)
	* [UART send 3-axis accelerometer](#UART-send-3-axis-accelerometer)
	* [UART recieve 3-axis accelerometer](#UART-recieve-3-axis-accelerometer)
	* [Training Model](#Training-Model)
	* [Put realtime 3-axis accelerometer observing result](#Put-realtime-3-axis-accelerometer-observing-result)


## Introduction
- 1 We use 3-axis accelerometer to determine user whether is go to sleep.
- 2 Using I²C to trasport ECG data to ARC EM9D board.
- 3 Using Convolutional Neural Networks model to recognize real time ECG data.
- 4 Show the result to user.

All hardware are in the picture following:

<img width="450" alt="PC2" src="https://user-images.githubusercontent.com/87894572/176442424-26c242db-f6ff-4690-a84b-176652868726.png">

## Hardware and Software Setup
### Required Hardware
-  ARC EM9D board
-  MAX86150
-  OLED1306

All hardware are in the picture following:

<img width="450" alt="PC2" src="https://user-images.githubusercontent.com/87894572/176450960-a8cc7ce9-5fe4-49f3-83ad-ed625e810ab7.png">




### Required Software
- Metaware or ARC GNU Toolset
- Serial port terminal, such as putty, tera-term or minicom
- VirtualBox(Ubuntu 20.04)

### Hardware Connection
- ARC EM9D, MAX861150 and OLED1306 connected by wire.

## User Manual

### Compatible Model

1. Download [Apnea-ECG Database](https://physionet.org/content/apnea-ecg/1.0.0/)

1. Give the **Apnea-ECG Database** which held Computers contest with Cardiology in 2000.

1. Use **Tensorflow 2.x** and **Python 3.7 up** to training model

1. Import module to read Apnea-ECG Database

```python
import wfdb
```

### Initial each I/O.

- Open workshop/Synopsys_SDK_V22/Example_Project/Lab2_I2C_OLED to know I²C Serial Transport.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include "embARC.h"
#include "embARC_debug.h"
#include "board_config.h"
#include "arc_timer.h"
#include "hx_drv_spi_s.h"
#include "spi_slave_protocol.h"
#include "hardware_config.h"

#include "hx_drv_iic_m.h"
#include "synopsys_i2c_oled1306.h"

#include "hx_drv_uart.h"
#define uart_buf_size 100

#define USE_SS_IIC_X USE_SS_IIC_1

DEV_UART *uart0_ptr;
char uart_buf[uart_buf_size] = {0};

DEV_IIC *iic1_ptr;
int main(void)
{
    // UART 0 is already initialized with 115200bps
    printf("This is Lab2_I2C_OLED\r\n");

    uart0_ptr = hx_drv_uart_get_dev(USE_SS_UART_0);

    sprintf(uart_buf, "I2C0 Init\r\n");
    uart0_ptr->uart_write(uart_buf, strlen(uart_buf));
    board_delay_ms(10);

    iic1_ptr = hx_drv_i2cm_get_dev(USE_SS_IIC_1);
    iic1_ptr->iic_open(DEV_MASTER_MODE, IIC_SPEED_FAST);
    
    //I2C read
    hx_drv_i2cm_read_data(USE_SS_IIC_X, GMA303KU_ADDRESS, read_buf, read_len);
    //I2C write
    a = hx_drv_i2cm_write_data(USE_SS_IIC_X, 0xbc, &data_write[0], 0, &data_write[0], 2); 
    
    //code
 }
```
- Initial MAX86150 to send ECG data.


```c
void InitMax86150(void){
    InitI2C();
    uint8_t initial_data_write[8][2] = { 
                                         {0x02, 0x80}, //Interrupt Enable 1
                                         {0x03, 0x04}, //Interrupt Enable 2
                                         {0x08, 0x10}, //FIFO Configuration 
                                         {0x09, 0x09}, //FIFO Data Control Register 1
                                         {0x0a, 0x00}, //FIFO Data Control Register 2
                                         {0x3c, 0x03}, //ECG Configuration 1   //0x02   0x03
                                         {0x3e, 0x00}, //ECG Configuration 3    //0x0D  0x00
                                         {0x0d, 0x04}}; //System Control; 
    for(int i = 0; i < 8; i++){
        Max86150_WriteData(initial_data_write[i][0], initial_data_write[i][1]);
    }
    board_delay_ms(500);
        OLED_Init();

	OLED_Clear();                        
	OLED_SetCursor(0, 0);
    OLED_DisplayString("Welcome To ARC EVK");

	OLED_SetCursor(1, 0);
	for(oled_i = 0; oled_i < 128; oled_i ++)
		oledSendData(oled_i);
}
```
- Initial OLED1306 to show result.
```c
OLED_Init();

OLED_Clear();                        
OLED_SetCursor(0, 0);
OLED_DisplayString("Welcome To ARC EVK");

OLED_SetCursor(1, 0);
for(oled_i = 0; oled_i < 128; oled_i ++)
	oledSendData(oled_i);
```

- Initial GMA303KU to send 3-axis acceleration.
```c
int16_t accel_x;
int16_t accel_y;
int16_t accel_z;
int16_t accel_t;

uint8_t xg_sign;
uint8_t yg_sign;
uint8_t zg_sign;
uint8_t temp_sign;

int16_t xg_10x;
int16_t yg_10x;
int16_t zg_10x;
int16_t temp_10x;
int main(){

    //your code
     iic1_ptr = hx_drv_i2cm_get_dev(USE_SS_IIC_0);
    iic1_ptr->iic_open(DEV_MASTER_MODE, IIC_SPEED_STANDARD); 
    uint8_t chip_id = GMA303KU_Init();
    board_delay_ms(100);

    if(chip_id == 0xA3)
        sprintf(uart_buf, "Chip ID: 0x%2X | OK\r\n\n", chip_id);    //It should be 0xA3
    else 
        sprintf(uart_buf, "Chip ID: 0x%2X | Error\r\n\n", chip_id);    //It should be 0xA3
    uart0_ptr->uart_write(uart_buf, strlen(uart_buf));
    board_delay_ms(10);
    //your code
}
```


### UART recieve 3-axis accelerometer

- Open Jupyter Notebook compile these code down

```cpp
from time import sleep
import serial
ser =serial.Serial("COM12", 115200,timeout=0.0835,writeTimeout=1) 
Arduino_cmd='8'
cmd=Arduino_cmd.encode("utf-8")
ser.write(cmd)
ser.flushInput()
sleep(0.06) 
while(1):
    
    
    rv=ser.read(12)
    print(rv)
ser.flushInput()
ser.close()
```
- Check recieving result
### Loading dataset & pre-processing
- Includes module
```python
import wfdb
import numpy as np
import tensorflow
from tensorflow.keras.callbacks import ModelCheckpoint
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense, Flatten, BatchNormalization,Dropout
from tensorflow.keras.layers import Conv2D, MaxPooling2D
from tensorflow.keras.regularizers import l2
import sklearn
from sklearn.model_selection import train_test_split
from sklearn.metrics import confusion_matrix
import matplotlib.pyplot as plt
```

- Load testing dataset

```python
adr =1
if adr ==1:
 change_link='/content/drive/MyDrive/CNN/ECG_model/sleep_apena_detect/apnea-ecg-database-1.0.0/'
elif adr == 2:
  change_link='/content/drive/MyDrive/ecg_data/apnea-ecg-database-1.0.0/'

x_test=[]
y_test=[]

test_string=['x01','x02','x03','x04','x05','x06','x07','x08','x09','x10','x11','x12',
             'x13','x14','x15','x16','x17','x18','x19','x20','x21','x22','x23','x24',
             'x25','x26','x27','x28','x29','x30','x31','x32','x33','x34','x35']

             
test_label_count=[522,468,464,481,504,449,508,516,507,509,456,526,505,489,497,514,399,458,486,
                  512,509,481,526,428,509,519,497,494,469,510,556,537,472,474,482]

test_label=np.loadtxt(change_link+'test-dataset-annos.txt',delimiter='\t',dtype=np.str)



tmp=[]

for i in test_label:
    tmp+=list(i)


for i in range(len(tmp)):
    if tmp[i]=='N':
        tmp[i]=0
    else:
        tmp[i]=1
y_test=np.array(tmp,dtype=np.int8)

count=0
for k in test_string:
    record = wfdb.rdrecord(record_name=change_link+k,return_res=16,physical=True)
    ecg_signal = record.p_signal
    ecg_signal=np.delete(ecg_signal,np.s_[((test_label_count[count])*6000):])
    x_test=np.append(x_test,ecg_signal)
    x_test = x_test.astype(np.float32)
    count+=1
```
- Load training dataset
```python
x_train=[]
y_train=[]
list_string=['a01','a02','a03','a04','a05','a06','a07','a08','a09','a10','a11','a12',
             'a13','a14','a15','a16','a17','a18','a19','a20','b01','b02','b03','b04',
             'b05','c01','c02','c03','c04','c05','c06','c07','c08','c09','c10']

for k in list_string:

    record = wfdb.rdrecord(record_name=change_link+k,return_res=16,physical=True)
    ann = wfdb.rdann(change_link+k,'apn')

    ecg_signal_label=ann.symbol
    ecg_signal_label=np.array(ecg_signal_label)

    ecg_signal = record.p_signal


    ecg_signal=np.delete(ecg_signal,np.s_[((len(ecg_signal_label)-1)*6000):])
    x_train=np.append(x_train,ecg_signal)
    x_train = x_train.astype(np.float32)

    for i in range(len(ecg_signal_label)):
        if ecg_signal_label[i]=='N':
            ecg_signal_label[i]=0
        else:
            ecg_signal_label[i]=1
    ecg_signal_label=np.array(ecg_signal_label,dtype=np.int8)
    ecg_signal_label=np.delete(ecg_signal_label,np.s_[(len(ecg_signal_label)-1):])
    y_train=np.append(y_train,ecg_signal_label)
```
- Merge two dataset in one
```python
tmp_data=np.append(x_train,x_test,)
tmp_label=np.append(y_train,y_test)
```
- Dataset normalization
```python
tmp_data_max_abs=np.max(np.abs(tmp_data))
print(tmp_data)
tmp_data=tmp_data/tmp_data_max_abs
print(tmp_data)
```
- Split dataset
```python
tmp_data=np.reshape(tmp_data,(int(tmp_data.size/6000),6000,1,1))

x_train, x_test, y_train, y_test = train_test_split(tmp_data, tmp_label, test_size=0.1)
x_train.shape,y_train.shape ,x_test.shape , y_test.shape
```
- Flatten Label
```python
y_train = y_train.flatten()
y_test = y_test.flatten()
```
- Encoder Label
```python
num_classes=2
y_train_encoder = sklearn.preprocessing.LabelEncoder()
y_train_num = y_train_encoder.fit_transform(y_train)
y_train_wide = tensorflow.keras.utils.to_categorical(y_train_num, num_classes)
y_test_num = y_train_encoder.fit_transform(y_test)
y_test_wide = tensorflow.keras.utils.to_categorical(y_test_num, num_classes)
```
```python
```
```python
```
- Preparing dataset

<img width="800" alt="image" src="https://user-images.githubusercontent.com/87894572/126873949-ebe2d658-df40-4cb2-8f57-ba96a8fc3cb8.png">

```cpp
TRAIN = "train/"
TEST = "test/"
# Load "X" (the neural network's training and testing inputs)
def load_X(X_signals_paths):
    X_signals = []
    
    for signal_type_path in X_signals_paths:
        file = open(signal_type_path, 'r')
        # Read dataset from disk, dealing with text files' syntax
        X_signals.append(
            [np.array(serie, dtype=np.float32) for serie in [
                row.replace('  ', ' ').strip().split(' ') for row in file
            ]]
        )
        file.close()
    
    return np.transpose(np.array(X_signals), (1, 2, 0))
X_train_signals_paths = [
    DATASET_PATH + TRAIN + "Inertial Signals/" + signal + "train.txt" for signal in INPUT_SIGNAL_TYPES
]
X_test_signals_paths = [
    DATASET_PATH + TEST + "Inertial Signals/" + signal + "test.txt" for signal in INPUT_SIGNAL_TYPES
]
X_train = load_X(X_train_signals_paths)
X_test = load_X(X_test_signals_paths)
# Load "y" (the neural network's training and testing outputs)
def load_y(y_path):
    file = open(y_path, 'r')
    # Read dataset from disk, dealing with text file's syntax
    y_ = np.array(
        [elem for elem in [
            row.replace('  ', ' ').strip().split(' ') for row in file
        ]], 
        dtype=np.int32
    )
    file.close()
    
    # Substract 1 to each output class for friendly 0-based indexing 
    return y_ - 1
y_train_path = DATASET_PATH + TRAIN + "y_train.txt"
y_test_path = DATASET_PATH + TEST + "y_test.txt"
y_train = load_y(y_train_path)
y_test = load_y(y_test_path)
```

- Additionnal Parameters
Here are some core parameter definitions for the training.

For example, the whole neural network's structure could be summarised by enumerating those parameters and the fact that two LSTM are used one on top of another (stacked) output-to-input as hidden layers through time steps.
```cpp
# Input Data 
training_data_count = len(X_train)  # 7352 training series (with 50% overlap between each serie)
test_data_count = len(X_test)  # 2947 testing series
n_steps = len(X_train[0])  # 128 timesteps per series
n_input = len(X_train[0][0])  # 9 input parameters per timestep
# LSTM Neural Network's internal structure
n_hidden = 32 # Hidden layer num of features
n_classes = 6 # Total classes (should go up, or should go down)
# Training 
learning_rate = 0.0025
lambda_loss_amount = 0.0015
training_iters = training_data_count*300 # Loop 300 times on the dataset
batch_size = 100
display_iter = 30000  # To show test set accuracy during training
# Some debugging info
print("Some useful info to get an insight on dataset's shape and normalisation:")
print("(X shape, y shape, every X's mean, every X's standard deviation)")
print(X_test.shape, y_test.shape, np.mean(X_test), np.std(X_test))
print("The dataset is therefore properly normalised, as expected, but not yet one-hot encoded.")
```
- Utility functions for training
```cpp
def LSTM_RNN(_X, _weights, _biases):
    # Function returns a tensorflow LSTM (RNN) artificial neural network from given parameters. 
    # Moreover, two LSTM cells are stacked which adds deepness to the neural network. 
    # Note, some code of this notebook is inspired from an slightly different 
    # RNN architecture used on another dataset, some of the credits goes to 
    # "aymericdamien" under the MIT license.
    # (NOTE: This step could be greatly optimised by shaping the dataset once
    # input shape: (batch_size, n_steps, n_input)
    _X = tf.transpose(_X, [1, 0, 2])  # permute n_steps and batch_size
    # Reshape to prepare input to hidden activation
    _X = tf.reshape(_X, [-1, n_input]) 
    # new shape: (n_steps*batch_size, n_input)
    
    # ReLU activation, thanks to Yu Zhao for adding this improvement here:
    _X = tf.nn.relu(tf.matmul(_X, _weights['hidden']) + _biases['hidden'])
    # Split data because rnn cell needs a list of inputs for the RNN inner loop
    _X = tf.split(_X, n_steps, 0) 
    # new shape: n_steps * (batch_size, n_hidden)
    # Define two stacked LSTM cells (two recurrent layers deep) with tensorflow
    lstm_cell_1 = tf.contrib.rnn.BasicLSTMCell(n_hidden, forget_bias=1.0, state_is_tuple=True)
    lstm_cell_2 = tf.contrib.rnn.BasicLSTMCell(n_hidden, forget_bias=1.0, state_is_tuple=True)
    lstm_cells = tf.contrib.rnn.MultiRNNCell([lstm_cell_1, lstm_cell_2], state_is_tuple=True)
    # Get LSTM cell output
    outputs, states = tf.contrib.rnn.static_rnn(lstm_cells, _X, dtype=tf.float32)
    # Get last time step's output feature for a "many-to-one" style classifier, 
    # as in the image describing RNNs at the top of this page
    lstm_last_output = outputs[-1]
    
    # Linear activation
    return tf.matmul(lstm_last_output, _weights['out']) + _biases['out']
def extract_batch_size(_train, step, batch_size):
    # Function to fetch a "batch_size" amount of data from "(X|y)_train" data. 
    
    shape = list(_train.shape)
    shape[0] = batch_size
    batch_s = np.empty(shape)
    for i in range(batch_size):
        # Loop index
        index = ((step-1)*batch_size + i) % len(_train)
        batch_s[i] = _train[index] 
    return batch_s
def one_hot(y_, n_classes=n_classes):
    # Function to encode neural one-hot output labels from number indexes 
    # e.g.: 
    # one_hot(y_=[[5], [0], [3]], n_classes=6):
    #     return [[0, 0, 0, 0, 0, 1], [1, 0, 0, 0, 0, 0], [0, 0, 0, 1, 0, 0]]
    
    y_ = y_.reshape(len(y_))
    return np.eye(n_classes)[np.array(y_, dtype=np.int32)]  # Returns FLOATS
```
- Get serious and build the neural network
```cpp
# Graph input/output
x = tf.placeholder(tf.float32, [None, n_steps, n_input])
y = tf.placeholder(tf.float32, [None, n_classes])
# Graph weights
weights = {
    'hidden': tf.Variable(tf.random_normal([n_input, n_hidden])), # Hidden layer weights
    'out': tf.Variable(tf.random_normal([n_hidden, n_classes], mean=1.0))
}
biases = {
    'hidden': tf.Variable(tf.random_normal([n_hidden])),
    'out': tf.Variable(tf.random_normal([n_classes]))
}
pred = LSTM_RNN(x, weights, biases)
# Loss, optimizer and evaluation
l2 = lambda_loss_amount * sum(
    tf.nn.l2_loss(tf_var) for tf_var in tf.trainable_variables()
) # L2 loss prevents this overkill neural network to overfit the data
cost = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(labels=y, logits=pred)) + l2 # Softmax loss
optimizer = tf.train.AdamOptimizer(learning_rate=learning_rate).minimize(cost) # Adam Optimizer
correct_pred = tf.equal(tf.argmax(pred,1), tf.argmax(y,1))
accuracy = tf.reduce_mean(tf.cast(correct_pred, tf.float32))
```
- Train the neural network
```cpp
# To keep track of training's performance
test_losses = []
test_accuracies = []
train_losses = []
train_accuracies = []
# Launch the graph
sess = tf.InteractiveSession(config=tf.ConfigProto(log_device_placement=True))
init = tf.global_variables_initializer()
sess.run(init)
# Perform Training steps with "batch_size" amount of example data at each loop
step = 1
while step * batch_size <= training_iters:
    batch_xs =         extract_batch_size(X_train, step, batch_size)
    batch_ys = one_hot(extract_batch_size(y_train, step, batch_size))
    # Fit training using batch data
    _, loss, acc = sess.run(
        [optimizer, cost, accuracy],
        feed_dict={
            x: batch_xs, 
            y: batch_ys
        }
    )
    train_losses.append(loss)
    train_accuracies.append(acc)
    
    # Evaluate network only at some steps for faster training: 
    if (step*batch_size % display_iter == 0) or (step == 1) or (step * batch_size > training_iters):
        
        # To not spam console, show training accuracy/loss in this "if"
        print("Training iter #" + str(step*batch_size) + \
              ":   Batch Loss = " + "{:.6f}".format(loss) + \
              ", Accuracy = {}".format(acc))
        
        # Evaluation on the test set (no learning made here - just evaluation for diagnosis)
        loss, acc = sess.run(
            [cost, accuracy], 
            feed_dict={
                x: X_test,
                y: one_hot(y_test)
            }
        )
        test_losses.append(loss)
        test_accuracies.append(acc)
        print("PERFORMANCE ON TEST SET: " + \
              "Batch Loss = {}".format(loss) + \
              ", Accuracy = {}".format(acc))
    step += 1
print("Optimization Finished!")
# Accuracy for test data
one_hot_predictions, accuracy, final_loss = sess.run(
    [pred, accuracy, cost],
    feed_dict={
        x: X_test,
        y: one_hot(y_test)
    }
)
test_losses.append(final_loss)
test_accuracies.append(accuracy)
print("FINAL RESULT: " + \
      "Batch Loss = {}".format(final_loss) + \
      ", Accuracy = {}".format(accuracy))
```
<img width="600" alt="image" src="https://user-images.githubusercontent.com/87894572/126874388-9cb3948a-4daa-4f91-a6b5-8934a73b4d98.png">
- Training is good, but having visual insight is even better
```cpp
# (Inline plots: )
%matplotlib inline
font = {
    'family' : 'Bitstream Vera Sans',
    'weight' : 'bold',
    'size'   : 18
}
matplotlib.rc('font', **font)
width = 12
height = 12
plt.figure(figsize=(width, height))
indep_train_axis = np.array(range(batch_size, (len(train_losses)+1)*batch_size, batch_size))
plt.plot(indep_train_axis, np.array(train_losses),     "b--", label="Train losses")
plt.plot(indep_train_axis, np.array(train_accuracies), "g--", label="Train accuracies")
indep_test_axis = np.append(
    np.array(range(batch_size, len(test_losses)*display_iter, display_iter)[:-1]),
    [training_iters]
)
plt.plot(indep_test_axis, np.array(test_losses),     "b-", label="Test losses")
plt.plot(indep_test_axis, np.array(test_accuracies), "g-", label="Test accuracies")
plt.title("Training session's progress over iterations")
plt.legend(loc='upper right', shadow=True)
plt.ylabel('Training Progress (Loss or Accuracy values)')
plt.xlabel('Training iteration')
plt.show()
```
- the multi-class confusion matrix and metrics
```cpp
# Results
predictions = one_hot_predictions.argmax(1)
print("Testing Accuracy: {}%".format(100*accuracy))
print("")
print("Precision: {}%".format(100*metrics.precision_score(y_test, predictions, average="weighted")))
print("Recall: {}%".format(100*metrics.recall_score(y_test, predictions, average="weighted")))
print("f1_score: {}%".format(100*metrics.f1_score(y_test, predictions, average="weighted")))
print("")
print("Confusion Matrix:")
confusion_matrix = metrics.confusion_matrix(y_test, predictions)
print(confusion_matrix)
normalised_confusion_matrix = np.array(confusion_matrix, dtype=np.float32)/np.sum(confusion_matrix)*100
print("")
print("Confusion matrix (normalised to % of total test data):")
print(normalised_confusion_matrix)
print("Note: training and testing data is not equally distributed amongst classes, ")
print("so it is normal that more than a 6th of the data is correctly classifier in the last category.")
# Plot Results: 
width = 12
height = 12
plt.figure(figsize=(width, height))
plt.imshow(
    normalised_confusion_matrix, 
    interpolation='nearest', 
    cmap=plt.cm.rainbow
)
plt.title("Confusion matrix \n(normalised to % of total test data)")
plt.colorbar()
tick_marks = np.arange(n_classes)
plt.xticks(tick_marks, LABELS, rotation=90)
plt.yticks(tick_marks, LABELS)
plt.tight_layout()
plt.ylabel('True label')
plt.xlabel('Predicted label')
plt.show()
```

<img width="400" alt="image" src="https://user-images.githubusercontent.com/87894572/126874320-1002051e-a1cc-4198-b093-bf4c9587f2a5.png">

<img width="400" alt="image" src="https://user-images.githubusercontent.com/87894572/126874324-a4bef091-dd4b-4b6a-84e4-a74506528102.png">

### Put realtime 3-axis accelerometer observing result
```cpp
import numpy as np
import tensorflow as tf
from time import sleep
import serial
ser =serial.Serial("COM12", 115200,timeout=0.0835,writeTimeout=1) 
ser.bytesize = serial.EIGHTBITS #number of bits per bytes
ser.parity = serial.PARITY_NONE #set parity check
ser.stopbits = serial.STOPBITS_ONE #number of stop bits
data=['+','0','1','2','-','3','4','5','+','6','7','8']
constitute=[]  
convert=[]
input_X=[]
PredictY=6
j=0
k=0
with tf.Session() as sess:
    new_saver = tf.train.import_meta_graph('./LSTM-Human-Activity-Recognition-master/123/model.meta')
    try:
        new_saver.restore(sess,tf.train.latest_checkpoint('./LSTM-Human-Activity-Recognition-master/123'))
    except ValueError:
        init=tf.global_variables_initializer()
        sess.run(init)
    graph = tf.get_default_graph()
    x = graph.get_tensor_by_name("x:0")
    y = graph.get_tensor_by_name("y:0")
    pred = graph.get_tensor_by_name("out:0")
    
    Arduino_cmd='8'
    cmd=Arduino_cmd.encode("utf-8")
    print(data)
    ser.write(cmd)
    ser.flushInput()
    sleep(0.06)
    
    while(1):
        j=j+1
        
        data=ser.read(12)
        ##print(data)
        if(len(data)!= 12):
            j=j-1
            continue
        for i in range(0,12,4):
            constitute.append(chr(data[i])+chr(data[i+1])+'.'+chr(data[i+2])+chr(data[i+3]))
        for i in range(k,k+3):
            convert.append(float(constitute[i]))
        
        if(j%128==0):
            j=0
            k=-3
            input_X=np.array([convert])
            input_X=input_X.reshape(1,128,3)           
            feed_dict = {x:input_X}
            ##print('PredictY=',np.argmax(sess.run([pred],feed_dict)))
            PredictY=np.argmax(sess.run([pred],feed_dict))
            if(PredictY==0):
                print("WALKING")
            elif(PredictY==1):
                print("WALKING_UPSTAIRS")
            elif(PredictY==2):
                print("WALKING_DOWNSTAIRS")
            elif(PredictY==3):
                print("SITTING")
            elif(PredictY==4):
                print("STANDING")
            elif(PredictY==5):
                print("LAYING")
            else:
                print("Error")
            constitute.clear()
            convert.clear()
        k=k+3
 ```
 <img width="200" alt="image" src="https://user-images.githubusercontent.com/87894572/126874611-0a391347-bdb6-46de-8510-ab14ceec7e63.png">




[0]: ./screen/35334.jpg        "All hardware"
