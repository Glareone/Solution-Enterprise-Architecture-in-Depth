# Designing Youtube | Netflix | Hulu
<img width="985" alt="image" src="https://github.com/user-attachments/assets/f421d9ee-8250-4325-8bfd-8f299c7ce4d6" />

## Questions upfront
* Can we leverage the existing cloud infrastructure from Amazon, Microsoft, or Google, or we are focusing on inventing them by ourselves?
  - We could expect to hear that we can leverage, because it's unrealistic for most of the companies.
* [Cost optimization on CDN level](Youtube-Netflix-Hulu.md#cost-optimizations)
  - Definitely worth to discuss because it costs a lot.

## Functiona moments and Requirements
* [Video uploading flow](Youtube-Netflix-Hulu.md#video-uploading-flow)
  - (Optional) Recommendation system based on your preferences (immediate changes, scheduled changes)
  - bitrates & transcoding questions

* [Video streaming flow](Youtube-Netflix-Hulu.md#video-streaming)
  - (Optional) Prepared set of 100 videos on the main screen
  - (Optional) Offline streaming?
  - Switching between bitrate for smooth user experience?
* (Optional, it leads us to another system design) [Live streaming](https://github.com/Glareone/Azure-Solution-and-Enterprise-Architecture-in-Depth/blob/main/System%20Design%20Interview%20%26%20Architecture%20in%20Practice/live-streaming.md): It refers to the process of how a video is recorded and broadcasted in real time.
The notable differences are:
  - Live streaming has a higher latency requirement, so it might need a different streaming protocol.
  - Live streaming has a lower requirement for parallelism because small chunks of data are already processed in real-time.
  - Live streaming requires different sets of error handling. Any error handling that takes too much time is not acceptable.


## Non-Functional Requirements
* Ability to upload videos fast
* Smooth video streaming
* Ability to change video quality
* Low infrastructure cost
* High availability, scalability, and reliability requirements
* Clients supported: mobile apps, web browser, and smart TV

## DAU & Costs
* Assume the product has 5 million daily active users (DAU).
* Users watch 5 videos per day.
* 10% of users upload 1 video per day.
* Assume the average video size is 300 MB.
* Total daily storage space needed: 5 million * 10% * 300 MB = 150TB
* CDN cost.
* When cloud CDN serves a video, you are charged for data transferred out of the

### CDN. Discussing how to reduce the cost of CDN might be very important on the interview
* Let us use Amazon’s CDN CloudFront for cost estimation. Assume 100% of traffic is served from the United States. The average cost per GB is $0.02.
For simplicity, we only calculate the cost of video streaming.
* 5 million * 5 videos * 0.3GB * $0.02 = $150,000 per day.

## High Level Design
<img width="602" height="400" alt="image" src="https://github.com/user-attachments/assets/661c79d6-20c5-46da-a1b3-a9050e875b82" />

* CDN and blob storage are the cloud services we will leverage.

## Video Uploading Flow
<img width="905" alt="image" src="https://github.com/user-attachments/assets/e0ccf9e5-894f-48ff-a39d-2c88e5ec8c09" />

* Load balancer: A load balancer evenly distributes requests among API servers.
* API servers: All user requests go through API servers except video streaming.
* Metadata DB: Video metadata are stored in Metadata DB. It is sharded and replicated to
meet performance and high availability requirements.
* Metadata cache: For better performance, video metadata and user objects are cached.
* Original storage: A blob storage system is used to store original videos. A quotation in
Wikipedia regarding blob storage shows that: “A Binary Large Object (BLOB) is a
collection of binary data stored as a single entity in a database management system” [6].
* Transcoding servers: Video transcoding is also called video encoding. It is the process of
converting a video format to other formats (MPEG, HLS, etc), which provide the best
video streams possible for different devices and bandwidth capabilities.
* Transcoded storage: It is a blob storage that stores transcoded video files.
* CDN: Videos are cached in CDN. When you click the play button, a video is streamed
from the CDN.
* Completion queue: It is a message queue that stores information about video transcoding
completion events.
* Completion handler: This consists of a list of workers that pull event data from the
completion queue and update metadata cache and database.

### Video Uploading Flow. Transcoding
Transcoding is computationally expensive and time-consuming.  
Meta uses DAG (Directed Acyclic Graph) programming model which defines tasks in stages so they can be executed parallelly or sequentially.
<img width="500" alt="image" src="https://github.com/user-attachments/assets/f9369051-ffbf-47ad-a672-d07c4ce6b750" />

* Inspection: Make sure videos have good quality and are not malformed.
* Video encodings: Videos are converted to support different resolutions, codec, bitrates, etc. Figure 14-9 shows an example of video encoded files.
* Thumbnail. Thumbnails can either be uploaded by a user or automatically generated by the system.
* Watermark: An image overlay on top of your video contains identifying information about your video

<img width="500" alt="image" src="https://github.com/user-attachments/assets/c45e6136-790e-4a15-9905-83ea2be99d92" />

### Transcoding Architecture
<img width="500" alt="image" src="https://github.com/user-attachments/assets/0b4e5f4d-b3fc-4b5e-a3fa-52439a599d51" /> 

**Preprocessor**  
1. Video splitting. Video stream is split or further split into smaller Group of Pictures (GOP) alignment. GOP is a group/chunk of frames arranged in a specific order. Each chunk is an independently playable unit, usually a few seconds in length.
2. Some old mobile devices or browsers might not support video splitting. Preprocessor split videos by GOP alignment for old clients.
3. DAG generation. The processor generates DAG based on configuration files client programmers write. Figure 14-12 is a simplified DAG representation which has 2 nodes and 1 edge:
4. Cache video segments. Preprocessor stores Video parts and metadata in temp storage. For resiliency (retry mechanism).

**DAG. Scheduler**  
<img width="500" alt="image" src="https://github.com/user-attachments/assets/71b013b1-f790-4b4c-a6b3-65bfff37855f" />
Idea: split the video processing onto independent tasks.

**Resource Manager. Also known as Task Scheduler**  
<img width="500" alt="image" src="https://github.com/user-attachments/assets/cf84171e-ce7a-40db-8ad9-b8fd5e848beb" />
* Is responsible for managing the resource allocation and priorities management.
* Find optimal worker for the certain task
* Job Queue management

**Task Workers**  
<img width="250" alt="image" src="https://github.com/user-attachments/assets/7d742cfe-ad66-4e21-9265-e333fd84bb19" />

**Encoded Video**  
Is the final output of the encoding pipeline.

### Video Processing Optimizations.
<img width="350" alt="image" src="https://github.com/user-attachments/assets/5aa45b13-0634-4e23-ac78-df520297d2ab" />

* Upload video closer to the end user
* High Parallelism


## Video Streaming
**Protocols**
* MPEG-DASH "Moving Picture Experts Group" - "Dynamic Adaptive Streaming over HTTP"
* Apple HLS. "Http Live Streaming"
* Microsoft Smooth Streaming
* Adobe Http Dynamic Streaming (HDS)

Naive video streaming diagram:  
<img width="400" alt="image" src="https://github.com/user-attachments/assets/9e4bfb78-0a12-4e63-8a1a-c0f93bc6d1ff" />

Recommendation: They all support different video encodings. You have to choose the right streaming protocol for your business case. 

## Cost Optimizations
* For less popular content, we may not need to store many encoded video versions. Short videos can be encoded on-demand.
* Some videos are popular only in certain regions. There is no need to distribute these videos to other regions.
* Build your own CDN like Netflix and partner with Internet Service Providers (ISPs). Building your CDN is a giant project; however, this could make sense for large streaming companies. An ISP can be Comcast, AT&T, Verizon, or other internet providers. 
<img width="300" alt="image" src="https://github.com/user-attachments/assets/3e7cf93e-9cb4-4efa-a314-93b825abfddf" />

