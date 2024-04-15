# OpenAI's Whisper in Snowpark Container Services

This guide explains how to run OpenAI's [Whisper](https://github.com/openai/whisper) in Snowpark Container Services.  
Afterwards you can easily transcribe any audio file into text and detect its language.

## Requirements
* Account with Snowpark Container Services enabled
* Docker installed

## Setup Instructions
### 1. Create image repository, stages and compute pool 
```sql
-- grant create compute pools rights to SYSADMIN
USE ACCOUNTADMIN;
GRANT CREATE COMPUTE POOL ON ACCOUNT TO ROLE SYSADMIN;

-- clean up
USE ROLE SYSADMIN;

DROP DATABASE AUDIO_DEMO;
DROP COMPUTE POOL MY_GPU_POOL;

-- create DB
CREATE OR REPLACE DATABASE WHISPER_DEMO;

-- create image repository
CREATE OR REPLACE IMAGE REPOSITORY IMAGE_REPOSITORY;

-- Create Stages
CREATE STAGE IF NOT EXISTS AUDIO_FILES ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = (ENABLE = TRUE);
CREATE STAGE IF NOT EXISTS DOCKER_SPECS ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = (ENABLE = TRUE);
CREATE STAGE IF NOT EXISTS WHISPER_MODELS ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = (ENABLE = TRUE);

-- Create Compute Pool
CREATE COMPUTE POOL MY_GPU_POOL
  MIN_NODES = 1
  MAX_NODES = 1
  INSTANCE_FAMILY = GPU_3;
```

### 2. Create 
```bash
git clone https://github.com/thywoof/scs_whisper.git
```

### 3. Build & Upload the container
use lower case on orgname and acctname. replace any `_` with a `-`.

```cmd
cd scs_whisper
docker build -t <ORGNAME>-<ACCTNAME>.registry.snowflakecomputing.com/whisper_demo/image_repository/whisper_app:latest .
docker push <ORGNAME>-<ACCTNAME>.registry.snowflakecomputing.com/whisper_demo/public/image_repository/whisper_app:latest
```

### 4. Upload files to stages  
Use your favourite way of uploading files and upload 
* the `spec.yml` to stage `DOCKER_SPECS`
* the audio files to stage `AUDIO_FILES`
- the whisper models to stage `WHISPER_MODELS`

### 5. Create the Whisper Service
```sql
-- Create Service
CREATE SERVICE WHISPER_APP
  IN COMPUTE POOL MY_GPU_POOL
  FROM @WHISPER_APP
  SPEC='spec.yml'
  MIN_INSTANCES=1
  MAX_INSTANCES=1;

-- Verify Service is running
SELECT SYSTEM$GET_SERVICE_STATUS('WHISPER_APP');
```

### 6. Create the service functions for language detection and transcription
```sql
-- Function to detect language from audio file
CREATE OR REPLACE FUNCTION DETECT_LANGUAGE(AUDIO_FILE TEXT, ENCODE BOOLEAN)
RETURNS VARIANT
SERVICE=WHISPER_APP
ENDPOINT=API
AS '/detect-language';

-- Function to transcribe audio files
CREATE OR REPLACE FUNCTION TRANSCRIBE(TASK TEXT, LANGUAGE TEXT, AUDIO_FILE TEXT, ENCODE BOOLEAN)
RETURNS VARIANT
SERVICE=WHISPER_APP
ENDPOINT=API
AS '/asr';
```

### 7. Call the service functions using files from a Directory Table
```sql
-- Run Whisper on a files in a stage
SELECT RELATIVE_PATH, 
       GET_PRESIGNED_URL('@whisper_demo.public.AUDIO_FILES', RELATIVE_PATH) AS PRESIGNED_URL,
       DETECT_LANGUAGE(PRESIGNED_URL,True)  AS WHISPER_RESULTS,
       WHISPER_RESULTS['detected_language']::text as DETECTED_LANGUAGE
FROM DIRECTORY('@whisper_demo.public.AUDIO_FILES');

SELECT RELATIVE_PATH, 
       GET_PRESIGNED_URL('@whisper_demo.public.AUDIO_FILES', RELATIVE_PATH) AS PRESIGNED_URL,
       TRANSCRIBE('transcribe','',PRESIGNED_URL,True) AS WHISPER_RESULTS,
       WHISPER_RESULTS['text']::TEXT as EXTRACTED_TEXT
FROM DIRECTORY('@whisper_demo.public.AUDIO_FILES');
```

### 8. Clean your environment
```sql
-- Clean Up
DROP SERVICE WHISPER_APP;
DROP COMPUTE POOL MY_GPU_POOL;
```

### Debugging: View Logs
If you want to know what's happening inside the container, you can retrieve the logs at any time.
```sql
-- See logs of container
SELECT value AS log_line
FROM TABLE(
 SPLIT_TO_TABLE(SYSTEM$GET_SERVICE_LOGS('whisper_app', 0, 'proxy'), '\n')
  );
```