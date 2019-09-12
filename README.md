# Compacting, Picking and Growing (CPG)

This is an experimental Pytorch implementation of CPG - a lifelong learning algorithm for object classification. For details about CPG please refer to the paper ***CPG*** by ......

---

## Experiment1 (Compact 20 tasks into VGG16 network)
- Datasets: Cifar100. We splited 100 classes into 20 tasks. For each task, there are 5 new classes to learn.

#### Demo

`Step 1. Download Cifar100 and organize the folders to Pytorch data loading format`

- use [cifar2png](https://github.com/knjcode/cifar2png) tool.
```
$ cifar2png cifar100superclass path/to/cifar100png
```
  
`Step 2. Fill the dataset path into dataset.py`


- In cifar100_train_loader function
```
    train_dataset = datasets.ImageFolder('path_to_train_folder/{}'.format(dataset_name),
            train_transform)
```
- In cifar100_val_loader function 
```
val_dataset = \
        datasets.ImageFolder('path_to_test_folder/{}'.format(
                dataset_name),
                transforms.Compose([
                    transforms.ToTensor(),
                    normalize,
                ]))
```

`Step 3. Run baseline_cifar100.sh or finetune_cifar100_normal.sh in experiement1 folder`
 
There are one important arg in baseline_cifar100.sh or finetune_cifar100_normal.sh:
- logfile: it is used to save baseline accuracy of current task to json file.

```
$ bash experiment1/baseline_cifar100.sh
```
or
```
$ bash experiment1/finetune_cifar100_normal.sh
```

After this step, we will get the accuracy goal for each task in the json file we specify.

`Step 4. Run CPG algorithm to learn 20 tasks in sequential`
- args:
  - baseline_cifar100_acc: it must be the same path as logfile in baesline_cifar100.sh, used to                          load the accuracy goal for CPG algorithm to chase. 
```
$ bash experiment1/CPG_cifar100_scratch_mul_1.5.sh
```

`Step 5. Inference`
```
$ bash experiment1/inference_CPG_cifar100_result.sh
```

---

## Experiment2 (Compact 6 tasks into VGG16/ResNet50 network)

`Step 1. Download multiple datasets`

- refer to [piggyback](https://github.com/arunmallya/piggyback)

`Step 2. Adjust set_dataset_paths function in utils.py`

`Step 3. Manually write each task's accuracy goal into json file`

#### For example, for VGG16, I will create logs/baseline_imagenet_acc_custom_vgg.txt
, and write accuracy goal according to [piggyback](https://arxiv.org/pdf/1801.06519.pdf) paper. 
```
{"imagenet": "0.7336", "cubs_cropped": "0.8023", "stanford_cars_cropped": "0.9059", "flowers": "0.9545", "wikiart": "0.7332", "sketches": "0.7808"}
```

`Step 4. Run CPG_imagenet_vgg.sh each time for each task and manually choose the best pruning ratio`

For example, I will first gradually prune the first task (ImageNet) by setting 

line 49,
```
In CPG_imagenet_vgg.sh

line 49: for task_id in `seq 1 1`; do
```

- Note: For first task (ImageNet), I set GPU_ID=0,1,2,3,4,5,6,7, in order to use 8 GPUs to accelerate pruning procedure. But for other task, I still use 1 gpu to avoid the loss of accuracy.

After gradually pruning, I will check the record.txt file in the checkpoint path,
in my case, it is ```checkpoints/CPG/custom_vgg/imagenet/gradual_prune/record.txt```
and copy the checkpoint file from appropriate pruning ratio folder to gradual_prune folder.

#### for example, original there will be folders named 0.1, 0.2, 0.3 ... 0.95 in checkpoint/CPG/custom_vgg/imagenet/gradual_prune/ folder, and according to the record.txt inside it, I fould that 0.4 is the best pruning ratio (It might be true, if 0.4 has the checkpoint which accuracy > 73.36, but 0.5 has the checkpoint which accuracy < 73.36)

Thus, I copied the checkpoint from 0.4 folder to its upper folder
```
In checkpoint/CPG/custom_vgg/imagenet/gradual_prune/

$ cp 0.4/checkpoint-46.pth.tar ./checkpoint-46.pth.tar
```

Now it's time to add second task by change line 49 in CPG_imagenet_vgg.sh
line 49,
```
In CPG_imagenet_vgg.sh

line 49: for task_id in `seq 2 2`; do
```
Then we repeat the check procedure to the second task, we check ```checkpoints/CPG/custom_vgg/cubs_cropped/gradual_prune/record.txt``` and copy the appropriate checkpoint with best pruning ratio to the upper folder, and again to the third, fourth tasks, ...

