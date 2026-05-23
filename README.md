# Market Basket Analysis with FP-Growth

**Author:** Sanket Madaan  

This project implements a complete market basket analysis and rule-based recommender system on the Instacart Market Basket dataset using a from-scratch implementation of the FP-Growth algorithm. The focus is on efficiently mining high-quality association rules and evaluating their usefulness for product recommendation.

---

## 1. Overview

The goal of this project is to discover meaningful associations between products frequently bought together and to use these associations to recommend new products to users based on the contents of their shopping cart.

Key components:

- Construction of a transactional dataset from raw Instacart order and product files  
- From-scratch implementation of FP-Growth for frequent itemset mining  
- Association rule generation with support, confidence, and lift  
- Rule-based recommendation engine and evaluation using precision, recall, and F1-score  
- Exploration of threshold trade-offs (min support, min confidence, lift) and their impact on rule quality and recommender performance  

---

## 2. Dataset and Preprocessing

The project uses the Instacart Market Basket dataset (publicly available on Kaggle). The following files are used:

- `orders.csv`: Contains order-level metadata  
- `order_products__train.csv`: Contains product IDs for orders in the training evaluation set  
- `products.csv`: Maps product IDs to human-readable product names  

### 2.1 Transaction Construction

1. Filter `orders.csv` to keep only rows where `eval_set == 'train'`.  
2. Join these orders with `order_products__train.csv` on `order_id` to get the products in each order.  
3. Group by `order_id` and aggregate product IDs into a set for each order, producing a list of transactions:
   - Each transaction = one order  
   - Each order = set of product IDs in that basket  

### 2.2 Train–Test Split

- Randomly split the transactional dataset into:
  - **80% training set** (≈ 105K transactions)  
  - **20% test set** (≈ 26K transactions)  
- Use a fixed random seed to ensure reproducibility.

The training set is used for mining frequent itemsets and rules; the test set is used purely for evaluating the recommendation component.

---

## 3. FP-Growth Implementation

The core of the project is a from-scratch implementation of the FP-Growth algorithm.

### 3.1 FP-Tree Construction

- First pass: Count item frequencies in the training transactions and filter out items below a minimum support count.  
- Build a header table mapping each frequent item to:
  - its global support count  
  - the head of a linked list of FP-tree nodes for that item  
- Second pass: For each transaction:
  - Filter to frequent items  
  - Sort items in descending order of global frequency  
  - Insert them into the FP-tree, incrementing counts along common prefixes  
  - Maintain node-links between nodes of the same item to efficiently traverse conditional pattern bases  

### 3.2 Mining Frequent Itemsets

- For each item in the header table (processed in ascending support order):
  - Collect all prefix paths leading to that item to form a **conditional pattern base**  
  - Build a **conditional FP-tree** from these paths  
  - Recursively mine the conditional FP-tree to generate larger frequent itemsets  
- The implementation records:
  - Each frequent itemset as a `frozenset` of item IDs  
  - Its support count (absolute number of transactions containing that itemset)  

This yields a compact set of frequent itemsets without explicit candidate generation, which is particularly valuable for large datasets.

---

## 4. Association Rules and Lift

### 4.1 Rule Generation

From the mined frequent itemsets, the project generates association rules of the form:

$$\{A\} \rightarrow Y$$

where:

- The antecedent $A$ is always a **single product**
- The consequent $Y$ can contain one or more products

For each frequent itemset with size ≥ 2:

- For every item $A$ in that itemset:
  - Define the antecedent as $\{A\}$
  - Define the consequent as the remaining items in the itemset

### 4.2 Metrics: Support, Confidence, Lift

For each rule, the following metrics are computed:

- **Support:**
  Fraction of transactions that contain both antecedent and consequent.
- **Confidence:**
  $$\text{conf} = \frac{\text{support}(\text{antecedent} \cup \text{consequent})}{\text{support}(\text{antecedent})}$$
- **Lift:**
  $$\text{lift} = \frac{\text{support}(\text{antecedent} \cup \text{consequent})}{\text{support}(\text{antecedent}) \cdot \text{support}(\text{consequent})}$$
  
  Lift measures how much more often the items appear together than would be expected if they were independent.

Rules are filtered based on minimum thresholds for support, confidence, and lift.

---

## 5. Threshold Exploration

To understand the trade-offs between coverage and quality, the project performs a parameter sweep over:

- **Minimum support ratio (`minsup`)** in approximately `[0.0005, 0.01]`  
- **Minimum confidence (`minconf`)** in `[0.01, 0.1]`  
- **Minimum lift (`min_lift`)** in `{1.0, 1.5, 2.0}`  

For each combination:

- Run FP-Growth with the corresponding minimum support count  
- Generate rules satisfying minimum confidence and minimum lift  
- Record the number of rules produced  

This analysis shows:

- Very low support thresholds produce tens of thousands of rules (high coverage but poor interpretability).  
- Very high support thresholds yield only a handful of trivial rules (e.g., popular staples).  
- An intermediate support threshold provides a balance: enough rules for useful recommendations without overwhelming noise.

A final configuration is chosen based on this trade-off as well as downstream recommendation performance:

- `minsup = 0.0015`  
- `minconf = 0.075`  
- `min_lift = 1.5`

---

## 6. Recommendation Engine and Evaluation

### 6.1 Rule-Based Recommender

The recommendation engine uses the mined rules as follows:

1. Build a dictionary keyed by antecedent item (single product).  
2. For each antecedent, store a list of consequents along with confidence and lift, sorted by confidence.  
3. Given a user basket \( I \):
   - For each item in \( I \), retrieve rules whose antecedent matches that item.  
   - Collect all candidate consequents not already in \( I \).  
   - Score candidates (e.g., by maximum confidence across contributing rules).  
   - Recommend the top-\( k \) items by score.

### 6.2 Test Protocol

The test set (20% hold-out) is used to evaluate recommendation quality:

- For each transaction with at least two items:
  - Split its items into:
    - Input set: first half of items  
    - Ground truth set: second half of items  
- For each value of \( k \) from 1 to 10:
  - Generate top-\( k \) recommendations from the input set  
  - Compare against the ground truth set  

### 6.3 Metrics: Precision, Recall, F1

For each transaction and each \( k \):

- **Precision:** fraction of recommended items that are actually in the ground truth  
- **Recall:** fraction of ground truth items that were successfully recommended  
- **F1-score:** harmonic mean of precision and recall  

The project computes:

- Average precision and recall across all test baskets for each \( k \)  
- F1-score for each `(minsup, minconf, min_lift)` configuration  

This produces:

- Curves of F1-score vs. number of recommendations \( k \)  
- Precision–recall curves highlighting the trade-off between accuracy and coverage  
- Scatter plots of rule-set size vs. F1-score, colored by average lift, to visualize the quality–quantity trade-off across parameter settings.

---

## 7. Analysis of Top Rules

For the final chosen thresholds, the project creates:

- A list of the **top 100 rules by support**  
- A list of the **top 100 rules by confidence**  

By intersecting these lists, it isolates rules that are both:

- Frequently observed in the data, and  
- Highly reliable when the antecedent appears  

Typical patterns observed include:

- Strong associations between various organic fruits and “Bag of Organic Bananas”  
- High-lift pairings such as herbs with complementary ingredients (e.g., cilantro with limes)  
- Common staple relationships, such as bananas being a frequent consequent for several other fruits  

These rules are exported as text files with product names for easy inspection.

---

## 8. Sample User Analysis

To provide a qualitative view, the project performs a detailed analysis for a small set of randomly chosen test baskets:

- For each selected user:
  - Display input items and ground truth items (using product names)  
  - Show how precision and recall evolve as \( k \) increases from 1 to 10  
  - Indicate which input items triggered rules and where successful hits occurred in the recommendation list  

These case studies illustrate:

- Late hits (correct items appearing only at larger \( k \))  
- Strong top-1 predictions but limited recall  
- Cold-start or sparsity cases where no recommendation is possible because the input items do not appear as antecedents in any rule.

---

## 9. Implementation Details

The project is implemented in Python using:

- `pandas` and `NumPy` for data handling and numerical operations  
- Custom classes and functions for FP-tree nodes, FP-tree construction, and recursive mining  
- Custom rule generation with explicit support, confidence, and lift computation  
- `Matplotlib` and `Seaborn` for plotting:
  - Number of rules vs. thresholds  
  - F1 vs. number of recommendations  
  - Precision–recall curves  
  - Rule-set size vs. F1, colored by average lift  

The code is structured into clearly separated stages:

1. Data loading and transactional dataset creation  
2. FP-Growth mining and rule generation  
3. Threshold experimentation and rule-set analysis  
4. Recommendation evaluation on the hold-out test set  
5. Visualization and user-level analysis  


---
