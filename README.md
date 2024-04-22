### LLM in your Transactional Database: 
## Build a Patent Search App with Spanner, Vector Search & Gemini 1.0 Pro!

In this article, you will build a patent search application that lets users search an idea or topic in a patent database (made publicly available in BigQuery) for published patents that closely match their topic or context of search using Similarity Vector Search in Spanner.

We've built a system that tackles these challenges head-on:
Spanner as the Foundation: We use Google Cloud Spanner, a globally distributed, strongly consistent database, to store our patent data. This ensures data integrity and high availability, crucial for a large-scale application.
Gemini 1.0 Pro for Summarization and Keyword Extraction: We tap into the power of Gemini 1.0 Pro, Google's advanced large language model, directly within Spanner SQL queries. This allows us to transform complex patent abstracts into concise summaries and extract relevant keywords on the fly.
Embeddings for Semantic Search: We convert both the LLM-generated summaries and user search queries into numerical representations called embeddings. This enables us to perform semantic search - finding patents based on meaning and context rather than just keyword overlap.
Real-time Matching with Cosine Similarity: When a user enters a search query, we convert it to embeddings and then calculate the cosine similarity between these embeddings and the stored patent embeddings. This quickly identifies the most relevant patents, even if the user's phrasing doesn't perfectly match the patent text.

Let's dive in!

## Hands-on Time
This implementation is done with Spanner SQL DDLs, DMLs and simple select queries. You can use these in your application to build beautiful user experiences for the use case.
In 5 easy steps, you can build a patent search application using Spanner and Vertex AI integration!

## Setup
In the Google Cloud Console, on the project selector page, select or create a Google Cloud project.
Make sure that billing is enabled for your Cloud project. Learn how to check if billing is enabled on a project.
You will use Cloud Shell, a command-line environment running in Google Cloud that comes preloaded with bq. From the Cloud Console, click Activate Cloud Shell on the top right corner.
Enable necessary APIs for this implementation if you haven't already: Cloud Spanner API. To do this, navigate to the Cloud Shell Terminal and enter the following command:
gcloud services enable spanner.googleapis.com

## Create Spanner Instance, Database and Table
Create an instance named "spanner-vertex" and a database named "spanner-vertex-embeddings". Create a table using the DDL:
CREATE TABLE patents_data (
   id string(25), type string(25), number string(20), country string(2), date string(20), abstract string(300000), title string(100000),kind string(5), num_claims numeric, filename string(100), withdrawn numeric, 
) PRIMARY KEY (id);

## Prepare & Load Patent Data
For building the Patent Search App, we will use the Patent Published dataset in BigQuery. For ease of implementation, I have already prepared the data and made it available here.
Run the INSERT scripts in Spanner Studio Editor. This should populate the patents_data table we created in the previous step. This is the patent data we will use for matching with the user search text.

## Create Remote Model for Gemini 1.0 Pro
We will convert the patent abstracts into a consolidated summary consisting of a title and keywords. For this we will use the Gemini 1.0 Pro model from Vertex AI remotely from Spanner. Run the following DDL from Spanned Studio Editor:
CREATE MODEL gemini_pro_model INPUT(
prompt STRING(MAX),
) OUTPUT(
content STRING(MAX),
) REMOTE OPTIONS (
endpoint = '//aiplatform.googleapis.com/projects/<<YOUR_PROJECT_ID>>/locations/us-central1/publishers/google/models/gemini-pro',
default_batch_size = 1
);

## Create Remote Model for Text Embeddings
We will convert the Gemini 1.0 Pro model's response (that is the consolidated summary consisting of a title and keywords) into embeddings for performing the match. For this we will use the Text Embedding Gecko 003 model from Vertex AI remotely from Spanner. Run the following DDL from Spanned Studio Editor:
CREATE MODEL text_embeddings INPUT(content STRING(MAX))
OUTPUT(
  embeddings
    STRUCT<
      statistics STRUCT<truncated BOOL, token_count FLOAT64>,
      values ARRAY<FLOAT64>>
)
REMOTE OPTIONS (
  endpoint = '//aiplatform.googleapis.com/projects/<<YOUR_PROJECT_ID>>/locations/us-central1/publishers/google/models/textembedding-gecko@003');

## Create Generative Insights from Patent Abstracts
We will use the remote Gemini 1.0 Pro model in Spanner to create insights and keywords from the abstract data and store the results in a separate table for generative insights. To create the table to store the generative AI response, run the following DDL in Spanner Studio Editor:

CREATE TABLE patents_data_gemini (id string(100), gemini_response STRING(MAX)) PRIMARY KEY (id);

# Design Tip: Remember you can use an application to populate the generative insights into this table in batch write or mutations method.

But for this sample dataset, you can run the below query in 3 to 4 repetitions to manually batch-insert the generative insights response to the table from Spanner Studio Editor:
INSERT INTO patents_data_gemini (id, gemini_response) 
SELECT id, content as gemini_response 
FROM ML.PREDICT(MODEL gemini_pro_model,
(select id, concat ('Identify the areas of work or keywords in this abstract', abstract) as prompt from patents_data b where id not in (select id from patents_data_gemini) limit 50
));

Notice the prompt used in the above query to generate the insights: "Identify the areas of work or keywords in this abstract". 
Check the results of the generated insights using this query:
select title, abstract, gemini_response from patents_data a inner join patents_data_gemini b
on a.id = b.id;

# Note: In the above step you can choose to update the insights results in the same table as the source data, but for keeping the transaction data normalized, I wanted to create a separate table for the Gemini 1.0 Pro model results.

## Generate Embeddings for the Generated Insights
In this step, we will generate embeddings for the generated insights. Let's run the following DDL and DML in the Spanner Studio Editor:
--DDL
CREATE TABLE patents_data_embeddings (id string(100), patents_embeddings ARRAY<FLOAT64>) PRIMARY KEY (id);

--DML
INSERT INTO patents_data_embeddings (id, patents_embeddings) 
SELECT id, embeddings.values as patents_embeddings 
FROM ML.PREDICT(MODEL text_embeddings,
(SELECT id, gemini_response as content FROM patents_data_gemini));
Check the results using the following query:
select title, abstract, b.patents_embeddings from patents_data a inner join patents_data_embeddings b 
on a.id = b.id;

## Similarity Vector Search
Now that the embeddings are created for the generative insights, let's create embeddings for the search text and prepare it for Vector Search. This is the search text that will be entered into the search application (by the user). 
Please note we haven't built the web app here, you can build a simple search app to visualize this use case with the data layer we are setting up here by following the web app example in this repo. 
For this implementation, we will be using the K-Nearest Neighbors Similarity Search capability. Remember you can also directly integrate with Vertex AI Vector Search from Spanner. Read about it here.
To create the embeddings for the search text / topic and run the Vector Search in Spanner, run the following query in the Spanner Studio Editor:
SELECT a.id, a.title, a.abstract, 'A new Natural Language Processing related Machine Learning Model' search_text, COSINE_DISTANCE(c.patents_embeddings,
(SELECT embeddings.values
FROM ML.PREDICT(
MODEL text_embeddings,
(SELECT 'A new Natural Language Processing related Machine Learning Model' as content)))) as distance
FROM patents_data a inner join patents_data_gemini b on a.id = b.id
inner join patents_data_embeddings c on a.id = c.id
ORDER BY distance
LIMIT 10;

If you notice, in the above query, I have searched for the text 'A new Natural Language Processing related Machine Learning Model' in the patents data to find the 10 closest matches using the COSINE_DISTANCE method.

As you can observe in your results, the matches are pretty close to the search text. That's it! It is that simple to perform Similarity Vector Search using Embeddings model on Spanner data.
