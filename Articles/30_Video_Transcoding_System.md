# System Design: Video Transcoding System (Live to VOD)

When an online live class ends, students often expect the recording to be available almost immediately. However, processing a 2.5-hour video file is a heavy task. If the system waits until the class is over to upload a massive video file, it will be slow, prone to network failures, and delay the availability of the recording. 

This article outlines an **Event-Driven Video Transcoding Architecture** that solves this problem by uploading video in chunks and stitching them together asynchronously using AWS serverless technologies.

---

## 1. The Core Strategy: Chunking

Instead of recording one giant video file locally on the instructor's device, the client application records the live class in small segments (e.g., 1-minute chunks). 
* Over a 2.5-hour class, this results in **100 to 200 chunks**.
* These chunks are uploaded to an **AWS S3 Bucket** continuously in the background while the live class is still ongoing.

This approach guarantees that when the instructor clicks "End Class", 99% of the video data is already safely stored in the cloud.

---

## 2. Step-by-Step Architecture Flow

When the class concludes, an event-driven pipeline takes over to process the chunks into a final video.

### Step 1: End Class Trigger
The instructor clicks "End Class". The client sends a request to the backend:
`POST /end-class/:classID`

The database updates the class status to `'Ended'`.

### Step 2: Raising the Event
The backend service raises a `class-ended` event. This event is sent to **AWS EventBridge**, a serverless event bus used to connect application components together, enabling a highly scalable event-driven application.

### Step 3: Stitching with AWS Lambda
EventBridge routes the `class-ended` event to an **AWS Lambda** function. 
* The Lambda function's job is to read the specific folder in S3 containing the 100-200 chunks for this `classID`.
* It merges (stitches) all these small files together into one continuous, raw video file.
* It uploads this merged file back to S3 (e.g., `s3:put /meeting/{id}/merged.mp4`).

### Step 4: Transcoding
The `s3:put` action (the creation of the merged file) triggers another rule in EventBridge.
* EventBridge invokes **AWS Elastic Transcoder** (or AWS MediaConvert).
* The transcoder pulls the raw merged file and converts it into standard formats and resolutions (e.g., 1080p HQ, 720p, 480p) suitable for different devices and bandwidths.

### Step 5: Final Storage and Availability
The Elastic Transcoder outputs the final processed videos back to a destination S3 bucket (e.g., `/meeting/{id}/stitched/HQ.mp4`).
* A final event notifies the backend database that the video processing is complete.
* The UI updates for students, changing the status to **"Recording Available"**.

### Step 6: Delivery via Adaptive Bitrate Streaming (ABR)
To send the video to customers with multiple variants (e.g., 1080p, 720p, 480p), the system uses **Adaptive Bitrate Streaming (ABR)** via protocols like HLS (HTTP Live Streaming) or MPEG-DASH.
* **Manifests and Chunks:** Instead of a single MP4, the transcoder generates a master playlist file (e.g., `.m3u8`) and many small video segments for each resolution.
* **CDN Caching:** A Content Delivery Network (CDN) like AWS CloudFront sits in front of the S3 bucket to cache these files globally.
* **Smart Client Player:** The user's video player downloads the master playlist. It continuously monitors the user's internet speed and CPU. If their bandwidth drops, the player seamlessly switches to downloading the 480p chunks instead of the 1080p chunks, ensuring no buffering.

---

## 3. High-Level Architecture Diagram

```mermaid
graph TD
    Client[Instructor UI] -->|1. POST /end-class/:classID| API[Backend API]
    API -->|2. Update DB status: 'Ended'| DB[(Database)]
    API -->|3. Raise 'class-ended' event| EB1[AWS EventBridge]
    
    subgraph Video Stitching
        EB1 -->|Trigger| Lambda[AWS Lambda <br> Stitches Chunks]
        S3_Chunks[(S3: Video Chunks <br> 100-200 files)] -.->|Read Chunks| Lambda
        Lambda -->|s3:put /meeting/id/merged.mp4| S3_Merged[(S3: Merged Video)]
    end
    
    subgraph Video Transcoding
        S3_Merged -->|s3:put event| EB2[AWS EventBridge]
        EB2 -->|Trigger| Transcoder[AWS Elastic Transcoder]
        Transcoder -.->|Read Merged Video| S3_Merged
        Transcoder -->|Write HLS Playlist & Chunks| S3_Final[(S3: Video Variants <br> 1080p, 720p, 480p)]
    end
    
    subgraph Delivery & Playback
        S3_Final -->|Origin| CDN[CDN <br> AWS CloudFront]
        CDN -.->|Cache Delivery| Player[Student Video Player]
        Player -.->|Adaptive Bitrate Request| CDN
    end
    
    style EB1 fill:#ff9900,stroke:#333,stroke-width:2px;
    style EB2 fill:#ff9900,stroke:#333,stroke-width:2px;
    style Lambda fill:#f9a825,stroke:#333,stroke-width:2px;
    style Transcoder fill:#f9a825,stroke:#333,stroke-width:2px;
    style S3_Chunks fill:#4caf50,stroke:#333,stroke-width:2px;
    style S3_Merged fill:#4caf50,stroke:#333,stroke-width:2px;
    style S3_Final fill:#4caf50,stroke:#333,stroke-width:2px;
    style CDN fill:#e1bee7,stroke:#8e24aa,stroke-width:2px;
    style Player fill:#b3e5fc,stroke:#03a9f4,stroke-width:2px;
```

### Why this architecture is powerful:
* **Fault Tolerance:** If stitching fails, the Lambda can simply retry without losing the raw chunks.
* **Serverless Scalability:** Using Lambda and EventBridge means the system scales automatically. If 500 classes end at exactly 5:00 PM, AWS spins up 500 Lambdas instantly to stitch the videos, with zero idle server costs.
* **Speed:** Because chunks are uploaded during the live class, the only latency after the class ends is the stitching and transcoding process.

---

## 4. The Hidden Flaws (Why this isn't a perfect design)

While the architecture above is a great starting point, it has significant bottlenecks that will fail at scale. In a system design interview, identifying these flaws is crucial:

### Flaw 1: AWS Lambda Limitations
AWS Lambda is not designed for heavy, sustained I/O operations on massive files. 
* **Storage Limits:** Lambda's `/tmp` directory has a maximum limit of 10GB. A 2.5-hour raw 1080p video could easily exceed this, causing the Lambda to crash out of memory/storage during the stitching phase.
* **Timeout Limits:** Lambda has a hard timeout of **15 minutes**. Downloading 200 chunks, stitching gigabytes of video, and uploading the massive merged file back to S3 could easily take longer than 15 minutes, resulting in a timeout failure.

### Flaw 2: Wasted I/O and Double Processing
The system reads all chunks from S3, writes a massive merged file to S3, and then the Elastic Transcoder reads that massive file from S3 *again* just to break it back down to transcode it. This is highly inefficient and incurs massive data transfer costs.

### Flaw 3: Delayed Availability
We are waiting for the class to end before we even start the heavy lifting (transcoding). For a 2.5-hour class, transcoding the entire merged file at the end will take a significant amount of time, frustrating students who want the recording immediately.

---

## 5. The Optimized Solution: On-The-Fly Transcoding

To build a truly production-ready, highly scalable system, we must change the paradigm: **Do not wait for the class to end to start transcoding.**

1. **Transcode Chunks Immediately:** As soon as a 1-minute chunk is uploaded to S3 *during* the live class, immediately trigger a serverless function or transcoder to convert *that specific chunk* into 1080p, 720p, and 480p variants.
2. **Store Transcoded Chunks:** Save these processed chunks directly into the final CDN-facing S3 bucket.
3. **Instant Availability at Class End:** When the instructor clicks "End Class", 99% of the video is **already transcoded**. The system only needs to process the very last chunk and generate the final `.m3u8` manifest playlist file. 
4. **Result:** The recording becomes available to students within **seconds** of the class ending, completely bypassing the massive Lambda stitching bottleneck.
