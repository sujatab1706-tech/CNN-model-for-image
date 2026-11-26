# Amazon Listing-Image Compliance (TLGW / NWB / Brand Infringement / Prurience)

Multi-label CNN that flags product images breaking marketplace image policy.

| Label | Meaning |
|---|---|
| TLGW | Text, Logo, Graphics, or Watermark on the image |
| NWB | Non-White Background (main images need pure white) |
| BRAND_INFRINGEMENT | Counterfeit / unauthorized brand marks |
| PRURIENCE | Sexually suggestive or adult content |

An image can carry several flags at once, so the model uses one sigmoid output per label
(not softmax) and BCE loss.

## End-to-end flow

    Seller uploads -> S3 raw/ -> Jenkins -> prepare_dataset -> S3 processed/
        -> SageMaker training (GPU) -> metrics to CloudWatch -> quality gate
        -> manual approval -> SageMaker endpoint -> smoke test
        -> CloudWatch Logs metric filters + alarms -> SNS email

## Setup order

1. Deploy `infra/template.yaml` (CloudFormation) -> bucket, SageMaker role, SNS topic.
2. Edit `config/pipeline.yaml` (bucket, role ARN, region, alarm email).
3. Upload `raw/images/*` and `raw/labels.csv` (`image,TLGW,NWB,BRAND_INFRINGEMENT,PRURIENCE`, values 0/1).
4. In Jenkins: install *Pipeline Utility Steps* and *AWS Credentials* plugins, add credential id
   `aws-sagemaker-ci`, create a Pipeline job pointing at this repo's `Jenkinsfile`.
5. Run with `RUN_TRAINING=true`; add `DEPLOY_TO_PROD=true` once you trust the gate.

## Where things live

| Path | Purpose |
|---|---|
| `config/pipeline.yaml` | all settings, thresholds, quality gate |
| `infra/template.yaml` | S3, IAM, SNS (CloudFormation) |
| `src/model/cnn.py` | the CNN |
| `src/train.py` | SageMaker training script |
| `src/inference.py` | endpoint handlers |
| `pipeline/*.py` | data prep, launch training, gate, deploy, smoke test |
| `monitoring/setup_cloudwatch.py` | metric filters + alarms |
| `Jenkinsfile` | CI/CD stages |

## CNN layers (224x224x3 input)

| # | Layer | Output shape | What it learns / does |
|---|---|---|---|
| 1 | Conv block 1 (2x Conv3x3 + BN + ReLU, MaxPool) | 32 x 112 x 112 | edges, colour boundaries, white vs non-white pixels |
| 2 | Conv block 2 | 64 x 56 x 56 | corners, strokes, simple textures |
| 3 | Conv block 3 | 128 x 28 x 28 | letter shapes, logo fragments, skin-like textures |
| 4 | Conv block 4 | 256 x 14 x 14 | words, logo shapes, body parts |
| 5 | Conv block 5 | 256 x 7 x 7 | whole objects / scene-level concepts |
| 6 | Global Average Pooling | 256 | one number per feature map |
| 7 | Dropout(0.4) -> Linear 256->128 + ReLU | 128 | combines features into compliance-relevant patterns |
| 8 | Dropout(0.4) -> Linear 128->4 | 4 logits | one score per label |
| 9 | Sigmoid (inference only) | 4 probabilities | thresholded per label |

About 2.4M trainable parameters.

## Honest limitations

- BRAND_INFRINGEMENT is the hardest label: telling a *genuine* logo from a *counterfeit/unauthorized* one needs
  brand registry data. A from-scratch CNN will only learn "a logo-like thing is present". For production, add logo
  detection + embedding matching against a brand database, and keep humans reviewing flagged items.
- NWB can also be checked with a rule (border pixels close to RGB 255,255,255). Consider using the rule as a
  cross-check or fallback.
- With limited data, fine-tune a pretrained backbone (ResNet/EfficientNet) instead of training from scratch.
- Tests and AWS calls were not run in the build sandbox (no GPU/PyTorch/AWS access); syntax and YAML were checked.
