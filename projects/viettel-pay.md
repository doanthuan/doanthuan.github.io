---
layout: page
title: 
permalink: /projects/viettel-pay/
---
<h3></h3>

## Viettel Pay FDS

**Date:** 2021

**Description:**
The project aims to develop a high-performance fraud detection system for real-time payment transactions in banks. This system will enhance the bank's ability to identify suspicious activities, reduce false positives, and prevent fraudulent transactions, ensuring secure and reliable financial services.

![Image](/assets/fds/gru-ads-overal.png)
<p style="text-align:center;">GruADS</p>


![Image](/assets/viettelpay/viettel-pay.png)
<p style="text-align:center;">Operational Model</p>




### Background
Fraudulent activities in banking transactions pose significant risks to both banks and customers. With the rapid rise of digital payments, detecting and preventing fraud has become a top priority. Traditional rule-based systems often struggle to keep up with the evolving nature of fraud, resulting in higher false positives or missing new patterns of fraudulent behavior, however it's easy to implement. Machine learning models, particularly tree-based algorithms, are well-suited for handling high-dimensional, non-linear data, making them an ideal choice for fraud detectio, however it requires high-quality labeled data and constant retraining to keep up with evolving fraud tactics. Our approach is hybrid, it combines rule-based systems with machine learning models to detect fraud more effectively.

**Technologies**
- Python, Pandas, matplotlib, dask, Java, FastAPI, Apache Kafka, Apache Flink, Apache Spark, PySpark, Hadoop, Docker, MariaDB


### Data

- **Data Collection**: Collect historical transactional data, including features like transaction amount, timestamp, location, customer behavior, and device information.
- **Class Imbalance Handling**: Use techniques such as Synthetic Minority Over-sampling Technique (SMOTE) or undersampling to address class imbalance.
- **Feature Engineering**: Extract meaningful patterns, including time-based features, user-specific behaviors, and transaction relationships.
- **Data Cleaning**: Handle missing values, outliers, and standardize/normalize feature scales.

![Image](/assets/viettelpay/data1.png)

![Image](/assets/viettelpay/data2.png)

![Image](/assets/viettelpay/data3.png)

![Image](/assets/viettelpay/data4.png)

![Image](/assets/viettelpay/data5.png)

![Image](/assets/viettelpay/data6.png)

![Image](/assets/viettelpay/data7.png)

### Model Development

- **Model Selection**: Train and compare various tree-based models, including:
  - **Random Forest**: An ensemble of decision trees that works well with large, unbalanced datasets and reduces overfitting.
  - **XGBoost**: A powerful gradient boosting algorithm known for its high accuracy and performance on structured data.
  - **LightGBM**: A gradient boosting framework that efficiently handles large datasets, ideal for real-time fraud detection.
- **Model Interpretability**: Use feature importance and SHAP values to interpret model predictions and understand key fraud indicators.
- **Hyperparameter Tuning**: Optimize model performance by tuning hyperparameters such as the number of trees, tree depth, and learning rate using cross-validation.

![Image](/assets/viettelpay/model1.png)

![Image](/assets/viettelpay/model2.png)

![Image](/assets/viettelpay/model3.png)

![Image](/assets/viettelpay/model4.png)

### Evaluation
- **Model Evaluation**: Evaluate models using metrics such as precision, recall, F1-score, and AUC-ROC, focusing on minimizing false positives and maximizing fraud detection.
- **Performance Monitoring**: Monitor model performance to detect drift and trigger retraining as necessary.
- **Fraud Simulation**: Test model robustness by simulating various fraud scenarios using synthetic data.

![Image](/assets/viettelpay/error1.png)

![Image](/assets/viettelpay/error2.png)

### Challenges
- **Class Imbalance**: Fraudulent transactions represent a small percentage of total transactions, making it difficult to model.
- **Evolving Fraud Techniques**: Fraud tactics may change over time, requiring the model to adapt continuously.
- **False Positives**: Balancing detection sensitivity with false-positive rates is critical to avoid disrupting normal customer activities.

#### Real-Time Integration and Deployment
- **Real-Time Pipeline**: Develop a pipeline for real-time fraud detection where new transactions are classified immediately by the model.
- **Threshold-Based Decision-Making**: Implement decision thresholds to classify transactions as “fraudulent,” “suspicious,” or “normal.”
- **Continuous Learning**: Establish a feedback loop for updating the model with new data, allowing it to adapt to evolving fraud patterns over time.

### Explainable AI (SHAP)

![Image](/assets/viettelpay/xai1.png)

![Image](/assets/viettelpay/xai2.png)

![Image](/assets/viettelpay/xai3.png)

![Image](/assets/viettelpay/xai4.png)


### Expected Outcomes
- A scalable, accurate, and interpretable fraud detection model capable of identifying fraudulent transactions in real-time.
- Reduced financial loss due to fraud and improved customer trust through secure payment systems.
- A continuously evolving fraud detection framework that adapts to new patterns and reduces manual intervention.