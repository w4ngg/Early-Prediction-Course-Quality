# Early-Prediction-Course-Quality
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

# CourseQuality Data Pipeline Architecture

This diagram describes the current local-first, S3-ready CourseQuality pipeline.
It follows a medallion-style data lake layout: Raw/Bronze, Silver, Gold, then
ML-ready datasets and model artifacts.

## End-to-End Pipeline

```mermaid
flowchart LR
    subgraph Runtime["Runtime and Config"]
        CFG["configs/pipeline.local.yaml<br/>configs/pipeline.s3.yaml"]
        DOCKER["Docker image<br/>course-quality"]
        ORCH["Local runner<br/>scripts/run_pipeline.py"]
        AIRFLOW["Future orchestration<br/>Airflow / MWAA"]
    end

    subgraph Storage["Storage Layer"]
        LOCAL["Local FS<br/>data_raw / data_lake / artifacts"]
        S3["Amazon S3<br/>raw / data_lake / artifacts"]
    end

    subgraph Raw["Raw Source Data"]
        RAW_COURSE["course.csv"]
        RAW_USER["user.csv"]
        RAW_VIDEO["user_video.csv"]
        RAW_PROBLEM["user_problem.csv"]
        RAW_COMMENT["user_comment.csv<br/>course_comment.csv"]
        RAW_RESOURCE["video.csv<br/>problem.csv<br/>exercise_problem.csv"]
    end

    subgraph Bronze["Bronze Snapshot"]
        BRONZE["bronze/snapshot_date=YYYY-MM-DD<br/>raw file copy + manifest.json"]
    end

    subgraph Silver["Silver Fact-Oriented Layer"]
        RESOURCE["course_resource_map"]
        ENROLL["user_course_enrollment_fact"]
        VIDEO["course_video_fact"]
        PROBLEM["course_problem_fact"]
        COMMENT["course_comment_fact<br/>offline sentiment labels"]
        LIFE["course_lifecycle_fact"]
    end

    subgraph Gold["Gold Analytical Layer"]
        COELO["COELO<br/>Cognitive Engagement<br/>TS / RV / IF / AV"]
        ACELO["ACELO<br/>Academic Engagement<br/>scores / attempts / results"]
        AFELO["AFELO<br/>Affective Engagement<br/>FV / FA sentiment"]
        LABELS["course_labels<br/>course_label / label_details / label_chapters"]
        FEATURES["course_phase_feature_snapshot<br/>G1 / G2 / G3 / FULL"]
        TRAINING["training_dataset<br/>train_full / valid_full<br/>test_G1 / test_G2 / test_G3<br/>split_manifest.json"]
    end

    subgraph MLOps["MLOps Layer"]
        ARTIFACTS["artifacts/models/course_quality"]
        TRAIN["Model training<br/>course quality prediction"]
        EVAL["Evaluation<br/>valid + G1/G2/G3 test"]
        SERVE["Batch scoring / API serving"]
    end

    CFG --> ORCH
    CFG --> DOCKER
    DOCKER --> ORCH
    AIRFLOW -. cloud orchestration .-> DOCKER

    ORCH --> LOCAL
    ORCH -. S3-ready .-> S3

    RAW_COURSE --> BRONZE
    RAW_USER --> BRONZE
    RAW_VIDEO --> BRONZE
    RAW_PROBLEM --> BRONZE
    RAW_COMMENT --> BRONZE
    RAW_RESOURCE --> BRONZE

    BRONZE --> RESOURCE
    BRONZE --> ENROLL
    BRONZE --> VIDEO
    BRONZE --> PROBLEM
    BRONZE --> COMMENT
    VIDEO --> LIFE
    PROBLEM --> LIFE
    COMMENT --> LIFE
    ENROLL --> LIFE

    RESOURCE -. resource/chapter map .-> COELO
    VIDEO --> COELO
    PROBLEM --> COELO

    RESOURCE --> ACELO
    VIDEO --> ACELO
    PROBLEM --> ACELO
    COMMENT --> ACELO

    VIDEO --> AFELO
    COMMENT --> AFELO

    COELO --> LABELS
    ACELO --> LABELS
    AFELO --> LABELS

    RESOURCE --> FEATURES
    ENROLL --> FEATURES
    VIDEO --> FEATURES
    PROBLEM --> FEATURES
    COMMENT --> FEATURES
    LIFE --> FEATURES
    LABELS --> FEATURES

    FEATURES --> TRAINING
    LABELS --> TRAINING
    TRAINING --> TRAIN
    TRAIN --> ARTIFACTS
    ARTIFACTS --> EVAL
    EVAL --> SERVE
```

## Data Modeling and Lineage

```mermaid
flowchart TB
    subgraph RawLayer["Raw Layer"]
        R1["Raw CSV files<br/>source format"]
    end

    subgraph SilverLayer["Silver Layer: Fact-Oriented Modeling"]
        D1["course_resource_map<br/>resource-course-chapter bridge"]
        F1["course_video_fact<br/>video view events"]
        F2["course_problem_fact<br/>problem attempt events"]
        F3["course_comment_fact<br/>comment events + sentiment"]
        F4["user_course_enrollment_fact<br/>enrollment relationship"]
        F5["course_lifecycle_fact<br/>course activity window"]
    end

    subgraph GoldLayer["Gold Layer: Aggregated Indicators"]
        G1["COELO<br/>Cognitive Engagement<br/>TS / RV / IF / AV"]
        G2["ACELO<br/>Academic Engagement<br/>scores / attempts / results"]
        G3["AFELO<br/>Affective Engagement<br/>FV / FA sentiment"]
        G4["Course labels<br/>CQS / CQS_num"]
        G5["Feature snapshots<br/>G1 / G2 / G3 / FULL"]
    end

    subgraph MLLayer["ML Dataset Layer"]
        M1["train_full.csv"]
        M2["valid_full.csv"]
        M3["test_G1.csv"]
        M4["test_G2.csv"]
        M5["test_G3.csv"]
        M6["split_manifest.json"]
    end

    R1 --> D1
    R1 --> F1
    R1 --> F2
    R1 --> F3
    R1 --> F4
    F1 --> F5
    F2 --> F5
    F3 --> F5
    F4 --> F5

    D1 -. resource/chapter map .-> G1
    F1 --> G1
    F2 --> G1

    D1 --> G2
    F1 --> G2
    F2 --> G2
    F3 --> G2

    F1 --> G3
    F3 --> G3

    G1 --> G4
    G2 --> G4
    G3 --> G4

    D1 --> G5
    F1 --> G5
    F2 --> G5
    F3 --> G5
    F4 --> G5
    F5 --> G5
    G4 --> G5

    G5 --> M1
    G5 --> M2
    G5 --> M3
    G5 --> M4
    G5 --> M5
    G4 --> M6
```

## Engagement Indicator Definitions

```text
COELO - Cognitive Engagement / Mức độ tham gia nhận thức:
Phản ánh nỗ lực tinh thần và mức độ xử lý thông tin sâu của người học.
Chỉ số này được tổng hợp từ 4 thành phần con: TS, RV, IF, và AV.
Các biến số chính được tính từ dữ liệu tương tác user-video và user-problem;
course_resource_map đóng vai trò hỗ trợ map resource/chapter.

ACELO - Academic Engagement / Mức độ tham gia học thuật:
Đo lường trách nhiệm và kết quả học tập thực tế. Chỉ số này được tính toán
dựa trên điểm số, số lần thử, và kết quả làm bài tập của người học được ghi
nhận trong hệ thống.

AFELO - Affective Engagement / Mức độ tham gia cảm xúc:
Đánh giá thái độ và sự hứng thú của người học. Chỉ số này được xác định qua
FV và FA. Trong đó, FA được xác định bằng sentiment analysis trên dữ liệu
comment và user-comment của học viên.
```

## Current Execution Modes

```mermaid
flowchart LR
    CODE["CourseQuality source code"]
    CONFIG["Config-driven paths<br/>raw_uri / data_lake_uri / artifacts_uri"]
    LOCAL["Local execution<br/>python scripts/run_pipeline.py"]
    DOCKER["Docker execution<br/>docker compose run pipeline"]
    CLOUD["Cloud-ready execution<br/>ECS / AWS Batch / MWAA"]
    LOCALFS["Local filesystem"]
    S3["Amazon S3"]

    CODE --> CONFIG
    CONFIG --> LOCAL
    CONFIG --> DOCKER
    CONFIG --> CLOUD
    LOCAL --> LOCALFS
    DOCKER --> LOCALFS
    DOCKER -. with S3 config .-> S3
    CLOUD --> S3
```
