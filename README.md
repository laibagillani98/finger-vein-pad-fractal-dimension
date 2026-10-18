# Finger-Vein Presentation Attack Detection Using Fractal Dimension

## 1. Project Overview

This project explores **Fractal Dimension (FD)** as a feature-extraction method for Finger-Vein Presentation Attack Detection (PAD).

The objective is to distinguish between:

- **Genuine images (Label 0):** Images captured from real fingers.
- **Spoofed images (Label 1):** Presentation attack images.

I extract numerical features from finger-vein images and use a **k-Nearest Neighbors (k-NN)** classifier with `k = 3` to classify them.

This repository documents my experiments, observations, and the reasons for selecting the current feature-extraction method.

## 2. Dataset

The current experiments use the **PLUS finger-vein dataset**.

| Description | Count |
|---|---:|
| Selected genuine images | 393 |
| Selected spoofed images | 393 |
| Total images | 786 |
| Unique subjects | 22 |

To construct a balanced dataset, I selected genuine images corresponding to available spoofed images using matching filenames.

The original biometric image datasets are not included in this repository.

## 3. What Is Fractal Dimension?

Fractal Dimension is a numerical measure used to describe the complexity of a pattern.

In image processing, it can help describe texture and structural variations.

For this project, I experimented with three approaches to creating Fractal Dimension feature vectors.

---

## 4. Approach 1 — Fractal Dimension Using Five Grayscale Thresholds

**Method:** Binary Box Counting

### How it works

1. Load a finger-vein image in grayscale.
2. Apply contrast enhancement.
3. Create five binary images using thresholds of `60, 90, 120, 150, 180`.
4. Apply Binary Box Counting separately to each binary image.
5. Combine the five estimated Fractal Dimension values into one feature vector.

### Example feature vector

```text
[1.6496, 1.8572, 1.9158, 1.9581, 1.9764]
```

**Feature vector length: 5**

### Observation

This method produced a very low preliminary classification error, approximately **0.28% ACER** in the earlier five-fold experiment.

However, I noticed substantial brightness differences between genuine and spoofed images. Because the method uses fixed intensity thresholds, classification performance may have been influenced by brightness rather than only finger-vein texture.

### Decision

I kept this approach as an initial experiment and explored a region-based alternative.

---

## 5. Approach 2 — Region-Based Fractal Dimension Using Edge Detection

**Method:** Canny Edge Detection + Binary Box Counting

### How it works

1. Load the finger-vein image.
2. Apply preprocessing and Canny edge detection.
3. Divide the image into a **3×3 grid**.
4. Calculate the Fractal Dimension of the detected edges in each region.
5. Combine the nine regional values into one feature vector.

**Intended feature vector length: 9**

### Observation

Some genuine finger-vein images contained very few detected edges in particular regions.

As a result, Fractal Dimension calculation sometimes produced invalid (`NaN`) values.

For example, certain regions contained no detected edge pixels, making the calculation unreliable in the current implementation.

### Decision

I did not continue with this approach because it could not consistently produce valid feature vectors without additional handling or preprocessing.

---

## 6. Approach 3 — Region-Based Differential Box Counting (DBC)

**Method:** Grayscale Differential Box Counting

**Current selected approach**

### Why I chose this approach

Unlike the previous edge-based approach, Differential Box Counting works directly with grayscale intensity variations.

I wanted to extract texture-related information from different parts of the finger-vein image without relying on binary edge detection.

### How it works

1. Load the image in grayscale.
2. Crop approximately 10% from each horizontal side to reduce border interference.
3. Apply CLAHE contrast enhancement.
4. Divide the processed image into a **3×3 grid**.
5. Apply the implemented Differential Box Counting estimator to each region.
6. Combine the nine estimates into a feature vector.

### Example feature vector

```text
[2.0602, 2.1036, 2.1074,
 2.0590, 2.1097, 2.0627,
 2.0827, 2.1062, 2.0888]
```

**Feature vector length: 9**

Each value represents the estimated grayscale texture complexity of one image region.

### Observation

The current implementation successfully extracted nine finite features from all 786 selected PLUS images.

The resulting feature matrix has the shape:

```text
X = (786, 9)
y = (786,)
```

Where:

- `X` contains the nine extracted features for each image.
- `y` contains the corresponding genuine or spoofed label.

**Important:** The implementation uses a simplified Differential Box Counting estimator. Its mathematical formulation and sensitivity to image characteristics still require validation before drawing final scientific conclusions.

---

## 7. Comparison of the Three Approaches

| Property | Approach 1 | Approach 2 | Approach 3 |
|---|---|---|---|
| Method | Binary Box Counting | Edge Detection + Binary Box Counting | Differential Box Counting |
| Image representation | Five binary images | Binary edge images | Grayscale images |
| Regional division | No | 3×3 | 3×3 |
| Number of features | 5 | 9 intended | 9 |
| Main observation | Possible brightness influence | Invalid features in some regions | Finite features for all selected images |
| Current status | Initial experiment | Not continued | Selected for baseline |

Approaches 1 and 2 use the same underlying Box Counting method but differ in their preprocessing and feature construction.

Approach 3 uses a different Fractal Dimension estimator.

## 8. Classification Using k-Nearest Neighbors

After extracting the features, I use a k-NN classifier with **k = 3**.

### Classification process

1. Extract nine features from each image.
2. Assign labels to training images:
   - `0` = Genuine
   - `1` = Spoofed
3. Standardize the features using `StandardScaler`.
4. Train the k-NN classifier on the labeled training feature vectors.
5. For each test image, find the three nearest training feature vectors.
6. Predict the class using the majority label of those three neighbors.

For example, if the three nearest training examples have labels `[0, 1, 0]`, the predicted label is `0` (Genuine).

The test image's true label is used for evaluation, not for selecting its neighbors.

## 9. Five-Fold Cross-Validation

I use **five-fold subject-disjoint cross-validation** to evaluate the classifier.

Images belonging to the same subject must not appear in both training and testing data within a fold.

| Fold | Training subjects | Testing subjects | Subject overlap |
|---|---:|---:|---:|
| 1 | 17 | 5 | 0 |
| 2 | 17 | 5 | 0 |
| 3 | 18 | 4 | 0 |
| 4 | 18 | 4 | 0 |
| 5 | 18 | 4 | 0 |

This allows the model to be evaluated on previously unseen subjects.

## 10. Evaluation Metrics

I use three metrics to evaluate PAD classification.

**APCER (Attack Presentation Classification Error Rate)**

The proportion of spoofed images incorrectly classified as genuine.

**BPCER (Bona Fide Presentation Classification Error Rate)**

The proportion of genuine images incorrectly classified as spoofed.

**ACER (Average Classification Error Rate)**

The average of APCER and BPCER:

```text
ACER = (APCER + BPCER) / 2
```

Lower values indicate better classification performance.

## 11. Preliminary Results — Regional DBC + k-NN

| Fold | APCER | BPCER | ACER |
|---|---:|---:|---:|
| 1 | 7.78% | 22.22% | 15.00% |
| 2 | 6.90% | 20.69% | 13.79% |
| 3 | 1.39% | 23.61% | 12.50% |
| 4 | 5.56% | 23.61% | 14.58% |
| 5 | 2.78% | 5.56% | 4.17% |
| **Mean** | **4.88%** | **19.14%** | **12.01%** |

The values above are preliminary fold-wise results from the current implementation.

### My observations

- The average ACER was approximately **12.01%**.
- The classifier identified spoofed images more reliably than genuine images in this experiment.
- BPCER was higher than APCER, indicating that genuine images were more frequently classified incorrectly.
- Performance varied between subject folds.
- Fold 5 achieved the lowest ACER, while Fold 1 had the highest ACER.
- No subjects overlapped between training and testing within any fold.

### Interpretation

Regional DBC produced usable numerical feature vectors for classification, but the current classifier still makes a noticeable number of errors on genuine images.

The results also show that successfully calculating features does not necessarily mean those features provide optimal classification performance.

Additional experiments and method validation are needed.

## 12. Current Progress

**Completed:**

- Selected and balanced the PLUS image subset.
- Implemented five-threshold Binary Box Counting.
- Experimented with edge-based regional Box Counting.
- Implemented regional Differential Box Counting.
- Extracted nine DBC features from 786 images.
- Assigned genuine and spoofed labels.
- Implemented k-NN with k = 3.
- Evaluated the DBC baseline with five-fold subject-disjoint cross-validation.
- Created feature-vector and nearest-neighbor demonstration code.

**Pending:**

- Validate the simplified DBC estimator.
- Evaluate DBC with reduced real training datasets (2/5 and 1/5).
- Complete experiments with IDIAP and SCUT datasets.
- Add synthetic-data training experiments.
- Compare experimental configurations and prepare the final results.

## 13. Conclusion

I explored three feature-extraction approaches to understand how Fractal Dimension can be used for finger-vein presentation attack detection.

The first approach used intensity thresholds and Binary Box Counting. The second used edge detection and regional Box Counting but encountered invalid feature values.

The third approach used regional Differential Box Counting and successfully created nine features for every selected image.

I selected regional DBC for the current baseline experiment and evaluated it using k-NN and five-fold subject-disjoint cross-validation.

The current average ACER is approximately **12.01%**, and further work will investigate the reliability of the features and their performance under different training conditions.

---

*University image-processing project — Work in progress.*
