---
layout: page
title: Viettel Pay Fraud Detection System
permalink: /projects/viettel-pay/
---

## Viettel Pay FDS

**Date:** 2021

**Overview:**
A production-grade, real-time fraud detection system for digital payments that combines deterministic rules with tree-based machine learning to flag suspicious transactions with low latency and fewer false positives. The solution strengthens protection while preserving customer experience across high-volume payment flows.

![Image](/assets/fds/gru-ads-overal.png)
<p style="text-align:center;">GruADS</p>


![Image](/assets/viettelpay/viettel-pay.png)
<p style="text-align:center;">Operational Model</p>




### Background
Digital payments have expanded the attack surface, making proactive fraud prevention essential. Rule-based systems are easy to deploy but often miss emerging patterns and inflate false positives. Tree-based ML models excel on high-dimensional, non-linear data, yet require quality labels and continuous retraining. This project implements a hybrid approach that fuses rules with ML to capture evolving fraud while controlling false alarms.

**Technologies**
- Python, Pandas, Matplotlib, Dask, Java, FastAPI, Apache Kafka, Apache Flink, Apache Spark, PySpark, Hadoop, Docker, MariaDB


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

### Responsibilities and Contributions

- Designed a hybrid rules + ML architecture for low-latency screening.
- Built streaming ingestion and scoring with Kafka/Flink; served models via FastAPI in Docker.
- Engineered time-window aggregates and behavior features for users/devices/merchants.
- Trained and compared Random Forest, XGBoost, and LightGBM with systematic tuning.
- Implemented SHAP-based explanations to support analyst review and decisioning.
- Established drift monitoring, decision thresholds, and a feedback loop for continuous learning.

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
- A scalable, low-latency fraud detection service operating in real time.
- Lower false positives and improved customer trust.
- A feedback-driven framework that adapts to new fraud patterns with minimal manual intervention.