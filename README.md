Index:

# Aim:

To create RAG based chat application based on a simple text column in a bigquery bale.

# Pre-requisites:

GCP Free trial account with $300 credits.

Make sure that your project is activated:

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/4c41c8aa-6a1c-40de-8f91-a25627cf7a80/image.png)

# **Steps:**

1. Create bg dataset
2. Call it myrag
3. Create table from the above avro file in github: [netflixtitles.avro](https://github.com/onkarvelhals/gcpbqvertexaiembeddings/blob/main/netflixtitiles.avro). Or download directly:

[netflixtitiles.avro](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/9186c9eb-5330-40bd-ace7-511a6eb345a3/netflixtitiles.avro)

1. A new table with all values should be created directly.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/60150e9d-82f2-4773-a1ac-b89aff028e1b/image.png)

Check the description column. This column will be used for creating embeddings:

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/9cccf642-84f7-4637-abf4-3641c4e9b2e7/image.png)

1. IMP: Do not forget: Enable all APIS in vertex ai and bigquery:
gcloud services enable [aiplatform.googleapis.com](http://aiplatform.googleapis.com/)
2. Go back to bq: And in bq command line type following to understand bq dataset location:
All datasets:
bq ls -d --format=json
Specific in which above table from #3 is present:
bq show --format=json <dataset_name>

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/c5a090de-14d6-4ef7-ba11-d7fcc231e2d5/image.png)

1. First make sure to create biguery connection to Vertex AI: From bigquery Explore section, click on Plus Add sign and find Create new connection:

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/60fbff8e-4b50-4857-ac81-5b20f58e8633/image.png)

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/5b875cb4-ab75-4cc4-881a-4716165fe05f/image.png)

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/b3063352-b0a2-4bf2-8456-6097e6492ecb/image.png)

1. Grab the service account id from above step, Goto IAM
2. In IAM, click grant access, and paste this service account id in the principle section. Grant Vertex AI user role. 
3. Check the [documentation for model creation and find the latest text generation model](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions#embeddings_stable_model_versions): 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/0be9c86b-6e29-4bce-9d43-5c122947e83f/image.png)

1. Create the model :
Syntax:

```sql
--- Syntax:
CREATE OR REPLACE MODEL <project_id>.<dataset>.<new_model_name>
REMOTE WITH CONNECTION <dataset_location_from_step6>.<connection_name> -- Must include region
OPTIONS (
ENDPOINT = '<model_name_from_step7>' -- Version specified here
);

--- Actual:
CREATE OR REPLACE MODEL grounded-atrium-448715-k8.myrag.text-embedding-005
REMOTE WITH CONNECTION us.bgtovertexai -- Must include region
OPTIONS (
ENDPOINT = 'text-embedding-005' -- Version specified here
);
```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/7c77a727-1cf7-4ca5-bd9c-7e2079fca925/image.png)

Ref:
https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-embedding#text-embedding

1. Create the embeddings in bigquery itself:

following sql works but doesn’t finish running even after 25 mins:

<aside>
🚫

following is bad query: return everything, doesn't restrict columns 

</aside>

```sql
CREATE OR REPLACE TABLE `grounded-atrium-448715-k8.myrag.netflixtitles_embeddings_usingstruct` AS
SELECT *
FROM
ML.GENERATE_EMBEDDING(
  MODEL `myrag.text-embedding-005`,
  (SELECT description as content, title, show_id
  FROM `grounded-atrium-448715-k8.myrag.netflixtitles`),
  STRUCT(TRUE AS flatten_json_output, 'RETRIEVAL_DOCUMENT' as task_type)
);
```

Need optimized query:

check the columns in the output of ML.GENERATE_EMBEDDING:

```sql
SELECT *
FROM ML.GENERATE_EMBEDDING(
  MODEL `myrag.text-embedding-005`,
  (SELECT "Sample text" AS content),
  STRUCT(TRUE AS flatten_json_output, 'RETRIEVAL_DOCUMENT' AS task_type)
)
LIMIT 1;

```

This returns following columns:

ml_generate_embedding_result : this column contains the actual vectors

content: which was passed in above query

then use appropriate columns:

## Create embeddings final:

```sql
CREATE OR REPLACE TABLE `grounded-atrium-448715-k8.myrag.netflixtitles_embeddings_usingstruct` AS
SELECT
  t.show_id,
  t.title,
  t.description AS content,
  -- Use the correct column name for the embedding
  generated_embedding.ml_generate_embedding_result AS embedding
FROM
  `grounded-atrium-448715-k8.myrag.netflixtitles100` AS t
INNER JOIN (
  SELECT
    show_id,
    -- Use the actual output column name (e.g., `embedding`)
    ml_generate_embedding_result,
    content
  FROM ML.GENERATE_EMBEDDING(
    MODEL `myrag.text-embedding-005`,
    (SELECT show_id, description AS content FROM `grounded-atrium-448715-k8.myrag.netflixtitles100`),
    STRUCT(TRUE AS flatten_json_output, 'RETRIEVAL_DOCUMENT' AS task_type)
  )
) AS generated_embedding
ON t.show_id = generated_embedding.show_id;

```

The above BQ ML statements still consume a lot of rs. Using the above 2 queries (which were aborted after 40 mins) we could see around 120rs charged to the bq account. Hence, trying to find a cheaper option. using python api?> even that doesnt work or takes a lot if time to check python dependencies. 

Lets lower the row count to 100 for base text column:

Above query for 100 rows works: Hence new table is created but the embeddings are not properly populated… IMP:  the embeddings should be created for each row.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/78d65ab8-a9df-471a-80cf-7d78f6579abe/image.png)

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/2d031e6c-f39e-4162-a27a-238c2d8cdf25/ebac0b82-07f1-4843-aa67-1ca807272ed1/image.png)

# Build RAG: Create Similarity search

```sql
WITH query_embedding AS (
  SELECT embedding from grounded-atrium-448715-k8.myrag.netflixtitles_embeddings_usingstruct -- Replace with your query embedding
)
SELECT 
  show_id,
  title,
  ML.DISTANCE_COSINE(q.embedding, p.embedding) AS similarity
FROM `grounded-atrium-448715-k8.myrag.netflixtitles_embeddings_usingstruct` p, query_embedding q
ORDER BY similarity DESC
---LIMIT 3;
```

# create streamlit app for chat:

```python
import streamlit as st
import pandas as pd
from google.cloud import bigquery
from google.cloud import aiplatform

# Authenticate with default credentials (free tier)
bq_client = bigquery.Client()
aiplatform.init(project="grounded-atrium-448715-k8", location="us-central1")

def main():
    st.title("Free-Tier Movie Chatbot")
    user_query = st.text_input("Ask about movies:")
    
    if user_query:
        # Generate query embedding
        embedding = generate_embedding(user_query)
        
        # Run similarity search: this one might work
        query = f"""
            SELECT show_id, title, description
            FROM `grounded-atrium-448715-k8.myrag.netflixtitles_embeddings`
            ORDER BY ML.DISTANCE_COSINE(embedding, {embedding}) DESC
            LIMIT 3
        """
        results = bq_client.query(query).to_dataframe()
        
        # Generate response with Vertex AI's text-bison (free tier)
        context = "\n".join([f"{row['title']}: {row['description']}" for row in results.itertuples()])
        prompt = f"Answer this: {user_query}\n\nContext:\n{context}"
        
        response = aiplatform.Endpoint(
            "projects/grounded-atrium-448715-k8/locations/us/connections/bgtovertexai"
        ).predict(instances=[{"content": prompt}])
        
        st.write(response.predictions[0]["content"])

if __name__ == "__main__":
    main()`
```

