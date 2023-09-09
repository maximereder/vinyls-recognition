The Vinymatic Image Recognition App is an application designed to provide image recognition services. The app uses the SIFT algorithm to extract features from images, and Faiss for indexing and searching through the feature vectors.

![vinyl_recognition_visual](pictures/vinyl_recognition_visual.gif)

The app is composed of three services:

- **api**: the main service that provides the vinyl recognition API
- **ingestor**: a service that continuously monitors an SQS queue for new images to be indexed
- **caddy**: a reverse proxy to manage incoming requests and route them to the appropriate service

## Workflow

When a new image needs to be added to the index, it is first uploaded to an S3 bucket. The ingestor service is watching an SQS queue for messages containing the path of the new image in the S3 bucket. Once it receives a message, it reads the image from the S3 bucket, extracts its features using SIFT, and adds the features to the index. The ingestor service also periodically saves a backup of the index to an S3 bucket.

When a query image is submitted to the API service, it extracts its features using SIFT and queries the index to find the most similar images. The API service returns the paths of the most similar images to the client.

The overall architecture is designed to be scalable and fault-tolerant. The Faiss index is stored on disk and backed up periodically to Amazon S3 to ensure that it can be quickly restored in case of failure. The SQS queue allows for decoupling of the ingestion process from the API service and enables the system to handle large volumes of image uploads in a distributed and fault-tolerant manner.

## API Service

### Request

The API service is responsible for providing the image recognition API. It is built using FastAPI, and expose endpoint:

- `/search`: search for similar images in the database.

### Response

The response is a dictionary containing the IDs of the most similar images to the input image and their corresponding similarity scores. If no objects are found in the image, an error message is returned.

```json
{
    "ID1": similarity_score1,
    "ID2": similarity_score2,
    ...
}

```

### Configuration

The API service is built using FastAPI and is configured using environment variables:

- `SIFT_DIMENSIONS`: Number of dimensions for the SIFT feature vector.
- `SIMILARITY_THRESHOLD`: Threshold for the cosine similarity score used for determining the similarity between two images.
- `NUM_FEATURES`: Maximum number of SIFT features to extract from an image.
- `BACKUP_FOLDER`: Folder where the Faiss index and SIFT model are stored.
- `NOR_X`: Normalized width of the images.
- `NOR_Y`: Normalized height of the images.
- `TOP_N`: Number of similar images to return in the response.

## Ingestor Service

### Architecture

The `ingestor` service runs in a Docker container and is responsible for monitoring an SQS queue for new messages containing the S3 object keys of images to be ingested. When a message is received, the `ingestor`downloads the image from S3, verifies its integrity, computes its SIFT features, and adds it to a batch for later indexing. When the batch size reaches a specified number of images, the `ingestor` indexes the batch using Faiss and stores the resulting index on disk. The `ingestor` service is also responsible for periodically backing up the index to S3 for disaster recovery purposes.

### Configuration

The ingestor service is responsible for continuously monitoring an SQS queue for new images to be indexed. It is built using Python and the boto3 library. It is configured using environment variables:

- `S3_ACCESS_KEY_ID`: The AWS access key ID to use for S3 operations.
- `S3_SECRET_ACCESS_KEY`: The AWS secret access key to use for S3 operations.
- `S3_REGION`: The AWS region in which the S3 bucket resides.
- `S3_BUCKET`: The name of the S3 bucket containing the images to be ingested.
- `SQS_QUEUE_URL`: The URL of the SQS queue from which to read messages.
- `SIFT_DIMENSIONS`: The number of dimensions in the SIFT feature vector.
- `NUM_FEATURES`: The number of SIFT features to extract from each image.
- `BACKUP_FOLDER`: The path to the directory in which to store the index backups.
- `RETRAINING_INTERVAL`: The number of images to ingest before retraining the index.
- `DEBUG`: Whether to enable debug logging.
- `NOR_X`: The normalized X dimension to resize the images to.
- `NOR_Y`: The normalized Y dimension to resize the images to.
- `PYTHONUNBUFFERED`: Whether to disable output buffering.

## Caddy Service

Caddy is a lightweight, HTTP/2-enabled web server that can serve static and dynamic content, proxy incoming requests to other services, and handle SSL/TLS encryption.

In the current architecture, the Caddy service acts as a reverse proxy, forwarding incoming HTTP requests to the API service running on port 5002. It is also responsible for handling SSL/TLS encryption using Let's Encrypt SSL certificates.