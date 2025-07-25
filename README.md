# customer_repurchase_prediction

## Description:
"The Look" is a fictional e-commerce business specializing in clothing and accessories, with a consolidated digital presence through multi-channel strategies, online marketing, and efficient logistics supported by multiple distribution centers.

We've been asked for a predictive model to help identify customers with a high probability of repurchasing in the next 60 days. (A "repurchase" is considered when a customer has made at least one purchase in the 60 days following their first recorded purchase.)


## 📁 Project Structure
```text
├── data/ # Raw, processed and analysis datasets for diagnosing the need of a predictive model
│ ├── analysis/
│ ├── raw/
│ └── processed/
├── graphs&presentation/ # graphs and executive presentation
│ └── main.py
├── modelos/ # Saved .pkl models
├── notebooks/
│ └── EDA+preprocessing.ipynb # EDA and data preprocessing
│ ├── graficos.ipynb # graphs for ppts and analysis
│ └── training.ipynb # model training and calibration for probabilities usage
├── sql/ # preprocessing and analysis querys to get the training data 
├── requirements.txt
└── README.md
```
## 🚀 How to Run the Project

### 🔧 Install Requirements

```bash
pip install -r requirements.txt

```
### 🧪 Run the Model
download the .pkl model, calibrated if will be used for probabilities, otherwise catboost, from [pickle model](https://drive.google.com/drive/folders/168H3YE6BwK2fZy_NXEJYEa3Q5WWgYdKE?usp=sharing) or retrain the model with the data from [train & test data](https://drive.google.com/drive/folders/1u5hVVOqiszFm7tSr34JFQFmJXSutvvZg?usp=sharing) and run the model.

## 🧹 Preprocessing Steps
- Drop duplicates
- Handle missing values
- Encode categorical variables (WOE and target encoding)
- Standardize numerical features
- Feature engineering (PCA of high correlated values)

## 🧠 Model Overview
Catboost then probability calibration

Evaluation:
- ROC-AUC
- Confusion Matrix
- precision-recall curve

## 📈 Results
| Model              | ROC_AUC   | Recall | Accuracy | F1-score |
| ------------------ | --------- | ------ | -------- | -------- |
| catboost           | 0.851     | 0.53   |   0.83   |   0.59   |

Our model can predict the probability of repurchase in the next 60 days, in thi case, we can 
## 🛠 In Production

Models saved as .pkl file

get preprocessing from sql queries:
1. Run "successful repurchases and purchases" and save as "repurchase_purchase_sucessful" to get all orders with repurchase in next 60 days and all orders with purchase.
2. Run "order, user and product info for modeling" and save as "order_user_product_data" this will get user and product information from all repurchased and purchased orders. 
3. Run "user behaviour from orders" and save as "user_order_behaviour", this will get the behaviour variables (quantity of orders cancelled, returned by the user, days between orders).
4. Run "user exposure", this will get the information from the traffic in the ecommerce.
5. Run "training data" to get the training data, in this project it's saved at data/raw.

A simplified ERD of the tables is presented to get a better idea of the data and the relationships.

```bash
+------------------+      +---------------------+
|      Users       |<-----|       Events        |
+------------------+      +---------------------+
| id (PK)          |      | user_id (FK)        |
| age              |      | event_type          |
| gender           |      | created_at          |
| traffic_source   |      | traffic_source      |
+------------------+      +---------------------+

     |
     | 1
     |-----< (user_id)
     |
     V

+------------------+      +----------------------+
|     Orders       |<-----|    Repurchases       |
+------------------+      +----------------------+
| order_id (PK)    |      | user_id (FK)         |
| user_id (FK)     |      | original_order (FK)  |
| created_at       |      | repurchase_order     |
| status           |      | repurchase_date      |
| num_of_item      |      | days_between         |
+------------------+      +----------------------+

     |
     | 1
     |-----< (order_id)
     |
     V

+----------------------+     +---------------------+
|    Order_Items       |<----|      Products       |
+----------------------+     +---------------------+
| order_id (FK)        |     | id (PK)             |
| product_id (FK)      |     | name, brand         |
| sale_price           |     | category, department|
| inventory_item_id    |     | retail_price        |
+----------------------+     | distribution_center_id (FK)
                             +---------------------+
                                       |
                                       V
                         +-----------------------------+
                         |    Distribution_Centers     |
                         +-----------------------------+
                         | id (PK)                     |
                         | name                        |
                         +-----------------------------+
```

In order to deploy this model into production the process above should be automated with an orchestrator like airflow and an architecture is proposed:

![alt text](graphs&presentation/neoConsulting.png)

## 📦 Dependencies
See requirements.txt for full list. Key packages:
- scikit-learn
- pandas
- catboost

## 📌 Further recommendations:
- Integrate with cloud solution (reading from Athena or redshift and run it in ECS)
- further analysis for repurchase, why is my current organic traffic decresing? Now, my customers tend o buy more products by order, what can i do with this information?
- add MLFlow tracking and maybe a FastAPI integration
- meet with business related people to add knowledge to engineer better features for the model.
- further analysis and recommendations at /graphs&presentation/ecomerce_business_case.pptx (eng/spa)
## Author
Ivonne Heredia | [Linkedin Profile](https://www.linkedin.com/in/ivonne-heredia/) |



