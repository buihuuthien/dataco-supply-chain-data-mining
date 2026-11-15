# dataco-supply-chain-data-mining

Market Basket Analysis (Apriori, FP-Growth) and RFM Customer Segmentation (K-Means, DBSCAN) on the DataCo Smart Supply Chain dataset.

## 1. Project Overview
This project applies data mining techniques to retail and supply chain transaction records to uncover product association patterns, delivery delay factors, and customer behavioral segments.

- **Problem 1 — Association Rule Mining:** Identify frequently co-purchased products and operational conditions leading to late deliveries using **Apriori** and **FP-Growth**.
- **Problem 2 — Customer Segmentation (RFM):** Group customers based on Recency, Frequency, and Monetary metrics using **K-Means** and **DBSCAN**.

## 2. Dataset
- **Name:** DataCo SMART SUPPLY CHAIN FOR BIG DATA ANALYSIS
- **Source:** [Mendeley Data (DOI: 10.17632/8gx2fvg2k6.5)](https://doi.org/10.17632/8gx2fvg2k6.5)
- **Scale:** 180,519 records, 53 features, 65,752 orders, and 20,652 customers (2015–2018).
- **Note:** Raw CSV files are excluded via `.gitignore` due to GitHub's 100 MB file size limit. Download `DataCoSupplyChainDataset.csv` from Mendeley Data and place it in `data/raw/`.

## 3. Repository Structure

    dataco-supply-chain-data-mining/
    ├── data/
    │   ├── raw/                  # Raw dataset from Mendeley Data
    │   └── processed/            # Cleaned data and extracted RFM features
    ├── docs/                     # Data dictionary, methodology notes, and reports
    ├── notebooks/                # Jupyter Notebooks for EDA, preprocessing, and modeling
    ├── result/                   # Generated rules, cluster labels, and visualizations
    ├── .gitignore
    ├── README.md
    └── requirements.txt

## 4. Project Roadmap
- [x] Initialize repository and `README.md` structure
- [ ] Set up folder hierarchy and `requirements.txt`
- [ ] Exploratory Data Analysis (EDA) & Data Preprocessing
- [ ] Problem 1: Association Rule Mining (Apriori & FP-Growth)
- [ ] Problem 2: RFM Feature Engineering & Clustering (K-Means & DBSCAN)
- [ ] Results Evaluation, Visualization & Final Report

## 5. Setup & Usage
*(To be updated once `requirements.txt` and notebooks are added.)*
