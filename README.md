**# Creating RAG Based chat application using simple text in a bigquery table.
**

**Pre-requisites:
**GCP Free trial account with $300 credits. 

**Steps:**

1. Create bg dataset
2. Call it myrag
3. Create table from the above avro file. netflixtitles.avro.
4. A new table with all values should be created directly.
5. IMP: Do not forget: Enable all APIS in vertex ai.
   gcloud services enable aiplatform.googleapis.com

6. Go back to bq: And in bq command line type following to understand bq dataset location:
  All datasets:
   bq ls -d --format=json
   Specific in which above table from #3 is present:
   bq show --format=json <dataset_name>

7. Check the documentation for model creation and find the latest text generation model: https://cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions#embeddings_stable_model_versions
8. Create the model first:
   Syntax:
   CREATE OR REPLACE MODEL `grounded-atrium-448715-k8.myrag.text-embedding-005`
REMOTE WITH CONNECTION `us.bgtovertexai`  -- Must include region
OPTIONS (
    ENDPOINT = 'text-embedding-005'  -- Version specified here
);

CREATE OR REPLACE MODEL `grounded-atrium-448715-k8.myrag.text-embedding-005`
REMOTE WITH CONNECTION `us.bgtovertexai`  -- Must include region
OPTIONS (
    ENDPOINT = 'text-embedding-005'  -- Version specified here
);



Ref:
   https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-embedding#text-embedding
   
