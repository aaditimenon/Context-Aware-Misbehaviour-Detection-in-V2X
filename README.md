# Context-Aware Misbehavior Detection in V2X

## 📌 Overview

This project implements a **hybrid Intrusion Detection System (IDS)** for **5G-V2X (Vehicle-to-Everything)** communication networks.
It combines **Stage 1: Machine Learning-based Intrusion Detection** with **Stage 2: V2X Performance Simulation** to detect malicious activities and evaluate their impact on vehicular safety and communication efficiency.

The system aims to improve **security, reliability, and safety** of connected and autonomous vehicles (CAVs) by detecting attacks in real-time and analyzing their effect on decision-making, throughput, and collision risk.

---

## 🚦 Project Stages

### **Stage 1: Intrusion Detection**

* Dataset: Preprocessed network traffic dataset.
* Model: Random Forest classifier.
* Features: Forward/Backward packet statistics, protocol, throughput ratios, etc.
* Output: Binary classification (**Attack** vs. **Benign**).
* Evaluation: Confusion matrix, classification report, feature importance plots.

### **Stage 2: V2X Simulation**

* Dataset: Preprocessed V2X communication dataset (with GPS, LiDAR, radar, packet metrics, latency, throughput, etc.).
* Stage 1’s IDS output is integrated with vehicle communication.
* Evaluates how blocking malicious packets affects:

  * **Decision Accuracy**
  * **Obstacle Detection Accuracy**
  * **Collision Rates**
  * **Throughput & Latency**

---

## ⚙️ Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/your-username/your-repo.git
   cd your-repo
   ```

2. Install dependencies:

   ```bash
   pip install -r requirements.txt
   ```

3. Launch Jupyter Notebook (if using notebooks):

   ```bash
   jupyter notebook
   ```

---

## ▶️ Usage

* **Stage 1**: Train the Random Forest classifier on the intrusion dataset.
* **Stage 2**: Run the V2X simulation dataset through the trained IDS model and evaluate its impact.

Outputs include:

* Confusion Matrix
* Classification Reports
* Feature Importance Graphs
* V2X Safety & Communication Metrics

---

## 📊 Results

* Stage 1 achieved **\~97% accuracy** with balanced precision/recall across classes.
* Stage 2 will extend evaluation by analyzing how IDS decisions influence real-world V2X safety outcomes.

---

## 🛠️ Technologies Used

* **Python 3.x**
* **Scikit-learn** (Random Forest, preprocessing, evaluation)
* **Pandas, NumPy** (data processing)
* **Matplotlib, Seaborn** (visualization)
* **Jupyter Notebook** (experiments and results)

---

## 🚀 Next Steps

* Extend Stage 2 by integrating IDS outputs into V2X decision-making.
* Simulate different attack scenarios and evaluate their effects on communication and safety.
* Explore deep learning methods for Stage 1 detection.

---
