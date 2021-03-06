What is DeepMetabolism?
--------------------------

DeepMetabolism is a deep learning algorithm that predicts cell phenotypes from genome sequencing data such as transcriptomics data. DeepMetabolism uses biological knowledge to design a neural network model and integrates unsupervised learning with supervised learning for model prediction. DeepMetabolism meets the design criteria of high accuracy, high speed, and high robustness. By doing so, DeepMetabolism can be used to enrich our understanding of genotype-phenotype correlation and be applied in fields of synthetic biology, metabolic engineering and precision medicine. 

The general design principles are:
* Interactions in biological networks can be simulated by a a computational network, represented as a graph model.
* Unsupervised machine learning techniques (such as autoencoder) can be used to train the computational network to approximate the biological network.
* The whole process is end-to-end, without tweaking between layers.

To develop DeepMetabolism, we implemented a two-step, deep-learning process that used different sets of data. The first step was unsupervised learning that used transcriptomics data, and the second step was supervised learning that used paired data of transcriptomics and phenotype (Figure 1). We expected that unsupervised learning would provide a “rough” neural network model that captured the essence of connections between transcriptomics data and phenotype, while supervised training would fine tune this model and lead to highly accurate predictions of phenotype from transcriptomics data.

![overview_deepmetabolism](https://github.com/gwh120104/deepmetabolism/blob/master/img/Figure_README.png)
Figure 1. Overview of DeepMetabolism algorithm

#### How to cite DeepMetabolism?
DeepMetabolism: A Deep Learning System To Predict Phenotype From Genome Sequencing [[bioRXiv]](http://biorxiv.org/content/early/2017/05/09/135574) [[arXiv]](https://arxiv.org/abs/1705.03094)

DeepMetabolism model
----------------------------------

For unsupervised learning of DeepMetabolism, we use a vanilla autoencoder, where the input is transcriptomics data (i.e., log2 fold changes of gene expression levels), to implement unsupervised learning process. We try to reconstruct gene expression level by going through the protein layer and the phenotype layer. Protein and phenotype layers are feed-forward layers (no recurrent or backward connections). The overall architecture of the autoencoder is:

<kbd>Gene</kbd> -> <kbd>Protein</kbd> -> <kbd>Phenotype</kbd> -> <kbd>Protein Reconstructed</kbd> -> <kbd> Gene Reconstructed</kbd>

The autoencoder model is defined in `autoencoder.py`. Note that the layers of the autoencoder model were not fully connected. Instead, we applied biological knowledge to rationally define the connections, which could reduce the risk of over-parameterization and increase the calculation speed. To connect the first layer (i.e., gene layer) and the second layer (i.e., protein layer), we applied the gene-protein association from well-developed, constraint-based reconstructed metabolic models. We recommand to download the high-quality constraint-based reconstructed metabolic models from [BiGG model database](bigg.ucsd.edu). To connect the second layer (i.e., protein layer) and the third layer (i.e., phenotype layer), we applied [cobrapy package](https://github.com/opencobra/cobrapy) to the well-developed constraint-based reconstructed metabolic model to identify the proteins that were essential for each phenotype (e.g., proteins that were essential for cell growth) and connected these proteins with the corresponding phenotype. 

When defining the model parameters, we pad the parameters such that weights between layers are defined by a 2-dimensional `[OUT, IN]` tensor `W` such that `W * v_in = v_out`. We mask `W` with masking matrix `M` such that `W * M * v_in = v_out` where `M` is a 0/1 matrix. `M` between each two layers can be obtained from `cobrapy_model.py`.

After unsupervised learning, we next designed the supervised learning with paired data of transcriptomics and phenotype, and used the same autoencoder model with the first three layers. 

<kbd>Gene</kbd> -> <kbd>Protein</kbd> -> <kbd>Phenotype</kbd>

Note that, due to the limited types of the available phenotype in the paired data, we need to pinpoint the indices of the phenotypes with paried training data in the phenotype layer (positions of the phenotype nodes with training data). The phenotype indices can be generated from `cobrapy_model.py`, too. 

DeepMetabolism usage
-------------------------------

### Input data format (including transcriptomics data for unsupervised learning and paired data for supervised learning)
Input files are required to be delimited by ',' (csv format). In the input files, each row represents one dataset (one measurement from practical experiments), and each column represents one node (e.g., one gene, or one phenotype). The paired data for supervised learning is paired based on the order of each row. Namely, the i<sup>th</sup> row in the transcriptomics input file is paired with i<sup>th</sup> row in the phenotype input file. In addition, the order of each column should follwo the difinition of each node in the input layer or the output layer (i.e., phenotype indices). 

### Model construction
DeepMetabolism uses masking matrices to define the connections between each two layers. The masking matrices can be automatically generated by `cobrapy_model.py` from a well-developed contraint-based reconstructed metabolic model written in SBML format. In addition, the `phenotype_indices_file` will be generated based on your available phenotype paired data. All the configurations of `cobrapy_model.py` is available in `model_preparation` module in `config.py`.

* Generate masking matrices `M`, and the phenotype indices `phenotype_indices_file` from a contraint-based reconstructed metabolic model
> python cobrapy_model.py

### Model training and validing
DeepMetabolism is designed to run on the GPU based on [TensorFlow](https://www.tensorflow.org/). But you can run it on either GPU and CPU. It is important to note that, if you run DeepMetabolism on CPU (i.e., skip the next step), the running speed will be dramatically decreased. Similarly, all the configurations of `train_auto_encoder.py` is available in `data_process`, `model_construct`, `unsupervised`, and `supervised` modules in `config.py`. The trained DeepMetabolism model will be saved in `ckpt` format to customized directory.

* Activate the virtual environment to use GPU (optional)
> source ~/tf/bin/activate

* Run the training code (including both unsupervised learning and supervised learning processes with 10-fold cross validation)
> python train_auto_encoder.py

### Training and validation outputs
There are two types of output results from the `train_auto_encoder.py`. For the unsupervised learning process, we apply the final trained autoencoder model to reconstruct the input transcriptomics data, and compare the input (first column) and reconstructed (second column), i.e., `unsupervised_training_result`. For the supervised learning process, we implement 10-fold cross validation and print the results to the result file, i.e., `supervised_training_result`. If you have n types of phenotypes, your result file will have 2n columns. The first n columns (the first half, 1:n) in your result file will represent the corresponding observed values of the n types of phenotypes, and the other half columns (n+1:2n) will represent the corresponding predicted values of the n types of phenotypes. The results from each fold will be stacked by rows.

### Prediction from a DeepMetabolism model
We also provide the code to use well-trained DeepMetabolism model for predicting phenotypes from transcriptomics data. By loading the saved model from `train_auto_encoder.py`, you can use `predict_auto_encoder.py` to predict phenotypes from your transcriptomics data via `model_input` module in configuration file `config.py`. It is important to note that the `config.py` file is currently written for the `toy_model` in this repository.

### A toy model example for central metabolism of *E. coli*:

The toy model for DeepMetabolism is generated from the metabolic model [e_coli_core](http://bigg.ucsd.edu/static/models/e_coli_core.xml.gz), including 136 genes (excluding all the pseudo-genes) as inputs and 1 phenotype (i.e., growth rate). Therefore, there are 136 nodes in gene and protein layers and 1 node in the phenotype layer. The masking matrices defining the model connection are available in `toy_model/toy_gene_pro_rule.csv` and `toy_model/toy_pro_pheno_rule.csv`, generated from `cobrapy_model.py`. All the configuration items for this `toy_model` are listed in `config.py`. There are 136 "one_to_one" connections between gene and protein layers, and 136 "all_to_one" connections between protein and phenotype layers. __You can follow the following commands to test DeepMetabolism with our toy model.__

* Generate masking matrices `toy_model/toy_gene_pro_rule.csv` and `toy_model/toy_pro_pheno_rule.csv` for 
>python cobrapy_model.py

* Activate the virtual environment `tf` to use GPU
> source ~/tf/bin/activate

* Run the training and validation code for `toy model`
> python train_auto_encoder.py

* Check your results with standard results
> python check_results.py result_type your_results.csv

### A genome model for *E. coli*: iJO1366DM model as the demo for DeepMetabolism.

If you want to run a larger-scale model, we provide the iJO1366DM model as a demo. You can easily change the inputs (including the constraint-based reconstructed model, available phenotype names, and all the input data) in the `config.py`, and follow the above instructions for the toy model to run this genome scale. All the materials related to iJO1366DM model is available [here](https://www.dropbox.com/s/7kaw8m7ozp3liyc/iJO1366_demo.tar.gz?dl=0).
