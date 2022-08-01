# skidos_vod

### Video On Demand Service Assignment - HLD and LLD for a Video on Demand backend microservice to serve fun learning videos

#### Requirements

1. Only authenticated users should have access to the videos
2. Video microservice should support adaptive bitrate streaming
3. Video microservice should support encryption of video
4. Video search feature

### High Level Design of VoD Service


#### Upload Microservice

![2022-07-31](https://user-images.githubusercontent.com/37400411/182039310-2cb57c42-b7d4-4752-afd4-5b83fc22ebdc.png)



The upload microservice will be triggered while creating the content on this application. This also helps in achieving requirement 2 shown above which is adaptive bitrate streaming. We will talk about it later in this document. 

When the uploader hits the upload file or create request on the API gateway, the request will be redirected to the Load balancer which will choose the upload service instance available from the cluster. The upload service will process the file and send the media file to the video encoder tool which will further process the video and output the number of files of different resolutions and also the encryption will be done at this point. Once done, the upload service will store the output files to File Storage/Video Storage Database and the metadata will be stored in SQL database. Also, once the encoded files are stored in File Storage, it will also be sent to the CDN and the CDN URL will be persisted in MetaData Storage.

#### Video Microservice

![2022-08-01](https://user-images.githubusercontent.com/37400411/182084007-cbdf2050-fa3a-49a0-bba6-77f986d00313.png)

When the end user makes a view request for a video or searches for any video using keywords, the request will be redirected to the Video microservice. It interacts with Metadata storage to fetch the information especially nearest CDN with video Url and responds back with that information and then the user makes request to the nearest CDN node to view the video. For Content discovery, video microservice interacts with the Elastic search and metadata storage to send back the matched video and the simailar results in the response.

### Overall Architecture


![2022-08-01 (1)](https://user-images.githubusercontent.com/37400411/182087754-2141b54b-5d53-4d2f-8c5f-e8e9da1e0969.png)

In the diagram above, you might have noticed that we are using AWS for our application as AWS has reach more than any other cloud provider all over the world.

#### EC2 -
Amazon EC2 is a cloud computing platform that can be auto-scaled to meet demand.
Different hardware and software configurations can be selected. Different geographical locations can be selected be closer to users, as well as providing redundancy in case of failures.
Persistent storage can be provided by Amazon EBS (Elastic Block Storage). Amazon S3 (Simple Storage Service) data can also be accessed with Amazon EC2 instances, and is free if they are in the same region.

#### ELB - 
1. This service distributes application traffic across services.
2. The Load Balancer is a single point of contact for incoming web traffic.
3. The single point of contact means that the traffic hits the Load Balancer, spreading out the load between the resources.
4. It balancer accepts requests and directs them to the appropriate instances.
5. It ensures that one resource won't get overloaded, and that the traffic is spread out.

#### Niginx Plus API gateway on AWS
NGINX Plus is an application delivery platform built on NGINX, an open-source web server and reverse proxy for high-traffic sites. NGINX Plus has enterprise-ready features for advanced load balancing, web and mobile acceleration, application security, monitoring, and management.

#### MySQL
For this application, we are using MySQL to store meta data related to the videos uploaded on server. It is high performant, highly accessible and typically runs with no downtime.

#### Amazon S3 storage
Amazon S3 is a secure and redundant storage system that uses a flat object storage structure.
Data is stored in three different physical availability zone that also provides redundancy.
Different storage classes options are offered, balacing cost and availability. Automated management of storage can move objects to cheaper storage classes if they are accessed infrequently. Objects can also be automatically expire and be deleted after a specified time.
Access can be managed by access control lists or AWS Identity and Access Management (IAM).
Server access requests to buckets can be logged for security and analytics. AWS CloudTrail also enables logging at object level.

Moreover S3 easily integrates with AWS cloudfront which is also useful for this application.

#### AWS Cloudfront
Amazon CloudFront is a globally distributed network of servers that can deliver content to users.
The netowrk has edges (servers) in many locations around the world. The servers cache content closer to the users to improve access speed which in this application's use case helps users to access videos with lowest possible latency.
Creation of new distributions can be automated.
Caching data in multiple locations also provide data redundancy, improving reliability of access.
Amazon CloudFront uses RTMP protocol for video streaming and HTTP or HTTPS for web content. It also uses CMAF which will be used for this application.
Content delivery networks are suited for delivery of bulky data, like video streaming, downloading larger files and software, and to make website access faster.

#### AWS Elemental MediaConvert

AWS Elemental MediaConvert is a file-based video transcoding service with broadcast-grade features. It allows you to easily create video-on-demand (VOD) content for broadcast and multiscreen delivery at scale.
Using MediaConvert allows content creators to provide users with multiple resolutions for each of the videos which helps in Adaptive bitrate streaming.

#### AWS ElasticSearch
Elasticsearch is a distributed, free and open search and analytics engine for all types of data, including textual, numerical, geospatial, structured, and unstructured. 
It approximately takes 10ms to fetch the data, for which RDBMS can take up to 10s.
However for this application we will be using AWS ElasticSearch.

### Low Level Design

#### API Design

##### Video Upload API

Path
POST /video-contents/v1/videos

Body
{

videoTitle : Title of the video

videoDescription : Description of the video

tags : Tags associated with the video

category : Category of the video, e.g. Maths, Science,

videoContent: Stream of video content to be uploaded

}

##### Video Search API

Path
GET /video-contents/v1/search-query/

Query Param
{

user-location: location of the user performing search

}

##### Video Stream API

Path
GET /video-contents/v1/videos/

Query Param
{

offset: Time in seconds from the beginning of the video

}

#### Data Model

Within the scope of this problem, we need to persist video metadata in the database. The data-model for storing the metadata is shown in the table below.


![Screenshot 2022-08-01 131350](https://user-images.githubusercontent.com/37400411/182098981-9d2dd7ee-c000-4891-877b-c54aab6fe9bc.png)

#### Video Uploader Component

This module is executed when a content creator/uploader uploads a content. It is responsible for distributing the content on CDN to provide optimal user experience.
![Video Uploader Component Design](https://user-images.githubusercontent.com/37400411/182103529-33fe9a16-0e9c-4974-88cb-a182de0af179.png)


The diagram above depicts the sequence of operations which gets executed when content creators upload the video content (TV Show or movie).

1. The content creator uploads the raw video content.
2. The Upload_Service sends the video to Video_Encoder (AWS Elemental) for encoding purpose.
3. The Video_Encoder encodes each of the segments in different codec and resolution.
4. The encoded file segments are stored in the file storage.
5. The Upload_Service reads the encoded file segments from the distributed file storage system.
6. The Upload_Service distributes the encoded file segments in CDN (Cloudfront).
7. The Upload_Service persists the CDN url links of the videos in the data_storage (MySQL).

AWS Elemental can convert the video file to various different formats like Mpeg, HLS, however we will be using CMAF in our application to support wider audience which actually is supported by all major video service providers like google, apple and microsoft.

##### Why CMAF?
As we said above, to reach a wider audience base, VoD providers need to create multiple copies of each stream file in different file containers. 

So these same files reduce productivity by nearly double the cost of packaging, storing, and competing with each other on CDN servers. 

 Let’s look at what Akamai said about this: 

“These same files, although representing the same content, cost twice as much to package, twice as much to store on origin, and compete with each other on Akamai edge caches for space, thereby reducing the efficiency with which they can be delivered.” 

The importance of CMAF comes into play here. As a standard streaming format across all platforms, it helps us with single-approach encoding, packaging and storage. So, Common Media Application Format makes the video streaming process much cheaper and less complicated.

Shortly, With CMAF streaming, you have one set of audio/video files in a fragmented MP4 format with very lightweight manifest files for all four adaptive bitrate (ABR) formats. Hypothetically, this cuts encoding and storage costs by 75% and makes your caching much more efficient. 

It also supports Common Encryption Format across all formats which keeps our data/video streams secure from unauthorized access.


#### Video View Component (Playback)

This workflow is responsible for orchestrating operations when a user places a playback request. It co-ordinates between different entities such as Video Microservice (for checking user authorization and licensing, for deciding the best playback experience), CDN, MySQL

The deta![View Video Component](https://user-images.githubusercontent.com/37400411/182110248-35bd2073-4cde-48a4-91dc-6213769d41db.png)

The details about each of the steps in the sequence diagram is listed below.

1. The client places a request to playback a video which gets directed to the Video_Microservice.
2. The Video_Microservice authenticates the users request using the token parameter in the request.
3. The Video_Microservice then picks the best CDN url from which the playback can be served.
4. The CDN url is returned to the client(mobile/laptop).
5. The client retrieves the content from CDN.
6. The client publishes the events for the playback experience to the Playback_Service.
7. The Video_microservice tracks events to measure the playback experience.
