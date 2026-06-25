### Early-Prediction-Course-Quality
- Designed a course quality prediction framework from the MoocubeX dataset by processing 13 relational learning analytics
tables with up to 11M+ records. The original dataset available at [https://github.com/THU-KEG/MOOCCubeX](here)
- Built a feature engineering pipeline to compute COELO, AFELO, and ACELO scores at the chapter level, combining multiple
sub-indicators from learner behavior, course engagement, affective feedback based on chapter-level activity. Integrated BERT-
based sentiment analysis on learner comments to generate affective learning signals for the AFELO component.
- Generated final Course Quality Score (CQS) labels by aggregating COELO, AFELO, and ACELO scores and applying
quantile-based discretization into Needs Improvement, Acceptable, and Excellent categories.
- Identified and analyzed predictive features. Compared imbalance learning strategies, including Tomek Links under-sampling,
SMOTE oversampling, CTGAN-based synthetic sampling, and hybrid resampling methods.
- Trained and evaluated multiple machine learning models, with Random Forest + SMOTE-Tomek achieving the best overall
result of 0.818 Macro-F1 and 0.90 recall for the Needs Improvement class.
- Details available at [this google drive link](https://drive.google.com/drive/u/0/folders/1r_ZOKB9jn8U8KBTChkmsPHwwmYisIw7V)
