# Content-Based Image Retrieval Syetem
## Project Objectives
* Extract keypoint detectors and local invariant descriptors of each image in the dataset and store them in HDF5.
* Cluster the extracted features in HDF5 to form a codebook (resulting centroids of each clustered futures) and visualize each codeword (the centroid) inside the codebook.
* Construct a bag-of-visual-words (BOVW) representation for each image by quantizing the associated feature vectors into histogram using the codebook created.
* Accept a query image from the user, construct the BOVW representation for the query, and perform the actual search.
* Implement term frequency-inverse document frequency and spatial verification to improve the accuracy of the system.

## Software/Package Used
* Python 3.5
* [OpenCV](https://docs.opencv.org/3.4.1/) 3.4
* [Imutils](https://github.com/jrosebr1/imutils)
* [Scikit-Learn](http://scikit-learn.org/stable/)
* [HDF5](https://www.h5py.org/)
* [redis](https://redis.io/)

## Algorithms & Methods Involved
* Keypoints and descriptors extraction
  * Fast Hessian keypoint detector algorithms
  * Local scale-invariant feature descriptors (RootSIFT)
* Feature storage and indexing
  * Structure HDF5 dataset
* Clustering features to generate a codebook
  * K-means algorithms
* Visualizing codeword entries (centroids of clustered features)
* Vector quantization
  * BOVW extraction
  * BOVW storage and indexing
* Inverted indexing
  * Implement redis for inverted index
* Search performing
* System accuracy evaluation
  * "Points-based" metric
* Term frequency-inverse document frequency
* Spatial verification

## Approaches
* The dataset is obtained from 1000 images inside [UKBench](https://archive.org/details/ukbench) dataset.
* The figure below shows the CBIR search pipelines.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/cbir_searching.jpg" width="600">

## Results
### Extract keypoints and descriptors
Using following command will store the keypoint detectors and local invariant descriptors of each image in HDF5. We will have a `features.hdf5` file shown below.
```
python index_features.py --dataset ukbench --features_db output/features.hdf5
```

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/hdf5_database.png" width="200">

The following picture shows the interior structure inside HDF5 file:

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/hdf_database_layout.png" width="500">

The `image_ids` dataset has shape (X,) where X is total number of examples in dataset (In this case, X = 1000). And `image_ids` is corresponding to the filename.

The `index` dataset has shape (X, 2) and stores two integers, indicating indexes into `features` dataset for image i.

The `features` dataset has shape (Y, 130), where Y is the total number of feature vectors extracted from X images in the dataset. First two columns are the (x, y)-coordinates of the keypoint associated with the feature vector. The other 128 columns are from RootSIFT feature vectors.

### Cluster features
Using following command will cluster the features inside HDF5 file to generate a codebook. The clustered features will store inside pickle file.
```
python cluster_features.py --features_db output/features.hdf5 --codebook output/vocab.cpickle --clusters 1536 --percentage 0.25
```

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/clustered_features.png" width="200">

### Visualize features
Using following command will create a visualization on each codeword inside codebooks (each centroid of clustered features).
```
python visualize_centers.py --dataset ukbench --features_db output/features.hdf5 --codebook output/vocab.cpickle --output output/vw_vis
```
This process takes about 60 - 90 mins to finish depend on the computers.

Here are a few samples (grayscale) of visualizing the features.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/vis_sample1.jpg" width="300"> <img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/vis_sample2.jpg" width="300">

Figure 1: Book-title features (left), Leaves-of-tree features (right).

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/vis_sample3.jpg" width="300"> <img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/vis_sample4.jpg" width="300">

Figure 2: Detailed grass features (left),  Car-light features (right).

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/vis_sample5.jpg" width="300"> <img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/vis_sample6.jpg" width="300">

Figure 3: Store-logo features (left), car-dashboard features (right).

### Vector quantization (Forming a BOVW)
Using following command will create a BOVW representation for each image by quantizing the associated features into histogram. Comparing to `features.hdf5` file in [previous](#extract-keypoints-and-descriptors) part, `bovw.hdf5` has much smaller size which only has 12.4 MB, as the figure shown below.  

```
python extract_bovw.py --features_db output/features.hdf5 --codebook output/vocab.cpickle --bovw_db output/bovw.hdf5 --idf output/idf.cpickle
```
<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/bovw_database.png" width="200">

### Inverted indexing
Using following command while making sure that redis server is on will build a corresponding inverted index.
```
python build_redis_index.py --bovw_db output/bovw.hdf5
```
Using redis not only ensure that we don’t have to perform an exhaustive linear search over all images in our dataset, it also speeds up the querying process, allowing our CBIR system to scale to millions of images rather than limit it to only a few thousand.

The redis database is using only about 1.9 MB of RAM to store 1000 images by checking the memory process.

### Search performance
Using following command will start the search engine to return 20 closest images to the query image you choose.
```
python search.py --dataset ukbench --features_db output/features.hdf5 --bovw_db output/bovw.hdf5 --codebook output/vocab.cpickle --relevant ukbench/relevant.json --query ukbench/ukbench00364.jpg
```

In the UKBench dataset, since every subject has 4 relevant images with different viewpoints, the best performance will have top 4 images which are relevant to the query image and the worst will have none relevant to the query image. The top 20 results from search engine will be displayed. The displayed images has been resized for displaying purposes only.

Here are two samples that top 4 results are all relevant to the query image.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/performance_sample1.png" width="800">

Figure 5: Query image ID: 364, search took: 0.91s.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/performance_sample2.png" width="800">

Figure 6: Query image ID: 697, search took: 1.07s.

Here is a sample that top 3 results are relevant to the query image.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/performance_sample3.png" width="800">

Figure 6: Query image ID: 819, search took: 1.67s.

Here is a sample that top 2 results are relevant to the query image.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/performance_sample4.png" width="800">

Figure 6: Query image ID: 788, search took: 1.75s.

Here is a sample that only 1 results are relevant to the query image.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/performance_sample5.png" width="800">

Figure 6: Query image ID: 333, search took: 0.64s.

### Evaluation
A point-based metric is implemented for evaluation. The system will get 1 point for each correct result in top-4 results. For example:
  * The system will receive 4 points if all four relevant images are in the top-4 results.
  * 3 points for three relevant images are in the top-4 results.
  * 2 points for two relevant images are in the top-4 results.
  * 1 point for only one relevant image in the top-4 results.
  * And 0 point for no relevant images in the top-4 results.

This scoring scheme is referenced from Nistér and Stewénius in their 2006 paper, *Scalable recognition with a vocabulary tree*. According to their paper, a score ≥ 3 should be considered to be good, implying that on average, across all images in our dataset, we should be able to find at least three relevant images in the top-4 results.

The figure below shows the evaluation results. The u stands for average and o stands for standard deviation.

<img src="https://github.com/meng1994412/CBIR/blob/master/image_search_engine/output/performance_evaluation.png" width="400">
